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

## Phase 1 progress: matmul-issue slot done, vector/scalar-unit slot next

Per `spec.md` Phase 1 — three slot types for prefill, each grounded in a
specific prior finding, not invented from scratch. Full working log for
everything below: **`notes.md` §5** — read that before continuing, this
file is just the pointer + status.

**Matmul-issue slot: done.** Two instruction types (`load-stationary`,
`steady-state-stream`), grounded in §2.1/§2.5's forced K/V-chunk-swap
pattern. Settled fields: opcode, source scratchpad address for each
(+ destination accumulator address for `steady-state-stream`), no length
fields (both lengths are hardware/workload constants — 128 = array's
physical PE count, 32 = `tile_q`), no dataflow-select field (§4.6), **and
a real derived result worth remembering: zero explicit transpose bits in
either instruction** — both instructions' transpose behavior turned out to
be a fixed hardware fact (stationary operand always fans out across many
PEs in one cycle → mismatches row-major storage → transpose permanently
on; moving operand always funnels into one PE over many cycles → matches
row-major → transpose permanently off), the same shape as the §4.6
dataflow-is-a-compile-time-constant finding. Full derivation, including a
real dead end (tried to resolve it via Gemmini's real A/B/D port
semantics, which turned out to be genuinely unverifiable from
`prefill_notes.md`'s own stated scope, §4.7/§6) and the eventual
first-principles resolution that didn't need that mapping at all: `notes.md`
§5.3. There's also a visual reference from this derivation — [**Fan-Out vs.
Funnel**](https://claude.ai/code/artifact/b8690974-a2a6-4616-987a-e581ea3a81dd),
an animated diagram of the fan-out-vs-funnel mechanism — worth a look if
picking this back up cold.

Bit-widths for matmul-issue's address fields are **deliberately deferred**,
not forgotten — planned as one pass across all three slots' addressing
needs together at the end of Phase 1, once vector/scalar and DMA are also
defined, so widths get chosen consistently against the real scratchpad/
accumulator capacities rather than piecemeal per slot.

**Next up: vector/scalar-unit slot** — the softmax sequence (max/exp/sum/
normalize), using the online-softmax algorithm from `prefill_notes.md`
§2.3/§4.5 (independently derived, then confirmed against Gemmini's real
`Normalizer` module).

**Then: DMA slot** — tile load/store, sized against the real
`tile_k=1024`/`tile_q=32` scratchpad budget (`prefill_notes.md` §2.3).

**Then: the deferred encoding pass** — address-field widths (sized to the
real scratchpad capacity, not arbitrary) and immediate operand widths for
all three slots together, plus fixed-width-per-slot vs. compact variable
bundle layout (state + defend the tradeoff, same "primary hypothesis +
explicit alternate" discipline `prefill_notes.md` §2.1 used for array
width).

**Deliverable**: a real, written instruction-format spec — concrete enough
that Phase 2 (now split into hand-scheduled *and* automated bundle
sequences, both required — see `spec.md`, this was updated after Phase 0)
has something unambiguous to target.

## Files in this repo

- `spec.md` — the project spec (source of truth for phase structure/
  deliverables; has been updated once already, e.g. Phase 2/3 restructured
  to require both a hand-scheduled and an automated bundle sequence)
- `notes.md` — full working log: Phase 0 (reading summaries, Decision 1/2
  derivations, open threads) + Phase 1 (§5 onward, slot-by-slot ISA design)
- `handoff.md` — this file
- `../workload-to-silicon/prefill_notes.md` — the real hardware hypothesis
  this ISA targets (128×128 array, WS dataflow, online-softmax, scratchpad
  sizing) — sibling repo, not inside this one
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
