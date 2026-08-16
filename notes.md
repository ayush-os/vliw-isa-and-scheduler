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
  **Superseded — see §6.3.** This single-instruction field list turned out
  to be wrong once the SFU slot's derivation forced a closer look: the
  destination isn't uniform across this instruction's two real uses. Kept
  here as the historical record of the original (incomplete) design.
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

### 5.5 Loop order within a Q-tile: chunk outer, head inner

Surfaced while working the vector/scalar-unit (SFU) slot's field list —
specifically, while checking whether `steady-state-stream`'s Q source
address could collapse to a single fixed/reused buffer the same way raw-S
turned out to (§6, once written up). Recorded here since it's a property
of the loop nest itself, not specific to one slot — it constrains both
`steady-state-stream`'s addressing (this section) and the SFU slot's.

**Finding**: within one Q-tile-iteration, the loop order is **K/V-chunk
outer, head inner** — not the reverse.

**Derivation**:
- §2.2's K/V-stationary GQA reuse mechanism means one `load-stationary`
  call (one K/V chunk into the array) gets reused across all 8 heads
  sharing that KV group before the array reloads a new chunk — that reuse
  *is* the mechanism, so for one loaded chunk, all 8 heads get processed
  before the next chunk load. Chunk is outer relative to head.
- Combined with §2.5's already-established "cycling all 8 chunks" within
  one Q-tile (Q-tile outer relative to K/V-chunk): the full nest for one
  Q-tile is **for each of 8 chunks: for each of 8 heads: process**.

**Consequence for addressing**: a given head gets revisited 8 separate
times within one Q-tile-iteration (once per chunk pass) before that Q-tile
finishes. So a head's Q data can't be treated as a single transient buffer
loaded once and discarded the way raw-S/P are (produced and consumed once
per sub-pass, then overwritten) — it has to stay resident across the
*entire* Q-tile-iteration. That means **8 simultaneously-live per-head Q
buffers**, addressed via the same 3-bit head-index as the per-head
accumulator state — not collapsible to a single fixed zero-bit address the
way raw-S/P are. Direct implication: `steady-state-stream`'s QK^T-mode
source address needs the head-index; only its destination (raw S) and the
·V-mode's source (P) collapse to zero bits.

*(Correction, §6.2: P does **not** actually collapse to zero bits — that
was wrong by analogy with raw-S. See §6.2 Correction 3.)*

---

## 6. Phase 1 — ISA Definition: Vector/Scalar-Unit (SFU) Slot, and Corrections to Matmul-Issue

Full derivation involved a long real back-and-forth with several genuine
self-corrections — recorded here rather than smoothed over, per this
project's own standard (`prefill_notes.md`'s own framing: "self-corrections
and wrong turns are kept where they carried real signal"). The corrections
turned out to be as load-bearing as the initial design.

### 6.1 Two instruction types, from the actual online-softmax recurrence

Standard online (Flash-Attention-style) softmax, per head, iterating over
the 8 K/V chunks `j=1..8` within one Q-tile. Initial state `m_0=-inf,
l_0=0, O_0=0`. Given `S_j = Q_tile · K_j^T` (produced by the matmul slot,
not SFU):

**softmax-update** (once per chunk):
```
m_j     = max(m_{j-1}, rowmax(S_j))
P_j     = exp(S_j - m_j)                    → scratchpad
alpha_j = exp(m_{j-1} - m_j)
l_j     = alpha_j · l_{j-1} + rowsum(P_j)
O_j     ← alpha_j · O_{j-1}                  rescale only — the += P_j@V_j
                                              add is steady-state-stream-v's job
```

**softmax-finalize** (once, after chunk 8):
```
O_final = O_8 / l_8
```

Matches real Gemmini's `Normalizer` (`prefill_notes.md` §4.5) split
exactly: running max/sum + max-subtracted `iexp` is update-side hardware,
the hardfloat `1/sum` divide is separate finalize-side hardware.

### 6.2 Field derivation, including the real corrections

Starting point (wrong): assumed `softmax-update` needed a full
`accum_src_addr` (raw S) + a full `scratchpad_dst_addr` (P).

**Correction 1 — "which head" needs a field, but not for the reason first
given.** Initial reasoning ("the loop is fixed, so no field needed") was
too broad — Decision 2's own point is that fixed/compile-time-known
doesn't mean operand-free (that's exactly what separates this project's
static bet from a naive one). The real mechanism: the persistent per-head
state (`m`, `l`, `O`) genuinely occupies 8 distinct addresses (§2.3's `×8`
sizing term), so a 3-bit head-index is real and necessary.

**Correction 2 — a naming trap hid the actual gap.** `accum_src_addr` was
quietly doing two different jobs across two instructions: on
`softmax-finalize` it correctly meant "the per-head state block" (does
vary by head). On `softmax-update` it actually meant "the raw-S buffer" —
which turned out to be a single fixed, reused accumulator location
(produced by `steady-state-stream-qk`, consumed by the very next
instruction, same head, no reload gap in between) — zero bits, not
per-head. So `update`'s real head-index need came from a field that had
gone unlisted (the per-head metadata), not from the field originally
assumed to carry it.

**Correction 3 — P is not fixed like raw-S.** Initially assumed (by
analogy with raw-S's "one reused buffer, not ×8" wording in §2.3) that P
also collapses to zero bits. Wrong: P's producer (`softmax-update`, during
the K-stationary phase) and consumer (`steady-state-stream-v`, during the
V-stationary phase) are separated by a hardware-forced reload boundary —
the array can only hold one stationary operand (K or V) at a time (§2.1's
shared-array finding), so all 8 heads' QK^T+update work finishes under
K-stationary before any ·V work can begin under V-stationary. All 8 heads'
P values must therefore coexist simultaneously, not one-at-a-time like raw
S. **Real correction to `prefill_notes.md` §2.3's own budget**: that phase
assumed a single reused 4,096 B P buffer when solving for `tile_k`; the
real requirement is `×8` = 32 KB (per-head P is `tile_q×128` = 4,096 B,
not the 1024×128 K/V-chunk size — a real number mix-up caught mid-session).
Checked against the 1 MiB ceiling: K/V's double-buffered term already uses
524,288 B + Q/output's 8,192 B = 536,576 B, leaving ~512 KB slack — the
extra 28 KB fits, `tile_k`=1024 still stands, but the original hypothesis's
P-sizing assumption was incomplete. (Not edited into `prefill_notes.md`
itself — that's a different, already-completed project's polished
deliverable; recorded here as where the gap was actually found.)

**Correction 4 — Q needs the same ×8 treatment as P, for a different
mechanistic reason.** Surfaced while checking whether
`steady-state-stream-qk`'s Q-source could also collapse to zero bits like
raw-S. It can't: K/V-stationary GQA reuse (§2.2) means one
`load-stationary` call is reused across all 8 heads before the array
reloads — so within one Q-tile, the loop order is chunk-outer/head-inner
(§5.5), and a given head gets revisited once per chunk (8 times) before
the Q-tile finishes. Q data has to stay resident the whole time, not get
discarded after one use — ×8 residency, head-indexed. (§2.4's own "(c)
K/V double-buffered, Q/output not" confirms Q doesn't need the extra
double-buffer bits `load-stationary` needed, just the head-index.)

**Correction 5 — `softmax-update`'s full read/write set, corrected late.**
Two more misses: `softmax-update` never *writes* S (only reads it — the
next `steady-state-stream-qk` call is what overwrites it, not `update`),
and `update` *does* touch the output accumulator (the `alpha_j · O_{j-1}`
rescale step), which was initially missed entirely.

- **Reads**: S (fixed, zero bits), metadata (`m`,`l` — head-idx), output
  (`O` — head-idx)
- **Writes**: metadata (head-idx), output (head-idx, rescale written in
  place), P (head-idx)

### 6.3 Bifurcation of `steady-state-stream` into two opcodes

Corrected from the original single `steady-state-stream` (§5.1/§5.2) once
the asymmetry above was traced through: its destination is zero-bit (raw
S) in QK^T-mode but head-indexed (output accumulator) in ·V-mode — not one
uniform field. Split into `steady-state-stream-qk` and
`steady-state-stream-v`. Both now happen to have identical field shape
(3-bit head-index only) — kept as separate opcodes anyway, because the
opcode itself is what tells the hardware which pair of hardcoded base
addresses (Q-base/S-base vs. P-base/output-base) to combine with that
index; the bit-width matching is coincidental, the operation and
addressing target genuinely differ.

### 6.4 Final field table, both slots

| Instruction | Fields |
|---|---|
| `load-stationary` | opcode + 5-bit src (2-bit buffer-select {K1,K2,V1,V2} + 3-bit slice-idx within the chosen 1024×128 double-buffered chunk) |
| `steady-state-stream-qk` | opcode + 3-bit head-idx (Q source; S destination fixed, zero bits) |
| `steady-state-stream-v` | opcode + 3-bit head-idx (services both P source and output destination, different hardcoded bases) |
| `softmax-update` | opcode + 3-bit head-idx (S read-only and fixed; metadata + output + P all head-idx'd) |
| `softmax-finalize` | opcode + 3-bit head-idx (reads metadata+output, writes output only) |

Bit-widths for the *base* addresses themselves (not the small index fields
derived here) remain deferred to the single combined pass across all three
Phase 1 slots, per §5.2's original plan — now to happen once DMA is
defined.

### 6.5 Correction: `softmax-finalize`'s destination moves from accum to scratchpad

Surfaced while scoping the DMA slot (§7): DMA should be HBM↔scratchpad
only, never accum-facing. Reasoning: every accum touch in this ISA already
has a direct compute-unit port (matmul writes raw S; SFU reads raw S,
reads/writes metadata+output, writes P) — the same pattern real Gemmini
uses (`ex_read_from_acc`/`ex_write_to_spad`, §4.3, kept separate from
`mvin`/`mvout`, which are strictly scratchpad-facing). Giving DMA a special
accum-facing path just to ship the final output would mean inventing a
hardware capability beyond anything the real reference hardware actually
has, and would cost DMA a memory-type selector field on top of its
addressing — not worth it for one case.

**Resolution**: `softmax-finalize` now writes `O_final` to a dedicated
scratchpad region (sized for the 8 per-head output tiles) instead of back
into the accumulator. Reads are completely unaffected — `l` and `O` still
come from the accumulator, same as before. Only the destination's
hardcoded base changes (`output-accum-base` → `output-scratch-base`); the
same 3-bit head-index still resolves it. No field added, no width change.

**Updated field table entry**: `softmax-finalize` — opcode + 3-bit
head-idx; reads accum (metadata `l` + output `O`), writes **scratchpad**
(output). This is what DMA now picks up as one of its store sources.

---

## 7. Phase 1 — ISA Definition: DMA Slot

### 7.1 Scoping: exactly 4 real cases, HBM↔scratchpad only

DMA's job is load Q, load K, load V, store O — nothing else. Grounded in
the fused regime (`prefill_notes.md` §1.2/§1.4): P and raw S never touch
HBM at all, so those never need DMA. And DMA never touches the
accumulator directly, matching real Gemmini's own separation
(`ex_read_from_acc`/`ex_write_to_spad` vs. `mvin`/`mvout`, §4.3) — every
accum interaction in this ISA already has a direct compute-unit port
(matmul writes raw S; SFU reads raw S, reads/writes metadata+output,
writes P). This scoping decision is what forced the `softmax-finalize`
correction in §6.5 above (its destination had to move to scratchpad,
since DMA has nowhere else to pick the final output up from).

### 7.2 Instruction count: 3, not 4 — K and V merge, mirroring `load-stationary`

Applied the exact same test that decided `load-stationary` handles both K
and V with one opcode: do the destination fields structurally differ
enough to force separate opcodes (the way `steady-state-stream` eventually
did), or is it "different address, same shape"? K and V's destinations
are both a 1-bit buffer-select into their own fixed base — identical
shape — so they merge into `load-K/V`. Q's destination shape (a 3-bit
head-index into 8 simultaneously-live slots) genuinely differs, so it
stays a separate opcode. Final: `load-Q`, `load-K/V`, `store-O`.

### 7.3 HBM addressing: hardcode the tensor base, carry a real offset

Initial framing (too strong): DMA needs a "full explicit HBM address."
Corrected: exactly like every scratchpad field in §5–§6, hardcode the
per-tensor HBM base (Q-base/K-base/V-base/O-base are compile-time-known
constants) and carry only the **offset** to the specific slice being
fetched. But unlike the scratchpad fields (which collapsed to 2–3 bits
because there were only a handful of *physical* destinations ever), the
offset has to distinguish every *logical* position DMA will ever be asked
to fetch across the whole program — a real, non-trivial bit count:

- **Q/O offset**: `32 batches × 8 groups × 256 Q-tiles` = 2¹⁶ → **16 bits**.
- **K/V offset**: `32 batches × 8 groups × 8 chunks` = 2¹¹ → **11 bits**.

Checked against §2.5's re-fetch finding: K/V's offset correctly has no
Q-tile component, since a given (batch, group, chunk) always resolves to
the identical HBM address regardless of which Q-tile triggered the fetch
— consistent with *why* the 256× re-fetch is wasteful in the first place,
not a reason to add bits.

### 7.4 The real catch: `load-Q`/`store-O` can't be one bulk transfer as first assumed

Initial assumption: DMA moves bulk data, so one `load-Q` instruction
should fetch all 8 heads' Q-tile data at once, destination collapsing to
zero bits the same way `steady-state-stream-qk`'s raw-S destination did.
**Never checked whether that's mechanically possible.** It isn't: Q's HBM
layout is `(batch, n_heads, seq_len, d_head)` — for a fixed (batch,
Q-tile-position), the 8 heads sharing one KV-group live at 8 *separated*
locations (each `seq_len × d_head` apart), not one contiguous block. A
single simple DMA burst can't cover that. Same problem, symmetrically, for
`store-O`.

Three options considered:
1. A strided/gather DMA descriptor (one instruction, internally loops 8×) — real precedent (Gemmini's own `mvin` supports striding), but a genuine new hardware capability (an address-generation unit for strided access).
2. Per-head instructions, compiler embeds 8 fully-distinct raw addresses.
3. A layout-assumption escape hatch (pre-arrange Q/O in HBM so heads are contiguous) — pushes cost onto an unaccounted-for preprocessing step.

**Resolution (better than any of the three)**: keep it one opcode, issued
8× per Q-tile (once per head), with a 3-bit head-idx — but **hardware**
computes the HBM address via `head_base + head_idx × (seq_len×d_head)`,
the exact same base+idx×stride mechanism every other head-indexed field
in this ISA already relies on for scratchpad addressing, just extended to
the HBM side. This is strictly better than option 1 (no new hardware
capability — it's the same address-generation logic already needed
elsewhere, not a new strided-DMA engine) and more precise than option 2
(the 16-bit offset stays identical across all 8 head-instructions; only
the 3-bit head-idx changes, rather than the compiler carrying 8 fully
independent addresses). Applies identically to `store-O`.

### 7.5 Final field table, DMA slot

| Instruction | Fields | Issued |
|---|---|---|
| `load-Q` | opcode + 16-bit HBM offset + 3-bit head-idx | 8× per Q-tile |
| `load-K/V` | opcode + 11-bit HBM offset + 2-bit dest-select (K-vs-V + buffer-select) | 1× per chunk |
| `store-O` | opcode + 3-bit head-idx + 16-bit HBM offset | 8× per Q-tile |

### 7.6 Phase 1 core ISA complete — the encoding pass (bundle layout + sizing) followed in §8

All three slots (matmul-issue §5–§6.4, SFU §6, DMA §7) are field-derived,
8 instruction types total. What remained — bundle layout and the combined
bit-width/sizing pass, both required by `spec.md`'s own encoding-decision
list — is worked through in §8 below.

---

## 8. Phase 1 — Encoding Pass: Bundle Layout and Sizing

The last two required decisions (`spec.md`: "address-field widths sized
to real capacity... how the bundle itself is laid out... state and defend
the tradeoff"), done last since bundle layout gates opcode width and
opcode width gates the final per-slot sizes.

### 8.1 Bundle layout: fixed-width-per-slot, checked against real numbers first

Initial instinct (user's): pipelining should keep every slot busy most of
the time, favoring fixed-width. **Checked, not assumed** — real
issuance counts per Q-tile-iteration (8 chunks × 8 heads):

- Matmul-issue: 16 `load-stationary` (2/chunk × 8) + 64 `steady-state-stream-qk` + 64 `steady-state-stream-v` = **144**
- SFU: 64 `softmax-update` + 8 `softmax-finalize` = **72**
- DMA: 8 `load-Q` + 16 `load-K/V` (2/chunk × 8) + 8 `store-O` = **32**

Ratio ≈ 4.5 : 2.25 : 1. Even under a best-case schedule bottlenecked by
matmul-issue (the busiest slot), SFU is idle ~50% of cycles and DMA
~78%. **The stated intuition ("always busy") doesn't hold up** — but the
conclusion (fixed-width) still does, for different reasons:

1. The idle-slot cost is code-density/storage, not throughput — an idle
   slot doesn't cost a cycle, just unused bits in that cycle's encoding.
2. Fixed-offset decode is real hardware simplicity, consistent with this
   project's standing preference for minimal control hardware over
   marginal efficiency (mirrors the DMA-scoping decision and the
   base+idx×stride resolution over a strided-DMA engine — both times,
   "don't add hardware capability you haven't earned" won).
3. Direct precedent: Groq's own dispatch is "144-wide VLIW instructions"
   (§1.3) — a fixed format, in a chip whose entire thesis is minimal ICU
   area, i.e. the real production analog for this project's own bet
   already made this exact tradeoff the same way.

**Primary hypothesis**: fixed-width-per-slot. **Explicit alternate**: a
compact/variable scheme (active-slot bitmask + only-present-slots'
fields) — would recover most of the idle-slot waste at the cost of
variable-length decode; the real tradeoff to revisit if code density ever
becomes the actual bottleneck (e.g. Phase 2's fully-unrolled program
turning out far larger than instruction memory can hold — this project
has no hardware loop construct at all, per the Note on Scope, so the
compiled program really is this large, un-shrunk by iteration).

### 8.2 Sizing pass: two more real corrections, same shape as P's

Verifying every region against real capacity (`prefill_notes.md` §2.3's
budgets) surfaced two more corrections in the same shape as Correction 3
(§6.2) — an original hypothesis assumed a value smaller/simpler than what
this session's own derivations actually require.

**§2.3's original scratchpad equation** had `"fixed Q/output (8,192 B)"`
as one combined term — implying Q and output were each conceived as
single, ~4,096 B transient buffers, the same (now-known-wrong) shape as
the original P assumption. But:

- **Q** actually needs ×8 simultaneous per-head residency (§6.2
  Correction 4, driven by the chunk-outer/head-inner loop order, §5.5) →
  8 × 4,096 B = **32,768 B**, not ~4,096 B.
- **Output** actually needs its own ×8 per-head scratchpad region (§6.5's
  `softmax-finalize` destination correction) → also **32,768 B**, at
  **int8**, not fp32 — fp32 was only ever needed for softmax's internal
  numerical stability; once `finalize` does the final divide, nothing
  downstream needs more precision than the workload's own stated
  int8-output convention (`prefill_notes.md`'s workload table).

Combined: 65,536 B against the original 8,192 B assumption — an 8×
gap, same shape as Correction 3, on two regions instead of one.

**Full capacity check** (both fit comfortably — see `isa.md` §5 for the
final tables):
- Scratchpad: K/V (524,288 B) + P (32,768 B) + Q (32,768 B) + output
  (32,768 B) = 622,592 B of 1,048,576 B (slack ≈ 416 KB).
- Accumulator: metadata+output-accum combined (133,120 B, §2.3's original
  `8×520×tile_q` term, unaffected by the scratchpad corrections above
  since that's the *accumulator*-resident copy of `O` during
  accumulation, separate from the *scratchpad* copy `finalize` writes) +
  raw-S/P transient (16,384 B) = 149,504 B of 262,144 B (slack ≈ 110 KB).

### 8.3 Final bundle width

Opcode widths, mechanical once bundle layout is fixed:
`ceil(log2(instruction count))` per slot — matmul-issue (3 types) → 2
bits, SFU (2 types) → 1 bit, DMA (3 types) → 2 bits.

| Slot | Opcode | Worst-case instruction fields | Slot width |
|---|---|---|---|
| Matmul-issue | 2 bits | `load-stationary`, 5-bit src | 7 bits |
| SFU | 1 bit | either instruction, 3-bit head-idx | 4 bits |
| DMA | 2 bits | `load-Q`/`store-O`, 16-bit offset + 3-bit head-idx | 21 bits |

**Total: 7 + 4 + 21 = 32 bits.** One clean 32-bit word per cycle.

### 8.4 Phase 1 complete — see `isa.md` for the consolidated deliverable

Every encoding decision `spec.md` requires is now made: all 8
instructions' fields, bundle layout (stated and defended, primary +
alternate), full capacity verification, final 32-bit bundle width. The
clean, non-narrative version of this whole spec is `isa.md` — that's the
actual Phase 1 deliverable. One item genuinely deferred to Phase 2,
non-blocking: `load-stationary`'s re-issue frequency (§5.5's open
thread) — affects instruction count, not any field width.

### 8.5 Post-completion review: two real clarifications caught, no field changes

Reviewing the finished spec surfaced two places where the *description*
was imprecise even though the underlying field counts were already
correct — both patched directly into `isa.md`, logged here for the
record.

**`load-K/V`'s dest-select bit does double duty.** Originally described
as a scratchpad-side selector only. Caught: K and V are different HBM
tensors with two genuinely separate hardcoded bases (`K-base`, `V-base`)
— "hardcode the base" was underspecified, since there are two bases, not
one combined "K/V base." Resolution: the K-vs-V half of the existing
2-bit `dest-select` field also drives an HBM-side mux (`selected_base =
K-vs-V ? V-base : K-base`), in addition to picking the scratchpad
destination — same field, two pieces of address-generation hardware
consuming it, no new bits. Same pattern as `steady-state-stream-v`'s
single head-idx resolving two different bases for source vs. destination
(§6.3) — this is a third instance of that shape.

**`load-Q`/`store-O` are one instruction type occurring 8×, not "8
separate instructions."** A conversational imprecision, not a doc error —
`isa.md` already correctly says "issued 8× per Q-tile transition (once
per head)," which is unambiguous. Worth noting anyway since the
distinction matters: it's the exact same "one opcode, many per-head
occurrences" pattern every other instruction in this ISA uses
(`steady-state-stream-qk`, `softmax-update`, etc.) — not a special
multi-opcode DMA construct. head-idx being an ISA-visible field (rather
than invisible, internal to a bulk-transfer instruction) is a consequence
of *choosing* per-occurrence issuance over the strided-descriptor
alternate (§7.4) — not an unavoidable fact of HBM non-contiguity by
itself. The strided-descriptor alternate would have solved the same
non-contiguity problem without ever exposing head-idx as a field.

---

## 9. Post-completion correction: fine-grained softmax granularity was missing from the loop nest and §8.1's counts

Caught while reconstructing the full loop nest in the Phase 2 session —
this is a real bug, not staleness, and it reaches further than the
instruction counts alone.

### 9.1 What went wrong

`load-stationary`'s 5-bit source field (2-bit buffer-select + 3-bit
slice-idx, §5.2/§7) was **already correctly designed** around the fact
that a 1024-wide chunk only fits the 128-row array in 8 slices — nothing
wrong there. But §8.1's instruction-count table (the 144:72:32 ratio used
to justify fixed-width bundle layout) silently used *coarse* granularity —
one `steady-state-stream-qk`/`softmax-update` per (chunk, head), as if
the whole 1024-wide chunk were one compute pass. That directly contradicts
`prefill_notes.md` §2.3's own explicit, already-settled finding: softmax
granularity is **fine-grained, per 128-wide array sub-pass** — coarse was
tried first and rejected because it overflows the accumulator budget by
2 KB (133,120 B running state + 131,072 B coarse raw-S > 262,144 B).
`load-stationary`'s field design correctly anticipated this; the loop-nest
reasoning built on top of it in §8.1 did not.

### 9.2 Corrected loop nest

A slice loop (0..8, matching `tile_k`/128 = 1024/128) sits *inside* chunk
and *outside* head, for both the K/QK^T half and the V/·V half:

```
for batch in 0..32:
  for kv_group in 0..8:
    for q_tile in 0..256:
      load-Q ×8 (once per head)                    # DMA

      for chunk in 0..8:                            # tile_k sweep
        for slice in 0..8:                          # 1024/128 array sub-pass — §2.3's fine-grained softmax
          load-stationary(K, slice)
          for head in 0..8:
            steady-state-stream-qk(head)            # → raw S (fixed accum slot)
            softmax-update(head)                    # → m, l, O-rescale, P[head] (this slice)

          load-stationary(V, slice)                 # same slice, right after K's — §9.4 decides
                                                      # this ordering specifically (not chunk-batched)
          for head in 0..8:
            steady-state-stream-v(head)              # P[head] × V[slice] → accumulate O[head]

        load-K/V(K, next chunk)                      # DMA prefetch, double-buffered
        load-K/V(V, next chunk)

      for head in 0..8:
        softmax-finalize(head)                        # O_final → scratchpad

      store-O ×8 (once per head)                      # DMA
```

### 9.3 Corrected instruction counts (per Q-tile)

| Slot | Instruction | Old (wrong) count | Corrected count |
|---|---|---|---|
| Matmul-issue | `load-stationary` | 16 | 2×8×8 = **128** |
| Matmul-issue | `steady-state-stream-qk` | 64 | 8×8×8 = **512** |
| Matmul-issue | `steady-state-stream-v` | 64 | **512** |
| SFU | `softmax-update` | 64 | **512** |
| SFU | `softmax-finalize` | 8 | **8** (unaffected — fires once per head, after all chunks/slices) |
| DMA | all three | 32 | **32** (unaffected — DMA moves whole 1024-wide chunks; array sub-passing is invisible to it) |

New per-slot totals: matmul-issue = 1,152, SFU = 520, DMA = 32. Ratio
≈ **36:16:1** (was 4.5:2.25:1). **Bundle-layout conclusion (§8.1) is
unaffected** — the reasoning (storage-not-throughput cost, decode
simplicity, Groq precedent) was never ratio-dependent, only the magnitude
of idle-slot waste got more extreme (DMA idle ~97% of cycles under a
best-case matmul-bound schedule, not ~78%). **Bundle width (32 bits,
§8.3) is also unaffected** — that's per-instruction field size, not
instruction count.

### 9.4 The deeper thing this surfaces — and why it had to be settled here, not deferred

Correction 3 (§6.2) concluded P needs ×8 residency because the K→V
reload boundary was assumed to sit at the **chunk** level. With slicing
now explicit, that boundary could instead sit at the **slice** level
(K[slice] → 8 heads → V[slice] → 8 heads → next slice) or the chunk level
(all 8 K-slices, then all 8 V-slices) — and unlike a pure scheduling
choice, **this one changes the ISA field count itself**, not just
scratchpad usage:

- `m`, `l`, `O` are persistent, incrementally-updated state — exactly one
  instance per head, always, regardless of scheduling. No slice dimension
  ever applies to them.
- **P is not persistent** — it's a fresh value produced once per (head,
  slice). Slice-interleaved: only one slice's 8 P's are ever alive at
  once, safely reusing the same 8 head-indexed slots every time — 3-bit
  head-idx stays sufficient. Chunk-batched: all 64 (head, slice)
  combinations coexist simultaneously, genuinely requiring a 6th field (a
  3-bit slice-idx alongside head-idx) on `softmax-update`'s P-write and
  `steady-state-stream-v`'s P-read — and asymmetrically so, since
  `steady-state-stream-v`'s O-destination would stay head-only while its
  P-source needed head+slice.

**Decided, not left open**: slice-interleaved. Total reload count is
identical either way (16 loads/chunk, regardless of K/V ordering), so
chunk-batched buys nothing — it costs a real field, 8× more P storage
(256 KB vs 32 KB — still under budget, but not free), and an asymmetric
instruction, for zero offsetting benefit. This is core ISA content (spec.md's
own "concrete enough that Phase 2 has something unambiguous to target"
bar), not a pure scheduling detail safe to defer — a scheduling choice
that changes field counts has to be settled at the ISA level, not left
for Phase 2 to discover. **`softmax-update`/`steady-state-stream-v` stay
exactly as specified — opcode + 3-bit head-idx, no 6th field — and P's
residency is genuinely settled at ×8, not conditional.**

### 9.5 Nothing else needed to change

No instruction, no field, no bit-width from §5–§8 was wrong — only the
*issuance counts* built on top of them (§8.1) and the loop-nest
documentation. `load-stationary`'s slice-idx field, `steady-state-stream-
qk/v`'s head-idx-only fields, `softmax-update/finalize`'s fields, the
32-bit bundle width, and the scratchpad/accumulator capacity totals in
§8.2 (which already used the correct fine-grained per-slice P size,
4,096 B — that number was never wrong, only the *count* of simultaneous
instances was left ambiguous) all stand as written.

---

## 10. Cross-project confirmation: GQA's benefit here is array-level reuse, not HBM bandwidth

Surfaced by comparing Q's and K/V's HBM-fetch-to-compute reuse ratios —
only possible to see clearly once §9's corrected per-slice counts were in
hand.

**Finding**: Q and K/V both achieve **64× compute-reuse per HBM fetch** in
this design. Q: one `load-Q` fetch (per head) is reused across 8 chunks ×
8 slices = 64 `steady-state-stream-qk` calls, since Q persists unchanged
across the whole K-sweep for a fixed head. K/V: one `load-K/V` fetch (per
chunk) is reused across 8 slices × 8 heads = 64 calls, since GQA-group
sharing lets one loaded chunk serve all 8 heads, and `load-stationary`
further sub-passes it into 8 slices. Numerically tied — but a coincidence
(both dimensions happen to be 8), not structural necessity, per the
reuse-tie discussion above.

**What breaks the tie open — the MHA counterfactual**: without GQA
(`n_kv_heads`=64, no group sharing), K/V's reuse would only be 8× (8
slices × 1 head, no cross-head sharing) — Q's 64× is independent of GQA
entirely, so under MHA, K/V would be markedly worse-amortized than Q.
**GQA's real, measurable effect is closing exactly that gap** — bringing
K/V's array-level reuse from 8× up to 64×, precisely the GQA group size —
but the benefit is realized entirely **on-chip**, inside
`load-stationary`'s stationary-reuse mechanism across the 8-head group. It
never reduces HBM/DMA traffic: `load-K/V` still refetches from HBM once
per Q-tile regardless of how well the array reuses that data within one
Q-tile.

**This independently confirms, via a completely different method
(instruction-issuance counting, not byte-counting), two findings already
on record in `prefill_notes.md`**:
- §2.5's "Major Open Finding" — K/V gets re-fetched ~256× (once per
  Q-tile) instead of once per group; GQA's theoretical HBM-byte savings
  never materializes under this tiling scheme.
- Phase 1a Key Takeaway #7 — "GQA's real prefill payoff is
  scratchpad/KV-cache pressure, not throughput"; fused prefill is
  compute-bound, so GQA's bytes reduction doesn't reduce execution time,
  its actual benefit sits elsewhere.

Two independent derivations — byte-counting in the sibling
hardware-hypothesis project, instruction-counting in this project's own
ISA — landing on the identical conclusion is the same kind of
cross-validation this whole project has valued throughout (direct
precedent: the online-softmax mechanism, independently derived, then
confirmed against Gemmini's real `Normalizer`, `prefill_notes.md` §4.5).

---

## 11. Phase 2 — Hand-Scheduling: Loop Nest, Latency Derivation, Clock Decision (IN PROGRESS)

Working log for the actual hand-scheduling session — real derivation, real
self-corrections, kept live rather than smoothed over, same standard as
§5–§9.

### 11.1 Scope

Per `spec.md` Phase 2's "hand-schedule the real bundle sequence... the same
way a human would if handed this ISA with no compiler at all" — scoped to
**one Q-tile's steady-state body**, written out in
[`phase2-loop.md`](phase2-loop.md) (line numbers there are the reference
for the rest of this section). The outer `batch`/`kv_group`/`q_tile` loops
are identical repeats of this same body at different addresses — not new
scheduling content, so excluded from the file on purpose.

### 11.2 Real gap found: no prologue/epilogue for the double-buffered K/V DMA

`phase2-loop.md` lines 21–22 (`load-K/V(K, next chunk)` /
`load-K/V(V, next chunk)`) only prefetch the chunk *after* the one
currently being computed on. Nothing in the body loads chunk 0's own K/V
data before the `chunk=0` slice loop tries to consume it, and on
`chunk=7` (the last chunk) that same trailing prefetch would target a
nonexistent chunk 8. Same shape as Itanium's rotating-register
prologue/kernel/epilogue for software-pipelined loops (§1.1) — a real
structural gap, **not yet resolved**, carried forward as the first open
item for the next session.

One real clarification already made about *how* to resolve it (not yet
executed): a proposed `if chunk == 0` framing was checked and corrected —
`chunk` is compile-time-known (Decision 2 check 1) and this ISA has zero
branch instructions (Note on Scope), so this can't be a runtime
conditional. The right shape is a literal prologue block, physically
placed once before the loop in the fully-unrolled compile-time program
(same as Itanium's rotating-register prologue) — never emitted for
`chunk`=1–7, not a hardware test emitted every iteration.

### 11.3 DMA latency: bytes → time → cycles

Method (checked, not assumed): bytes moved ÷ HBM bandwidth → real time →
× clock frequency → cycles. HBM bandwidth reused from real precedent
already in this project — `prefill_notes.md` §1.3's `8.2×10¹¹ B/s` (TPU
v5e), the same number Phase 1c's own Timeloop config used for DRAM
bandwidth (§3 there) — not a fresh assumption.

Byte counts per instruction (int8, per the workload table,
`prefill_notes.md` §0):
- `load-K/V`: `tile_k × d_head` = 1024×128 = **131,072 B**
- `load-Q` / `store-O`: `tile_q × d_head` = 32×128 = **4,096 B**

Raw times: `load-K/V` ≈ **159.844 ns**; `load-Q`/`store-O` ≈ **4.995 ns**.
Cycle count depends on clock (deferred until §11.5, once compute-cycle
numbers existed to check against).

### 11.4 Compute latency: matmul-issue and SFU — real mechanisms, not just numbers

**`load-stationary` = 128 cycles.** Real derivation, with one real dead
end: an initial description ("shifts one cycle at a time down thru the
array") sounded like a value broadcasting/replicating into every row of a
column, which would be wrong — `load-stationary` needs 128 *distinct*
K-position vectors, one per row, not one vector replicated across rows.
Resolved by tracing a concrete 3×3 example by hand (same
build-a-small-example technique that resolved Phase 1's transpose-bit
dead end, §5.3): feed a new row in at the entry column each cycle, shift
everything already in the array over by one column to make room. Traced
cycle-by-cycle for 3 rows → 3 cycles, confirming the general case is
exactly `N` cycles (not `2N-1`) because the shift-out-of-the-way happens
*concurrently* with each new row's feed, not as a separate phase
afterward. Generalizes directly: 128×128 case → **128 cycles**.

**`steady-state-stream-qk` = 159 cycles** (and `steady-state-stream-v` =
159 by symmetry, not separately re-traced). Standard systolic pipeline
formula `N + D − 1`: `N`=32 (`tile_q`, the feed count — one new Q-position
starts streaming per cycle) + `D`=128 (`d_head`, the drain — cycles for
the *last*-fed Q-position's data to finish propagating across the array's
spatial depth before its output is valid) − 1 (the feed/drain boundary
cycle is shared, not double-counted) = **159**. Grounded in
`prefill_notes.md` §2.1's own spatial/temporal split (`QK^T` spatial =
`(d_head, k-tile)`, temporal = `seq_len_q`; `·V` spatial = `(k-tile,
d_head)`, temporal = same — identical shape, hence the symmetry claim for
`-v` without re-deriving). One real correction mid-derivation: an initial
`128 + 32` guess had the feed/drain labels swapped (called `tile_q`=32 the
"drain") and was off by one (no `−1`) — both caught before finalizing.

**`softmax-update` ≈ `softmax-finalize` ≈ 32 cycles — a genuinely
different kind of estimate than the two above, flagged as such.** Unlike
DMA (real bandwidth number) and the array (real 128×128 structural fact),
the SFU has **no real anchor** — `prefill_notes.md` §4.5's `Normalizer`
finding confirms *what* it computes, never *how wide* it is. Adopted as an
explicit hypothesis, same "primary + alternate" discipline as elsewhere
(§2.1, §8.1): **primary = 128-wide** (matches the array's own width, so
the SFU doesn't become an obvious new bottleneck on its producer);
**alternate = narrower**, grounded in the one real lane-count this project
actually has on record — `decode_notes.md`'s 32-lane SIMD engine (a
*different* machine, cross-machine analogy only, not directly
transferable). Under the primary hypothesis: `softmax-update` processes
`tile_q`=32 rows (128 elements/row, one row/cycle) → 32 cycles.
`softmax-finalize` is the same shape (`O` is `tile_q×d_head` = 32×128,
`l`'s per-row scalar broadcast across each row) → 32 cycles.

Real dependency-chain check done before accepting "32 cycles, one row per
cycle" as sufficient: does `m_j → P_j → l_j → O_j` force separate
full passes over all 32 rows (max first, then exp/sum), the way a
numerically-stable softmax normally needs two passes over an *entire* row
before any output is valid? Checked explicitly — **no**, because every
term in the recurrence is scoped to one query row at a time (row `i`'s
values depend only on row `i`'s own 128 `S_j` elements and row `i`'s own
prior state, never on other rows), and under the 128-wide hypothesis all
128 elements of one row arrive together in that row's own cycle — so
`rowmax`, `exp`, `rowsum`, and the `O` rescale can all resolve within that
row's processing without re-streaming the other 31 rows. **What is
*not* resolved**, explicitly flagged as a stated limitation rather than
chased further: the *internal* pipeline depth within one row's processing
(a max-reduction tree, an exp unit, a sum-reduction tree all have real
latency) isn't modeled — decided not worth deriving because (a) there's no
real hardware anchor for it either, same problem as the width question,
and (b) the SFU already has large idle slack (~55% under a matmul-bound
schedule, per the corrected §9.3 ratio) so it's unlikely to actually be
load-bearing; revisit only if a real schedule turns out SFU-bound.

**Final latency table:**

| Instruction | Cycles |
|---|---|
| `load-stationary` | 128 |
| `steady-state-stream-qk` / `-v` | 159 |
| `softmax-update` / `softmax-finalize` | 32 |
| `metadata-init` | 32 (added §11.9/§11.11 — writes `O` (32×128) through the same per-row write path `softmax-update`/`-finalize` use, so assumed to share their 32-cycle shape rather than being near-instant) |
| `load-K/V` | **160** (⌈159.844⌉ — see correction below) |
| `load-Q` / `store-O` | **5** (⌈4.995⌉ — see correction below) |

**Correction, caught while building the actual bundle table (§11.13 pickup):**
the raw `159.844`/`4.995` DMA figures were carried through every downstream
cycle-number derivation (§11.11's prologue timeline, §11.12's slice-0/
chunk/tail figures) as literal fractional-cycle latencies, and that's
physically wrong for this design — one bundle issues per clock, so a slot
can't hand off a result mid-cycle; the earliest a dependent instruction can
actually see it is the next cycle boundary. **Rule: ceiling each DMA
instruction's own latency to the next whole cycle independently** (not
round-to-nearest, and not accumulated/banked across separate instruction
instances). `load-K/V`: ⌈159.844⌉=160; `load-Q`/`store-O`: ⌈4.995⌉=5. Every
`.844`/`.839`/etc. cycle number appearing earlier in §11.11/§11.12
(287.844, 446.844, 22,559.844, the 179,396.839 tail figure, etc.) is stale
and superseded by this fix — none of it had been transcribed into a file
yet when this was caught, so no rework debt, but don't trust those exact
figures if reading this section out of order. Structural findings
(K→Q×8→V DMA ordering, margin sizes, hazard-free conclusions) are
unaffected — the shifts are ~1-5 cycles per instruction, nowhere close to
closing any of the margins already found. Re-derivation with corrected
latencies starts at §11.13.

### 11.5 Clock decision: 1 GHz

Real mechanism established before picking: matmul-array cycle counts are
**clock-invariant** (structural — confirmed directly via Timeloop's own
identical QK^T cycle count across its 1 GHz and 2 GHz runs, §3.1), but
DMA's *cycle* cost scales linearly with clock since its real-world time is
fixed by bandwidth, not cycles/s — `load-K/V` is 159.844 cycles @ 1 GHz vs.
319.688 @ 2 GHz for the identical real transfer.

This is a real, already-documented risk, not hypothetical:
`prefill_notes.md` §3.2 finding 2 shows the *identical* schedule
(`primary_v`/`primary_v_v2`) going from 100% utilization at 1 GHz to
80.63% at 2 GHz, purely from the clock doubling while DRAM bandwidth
stayed fixed in bytes/s — direct evidence clock choice can flip whether
DMA hides behind compute.

**Decided: 1 GHz**, on two grounds, not an arbitrary pick:
1. **No synthesis pass was ever run** for this design (`prefill_notes.md`
   §4.6 — Phase 1d explicitly "stopped at RTL/Verilator, before any
   synthesis pass"), so there's no evidence for what frequency this custom
   128×128 array + SFU could actually achieve. Reaching for "higher clock,
   more perf" (raised and rejected mid-session) has nothing backing it.
2. Timeloop's own 2 GHz run wasn't a free clock bump either — §3.1 labels
   it "int8-pumped," a specific real hardware technique (double-pumping an
   int8 datapath), not an arbitrary frequency choice. 1 GHz is Timeloop's
   own *unmodified* baseline — the only clock value with actual precedent
   behind it.

### 11.6 DMA-hiding check: real numbers, one real correction

Checked whether `load-K/V(next chunk)`'s DMA cost can be hidden behind the
current chunk's compute, now that real cycle numbers exist for both sides.

Per-slice matmul-issue cost: `load-stationary(K)`=128 + 8×
`steady-state-stream-qk`=1,272 + `load-stationary(V)`=128 + 8×
`steady-state-stream-v`=1,272 = **2,800 cycles/slice**. A chunk has 8
slices → **22,400 matmul-issue cycles/chunk** available before the next
chunk's `load-stationary` needs the prefetched data.

Against that, worst-case DMA cost (`load-K/V(K)` + `load-K/V(V)`,
serialized, no overlap) ≈ `2×159.844 ≈ 320 cycles` @ 1 GHz. Margin ≈ **70×**
— trivially hideable. (`load-Q`/`store-O` correctly excluded from this
check — they're once-per-Q-tile, not once-per-chunk, irrelevant to the
per-chunk prefetch question.)

**One real correction along the way**: an initial pass estimated the
hiding window as "~1,000+ cycles," counting only one slice's inner 8-head
loop (8×159≈1,272) rather than the full chunk's 8 slices (22,400) — a
~22× undercount, caught by actually computing the real number rather than
eyeballing it. Same shape as §8.1's original ratio miscount (the "always
busy" intuition that didn't hold up until checked) — this project keeps
re-learning the same lesson at different levels, worth remembering rather
than assuming a real number "obviously" holds without computing it.

**Side effect worth keeping**: this margin is large enough (~70× at 1 GHz,
~35× at 2 GHz) that the clock choice from §11.5 turns out **not to bind**
on this specific hiding question either way — the §3.2 risk is real *in
general*, it just doesn't apply to this loop nest's specific DMA:compute
ratio, which is far more lopsided than the Timeloop `·V` case that
surfaced it. Both facts stand together, not in tension: clock choice
matters in general, and 1 GHz was still picked on independent grounds
(§11.5), not because 2 GHz would have broken this check.

### 11.7 Where this session stopped — pick up here

Latency derivation and the clock decision are both done (§11.3–§11.6).
**Not yet done, in order**:

1. **Resolve the §11.2 prologue/epilogue structurally** — decide the
   actual instructions for chunk 0's initial K/V load and chunk 7's
   missing trailing prefetch, as a literal one-time block in the
   unrolled program (not a runtime branch, see §11.2).
2. **Build general producer/consumer placement rules** — for every
   dependency edge in the loop nest, the minimum cycle gap is the
   producer's latency (§11.4 table); the open question is what fills that
   gap (other independent instructions, ideally, not NOPs) in each case.
3. **First concrete case, in progress when the session ended**:
   `load-stationary(K, slice)` → `steady-state-stream-qk(head=0)` — the
   very first instruction of a slice, so unlike later heads/slices (which
   have prior heads' or prior slices' independent work available to fill
   the gap), nothing had yet been identified to fill `load-stationary`'s
   128-cycle latency before `head=0`'s stream instruction can safely
   issue. This is the exact point to resume at.
4. Once placement rules exist generally, build the actual bundle
   sequence — start with one (chunk, slice) sub-iteration as the
   representative unit, verify it, then generalize across the full loop
   nest with the prologue/epilogue folded in.

### 11.8 Methodology correction, two settled decisions, and a full hazard pass — before any bundle placement

A first attempt at item 3 above (interleaving `load-Q` issuance with the
first slice's `steady-state-stream-qk`/`load-stationary` to fill idle
slots) got caught mid-derivation as premature optimization — packing
bundles tightly before the underlying dependency graph and execution
model were actually settled. Correct order, same shape as Itanium's own
kernel-before-rotating-register-overlap sequencing (§1.1) and CS243 list
scheduling's own dependency-graph-before-packing structure: **settle
hazards and the slot-occupancy model, build a naive correct schedule,
then optimize.** The two real open items surfaced during that premature
pass turned out to be genuine correctness prerequisites, not
optimizations — resolved here.

**Real hazard found: raw S's single accumulator location.** `isa.md`
(pre-edit) gave raw S a single fixed accumulator location, zero bits,
justified by "`softmax-update` consumes it immediately... no reload gap."
That's only safe *within* one head. Across heads it's a live WAR hazard:
`steady-state-stream-qk(head i+1)` cannot overwrite S until
`softmax-update(head i)` has read it. Unfixed, under the busy-for-
full-latency model decided below, that's a real stall of
`softmax-update`'s full 32-cycle latency at every head transition — 7
transitions/slice × 8 slices/chunk × 8 chunks = 448/Q-tile × 32 =
**14,336 stall cycles/Q-tile**, an avoidable ~8% tax on top of the
179,200 real matmul-issue cycles/Q-tile (§11.6).

**Rejected fix: ×8 (one S location per head), same as P/O/metadata.**
Checked against capacity, not assumed — S is `tile_q × 128 × 4B (fp32)`
= 16,384 B/instance, so ×8 = 131,072 B, pushing the accumulator to
264,192 B against the 262,144 B budget, over by ~2 KB. (This exact
number was already on record without the connection being made at the
time — §9.1's "coarse was tried first and rejected because it overflows
the accumulator budget by 2 KB (133,120 B + 131,072 B coarse raw-S)" is
the identical ×8/coarse-raw-S overflow, independently re-derived here.)

**Chosen fix: double-buffer S (×2), not ×8 — and zero new instruction
bits, not one.** `softmax-update` (32 cycles) is far shorter than
`steady-state-stream-qk` (159 cycles), so alternating `qk(head i)`
between two S buffers means the *other* buffer is always long-since free
by the time it's needed again three heads-worth of latency later — the
hazard disappears completely, not just shrinks. Cost: +16,384 B
(accumulator total 165,888 B, slack ≈ 94 KB — comfortable, same shape as
every other capacity check in this project).

On the encoding: an initial framing proposed a dedicated 1-bit
buffer-select field on `steady-state-stream-qk`. Checked against this
project's own standing principle (§ "compile-time-known ≠ operand-free,"
handoff.md) run in the other direction — a value only needs a real bit
if hardware *can't* derive it from something already present. Buffer
selection here is exactly `head_idx & 1`, and `head-idx` (3 bits) is
already carried on both `steady-state-stream-qk` (for the Q source) and
`softmax-update` (for metadata/O/P) — so **zero new bits**, hardware
just reads one more bit out of a field already there. Direct precedent
already in this ISA: DMA's `dest-select` K-vs-V bit already does double
duty as both an HBM-base mux and a scratchpad-region selector (§7.3,
`isa.md` `load-K/V`) — same move. (Doesn't change total bundle width
either way here, since `load-stationary`'s 7-bit slot is already the
matmul-issue ceiling — but the zero-bit version is still preferred: it's
structurally impossible for a compiler to get wrong, versus a
compiler-supplied bit that has to be computed correctly every time.)
Applied to `isa.md` §1 (`steady-state-stream-qk`'s field table), §2
(`softmax-update`'s Reads line), and §5 (capacity table, now 165,888 B /
94 KB slack).

**Slot-occupancy model, decided: busy for the instruction's full latency,
uniformly across all three slots** (matmul-issue, SFU, DMA) — not
issue-and-free. Two independent grounds, not just "simpler to program":
(1) for matmul-issue it's physically forced, not a modeling choice — the
array can't accept a new `load-stationary` or start a new stream while
one is mid-flight, same physical PEs; (2) making DMA follow the same rule
keeps one uniform rule instead of two, costs nothing given DMA's existing
~70× slack (§11.6), and avoids needing any completion-tracking hardware
for outstanding transfers — which would edge toward the OOO-ish
machinery Decision 2 already ruled out (§3). Uniform blocking keeps the
whole model purely static, consistent with the compiler already knowing
every instruction's completion time ahead of time.

**Deliberate hazard pass (checked every other shared piece of state, not
just S):**

- **Array stationary registers** (one operand at a time) — safe, and not
  a separate hazard: matmul-issue slot serialization (from the
  busy-for-full-latency decision above) already forces `load-stationary`
  to wait for the prior stream to finish on the same slot.
- **P (×8 per-head, scratchpad)** — WAR across slices (`softmax-update`
  of slice `s+1` overwriting P before slice `s`'s `steady-state-stream-v`
  has read it) is safe *only because* of the established strict
  K-phase-then-V-phase, no-reordering execution order (§6.3/§9.4's
  slice-interleaving rule). Flagged, not fixed: this is currently safe by
  construction, not by an explicit guard — must be re-checked if any
  later optimization pass ever reorders across slice boundaries.
- **O accumulator, two different writers** — `softmax-update(head i)`
  rescales O, then later in the same slice `steady-state-stream-v(head
  i)` adds `P@V` into the same O. Real ordering dependency, not
  previously named explicitly as a hazard — but automatically satisfied
  by the same K-phase-before-V-phase rule, and unlike S, O already has
  proper ×8 per-head addressing, so there's no cross-head version of this
  problem the way there was for S.
- **Metadata (`m`,`l`) recurrence across chunks** — ordinary sequential
  RAW chain per head, already enforced by chunk-loop program order, not a
  scheduling hazard.
- **`softmax-finalize` → `store-O`** — fully sequenced by the loop nest
  (all finalizes before any stores), ample slack.

**Net finding: S was the only instance producing a live scheduling
hazard.** Everything else holds, though P's WAR-safety and O's
write-ordering both currently depend on the naive in-order, no-reordering
execution model — worth re-verifying rather than assuming once
optimization starts moving things around.

### 11.9 Second prologue gap found while resolving item 1: metadata has no initialization instruction

While working out item 1 (the §11.2 K/V prologue), a proposed fix of
"prologue = `load-K/V(K, chunk 0)` + `load-K/V(V, chunk 0)`" correctly
covers the K/V half of §11.2 but missed a separate, previously-uncaught
gap: `isa.md` §2's own recurrence requires `m_0=-inf, l_0=0, O_0=0` before
chunk 1's *real* `softmax-update` can run (it reads `m_0`/`l_0`/`O_0` as
genuine prior state, not a special case), and **no instruction in the ISA
writes those values anywhere**. Missed even in §11.8's "deliberate hazard
pass" — that pass checked the chunk-to-chunk metadata *recurrence* (safe,
ordinary RAW chain) but never checked the recurrence's *base case*.

Same granularity as `load-Q`, for the same reason: the 8 per-head
accumulator slots are physically reused across all 65,536 (batch,
kv_group, q_tile) instances, so this has to happen fresh every Q-tile, not
once globally.

**Three options weighed:**
1. New SFU opcode (`metadata-init`), separate instruction.
2. Mode bit on `softmax-update` itself for the chunk-1 case.
3. Piggyback on `load-Q` (already fires once per Q-tile per head — same
   frequency, same head-idx addressing) as a free side effect.

**Option 3 rejected outright, not just weighed** — `isa.md` §3 states DMA
is HBM↔scratchpad only, never accumulator-facing, and §6.5 already used
that exact rule once before to move `softmax-finalize`'s destination out
of the accumulator. Tying metadata-init to `load-Q` would need the DMA
engine to write the accumulator directly, violating a boundary this
project already settled on real grounds, not just adding an awkward
coupling.

**1 vs. 2 checked and found to cost the same, contrary to how they first
looked.** `softmax-update` has no chunk-index field today (chunk is
implicit in schedule position, never encoded in any instruction) — so
telling hardware "this is chunk 1" needs a new bit *somewhere* either way.
Whether that bit widens the opcode (option 1: 2 SFU instructions → 3) or
is added as a same-instruction mode flag (option 2), SFU's opcode grows
1→2 bits and the slot grows 4→5 bits regardless — identical cost.

**Chosen: option 1.** Given equal cost, a separate opcode keeps
`softmax-update` (512 issues/Q-tile, the hot path) uniform — it always
reads real accumulator state, never branching on which chunk it's
processing — and confines the new complexity to a rare (8 issues/Q-tile),
trivially simple instruction instead. Same move this project already made
once for `load-stationary`'s transpose bit: permanently-engaged,
opcode-as-proxy, rather than a runtime-conditional flag inside a shared
instruction (`isa.md` §1).

**ISA changes applied** (`isa.md`): new `metadata-init` instruction added
to the SFU slot (§2) — head-idx only, no value fields (`-inf`/`0`/`0` are
genuine hardware constants, zero bits, same distinction as everywhere else
in this ISA). SFU is now 3 instructions, not 2: opcode 1→2 bits, slot
width 4→5 bits, **total bundle width 32→33 bits** (§4, §6). SFU per-Q-tile
count 520→528. No capacity change — it writes into already-provisioned
accumulator locations, no new storage.

**Resolved**: `phase2-loop.md` now opens with the full prologue block —
`load-K/V(K, chunk 0)`, `load-K/V(V, chunk 0)`, `metadata-init ×8`,
`load-Q ×8` — replacing the old standalone `load-Q ×8` line. Order among
the three groups isn't load-bearing (mutually independent, no shared
resource); `load-K/V(K, chunk 0)` is placed first only because it's the
one on the real critical path (gates the first `load-stationary`).

### 11.10 Epilogue resolved: chunk 7 peeled out of the loop, no conditional

The other half of §11.2 — the in-loop trailing prefetch
(`load-K/V(K/V, next chunk)`) targets `chunk + 1`, which doesn't exist for
`chunk`=7 (chunks are 0..7, 8 total). Same discipline as the prologue:
this is a literal, compile-time-unrolled structural difference in the
last iteration's body, not a runtime guard — this ISA has zero branch
instructions (Note on Scope), so an `if chunk < 7` framing was never on
the table.

**Resolved by peeling chunk 7 out of the loop** in `phase2-loop.md`: the
loop now runs `for chunk in 0..7` (chunks 0–6, each still followed by its
trailing prefetch) with chunk 7's identical slice-loop body written out
separately afterward, minus the trailing `load-K/V` pair. Straightforward
— no new instructions, no field/capacity changes, unlike the prologue and
`metadata-init` fixes.

**§11.2 (both halves) is now fully resolved.** `phase2-loop.md` reflects
both fixes.

### 11.11 First real timeline computed: the prologue, and a stall found in the original DMA order

First application of §11.8's busy-for-full-latency model to real numbers,
using the prologue as the smallest unit (`notes.md` §11.4's latency
table, including `metadata-init`'s newly-assigned 32 cycles above).

**Under the DMA order originally written into `phase2-loop.md`** (`K`,
`V`, then `Q ×8`): `K` done at 159.844; `V` done at 319.688;
`Q(head 0)` done at 324.683. Meanwhile `load-stationary(K, slice 0)`
starts the moment `K` lands (159.844) and finishes 128 cycles later, at
287.844 — which is when `steady-state-stream-qk(head 0)` wants to issue.
It needs `Q(head 0)`, not ready until 324.683 → **a real ~37-cycle stall**
(324.683 − 287.844), sitting on the matmul-issue slot, that nothing in
the earlier derivation had caught because latency numbers hadn't been
walked through as an actual timeline yet.

**Fix: reorder DMA to `K`, `Q ×8`, `V`.** Mechanism, not just a
retry-until-it-works swap: total DMA occupancy across the three groups is
order-invariant (it's a sum of fixed latencies, 359.648 cycles either
way) — reordering only changes *which* instruction becomes ready first,
not how much total DMA time elapses. So the right move is to rank the
three groups by how tight their real deadline is and issue the tightest
first. `Q(head 0)`'s deadline (287.844, set by `load-stationary`) is by
far the tightest in the prologue; `V`'s deadline (~1,560, set by when
slice 0's full K-phase finishes and `load-stationary(V, slice 0)` can
issue) is by far the loosest. Under `K → Q ×8 → V`: `Q(head 0)` finishes
at 164.839 (no stall), `V` finishes at 359.648 (still ~1,200 cycles of
slack before its own deadline). Zero-cost fix — moves a stall, creates
none.

Checked whether `Q(head 1..7)` had the same exposure under the *original*
order too: no — each later head's deadline is ~159 cycles further out
than the previous (`steady-state-stream-qk` chained on the matmul-issue
slot), which was already enough slack to cover the original
`K → V → Q` ordering for every head except head 0. Head 0 was the one
real problem, consistent with it being the very first consumer in the
whole Q-tile.

**`metadata-init ×8` needs no such reordering** — SFU-slot, fully
independent of the DMA-side timeline above, finishes all 8 heads by cycle
256 regardless of DMA order, well before its own first consumer
(`softmax-update(head 0)`, unreachable before `steady-state-stream-qk
(head 0)` completes at 446.844). Considered explicitly pairing it
head-for-head with `load-Q` in the same bundle cycle — rejected: the two
slots' natural timelines don't line up per-head anyway (SFU runs far
ahead of DMA under the corrected order), and forcing alignment would mean
artificially stalling the faster slot for no benefit. Left as its own
independent list.

`phase2-loop.md` updated to the corrected order (`K`, `Q ×8`, `V`, then
`metadata-init ×8` as its own list).

### 11.12 Slice 0's full timeline, the slice/chunk generalization, and the chunk-boundary DMA analysis

Building the naive schedule forward from the prologue's end (287.844 —
the cycle `load-stationary(K, chunk 0, slice 0)` finishes and
`steady-state-stream-qk(head 0)` can issue, per §11.11).

**K-phase pipelining, and a real off-by-one caught along the way.** The
first attempt at walking this forward started each head's instructions
one cycle later than necessary (e.g. `qk(head 0)` "288→447" then the
*next* instruction at 448, not 447) — checked and corrected: `447` is
already `qk(head 0)`'s first free cycle (`288+159=447` exactly), so the
next instruction can start *at* 447, not 448. Uncorrected, this silently
inserts a 1-cycle bubble on the matmul-issue slot per head transition —
7 of them per slice, 448/Q-tile if the habit had propagated, comparable
in kind (if far smaller in magnitude) to the S-accumulator stall §11.8
fixed. Corrected general pattern, for head `i`, K-phase starting at 288:

- `steady-state-stream-qk(head i)`: `288 + 159i` → `288 + 159(i+1)`
- `softmax-update(head i)`: starts the same cycle `qk(head i)` ends
  (no wait — this is exactly the payoff of §11.8's S double-buffering fix:
  `qk(head i+1)` doesn't need `softmax-update(head i)` to finish at all,
  since they write different S buffers), runs 32 cycles. SFU is never the
  bottleneck (32 ≪ 159 gap between heads), consistent with the ~55% SFU
  idle finding already on record (`isa.md` §4).

K-phase (8 heads) runs 288 → 1,560.

**V-phase: checked for hazards, found genuinely simple.** Confirmed
`load-stationary(V, slice 0)` co-issues with `softmax-update(head 7)` at
1,560 (different slots, `load-stationary` touches neither S, P, nor the
accumulator, so zero dependency on `softmax-update`). Confirmed
`steady-state-stream-v ×8` really is just sequential back-to-back with
nothing to schedule around it — unlike `qk`'s raw-S problem, P and O
already had proper ×8 per-head addressing from Phase 1, so there's no
cross-head hazard, and both SFU and DMA are completely idle throughout
(SFU's only K-phase-scoped work is done; DMA has nothing chunk-0-related
left to do, see below). V-phase: 1,688 → 2,960.

**Slice 0 finishes at 2,960.** Cross-check: total matmul-issue span
(159.844 → 2,959.844) ≈ 2,800 cycles, exactly matching §11.6's
independently-derived per-slice figure (128+1,272+128+1,272) — everything
built cycle-by-cycle here is consistent with the earlier aggregate
estimate.

**Slices 1–6 generalize, but are simpler than slice 0, not identical.**
`load-stationary(K, slice N)` for `N≥1` has no DMA gate at all — chunk
0's K/V is already fully resident, so slice `N` starts the instant the
matmul-issue slot frees up (right when slice `N-1`'s V-phase ends), no
prologue-style wait. Each slice after the first is exactly 2,800 cycles,
back-to-back. Chunk 0 (8 slices) finishes at matmul-issue cycle
159.844 + 8×2,800 = **22,559.844** — matches §11.6's 22,400
cycles/chunk figure exactly (22,559.844 − 159.844).

**Closing out §11.8's flagged P/O cross-slice hazards, with real
numbers, not just "safe by construction."** §11.8 flagged P's WAR-safety
and O's rescale-then-add ordering as correct only *because of* strict
K-phase-then-V-phase, no-reordering execution — worth re-checking, not
assuming. Checked: slice `N+1`'s `softmax-update(head i)` (which
overwrites P[head i] and rescales O[head i]) happens at minimum a full
slice's worth of cycles (~2,800) after slice `N`'s `steady-state-stream-v
(head i)` (which read P[head i] and added into O[head i]) — nowhere close
to zero margin. **Both items are now confirmed safe with numbers, closed
out.**

**Chunk-boundary DMA: the in-order-per-slot model has a real, useful
consequence — textual position in `phase2-loop.md` doesn't change the
actual timeline unless something else genuinely competes for the slot.**
DMA is in-order (no OOO, Decision 2), so it processes its own
instructions in whatever order they appear across the *whole* unrolled
program. Chunk 0's entire slice loop contains zero DMA instructions, so
`load-K/V(K, chunk 1)` is already DMA's literal next instruction after
the prologue's `V(chunk 0)` — whether it's textually written inside the
prologue block or left at the end of chunk 0's loop iteration produces
the identical schedule.

**Correction (caught in the §11.13 bundle-transcription session): this was
never actually edited into `phase2-loop.md`.** The line above originally
claimed it had been "folded into the prologue... for clarity," but the
file still has it as the trailing lines inside `for chunk in 0..7`
(`phase2-loop.md:54-55`) — the claim was aspirational, not done. On
reflection, not worth doing now either: moving just chunk 0's trailing
prefetch up would mean peeling chunk 0 out of the loop the same way
chunk 7 already is for the epilogue, real structural complexity for a
change that's explicitly schedule-*identical* either way (the whole point
of this section) — same "no benefit to moving it" call already made for
chunk 2's prefetch two paragraphs down. Left exactly as it already sits in
`phase2-loop.md`; this note is the actual record now, not a to-do.

Checked whether the same move should be made for chunk 2's prefetch
(`load-K/V(K/V, chunk 2)`, which needs the *other* double-buffer instance
chunk 0 is using) — **decided not to, and confirmed with real numbers
rather than left as an assumption:**
- Chunk 0's last reads of K1/V1 (`load-stationary(K/V, chunk 0, slice
  7)`) finish at 19,887.844 / 21,287.844 respectively.
- DMA (idle since 679.336, per the same in-order argument) issues
  `K(chunk 2)` the moment K1 frees (19,887.844→20,047.688), then
  `V(chunk 2)` the moment V1 frees (21,287.844→21,447.688) —
  **regardless of whether the instruction is textually attached to
  chunk 0's or chunk 1's loop iteration**, since chunk 1's slice loop
  also has zero DMA instructions in it.
- Chunk 2's data is ready by 21,447.688; chunk 1 (starting right where
  chunk 0 ends, no stall of its own) doesn't reach `load-stationary(K,
  chunk 2, slice 0)` until 44,959.844 — **~23,500 cycles of margin**,
  more than a full chunk's worth, tighter and better-grounded than
  §11.6's general ~70× conservative bound.
- **Restructuring the loop to explicitly move chunk 2 (and beyond)
  earlier would cost real complexity for zero benefit**: it would double
  the §11.10 epilogue special-casing (both chunk 6's and chunk 7's
  would-be trailing prefetches would then target nonexistent chunks,
  instead of just chunk 7's). Left exactly where it already sits.

**Honest status check — this is not yet "the hand schedule, optimized."**
Real ground covered (prologue, slice 0 in full, the slice/chunk
generalization, chunk-boundary DMA through chunk 2), but explicitly not
done:
1. **The tail is completely undeveloped** — `softmax-finalize ×8` and
   `store-O ×8`, after all 8 chunks. Real dependency chain
   (`softmax-finalize(head i)` needs chunk 7's `softmax-update(head i)`;
   `store-O(head i)` needs `softmax-finalize(head i)`) never walked
   through.
2. **Nothing has actually been transcribed into a literal bundle-by-bundle
   table yet** — everything above is start/end cycle reasoning per
   instruction, not a cycle-by-cycle bundle document.
3. **The outer `batch`/`kv_group`/`q_tile` loops are excluded from scope
   by design (§11.1), and that hides a real, unexplored optimization**:
   the ~160-cycle prologue wait recurs every Q-tile (65,536× total).
   Whether the next Q-tile's prologue can overlap with the current
   Q-tile's tail (cross-iteration software pipelining, the same Itanium
   technique cited since §1.1) has never been asked, because it was
   scoped away — a deliberate choice, not a gap that was missed, but
   worth being explicit that it's still open. **Update, §11.17: evaluated
   on real numbers (≈0.09% of total runtime) and deliberately dropped, not
   picked up.**
4. **"No avoidable stalls found" is a more honest claim than
   "optimized."** Everything checked out because the hard architectural
   constraints (one stationary operand at a time; the softmax recurrence's
   sequential chunk dependency) left little room to begin with — but no
   alternative orderings were tried and compared, so this shouldn't be
   asserted as proven-optimal.
5. **The automated scheduler (`spec.md`'s second required Phase 2
   deliverable, real list-scheduling/software-pipelining, same loop nest
   and ISA, specifically so Phase 3 can compare the two honestly) hasn't
   been started at all.**

**Next**: derive the tail (`softmax-finalize`/`store-O`'s real dependency
chain and timeline), then transcribe the full prologue→slice-0→
generalization→tail derivation into an actual bundle-by-bundle sequence
(item 2 above) — that's the literal Phase 2 hand-schedule deliverable.
Items 3 and 5 above are real, explicitly-flagged open scope, not
next-in-line.

### 11.13 Bundle-table transcription started: format decided, prologue done with corrected latencies

Picking up item 2 above (the literal bundle-by-bundle table). Two things
settled before writing any real table, both worth keeping:

**Scope: representative segments, not a literal 179k-row table.** A row
per real cycle across the whole Q-tile isn't a thing to hand-transcribe —
almost all of it is idle bundles between real issue points, same
"idle-slot cost is storage/code-density, not throughput" framing already
on record from §8.1. Instead: one real bundle table per distinct segment
(prologue, slice 0, a generic steady-state slice, chunk-boundary DMA, the
tail), with repeats (slices 1–6, chunks 0–7) called out symbolically —
same discipline `phase2-loop.md` already uses at the loop-nest level, now
one level down at the bundle level.

**Row/column format: merged (not per-slot) table, one row per issue
event.** A row exists for every cycle where ≥1 of the three slots
dispatches a *new* instruction; cycles where a slot merely stays busy with
something dispatched earlier don't get their own row. Each cell is one of
three explicit states, not just filled-or-blank:
- a new dispatch this cycle — mnemonic + operands (e.g. `load-Q(head 0)`),
- still busy with something dispatched earlier — `⋯ <mnemonic>, busy til
  <cycle>`,
- genuinely idle, nothing to do — `idle`.

Mnemonic + named operands (head-idx, slice-idx, K/V-select, etc.), not raw
encoded bits — the raw 33-bit encoding is a mechanical last step once the
schedule itself is right, not where any real synthesis happens. Caveat
worth keeping: "idle" describes that specific cycle only, not the whole
gap since the previous row — a slot can in principle go idle→busy→idle
between two consecutive rows if something else dispatches on it in
between (doesn't happen in the prologue, since each slot's own
instructions are already spaced apart, but will start showing up once
DMA's chunk-prefetches interleave with steady-state work).

**Prologue transcribed with the §11.4 ceiling fix applied** — full table
in `bundle-schedule.md`. Structure and ordering match §11.11 exactly
(`K → Q ×8 → V`, `metadata-init ×8` independent on SFU); only the exact
cycle numbers changed, and all landed on clean integers once ceiling'd
(160, 165, 170, ..., 288, 447 — no more `.844`/`.839` fractions), which is
itself a small confirmation the fix was right, not just physically
motivated. Head 0's `Q` still lands (165) well before `load-stationary`
frees the matmul-issue slot (288), so the original stall-avoidance finding
is unaffected — margins shrank by single-digit cycles, nowhere close to
reopening anything.

**Not yet re-derived with ceiling'd latencies**: slice 0's full timeline,
the chunk cadence, and the tail figures from §11.12 — all still carry the
stale fractional numbers until they're redone the same way the prologue
just was. Do that before transcribing those segments into
`bundle-schedule.md`.

### 11.14 Slice 0 transcribed, and a reframing: the matmul-issue/SFU table is universal, not per-slice

Slice 0 re-derived with ceiling'd latencies and transcribed into
`bundle-schedule.md` — cross-checks exactly against the established
2,800-cycle/slice figure. Confirmed along the way: `load-stationary(V,
slice 0)` is forced to wait for `qk(head 7)` (matmul-issue slot
serialization) but *not* for `softmax-update(head 7)` (different
resource) — consistent with §11.8's hazard pass. Also confirmed the
chunk-1 DMA prefetch genuinely issues at cycle 360 (the moment `V(chunk
0)` frees DMA), not later — direct application of the in-order/
textual-position-doesn't-matter finding from §11.12, this time to a case
that was actually asked about live rather than found unprompted.

**Reframing, prompted by checking "do slices 1–6 just drop the
metadata-init/DMA activity slice 0 had": no — those were never part of
slice 0's own pattern.** `metadata-init` and the chunk-1 prefetch are
independent SFU/DMA events that merely overlap slice 0's *early* cycles in
absolute time; stripping them out, slice 0's own matmul-issue/SFU shape
matches a slice-generic relative-offset table cycle-for-cycle. So the
right structure isn't "one table per slice type" — it's **one universal
matmul-issue/SFU table (parameterized by `slice_start`) plus a DMA overlay
that's the only thing actually varying per slice**: slice 0 gets the
prologue continuation, slices 1–6 get nothing, slice 7 gets the
chunk-boundary prefetch. `bundle-schedule.md` restructured around this;
its scope note now explains why the original five-segment plan (prologue/
slice-0/generic-slice/chunk-boundary/tail as five separate full tables)
undersold how much of the schedule is actually shared.

**Next**: the chunk-boundary DMA overlay (slice 7 of chunks 0–6) — needs
the WAR-hazard-clears cycle (when chunk `c`'s last read of the reused
double-buffer instance actually frees it) re-derived with ceiling'd
latencies; old fractional version is in §11.12's chunk-2 case. Then the
tail, same re-derivation treatment.

### 11.15 Chunk-boundary DMA overlay: K/V halves gated by different real readers, transcribed

Re-derived with ceiling'd latencies and written into `bundle-schedule.md`.
One real correction caught mid-derivation: a first pass tied `load-K/V(K,
chunk+2)`'s issue to `load-stationary(V, slice 7)` (temporally nearby,
same slice) rather than to its own actual WAR-hazard gate,
`load-stationary(K, chunk c, slice 7)` — the true *last reader* of the
K-buffer instance being reused (each slice's `load-stationary(K)` is the
only thing that ever reads the K-scratchpad; `steady-state-stream-qk`
reads from the array's stationary registers afterward, not the buffer
directly). Since DMA is idle and in-order with nothing else queued, tying
it to `load-stationary(V, slice 7)` would just be an arbitrary, unforced
delay — same class of mistake already rejected once for pairing
`metadata-init` with `load-Q` in §11.11.

Corrected: `load-K/V(K, chunk+2)` issues the instant `load-stationary(K,
chunk c, slice 7)` finishes; `load-K/V(V, chunk+2)` issues the instant
`steady-state-stream-v(chunk c, slice 7, head 7)` finishes (the V-buffer's
own last reader) — this half was right from the start. Worked example,
chunk 0→2 (`slice_start`=19,760): `K(chunk 2)` issues at 19,888→20,048;
`V(chunk 2)` issues at 22,560→22,720 (same cycle chunk 1's slice 0
begins). Generalizes to `c`=0..6 by shifting `slice_start` by `22,400c`.
Margins are enormous either way (chunk 1's slice 0, which needs this data,
doesn't start until `22,400c` later) — this correction changes *when* the
prefetch fires by ~1,300 cycles, not whether it's safe.

**Correction (§11.18): the V half was not right from the start — this
was the exact same class of mistake as the K half, just missed at the
time.** `steady-state-stream-v` doesn't read the V scratchpad at all,
same as `steady-state-stream-qk` doesn't read the K scratchpad — both
stream against the array's already-loaded stationary registers. The V
buffer's real last reader is `load-stationary(V, slice 7)`, 1,272 cycles
earlier than the gate used above. Also: the range in the paragraph above
("chunks 0–6") and the worked example's "22,400c" generalization are both
off by one — the boundary prefetch only happens on chunks 0–5's slice 7
(chunk 6 and chunk 7's slice 7 have no `chunk+2` to fetch, since chunks 8
and 9 don't exist). Corrected numbers and range now in
`bundle-schedule.md`'s DMA-overlay section directly.

**Next**: the tail — re-derive with ceiling'd latencies (dependency chain
already fixed: `softmax-finalize(head i)` gated by `steady-state-stream-v
(head i)` of chunk 7/slice 7, not `softmax-update(head i)`, per the
correction made earlier this session but not yet written up numerically),
then transcribe into `bundle-schedule.md`.

### 11.16 Tail transcribed — hand-schedule deliverable complete

Re-derived with ceiling'd latencies, confirmed the shape already known
(each head's `softmax-finalize`+`store-O` pair rides directly behind its
own `steady-state-stream-v(head i)` in chunk 7/slice 7, no cross-head
contention — SFU/DMA both idle otherwise, 37 cycles of tail work fits
comfortably inside `v`'s own 159-cycle per-head cadence). Full numbers in
`bundle-schedule.md`. `v(head 7)`'s finish (179,360) cross-checks exactly
against the independently-derived chunk-7-finish figure
(`160 + 8×22,400`).

**Total Q-tile latency: 179,397 cycles** (`store-O(head 7)`'s finish) —
the complete hand-schedule result, prologue through tail, all with
ceiling'd latencies. This is the number Phase 3 will compare the automated
scheduler against.

**This closes out `bundle-schedule.md`'s five segments** (prologue, slice
0, universal matmul-issue/SFU table + DMA overlay, chunk-boundary DMA,
tail) — `spec.md`'s hand-scheduled bundle sequence deliverable is done.

### 11.17 Roadmap for the rest of Phase 2, evaluated on learning-value-to-time, not `spec.md` as a checklist

Two items were sitting in the "not yet done" list: cross-iteration
software pipelining, and the automated scheduler. Explicitly evaluated
both rather than doing either just because `spec.md` mentions them —
`spec.md` is a starting point, not the reason to do work that isn't worth
its time cost.

**Cross-iteration software pipelining — evaluated and dropped, not
deferred.** Real numbers, not a vibe: the savings are the ~160-cycle
prologue wait that recurs once per Q-tile, × 65,536 Q-tiles (the outer
`batch`/`kv_group`/`q_tile` loops excluded from `phase2-loop.md`'s scope)
≈ 10.5M cycles. Against the full workload's total cost (65,536 ×
179,397 ≈ 11.76 billion cycles), that's **≈0.09% of total runtime**. The
technique itself — Itanium-style rotating-register cross-iteration
overlap — is already this project's standing conceptual reference point
from Phase 0's background reading (§1.1), cited repeatedly since; actually
implementing it here would mean re-running Phase 2's entire
hazard-then-schedule methodology one level up (the S double-buffer and
accumulator state would need their own cross-Q-tile hazard pass, a new
outer prologue/steady-state/epilogue structure) for a fractional-percent
win. Bad time-to-learning ratio on both axes — the concept isn't new
territory, and the payoff is negligible even at this workload's real
scale. **Decision: not doing this**, and not calling it "still open" going
forward — it was genuinely evaluated, not skipped by default.

**Automated scheduler — worth doing, scoped down from `spec.md`'s literal
"real list-scheduling/software-pipelining algorithms."** The actual reason
it's worth the time: it's the only way to test the hand-schedule's own
unresolved caveat from §11.12 — "no avoidable stalls found" was
explicitly flagged as weaker than "optimal," since no alternative
orderings were ever tried or compared, only checked for correctness.
Skipping the automated scheduler would leave that claim asserted rather
than tested, and would collapse Phase 3's three-way comparison to two-way
(hand vs. Gemmini-serial), losing the actual Itanium-relevant question
(does static scheduling by a real algorithm match hand-tuning?) this
project has been building toward since Phase 0's Decision 2.

**But scoped to what the comparison actually needs**, not full CS243
generality: this problem is a single basic block (no control flow — zero
branch instructions in this ISA, a standing design fact), 3 resource slots
with fixed full-latency occupancy (§11.8's decided model), no register
allocation. A greedy list-scheduler over the same dependency graph already
derived in §11.8 (S double-buffer WAR hazard, P/O cross-slice ordering,
metadata recurrence, all the DMA WAR hazards from §11.15) is sufficient —
no need to build generalized modulo-scheduling machinery for loop-carried
dependencies this problem doesn't have (cross-iteration pipelining was
just ruled out above, so there's no loop-carried scheduling problem to
solve for).

**Prediction, on record before building it so it's checkable
afterward**: given how constraint-bound the hand schedule turned out to
be — matmul-issue is the real bottleneck almost everywhere, SFU/DMA sit
idle because there's genuinely nothing for them to do (not from
scheduling slack, per the exact idle-slack numbers in `isa.md` §4 and
§11.8's hazard pass) — expect the automated result to land very close to
179,397 cycles. If true, that's itself the interesting Phase 3 finding:
confirms this workload's hard architectural constraints (one stationary
operand at a time; the softmax recurrence's sequential chunk dependency)
leave little room for a compiler to either win or lose against hand-tuning
here, unlike Itanium's real-world experience.

**Next**: build the scoped automated scheduler.

### 11.18 Independent audit of `bundle-schedule.md`: two real doc bugs fixed (zero cycle impact), and one potentially major finding about the occupancy model itself

Before starting the automated scheduler, dispatched an independent agent
to sanity-check `bundle-schedule.md` end to end (correctness, hazard
completeness, missed optimizations), specifically to avoid just
rubber-stamping the last several sessions' own work. Full report kept
externally; findings below.

**Real doc bugs found and fixed** (both zero cycle impact — corrections
already applied to `bundle-schedule.md` and §11.15 above):
1. The chunk-boundary DMA prefetch range was off by one — "slice 7 of
   chunks 0–6" should be "chunks 0–5" (chunk 6 and 7's slice 7 have no
   `chunk+2` to fetch).
2. §11.15's V-buffer WAR-hazard gate was wrong — gated on
   `steady-state-stream-v(head 7)` when it should be `load-stationary(V,
   slice 7)`, exact same class of mistake already correctly avoided for
   K one line up (`steady-state-stream-v` doesn't read the V scratchpad
   at all, same as `-qk` doesn't read K's). This also silently caused a
   real internal contradiction the audit caught independently: the old
   gate had `V(chunk+2)` spilling into the next slice's first 160 cycles,
   directly contradicting the "slices 1–6 genuinely idle" claim next to
   it.

**Validation, stronger than what was claimed before**: independent
re-derivation of every table's arithmetic came back clean, every `isa.md`
citation checked out, and — going further than our own "no avoidable
stalls found" framing — the audit proved 179,397 cycles is the *exact*
lower bound achievable under our stated occupancy model (matmul-issue is
the sole bottleneck resource, 99.89% occupied, zero bubbles from cycle
160 onward). No reordering inside the current model can save a cycle;
that's now a proven claim, not a checked-and-not-disproven one. No new
*live* hazard was found in a from-scratch re-enumeration of every shared
resource (S, P, O, metadata, both scratchpad buffers, the array's
stationary registers) — §11.8's hazard pass holds up.

**The major finding, not yet acted on — flagged for real discussion, not
just accepted:** the audit argues the busy-for-full-latency occupancy
model (§11.8) conflates *latency* with *occupancy* for
`steady-state-stream-qk`/`-v`. The 159-cycle figure was itself derived
(§11.4) as `N + D − 1` (32-cycle feed + 128-cycle drain) — a systolic
*pipeline* latency formula, not evidence the array's input port stays
busy for all 159 cycles. Since all 8 heads within a slice's K-phase (or
V-phase) stream against the *same* stationary data (`load-stationary`
loads once per slice), the audit argues there's no physical reason
head `i+1`'s feed couldn't start the cycle after head `i`'s own 32-cycle
feed completes, rather than waiting for head `i`'s full 159-cycle drain —
which would mean real occupancy is 32 cycles/stream, not 159. Claimed
impact if correct: **~33,000 cycles instead of 179,397 (≈5.5× faster)**,
with matmul-issue utilization rising from 18.3% to 100% (cross-checked
independently against the workload's raw MAC-bound floor:
5.369×10⁸ MACs / 16,384 MAC-cycle⁻¹ = 32,768 cycles — matches).

Companion, smaller, and independent of the above: `load-stationary`
(16,384 cycles/Q-tile, 9.1% of the matmul slot) is currently fully
serialized ahead of any stream. A second "shadow" stationary register
(matching Gemmini's real `PRELOAD`/`COMPUTE` split, already cited in this
project — §5.2/`notes.md` line ~280 — but read as mutually-exclusive
occupancy rather than the double-buffered-weights mechanism it actually
is) would let the next slice's `load-stationary` hide under the current
slice's streaming: 128 &lt; 159, fully hideable. Claimed impact: 163,141
cycles (9.06% faster) standalone, and a structural prerequisite for the
bigger claim above (which needs 128 cycles of preload hidden under a
256-cycle feed window).

**Not accepted or rejected yet — deliberately.** This contradicts an
explicit, reasoned §11.8 decision ("physically forced... same physical
PEs"), and if right, invalidates the latency/occupancy model everywhere
in this project, not just this one schedule — real rework across Phase 1
(capacity/timing) and all of Phase 2 (the whole `bundle-schedule.md`) if
adopted, which is a large enough cost that it deserves a real
investigation (checked against `prefill_notes.md`'s actual hardware
hypothesis, not just against internal consistency of the argument) before
committing either way. Deliberately not starting the automated scheduler
until this is resolved, since building it against the current occupancy
model would be wasted work if the model changes.

**Next**: investigate both findings (the `load-stationary` shadow-register
idea and the systolic-pipelining occupancy question) and decide whether
either is worth adopting before building the automated scheduler.

### 11.19 DECISION: adopt the corrected occupancy model — fix this before building the automated scheduler

Investigated both findings from §11.18 directly. Result: **the big
finding is confirmed, not just plausible; the small one is real
precedent but unconfirmed by this project's own evidence; and they are
not independent knobs.** Decision made: fix the occupancy model and
re-derive the hand schedule *before* touching the automated scheduler —
building the scheduler against a known-wrong occupancy model would be
wasted work.

**Why the big finding (occupancy = feed-rate, not full latency) counts as
confirmed, not just argued:** `prefill_notes.md` §3.1 already modeled
this *exact* hardware (single 128×128 array, 1 GHz) in Timeloop back in
Phase 1c, independently of this session, and found QK^T achieves **100%
utilization** — reported cycle count for the full kernel is exactly
2³⁰ = 1,073,741,824. Cross-check: the audit's from-scratch MAC-bound floor
(32,768 cycles/Q-tile for both matmuls combined) ÷ 2 (QK^T's even share
of FLOPs) × 65,536 Q-tiles = 1,073,741,824 — an *exact* match, not
approximate. Three independent sources (Timeloop's real simulation, the
audit's own derivation, this cross-multiplication) agree exactly. This
also directly falsifies §11.8's "physically forced... same physical PEs"
reasoning: if a stream genuinely had to fully drain (159 cycles) before
the next could start, 100% utilization would be mathematically
impossible — the ceiling would be `N/(N+D-1)` = 32/159 ≈ 20%, suspiciously
close to this schedule's actual 18.3%.

**Applies symmetrically to `steady-state-stream-v`, and the evidence is
actually cleaner there.** Same mechanism: `notes.md` §11.4 already derived
`-v`'s 159 cycles "by symmetry" with `-qk`, and within a slice's V-phase
all 8 heads stream against the same stationary V (`load-stationary(V,
slice)` loads once, reused by all 8 `steady-state-stream-v` calls) —
identical structure to K-phase. Checked `prefill_notes.md` §3.1's `·V`
numbers specifically (not just assumed by analogy): the reported "100%"
row used output-stationary dataflow, not the weight/V-stationary dataflow
this ISA actually implements, so that specific number isn't a clean
match — but the *matching*-dataflow row (`primary_v_v2`, weight-stationary,
"matches the Phase 1b hypothesis") only reported 80.63%, and the write-up
explicitly flags that as a local optimum, not the ceiling: the killed
`_v3` deeper-search run "already showed multiple 100%-utilization samples
under v2's architecture" — under that same matching dataflow. So 100%
utilization was directly observed under our actual dataflow for `·V`, not
just inferred by symmetry with QK^T's own (dataflow-mismatched) citation.

**Why the small finding (`load-stationary` shadow register) is treated
with more caution:** checked whether this project's own Gemmini RTL work
(`prefill_notes.md` §4, Phase 1d) already confirms real Gemmini hardware
lets `PRELOAD` overlap a prior `COMPUTE` — it doesn't. §5.1's existing
citation of Gemmini's `PRELOAD`/`COMPUTE` split is only about instruction
*taxonomy* (two separate ops exist, mirrored by our `load-stationary`/
`steady-state-stream` split), not about whether the real hardware
double-buffers weights to let them overlap. Real precedent for the
pattern exists in accelerator design generally, but it's not independently
validated by anything already in this project the way the big finding is.

**Important correction to how the audit framed its own headline number**:
"~33,000 cycles, 5.47×" is not attainable from fixing stream occupancy
alone — the audit's own 512-cycle/slice derivation already assumes
`load-stationary` is hidden (i.e., already assumes the small finding is
also adopted). Disentangled:
- Stream-occupancy fix alone (`load-stationary` still fully serialized):
  ≈49,000 cycles (≈3.6× over 179,397).
  small finding alone (old 159-cycle stream occupancy, `load-stationary`
  hidden): 163,141 cycles (the already-quoted 9.06%).
- Both together: ≈32,768 + prologue/tail overhead ≈ 33,000 cycles (5.47×).

**Blast radius, scoped before committing — smaller than "redo Phase 1 and
2," worth being precise about**: this is fundamentally an
occupancy-*model* correction, not an ISA redesign. Expected to touch:
the latency/occupancy table (§11.4); all of `bundle-schedule.md`'s cycle
numbers (same format/methodology, genuinely different numbers — a
recompute, not a redesign); S's buffer depth in `isa.md`'s capacity table
(§11.8 found double-buffering gives zero margin under the corrected
model — audit suggested ×4 may be needed, needs its own real hazard
re-derivation, not just asserted). Expected **not** to touch: the ISA's
instruction set, field definitions, or bundle width (33 bits) — those
were sized by data/state requirements, never by this timing assumption;
DMA's numbers and hazards (untouched by this, since DMA was never subject
to the drain-vs-feed conflation — it's not a systolic pipeline); the
overall Phase 1 capacity/sizing pass beyond S's own entry.

**`bundle-schedule.md` (179,397 cycles) is now superseded, not deleted** —
banner added at the top of that file. It remains a fully correct answer
*to the wrong occupancy model*; useful as a reference for the
before/after comparison Phase 3 will want, not as the current answer.

**Next, in order — this is the real pickup point for a fresh session:**
1. Re-derive the occupancy model itself: confirm the corrected
   32-cycle/stream occupancy rule precisely (including whether
   `load-stationary`'s 128 cycles are genuinely all "feed," already
   checked informally in this session — every cycle feeds a distinct new
   row, unlike a stream's drain tail, so no analogous conflation there),
   and settle the `load-stationary` shadow-register question on its own
   merits (real hazard/mechanism check, not just precedent) since it's a
   precondition for the full 5.47× figure.
2. Full hazard re-pass under the corrected model — §11.8's hazards don't
   just get tighter margins, at least one (S double-buffering) goes to
   *zero* margin and may need to change (×2→×4 or similar); P/O margins
   shrink from ~1,400 to ~380 cycles per the audit's estimate, needs its
   own real check, not just the audit's number taken on faith.
3. Re-derive `bundle-schedule.md` from scratch under the corrected model
   — same five-segment structure/format already built, new numbers.
4. *Then* build the automated scheduler (§11.17's scoping still applies),
   now against the corrected occupancy model so the comparison is
   meaningful.

### 11.20 Sequencing decision: `load-stationary` shadow register punted; explicit occupancy table for the confirmed fix

**Sequencing decision, made before starting §11.19 item 1**: do the
stream-occupancy fix alone first; treat the `load-stationary`
shadow-register idea as a separate follow-on, not side-by-side. Reasons:
the stream fix is confirmed three independent ways (Timeloop, the audit's
MAC-bound derivation, the cross-multiplication — §11.19), while the
shadow register is unconfirmed by anything in this project (checked: the
existing Gemmini `PRELOAD`/`COMPUTE` citation, §5.1, is about instruction
taxonomy, not a real double-buffered-weights mechanism). Bundling an
unconfirmed change into the same rebuild as a confirmed one would make
every downstream hazard number contingent on an unverified assumption,
and would repeat the exact ordering mistake already corrected once in
this project (interleaving/placement before the base occupancy model was
settled — see the "How to work with me" bullet in `handoff.md`). Net:
`load-stationary` stays fully serialized ahead of its slice's streams for
now; this rebuild targets the ≈49,000-cycle stream-fix-alone figure, not
the ≈33,000-cycle combined one.

**Explicit occupancy table**, resolving §11.19 item 1's "confirm the
corrected rule precisely." The key clarification: *latency* (issue →
result ready) is unchanged for every instruction — nothing about any
`N+D−1` or DMA-bandwidth derivation from Phase 1/§11.4 was wrong.
*Occupancy* (cycles before the slot can accept its next issue) is the
concept that was missing, and it only diverges from latency for the two
streams — the value isn't a new derivation, it's `N` from the existing
`N+D−1=159` breakdown (§11.4), just adopted as occupancy instead of
conflated with latency.

| Instruction | Latency | Occupancy | Why |
|---|---|---|---|
| `load-stationary` | 128 | 128 | No feed/drain split exists for it — every cycle loads a distinct new row into the array (checked informally already, §11.19 item 1's parenthetical); physically forced regardless (same physical PEs). |
| `steady-state-stream-qk` / `-v` | 159 (`N+D−1`, unchanged) | **32** (`N`, the feed count) | This is the actual fix. Next head's feed can start the cycle after the current head's own 32-cycle feed completes — it doesn't need to wait for the 128-cycle drain tail, since the drain is pipelined *behind* the feed of the next head, not blocking it. Confirmed, not just argued (§11.19). |
| `softmax-update` / `softmax-finalize` / `metadata-init` | 32 | 32 | Occupancy=latency here was always a §11.8 *choice* (cheap given ~55% idle slack, avoids OOO-style completion tracking), not a physical claim — untouched by the audit, and SFU's own internal pipeline depth is deliberately unmodeled per the latency table's existing footnote. |
| `load-K/V` | 160 | 160 | Not a systolic pipeline — no feed/drain structure to conflate. Untouched. |
| `load-Q` / `store-O` | 5 | 5 | Same. |

**Next**: full hazard re-pass (§11.19 item 2) under this table — S
double-buffer margin (expected to go to ~zero, may need ×2→×4) and P/O
ordering margin (audit estimate ~380 cycles, needs a real check) are the
two flagged items; metadata recurrence and DMA hazards are expected
unaffected but worth a from-scratch sanity pass since absolute cycle
numbers shift throughout.

### 11.21 S hazard re-derived with real numbers: ×2 leaves ~1 cycle margin, adopted ×4 instead

Worked the S double-buffer hazard cycle-by-cycle under the corrected
model (heads issue 32 apart, but a stream's actual write to S/O trickles
out during its *drain* phase — the last 32 of its 159 latency cycles,
`[issue+128, issue+159]` — which follows directly from the existing
`N+D−1` derivation, not a new assumption). Real numbers, head 0 vs. head 2
(same S-buffer parity under ×2): `qk(head 0)` issues at 288, writes S0
across [416,447]; `softmax-update(head 0)` runs [447,479]; `qk(head 2)`
issues at 288+2×32=352, its earliest S0 write lands at 352+128=480.
**Margin = 480−479 = 1 cycle.** Positive, not negative — ×2 technically
survives — but with zero room for any unmodeled effect (e.g. whether
`softmax-update` genuinely needs *all* 32 rows before starting, which its
own latency table footnote already flags as unverified) or for the kind
of off-by-one this project has already produced twice (§11.12, and again
mid-derivation this session on the head-cadence arithmetic).

**Decision: adopt ×4 instead of ×2.** Reasoning: static scheduling gives
precision against *runtime* jitter, not against *derivation* error — a
1-cycle margin has zero tolerance for the latter, and this project has a
real track record of exactly that failure mode in this class of
computation. Cost is negligible: S already had ~94 KB of accumulator
slack (`isa.md` §5); ×4 costs an extra 32 KB. With ×4 (`head_idx & 3`,
still zero new instruction bits — 2 of the existing 3 head-idx bits are
just reused instead of 1), the same-buffer reuse period doubles from 2
heads to 4: head 4 (not head 2) shares head 0's buffer, issuing at
288+4×32=416, earliest write at 416+128=544. **Margin = 544−479 = 65
cycles** — real headroom instead of a razor's edge. Applied in `isa.md`
§3/§5 (buffer selector, capacity table) and throughout the schedule
rebuild below.

### 11.22 Full hazard re-pass under the corrected model: everything besides S survives, some by construction rather than margin

Went through every item in §11.8's original hazard list against the
corrected occupancy model:

- **Array stationary registers ("one operand at a time")**: re-confirmed,
  and clarified exactly what it requires now that occupancy≠latency for
  streams — `load-stationary` was always gated by the *prior stream's
  full completion* (§11.8's original "forces `load-stationary` to wait
  for the prior stream to finish," true then and unchanged now), never by
  occupancy. This isn't a new rule; it only becomes numerically visible
  now that occupancy (32) and latency (159) diverge for streams — under
  the old model they were equal, so the two phrasings were
  indistinguishable. Consequence: `load-stationary(V, slice N)` issues at
  `qk(head 7)`'s full-latency completion, not its occupancy-freed cycle —
  this produces a real, unavoidable **127-cycle idle bubble** on
  matmul-issue at every K→V and V→K transition (§11.23 below), since no
  shadow register exists to hide it (punted, see "Next" below).
- **P (×8 per-head, WAR across slices)**: real numbers, not just
  "safe by construction" as before. `steady-state-stream-v(head 0)`
  reads P[head 0] during its feed window, done by 639+32=671 cycles
  (slice-N-local). Slice N+1's `softmax-update(head 0)` — same cadence
  derivation as `qk`, starts at Δ287 relative to its own slice start
  (1,022 cycles later) = 1,309 absolute. **Margin ≈ 638 cycles** —
  comfortable, because P's reuse period is once per ~1,022-cycle slice,
  not once per 64-cycle (2-head) window like S. No buffering change
  needed.
- **O accumulator (rescale-then-add ordering)**: confirmed safe, and by
  construction rather than margin — every K-phase `softmax-update`
  (including the last, head 7, finishing at Δ543) completes well before
  the V-phase even begins (Δ639), which falls directly out of the
  `load-stationary` full-latency gating rule above. Not a numeric-margin
  question at all.
- **Metadata (`m`,`l`) recurrence across chunks**: unaffected by the fix,
  confirmed still safe by construction — genuine RAW chain on a single
  in-order SFU slot, which is safe regardless of absolute cycle numbers
  by definition (a slot's own prior write always precedes its own next
  read of it, in-order).
- **`metadata-init` base case**: unaffected, still enormous margin — all
  8 instances finish within the prologue's first ~256 SFU cycles,
  untouched by the stream-occupancy fix.
- **`softmax-finalize`→`store-O` (tail)**: revisited in detail in §11.24
  below — safe, but the margin story changed qualitatively, not just
  numerically.
- **DMA (chunk-boundary prefetch)**: revisited with real numbers in
  §11.23 below — still safe, margin shrinks from ~23,500 to ~8,400
  cycles, nowhere near tight.

**Net: S was the only hazard that actually broke.** Everything else
survives the corrected model, via a mix of real (smaller but comfortable)
numeric margins and by-construction guarantees that were never
timing-dependent in the first place.

### 11.23 `bundle-schedule-v2.md`: full rebuild under the corrected model

New file, not an edit to the old one — `bundle-schedule.md` (179,397
cycles) stays as-is, preserved for the before/after comparison, with its
existing superseded banner now pointing at the replacement.

**Prologue**: identical through cycle 288 to the old schedule — nothing
in it depends on stream occupancy (`load-stationary(K, slice 0)` is gated
by `load-K/V(K, chunk 0)` finishing at 160, `metadata-init`/`load-Q`
cadence is SFU/DMA-only). Diverges at the row after 288: old model's next
event was at 447 (159-cycle spacing), this model's is at 320 (32-cycle
spacing).

**Slice 0 / universal table**: derived cycle-by-cycle (`bundle-schedule-v2.md`
has the full tables). Key numbers: K-phase heads issue 32 apart starting
at Δ128, `load-stationary(V)` gated by `qk(head 7)`'s full latency at
Δ511 (not its Δ384 occupancy-free point — the 127-cycle bubble from
§11.22), V-phase heads issue 32 apart starting at Δ639, slice ends at
Δ1,022. **New slice length: 1,022 cycles**, down from 2,800.

**DMA overlay / chunk-boundary**: same four structural cases as the old
schedule (slice 0 = prologue continuation, slices 1–6 idle, slice 7 =
chunk+2 prefetch, chunks 6–7 = no prefetch) — the reasoning behind them
(DMA in-order, nothing else competing, fires the instant its WAR gate
clears) never depended on occupancy values, so it carries over unchanged.
Only the literal cycle numbers changed. Worked chunk 0→2: `load-K/V(K,
chunk 2)` issues at 7,442 (gated by `load-stationary(K, slice 7)`'s
completion), `load-K/V(V, chunk 2)` at 7,953 — both finish inside chunk
0's own slice 7 window. **Margin ≈ 8,399 cycles**, down from ~23,500 but
nowhere close to tight. Confirmed zero DMA-induced stall anywhere in the
Q-tile (`chunk_start(c+1) = chunk_start(c) + 8,176` holds exactly for all
`c`): within a chunk, all 8 slices reuse the same already-resident K/V
buffer, so matmul-issue never touches DMA mid-chunk; every chunk's
prefetch (checked for chunks 0, 1, and the general `c`≥2 case) finishes
thousands of cycles before it's needed.

### 11.24 Tail re-derived: same rule, genuinely different SFU margin story — and it generalizes to the whole K-phase

`softmax-finalize(head i)` gated by `steady-state-stream-v(head i)`'s
full latency (same reasoning as `softmax-update`/S — needs *all* of O's
rows written, only true at full completion), `store-O(head i)` follows
immediately — same rule as `bundle-schedule.md`. But the numbers reveal a
real qualitative change, not just smaller magnitudes: chunk 7/slice 7
starts at 64,546; `v(head 0)` finishes at 65,344; `finalize(head 0)`
finishes at 65,376 — which is *exactly* `v(head 1)`'s own finish time
(65,344+32), because `v`'s occupancy (32) now exactly equals
`finalize`'s own occupancy (32). Every `finalize(head i+1)` starts the
instant `finalize(head i)` ends — SFU runs the entire tail at **zero
idle cycles**, versus the old model's huge slack (159-cycle `v` cadence
vs. 37-cycle `finalize`+`store-O`). Still hazard-free (32=32 is an exact
match, not a shortfall) but resting on a coincidence, not margin — same
category of concern as S's 1-cycle case in §11.21, and it directly
revives the SFU latency table's own "internal pipeline depth not
modeled, unlikely to matter" caveat as a live question rather than a safe
assumption. **`store-O`/DMA still has real slack** (~27 idle cycles
between consecutive `store-O`s) — this exact-saturation effect is SFU-only.

**Generalizes beyond the tail**: the same back-to-back, zero-gap pattern
already exists in the K-phase's `softmax-update` sequence in the Slice 0
table (§11.23) — 447→479→511→543→575→607→639→671→703, fully contiguous.
So SFU flips from "~55% idle, comfortable slack" (the original Phase 1
characterization) to **100% utilized during every active window** (K-phase's
256-cycle burst, and the tail), though its *overall* duty cycle per slice
is still ~25% (idle during `load-stationary` and all of V-phase, where
nothing schedules SFU work). Not a new hazard, but a real shift in SFU's
risk profile that's worth remembering wherever the "SFU has slack, this
detail doesn't matter" reasoning gets reused later.

**Total Q-tile latency: 65,605 cycles** (chunk 7/head 7's `store-O`
completion) — down from 179,397, a **2.73× speedup** from the
stream-occupancy fix alone (`load-stationary` still fully serialized,
shadow register not adopted). Full tables: `bundle-schedule-v2.md`.

### 11.25 Hand-schedule deliverable (spec.md's Phase 2 ask) is now complete under the corrected model; two real items remain before Phase 2 is fully closed out

`bundle-schedule-v2.md` satisfies `spec.md`'s literal Phase 2 hand-schedule
requirement — same status `bundle-schedule.md` held before the occupancy
bug was found. Two things remain, both already scoped, neither part of
the hand-schedule deliverable itself:

1. **`load-stationary` shadow register — worth a real investigation now,
   given a real number.** Punted in §11.20 pending confirmation (Gemmini's
   own RTL doesn't validate double-buffered weight overlap, only the
   PRELOAD/COMPUTE instruction split). The 127-cycle-per-transition bubble
   quantified in §11.22/§11.23 (~254 cycles/slice, ~16,300 cycles total
   across 64 slices, ~25% of the streaming portion) is a real, non-trivial
   cost — unlike cross-iteration pipelining (§11.17, ≈0.09% of runtime,
   correctly dropped), this clears the bar for being worth a real
   mechanism/hazard check. If adopted, it would mean a further
   `bundle-schedule-v3.md` rebuild (`bundle-schedule-v2.md` would become
   the next before/after baseline, same role `bundle-schedule.md` plays
   for `-v2`).
2. **Automated scheduler (§11.17's scoped-down list-scheduler)** — not
   started. Same logic that motivated fixing the occupancy model before
   ever starting this applies again: building it against a model that
   might still change (if the shadow register is adopted) risks wasted
   work, so item 1 should resolve first.

**Next**: investigate the shadow-register mechanism on its own merits
(real hazard/mechanism check, matching how S/P/O were each closed out
with real numbers, not precedent alone); then the automated scheduler.

### 11.26 Shadow-register investigation: real, and confirmed in Gemmini's actual RTL — stronger evidence than what was available at the punt

Checked three independent primary sources rather than relying on the
project's earlier (higher-level) look:

1. **Gemmini's README/ISA docs**: silent. `matmul.preload` commits in one
   cycle, then "the systolic array remains idle until the subsequent
   OS/WS specific instructions are seen" — no discussion of overlap.
2. **The original Gemmini paper** (Genç et al.): describes the tile/PE
   pipeline architecture generally, no discussion of back-to-back
   weight-stationary throughput or PE-level weight double-buffering.
3. **Gemmini's actual RTL source, `PE.scala`**: settles it, positively.
   Every PE has **two registers for the stationary value, `c1` and `c2`**,
   gated by a `propagate` control signal (`COMPUTE=0`/`PROPAGATE=1`). In
   weight-stationary mode: when `prop === PROPAGATE`, `c1` feeds forward
   into the compute datapath while `c2` simultaneously receives new
   input; roles swap otherwise. Real, hardware-level double-buffered
   stationary registers — structurally exactly the shadow-register idea.

**Why this is stronger than the original punt's basis**: the punt
(§11.19/§11.20) only checked the README/instruction-taxonomy level
(`PRELOAD` vs. `COMPUTE` exist as separate ops) — never opened `PE.scala`.
Same category of gap Phase 1d's own §4.6 already flagged and corrected
once ("pure grep/naming archaeology has a real ceiling — reading the
actual Chisel source directly is what settles it," `prefill_notes.md`
§4.8). Doing that here reverses the earlier "unconfirmed" verdict.

**What's still genuinely open, not swept under the rug**: the public docs
don't describe this mechanism at the *instruction* level at all — not
fully confirmed whether Gemmini's compiler exposes a way to trigger a
full ~128-cycle preload of an independent next tile overlapping a full
~159-cycle previous compute (our exact use case), vs. this `c1`/`c2`
ping-pong being a narrower internal detail of one preload→compute
transition. Fully closing that would need reading `ExecuteController.sv`'s
actual sequencing logic (same primary-source approach that resolved the
axis-routing question in Phase 1d) — not done here. Judged sufficient to
adopt given real RTL-level confirmation of the underlying mechanism,
consistent with how much confidence this project's other Gemmini-grounded
decisions have required.

**Decision: adopt.** Real, not fabricated, and better-grounded than
general accelerator-literature precedent (TPU's Weight FIFO/double
buffering was also checked as corroborating context — real and
well-documented, but a different accelerator; the Gemmini RTL finding is
the load-bearing evidence here since it's this project's own reference
implementation).

### 11.27 Real impact re-derived — "cost → 0" was an overstatement, caught before it propagated into the rebuild

Initial framing (this session) claimed the shadow register makes
`load-stationary`'s cost "basically 0." Checked and corrected before
building anything on it:

**What's NOT true**: `load-stationary` cannot overlap with the *active*
issuance burst of a stream phase — during a phase's 8 heads (256 cycles,
occupancy 32 × 8, back-to-back, zero gaps), the single matmul-issue slot
is already fully packed. No free issue cycle exists there regardless of
the shadow register.

**What IS true**: the shadow register eliminates the *bubble after the
last head* — `load-stationary(V, slice N)` now issues right when
`qk(head 7)`'s occupancy frees the slot (Δ384) instead of waiting for its
full 159-cycle latency (Δ511), saving exactly the 127-cycle gap
(`latency−occupancy`). Symmetric at the V→K transition.

**But `load-stationary`'s own 128 cycles remain real, unavoidable
occupancy** — the loaded operand isn't *active* until the load's own full
128 cycles complete, and nothing else can issue on the single matmul-issue
slot while it's mid-load. Real new slice length, recomputed component by
component: `128 (load-stat K) + 256 (qk, 8×32) + 128 (load-stat V) + 256
(v, 8×32) = 768` — down from 1,022, **not down to ~0 additional cost**.
Cross-checked via issue-to-issue spans: 224 (qk head0→head7 issue-to-issue)
+ 160 (gap: 32 remaining qk(head7) occupancy + 128 load-stat) + 224 (v
span) + 160 (gap) = 768. Matches.

**Interesting cross-check against the original audit's numbers**: 64
slices × 768 ≈ 49,150 lands close to the audit's original "~49,000,
stream-fix-alone" estimate (§11.19) — which was already flagged as too
optimistic for stream-fix-*alone* (real answer there, without the shadow
register, is 65,605, §11.24). Looks like that estimate was actually
closer to what *both* fixes together produce, just mislabeled at the
time. The audit's "~33,000, both together" figure is also now revealed as
too optimistic — it implicitly assumed something close to zero
`load-stationary` cost, which isn't achievable without a structurally
separate weight-load port (a bigger ISA change, not part of this
adoption). Real answer for both fixes together: see §11.28.

### 11.28 Hazard re-pass under the shadow-register model: S untouched, P and O both shrink by exactly 127 cycles (same root cause), one old justification breaks and needs replacing

Checked every cross-boundary hazard against the new model rather than
assuming "no S/P/O buffer issues" was free:

- **S (×4)**: confirmed **unaffected**. S's hazard is entirely
  intra-K-phase (head `i` vs. `i+4`, both inside one phase's 224-cycle
  burst) — untouched by a fix that only changes the transition *between*
  phases. Margin stays 65 cycles.
- **P (WAR across slices)**: margin shrinks **638 → 511** cycles. Real
  shrinkage, not "unaffected" — P's hazard spans exactly the K→V-then-next-K
  boundary that just compressed by 127 cycles per transition. Still safe.
- **O (rescale-then-add)**: margin shrinks **320 → 193** cycles — same
  127-cycle shrink, same root cause (both P and O are gated by the same
  compressed transition). **The old justification literally breaks**:
  §11.22 argued safety via "every K-phase `softmax-update` finishes
  before any V-phase `v` starts" — no longer true (`v(head 0)` now issues
  at Δ512, before `softmax-update(head 7)` finishes at Δ543). Not
  actually unsafe: the real requirement is per-head
  (`softmax-update(i)` vs. `v(i)`, same `i` — different heads' O
  locations never conflict, already ×8), and that margin is a *constant*
  193 cycles regardless of head (`v(i)` issues at `512+32i`,
  `softmax-update(i)` finishes at `319+32i` — the `+32i` terms cancel).
  Worth remembering: a hazard justification that held under one model
  doesn't automatically transfer to the next — this one needed
  re-deriving from scratch, and happened to still hold, for a different
  reason than before.
- **Metadata recurrence, DMA**: genuinely unaffected (same-slot in-order
  RAW chain; DMA margin shrinks proportionally with the shorter chunk —
  6,240 cycles at the tightest point, still nowhere close to DMA's own
  ~320-cycle need).

**`bundle-schedule-v3.md` built against this model. Total: 49,476
cycles** — 1.33× faster than `-v2` (65,605), 3.63× faster than the
original (179,397), still 1.51× above the theoretical 32,768 MAC-bound
floor (real, structural gap: `load-stationary` still consumes 33% of
every slice's matmul-issue-slot cycles, since it shares the single issue
slot with the compute streams — the shadow register fixed the *data*
hazard, not the *issue-bandwidth* constraint). Full tables:
`bundle-schedule-v3.md`.

**Phase 2 status**: hand-schedule deliverable now fully settled at
49,476 cycles. Per `spec.md`, Phase 2 requires *both* a hand-scheduled
and an automated bundle sequence — the hand-schedule half is done;
**the automated scheduler (§11.17's scoped-down list-scheduler) is the
one remaining item before Phase 2 as a whole is closed out.**

### 11.29 Independent Opus audit of `bundle-schedule-v3.md` and `isa.md`: one finding of the same magnitude class as the shadow register, one real bug in the automated-scheduler handoff spec, one recorded number wrong by 7×

Dispatched before starting the automated scheduler, same rationale as
§11.18's audit — catch what earlier passes (including that first audit)
missed, rather than build the next phase against an unverified number.
Full findings below; all arithmetic and hazard margins in
`bundle-schedule-v3.md` were independently re-derived and came back
clean — every finding here is structural, not a cycle-count error.

**1. `load-stationary`'s remaining 256 cycles/slice is more avoidable
than §11.24's "would need a bigger ISA change" framing suggested —
confirmed against real Gemmini RTL, not just argued.** The audit read
`ExecuteController.scala` directly (the file §11.26 flagged as the
remaining open question) and found a real overlap-compute-and-preload
path (`perform_mul_pre`, independent per-operand fire counters) — genuine
instruction-level overlap of a preload with a compute, not just the
`PE.scala` register-level double-buffer §11.26 already found. Combined
with `PE.scala`'s weight-chain/psum-chain separation (`mac_unit.io.in_b
:= c2` vs. `io.out_b := mac_unit.io.out_d` — physically different wires),
a full weight load costs **zero cycles of the activation/psum datapath**
in the reference hardware — overlappable with *active streaming*, not
just with drain. Recomputed schedule if this ISA had an independent
weight-load feeder (not sharing the matmul-issue slot): slice = `256 (qk)
+ 256 (v)` = 512, both `load-stationary`s fully hidden inside them. Total:
**33,220 cycles** (prologue 288, 64×512=32,768, tail overhead 452) — a
further **1.49×** over `-v3`, landing 1.4% above the 32,768 MAC-bound
floor. This is genuinely a **new instruction-set decision** (a 4th slot,
or a fused `stream-and-preload` opcode), not adoptable as a same-ISA
timing fix the way the shadow register was — the audit is explicit that
Gemmini's own fused mul-pre couples preload+compute as one command with
shared `total_rows`, so spreading a 128-row preload across four 32-row
streams isn't literally what Gemmini's ISA does; this project would need
to design the affordance itself. Real precedent for the *mechanism*
(separate weight/psum datapaths), not for this project's specific
encoding of it. **Not yet decided whether to pursue — see "Next" below.**

**2. Real correctness bug in `handoff.md`'s automated-scheduler
handoff, now fixed.** The hazard list stated "array stationary
registers... enforced automatically by slot serialization" — true under
v1/v2 (occupancy=latency for streams made this a free consequence), but
**false under v3**: `load-stationary` can now issue while a *different*
prior stream is still draining out of the other register, so slot
serialization no longer implies register safety. The real constraint
(derived by the audit, re-verified here): `load-stationary` writing
register X must wait for X's *last reader* — the stream phase *two*
phases back, not one — to fully drain (full latency). Margin in the
actual hand-schedule: **257 cycles**, safe, but nowhere stated as its own
explicit constraint before this — a greedy scheduler built to the old
spec had no guard against violating it. Fixed in `handoff.md`'s hazard
list directly (this session, before any scheduler work started against
the old spec).

**3. Cross-Q-tile pipelining's recorded math (§11.17) was wrong by
~7×, though the conclusion survives.** Audit re-derived the percentage
independently, as asked (rather than trusting the "conclusion doesn't
change so not worth recomputing" note from §11.17 itself — the *numerator*
had also changed, not just the denominator that note flagged). Real
total non-steady-state overhead: 160-cycle prologue + a **164-cycle tail
gap** §11.17 never counted (matmul idle 49,312→49,476 while SFU/DMA
finish) = 324 cycles against 49,476 total = **0.655%**, not 0.09%. Also:
the prologue half is cheaper to capture than §11.17 claimed ("would mean
redoing the whole hazard-then-schedule methodology one level up") — the
audit shows it's just two extra `load-K/V` prefetch instructions with
~6,500 cycles of DMA slack, not a new prologue/epilogue structure. Still
**correctly left dropped** (0.655% is still genuinely negligible), but
the recorded reasoning was wrong on the number, on what's at stake, and
on the cost of capturing the prologue half specifically. Also checked (at
the project owner's request, re: a similar issue found in a different
project): K/V double-buffer parity, `load-Q` timing, and aggregate HBM
bandwidth all check out clean across Q-tile boundaries — no analogous
cross-Q-tile problem found beyond the already-known prologue/tail gap.

**4. SFU-saturation "curiosity" (§11.24) is a real sensitivity cliff, not
just a noted oddity, and it's the actual blocker for capturing finding
3's tail half.** Under any cross-Q-tile overlap attempt, SFU would need
24 instructions (8 `finalize` + 8 `metadata-init` + next tile's 8
`softmax-update`) through a window that only fits ~22 — **oversubscribed
by ~160 cycles**, and delaying the next tile's `softmax-update(head 0)`
to compensate breaks the S hazard outright (`qk(head 4)` of the new tile
would write S[0] before `softmax-update(head 0)` reads it). Separately:
the schedule is only correct because SFU's occupancy (32 cycles — already
flagged in the latency table as "no real hardware anchor") happens to be
≤48; at exactly 32 there's zero absorption anywhere in the schedule for
that unanchored number to be wrong by. Also: §11.8's original reason for
choosing occupancy=latency for SFU ("costs nothing given their slack")
no longer holds under v3 (SFU has no slack in its active windows) — the
choice still doesn't cost cycles, but §11.28's hazard re-pass didn't
revisit this specific stale justification the way it did O's.

**5. Fallback if the ISA can't change: `tile_q`=64 gives 16.7% for zero
encoding cost, but is blocked by 4,096 B of accumulator budget** — and
`prefill_notes.md` §7 already logs accumulator capacity as swept-never
"real free parameter (128–512 KB)." At 512 KB (Gemmini's own published
"BigSP" config), `tile_q`=64 fits comfortably. **Subsumed by finding 1**
if that's pursued (a weight-load slot makes `tile_q` irrelevant to this
overhead entirely) — recorded as the fallback, not an addition.

**6. Minor, all fixed this session, zero cycle impact**: `isa.md`'s S
buffer justification said "head `i+2`" (survives from the ×2 design,
should be `i+4` under ×4 — a real bug introduced when §11.21 did the ×2→×4
edit, caught by the audit, not self-caught); two mislabeled "idle" cells
in `bundle-schedule-v3.md`'s universal table (Δ383 matmul-issue and Δ512
SFU were both still mid-occupancy, not idle — `bundle-schedule-v2.md` has
the identical defect, not fixed there since it's already superseded); the
SFU duty-cycle figure carried over from `-v2` (25%) without updating for
the shorter slice (real number: 256/768=33%); `notes.md` §11.26 cites
`ExecuteController.sv` — Gemmini is Chisel, the real file is
`ExecuteController.scala`, the `.sv` is generated output. Also flagged,
not fixed (no decision needed, just worth recording): one Q-tile's fully
unrolled program is 49,476 bundles × 33 bits ≈ 199 KB, of which only
~1.2% of slot-entries are non-NOP — never previously computed, but the
fixed-width bundle-layout conclusion (`isa.md` §4) is if anything
*strengthened* by it, not threatened (occupancy went from ~1% under v1 to
~3.5% under v3, i.e. progressively less wasteful, not more).

**What checked out clean, independently re-derived rather than trusted**:
every cycle number in `bundle-schedule-v3.md` (slice/chunk lengths, the
full tail table, all three speedup ratios); every hazard margin (S=65,
P=511, O=193, all re-derived from scratch); the full DMA WAR chain
end-to-end; `isa.md` §5's capacity tables; the shadow-register premise
itself (finding 1 if anything *strengthens* confidence in it, not
weakens it — no risk of reverting toward `-v2`).

**Next, genuinely undecided — a real judgment call, not resolved here**:
whether finding 1 (a weight-load-slot ISA extension, ~1.49× further,
33,220 cycles) is worth pursuing now, before the automated scheduler, the
same way the shadow register was — or whether to bank it as a known,
quantified future opportunity and proceed to the automated scheduler
against the current 49,476-cycle `-v3` target as planned.

### 11.30 Decision: pursue the ISA extension next; automated scheduler's status changed from "required" to "open"

Investigated finding 1's feasibility before deciding (not just estimating
magnitude): checked whether the array can physically support a weight-feed
and an activation-feed running concurrently, since that's the actual
crux of whether this is buildable or a much bigger redesign than it
first looks. Found real, positive evidence: Gemmini's `MeshWithDelays`
module feeds three independent operands (`A`, `B`, `D`) into the array
**simultaneously, one row each per cycle, as its normal mode of
operation** — not a special case invented for this question. `B` (the
stationary/weight path) and `A` (the moving/stream path) are already on
physically separate channels in the reference hardware; that's the actual
precondition that makes the audit's `perform_mul_pre` overlap finding
(§11.29) possible at all. So the underlying mechanism isn't speculative —
it's a real, already-provisioned feature of the hardware this project
targets. The gap is entirely on this project's own bundle-format side
(one shared opcode field for all matmul-issue work), which is a bounded,
well-precedented extension to design, not an open-ended redesign.

**Decision, made explicitly rather than defaulting to `spec.md`'s
checklist**: pursue this ISA extension next, sequenced before the
automated scheduler (same "settle the model before building against it"
logic as every previous sequencing call this session). Real reasoning for
prioritizing it over the scheduler: comparable magnitude to the shadow
register (1.49× vs. 1.33×), genuinely new learning content (the first
time this project has had to ask how many independent operand paths the
array really has and whether the ISA exposes them — a real codesign
question, not a rehash), and confirmed to be the last major lever
(1.4% remains after it, matching the user's own instinct going in).

**`spec.md`'s Phase 2 "both a hand-scheduled and an automated sequence"
requirement is explicitly being treated as a starting point, not a
binding constraint** — direct correction from the project owner to a
framing I'd been carrying since §11.25 (that the automated scheduler was
"the one remaining item... required before Phase 2 is closed out").
`spec.md` was already established as a non-binding checklist for scope
decisions once before (§11.17's own framing: "`spec.md` is a starting
point, not the reason to do work that isn't worth its time cost") — this
extends that same stance to whether a stated deliverable happens at all,
not just to optional side-quests. **Automated scheduler status: open,
not required, to be revisited after the ISA extension** — may or may not
happen depending on how that revisit goes.

**Next**: design the ISA extension (encoding approach, hazard
re-derivation under genuine A/B-path concurrency, capacity recheck),
target 33,220 cycles, `bundle-schedule-v4.md`. Full brief for a fresh
session: `handoff.md`'s "Starting fresh on the ISA extension" section.

