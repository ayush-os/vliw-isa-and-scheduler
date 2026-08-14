# Phase 0 Notes — Reading + the Two Real Decisions

Working log for Phase 0 (setup + reading + Decision 1 + Decision 2), kept live
during the actual back-and-forth rather than written up after the fact.
Mirrors the Prediction/Log style `workload-to-silicon/prefill_notes.md` used
before its own final polish pass — nothing here has been through that polish
yet.

---

## 1. Reading

### 1.1 Itanium/EPIC — why static scheduling broke (mechanistic, not business-history)

- **Predication overhead**: if-converted branches issue *both* sides to
  functional units — only write-back is suppressed for the untaken path.
  Real issue slots/energy burned on discarded work. Measured: if-conversion
  killed ~29% of mispredictions on SPEC2000CINT but delivered a much smaller
  overall speedup than that implies, because in-order memory stalls were
  already dominating and shadowing whatever branch-misprediction cost
  existed.
- **Unpredictable memory latency — the deeper break**: compiler picks one
  fixed hoist-distance for a load. Schedule for the common case (L2 hit) →
  every L3/DRAM miss stalls the whole in-order pipeline (no OOO to hide it).
  Schedule for worst case → wastes registers/issue slots on every hit. No
  single static distance is right for both the common case and the tail.
  Itanium's mitigation (`ld.a`/ALAT — Advanced Load Address Table — plus
  `ld.s`/NaT-bit control speculation) fixes *legality* of hoisting past
  aliasing stores/faulting derefs, not the *distance* problem. Quantified
  elsewhere: "flea-flicker" two-pass in-order pipelining (a later research
  fix) only recovered 32–67% of full-OOO's achievable speedup.
- **Genuinely useful, non-obvious data point**: Itanium's own SPECfp
  (dense numeric loop nests) was "phenomenal"; SPECint (pointer-chasing,
  data-dependent branches) was "very weak" — losing even to Intel's own
  Xeon. Itanium's own benchmark split is direct empirical evidence *for*
  this project's thesis: the static bet won exactly where the code looked
  like an attention kernel and lost where it didn't.
- **The one part that worked as intended**: rotating register files for
  software-pipelined loops (`br.ctop` auto-slides live values through
  prologue/kernel/epilogue) — a direct ancestor of software pipelining,
  and the mechanism behind Itanium's SPECfp strength. A real positive
  precedent to cite, not just failure analysis.
- **Compilers never delivered**: Knuth — the required compilers turned out
  "basically impossible to write." The compiler had to jointly solve
  if-conversion + speculation + scheduling distance + register allocation
  as one interacting NP-hard problem; real server code left bundles
  28–32% NOP-padded.

### 1.2 TI C6000 / Qualcomm Hexagon — why the DSP bet succeeds

- **The mechanism is a clean two-part fix to Itanium's two failures**:
  fixed trip count + no data-dependent branching kills the predication tax;
  affine, statically-known access patterns let a DMA engine stage data into
  scratchpad/SRAM instead of a cache, so load latency becomes a hardware
  constant the compiler can trust rather than a guess.
- **C6000**: 8 functional units across two symmetric datapaths (.L/.S/.M/.D
  ×2), no scoreboard/interlock hardware at all — every instruction has a
  fixed, documented latency and the compiler is solely responsible for
  filling delay slots or inserting NOPs. L2 is configurable as cache *or*
  directly-addressed SRAM; hot loops pin the working set in SRAM fed by
  double-buffered EDMA transfers specifically to get deterministic latency.
  `SPLOOP` (hardware loop buffer) builds one steady-state software-pipelined
  iteration at a fixed, statically-computed initiation interval.
- **Hexagon**: 4-wide VLIW packets, 8 MB TCM (Tightly Coupled Memory) —
  software-managed scratchpad, explicitly built to avoid tag-comparison
  overhead for gather/scatter vector workloads.
- **Real, load-bearing complication (worth being honest about)**: Hexagon
  doesn't rely on pure regularity alone — it hedges with cheap hardware
  multithreading (switch threads on an L2 miss) specifically to cover
  latency that even TCM staging doesn't eliminate. Not OOO, not dynamic
  scheduling within a thread, but a real admission that "purely static"
  wasn't quite sufficient even in the DSP success case. Open question
  carried into Phase 1/2: does this project's design need (or explicitly
  reject) an analogous escape valve?
- Both are live, shipping, high-volume products today (Hexagon in every
  Snapdragon SoC; C6000 in telecom/industrial/medical infra) — not legacy
  curiosities.

### 1.3 Groq — the AI-inference-extreme version, ISA-level detail

- **ISA-exposed latency, literally**: every instruction carries `d_func`
  (fixed cycle latency) and `d_skew` (operand-arrival timing). Compiler
  placement is a plain equation: `T = N + d_func + δ(j,i)`, not a
  heuristic. Zero-runtime-variance is the explicit design target, and it's
  measured, not just claimed: BERT-base batch-1 p99 latency was 0.4% over
  the mean on Groq vs. a 25% p99 blowup on an A100 from cache contention.
- **The genuinely new idea beyond ordinary VLIW — 2D (temporal *and*
  spatial) scheduling**: an ordinary VLIW bundle only decides *which slot,
  this cycle*. Groq's compiler also decides *where in a spatial mesh* a
  value physically is — data moves as fixed-direction "streams" through a
  functionally-sliced grid with no arbiters; spatial placement is compile-
  time coordinate arithmetic, not routing. Groq's own architect calls the
  144 lockstep dispatch paths **"144-wide VLIW instructions"** — a direct,
  source-backed self-comparison.
- **What's removed vs. a GPU/CPU**: no branch predictors, no OOO, no cache
  hierarchy (flat 220 MiB on-chip SRAM, directly addressed by the
  compiler), no arbiters/dynamic routing. ICU (instruction dispatch) is
  <3% of die area. Claimed payoff: that area becomes ALUs instead —
  >1 TOPS/mm² at 14nm, ~5× computational density vs. contemporary
  GPU/TPU designs.
- **Why AI inference fits**: a compiled model's forward-pass dataflow graph
  is the same DAG every invocation — no data-dependent branching analogous
  to general-purpose code's `if (x[i] > 0)`. Exactly the property Itanium's
  actual target workload (general C/C++, pointer-chasing) lacked.
- **Real, load-bearing complication (worth being honest about)**: Groq's
  purity is maintained by **fixed-shape kernels per sequence-length
  bucket** — variable-length inputs get padded/bucketed to a discrete set
  of precompiled programs, not handled with runtime flexibility. Direct
  implication for this project: claiming "static scheduling works for
  attention" likely needs the same move (fixed tile sizes per bucket), and
  that should be stated explicitly rather than assumed away by the loop
  nest's already-fixed tile sizes.

### 1.4 Saman Amarasinghe — Ken Kennedy Award lecture, "Compiler 2.0"

- **Stagnation context** for the spec's "pre-4GL years" framing: x86 has
  ~3,600 instructions, LLVM natively generates ~1,000. Intel's AVX ADX
  instruction (2004) took **10 years** to get an LLVM codegen path. Compiler
  backends structurally lag hardware releases by years — for a brand-new
  custom ISA, that lag isn't a temporary inconvenience, there is no
  existing backend to lag on at all.
- **ITEMAL — directly load-bearing for Phase 3**: hand-built analytical
  cost models (LLVM-MCA, Intel's own IACA) run ~20% error against real
  hardware, built from incomplete/contradictory vendor manual text, and go
  stale (the model LLVM used for a while was built for Sandy Bridge/early
  AMD). ITEMAL's fix: train an LSTM directly on millions of *measured*
  basic-block executions on real silicon → ~2× more accurate, ports to a
  new architecture in ~days of data collection. In real production use in
  Google's XLA today.
  - **Why this project starts from a worse position than even ITEMAL's
    20%-error baseline**: ITEMAL's fix requires real silicon to measure
    against. This project's Phase 1 hardware latencies are hand-derived
    from `prefill_notes.md`, never measured on real hardware at all. Phase
    3's "is the automated scheduler working from an inaccurate cost model"
    question has no empirical check available — worth stating as a real
    limitation, not glossing over.
- **Vegan — direct methodological precedent for Phase 1→2**: auto-generates
  a full compiler backend for x86 AMX directly from Intel's own
  pseudocode-annotated architecture manual, no hand-written backend. Found
  real bugs in Intel's own published pseudocode (nobody had ever mechanically
  executed it before). On a real TVM kernel, found and used a dot-product
  instruction that hand-written ICC/GCC had missed for *years* after it
  shipped — formal-description-driven generation beating hand-tuned
  production compilers outright. Legitimate precedent for "formalize the
  ISA precisely (Phase 1), then generate/schedule from that formal
  description (Phase 2)" as a real methodology, not just intuition.
- Smaller, non-load-bearing but interesting: 2003 genetic-programming
  result — learning *when* to if-convert a branch (hyperblock selection)
  beat hand-written heuristics by 23%, with zero interpretability. Early,
  narrow evidence that even a single scheduling *decision* (not a whole
  scheduler) can beat hand-tuning — flavor for Phase 3, not proof of
  anything at the scale of "hand-schedule vs. automated schedule."

### 1.5 Gemmini's own ISA (contrast case)

Already fully covered via `workload-to-silicon/prefill_notes.md` §4
(Phase 1d) — re-read, not re-sourced. Host-core-issued (Rocket/BOOM),
sequential single instruction stream, custom opcodes (`mvin`/`mvout`/
`compute`/etc). Key facts already on hand and directly reusable for Phase 1
ISA decisions:
- §4.6: `dataflow=WS` is a **compile-time Chisel literal**
  (`current_dataflow = dataflow.id.U` when `dataflow != BOTH`), not a
  runtime-selected field — only `Dataflow.BOTH` generates a real runtime
  register. Direct implication: this project's matmul-issue slot doesn't
  need a runtime dataflow-select field at all.
- §4.5: Gemmini has a native `SOFTMAX` activation code and a `Normalizer`
  module (running max/sum registers, max-subtracted `iexp`, hardfloat
  `1/sum` divide) — independently confirms the online-softmax mechanism
  `prefill_notes.md` §2.3 derived by hand before knowing the Flash
  Attention name for it.

---

## 2. Decision 1 — which machine: prefill

**Chosen: prefill.** Reasoning (confirmed against `prefill_notes.md` §2.4
and `decode_notes.md` §2.7):

- Prefill has **three architecturally separate issue targets**: the
  128×128 systolic array (matmul), a softmax vector/scalar unit (SFU), and
  DMA. Three genuinely heterogeneous units, none sharing a datapath.
- Decode has only **two**: a SIMD/vector engine (32 lanes) and DMA. The
  per-`d_head` reduction inside each lane (adder tree vs. serial — left
  unresolved in `decode_notes.md` §2.2) lives *inside* the SIMD unit, not
  as a separate slot type — so decode's ceiling on bundle-slot richness
  isn't "only 2 things to manage," it's that one of its two things is a
  single monolithic unit where prefill's three are structurally distinct.
- This directly matches the spec's own stated reasoning: three
  heterogeneous unit types gives VLIW's bundle-slot idea more to actually
  do.
- Decode stays flagged, per spec, as a cheap future "second machine, same
  ISA family" extension — not required for this project.

---

## 3. Decision 2 — the Itanium-vs-Groq argument, derived against the real prefill loop nest

Walked the online-softmax tiled loop from `prefill_notes.md` §2.3/§2.5
(fixed `tile_q=32`/`tile_k=1024`, K/V-stationary across the 8-head GQA
group) against the three checks the spec requires. All three: **yes**,
with real caveats — the caveats are the actual content, not hedging.

1. **Are all trip counts known at compile time? Yes — but only under a
   fixed-shape idealization.** Batch=32, seq_len=8192 fixed → every loop
   level (batch, KV-group, Q-tile, K/V-chunk, head, array sub-pass) has a
   compile-time-known trip count. This is not a free fact: it holds only
   because shape is fixed. Real serving has variable batch/seq_len; the
   Groq precedent (§1.3 above) is the direct playbook — pad/bucket to a
   discrete set of precompiled fixed-shape programs. Worth stating
   explicitly in Phase 1/4 rather than leaving "what happens at a
   different seq_len" unanswered.

2. **Does any instruction's latency depend on runtime data? No — and the
   *mechanism* is specific, not just asserted.** DMA loads are
   scratchpad-via-explicit-DMA (`prefill_notes.md` §2.1/§2.3), the same
   TI/Hexagon pattern (§1.2 above): the compiler's DMA instruction
   *schedules* the fetch ahead of time — it is not triggered reactively by
   a cache miss the way Itanium's loads were. That's the actual reason
   latency is compile-time-knowable: it's not "DMA is fast," it's "DMA is
   scheduled, not requested reactively," which is the same distinction
   that separates a scratchpad from a cache. Matmul-array latency is
   likewise fixed (array cycle behavior, known at compile time, no dynamic
   dataflow-mode select per §4.6).

3. **Is there any data-dependent branching in this loop nest? No — and
   §2.5's tile_q/GQA-reuse tension is the right example to cite, not a
   counterexample.** The forced Q-tile-outer loop order (K/V chunks
   re-fetched ~256× instead of once per group) is *still* fully static —
   the re-fetch count is fixed and known at compile time, not runtime-
   dependent. But it's a concrete demonstration that "fully static" is not
   the same as "free" or "obviously optimal" — the real fetch cost only
   surfaced from actually walking the loop nest carefully in Phase 1b, not
   from asserting determinism in the abstract. Directly useful precedent
   for why Decision 2 needed the derived walk rather than the one-line
   assertion.

**Net**: static scheduling is the right bet for this specific loop nest,
for reasons that are mechanistically the DSP-success-case reasons (§1.2)
and the Groq reasons (§1.3), not borrowed assertion — with two caveats
carried forward as open threads: (a) fixed-shape is an idealization
requiring bucketing to generalize, (b) "fully static" still requires real
scrutiny to find its actual costs (§2.5-style), it doesn't grant them away
for free.

---

## 4. Open threads carried into Phase 1+

- Hexagon's SMT-as-latency-hedge (§1.2): does this project's design need,
  or explicitly reject, an analogous escape valve for residual
  unpredictability? Current default: reject (spec's Note on Scope
  explicitly rules out modeling any dynamic/OOO behavior) — but worth one
  explicit sentence in Phase 1 or Phase 4 stating this is a deliberate
  choice, not an oversight.
- Fixed-shape idealization (§1.3, §2 Decision 2 point 1): if Phase 4
  synthesis wants to address realistic serving, the honest answer is
  shape-bucketing, not dynamic scheduling — consistent with the project's
  own static-only thesis.
- Phase 3's cost-model-accuracy question (§1.4 ITEMAL): this project has
  no real-silicon measurement to check hand-derived latencies against —
  flag explicitly in Phase 3 rather than silently assuming the hand-derived
  numbers are trustworthy.

---

## 5. Phase 1 — ISA Definition: Matmul-Issue Slot

First of the three Phase 1 slot types (matmul-issue, vector/scalar-unit,
DMA — `spec.md`'s structure). Mostly user-derived with real-time checking/
pushback, per the project's stated working style — including one genuine
dead end that got recognized and abandoned, not forced.

### 5.1 Two instruction types, not one

Grounded in §2.1 (WS dataflow, K/V-stationary) + §2.5 (forced Q-tile-outer
loop order — within one Q-tile, the stationary K/V chunk in the array gets
swapped ~8× as the loop sweeps the group's chunks). "Load a new stationary
operand into the array" and "stream the current Q/P tile through whatever's
already loaded" are structurally distinct operations — the same split real
Gemmini makes (`PRELOAD` vs `COMPUTE`), independently re-derived here
rather than copied from it.

- **load-stationary**: moves a chunk of K or V from scratchpad into the
  array's stationary registers.
- **steady-state-stream**: streams the current Q or P tile through the
  array (whatever's currently loaded as stationary), writing results to
  the accumulator.

### 5.2 Field list, settled

- **opcode**: which of the two instruction types.
- **load-stationary**: source scratchpad address (K/V chunk). No length
  field — the load is always exactly 128 wide, the array's own physical
  PE-row count, a hardware constant (not a workload/tiling choice), even
  more rigidly fixed than `tile_q`.
- **steady-state-stream**: source scratchpad address (Q/P) + destination
  accumulator address. No length field — always exactly `tile_q`=32 (fixed
  under the Decision 2 fixed-shape idealization; `seq_len_q`/`tile_q` =
  8192/32 = 256 divides evenly, no ragged remainder to worry about).
- **No dataflow-select field**, confirmed still applicable (§4.6, carried
  over from Phase 0).
- **No transpose bits** — see §5.3, this took real derivation (and one
  real dead end) to land on.

Bit-widths for the address fields deliberately **deferred** to a single
pass across all three Phase 1 slots at the end (matmul-issue + vector/
scalar + DMA together) rather than picked per-slot, since all three slots'
addresses draw on the same real scratchpad/accumulator capacity numbers
(§2.3/§2.4) and picking widths in isolation risked inconsistency.

### 5.3 Transpose-bit derivation (real back-and-forth, real dead end, real resolution)

Grounded in §4.7's real Gemmini finding: one physical array serves both
QK^T and ·V via a hardware transposer, gated by software-visible flags,
because a matrix's row-major scratchpad storage doesn't always match the
order the array wants to consume it in.

- **First conflation, caught and corrected**: the `^T` in "QK^T" (math —
  which dim contracts) is not the same thing as the hardware transpose bit
  (data layout vs. array feed order, "independent of programmer intent"
  per §4.7's own wording) — a matrix can need the hardware transposer for
  reasons that have nothing to do with whether the math has a `T` in it.
- **Dead end**: tried to pin the bit count down via the real Chisel
  equations (`a_transpose`, `bd_transpose` are software-controlled; a
  third port is hardwired `false` always under WS). This bounds the
  hardware at ≤2 software-controllable bits total — a fact that *is*
  provable, no mapping knowledge required — but assigning those bits to
  our two instructions requires knowing which real Gemmini port (A/B/D)
  maps to which of our four operands (Q/K/P/V), and that mapping was
  **never verified** anywhere in this project — `prefill_notes.md` §4.7's
  own scope caveat and §6's open thread both say this exact question was
  deliberately left unchecked (would need a real kernel, descoped in
  Phase 1d). Spent real time circling on this before recognizing it as a
  dead end rather than a puzzle to push harder on.
- **Resolution — genuine first principles, independent of the unverified
  port mapping**: using only two already-established facts (row-major
  storage, §4.7; the array wants "same-row elements entering the same PE
  sequentially rather than adjacent PEs simultaneously," §4.7's own
  phrasing) —
  - **load-stationary** (K/V): the stationary operand is *by definition*
    spread spatially across many PEs at once — that's what "stationary"
    means physically. A row's elements landing in different PEs in the
    same cycle always mismatches row-major's single-contiguous-stream
    nature. Transpose is **permanently engaged** — a hardwired fact of
    this instruction, not a runtime choice.
  - **steady-state-stream** (Q/P): the moving operand is *by definition*
    fed into one PE, one element per cycle, over time — exactly what
    row-major storage naturally provides on readout. Transpose is
    **permanently disengaged**.
  - Same shape as the §4.6 dataflow-is-a-compile-time-constant finding:
    something true 100% of the time doesn't need a software field, it's
    just hardware behavior.
- **Final: 0 explicit transpose bits, either instruction.**
- Visual reference built to nail this down for future recall:
  [**Fan-Out vs. Funnel**](https://claude.ai/code/artifact/b8690974-a2a6-4616-987a-e581ea3a81dd)
  — animated diagram, one row-major memory row routed two ways (fan-out to
  4 PEs in 1 cycle vs. funnel to 1 PE over 4 cycles).

### 5.4 Process note for next session

The "check, don't hand over the derivation" mode (per `handoff.md`) works,
but watch the calibration: pushing back with more open questions instead
of stating what's actually known — especially across several turns on the
same sub-question — reads as cryptic/evasive rather than Socratic, and got
explicit direct user feedback on this mid-session. When a claim's already
been checked once and a follow-up is genuinely just asking for the
concrete mechanism in plain language, give it plainly. When directly asked
"what do you recommend" or "give me a final decision," give one, directly
— that's not the same as doing the derivation *for* them unprompted;
asking for a recommendation on a judgment call is a different request
than skipping their own derivation entirely.
