# Handoff — start here in a new chat

This project is a **pedagogical exercise**, not a "get it built" task. Read
this whole file before doing anything else. Phase 0 and Phase 1 are
complete. **The hand-schedule half of Phase 2 is settled for now** at
[`bundle-schedule-v3.md`](bundle-schedule-v3.md) (**49,476 cycles**, down
from 179,397 originally) — both the occupancy-model bug and the
`load-stationary` shadow register are fixed/adopted. **The real pickup
point right now is a new ISA extension**, not the automated scheduler: an
independent audit (`notes.md` §11.29) found real evidence, in Gemmini's
actual RTL, that the array can feed weight-load and streaming operands
through genuinely separate physical ports concurrently — meaning
`load-stationary`'s remaining 256 cycles/slice (33% of matmul-issue-slot
time) is a limitation of *this project's own bundle format* (one shared
issue slot), not the hardware. Pursuing this next, target **33,220
cycles** (1.49× further, 1.4% above the theoretical floor) — see
"Starting fresh on the ISA extension" below for the full brief.
**`spec.md`'s "both a hand-scheduled and an automated sequence"
requirement is being treated as a starting point, not a hard
requirement** — explicit decision, the automated scheduler's status is
now open, to be revisited after this extension, not a fixed commitment
(`notes.md` §11.30). `bundle-schedule.md` (original, 179,397 cycles) and
`bundle-schedule-v2.md` (stream-occupancy fix alone, 65,605 cycles) are
both kept as before/after baselines — each marked superseded at its own
top, pointing forward to the next version.

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
  work, worth carrying forward exactly, not re-litigating — except the
  specific occupancy rule in (1)'s parenthetical, which is now actively
  being corrected, see `notes.md` §11.19 and "Project status" below
  before assuming it still holds**:
  1. "Naive schedule" means every slot proceeds independently at its own
     pace, gated only by its own occupancy (~~decided: busy for an
     instruction's full latency, uniform across all three slots — not
     issue-and-free~~ **superseded, §11.19: full-latency occupancy is
     wrong for the two systolic-stream instructions specifically — it
     conflates pipeline drain time with real issue occupancy, confirmed
     against `prefill_notes.md`'s own Timeloop results. The
     issue-and-free-vs-full-latency framing itself may need a third
     option: full-latency for `load-stationary`/SFU/DMA, feed-rate for
     the two streams — TBD, this is the open question**) and real data
     dependencies. It does **not** mean
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

## Phase 2: hand-schedule at 49,476 cycles for now — pursuing one more ISA extension (target 33,220), automated scheduler status open

**Status in one paragraph, read this first**: the hand-schedule went
through three versions. **v1** (`bundle-schedule.md`, 179,397 cycles) was
built on a slot-occupancy model (busy for an instruction's *full*
latency, uniform across all three slots) that an independent audit found
conflates a systolic pipeline's *drain time* with real issue *occupancy*
for the two streaming instructions — confirmed against real Timeloop data
in `prefill_notes.md` §3.1. **v2** (`bundle-schedule-v2.md`, 65,605
cycles, `notes.md` §11.19–§11.24) fixed that: stream latency stays 159
cycles (`N+D−1`), occupancy is 32 (`N`, the feed count). Full hazard
re-pass done: S's double-buffer margin collapsed to ~1 cycle under the new
model, so S is now **×4** (`isa.md` §3/§5); P/O, metadata, DMA all survive
with real (smaller but comfortable) margins. v2 still fully serialized
`load-stationary` ahead of its slice's streams, though, producing a real
127-cycle idle bubble at every K↔V transition (~25% of each slice) — not
a scheduling gap, a hard resource constraint (one physical set of weight
registers, can't be overwritten until the prior stream fully drains).
**v3** (`bundle-schedule-v3.md`, **49,476 cycles**, `notes.md`
§11.26–§11.28) fixes that too: a `load-stationary` shadow register,
confirmed *real* by reading Gemmini's actual RTL directly (`PE.scala` has
two registers per PE, `c1`/`c2`, gated by a `propagate` signal — genuine
double-buffered stationary storage, not fabricated, and stronger evidence
than the higher-level README/paper check that originally punted this).
`load-stationary`'s own 128-cycle occupancy is *not* eliminated (it still
fully occupies the single matmul-issue slot) — only the *extra* bubble is,
saving 127 of every 254 cycles per slice, not "all" of it (an earlier
"cost → 0" framing this session was caught and corrected, `notes.md`
§11.27). Hazard re-pass again: S unaffected (its hazard is intra-phase,
untouched by an inter-phase-transition fix); P and O both shrink by
exactly 127 cycles (638→511, 320→193) — same root cause, still safe; O's
*old* safety justification actually breaks under the new timing and needs
replacing with a real per-head check (still safe, for a different reason,
`notes.md` §11.28).

**Net result: 3.63× faster than the original hand-schedule**, landing at
49,476 cycles — still 1.51× above the theoretical 32,768-cycle MAC-bound
floor, a real structural gap (`load-stationary` still eats 33% of every
slice's matmul-issue-slot cycles, since it shares the single issue slot
with the compute streams — the shadow register fixed the *data* hazard,
not the *issue-bandwidth* one; closing that would need a separate
weight-load path, a bigger ISA change, not part of this).

Per `spec.md` Phase 2: hand-schedule the real bundle sequence first
(before the automated scheduler). **Full working log: `notes.md` §11**
— §11.1–§11.7 is latency/clock-decision groundwork from an earlier
session; §11.8–§11.16 is the hand-schedule derivation (now superseded by
the occupancy-model bug, but the hazard-finding methodology in it is
still correct and will be re-applied); **§11.17–§11.19 is the current
real pickup point** — §11.17's roadmap decision (skip cross-iteration
pipelining, scope the automated scheduler down), §11.18's audit findings,
§11.19's decision to fix the occupancy model first, with a concrete
numbered "Next" list to resume from.

**Scope**: one Q-tile's steady-state body, written out in
[`phase2-loop.md`](phase2-loop.md) — the outer `batch`/`kv_group`/`q_tile`
loops are identical repeats at different addresses, not new scheduling
content, so they're intentionally excluded from that file. **This
exclusion hides a real, unexplored optimization (cross-iteration software
pipelining) — evaluated below in "Roadmap for the rest of Phase 2" and
deliberately dropped on real numbers, not a forgotten thread.**

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

**The hand-schedule deliverable is done, three times over.**
[`bundle-schedule.md`](bundle-schedule.md) was the original answer — full
bundle table, prologue through tail, real cycle numbers — built against
the *wrong* occupancy model (busy-for-full-latency, uniform across all
three slots). **179,397 cycles.** Full derivation, every correction:
`notes.md` §11.13–§11.16. Kept as-is, not deleted, for the before/after
comparison.

[`bundle-schedule-v2.md`](bundle-schedule-v2.md) fixed the occupancy
model (32-cycle stream occupancy, S at ×4) but still fully serializes
`load-stationary` ahead of its slice's streams. **65,605 cycles — 2.73×
faster than v1.** Full derivation: `notes.md` §11.19–§11.24.

[`bundle-schedule-v3.md`](bundle-schedule-v3.md) is the current answer —
adds a `load-stationary` shadow register (confirmed real in Gemmini's
actual RTL, `notes.md` §11.26), removing the 127-cycle idle bubble v2 has
at every K↔V transition. **49,476 cycles — 1.33× faster than v2, 3.63×
faster than v1.** Full derivation: `notes.md` §11.26–§11.28.

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
not just followed from `spec.md`'s checklist** (`notes.md` §11.17, §11.25,
§11.26–§11.28 have the full reasoning):

1. **Cross-iteration software pipelining: evaluated and deliberately
   dropped, not just deferred.** The savings are the ~160-cycle prologue
   wait × 65,536 Q-tiles ≈ 10.5M cycles, against a total workload of
   65,536 × 179,397 ≈ 11.76 billion cycles — **≈0.09% of total runtime**.
   The technique (Itanium-style rotating-register pipelining) is already
   the project's conceptual reference point from Phase 0's reading;
   actually implementing it here would mean redoing Phase 2's whole
   hazard-then-schedule methodology at the outer-loop level for a
   fraction-of-a-percent win. Bad time-to-learning ratio — not picking
   this up unless something changes that math. (Note: the workload total
   here still cites the pre-fix 179,397; not worth recomputing since the
   conclusion — negligible percentage either way — doesn't change.)
2. **`load-stationary` shadow register — done, adopted.** Was punted,
   then investigated for real (`notes.md` §11.26): Gemmini's actual RTL
   (`PE.scala`) has genuine double-buffered stationary registers (`c1`/
   `c2`, gated by a `propagate` signal) — real, not fabricated, and
   better-grounded than the higher-level docs check that originally
   punted it. Adopted, rebuilt as `bundle-schedule-v3.md` (49,476 cycles,
   1.33× faster than v2). An initial "this makes `load-stationary`'s cost
   ~0" framing was too strong and got corrected (`notes.md` §11.27) — the
   real saving is the 127-cycle bubble only, not the instruction's own
   128-cycle occupancy, which is unavoidable given the single matmul-issue
   slot.
3. **`load-stationary`/stream concurrent-issue ISA extension — the
   active next step** (`notes.md` §11.29–§11.30). An independent audit
   read Gemmini's `ExecuteController.scala` and `PE.scala` directly and
   found real evidence the reference hardware feeds weight (`B`) and
   activation (`A`/`D`) operands through genuinely separate physical ports
   *simultaneously, every cycle, as normal operation* — not a special
   case. `load-stationary`'s remaining 256 cycles/slice (33% of
   matmul-issue-slot time) is a limitation of this project's own
   single-opcode bundle format, not the hardware. Target: **33,220
   cycles** (1.49× further, 1.4% above the theoretical 32,768-cycle
   floor) — see "Starting fresh on the ISA extension" below for the full
   brief. Real, bounded work (bundle-width extension, hazard re-derivation
   under genuine concurrency, one more schedule rebuild) — not adoptable
   for free the way the shadow register was, but well-precedented, unlike
   speculative redesign.
4. **Automated scheduler — status now open, not a fixed requirement.**
   `spec.md`'s "both a hand-scheduled and an automated sequence" framing
   is being treated as a starting point, not binding (explicit project
   decision, `notes.md` §11.30) — worth revisiting *after* item 3, not
   before, and not guaranteed to happen at all. If pursued: still worth
   doing for the reasons already on record (only way to test the
   hand-schedule's "no avoidable stalls ≠ optimal" caveat, direct payoff
   of the project's Phase 0 motivating question), still scoped down to a
   greedy list-scheduler (not full CS243 generality) over the same
   dependency graph already derived. See "Starting fresh on the automated
   scheduler" below if picking this up instead of/after item 3 — that
   section is unchanged and still accurate, just no longer the immediate
   next step.

**One framing to hold onto**: don't call the current derivation
"optimized." "No avoidable stalls found" is the honest, weaker claim —
everything checked out mainly because the hard architectural constraints
(one stationary operand at a time; the softmax recurrence's sequential
chunk dependency) left little room to begin with, not because
alternative orderings were tried and compared.

## Starting fresh on the ISA extension — read this section first if you're picking this project up new

Everything the audit and follow-up derived, consolidated in one place so
you don't need to read all of §11's history to get moving.

**Files to actually read**: `isa.md` (the ISA — 9 instructions, 3 slots),
`bundle-schedule-v3.md` (current hand-schedule, the thing you're
improving on), `phase2-loop.md` (the loop nest). `notes.md` §11.29–§11.30
has the full audit findings and reasoning if you need to double-check
something below.

**The actual task**: `load-stationary` currently shares the single
matmul-issue slot with the two stream instructions, so its 128-cycle
occupancy fully blocks streaming even though the shadow register (already
adopted, `-v3`) lets it write into an inactive register concurrently.
Real Gemmini hardware feeds weight (`B`) and activation (`A`/`D`)
operands through separate physical ports *simultaneously, every cycle, as
normal operation* (confirmed via `MeshWithDelays`/`ExecuteController.scala`
— not a special case, this is how the array always works) — so the
underlying mechanism is real and precedented; the gap is purely that this
project's own bundle format has no way to issue a weight-feed step and a
stream step in the same cycle.

**What needs deciding/designing** (genuinely open — this is real ISA
design work, not mechanical transcription, same spirit as Phase 1's
original slot decisions):
1. **Encoding approach**: a second, independent matmul-related bundle
   field (effectively a 4th slot) vs. a fused opcode that issues a stream
   instruction and advances an in-progress weight-load simultaneously.
   Trade-offs to work through: bundle-width cost, whether "weight-load
   in progress" needs to be explicit state or can be inferred, how this
   interacts with the existing `opcode`/`head-idx`/`src` fields already
   in `isa.md` §1–§3.
2. **Hazard re-derivation under genuine concurrency**: previous hazard
   passes (S/P/O/metadata/DMA) all assumed instructions execute one at a
   time on a given resource. With a weight-feed and a stream genuinely
   concurrent, re-check whether any *new* hazard appears (e.g., does the
   weight-feed's own progress interact with anything the stream touches?)
   — expect this to be a smaller pass than S/P/O's original derivation,
   since the two paths are physically independent in the reference
   hardware, but don't assume, check.
3. **Capacity/timing recheck**: confirm `isa.md` §5's capacity tables and
   the latency/occupancy table still hold (should — this doesn't change
   any instruction's own latency, only issue concurrency).
4. **Schedule rebuild**: `bundle-schedule-v4.md`, same five-segment
   structure as `-v2`/`-v3`. Target: **33,220 cycles** (audit's own
   derivation, `notes.md` §11.29 — re-verify rather than take on faith,
   same standard every number in this project has been held to). Expect
   to land within ~1.4% of the theoretical 32,768-cycle MAC-bound floor —
   if the real result is far off, that's a signal to double check the
   model, not just accept it.

**Honest scope note from the audit, worth remembering**: this is *not*
adoptable for free the way the shadow register was (that was a pure
timing-model correction, zero new ISA bits). This genuinely adds
encoding — expect real bundle-width growth and a real (if likely modest)
hazard pass, not a one-line fix.

**On the automated scheduler**: status is now open, not a fixed
requirement — `spec.md`'s two-deliverable framing for Phase 2 is being
treated as a starting point, not binding (`notes.md` §11.30). Worth
revisiting after this extension, not before. See the section below if/
when that happens — it's unchanged and still accurate, just not the
immediate next step.

## Starting fresh on the automated scheduler — read this section if picking up the scheduler instead of/after the ISA extension above

Everything the hand-schedule work derived, consolidated in one place so
you don't need to read all of §11's history to get moving. The dependency
graph and resource model below are **settled, not open questions** —
don't re-derive them; the automated scheduler's job is to search over
*ordering choices* within these fixed constraints, not to re-litigate the
constraints themselves. Note: if the ISA extension above has been adopted
by the time you read this, the resource model and target cycle count
below are stale — check `bundle-schedule-v4.md` and `isa.md`'s current
state first.

**Files to actually read**: `isa.md` (the ISA — 9 instructions, 3 slots:
matmul-issue/SFU/DMA), `phase2-loop.md` (the loop nest to schedule — one
Q-tile's steady-state body), `bundle-schedule-v3.md` (the hand-scheduled
answer to match/beat). `notes.md` §11 has the full derivation if you need
to double-check something below, but shouldn't be required reading to
start.

**Resource model** — one instruction issues per cycle per slot (3 slots,
in-order each), occupancy ≠ latency for two instruction types:

| Instruction | Slot | Latency | Occupancy | Gating on issue |
|---|---|---|---|---|
| `load-stationary` | matmul-issue | 128 | 128 | Prior stream's *occupancy*-end (not full latency — shadow register adopted, confirmed real in Gemmini's RTL, `notes.md` §11.26) |
| `steady-state-stream-qk`/`-v` | matmul-issue | 159 | **32** | Prior same-slot instruction's occupancy-end |
| `softmax-update`/`-finalize`/`metadata-init` | SFU | 32 | 32 | Prior SFU instruction's occupancy-end, and its own data dependencies (below) |
| `load-K/V` | DMA | 160 | 160 | Prior DMA instruction's occupancy-end, and WAR hazard (below) |
| `load-Q`/`store-O` | DMA | 5 | 5 | Same |

**Data dependencies / hazards already resolved** (don't re-derive, just encode):
- **S** (raw accumulator, ×4 buffered, `head_idx & 3` selects buffer):
  `steady-state-stream-qk(head i+4)`'s write must not start before
  `softmax-update(head i)` finishes reading — same-parity heads only.
- **P** (scratchpad, ×8 per-head): `softmax-update(head i)` of slice `N+1`
  must not write P[head `i`] before `steady-state-stream-v(head i)` of
  slice `N` finishes reading it (WAR, cross-slice).
- **O** (accumulator, ×8 per-head): `softmax-update(head i)` must precede
  `steady-state-stream-v(head i)` for the *same* head (rescale-then-add
  ordering) — per-head only, different heads never conflict.
- **metadata** (`m`,`l`, ×8 per-head): ordinary RAW chain across chunks,
  automatically satisfied by correct program order on the in-order SFU
  slot — needs no special scheduling logic beyond emitting instructions in
  dependency order. `metadata-init(head i)` must precede that head's first
  real `softmax-update` in the Q-tile (recurrence base case).
- **DMA double-buffering** (K1/K2/V1/V2): a `load-K/V` targeting a given
  buffer instance must not start before that instance's *last* reader
  (`load-stationary` for the chunk currently occupying it) finishes (WAR).
- **Array stationary registers — NOT automatically safe from slot
  serialization, encode this explicitly.** True in v1/v2 (occupancy=latency
  for streams, so slot serialization implied register safety for free);
  **false under v3's shadow register**, which is precisely the point of
  adopting it — `load-stationary` can now issue while a *different* prior
  stream is still draining out of the other physical register. The real
  constraint: `load-stationary` writing register X must not begin before
  X's *last reader* — the stream phase *two* phases back, not one — has
  fully drained (full latency, not occupancy). Confirmed margin in the
  hand-schedule: 257 cycles, safe but not automatic — a naive
  slot-serialization-only scheduler could produce a schedule that violates
  this, so it must be its own explicit constraint, not assumed to follow
  from the single-issue-slot property. Caught by the audit in §11.29,
  not present in any hazard list before that.

**Scope, already decided** (`notes.md` §11.17) — don't build more than
this: one basic block (the steady-state body — no control flow, no
branches in this ISA), no register allocation, no OOO/dynamic modeling
(ruled out since Phase 0's Decision 2). **A greedy list-scheduler over the
dependency graph above is sufficient.** No modulo-scheduling / software-
pipelining machinery needed — cross-iteration pipelining was evaluated and
explicitly dropped (negligible savings, item 1 above), so there's no
loop-carried scheduling problem to solve.

**Target**: **49,476 cycles** for one Q-tile (`bundle-schedule-v3.md`).
Prediction on record, worth checking against once built: expect the
automated result to land very close to this, since matmul-issue is the
near-total bottleneck throughout and SFU/DMA are idle because there's
genuinely nothing for them to do, not from scheduling slack — if the
automated scheduler lands far off from 49,476 in either direction, that's
itself a signal something in the model or the implementation is off,
worth checking before trusting the number.

**Genuinely open, not yet decided — ask the user rather than assume**:
implementation language/tooling, where the code should live in this repo,
and how success is validated (presumably: run it, compare its cycle count
and bundle sequence against `bundle-schedule-v3.md`). None of this is
specified anywhere in `spec.md` or prior sessions.

**Working style, carries over unchanged**: sounding-board role, not
answer-giver (see "How to work with me" at the top of this file) — but
note that *building* the scheduler is closer to execution than to a
derivation the user needs to walk themselves through (unlike the hand-
schedule's own hazard/timing derivations, which were genuinely the user's
learning target). Real *design* decisions within the scheduler (priority
heuristic, tie-breaking, how ties in the dependency graph get resolved)
are still worth surfacing as choices rather than silently picking one,
consistent with how Phase 0/1's two 🧠-marked decisions were handled.

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
  fixes + prologue/epilogue + slice-0/chunk-boundary derivation (original,
  pre-occupancy-fix schedule), §11.13–§11.16 the original bundle-table
  transcription, §11.17 the evaluated roadmap decision, §11.18 the
  independent audit that found the occupancy bug, §11.19 the decision to
  fix it, §11.20–§11.24 the occupancy re-derivation — occupancy table,
  S ×4, P/O/metadata/DMA hazard re-pass, `bundle-schedule-v2.md`'s
  derivation, the tail's SFU-saturation finding, §11.25 v2 status, §11.26
  the shadow-register investigation (real, confirmed in Gemmini's
  `PE.scala` RTL), §11.27 the "cost→0" overstatement caught and corrected,
  §11.28 the hazard re-pass under the shadow-register model +
  `bundle-schedule-v3.md`'s derivation — **current real pickup point:
  §11.28's closing note — the automated scheduler is the one remaining
  Phase 2 item**)
- `phase2-loop.md` — the Phase 2 loop nest actually being hand-scheduled
  (one Q-tile's steady-state body), kept separate from `notes.md` so it's
  easy to reference by line number
- `bundle-schedule.md` — **v1, superseded.** Built against the occupancy
  model later found wrong (busy-for-full-latency, uniform across all
  three slots). **179,397 cycles.** Kept as the before/after baseline.
  Derivation: `notes.md` §11.13–§11.16.
- `bundle-schedule-v2.md` — **v2, superseded.** Fixed the occupancy model
  (32-cycle stream occupancy, S at ×4) but still fully serializes
  `load-stationary`. **65,605 cycles** — 2.73× faster than v1. Kept as the
  before/after baseline for the shadow-register fix specifically.
  Derivation: `notes.md` §11.19–§11.24.
- `bundle-schedule-v3.md` — **v3, current.** `spec.md`'s literal Phase 2
  hand-schedule deliverable. Adds the `load-stationary` shadow register
  (confirmed real in Gemmini's RTL): same five-segment structure, all real
  cycle numbers. **Total Q-tile latency: 49,476 cycles** — 3.63× faster
  than v1, 1.33× faster than v2, 1.51× above the theoretical 32,768-cycle
  MAC-bound floor (a real, structural remainder — `load-stationary`'s own
  occupancy, not further reducible without a separate weight-load port).
  Derivation: `notes.md` §11.26–§11.28.
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
