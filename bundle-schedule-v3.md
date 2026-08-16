# Bundle schedule v3 — shadow-register model (Phase 2 rebuild #2)

Rebuild of `bundle-schedule-v2.md` (65,605 cycles) under the adopted
`load-stationary` shadow register (`notes.md` §11.26–§11.28). `-v2` is
preserved as the before/after baseline for this specific fix, same role
`bundle-schedule.md` plays for `-v2`.

**Model rules in effect here, only one changed from `-v2`:**
- `steady-state-stream-qk`/`-v`: latency 159, occupancy 32 — unchanged.
- **`load-stationary`: latency = occupancy = 128, now gated by the prior
  stream's *occupancy*-end, not its full latency** — confirmed real
  double-buffered stationary registers per PE in Gemmini's actual RTL
  (`PE.scala`'s `c1`/`c2`, gated by a `propagate` signal), checked
  directly rather than inferred from the higher-level ISA docs, which are
  silent on it. This removes the 127-cycle idle bubble at each K↔V
  transition — but **not** `load-stationary`'s own 128-cycle occupancy,
  which still fully consumes the single matmul-issue slot (nothing else
  can issue while it's mid-load, shadow register or not). "Cost → 0" was
  an overstatement, corrected in `notes.md` §11.27: real saving is 127
  cycles/transition (254/slice), not all of `load-stationary`'s cost.
- SFU, DMA: unchanged.
- S (×4): **unaffected** — its hazard is entirely intra-K-phase (head `i`
  vs. `i+4`, both inside one phase's 224-cycle burst), untouched by a
  fix that only changes the transition *between* phases. Margin stays 65
  cycles.
- P (WAR across slices): re-checked, margin shrinks **638→511** cycles
  (same 127-cycle shrink as the removed bubble, since P's hazard spans
  exactly the compressed boundary). Still safe.
- O (rescale-then-add): re-checked — and the *old* justification ("every
  K-phase `softmax-update` finishes before any V-phase `v` starts")
  actually breaks under this model (`v(head 0)` now issues at Δ512,
  before `softmax-update(head 7)` finishes at Δ543). Not actually a
  problem: the real per-head margin (`softmax-update(i)` vs. `v(i)`,
  same `i` — different heads' O locations never conflict, already ×8)
  is a constant 193 cycles regardless of head, since both times share
  the same `+32i` term. Shrank from 320 (v2) → 193, same root cause as P,
  but never close to tight.

**Status: complete.** Total Q-tile latency: **49,476 cycles** — 1.33×
faster than `-v2`'s 65,605, 3.63× faster than the original 179,397.

## Prologue

Identical through cycle 288 to both prior schedules — this boundary was
always DMA-gated (`load-stationary(K, slice 0)` waits on `load-K/V(K,
chunk 0)`), never involved a full-latency stream wait, so neither fix
touches it. See `bundle-schedule-v2.md`'s Prologue section for the table.

## Slice 0

| cycle | matmul-issue | SFU | DMA |
|---|---|---|---|
| 288 | `steady-state-stream-qk(head 0)` | idle | ⋯ `load-K/V(V, chunk 0)`, busy til 360 |
| 320 | `steady-state-stream-qk(head 1)` | idle | ⋯ busy til 360 |
| 352 | `steady-state-stream-qk(head 2)` | idle | ⋯ busy til 360 |
| 360 | ⋯ `qk(head 2)`, busy til 384 | idle | `load-K/V(K, chunk 1)` |
| 384 | `steady-state-stream-qk(head 3)` | idle | ⋯ busy til 520 |
| 416 | `steady-state-stream-qk(head 4)` | idle | ⋯ busy til 520 |
| 447 | ⋯ `qk(head 4)`, busy til 448 | `softmax-update(head 0)` | ⋯ busy til 520 |
| 448 | `steady-state-stream-qk(head 5)` | ⋯ `su(head 0)`, busy til 479 | ⋯ busy til 520 |
| 479 | ⋯ `qk(head 5)`, busy til 480 | `softmax-update(head 1)` | ⋯ busy til 520 |
| 480 | `steady-state-stream-qk(head 6)` | ⋯ `su(head 1)`, busy til 511 | ⋯ busy til 520 |
| 511 | ⋯ `qk(head 6)`, busy til 512 | `softmax-update(head 2)` | ⋯ busy til 520 |
| 512 | `steady-state-stream-qk(head 7)` | ⋯ `su(head 2)`, busy til 543 | ⋯ busy til 520 |
| 520 | ⋯ `qk(head 7)`, busy til 544 | ⋯ busy til 543 | `load-K/V(V, chunk 1)` |
| 543 | ⋯ busy til 544 | `softmax-update(head 3)` | ⋯ `V(chunk 1)`, busy til 680 |
| 544 | `load-stationary(V, slice 0)` | ⋯ `su(head 3)`, busy til 575 | ⋯ busy til 680 |
| 575 | ⋯ `load-stationary`, busy til 672 | `softmax-update(head 4)` | ⋯ busy til 680 |
| 607 | ⋯ busy til 672 | `softmax-update(head 5)` | ⋯ busy til 680 |
| 639 | ⋯ busy til 672 | `softmax-update(head 6)` | ⋯ busy til 680 |
| 671 | ⋯ busy til 672 | `softmax-update(head 7)` | ⋯ busy til 680 |
| 672 | `steady-state-stream-v(head 0)` | ⋯ `su(head 7)`, busy til 703 | ⋯ busy til 680 |
| 680 | ⋯ `v(head 0)`, busy til 704 | ⋯ busy til 703 | idle |
| 703 | ⋯ busy til 704 | idle | idle |
| 704 | `steady-state-stream-v(head 1)` | idle | idle |
| 736 | `steady-state-stream-v(head 2)` | idle | idle |
| 768 | `steady-state-stream-v(head 3)` | idle | idle |
| 800 | `steady-state-stream-v(head 4)` | idle | idle |
| 832 | `steady-state-stream-v(head 5)` | idle | idle |
| 864 | `steady-state-stream-v(head 6)` | idle | idle |
| 896 | `steady-state-stream-v(head 7)` | idle | idle |

Slice 0's next event: `load-stationary(K, slice 1)` issues at 928 (right
when `v(head 7)`'s occupancy frees the slot, not its full latency).
**Matmul-issue span 160→928 = 768 cycles/slice**, down from `-v2`'s 1,022.

Note what's *unchanged* from `-v2` here: every SFU and DMA row is
identical (K-phase's `softmax-update` cadence, the DMA prologue tail) —
the shadow register only moved the K→V handoff (`load-stationary(V)` now
at 544 instead of 671) and everything downstream of it.

## Universal matmul-issue/SFU table (every slice)

| Δ | matmul-issue | SFU |
|---|---|---|
| 0 | `load-stationary(K, slice N)` | idle |
| 128 | `steady-state-stream-qk(head 0)` | idle |
| 160 | `steady-state-stream-qk(head 1)` | idle |
| 192 | `steady-state-stream-qk(head 2)` | idle |
| 224 | `steady-state-stream-qk(head 3)` | idle |
| 256 | `steady-state-stream-qk(head 4)` | idle |
| 287 | ⋯ | `softmax-update(head 0)` |
| 288 | `steady-state-stream-qk(head 5)` | ⋯ |
| 319 | ⋯ | `softmax-update(head 1)` |
| 320 | `steady-state-stream-qk(head 6)` | ⋯ |
| 351 | ⋯ | `softmax-update(head 2)` |
| 352 | `steady-state-stream-qk(head 7)` | ⋯ |
| 383 | ⋯ `qk(head 7)`, busy til 384 | `softmax-update(head 3)` |
| 384 | `load-stationary(V, slice N)` | ⋯ |
| 415 | ⋯ | `softmax-update(head 4)` |
| 447 | ⋯ | `softmax-update(head 5)` |
| 479 | ⋯ | `softmax-update(head 6)` |
| 511 | ⋯ | `softmax-update(head 7)` |
| 512 | `steady-state-stream-v(head 0)` | ⋯ `softmax-update(head 7)`, busy til 543 |
| 544 | `steady-state-stream-v(head 1)` | idle |
| 576 | `steady-state-stream-v(head 2)` | idle |
| 608 | `steady-state-stream-v(head 3)` | idle |
| 640 | `steady-state-stream-v(head 4)` | idle |
| 672 | `steady-state-stream-v(head 5)` | idle |
| 704 | `steady-state-stream-v(head 6)` | idle |
| 736 | `steady-state-stream-v(head 7)` | idle |

Slice ends at Δ=768 (`load-stationary(K, slice N+1)` issues there).

`slice_start` values: `chunk_start(c) + s×768`, where `chunk_start(c) =
160 + c×6,144`.

## DMA overlay / chunk-boundary

Same four structural cases and same reasoning as `-v2` (DMA in-order,
fires the instant its WAR gate clears — untouched by either occupancy
fix). Only the numbers shrink further, tracking the new 768-cycle slice /
6,144-cycle chunk.

- **Slice 0 of chunk 0**: prologue continuation, unchanged from `-v2`'s
  table above (`load-K/V(K/V, chunk 1)` at 360/520, done by 680).
- **Slices 1–6, and slice 0 of chunks 1–7**: idle, unchanged.
- **Slice 7 of chunks 0–5**: chunk-boundary prefetch for `chunk+2`.
  Worked example, chunk 0 → chunk 2 (slice 7 starts at `160+7×768=5,536`):
  - `load-stationary(K, slice 7)` issues at 5,536, finishes at 5,664 →
    `load-K/V(K, chunk 2)` issues at **5,664**, done 5,824.
  - `load-stationary(V, slice 7)` issues at `5,536+384=5,920`, finishes at
    6,048 → `load-K/V(V, chunk 2)` issues at **6,048**, done 6,208.
  - Chunk 2 needs this at `chunk_start(2)=160+2×6,144=12,448`. **Margin ≈
    12,448−6,208=6,240 cycles** — down from `-v2`'s 8,399, but DMA's own
    need (~320 cycles) is nowhere close to competing at any of the three
    schedules' scales. Generalizes to every `c`→`c+2` (`c`=0..5).
- **Slice 7 of chunks 6 and 7**: no prefetch, unchanged.

**Zero DMA-induced stall confirmed**: `chunk_start(c+1)=chunk_start(c)+6,144`
holds exactly for all `c` — same reasoning as `-v2` (within a chunk, all 8
slices reuse the same already-resident K/V buffer; chunk 0/1 both load in
the prologue with margin ≥5,600 cycles; chunk `c`≥2 has the ≈6,240-cycle
margin above).

## Tail (after chunk 7)

Same rule as `-v2`: `softmax-finalize(head i)` gated by `steady-state-stream-v
(head i)`'s full latency, `store-O(head i)` follows immediately. Chunk 7's
slice 7 starts at `chunk_start(7)+7×768 = 43,168+5,376 = 48,544`. V-phase
starts at `48,544+512=49,056`.

| head | `steady-state-stream-v(i)` finishes | `softmax-finalize(i)` finishes | `store-O(i)` finishes |
|---|---|---|---|
| 0 | 49,215 | 49,247 | 49,252 |
| 1 | 49,247 | 49,279 | 49,284 |
| 2 | 49,279 | 49,311 | 49,316 |
| 3 | 49,311 | 49,343 | 49,348 |
| 4 | 49,343 | 49,375 | 49,380 |
| 5 | 49,375 | 49,407 | 49,412 |
| 6 | 49,407 | 49,439 | 49,444 |
| 7 | 49,439 | 49,471 | **49,476** |

Same SFU-saturation finding as `-v2` still applies here (unaffected by
this fix — it's about `v`'s occupancy matching `finalize`'s own, both
untouched by the shadow register): zero idle SFU cycles through the whole
tail, and the identical pattern in the K-phase's `softmax-update`
sequence above. **Duty cycle number corrected** (audit finding, `notes.md`
§11.29): SFU's active-window fraction is 256/768 = **33%** now, not the
25% carried over from `-v2`'s 256/1,022 — the shorter slice concentrates
the same absolute SFU work into a larger share of it. This is also a real
sensitivity, not just a curiosity: the schedule is only correct because
SFU occupancy (32, `handoff.md`'s own "no real hardware anchor" latency
table entry) is ≤48 — at 32 there is zero absorption anywhere, so any
error in that unanchored number is spent entirely out of S's margin.

## Total Q-tile latency: 49,476 cycles

- vs. `bundle-schedule.md` (179,397): **3.63× faster**.
- vs. `bundle-schedule-v2.md` (65,605, stream-occupancy fix alone):
  **1.33× faster** — the shadow register's real, standalone contribution.
- vs. the theoretical MAC-bound floor (32,768, `notes.md` §11.19): still
  **1.51×** above it. Gap is real and structural, not a modeling gap:
  `load-stationary` still consumes 256 of every 768 matmul-issue-slot
  cycles (33%) since it shares the *same single issue slot* as the
  compute streams — the shadow register fixed the *data* hazard (weight
  overwrite), not the *issue-bandwidth* constraint. Closing that gap
  would need a structurally separate weight-load path, a bigger ISA
  change than what's adopted here.
