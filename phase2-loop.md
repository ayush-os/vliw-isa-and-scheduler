# Phase 2 — loop nest being hand-scheduled

Scope: one Q-tile's steady-state body (`notes.md` §9.2, minus the outer
`batch`/`kv_group`/`q_tile` loops — those are identical repeats of this
same body at different addresses, not new scheduling content).

Prologue block (`notes.md` §11.2, §11.9, §11.11) replaces the old
standalone `load-Q ×8` line: chunk 0's K/V isn't prefetched by anything
(the in-loop `load-K/V` only prefetches the *next* chunk), and the
accumulator's `m`/`l`/`O` need fresh state every Q-tile since the same 8
physical per-head locations are reused across all 65,536 Q-tile
instances.

DMA-slot order matters here, unlike a first pass might suggest:
`load-K/V(K, chunk 0)` has to come first (gates the first
`load-stationary`), and `load-Q ×8` has to come before
`load-K/V(V, chunk 0)`, not after — `Q(head 0)` has the tightest deadline
in the whole prologue (needed the moment `load-stationary(K, slice 0)`
finishes, at cycle 287.844), while `V` has by far the loosest (not needed
until slice 0's whole K-phase completes, ~1,560 cycles in). Putting `V`
last costs nothing and removes a real ~37-cycle stall on `Q(head 0)` that
a `K → V → Q` ordering would otherwise cause (`notes.md` §11.11).
`metadata-init ×8` is SFU-slot and independent of all of this — it
finishes all 8 heads well before anything needs it regardless of the DMA
ordering above, so it's left as its own list rather than paired/
interleaved with `load-Q`.

Epilogue (`notes.md` §11.2, resolved below): the trailing prefetch at the
bottom of the chunk loop targets `chunk + 1`, which doesn't exist for the
last iteration (`chunk`=7 — chunks are 0..7, 8 total). Same discipline as
the prologue: this is a literal, compile-time-unrolled structural
difference in the last iteration's body, not a runtime guard (this ISA
has zero branch instructions), so chunk 7 is peeled out of the loop below
rather than written as an `if chunk < 7` conditional.

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
