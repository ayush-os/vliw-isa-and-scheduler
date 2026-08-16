# Prefill Attention VLIW ISA

A from-scratch VLIW instruction set and hand-scheduled bundle sequence for fused online-softmax attention prefill on a 128×128 systolic array — reaching **1.4% of the theoretical MAC-bound floor**, with an RTL-grounded finding that wide-bundle co-issue itself contributed **zero net cycles** to that result.

`ISA design` · `VLIW bundle scheduling` · `systolic array codesign` · `Gemmini RTL (Chisel)` · `hazard analysis`

- **33,220 cycles/Q-tile**, 1.4% above the 32,768-cycle theoretical floor — **5.4× faster** than a naive full-latency-occupancy schedule, reached via 3 independent architectural fixes, each confirmed against Gemmini's actual Chisel RTL, not assumed.
- A **`PE.scala`-confirmed weight shadow register** and a **`MeshWithDelays`-confirmed concurrent weight-load port** turned a 33%-of-matmul-slot bottleneck into a dedicated bundle slot — a real ISA *encoding* decision, not a scheduling trick.
- **Reproduced the identical 33,220-cycle result under a single-issue (non-VLIW) dispatch model** — wide-bundle multi-slot co-issue buys nothing for this workload's shape; the real payoff was static latency-exactness and zero dynamic hardware, not instruction-word width.

---

## What's here

- **[`isa.md`](isa.md)** — a 4-slot, 36-bit VLIW bundle format (matmul-issue, weight-load, vector/scalar-unit, DMA), 9 instruction types, every field sized against real scratchpad/accumulator capacity.
- **[`schedule.md`](schedule.md)** — the cycle-accurate hand schedule against that ISA: resource model, every hazard margin derived, full prologue/steady-state/tail derivation.

Target workload: fused, online-softmax attention prefill (Flash-Attention-style), GQA group size 8, `tile_q`=32/`tile_k`=1024, weight-stationary dataflow — a fixed-shape idealization (`batch`=32, `seq_len`=8192, `n_heads`=64).

## Why static scheduling, here specifically

VLIW's premise — expose ILP to the compiler instead of discovering it in hardware at runtime — famously failed for Itanium (unpredictable branches, unpredictable memory latency) and just as famously works for Groq's inference chips and DSPs like TI C6000/Hexagon, where the workload is regular enough for the compiler to actually get it right. This project's loop nest checks clean against that distinction: trip counts fixed at compile time, no data-dependent branching, and — the real mechanism, not just "DMA is fast" — every load is *scheduled* ahead of time from a scratchpad, not triggered reactively by a cache miss. No dynamic/out-of-order hardware anywhere in this design.

## The optimization story

Three real, independently-derived findings — not incremental tuning — each verified against Gemmini's actual RTL before being adopted, not assumed from documentation:

| Model | Cycles/Q-tile | Fix | Verified against |
|---|---|---|---|
| Naive (full-latency occupancy) | 179,397 | baseline | — |
| Corrected occupancy model | 65,605 | streaming instructions occupy the issue slot for 32 cycles (the feed count), not their full 159-cycle pipeline drain | cross-checked against a Timeloop simulation of the same hardware |
| + weight shadow register | 49,476 | a second physical stationary register lets the next weight load start before the active stream fully drains | `PE.scala`: real `c1`/`c2` double-buffered registers, gated by a `propagate` signal |
| + weight-load bundle slot | **33,220** | weight-load and streaming issue through genuinely separate physical ports — the ISA's original single shared opcode field couldn't express that concurrency | `MeshWithDelays` (independent per-operand ports) + `ExecuteController.scala`'s `perform_mul_pre` (real preload/compute overlap) |

Final gap to the theoretical floor (`5.369×10⁸` MACs / `16,384` MAC/cycle = 32,768 cycles) is entirely prologue (288 cycles) + tail drain (164 cycles) — structural startup/drain cost with no preceding or following phase to hide inside, not unclaimed scheduling slack. Every one of the 64 steady-state slices already runs at exactly the MAC-bound rate.

## Does the wide bundle actually matter?

Checked directly rather than assumed: re-derived the schedule under a **dispatch-width-1** model — only one instruction issued per cycle, globally, across all four slot types — while keeping each unit's real asynchronous execution once dispatched (matching how a real accelerator's host-issued, async-engine model actually works).

Went through every same-cycle collision in the schedule by hand. There were almost none — one per steady-state slice, a few in the prologue and tail — and every one resolved for free by delaying whichever instruction had real slack (`softmax-update`, `metadata-init`, DMA all have tens-to-thousands of cycles of margin) rather than the one that didn't (`load-stationary`'s 1-cycle-margin register-refill window).

**Result: identical 33,220 cycles.** Mechanistically: matmul-issue dominates 98.6% of the schedule but only needs to *dispatch* once every 32 cycles despite being occupied continuously — leaving far more free dispatch cycles than the other three slot types' combined 688 instructions ever need. Dispatch bandwidth was never the real constraint; each unit's own long occupancy was. The literal "pack multiple instructions into one wide word" mechanism of VLIW contributed nothing measurable here — what actually paid off was eliminating dynamic hardware entirely (exact, compile-time-known latencies down to a real 1-cycle hazard margin, trusted because nothing at runtime can violate it) and exposing a real hardware concurrency the ISA's original encoding couldn't express. Different lever than "bundle width," same static-scheduling bet.

## Bundle layout

```
┌─────────────┬─────────────┬─────────┬──────────────────────┐
│ matmul-issue│ weight-load │   SFU   │         DMA          │
│    4 bits   │   6 bits    │ 5 bits  │       21 bits         │
└─────────────┴─────────────┴─────────┴──────────────────────┘
                     36-bit bundle, fixed-width per slot
```

Fixed-width chosen over a compact/variable encoding deliberately: idle-slot cost here is bits, not cycles, and fixed-offset decode matches real precedent (Groq's own dispatch is described as "144-wide VLIW instructions," a fixed format, in a chip whose whole thesis is minimal instruction-control-unit area).
