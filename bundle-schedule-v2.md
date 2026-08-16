# Bundle schedule v2 — corrected occupancy model (Phase 2 rebuild)

> **SUPERSEDED — see [`bundle-schedule-v3.md`](bundle-schedule-v3.md) for
> the current schedule.** This file (65,605 cycles) fixed the
> stream-occupancy bug but still fully serializes `load-stationary` ahead
> of its slice's streams. Adopting a `load-stationary` shadow register
> (confirmed real in Gemmini's actual RTL — `PE.scala`'s `c1`/`c2`
> double-buffered stationary registers, `notes.md` §11.26) removes the
> 127-cycle idle bubble this file has at every K↔V transition.
> `bundle-schedule-v3.md` has the rebuild: **49,476 cycles** (1.33×
> faster than this file). Preserved here as the before/after baseline for
> that specific fix — do not build the automated scheduler against these
> numbers, use `-v3`.

Rebuild of `bundle-schedule.md` under the corrected occupancy model
(`notes.md` §11.19–§11.21). That file (179,397 cycles) is preserved as
the answer to the old, wrong model — this file is the current one.

**Model rules in effect here** (`notes.md` §11.20–§11.22):
- `steady-state-stream-qk`/`-v`: latency 159 (`N+D−1`, unchanged), occupancy
  **32** (`N`, the feed count) — next same-buffer stream can issue 32
  cycles after the previous one, not 159.
- `load-stationary`: latency = occupancy = 128, but **gated by the prior
  stream's full 159-cycle latency, not its 32-cycle occupancy** — no
  shadow register (punted, `notes.md` §11.20), so the array's weight
  registers can't be overwritten until the prior stream's drain is
  genuinely complete. This produces a real idle bubble on matmul-issue at
  every K-phase→V-phase and V-phase→K-phase transition — see the note
  after the Slice 0 table.
- SFU (`softmax-update`/`-finalize`/`metadata-init`) and DMA: latency =
  occupancy, unchanged from before — untouched by this fix.
- S: **×4** buffering (was ×2), `head_idx & 3` selects the buffer —
  ×2 left only ~1 cycle of margin under the corrected model, ×4 gives 65
  (`notes.md` §11.21).
- P/O: confirmed safe under the corrected model with real margin (~638
  cycles for P's WAR hazard), no buffering change needed (`notes.md`
  §11.22).

**Status: complete.** Prologue, slice 0, universal table, DMA overlay,
chunk-boundary, and tail all derived below. **Total Q-tile latency:
65,605 cycles** — 2.73× faster than `bundle-schedule.md`'s 179,397
(stream-occupancy fix alone; `load-stationary` still fully serialized, no
shadow register). Full narrative derivation: `notes.md` §11.19–§11.24.

## Prologue

Identical to `bundle-schedule.md`'s prologue through cycle 288 — nothing
here depends on stream occupancy, only on DMA (`load-K/V`, `load-Q`) and
SFU (`metadata-init`) timing, both untouched by the fix.

| cycle | matmul-issue | SFU | DMA |
|---|---|---|---|
| 0 | idle | `metadata-init(head 0)` | `load-K/V(K, chunk 0)` |
| 32 | idle | `metadata-init(head 1)` | ⋯ `load-K/V(K, chunk 0)`, busy til 160 |
| 64 | idle | `metadata-init(head 2)` | ⋯ busy til 160 |
| 96 | idle | `metadata-init(head 3)` | ⋯ busy til 160 |
| 128 | idle | `metadata-init(head 4)` | ⋯ busy til 160 |
| 160 | `load-stationary(K, slice 0)` | `metadata-init(head 5)` | `load-Q(head 0)` |
| 165 | ⋯ `load-stationary`, busy til 288 | ⋯ `metadata-init(head 5)`, busy til 192 | `load-Q(head 1)` |
| 170 | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 2)` |
| 175 | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 3)` |
| 180 | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 4)` |
| 185 | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 5)` |
| 190 | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 6)` |
| 192 | ⋯ busy til 288 | `metadata-init(head 6)` | ⋯ `load-Q(head 6)`, busy til 195 |
| 195 | ⋯ busy til 288 | ⋯ busy til 224 | `load-Q(head 7)` |
| 200 | ⋯ busy til 288 | ⋯ busy til 224 | `load-K/V(V, chunk 0)` |
| 224 | ⋯ busy til 288 | `metadata-init(head 7)` | ⋯ busy til 360 |

## Slice 0

Diverges from `bundle-schedule.md` starting at the row after 288 — old
model had `qk(head 1)` at 447 (159-cycle spacing), this model has it at
320 (32-cycle spacing).

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
| 543 | **idle** (see note) | `softmax-update(head 3)` | ⋯ `V(chunk 1)`, busy til 680 |
| 575 | idle | `softmax-update(head 4)` | ⋯ busy til 680 |
| 607 | idle | `softmax-update(head 5)` | ⋯ busy til 680 |
| 639 | idle | `softmax-update(head 6)` | ⋯ busy til 680 |
| 671 | `load-stationary(V, slice 0)` | `softmax-update(head 7)` | ⋯ busy til 680 |
| 680 | ⋯ `load-stationary`, busy til 799 | ⋯ `su(head 7)`, busy til 703 | idle |
| 703 | ⋯ busy til 799 | idle | idle |
| 799 | `steady-state-stream-v(head 0)` | idle | idle |
| 831 | `steady-state-stream-v(head 1)` | idle | idle |
| 863 | `steady-state-stream-v(head 2)` | idle | idle |
| 895 | `steady-state-stream-v(head 3)` | idle | idle |
| 927 | `steady-state-stream-v(head 4)` | idle | idle |
| 959 | `steady-state-stream-v(head 5)` | idle | idle |
| 991 | `steady-state-stream-v(head 6)` | idle | idle |
| 1023 | `steady-state-stream-v(head 7)` | idle | idle |

Slice ends at 1,182 (`v(head 7)`'s full-latency completion, 1023+159 —
also the moment slice 1's `load-stationary(K)` can issue). **Matmul-issue
span 160 → 1,182 = 1,022 cycles/slice**, down from the old model's 2,800.

**Real finding worth flagging, not glossed over**: matmul-issue is *not*
bubble-free even under the corrected model. There's a genuine 127-cycle
idle gap at 544→671 (after `qk(head 7)`'s occupancy frees the slot at
544, before its full latency lets `load-stationary(V)` actually issue at
671) — and a symmetric one at the K↔V transition into the next slice
(1055→1182). That's 254 idle cycles out of every 1,022-cycle slice
(**~25% idle**, not the ~0% the "confirmed 100% utilization" framing from
§11.19 might suggest). This is exactly the gap the punted shadow-register
idea would eliminate — it's not a new hazard, just the concrete cost of
having punted that idea. Worth noting: this makes the real "stream-fix-alone"
total meaningfully higher than the audit's rough ~49,000-cycle estimate —
64 slices × 1,022 ≈ 65,400 cycles just from the steady-state portion,
before prologue/tail, which suggests the audit's estimate assumed
load-stationary's transition cost away rather than actually accounting for
this gating rule.

## Universal matmul-issue/SFU table (every slice)

Δ relative to each slice's own `load-stationary(K)` issue (`slice_start`).

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
| 383 | idle | `softmax-update(head 3)` |
| 415 | idle | `softmax-update(head 4)` |
| 447 | idle | `softmax-update(head 5)` |
| 479 | idle | `softmax-update(head 6)` |
| 511 | `load-stationary(V, slice N)` | `softmax-update(head 7)` |
| 639 | `steady-state-stream-v(head 0)` | idle |
| 671 | `steady-state-stream-v(head 1)` | idle |
| 703 | `steady-state-stream-v(head 2)` | idle |
| 735 | `steady-state-stream-v(head 3)` | idle |
| 767 | `steady-state-stream-v(head 4)` | idle |
| 799 | `steady-state-stream-v(head 5)` | idle |
| 831 | `steady-state-stream-v(head 6)` | idle |
| 863 | `steady-state-stream-v(head 7)` | idle |

Slice ends at Δ=1,022.

`slice_start` values, chunk 0: `160 + s×1,022` for `s`=0..7. Generalizes
to `chunk_start(c) + s×1,022` where `chunk_start(c) = 160 + c×8,176` —
confirmed zero-stall below.

## DMA overlay (per-slice, the only column that varies)

Same four structural cases as `bundle-schedule.md`, same reasoning (DMA is
in-order with nothing else competing, so each prefetch fires the instant
its own WAR-hazard gate clears — not tied to program-order position). Only
the literal cycle numbers change, since they're downstream of the new
1,022-cycle slice / 8,176-cycle chunk length.

- **Slice 0 of chunk 0**: prologue continuation — see the Slice 0 table
  above (`load-K/V(K/V, chunk 1)` at 360/520, both done by 680 — comfortably
  inside chunk 0's ~8,176-cycle span).
- **Slices 1–6 of every chunk, and slice 0 of chunks 1–7**: genuinely idle,
  same as before — nothing else ever queues on DMA in between.
- **Slice 7 of chunks 0–5**: chunk-boundary prefetch for `chunk+2`.
  Chunk `c`'s slice 7 starts at `chunk_start(c) + 7×1,022`, where
  `chunk_start(c) = 160 + c×8,176`. Worked example, chunk 0 → chunk 2
  (`chunk_start(0)=160`, slice 7 starts at 7,314):
  - `load-stationary(K, slice 7)` issues at 7,314, finishes (last K-buffer
    read) at 7,442 → `load-K/V(K, chunk 2)` issues at **7,442**, done 7,602.
  - `load-stationary(V, slice 7)` issues at `7,314+511=7,825`, finishes at
    7,953 → `load-K/V(V, chunk 2)` issues at **7,953**, done 8,113.
  - Both finish inside chunk 0's own slice 7 window (`[7,314, 8,336)`).
  - Chunk 2 needs this data at `chunk_start(2)=16,512`. **Margin ≈ 16,512 −
    8,113 = 8,399 cycles** — down from the old model's ~23,500, but still
    nowhere close to tight. Generalizes to every `c`→`c+2` (`c`=0..5) by the
    same translation-invariant argument as `bundle-schedule.md`.
- **Slice 7 of chunks 6 and 7**: no prefetch (no chunk 8/9), same as before.

## Chunk-boundary: zero DMA-induced stall, confirmed

`chunk_start(c+1) = chunk_start(c) + 8,176` holds exactly, for every
`c`=0..7 — matmul-issue never waits on DMA anywhere in the Q-tile. Reason:
within a chunk, all 8 slices read 128-wide sub-slices of the *same
already-resident* K/V chunk buffer (loaded once via `load-K/V`), so
`load-stationary` timing is self-contained and never touches DMA. The only
place DMA *could* stall matmul-issue is at a chunk transition if its
prefetch weren't ready in time — checked for all three cases and never
close:
- Chunk 0: loaded directly in the prologue, gates `load-stationary(K,
  slice 0)` exactly (that's the prologue's own structure, unaffected by
  this fix).
- Chunk 1: loaded in the prologue, done by 680; needed at
  `chunk_start(1)=8,336` → margin 7,656.
- Chunk `c`≥2: loaded during chunk `c-2`'s slice 7 → margin ≈8,399 (the
  DMA overlay computation above, which *is* this confirmation, just stated
  generally here).

## Tail (after chunk 7)

`softmax-finalize(head i)` is gated by `steady-state-stream-v(head i)`'s
*full* latency completion (same reasoning as `softmax-update` needing
`qk`'s full S-write — `finalize` needs all of O's rows written, which
only happens at full latency), not by `softmax-update`. `store-O(head i)`
follows `softmax-finalize(head i)` immediately.

Chunk 7's slice 7 starts at `chunk_start(7) + 7×1,022 = 57,392+7,154 =
64,546`. V-phase starts at `64,546+639 = 65,185`.

| head | `steady-state-stream-v(i)` finishes | `softmax-finalize(i)` finishes | `store-O(i)` finishes |
|---|---|---|---|
| 0 | 65,344 | 65,376 | 65,381 |
| 1 | 65,376 | 65,408 | 65,413 |
| 2 | 65,408 | 65,440 | 65,445 |
| 3 | 65,440 | 65,472 | 65,477 |
| 4 | 65,472 | 65,504 | 65,509 |
| 5 | 65,504 | 65,536 | 65,541 |
| 6 | 65,536 | 65,568 | 65,573 |
| 7 | 65,568 | 65,600 | **65,605** |

**Real finding, not just smaller numbers than before**: `v`'s per-head
occupancy (32) now exactly equals `softmax-finalize`'s own occupancy
(32), so every `v(head i)` finish lands exactly on the previous
`finalize(head i-1)`'s finish — SFU runs the entire tail at **zero idle
cycles**, versus `bundle-schedule.md`'s huge slack there (159-cycle `v`
cadence vs. 37-cycle `finalize`+`store-O`). Still hazard-free (32=32 is an
exact match, not a shortfall), but resting on a coincidence rather than
margin — same category of concern as S's 1-cycle case before the ×4 fix.
`store-O`/DMA keeps real slack (~27 idle cycles between consecutive
`store-O`s); this exact-saturation effect is SFU-only. It also isn't
tail-specific — the identical back-to-back pattern already shows up in
the K-phase's `softmax-update` sequence in the Slice 0 table above
(447→479→511→543→575→607→639→671→703, fully contiguous). SFU flips from
"~55% idle, comfortable slack" (original Phase 1 characterization) to
**100% utilized during every active window**, though its overall duty
cycle per slice is still only ~25% (idle during `load-stationary` and all
of V-phase).

## Total Q-tile latency: 65,605 cycles

Down from `bundle-schedule.md`'s 179,397 — **2.73× faster**, from the
stream-occupancy fix alone (`load-stationary` still fully serialized
ahead of its slice's streams, no shadow register). Full narrative:
`notes.md` §11.19–§11.24.
