# Project Spec: VLIW ISA + Scheduler — Giving Project #1's Hardware a Real ISA

**Continuity note:** project #7. Targets your own attention hardware
hypothesis (`prefill_notes.md`, Phase 1b/1d) directly — the 128×128
systolic array, K/V-stationary WS dataflow, the online-softmax unit
independently derived by hand and then confirmed against Gemmini's real
`Normalizer` module, and the real tile_k=1024/tile_q=32 scratchpad sizing.
That hardware never got a real instruction set — Gemmini's actual ISA is a
given, external thing you used, not something you designed. This project
designs one, then writes a real (if minimal) scheduler that targets it
against your own already-derived loop nest.

**The core bet, and why it's not Itanium's mistake.** VLIW's whole premise
is exposing instruction-level parallelism to the compiler instead of
discovering it in hardware at runtime (no OOO, no dynamic scheduling) — a
bet that famously failed for Itanium, where general-purpose code's
unpredictable branches and memory latency defeated static scheduling. Groq's
real, current production architecture is the strongest live counter-case:
their ISA explicitly exposes each instruction's *latency*, the compiler
pre-schedules execution down to the clock cycle (both temporal *and*
spatial), and the hardware is deliberately simplified — no branch
predictors, no cache controllers, no OOO — specifically because AI inference
kernels are regular, predictable, loop-nest-shaped work, not general-purpose
control flow. Your own attention loop nest (fixed trip counts, no
data-dependent branching, every instruction's latency knowable at compile
time) is exactly the regime where this bet should hold. Don't just assert
this — Phase 0 makes you derive it against your own workload specifically.

**Legend:** 🔧 = boilerplate/setup. 🧠 = your job.

---

## Phase 0 — Setup (🔧, two real 🧠 decisions)

### Reading (🔧)

- **Itanium/EPIC** — the real failure case. Understand specifically *why*
  static scheduling broke: predication overhead, unpredictable memory
  latency, general-purpose branch unpredictability defeating the
  compiler's schedule.
- **TI C6000 / Qualcomm Hexagon** — real, currently-shipping VLIW DSPs.
  The success case: regular, predictable DSP kernels are exactly where the
  static-scheduling bet holds. Read for *why* these succeeded where Itanium
  didn't, not just that they did.
- **Groq's real architecture** (multiple current sources, e.g. groq.com's
  own technical writeups) — the extreme, AI-inference-specific version of
  this bet: ISA-exposed instruction latency, compiler-driven temporal *and*
  spatial scheduling, deliberately simplified hardware. Read specifically
  for how the ISA itself is structured to make static scheduling possible
  (not just that the compiler is smart).
- **Saman Amarasinghe's Ken Kennedy Award lecture ("Compiler 2.0")** —
  required, not optional. Direct, current, expert-level evidence for the
  real tension this project has to take seriously: on frontier hardware
  (Blackwell), engineers are hand-writing PTX today, precisely because
  compiler infrastructure lags hardware — "we have gone all the way back
  to pre-4GL years." His ITEMAL project's finding matters most for Phase 3:
  even mature analytical cost models (LLVM-MCA, Intel's own IACA) run
  ~20% error against real hardware, because they're built from incomplete,
  sometimes-wrong vendor documentation rather than real execution data —
  and a **brand-new custom ISA with zero historical profiling data** (what
  this project builds) should have this problem worse, not better. His
  **Vegan** project (auto-generating a compiler backend directly from an
  architecture's own precise instruction description) is close to a direct
  methodological precedent for this project's own Phase 1→2 structure —
  worth reading as validation of the approach, not just a data point on
  the "compilers are struggling" side.
- **Gemmini's own real ISA** (you already have this — `prefill_notes.md`
  Phase 1d, the RTL/source you already read) — re-read specifically as the
  *contrast* case: host-core-issued (Rocket/BOOM), sequential single
  instruction stream, custom opcodes (`mvin`/`mvout`/`compute`/etc.). This
  is your built-in "existing design, here's the alternative" comparison —
  no need to source a new one.

### 🧠 Decision 1: which machine — prefill or decode

Two real candidates, both already fully characterized in this repo:
**prefill's** hardware hypothesis (128×128 systolic array + softmax
vector/scalar unit + DMA — three genuinely heterogeneous unit types, the
richer bundle-slot design) or **decode's** (SIMD/vector engine, 32 lanes +
per-lane reduction + DMA, `decode_notes.md` §2). Recommend prefill as
primary — three heterogeneous units gives VLIW's bundle-slot idea more to
actually do than decode's more uniform SIMD-centric design. Decide and
state why; keep decode explicitly flagged as a cheap future "second
machine, same ISA family" extension (the bundle-slot *structure* would
carry over directly, given decode's hardware hypothesis is already
derived) — not required for this project.

### 🧠 Decision 2: the Itanium-vs-Groq argument, derived against your own workload

Don't just cite Groq's rationale — apply it. Walk your own prefill loop
nest (the online-softmax tiled loop from `prefill_notes.md` §2.3/§2.5:
fixed `tile_q=32`/`tile_k=1024`, K/V-stationary across the 8-head GQA
group) and check, explicitly: are all trip counts known at compile time?
Does any instruction's latency depend on runtime data (vs. Itanium's
unpredictable memory latency)? Is there any data-dependent branching
anywhere in this loop nest? A real, derived yes/no on each, not an assumed
one — this is the actual argument for why static scheduling is the right
bet *here*, grounded in your own already-derived hardware, not borrowed
from Groq's marketing.

**Checkpoint:** target machine chosen and defended; the Itanium-vs-Groq
argument derived against your own loop nest, not just cited.

---

## Phase 1 — ISA definition (🧠, the actual design work)

Define bundle width and slot types for your chosen machine. For prefill,
three real slot types, each grounded in a real prior finding — not
invented from scratch:

- **Matmul-issue slot**: tile coordinates into the scratchpad, addressing
  operands. Real, derived simplification available to you: `prefill_notes.md`
  §4.6 found WS-only dataflow is a **compile-time constant**, not a
  runtime-selected mode (the PE's `current_dataflow` field is a Chisel
  literal when `dataflow != BOTH`) — meaning your matmul-issue instruction
  doesn't need a runtime dataflow-select field at all. A real ISA decision
  that falls directly out of a finding you already made, not a fresh guess.
- **Vector/scalar-unit slot**: the softmax sequence (max/exp/sum/normalize)
  — use the real online-softmax algorithm you independently derived and
  then confirmed against Gemmini's actual `Normalizer` module (§4.5).
- **DMA slot**: tile load/store, sized against your own real
  `tile_k=1024`/`tile_q=32` scratchpad budget (§2.3).

**Real encoding decisions to make, not skip past**: address-field widths
(sized to your real scratchpad capacity, not arbitrary), immediate operand
widths, how the bundle itself is laid out (fixed-width fields per slot vs.
a more compact variable scheme — state and defend the tradeoff, the same
"primary hypothesis + explicit alternate" discipline `prefill_notes.md`
§2.1 used for array width).

**Deliverable**: a real, written instruction-format spec — not a full
production ISA manual, but concrete enough that Phase 2's scheduler has
something unambiguous to emit bundles against.

---

## Phase 2 — Hand-scheduled and automated bundle sequences, both required (🧠)

**Two deliverables, not one — this is the direct response to a real
tension, not hedging.** If your actual target company's real practice is
closer to "wrap the ISA, hand it to engineers" than "ship a full optimizing
compiler," the skill that matters most is producing a near-optimal schedule
*by hand* — reasoning directly about your Phase 1 ISA and the real hardware
latencies already derived in `prefill_notes.md`. That's deliverable one,
and it comes first: hand-schedule the real bundle sequence for your
prefill loop nest yourself, the same way a human would if handed this ISA
with no compiler at all.

**Then, and only after the hand-schedule exists**: build the automated
scheduler (CS243's real list-scheduling/software-pipelining algorithms,
reused directly — the gap was never the algorithms). Consuming the same
loop nest, the same real hardware constraints, targeting the same ISA.

**Deliverable**: both a hand-scheduled bundle sequence and an
automatically-scheduled one, for the identical loop nest and ISA — set up
specifically so Phase 3 can compare them honestly.

**Checkpoint:** ISA spec complete (Phase 1); both a hand-built and an
automated bundle schedule exist for your own already-derived attention
loop nest.

---

## Phase 3 — Three-way comparison: hand-scheduled, automated, and Gemmini's real serial ISA (🧠)

**The real question this project now has to answer, not assume**: does
your hand-scheduled bundle sequence beat the automated one, and if so,
why — mechanistically, not just "hand-tuning is better." Amarasinghe's
own ITEMAL finding gives you a specific mechanism to check for, not just
cite: is the automated scheduler's list-scheduling heuristic working from
an inaccurate cost model (no profiling data exists for a brand-new ISA,
the exact problem ITEMAL identified for real x86 cost models), and does
that inaccuracy actually show up as a real gap versus your hand-derived
latencies? If the automated scheduler matches or beats the hand-schedule,
that's also a real, legitimate finding — don't go in assuming hand-tuning
wins by default just because current industry practice leans that way for
frontier GPU hardware.

**Direct callback, don't re-derive from nothing**: `prefill_notes.md`
Phase 1c's own Timeloop results already flagged "a real DMA/compute-overlap
effect" mattering for dataflow choice — this phase is the natural
continuation of that finding, now with two real scheduled alternatives to
compare against Gemmini's actual sequential-issue ISA, not just one.

**Derive, don't assume, the full comparison**: cycle count under (a) your
hand-scheduled bundles, (b) your automated scheduler's bundles, (c)
Gemmini's real sequential-issue model (check its actual overlap capability
honestly, per the original Phase 3 guidance below — don't assume
worst-case serial). A real, honest three-way finding is required, stated
mechanistically — including the possibility that the ranking isn't what
either "compilers win" or "hand-tuning wins" would predict going in.

---

## Phase 4 — Synthesis (🧠)

Two real connections to make explicit:

1. **Direct relevance to your #1 target (MatX)**: their own postings name
   VLIW twice, alongside "define the architecture...including ISA." This
   project is close to a direct answer to that ask — say plainly what it
   demonstrates and where its real limits are (a minimal ISA/scheduler, not
   a production compiler).
2. **The Itanium-vs-Groq argument, closed out**: return to Decision 2's
   derived answer in light of what Phase 2/3 actually found. Did the static
   scheduling bet pay off the way the Groq comparison predicted, or did
   something in your own specific workload complicate it? A real answer,
   not a restatement of the Phase 0 hypothesis.

---

## Note on scope

Resist three real temptations: (1) a full, general-purpose instruction
set — stick to the three real slot types your own hardware hypothesis
actually has; (2) modeling any dynamic/out-of-order behavior — the whole
point is static scheduling, adding a dynamic fallback path defeats the
project's own thesis; (3) decode's SIMD machine as a required second
target — flagged in Decision 1 as a cheap future extension, not this
project's job to build.

## Fallback

Phases 0–2 (ISA definition plus a scheduler that emits a real bundle
sequence for your own attention loop nest) stand alone as a complete
artifact — the real deliverable (a designed ISA + working codegen against
it) exists there. Phase 3's quantified comparison and Phase 4's synthesis
are real additions, not prerequisites for a complete project.