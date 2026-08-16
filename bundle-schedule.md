# Bundle schedule — hand-scheduled (Phase 2 deliverable)

> **SUPERSEDED — read `notes.md` §11.19 before trusting any number below.**
> This entire schedule (179,397 cycles) was built on a slot-occupancy
> model (busy for an instruction's *full* latency, including a systolic
> stream's drain time) that an independent audit found — and
> `prefill_notes.md` §3.1's own Timeloop results confirm exactly — is
> wrong: a real 128×128 array can sustain much higher throughput than
> this model assumed (Timeloop measured 100% utilization on the identical
> array; this schedule implies ~18%). Decision made (§11.19): fix the
> occupancy model and re-derive this file from scratch *before* building
> the automated scheduler. Everything below is preserved as the correct
> answer to the *old* model — useful for the eventual before/after
> comparison, not the current truth. **Do not build the automated
> scheduler against these numbers.**

The hand-scheduled bundle sequence `spec.md`'s Phase 2 requires, for the
loop nest in `phase2-loop.md` against the ISA in `isa.md`. Derivation,
every correction, and the reasoning behind the format are in `notes.md`
§11 (§11.13 onward for the bundle-table work specifically) — this file is
the clean, literal table only.

**Scope**: representative segments, not a literal cycle-by-cycle table
across the whole ~179k-cycle Q-tile (almost all of that is idle bundles
between real issue points). Structure, revised once the generic-slice work
below made it clear the original five-way split was slightly off: the
**matmul-issue/SFU pattern is identical for every slice** (prologue,
steady-state, chunk-boundary — all of it), so that's one universal table
below, parameterized by `slice_start`. **DMA is the only column that
actually differs slice-by-slice**, tracked as a separate overlay per
distinct case (prologue, chunk-boundary prefetch) rather than baked into
per-slice copies of the whole table. Prologue and slice 0 are kept as
their own fully-merged tables below since they were derived and written
before this reframing and the merged form is still the clearest way to see
the actual bundle contents for those two; everything after reuses the
universal table + DMA overlay.

**Format**: merged across all three slots (matmul-issue / SFU / DMA), one
row per issue event — a cycle where ≥1 slot dispatches a *new*
instruction. Each cell is one of three states:
- a new dispatch this cycle — mnemonic + operands,
- still busy with something dispatched earlier — `⋯ <mnemonic>, busy til
  <cycle>`,
- genuinely idle — `idle`.

Mnemonic + named operands, not raw 33-bit encodings — the encoding is a
mechanical last step once the schedule is right. "Idle" describes that one
cycle only, not the whole gap since the previous row.

Latencies used throughout (ceiling'd — see `notes.md` §11.4's correction):
`load-stationary`=128, `steady-state-stream-qk`/`-v`=159,
`softmax-update`/`softmax-finalize`/`metadata-init`=32, `load-K/V`=160,
`load-Q`/`store-O`=5.

## Prologue

`load-K/V(K, chunk 0)` → `load-Q ×8` → `load-K/V(V, chunk 0)` on DMA
(deadline-ordered, `notes.md` §11.11); `metadata-init ×8` independent on
SFU. Ends at the handoff into slice 0's K-phase (`steady-state-stream-qk
(head 1)` / `softmax-update(head 0)` co-issuing at cycle 447 confirms the
S double-buffer fix is doing its job — `qk(head 1)` doesn't wait on
`softmax-update(head 0)` at all).

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
| 288 | `steady-state-stream-qk(head 0)` | idle | ⋯ busy til 360 |
| 447 | `steady-state-stream-qk(head 1)` | `softmax-update(head 0)` | idle |

## Slice 0

K-phase pipelines `steady-state-stream-qk(head i+1)` against
`softmax-update(head i)` with zero stall (the S double-buffer payoff,
§11.8). DMA, independently, doesn't sit idle after the prologue — it's
in-order with zero competing instructions until deep into chunk 0, so
`load-K/V(K/V, chunk 1)` issues the moment `V(chunk 0)` frees the DMA slot
(cycle 360), regardless of where those lines sit textually
(`notes.md` §11.12's chunk-boundary finding). `load-stationary(V, slice 0)`
issues the moment `qk(head 7)` frees the matmul-issue slot (1,560) — forced
by slot serialization, no dependency on `softmax-update(head 7)`.

| cycle | matmul-issue | SFU | DMA |
|---|---|---|---|
| 288 | `steady-state-stream-qk(head 0)` | idle | ⋯ `load-K/V(V, chunk 0)`, busy til 360 |
| 360 | ⋯ `qk(head 0)`, busy til 447 | idle | `load-K/V(K, chunk 1)` |
| 447 | `steady-state-stream-qk(head 1)` | `softmax-update(head 0)` | ⋯ `load-K/V(K, chunk 1)`, busy til 520 |
| 520 | ⋯ `qk(head 1)`, busy til 606 | idle | `load-K/V(V, chunk 1)` |
| 606 | `steady-state-stream-qk(head 2)` | `softmax-update(head 1)` | ⋯ `load-K/V(V, chunk 1)`, busy til 680 |
| 765 | `steady-state-stream-qk(head 3)` | `softmax-update(head 2)` | idle |
| 924 | `steady-state-stream-qk(head 4)` | `softmax-update(head 3)` | idle |
| 1083 | `steady-state-stream-qk(head 5)` | `softmax-update(head 4)` | idle |
| 1242 | `steady-state-stream-qk(head 6)` | `softmax-update(head 5)` | idle |
| 1401 | `steady-state-stream-qk(head 7)` | `softmax-update(head 6)` | idle |
| 1560 | `load-stationary(V, slice 0)` | `softmax-update(head 7)` | idle |
| 1688 | `steady-state-stream-v(head 0)` | idle | idle |
| 1847 | `steady-state-stream-v(head 1)` | idle | idle |
| 2006 | `steady-state-stream-v(head 2)` | idle | idle |
| 2165 | `steady-state-stream-v(head 3)` | idle | idle |
| 2324 | `steady-state-stream-v(head 4)` | idle | idle |
| 2483 | `steady-state-stream-v(head 5)` | idle | idle |
| 2642 | `steady-state-stream-v(head 6)` | idle | idle |
| 2801 | `steady-state-stream-v(head 7)` | idle | idle |

Slice 0 ends at 2,960 (`steady-state-stream-v(head 7)`'s completion).
Matmul-issue span 160 → 2,960 = **2,800 cycles**, matching the
independently-derived per-slice figure exactly (`notes.md` §11.6/§11.12).
DMA goes idle at 680 and stays idle through the rest of slice 0 — chunk 2's
prefetch (the next DMA work) doesn't fire until much later in chunk 0, once
the buffer it needs actually frees (chunk-boundary segment, below).

## Universal matmul-issue/SFU table (every slice)

Checked directly against slice 0's own merged table above by re-deriving
in relative-offset form: identical cycle-for-cycle once you subtract out
slice 0's `slice_start` (160) — `metadata-init`/the `chunk 1` prefetch were
never actually *part of* slice 0's pattern, just independent SFU/DMA
events that happened to temporally overlap its early cycles. So this table
is genuinely universal, not slice-0-specific: `Δ` is offset from each
slice's own `slice_start`.

| Δ | matmul-issue | SFU |
|---|---|---|
| 0 | `load-stationary(K, slice N)` | idle |
| 128 | `steady-state-stream-qk(head 0)` | idle |
| 287 | `steady-state-stream-qk(head 1)` | `softmax-update(head 0)` |
| 446 | `steady-state-stream-qk(head 2)` | `softmax-update(head 1)` |
| 605 | `steady-state-stream-qk(head 3)` | `softmax-update(head 2)` |
| 764 | `steady-state-stream-qk(head 4)` | `softmax-update(head 3)` |
| 923 | `steady-state-stream-qk(head 5)` | `softmax-update(head 4)` |
| 1,082 | `steady-state-stream-qk(head 6)` | `softmax-update(head 5)` |
| 1,241 | `steady-state-stream-qk(head 7)` | `softmax-update(head 6)` |
| 1,400 | `load-stationary(V, slice N)` | `softmax-update(head 7)` |
| 1,528 | `steady-state-stream-v(head 0)` | idle |
| 1,687 | `steady-state-stream-v(head 1)` | idle |
| 1,846 | `steady-state-stream-v(head 2)` | idle |
| 2,005 | `steady-state-stream-v(head 3)` | idle |
| 2,164 | `steady-state-stream-v(head 4)` | idle |
| 2,323 | `steady-state-stream-v(head 5)` | idle |
| 2,482 | `steady-state-stream-v(head 6)` | idle |
| 2,641 | `steady-state-stream-v(head 7)` | idle |

Slice ends at Δ=2,800 (matches the independently-derived per-slice figure
exactly).

`slice_start` values, chunk 0 (chunk `c`'s slice `s` starts at
`160 + c×22,400 + s×2,800`): slice 0=160, 1=2,960, 2=5,760, 3=8,560,
4=11,360, 5=14,160, 6=16,960, 7=19,760.

## DMA overlay (per-slice, the only column that varies)

- **Slice 0 of chunk 0**: prologue continuation — see the Slice 0 merged
  table above (`load-K/V(K/V, chunk 1)` at Δ=200/360).
- **Slices 1–6 of every chunk, and slice 0 of chunks 1–7**: genuinely
  idle, zero DMA activity — confirmed, nothing queued.
- **Slice 7 of chunks 0–5**: chunk-boundary prefetch for `chunk+2` (the
  *other* double-buffer instance, since `chunk+1` was already loaded
  during the previous chunk's slice 0). **Note on the `+2`, so this
  doesn't read as contradicting `phase2-loop.md`**: this table shows
  real-time bundle co-occurrence — which target chunk's DMA instruction
  actually fires during chunk `c`'s real-time slice-7 window. That's a
  different question from source position: in `phase2-loop.md`, the
  instruction targeting chunk `c+2` is still textually chunk `(c+1)`'s own
  trailing line (`+1` relative to *its own* iteration, unchanged, per the
  earlier decision not to restructure). It just doesn't actually issue
  until real time reaches chunk `c`'s slice 7 — long before chunk
  `(c+1)`'s own slice loop starts on the matmul-issue slot — because DMA
  is in-order and stalls on the real WAR hazard, not on program-order
  proximity to any particular slice loop. DMA is idle and in-order with
  nothing else queued, so each half issues the instant its own real
  WAR-hazard gate clears — **not** tied to `load-stationary(V, slice 7)`
  (that's unrelated to the K-buffer and would just be an arbitrary delay,
  same class of mistake `metadata-init`-paired-with-`load-Q` was rejected
  for in §11.11):

  | Δ (from slice 7's `slice_start`) | matmul-issue | DMA |
  |---|---|---|
  | 128 | `steady-state-stream-qk(head 0)` | `load-K/V(K, chunk+2)` — issues the instant `load-stationary(K, slice 7)` finishes (its own last reader) |
  | 287 | `steady-state-stream-qk(head 1)` | ⋯ `load-K/V(K, chunk+2)`, busy til Δ288 |
  | 1,528 | `steady-state-stream-v(head 0)` | `load-K/V(V, chunk+2)` — issues the instant `load-stationary(V, slice 7)` finishes (its own last reader — **not** `steady-state-stream-v(head 7)`, corrected below) |

  All other Δ in this slice: DMA idle, same as the universal table's
  default. Worked example, chunk 0 → chunk 2 (`slice_start`=19,760):
  `load-K/V(K, chunk 2)` issues at 19,888 (→20,048); `load-K/V(V, chunk
  2)` issues at 21,288 (→21,448) — both complete well inside chunk 0's own
  slice 7, before the next slice even starts (22,560). Generalizes to
  every `c`=0..5 by shifting `slice_start` by `22,400c`.

  **Correction** (caught by an independent audit, `notes.md` §11.18): an
  earlier version of this row gated `load-K/V(V, chunk+2)` on
  `steady-state-stream-v(head 7)` instead — wrong, by the same logic
  already correctly applied to K one row up: each slice's
  `load-stationary` is the *only* thing that ever reads its scratchpad
  buffer; the streaming instruction (`steady-state-stream-v`) reads from
  the array's already-loaded stationary registers, not the V scratchpad
  directly (`isa.md`'s own `steady-state-stream-v` description never
  names the V buffer as an operand). Real last reader is
  `load-stationary(V, slice 7)`, 1,272 cycles earlier than the old gate.
  Zero cycle impact either way (margin is enormous either way), but it
  also silently contradicted the "genuinely idle" claim above — the old
  Δ=2,800 gate had `V(chunk+2)` spilling into the next slice's first 160
  cycles, which the corrected Δ=1,528 gate no longer does.
- **Slice 7 of chunks 6 and 7**: no prefetch (no `chunk 8`/`chunk 9` to
  fetch; chunk 7's slice 7 leads into the tail instead, `notes.md` §11.10).

## Tail (after chunk 7)

`softmax-finalize(head i)` is gated by `steady-state-stream-v(head i)` of
chunk 7/slice 7 (the V-phase, which does the last real write to `O`) —
**not** by `softmax-update(head i)`, since `softmax-update` only rescales
`O`, it doesn't add the final `P@V` contribution (`isa.md`'s own
`softmax-update` description, "add is `steady-state-stream-v`'s job").
`store-O(head i)` follows `softmax-finalize(head i)` immediately — both
SFU and DMA are otherwise idle in the tail, so each head's `finalize`+
`store-O` pair rides right behind its own `v(head i)`, with no cross-head
contention (each pair finishes in 37 cycles, `v`'s own per-head cadence is
159).

Chunk 7's slice 7 `slice_start` = 176,560 (`160 + 7×22,400 + 7×2,800`);
V-phase starts at `slice_start + 1,528` = 178,088.

| head | `steady-state-stream-v(i)` finishes | `softmax-finalize(i)` finishes | `store-O(i)` finishes |
|---|---|---|---|
| 0 | 178,247 | 178,279 | 178,284 |
| 1 | 178,406 | 178,438 | 178,443 |
| 2 | 178,565 | 178,597 | 178,602 |
| 3 | 178,724 | 178,756 | 178,761 |
| 4 | 178,883 | 178,915 | 178,920 |
| 5 | 179,042 | 179,074 | 179,079 |
| 6 | 179,201 | 179,233 | 179,238 |
| 7 | 179,360 | 179,392 | **179,397** |

`v(head 7)`'s finish (179,360) matches the chunk-7-finish figure from the
`slice_start` formula exactly (`160 + 8×22,400`) — cross-check holds.

**Total Q-tile latency: 179,397 cycles** — the full hand-schedule result
for one Q-tile's steady-state body, prologue through tail.
