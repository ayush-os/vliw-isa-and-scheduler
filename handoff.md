# Handoff — start here in a new chat

This project is a **pedagogical exercise**, not a "get it built" task. Read
this whole file before doing anything else, then pick up at **Phase 2**
(§ below) — Phase 0 and Phase 1 are complete.

## How to work with me on this project

I (the user) am doing this to learn, not to get a deliverable produced for
me. Your default role is **sounding board / Socratic helper, not answer-
giver**:

- When I make a claim or derivation, check it against the real source
  material (`spec.md`, `notes.md`, `workload-to-silicon/prefill_notes.md`,
  `decode_notes.md`) and push back on anything that doesn't hold up —
  don't just validate it.
- When the spec asks me to derive/decide something (🧠 in `spec.md`'s
  legend), don't do that derivation for me unprompted, even if I ask a
  vague "shall we do X" — ask how I want to work through it first (I derive
  and you check / we talk it through together / you draft and I react),
  the way we did for Phase 0's two decisions.
- If I assert a conclusion without actually walking through it (e.g. "it's
  pretty obvious that..."), that's a cue to push back and ask for the
  actual walk, not to accept the shortcut — that's exactly what happened
  with Decision 2 in Phase 0, and the derivation surfaced real content
  (the padding/bucketing caveat, the DMA-is-scheduled-not-cache-triggered
  mechanism) that the one-line version missed.
- **Calibration note from Phase 1's matmul-issue slot work**: "check,
  don't hand over the derivation" can tip into being cryptic if it's just
  another open question every time, several turns in a row, on the same
  sub-point — got direct user feedback on this once already, don't repeat
  it. If a claim's already been checked once and a follow-up is genuinely
  asking "what do you actually mean, concretely" — answer that plainly,
  don't volley another question back. And if I'm directly asked "what do
  you recommend" or "give me a final decision" on a judgment call under
  real uncertainty, give one, directly, with reasoning — that's a
  different request than doing someone's own derivation for them
  unprompted, and stalling on it with more hedging is its own failure
  mode, not extra rigor.
- **Exception, and it's explicit**: if I flag something myself as
  boilerplate, busywork, or "not contributing to my learning" — I'll say
  so directly — just do it or give me the answer. Don't make me ask twice.
  Research/reading tasks (like Phase 0's background reading) are also fair
  game to just execute fully, since the "learning" there is in absorbing
  the material, not in me re-deriving facts you can look up.
- **Working technique confirmed in Phase 2, worth defaulting to**: when a
  hardware-mechanism claim doesn't obviously resolve from description
  alone (e.g. "does loading a stationary chunk take N or 2N-1 cycles"),
  building a small concrete numeric example and tracing it by hand
  cycle-by-cycle settles it fast and catches real mistakes (a broadcast-
  vs-distinct-row mix-up in `load-stationary`'s mechanism). Same shape as
  Phase 1's "Fan-Out vs. Funnel" visual for the transpose-bit dead end
  (`notes.md` §5.3) — reach for a small worked example before a longer
  abstract argument when a mechanism claim feels unsettled.
- **Two modeling definitions locked in during Phase 2's bundle-scheduling
  work, worth carrying forward exactly, not re-litigating**:
  1. "Naive schedule" means every slot proceeds independently at its own
     pace, gated only by its own occupancy (decided: busy for an
     instruction's full latency, uniform across all three slots — not
     issue-and-free) and real data dependencies. It does **not** mean
     artificially serializing independent slots, and it does **not** mean
     reordering/interleaving to manufacture more overlap than falls out
     for free — natural overlap between independent slots is just what
     "naive" looks like, not an optimization. A first attempt at Phase
     2's bundle work blurred this (jumped to deliberately interleaving
     `load-Q` with compute before hazards/occupancy were even settled)
     and got corrected — settle the dependency graph and occupancy model
     *before* touching placement, same order as Itanium's kernel-before-
     rotating-register-overlap and CS243's dependency-graph-before-
     packing.
  2. Direct consequence of (1): for an in-order slot with nothing else
     competing for it, an instruction's real issue cycle does **not**
     depend on where it's textually written in the loop-nest pseudocode
     — only on real dependencies. Getting this backwards once caused a
     real (if minor) off-by-one — starting a dependent instruction 1
     cycle later than necessary — and nearly caused restructuring the
     loop nest for a change (moving a DMA chunk-prefetch earlier in the
     text) that would have cost real epilogue complexity for zero actual
     benefit, since the instruction was already resolving at that same
     real cycle regardless of its textual position. `notes.md`
     §11.8/§11.11/§11.12 have the full detail and worked examples.

## Project status

**Phase 0: complete.** Reading done, both decisions made and derived (not
just asserted). Full log with reasoning and citations: **`notes.md`** —
read that before Phase 1, it has the real content, this file is just a
pointer + status.

- **Decision 1 (which machine): prefill.** Three architecturally separate
  issue targets (128×128 systolic array, softmax vector/scalar unit, DMA)
  vs. decode's two (SIMD engine + DMA, where the per-lane reduction lives
  inside the SIMD unit, not as a separate slot type). Decode stays flagged
  as a future "second machine, same ISA family" extension — not this
  project's job.
- **Decision 2 (Itanium-vs-Groq, derived against the real prefill loop
  nest): yes on all three checks**, each with a real caveat worth carrying
  forward, not just a yes/no:
  1. Trip counts known at compile time — yes, but only under a fixed-shape
     idealization; real serving needs Groq-style shape bucketing to
     generalize.
  2. No runtime-data-dependent instruction latency — yes, because DMA is
     *scheduled* by the compiler ahead of time, not triggered reactively
     by a cache miss (the actual scratchpad-vs-cache distinction, not just
     "DMA is fast").
  3. No data-dependent branching — yes, and `prefill_notes.md` §2.5's
     tile_q/GQA-reuse tension (forced ~256× K/V re-fetch) is good evidence
     *for* this, not against it: still fully static, but a real
     demonstration that "static" isn't the same as "free" — real scrutiny
     found real cost that a one-line "it's obviously static" would have
     missed.

Full reasoning, sources, and three background-reading summaries (Itanium/
EPIC, TI C6000/Hexagon, Groq's real ISA) plus the Saman Amarasinghe
"Compiler 2.0" lecture TLDR are all in `notes.md` §1.

## Phase 1: COMPLETE — full ISA defined, all three slots, deliverable written

Per `spec.md` Phase 1 — three slot types for prefill, each grounded in a
specific prior finding, not invented from scratch. **The clean deliverable
is `isa.md`** — read that first for the actual spec (8 instruction types
across matmul-issue/SFU/DMA, every field, no narrative). Full derivation
and — importantly — **every real correction made along the way** is in
`notes.md` §5–§7; read that before touching Phase 2, since the corrections
carry real methodological lessons (below) that will recur in Phase 2's own
derivation.

**Final structure**: 3 instructions in matmul-issue (`load-stationary`,
`steady-state-stream-qk`, `steady-state-stream-v` — this one bifurcated
from an original single `steady-state-stream`, see below), 2 in SFU
(`softmax-update`, `softmax-finalize`), 3 in DMA (`load-Q`, `load-K/V`,
`store-O`). 8 total.

### The single biggest recurring lesson: compile-time-known ≠ operand-free

This tripped up nearly every "does this field need to exist" question
across all three slots until it became the default lens. Static/fixed-
schedule does **not** mean an address disappears — it means the compiler
computes it ahead of time instead of hardware computing it at runtime.
That's Decision 2's own point (Itanium/Groq), and it kept resurfacing at
the ISA level: a field only truly collapses to zero bits when the value is
a genuine hardware/workload **constant** (always the same, every instance
— e.g. array width, `tile_q`, dataflow, transpose). A value that's
compile-time-*known* but **varies** across instances (which head, which
buffer, which HBM offset) still needs real bits — just fewer than a naive
full address, once you check whether the set of real destinations is
small and enumerable (collapses to a compact index) or genuinely large
(needs a real offset field, e.g. DMA's 16/11-bit HBM offsets).

### Other real corrections worth knowing before Phase 2 (full detail: `notes.md` §6.2, §7.4)

- **A field name silently meant two different things across two
  instructions** (`accum_src_addr` = "the per-head state block" on
  `softmax-finalize`, but = "the raw-S buffer" on `softmax-update`) and
  that ambiguity hid a real missing field for several turns. Lesson: check
  what a field *actually addresses* per-instruction, not just its name.
- **P (post-softmax, scratchpad) needed ×8 per-head residency, not a
  single reused buffer like raw-S** — because the array can only hold one
  stationary operand (K or V) at a time, so all 8 heads' P gets produced
  under K-stationary before any of it is consumed under V-stationary. This
  single hardware fact (one stationary operand at a time) is what forced
  `steady-state-stream` to bifurcate into `-qk`/`-v` in the first place —
  the highest-leverage finding of the whole phase.
- **This surfaced a real, un-clean-slate correction to
  `prefill_notes.md` §2.3's own scratchpad budget** — that phase assumed a
  single reused 4,096 B P buffer; the real requirement is ×8 = 32 KB.
  Checked against the 1 MiB ceiling: still fits (plenty of slack), so
  `tile_k`=1024 stands, but the original hypothesis's P-sizing was
  incomplete. Not edited into `prefill_notes.md` itself (a different,
  completed project's polished deliverable) — recorded in `notes.md` §6.2
  as where the gap was actually found.
- **DMA's granularity comes from HBM contiguity, not an analogy to
  compute's per-head structure.** It was tempting to assume DMA needed the
  same head-idx treatment as the compute slots by analogy; the real
  constraint turned out to be Q/O's non-contiguous-across-heads HBM
  layout, which forced per-head DMA issuance for an entirely different
  reason (§7.4).
- **DMA is HBM↔scratchpad only, never accumulator-facing** — matches real
  Gemmini's own `ex_read_from_acc`/`ex_write_to_spad` vs. `mvin`/`mvout`
  separation. This forced `softmax-finalize`'s destination to move from
  accumulator to a dedicated scratchpad region late in the process (§6.5)
  — worth remembering that a downstream slot's scoping decision reached
  back and changed an already-"settled" earlier instruction.

### Encoding pass: also done — bundle layout + full sizing (notes.md §8)

The bit-widths and bundle-layout items that were still open as of the
first Phase-1-complete pass are now resolved too:

- **Bundle layout: fixed-width-per-slot**, chosen only after checking real
  per-slot issuance rates rather than assuming — matmul-issue:SFU:DMA per
  Q-tile ≈ 144:72:32, meaning even a best-case schedule leaves DMA idle
  ~78% of cycles. The "pipelining keeps every slot busy" intuition that
  motivated this doesn't actually hold up against the numbers — the
  fixed-width conclusion stands anyway, for different reasons (idle-slot
  cost is storage/code-density, not throughput; decode simplicity matches
  this project's standing preference for minimal control hardware; direct
  Groq precedent — "144-wide VLIW instructions," a fixed format). Compact/
  variable is the stated explicit alternate, worth revisiting only if code
  density becomes the real bottleneck (this design has zero hardware loop
  construct, so a fully-unrolled program could get large).
- **Full capacity/sizing pass** — surfaced two more real gaps in the same
  shape as the P-sizing correction: `prefill_notes.md` §2.3's original
  "fixed Q/output (8,192 B)" scratchpad term assumed both were small
  transient buffers, but Q needs ×8 per-head residency (loop-order-driven,
  same as P) and output needs its own ×8 region too (post-`finalize`
  destination correction), at int8 not fp32 (fp32 was only ever needed
  for softmax's internal stability). Real total: 65,536 B against the
  original 8,192 B — still fits comfortably (full scratchpad/accumulator
  tables in `isa.md` §5).
- **Final bundle width: 32 bits** — 7 (matmul-issue) + 4 (SFU) + 21 (DMA),
  opcode widths mechanical once layout is fixed.

**One item genuinely deferred to Phase 2, non-blocking**:
`load-stationary`'s re-issue frequency (whether it's issued once per
128-wide slice per chunk-load, reused across all 8 heads before the next
slice) — doesn't change any field width, only how many instructions
Phase 2 ends up emitting.

**Phase 1 is fully complete.** Every encoding decision `spec.md` requires
is made and written up in `isa.md`. Next: Phase 2 itself — hand-scheduled
bundle sequence first, then the automated scheduler, both against
`isa.md` (per `spec.md`, updated after Phase 0 to require both).

### Post-completion correction found while reconstructing the loop nest in the Phase 2 session (`notes.md` §9)

Worth knowing before trusting any instruction-count number from earlier
in this file or from memory of this project: the original bundle-layout
justification (144:72:32 matmul:SFU:DMA) used *coarse* granularity for
`steady-state-stream-qk/v` and `softmax-update` — one call per (chunk,
head) — when `prefill_notes.md` §2.3 already establishes softmax runs
**fine-grained, per 128-wide array sub-pass** (8 slices per chunk,
exactly matching `load-stationary`'s slice-idx field, which *was*
correctly designed around this from the start). No field or bit-width was
actually wrong — only the issuance counts and loop-nest documentation
were. Corrected ratio: **1,152:520:32** (≈36:16:1). The fixed-width
bundle-layout conclusion still holds (the reasoning was never
ratio-dependent), just with more extreme idle-slot waste than originally
stated.

This also forced one real decision that had gone unmade: whether P's
residency is ×8 or ×64 depends on whether the K/V phases interleave per
slice or batch across the whole chunk — and unlike a pure scheduling
choice, this one changes field counts (`softmax-update`/
`steady-state-stream-v` would need a 6th field, a slice-idx, under
chunk-batching). Settled here as slice-interleaved (identical reload
count either way, so batching buys nothing) — P-residency is genuinely
×8, not conditional; full reasoning in `notes.md` §9.4. `isa.md` §3/§4/§6
and `notes.md` §9 are the current, correct state — anything computed
before this correction (including earlier in this file, if you're reading
an old version) is superseded.

**Second post-completion change, found *during* Phase 2's own hazard
work, not a Phase 1 gap**: the ISA now has **9 instruction types, not 8**,
and the **bundle is 33 bits, not 32**. A new `metadata-init` instruction
(SFU slot) was added to fix a real gap — nothing initialized the
online-softmax recurrence's `m_0=-inf, l_0=0, O_0=0` before each Q-tile's
first real `softmax-update` — which pushed SFU's opcode from 1 to 2 bits.
Full reasoning: `notes.md` §11.9. `isa.md` is current; every "8
instructions" / "32 bits" reference elsewhere in this file (including
just above) reflects Phase 1's state at the time and is superseded.

## Phase 2: IN PROGRESS — prologue/epilogue resolved, slice 0 + generalization + chunk-boundary DMA derived; tail and literal bundle write-up still open

Per `spec.md` Phase 2: hand-schedule the real bundle sequence first
(before the automated scheduler). **Full working log: `notes.md` §11**
— §11.1–§11.7 is latency/clock-decision groundwork from an earlier
session; **§11.8 onward is the current session's work and the real
pickup point.**

**Scope**: one Q-tile's steady-state body, written out in
[`phase2-loop.md`](phase2-loop.md) — the outer `batch`/`kv_group`/`q_tile`
loops are identical repeats at different addresses, not new scheduling
content, so they're intentionally excluded from that file. **This
exclusion hides a real, unexplored optimization — see "Not yet done"
item 3 below, it's a deliberate scoping choice, not a forgotten thread.**

**Done, this session, in order** (`notes.md` §11.8–§11.12 has full
derivations, every real correction, and worked numbers for all of this):

1. **Slot-occupancy model decided**: busy for an instruction's full
   latency, uniform across all three slots — not issue-and-free.
   Physically forced for matmul-issue (the array can't do two things at
   once); made uniform for DMA/SFU too since it costs nothing given their
   slack, and avoids the OOO-like completion-tracking hardware Decision 2
   already ruled out. §11.8.
2. **Real hazard found and fixed: raw S's single accumulator location.**
   `steady-state-stream-qk(head i+1)` writing S before
   `softmax-update(head i)` finished reading it would cost a real 14,336
   stall cycles/Q-tile if left alone. The obvious ×8 fix doesn't actually
   fit (over the accumulator budget by ~2 KB — confirmed against a number
   already on record from §9.1). **Fixed with double-buffering (×2)
   instead** — a complete fix (softmax-update's 32-cycle latency ≪
   qk's 159, so the "other" buffer is always long-free), zero new
   instruction bits (`head_idx & 1`, derived from a field already
   present on both instructions). `isa.md` + `notes.md` §11.8.
3. **Full hazard pass** over every other shared resource (array, P, O,
   metadata) — S was the only *live* one. P's cross-slice WAR-safety and
   O's rescale-then-add ordering were flagged as correct only by virtue
   of strict in-order execution — later confirmed with real numbers
   in §11.12 (margin ≈ one full slice, ~2,800 cycles, not close to zero).
4. **Second real gap found: nothing initializes `m_0=-inf, l_0=0, O_0=0`**
   before each Q-tile's first real `softmax-update` (the 8 per-head
   accumulator slots are physically reused across all 65,536 Q-tile
   instances). New instruction **`metadata-init`** added (SFU slot,
   head-idx only, zero value bits — real hardware constants). Considered
   piggybacking on `load-Q` (same per-Q-tile-per-head frequency) —
   **rejected outright**, violates the already-settled
   DMA-never-touches-accumulator rule (§6.5). Considered a mode bit on
   `softmax-update` instead of a new opcode — found to cost the *same*
   number of bits either way; separate opcode chosen anyway to keep the
   hot-path instruction (512 issues/Q-tile) uniform, same move as
   `load-stationary`'s transpose bit. **SFU is now 3 instructions, opcode
   1→2 bits, total bundle 32→33 bits.** `metadata-init`'s own latency
   (32 cycles) assigned by analogy to `softmax-update`/`-finalize`'s
   write path, not independently derived. `isa.md` + `notes.md` §11.9.
5. **§11.2's prologue and epilogue gaps, both resolved.** Prologue is
   `load-K/V(K, chunk 0)` → `load-Q ×8` → `load-K/V(V, chunk 0)` (**this
   order matters** — `K,V,Q` would cost a real ~37-cycle stall on
   `Q(head 0)`, found by actually computing the timeline, not by
   inspection, §11.11) plus `metadata-init ×8` as its own independent
   SFU-slot list (no forced same-bundle pairing with `load-Q` — considered
   and rejected, would only ever cost cycles). Epilogue: chunk 7 peeled
   out of the loop with its own body, minus the trailing prefetch —
   compile-time-structural, not a runtime guard (zero branch instructions
   in this ISA). `phase2-loop.md` reflects both.
6. **Slice 0's full timeline derived, cycle by cycle**, starting at
   287.844 (prologue's end): K-phase pipelined across 8 heads with zero
   inter-head stall (the direct payoff of the S double-buffer fix),
   V-phase checked and found genuinely hazard-free (P/O already had
   proper ×8 addressing from Phase 1). Slice finishes at 2,960 — cross-
   checked exactly against §11.6's independently-derived 2,800-cycle/slice
   figure. One real off-by-one caught and fixed along the way (starting a
   dependent instruction 1 cycle later than its true free cycle). §11.12.
7. **Generalized to slices 1–6**: simpler than slice 0, not identical —
   no DMA gate (chunk 0's K/V already resident), so each slice is exactly
   2,800 back-to-back cycles. Chunk 0 finishes at cycle 22,559.844 —
   matches §11.6's 22,400-cycles/chunk figure exactly. §11.12.
8. **Chunk-boundary DMA worked out through chunk 2, with a general
   finding worth keeping**: for an in-order slot with nothing else
   competing for it, an instruction's real issue cycle doesn't depend on
   where it's textually written in the loop nest — only real dependencies
   matter (see the new "How to work with me" bullet above). Chunk 1's
   prefetch moved into the prologue in `phase2-loop.md` (confirmed
   zero-cost). Chunk 2's prefetch checked and deliberately **left where
   it already sits** — real numbers show ~23,500 cycles of margin either
   way, and moving it would double the §11.10 epilogue special-casing
   (chunks 6 *and* 7 both hitting nonexistent-chunk trailing prefetches,
   instead of just 7) for zero benefit. §11.12.

Latency table (`notes.md` §11.4, now including `metadata-init`):

| Instruction | Cycles |
|---|---|
| `load-stationary` | 128 |
| `steady-state-stream-qk` / `-v` | 159 |
| `softmax-update` / `softmax-finalize` / `metadata-init` | 32 (SFU width is a stated hypothesis, no real hardware anchor — internal per-row pipeline depth not modeled, unlikely to matter, SFU has ~55% idle slack) |
| `load-K/V` | 159.844 (cycles = ns at 1 GHz) |
| `load-Q` / `store-O` | 4.995 (same) |

Clock: **1 GHz**, decided on real grounds (no synthesis pass ever run for
this design; Timeloop's 2 GHz run was "int8-pumped," not a free bump) —
`notes.md` §11.5. DMA-hiding checked with real numbers throughout, not
assumed — margins ranging from ~70× (general, §11.6) up to ~23,500 cycles
absolute (chunk 2's specific case, §11.12).

**The hand-schedule deliverable is done.** [`bundle-schedule.md`](bundle-schedule.md)
is the actual bundle table `spec.md` asks for — prologue, slice 0, a
universal matmul-issue/SFU table (identical for every slice, parameterized
by `slice_start`) with a per-slice DMA overlay, chunk-boundary DMA, and the
tail, all with real cycle numbers. **Total Q-tile latency: 179,397
cycles** — the number Phase 3 will compare the automated scheduler
against. Full derivation, every correction, in `notes.md` §11.13–§11.16.

**One real modeling gap caught and fixed along the way, worth knowing
about since it touches every cycle number in this project**: the
`.844`/`.995`-style fractional DMA cycle counts used throughout
§11.11/§11.12 (an earlier session) weren't physically valid — one bundle
issues per clock, so a slot can't hand off mid-cycle. **Fix: ceiling each
DMA instruction's own latency independently** — `load-K/V`=160,
`load-Q`/`store-O`=5, not the raw `159.844`/`4.995` (`notes.md` §11.4).
Every cycle number in `bundle-schedule.md` uses the corrected values;
anything computed before this fix (including earlier revisions of this
file) is stale. Structural findings (DMA ordering, margin sizes,
hazard-free conclusions) were never affected by this, only the exact
numbers.

**Roadmap for the rest of Phase 2 — evaluated on learning-value-to-time,
not just followed from `spec.md`'s checklist** (`notes.md` §11.17 has the
full reasoning):

1. **Cross-iteration software pipelining: evaluated and deliberately
   dropped, not just deferred.** The savings are the ~160-cycle prologue
   wait × 65,536 Q-tiles ≈ 10.5M cycles, against a total workload of
   65,536 × 179,397 ≈ 11.76 billion cycles — **≈0.09% of total runtime**.
   The technique (Itanium-style rotating-register pipelining) is already
   the project's conceptual reference point from Phase 0's reading;
   actually implementing it here would mean redoing Phase 2's whole
   hazard-then-schedule methodology at the outer-loop level for a
   fraction-of-a-percent win. Bad time-to-learning ratio — not picking
   this up unless something changes that math.
2. **Automated scheduler: the one real remaining item, scoped down.**
   Worth doing — it's the only way to actually test the hand-schedule's
   unverified "no avoidable stalls found ≠ optimal" caveat (§11.12) rather
   than asserting it, and it's the direct payoff of the project's own
   Phase 0 motivating question (does static scheduling match hand-tuning,
   Itanium's real historical problem). **But scoped to what the
   comparison actually needs, not full CS243 generality**: this problem
   is one basic block, 3 fixed-occupancy resource slots, no control flow,
   no register allocation — a greedy list-scheduler over the same
   dependency graph already derived in §11.8 (S double-buffer, P/O
   ordering, metadata recurrence) is enough. Prediction worth stating
   before building it, so it's checkable afterward: given how
   constraint-bound the hand schedule already was (matmul-issue is the
   real bottleneck almost everywhere; SFU/DMA idle because there's
   genuinely nothing for them to do, not from scheduling slack), expect
   the automated result to land very close to 179,397 cycles — itself the
   interesting finding if true.

**One framing to hold onto**: don't call the current derivation
"optimized." "No avoidable stalls found" is the honest, weaker claim —
everything checked out mainly because the hard architectural constraints
(one stationary operand at a time; the softmax recurrence's sequential
chunk dependency) left little room to begin with, not because
alternative orderings were tried and compared.

## Files in this repo

- `spec.md` — the project spec (source of truth for phase structure/
  deliverables; has been updated once already, e.g. Phase 2/3 restructured
  to require both a hand-scheduled and an automated bundle sequence)
- `isa.md` — **the living ISA spec**, kept current through Phase 2 (not
  frozen at Phase 1): clean, final instruction-format spec, now 9
  instructions (Phase 2 added `metadata-init`, §11.9), 33-bit bundle, no
  narrative. Start here for Phase 2.
- `notes.md` — full working log: Phase 0 (reading summaries, Decision 1/2
  derivations, open threads) + Phase 1 (§5–§9, slot-by-slot ISA design
  *including every correction and dead end*, not smoothed over) + §10
  (a standalone cross-project GQA-reuse finding, unrelated to Phase 2) +
  §11 (Phase 2 — §11.1–§11.7 latency/clock groundwork, §11.8–§11.12 hazard
  fixes + prologue/epilogue + slice-0/chunk-boundary derivation,
  §11.13–§11.16 the bundle-table transcription with the ceiling-latency
  correction, §11.17 the evaluated roadmap decision for the rest of Phase
  2 — real pickup point is §11.17's "Next": the automated scheduler)
- `phase2-loop.md` — the Phase 2 loop nest actually being hand-scheduled
  (one Q-tile's steady-state body), kept separate from `notes.md` so it's
  easy to reference by line number
- `bundle-schedule.md` — **the complete hand-scheduled bundle sequence**,
  `spec.md`'s literal Phase 2 deliverable, done: merged matmul-issue/SFU/
  DMA table, one row per issue event, all five segments (prologue, slice
  0, universal matmul-issue/SFU table + per-slice DMA overlay,
  chunk-boundary DMA, tail). **Total Q-tile latency: 179,397 cycles.**
  Clean deliverable only — derivation and format decisions are in
  `notes.md` §11.13–§11.16.
- `handoff.md` — this file
- `../workload-to-silicon/prefill_notes.md` — the real hardware hypothesis
  this ISA targets (128×128 array, WS dataflow, online-softmax, scratchpad
  sizing) — sibling repo, not inside this one. Note: `notes.md` §6.2 found
  a real gap in this document's own §2.3 P-sizing (not fixed there, see
  above).
- `../workload-to-silicon/decode_notes.md` — decode's hardware hypothesis
  (context for Decision 1, and the flagged future second-machine target)

## Open threads to keep in mind for later phases

From `notes.md` §4 — not blocking Phase 1, but don't let them get lost:

- Hexagon's SMT-as-latency-hedge: worth one explicit sentence somewhere
  (Phase 1 or 4) that this project deliberately rejects any analogous
  dynamic escape valve, per the spec's own "no OOO/dynamic modeling"
  scope note — a stated choice, not an oversight.
- Fixed-shape idealization: if Phase 4 synthesis touches realistic
  serving, the honest answer is shape-bucketing (Groq's move), not
  dynamic scheduling.
- Phase 3's cost-model-accuracy question (Amarasinghe/ITEMAL): this
  project has no real-silicon measurement to check hand-derived latencies
  against, unlike ITEMAL's training data — flag this honestly in Phase 3
  rather than assuming the hand-derived numbers are trustworthy.
