# Phase 2 — loop nest being hand-scheduled

Scope: one Q-tile's steady-state body (`notes.md` §9.2, minus the outer
`batch`/`kv_group`/`q_tile` loops — those are identical repeats of this
same body at different addresses, not new scheduling content).

```
load-Q ×8 (once per head)

for chunk in 0..8:
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

for head in 0..8:
  softmax-finalize(head)

store-O ×8 (once per head)
```
