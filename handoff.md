# Handoff — start here in a new chat

This project is a **pedagogical exercise**, not a "get it built" task. Read
this whole file before doing anything else, then pick up at Phase 1.

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

## Files in this repo

- `spec.md` — the project spec (source of truth for phase structure/
  deliverables; has been updated once already, e.g. Phase 2/3 restructured
  to require both a hand-scheduled and an automated bundle sequence)
- `isa.md` — **the Phase 1 deliverable**: clean, final instruction-format
  spec, all 8 instructions, no narrative. Start here for Phase 2.
- `notes.md` — full working log: Phase 0 (reading summaries, Decision 1/2
  derivations, open threads) + Phase 1 (§5–§7, slot-by-slot ISA design
  *including every correction and dead end*, not smoothed over)
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
