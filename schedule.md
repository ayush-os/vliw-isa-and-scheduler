# Hand-Scheduled Bundle Sequence

Cycle-accurate hand schedule for one Q-tile's steady-state body, targeting
[`isa.md`](isa.md). **33,220 cycles — 1.4% above the theoretical
32,768-cycle MAC-bound floor**, with every hazard margin derived and
checked, not assumed safe by construction.

## Loop nest

One Q-tile's steady-state body (the outer `batch`/`kv_group`/`q_tile`
loops are identical repeats of this same body at different addresses —
not new scheduling content, so excluded here).

DMA-slot order matters: `load-K/V(K, chunk 0)` has to come first (gates
the first `load-stationary`), and `load-Q ×8` has to come before
`load-K/V(V, chunk 0)`, not after — `Q(head 0)` has by far the tightest
deadline in the prologue (needed the moment `load-stationary(K, slice 0)`
finishes), while `V` has by far the loosest (not needed until slice 0's
whole K-phase completes). Putting `V` last costs nothing and avoids a real
stall on `Q(head 0)` that a `K → V → Q` ordering would otherwise cause.
`metadata-init ×8` is SFU-slot and independent of the DMA ordering — it
finishes all 8 heads well before anything needs it regardless.

Chunk 7 (the last chunk) is peeled out of the loop rather than written as
a runtime `if chunk < 7` guard — this ISA has zero branch instructions,
so the trailing prefetch that targets `chunk + 1` genuinely doesn't exist
for the last iteration and has to be a literal, compile-time-unrolled
structural difference, not a conditional.

```
load-K/V(K, chunk 0)
load-Q ×8 (once per head)
load-K/V(V, chunk 0)

metadata-init ×8 (once per head, SFU — independent of the DMA order above)

for chunk in 0..7:
  for slice in 0..8:
    load-stationary(K, slice)
    for head in 0..8:
      steady-state-stream-qk(head)
      softmax-update(head)

    load-stationary(V, slice)
    for head in 0..8:
      steady-state-stream-v(head)

  load-K/V(K, next chunk)
  load-K/V(V, next chunk)

# chunk 7 (last chunk) — same slice loop, no trailing prefetch
for slice in 0..8:
  load-stationary(K, slice)
  for head in 0..8:
    steady-state-stream-qk(head)
    softmax-update(head)

  load-stationary(V, slice)
  for head in 0..8:
    steady-state-stream-v(head)

for head in 0..8:
  softmax-finalize(head)

store-O ×8 (once per head)
```

## Resource model

One instruction issues per cycle per slot (4 slots, each in-order).
**Occupancy ≠ latency for the two streaming instructions** — this is the
single most load-bearing modeling decision in the whole schedule:

| Instruction | Slot | Latency | Occupancy | Why occupancy < latency |
|---|---|---|---|---|
| `steady-state-stream-qk`/`-v` | matmul-issue | 159 (`N+D−1`: 32-cycle feed + 128-cycle drain, minus the shared boundary cycle) | **32** (`N`, the feed count) | The next head's feed can start the cycle after the current head's own 32-cycle feed completes — it doesn't need to wait for the 128-cycle drain tail, since the drain is pipelined *behind* the next head's feed, not blocking it. A naive model that conflates the two (busy for the full 159-cycle latency) understates real throughput by ~5×. |
| `load-stationary` | weight-load | 128 | 128 | No feed/drain split — every cycle loads a distinct new row into the array. |
| `softmax-update`/`-finalize`/`metadata-init` | SFU | 32 | 32 | Occupancy=latency here is a real modeling choice (cheap given SFU's slack, avoids OOO-style completion tracking), not a physical claim. |
| `load-K/V` | DMA | 160 | 160 | Not a systolic pipeline — no feed/drain structure. |
| `load-Q`/`store-O` | DMA | 5 | 5 | Same. |

**Two mechanisms, both confirmed against real Gemmini RTL rather than
assumed, are what let `load-stationary` avoid becoming the bottleneck**:

1. **A shadow register.** `PE.scala` gives every PE two registers for the
   stationary value (`c1`/`c2`), gated by a `propagate` control signal —
   real, hardware-level double-buffered stationary storage. This lets a
   new `load-stationary` write into the *inactive* register while the
   *active* one is still being streamed against, rather than needing to
   wait for the active stream to fully drain first.
2. **A genuinely separate issue path.** `MeshWithDelays` feeds the
   stationary (`B`) and moving (`A`/`D`) operands through independent
   physical ports every cycle, and `ExecuteController.scala`'s
   `perform_mul_pre` shows real preload/compute overlap — not just
   register-level double-buffering, but instruction-level concurrency.
   This is why `load-stationary` gets its own bundle slot (`isa.md` §2)
   instead of sharing matmul-issue's opcode field: a single shared field
   can't express "issue a stream and advance a weight-load in the same
   cycle," even though the hardware supports exactly that.

## Hazards

Every shared resource, checked directly rather than assumed safe:

- **S** (raw accumulator, ×4 buffered, `head_idx & 3` selects buffer):
  intra-K-phase only (head `i` vs. `i+4`) — `steady-state-stream-qk(head
  i+4)`'s write must not start before `softmax-update(head i)` finishes
  reading. **Margin: 65 cycles.**
- **P** (scratchpad, ×8 per-head, WAR across slices): a source-read
  hazard, not a destination-write one — `steady-state-stream-v` reads P
  as its moving operand, so "finishes reading" resolves at its
  occupancy-end (32 cycles), not its full 159-cycle latency, unlike S/O
  below. **Margin: 383 cycles.**
- **O** (accumulator, ×8 per-head, rescale-then-add ordering):
  `softmax-update(head i)` must complete before `steady-state-stream-v
  (head i)` starts, same head only. **Margin: 65 cycles.**
- **Array stationary registers — the tightest real constraint in the
  schedule.** `load-stationary` writing register X must wait for X's
  *last reader* — the same-type stream instruction, two phases back — to
  fully drain (full latency, not occupancy) before writing. Worked
  cycle-by-cycle: the last reader's occupancy ends exactly at its phase
  boundary by construction, but its true drain trails that boundary by
  `latency−occupancy=127` cycles; the new load must then finish its own
  128-cycle occupancy before the *next* phase needs the register, 256
  cycles later. Legal window: exactly **[127, 128]** — a genuine 1-cycle
  margin, symmetric in both directions (K-during-V-phase,
  V-during-K-phase). This is load-bearing for the final cycle count, not
  incidental — trusted as derived rather than padded with defensive
  slack.
- **Metadata** (`m`,`l`, ×8 per-head): ordinary in-order RAW chain,
  automatically satisfied by program order on the in-order SFU slot.
- **DMA double-buffering** (K1/K2/V1/V2): a `load-K/V` targeting a given
  buffer instance must wait for that instance's last reader
  (`load-stationary` for the chunk currently occupying it) to finish.
  Margins in the thousands of cycles throughout.

## Prologue

Cycle 0–288. Nothing here depends on `load-stationary`'s slot placement,
since nothing else could use matmul-issue during this window anyway
(`qk` can't start until K is stationary).

| cycle | matmul-issue | weight-load | SFU | DMA |
|---|---|---|---|---|
| 0 | idle | idle | `metadata-init(head 0)` | `load-K/V(K, chunk 0)` |
| 32 | idle | idle | `metadata-init(head 1)` | ⋯ busy til 160 |
| 64 | idle | idle | `metadata-init(head 2)` | ⋯ busy til 160 |
| 96 | idle | idle | `metadata-init(head 3)` | ⋯ busy til 160 |
| 128 | idle | idle | `metadata-init(head 4)` | ⋯ busy til 160 |
| 160 | idle | `load-stationary(K, slice 0)` | `metadata-init(head 5)` | `load-Q(head 0)` |
| 165 | idle | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 1)` |
| 170 | idle | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 2)` |
| 175 | idle | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 3)` |
| 180 | idle | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 4)` |
| 185 | idle | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 5)` |
| 190 | idle | ⋯ busy til 288 | ⋯ busy til 192 | `load-Q(head 6)` |
| 192 | idle | ⋯ busy til 288 | `metadata-init(head 6)` | ⋯ busy til 195 |
| 195 | idle | ⋯ busy til 288 | ⋯ busy til 224 | `load-Q(head 7)` |
| 200 | idle | ⋯ busy til 288 | ⋯ busy til 224 | `load-K/V(V, chunk 0)` |
| 224 | idle | ⋯ busy til 288 | `metadata-init(head 7)` | ⋯ busy til 360 |

## Slice 0

One genuine special case in the whole schedule: `load-stationary(V, slice
0)` has no prior register-occupant to wait for (first use of that
register this Q-tile), so it's gated by **DMA readiness** (`load-K/V(V,
chunk 0)` done at 360) instead of the register hazard. Every other
transition, and every subsequent slice, uses the register-hazard gate.

| cycle | matmul-issue | weight-load | SFU | DMA |
|---|---|---|---|---|
| 288 | `steady-state-stream-qk(head 0)` | idle | idle | ⋯ `load-K/V(V, chunk 0)`, busy til 360 |
| 320 | `qk(head 1)` | idle | idle | ⋯ busy til 360 |
| 352 | `qk(head 2)`, busy til 384 | idle | idle | ⋯ busy til 360 |
| 360 | ⋯ busy til 384 | `load-stationary(V, slice 0)` fires | idle | `load-K/V(K, chunk 1)` |
| 384 | `qk(head 3)` | ⋯ busy til 488 | idle | ⋯ busy til 520 |
| 416 | `qk(head 4)` | ⋯ busy til 488 | idle | ⋯ busy til 520 |
| 447 | ⋯ `qk(head 4)`, busy til 448 | ⋯ busy til 488 | `softmax-update(head 0)` | ⋯ busy til 520 |
| 448 | `qk(head 5)` | ⋯ busy til 488 | ⋯ `su(head 0)`, busy til 479 | ⋯ busy til 520 |
| 479 | ⋯ `qk(head 5)`, busy til 480 | ⋯ busy til 488 | `softmax-update(head 1)` | ⋯ busy til 520 |
| 480 | `qk(head 6)` | ⋯ busy til 488 | ⋯ `su(head 1)`, busy til 511 | ⋯ busy til 520 |
| 488 | ⋯ `qk(head 6)`, busy til 512 | idle (done) | ⋯ busy til 511 | ⋯ busy til 520 |
| 511 | ⋯ busy til 512 | idle | `softmax-update(head 2)` | ⋯ busy til 520 |
| 512 | `qk(head 7)`, busy til 544 | idle | ⋯ `su(head 2)`, busy til 543 | ⋯ busy til 520 |
| 520 | ⋯ busy til 544 | idle | ⋯ busy til 543 | `load-K/V(V, chunk 1)` |
| 543 | ⋯ busy til 544 | idle | `softmax-update(head 3)` | ⋯ busy til 680 |
| 544 | `steady-state-stream-v(head 0)` | idle | ⋯ `su(head 3)`, busy til 575 | ⋯ busy til 680 |
| 575 | ⋯ `v(head 0)`, busy til 576 | idle | `softmax-update(head 4)` | ⋯ busy til 680 |
| 576 | `v(head 1)` | idle | ⋯ `su(head 4)`, busy til 607 | ⋯ busy til 680 |
| 607 | ⋯ `v(head 1)`, busy til 608 | idle | `softmax-update(head 5)` | ⋯ busy til 680 |
| 608 | `v(head 2)` | idle | ⋯ `su(head 5)`, busy til 639 | ⋯ busy til 680 |
| 639 | ⋯ `v(head 2)`, busy til 640 | idle | `softmax-update(head 6)` | ⋯ busy til 680 |
| 640 | `v(head 3)` | idle | ⋯ `su(head 6)`, busy til 671 | ⋯ busy til 680 |
| 671 | ⋯ `v(head 3)`, busy til 672 | `load-stationary(K, slice 1)` fires | `softmax-update(head 7)` | ⋯ busy til 680 |
| 672 | `v(head 4)` | ⋯ busy til 799 | ⋯ `su(head 7)`, busy til 703 | ⋯ busy til 680 |
| 680 | ⋯ `v(head 4)`, busy til 704 | ⋯ busy til 799 | idle | idle |
| 704 | `v(head 5)` | ⋯ busy til 799 | idle | idle |
| 736 | `v(head 6)` | ⋯ busy til 799 | idle | idle |
| 768 | `v(head 7)`, busy til 800 | ⋯ busy til 799 | idle | idle |
| 799 | ⋯ busy til 800 | idle (done) | idle | idle |
| 800 | slice 1: `qk(head 0)` | idle | idle | idle |

Slice 0's matmul-issue span is 288→800 = exactly **512 cycles**, identical
to every steady-state slice — no lingering special-casing beyond the
V-load's one-time looser gate.

## Universal per-slice table

Every steady-state slice follows this pattern exactly (Δ relative to the
slice's own qk-phase start).

| Δ | matmul-issue | weight-load | SFU |
|---|---|---|---|
| 0 | `steady-state-stream-qk(head 0)` | idle | idle |
| 32 | `qk(head 1)` | idle | idle |
| 64 | `qk(head 2)` | idle | idle |
| 96 | `qk(head 3)`, busy til 128 | idle | idle |
| 127 | ⋯ | `load-stationary(V, slice N)` fires | idle |
| 128 | `qk(head 4)` | ⋯ busy til 255 | idle |
| 159 | ⋯ | ⋯ | `softmax-update(head 0)` |
| 160 | `qk(head 5)` | ⋯ | idle |
| 191 | ⋯ | ⋯ | `softmax-update(head 1)` |
| 192 | `qk(head 6)` | ⋯ | idle |
| 223 | ⋯ | ⋯ | `softmax-update(head 2)` |
| 224 | `qk(head 7)`, busy til 256 | ⋯ | idle |
| 255 | ⋯ | ⋯ done at 255 | `softmax-update(head 3)` |
| 256 | `steady-state-stream-v(head 0)` | idle | idle |
| 287 | ⋯ | idle | `softmax-update(head 4)` |
| 288 | `v(head 1)` | idle | idle |
| 319 | ⋯ | idle | `softmax-update(head 5)` |
| 320 | `v(head 2)` | idle | idle |
| 351 | ⋯ | idle | `softmax-update(head 6)` |
| 352 | `v(head 3)` | idle | idle |
| 383 | ⋯ | `load-stationary(K, slice N+1)` fires | `softmax-update(head 7)` |
| 384 | `v(head 4)` | ⋯ busy til 511 | idle |
| 416 | `v(head 5)` | ⋯ | idle |
| 448 | `v(head 6)` | ⋯ | idle |
| 480 | `v(head 7)`, busy til 512 | ⋯ done at 511 | idle |

Slice ends at Δ=512 (`load-stationary(K, slice N+1)` finishes with 1
cycle to spare before `steady-state-stream-qk(head 0)` of the next slice
needs it).

`slice_start(c,s) = chunk_start(c) + s×512`, `chunk_start(c) = 288 +
c×4,096`.

## DMA overlay / chunk boundaries

DMA is in-order, nothing else competes for it, and it fires the instant
its WAR gate clears — the gate is `load-stationary`'s own completion
(which fires mid-phase, not at a slice boundary), not a stream's full
latency.

- **Chunk 1**: no real WAR hazard at all — K2/V2 buffers are fresh, never
  used, so DMA just fires back-to-back the moment the slot's free:
  `load-K/V(K, chunk 1)` at 360→520, `load-K/V(V, chunk 1)` at 520→680.
  Needed at `chunk_start(1)=4,384` → margin **3,704 cycles**.
- **Chunk `c`≥2** (worked example, chunk 0→2, slice 7 of chunk 0 starts
  at `288+7×512=3,872`):
  - `load-stationary(K, slice 7)` fires during slice 6's *v*-phase (not
    slice 7's start) at `3,360+256+127=3,743`, finishes reading chunk 0's
    K-buffer at 3,871 → `load-K/V(K, chunk 2)` issues at **3,871**, done
    4,031.
  - `load-stationary(V, slice 7)` fires during slice 7's own *qk*-phase at
    `3,872+127=3,999`, finishes at 4,127 → `load-K/V(V, chunk 2)` issues
    at **4,127**, done 4,287.
  - Chunk 2 needed at `chunk_start(2)=8,480` → margin **4,193 cycles**.
    Generalizes to every `c`→`c+2` (`c`=0..5).
- **Slice 7 of chunks 6 and 7**: no prefetch (no chunk 8/9).

`chunk_start(c+1)=chunk_start(c)+4,096` holds exactly for all `c` —
matmul-issue never waits on DMA anywhere in the schedule.

## Tail

`softmax-finalize(head i)` gated by `steady-state-stream-v(head i)`'s
full latency (needs all of O's rows written, which only happens at full
drain), `store-O(head i)` follows immediately. `v`'s occupancy (32)
exactly equals `finalize`'s own occupancy (32), so every `finalize(head
i+1)` starts the instant `finalize(head i)` ends — SFU runs the entire
tail at zero idle cycles.

Chunk 7's slice 7 starts at `chunk_start(7)+7×512 = 28,960+3,584 =
32,544`. V-phase starts at 32,800.

| head | `steady-state-stream-v(i)` finishes | `softmax-finalize(i)` finishes | `store-O(i)` finishes |
|---|---|---|---|
| 0 | 32,959 | 32,991 | 32,996 |
| 1 | 32,991 | 33,023 | 33,028 |
| 2 | 33,023 | 33,055 | 33,060 |
| 3 | 33,055 | 33,087 | 33,092 |
| 4 | 33,087 | 33,119 | 33,124 |
| 5 | 33,119 | 33,151 | 33,156 |
| 6 | 33,151 | 33,183 | 33,188 |
| 7 | 33,183 | 33,215 | **33,220** |

## Total: 33,220 cycles

**1.4% above the theoretical 32,768-cycle MAC-bound floor.** The entire
gap is prologue (288 cycles) + tail drain (164 cycles) — both structural:
the very first `load-stationary` has no preceding phase to hide inside
(nothing is running yet), and the final `softmax-finalize`/`store-O`
drain has no following phase to hide inside either (nothing is left to
run). Every one of the 64 steady-state slices in between is already
exactly at the 512-cycle MAC-bound rate, zero slack.

Worth being precise about what this claim is and isn't: "no avoidable
stalls found," not "proven optimal" — the two techniques that could
theoretically absorb the remaining 452-cycle gap (cross-iteration
software pipelining across Q-tiles) were evaluated on real numbers
(≈0.655% of total workload runtime) and deliberately not pursued, not
overlooked.
