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
