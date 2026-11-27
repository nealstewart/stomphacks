# TMS320C6745 Memory & Audio Latency Guide

The Zoom MS Plus pedals (MS-70CDR+, MS-50G+, MS-60B+, MS-200D+, MS-80IR+) run on a **TI TMS320C6745** DSP. Its external SDRAM sits behind a narrow 16-bit bus (**EMIFB**) and two levels of on-chip cache (**L1P / L1D / L2**).

The DSP core can run up to 8 instructions per cycle, while external SDRAM is much slower. **How your effect reads and writes memory—in order vs. jumping around, and at what spacing—decides whether it uses a few percent of the CPU or stalls waiting for RAM.**

> **Quick reference — `instance` (`void **instance`):** Every audio and parameter handler receives `instance` as its first argument (see [The ZD2 ABI — The instance](zd2-abi.md#the-instance) for full details):
> - `instance[0]` — Host parameter object (passed to `get_param`)
> - `instance[1]` — Live `float32` coefficient table (`coef[16]`, knob values & bypass fade)
> - `instance[2]` — Per-instance state area (`state_bytes`, zeroed at init; proven safe floor `192` B)
> - `instance[3]` — Pointer to this slot's 3-word external SDRAM delay-buffer descriptor (covered below)

---

## 1. External SDRAM: The 16-Bit EMIFB Bus

Unlike the larger TMS320C6747 (which has a 32-bit SDRAM bus), the 176-pin **TMS320C6745** only has a **16-bit external SDRAM bus** (**EMIFB**). The MS-70CDR+ has 8 MB of SDRAM total, and gives each effect slot `705,600` bytes (`176,400` `float32` samples) at `instance[3]`.

* **Every `float32` takes two transfers**: A `float32` is 4 bytes (32 bits), so reading or writing one sample in SDRAM takes **two** 16-bit transfers over EMIFB.
* **Cache-line fills amplify this**: Pulling one 64-byte L1D cache line (16 `float32` samples) takes **32** 16-bit transfers. Pulling one 128-byte L2 cache line (32 `float32` samples) takes **64** 16-bit transfers.
* **Row hits vs. row misses**: SDRAM is split into banks, rows, and columns.
  * **Page hit (same row)**: Reading consecutive samples in an already-open row streams at 1 halfword per EMIFB clock after the initial `CAS` delay.
  * **Page miss (row conflict)**: Jumping to a different row in the same bank forces a **Precharge (`tRP`) → Activate (`tRCD`) → CAS (`CL`)** cycle before any data moves. That adds dozens of stall cycles on top of the transfer itself.

### Per-Slot SDRAM Buffer (`instance[3]`) & Safe Limits

Each effect slot gets a pre-allocated SDRAM delay buffer described by a 3-word (`uint32_t[3]`) struct at `instance[3]`:

| Word | Field | Meaning |
|---|---|---|
| `dd[0]` | `base_addr` | Start of the slot's SDRAM buffer (4-byte aligned, non-zero) |
| `dd[1]` | `end_addr` | End of the buffer (`base_addr + size_bytes`) |
| `dd[2]` | `size_bytes` | Buffer size in bytes (`705,600` B = `176,400` `float32` samples = 4.0 s @ 44.1 kHz) |

* **How we know `705,600` bytes is the slot size**: Stock 4-second delays (`Delay`, `Hold Delay`, `Ping-Pong Delay`) index `4 * D` bytes inside `[dd[0], dd[1])` at 44.1 kHz, up to $D_{\max} = (4000 \times 441) / 10 = 176{,}400$ samples (`705,600` bytes), wrapping against `dd[1]` and `dd[2]`.
* **Confirmed DIY usage on hardware**:
  * **~512 KB (`524,352`–`524,544` bytes)** — stereo `2 × 65,536`-sample `float32` ring buffers with `+64`-byte cache skew (~1.49 s stereo @ 44.1 kHz), leaving ~181 KB free below the `705,600`-byte limit.
  * Smaller tested sizes include `32,768` B (`2 × 4,096` float32 stereo pitch buffers) and `164,480` B (~160.6 KB multi-line stereo reverb).
  * A safe DIY cap is **`529,200` bytes** (3.0 s mono / 1.5 s stereo @ 44.1 kHz), which leaves a full 1.0 s (`176,400` B) margin.
* **Always validate `instance[3]` first**: Before using `instance[3]`, check all three words and output silence if anything looks wrong. During patch changes or previews, the audio callback can run while pointers aren't ready yet:
  ```c
  const uint32_t * restrict dd = (const uint32_t * restrict)instance[3];
  uintptr_t dbaseAddr = 0u;
  if (!dd) {
      ok = 0u;
  } else {
      uint32_t b = dd[0], e = dd[1], sz = dd[2];
      if (b == 0u || (b & 3u) != 0u || sz < NEED_BYTES || e <= b || (uint32_t)(e - b) != sz) {
          ok = 0u;
      } else {
          dbaseAddr = (uintptr_t)b;
      }
  }
  ```

---

## 2. The Cache Hierarchy (L1D → L2 → SDRAM)

Between the CPU registers and external SDRAM sit three fast on-chip memory blocks:

* **L1P (32 KB)**: Direct-mapped program cache (**32-byte line** = 8 instructions).
* **L1D (32 KB)**: **2-way set-associative** data cache (**64-byte line** = 16 `float32` samples).
* **L2 (256 KB)**: **4-way set-associative** shared program/data cache (**128-byte line** = 32 `float32` samples).

When your code loads (`LDW`) a delay sample from `instance[3]`, the chip checks **L1D → L2 → EMIFB/SDRAM**:

| Memory Tier | Size & Layout | Line Size | Approx. Latency | What Usually Lives Here |
|---|---|---|---|---|
| **Registers (`A0..A31`, `B0..B31`)** | 64 × 32-bit registers (split `.A`/`.B` sides) | 4 B / 8 B (`LDW` / `LDDW`) | **1 cycle** (up to 2 loads/cycle via `.D1` + `.D2`) | Unpacked `instance[2]` filter state (`st_*`), block-rate constants (`k_*`), current sample |
| **L1D Cache** | 32 KB, **2-way** | **64 B** (16 floats) | **1 cycle** (`4` delay slots hidden by `SPLOOP`) | `ctx[1]` 16-frame stereo bus (`128 B`), `instance[1]` `coef[16]` (`64 B`), `instance[2]` state (`192 B`), stack (`B15`), active delay taps |
| **L2 Cache / SRAM** | 256 KB, **4-way** | **128 B** (32 floats) | **~8–12 cycles** (L1D miss → L2 hit) | `.text` code, `.const` tables, short delay lines (`<= 32–64 KB`, e.g. pitch shifters, flangers) |
| **EMIFB (16-bit SDRAM)** | 8 MB off-chip (`705,600 B` per slot at `instance[3]`) | Filled in **64 B / 128 B** lines | **~65–85 cycles** (page hit)<br>**~110–150+ cycles** (row conflict) | Long delays, loopers, multi-tap reverbs that don't fit in L2 |

### How L1D Maps Addresses

Physically, the C6745's 2-way L1D cache is a 2D table of **`128` rows (sets) × `2` columns (ways)** (`256` rows if no L1D is reserved as SRAM), where each cell stores a `tag` and one **64-byte line (16 `float32` samples)**. On every memory access, the CPU slices the 32-bit pointer address into three fields—effectively using bits `12..6` as a hardware hash into the table:

```
31                    13 12          6 5            0
+-----------------------+-------------+--------------+
|        Tag (19)       | Set Idx (7) | Offset (6)   |
+-----------------------+-------------+--------------+
  "which 8 KB region?"    "which row?"  "byte 0..63"
```

1. The CPU jumps directly to **one row**: `row = cache[Set_Index]` (it never searches other rows).
2. It compares `Tag` against `row[0].tag` and `row[1].tag`:
   * **Hit:** Return the float at `Offset` (1 cycle).
   * **Miss:** Fetch 64 bytes (`16` floats) from L2/SDRAM and overwrite the older of `row[0]` or `row[1]`.

### How much time do we have?

At 44.1 kHz with 16 stereo frames per block, the pedal calls your audio function every **362.8 microseconds** (`2,756.25` times/sec). At `~300–375 MHz`, the **entire 6-slot chain plus firmware** shares about `110,000–135,000` cycles per block (`~7,000–8,400` cycles per stereo sample).

One uncached SDRAM row-conflict miss (`~120` cycles) costs as much time as **hundreds** of pipelined float multiplies and adds (`MPYSP` / `ADDSP`).

---

## 3. EDMA3 (Background Transfers)

The C6745 has an **EDMA3** DMA controller:
* **Firmware**: Uses EDMA3 to copy 16-frame audio blocks between the **McASP** audio ports (ADC/DAC) and internal RAM in the background without stopping the DSP.
* **`.ZD2` plugins**: Dynamically loaded `.ZD2` effects don't own EDMA channels or interrupts—they just read/write memory through the L1D and L2 caches. That's why cache-friendly access patterns matter so much.

---

## 4. Four Common DSP Access Patterns

```
  C674x Core (A0..A31, B0..B31)
       │  2x 32-bit loads/cycle (.D1 + .D2)
       ▼
  ┌────────────────────────────────────────────────────────┐
  │ L1D Cache (32 KB, 2-Way Set-Associative, 64 B Line)    │ ◄── Pattern A: ctx[1] (128 B) & instance[2] (192 B)
  │   1 Line = 16 consecutive float32 samples              │     stay 100% pinned here (0 misses in loop)
  └───────────────────────────┬────────────────────────────┘
                              │ L1D Miss (64 B fill, ~8-12 cycles from L2)
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │ L2 Unified Cache (256 KB, 4-Way, 128 B Line)           │ ◄── Pattern C: 32 KB pitch ring buffer
  │   1 Line = 32 consecutive float32 samples              │     fits inside L2 when not evicted by other slots
  └───────────────────────────┬────────────────────────────┘
                              │ L2 Miss (32 or 64 x 16-bit transfers over EMIFB)
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │ EMIFB 16-Bit External SDRAM (8 MB total; 705 KB/slot)  │ ◄── Pattern B & D: Multi-second delays & multi-line
  │   RowHit: ~65-85 cycles   RowConflict: ~110-150 cycles │     tanks (spatial locality & set skew critical!)
  └────────────────────────────────────────────────────────┘
```

### Pattern A: Register-Only State (Filters, Drives, Dynamics)

* **How it works**: Copy `instance[2]` (`<= 188` bytes, at most 3 L1D lines) into local C variables (`st_*`) **once** before the 16-sample loop, and compute knob math (`sinf`, `rcp`, `expf`) once per block into locals (`k_*`).
* **Cache behavior**: Inside `for (i0 = 0; i0 < 16; i0++)`, the **only** memory touches are `eff[i0]` (Left, 64 B = **1 L1D line**) and `eff[i0 + 16]` (Right, 64 B = **1 L1D line**), which are already hot in L1D.
* **Cost**: **0 L1D misses, 0 L2 accesses, 0 SDRAM transfers** inside the sample loop.

### Pattern B: Sequential Single/Dual-Tap Delay (`DELAY_C.ZD2`, Flangers, Echoes)

* **How it works**: `wpos` steps forward by `+1` sample (`+4` bytes) per frame, and `buf[(wpos - D) & mask]` also steps forward by `+1` sample per frame.
* **Cache behavior**:
  * When a tap crosses a 16-sample (64-byte) boundary, it misses L1D once and loads a **64-byte line** containing **16 consecutive floats**.
  * The other **15 frames** in that 16-sample block hit the same L1D line in 1 cycle.
  * Even 4-point cubic interpolation (`i0 + 1` .. `i0 - 2`, 16 bytes wide) stays inside one L1D line 81% of the time (or two adjacent lines at a boundary).
* **Cost**: At most **1 cache-line fill per tap per 16-sample block**. Even a `~100`-cycle SDRAM miss averages out to **`~6` cycles per sample**.

### Pattern C: 1D Search / Mini-WSOLA Pitch Shifters

* **How it works**: When a pitch grain crossfades, WSOLA searches `[-128..+128]` samples (`256` floats = `1,024` B = `16` L1D lines) to find the best phase match ($\sum_{k=0}^{15} |\text{ref}[k] - \text{cand}[k]|$).
* **How to keep it fast**:
  1. **Copy `ref[16]` to the stack once**: 16 floats = 64 bytes (1 L1D line) on `B15`, which stays hot in L1D instead of re-reading the ring buffer every candidate step.
  2. **Coarse-then-fine search**: Scan coarsely across the 1,024-byte window first (warming those 16 L1D lines in order), then refine within `+-8` samples (`+-32` bytes) using lines already in L1D.
  3. **At most 1 search per 16-frame block**: Prevents two pitch voices from both running searches in the same 0.36 ms block.

### Pattern D: Multi-Line Reverb Tanks & L1D Cache Hash Collisions

There is no `malloc()` in a `.ZD2` effect. The pedal gives you one flat SDRAM array (`float *sdram = (float *)dd[0]`), and you slice it into delay buffers with pointer offsets.

#### The power-of-two collision trap

Suppose a reverb slices `sdram` into **8 ring buffers of `1024` floats (`4096` B = `0x1000` B)**, all indexed by the same cursor `wpos`:

```c
float *dline_0 = sdram + 0 * 1024;   // base + 0x0000
float *dline_1 = sdram + 1 * 1024;   // base + 0x1000
float *dline_2 = sdram + 2 * 1024;   // base + 0x2000
...
```

Look at the **Set Index** bits (`bits 12..6`) for the 4 even buffers at the same `wpos`:

```
Address        | Tag (31..13) | Set Index (12..6) | Offset (5..0)
---------------+--------------+-------------------+--------------
base + 0x0000  | ...000       | 0000000  (Set 0)  | 000000
base + 0x2000  | ...001       | 0000000  (Set 0)  | 000000
base + 0x4000  | ...010       | 0000000  (Set 0)  | 000000
base + 0x6000  | ...011       | 0000000  (Set 0)  | 000000
```

All 4 buffers hash to **`cache[0]`** (and the 4 odd buffers all hash to **`cache[64]`**), leaving the other 126 rows unused. Because `cache[0]` only holds 2 ways, touching all 4 buffers inside the sample loop overwrites the older lines before the next sample `wpos + 1` can reuse the other 15 floats—causing **conflict misses on every sample** instead of once every 16 samples.

#### The fix: `+64`-byte (`+16`-float) array padding

Leave **1 unused cache line (`64` B = `16` floats)** of padding after each buffer (`1040`-float = `0x1040`-byte stride), while still wrapping accesses with `wpos & 1023u`:

```c
float * restrict dline_0 = sdram + 0u * 1040u;   // base + 0x0000
float * restrict dline_1 = sdram + 1u * 1040u;   // base + 0x1040
float * restrict dline_2 = sdram + 2u * 1040u;   // base + 0x2080
...
```

Adding `0x40` (`64` B) is the smallest increment that flips bit 6 (the lowest bit of **Set Index**) without breaking 64-byte cache-line alignment—costing only `512` B total for 8 buffers (`0.07%` of the slot)—and breaks the hash collision so each buffer lands in its own row:

```
Address        | Tag (31..13) | Set Index (12..6) | Offset (5..0)
---------------+--------------+-------------------+--------------
base + 0x0000  | ...000       | 0000000  (Set 0)  | 000000
base + 0x2080  | ...001       | 0000010  (Set 2)  | 000000
base + 0x4100  | ...010       | 0000100  (Set 4)  | 000000
base + 0x6180  | ...011       | 0000110  (Set 6)  | 000000
```

Now all 8 buffers use 8 separate rows (`cache[0, 2, 4, 6]` and `cache[65, 67, 69, 71]`). Each 64-byte fetch on sample `0` stays in L1D for samples `1..15` (**8 fetches per block instead of 128** — **16× fewer RAM stalls**).

---

## 5. Practical Rules for Writing Effects

1. **Unpack `instance[2]` into local variables before the 16-sample loop**: Don't load and store `state[k]` on every sample when it can sit in CPU registers.
2. **Move knob math out of the sample loop**: Compute `sin`, `cos`, `tan`, `exp`, `log`, `pow`, and `1/x` once per 16-sample block, so the inner loop only does simple multiplies and adds.
3. **Never space parallel power-of-two buffers by exact powers of two**: Pad the stride by `+64` bytes (`+16` floats, one L1D line) so parallel taps don't collide in the 2-way L1D cache (e.g. `1040`-float stride for `1024`-sample buffers).
4. **Cap per-block SDRAM searches**: Cache any reference window on the stack (`B15`) once, and limit searches to at most one per 16-frame block.
5. **Prefer small inline math over big `.const` lookup tables**: A random lookup into a large `.const` table can miss L1D (`~10` cycles from L2 or `~100` from SDRAM), while a few register instructions (`_rcpsp`, `_rsqrsp`, or a short polynomial) take a few cycles with zero cache misses.
