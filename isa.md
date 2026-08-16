# Prefill Attention VLIW ISA

A minimal VLIW instruction set for a 128×128 systolic-array accelerator
running fused, online-softmax attention prefill (Flash-Attention-style),
targeting a from-scratch hardware hypothesis cross-checked against real
Gemmini RTL at several points — the softmax `Normalizer` match, the
`PE.scala` stationary-register mechanism, and the `MeshWithDelays`/
`ExecuteController.scala` concurrent-port evidence, all cited inline
below where they ground a specific design decision.

**Workload constants this ISA is built against** (fixed-shape): `batch`=32,
`seq_len`=8192, `n_heads`=64, `n_kv_heads`=8 (GQA group size = 8),
`d_head`=128, array=128×128, `tile_q`=32, `tile_k`=1024 (8 K/V chunks per
sweep, 256 Q-tiles per sequence).

**Four bundle slots**, each mapping to one heterogeneous hardware unit:
matmul-issue, weight-load, vector/scalar-unit (SFU), DMA. 9 instruction
types total, 36-bit bundle. Weight-load is a separate slot from
matmul-issue specifically so a weight preload can issue *concurrently*
with active streaming: real Gemmini feeds the stationary (`B`) and moving
(`A`/`D`) operands through independent physical ports every cycle
(`MeshWithDelays`), and `ExecuteController.scala`'s `perform_mul_pre`
shows genuine preload/compute overlap in the reference hardware. A single
shared opcode field can't express that concurrency — hence its own field.
No dynamic/out-of-order behavior anywhere — every field below is either a
hardware/workload constant (zero bits) or a value the compiler computes
and embeds at compile time (static, not runtime-resolved).

---

## 1. Matmul-issue slot (2 instructions)

### `steady-state-stream-qk`
Streams the current Q tile through the array (K currently stationary),
producing raw S in the accumulator. One instance covers exactly one
128-wide array sub-pass (`tile_k`/128 = 8 sub-passes per chunk, matching
`load-stationary`'s slice-idx) for one head — issued once per (chunk,
slice, head).

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `steady-state-stream-qk` |
| head-idx | 3 bits | Addresses the Q source (fixed Q-region-base + idx). Destination (raw S) is one of **four** buffered accumulator locations, selected by head-idx's low 2 bits (`head_idx & 3`) — zero new bits, since head-idx is already carried for Q addressing. Buffered (not single) because `softmax-update(head i)` must finish reading S before `steady-state-stream-qk(head i+4)` can safely overwrite it (the ×4 buffer means heads `i` and `i+4` share a location) — a single shared location would force a stall every head transition. ×4, not ×2 — ×2 leaves only ~1 cycle of real margin under the actual occupancy model (see `schedule.md`); ×4 gives ~65 cycles of real headroom. |

- No length field — always `tile_q`=32.
- No dataflow-select field.
- Transpose **permanently disengaged** (moving operand funnels into one PE over time, matching row-major storage naturally).

### `steady-state-stream-v`
Streams the current P tile through the array (V currently stationary),
accumulating into the output. Same per-(chunk, slice, head) granularity as
`-qk`.

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `steady-state-stream-v` |
| head-idx | 3 bits | Same index resolves *both* the P source (fixed P-region-base + idx) and the output-accumulator destination (fixed output-region-base + idx) — two different hardcoded bases, one index. |

- No length field, no dataflow-select field, transpose permanently disengaged (same reasoning as `-qk`).
- Kept as a separate opcode from `-qk` despite an identical field shape (both just head-idx): the opcode is what tells hardware *which pair* of hardcoded bases to combine with the index — the underlying operation and addressing target genuinely differ, the bit-width match is coincidental.
- **Head-idx-only addressing for P requires a specific scheduling constraint**: each 128-wide slice's K-phase (`steady-state-stream-qk` + `softmax-update` for all 8 heads) must be immediately followed by that *same* slice's V-phase (`steady-state-stream-v` for all 8 heads) before the next slice begins — never batching multiple slices' K-phases before their V-phases. This keeps only one slice's 8 P-values alive at a time. Batching would need a 6th field (a slice-idx alongside head-idx, asymmetrically so, since O's addressing would stay head-idx-only) for zero offsetting benefit — total array-reload count is identical either way.

---

## 2. Weight-load slot (1 instruction)

Given its own field so `load-stationary` can issue concurrently with the
stream instructions instead of serializing on a shared opcode field —
real Gemmini's separate physical ports (above) are what make this a
genuine hardware capability, not just an encoding trick.

### `load-stationary`
Moves a chunk of K or V from scratchpad into the array's stationary
registers.

| Field | Width | Meaning |
|---|---|---|
| valid | 1 bit | 0 = idle this cycle (no load being triggered); 1 = trigger a new load |
| src | 5 bits | 2 bits: buffer-select {K1,K2,V1,V2}; 3 bits: which 128-wide slice within the chosen 1024×128 double-buffered chunk |

- No destination field (loads into the array's internal registers, not an addressed memory location).
- No length field — always exactly 128 wide (array's physical PE-row count, hardware constant).
- No dataflow-select field — weight-stationary dataflow is a compile-time Chisel constant on real Gemmini, not runtime-selected.
- No transpose bit — transpose is **permanently engaged** for this instruction (the stationary operand always fans out spatially across many PEs at once, which always mismatches row-major storage). The instruction itself is the proxy for transpose-on, not a runtime choice.
- No opcode needed — this slot has exactly one instruction type, so the valid bit alone distinguishes "fire" from "idle." (Contrast with matmul-issue/SFU/DMA, which each need a real opcode to pick among multiple instruction types.)
- **Occupancy/latency: 128 cycles** — the array's physical PE-row count, a hardware constant.
- **Fire-timing is the real design constraint this slot creates**: to get the full concurrency benefit, the trigger must fire as early as the target register's prior occupant allows — that occupant's *last reader* (the same-type stream instruction, two phases back) must fully drain (full latency, 159 cycles, not just its 32-cycle occupancy) before this can safely write. Worked cycle-by-cycle in `schedule.md`: the legal window is exactly 1 cycle in steady state — real and confirmed by a full trace, genuinely tight, no additional buffering needed to make it work.

---

## 3. Vector/scalar-unit (SFU) slot (3 instructions)

Grounded in the actual online-softmax recurrence, per head, over K/V
chunks `j=1..8` within one Q-tile (`m_0=-inf, l_0=0, O_0=0`; `S_j` produced
by the matmul slot):

```
softmax-update (once per chunk):
  m_j     = max(m_{j-1}, rowmax(S_j))
  P_j     = exp(S_j - m_j)                    → scratchpad
  alpha_j = exp(m_{j-1} - m_j)
  l_j     = alpha_j · l_{j-1} + rowsum(P_j)
  O_j     ← alpha_j · O_{j-1}                  rescale only; the += P_j@V_j
                                                add is steady-state-stream-v's job

softmax-finalize (once, after chunk 8):
  O_final = O_8 / l_8
```

Matches real Gemmini's `Normalizer` module exactly: running max/sum +
max-subtracted `iexp` is update-side hardware, the hardfloat `1/sum`
divide is separate finalize-side hardware.

### `softmax-update`
Issued once per (chunk, slice, head) — one 128-wide array sub-pass'
contribution for one head, matching `steady-state-stream-qk`'s granularity.

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `softmax-update` |
| head-idx | 3 bits | Resolves metadata (`m`,`l`), output (`O`), and P against three different hardcoded bases. |

- **Reads**: S (one of four buffered accumulator locations, selected by head-idx's low 2 bits — zero new bits, read-only — never written by this instruction), metadata (head-idx), output `O` (head-idx).
- **Writes**: metadata (head-idx), output `O` (head-idx — rescale written in place), P (scratchpad, head-idx).
- `m`/`l`/`O` are persistent, incrementally-updated state — exactly one
  instance per head regardless of scheduling. **P is not** — it's a fresh
  value per (head, slice), and staying at 3-bit head-idx (rather than
  needing a slice-idx too) depends on the slice-interleaved scheduling
  constraint described under `steady-state-stream-v` above.

### `softmax-finalize`

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `softmax-finalize` |
| head-idx | 3 bits | Resolves metadata `l` and output `O` in the accumulator (reads); resolves the scratchpad output slot (write). |

- **Reads**: `l` and `O`, both accumulator, head-idx.
- **Writes**: `O_final` to a **dedicated scratchpad region** (sized for the 8 per-head output tiles) — not back into the accumulator. DMA is HBM↔scratchpad only, never accum-facing (§4), so the finished output has to land somewhere DMA can already reach.

### `metadata-init`
Issued once per head, at the top of each Q-tile — writes the online-softmax
recurrence's initial state (`m_0=-inf, l_0=0, O_0=0`) into that head's
accumulator location, so chunk 1's real `softmax-update` can read genuine
prior state instead of needing to special-case the first chunk (the 8
per-head accumulator slots are physically reused across every Q-tile
instance, so this has to happen fresh every time).

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `metadata-init` |
| head-idx | 3 bits | Resolves `m`, `l`, `O` against the same three hardcoded bases `softmax-update` uses. |

- No value fields — `-inf`/`0`/`0` are genuine hardware constants (same every head, every Q-tile), not compile-time-varying operands, so they cost zero bits.
- Kept as its own opcode rather than a mode bit on `softmax-update`: both cost SFU the same extra bit (2 vs. 3 SFU instructions both need a 2-bit opcode), so it's not cheaper — but a separate opcode keeps the hot-path instruction (512 issues/Q-tile) uniform, never branching on which chunk it's processing. Same move as `load-stationary`'s permanently-engaged transpose bit (§2): the opcode is the proxy for the special case, not a runtime flag inside a shared instruction.

---

## 4. DMA slot (3 instructions)

**Scope: HBM↔scratchpad only, never accumulator-facing.** Every accum
touch in this ISA already has a direct compute-unit port (matmul writes
raw S; SFU reads raw S, reads/writes metadata+output, writes P) — the
same separation real Gemmini uses (`ex_read_from_acc`/`ex_write_to_spad`
vs. `mvin`/`mvout`). DMA handles exactly the 4 real HBM-crossing cases —
load Q, load K, load V, store O — nothing else ever leaves the chip,
since the fused regime means P and raw S never touch HBM at all.

Every field here is either a hardcoded tensor base (Q-base/K-base/V-base/
O-base, compile-time-known HBM locations) plus a real offset, or a small
scratchpad-side selector — the same base+idx×stride address-generation
mechanism used everywhere else in this ISA, just extended to the HBM
side.

### `load-Q`

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `load-Q` |
| HBM offset | 16 bits | `log2(32 batches × 8 groups × 256 Q-tiles)` — identifies which (batch, KV-group, Q-tile) is being fetched. |
| head-idx | 3 bits | Drives *both* the HBM-side address (`head_base + head_idx × seq_len×d_head`, computed by hardware) and the scratchpad-side destination slot. |

- Issued **8× per Q-tile transition** (once per head), not as one bulk transfer — Q's HBM layout (`batch, n_heads, seq_len, d_head`) separates the 8 heads sharing a KV-group by a large stride, not contiguous, so one simple contiguous transfer per head is the right granularity. Hardware does the stride arithmetic (the same address-generation mechanism already used for every scratchpad head-idx in this ISA), so this adds no new hardware capability.
- No buffer-select bit — Q is not double-buffered.

### `load-K/V`

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `load-K/V` |
| HBM offset | 11 bits | `log2(32 batches × 8 groups × 8 chunks)` — identifies which (batch, KV-group, chunk) is being fetched. |
| dest-select | 2 bits | 1 bit K-vs-V + 1 bit buffer-select (1-or-2) — same 2-bit scheme as `load-stationary`'s source selector, just now on the write side. |

- **The K-vs-V bit does double duty.** K and V are different tensors in HBM with two genuinely separate hardcoded bases (`K-base`, `V-base`) — there is no single combined "K/V base." The K-vs-V half of `dest-select` drives a small mux on the HBM side too (`selected_base = K-vs-V ? V-base : K-base`, then `HBM address = selected_base + offset`), in addition to picking the scratchpad destination region. The buffer-select half (1-or-2) is scratchpad-only — double-buffering doesn't affect the HBM source, since both buffer slots for K pull from the same K tensor. No extra field needed; the existing 2 bits just get consumed by two different pieces of address-generation hardware.
- Issued **1× per chunk transition** — no head-idx, since a K/V chunk is shared across the whole 8-head group by construction (the GQA-reuse mechanism).
- HBM read is a simple contiguous burst — for fixed (batch, kv-head), `tile_k` consecutive sequence positions × the full `d_head` range is contiguous in standard row-major layout. No gather problem, unlike Q/O.

### `store-O`

| Field | Width | Meaning |
|---|---|---|
| opcode | — | `store-O` |
| head-idx | 3 bits | Scratchpad-side source slot; drives the HBM-side destination stride term, symmetric with `load-Q`. |
| HBM offset | 16 bits | Same (batch, KV-group, Q-tile) addressing as `load-Q`. |

- Issued **8× per Q-tile completion** (once per head), same reasoning as `load-Q` — O's HBM layout has the identical non-contiguous-across-heads structure as Q.

---

## 5. Bundle format

**Layout: fixed-width-per-slot.** Every bundle reserves a fixed-position,
fixed-width field for all four slots, every cycle, whether or not a given
slot has real work that cycle (idle slots encode a NOP).

**Why fixed over compact/variable**: real per-Q-tile instruction counts —
matmul-issue=1,024 (streams only), weight-load=128, SFU=528, DMA=32 — mean
every slot besides matmul-issue is idle most cycles under any schedule.
Fixed-width is the right tradeoff anyway: (1) the cost is code-density/
storage, not throughput — an idle slot doesn't cost a cycle, just unused
bits in that cycle's bundle; (2) fixed-offset decode is real hardware
simplicity, consistent with a standing preference for minimal control
hardware over marginal efficiency; (3) direct precedent — Groq's own
dispatch is described as "144-wide VLIW instructions," a fixed format, in
a chip whose whole thesis is minimal instruction-control-unit area.
**Explicit alternate**: a compact/variable scheme (active-slot bitmask +
only-present-slots' fields) would recover most of the idle-slot waste, at
the cost of variable-length decode — the real tradeoff to revisit if code
density ever becomes the actual bottleneck (this ISA has no hardware loop
construct at all, so a fully-unrolled program is real and large).

**Opcode widths** (mechanical, `ceil(log2(instruction count))` per slot):
matmul-issue (2 types) → 1 bit; weight-load (1 type) → 0-bit opcode, just
its 1-bit valid/fire flag; SFU (3 types) → 2 bits; DMA (3 types) → 2 bits.

**Per-slot width** = opcode + the slot's worst-case instruction (fixed
width means every instruction in a slot must fit the slot's max):

| Slot | Opcode | Worst-case fields | Slot width |
|---|---|---|---|
| Matmul-issue | 1 bit | 3-bit head-idx (both instructions identical) | **4 bits** |
| Weight-load | — (1-bit valid) | 5-bit src | **6 bits** |
| SFU | 2 bits | 3-bit head-idx (all three instructions identical) | **5 bits** |
| DMA | 2 bits | `load-Q`/`store-O`'s 19 bits (16-bit offset + 3-bit head-idx) | **21 bits** |

**Total bundle width: 4 + 6 + 5 + 21 = 36 bits.** One 36-bit word per
cycle, fixed position per slot, no parsing required to decode.

## 6. Capacity check

**Scratchpad** (≤1,048,576 B budget):

| Region | Size |
|---|---|
| K1/K2/V1/V2 (double-buffered) | 524,288 B |
| P (×8 heads) | 32,768 B |
| Q (×8 heads) | 32,768 B |
| Output (×8 heads, int8) | 32,768 B |
| **Total** | **622,592 B** (slack ≈ 416 KB) |

**Accumulator** (≤262,144 B budget):

| Region | Size |
|---|---|
| Per-head metadata (`m`,`l`) + output accumulator `O`, combined | 133,120 B |
| Raw-S (×4 buffered, fp32) | 65,536 B |
| **Total** | **198,656 B** (slack ≈ 62 KB) |

Both fit comfortably. Underlying address-bus widths (RTL detail, not
instruction-encoding — these never appear as instruction bits, since the
whole point of hardcoding the bases is that they don't need to):
scratchpad `log2(1 MiB)`=20 bits, accumulator `log2(256 KiB)`=18 bits,
HBM ≥`log2(4.5 GiB)`≈33 bits.

---

## 7. Summary table — all 9 instructions

| Slot | Instruction | Fields | Slot width | Issued (per Q-tile) |
|---|---|---|---|---|
| Matmul-issue | `steady-state-stream-qk` | opcode(1) + 3-bit head-idx | 4 bits | per (chunk, slice, head) = 8×8×8 = **512** |
| Matmul-issue | `steady-state-stream-v` | opcode(1) + 3-bit head-idx | 4 bits | per (chunk, slice, head) = **512** |
| Weight-load | `load-stationary` | valid(1) + 5-bit src (2-bit buffer-select + 3-bit slice-idx) | 6 bits | per (chunk, slice, K-or-V) = 8×8×2 = **128** |
| SFU | `softmax-update` | opcode(2) + 3-bit head-idx | 5 bits | per (chunk, slice, head) = **512** |
| SFU | `softmax-finalize` | opcode(2) + 3-bit head-idx | 5 bits | per head, after last chunk = **8** |
| SFU | `metadata-init` | opcode(2) + 3-bit head-idx | 5 bits | 8× per Q-tile, before chunk loop = **8** |
| DMA | `load-Q` | opcode(2) + 16-bit HBM offset + 3-bit head-idx | 21 bits | 8× per Q-tile = **8** |
| DMA | `load-K/V` | opcode(2) + 11-bit HBM offset + 2-bit dest-select | 21 bits (padded) | 1× per chunk × 2 (K,V) = **16** |
| DMA | `store-O` | opcode(2) + 3-bit head-idx + 16-bit HBM offset | 21 bits | 8× per Q-tile = **8** |

**Total bundle: 36 bits** (4 + 6 + 5 + 21, fixed-width-per-slot, §5).
Per-slot totals: matmul-issue = 1,024, weight-load = 128, SFU = 528
(512 + 8 + 8), DMA = 32.

**Notable emergent property**: every head-touching instruction — both
matmul-issue streams, all three SFU instructions, all three DMA
instructions — ends up issued exactly once per head, never bulk, never
multi-head-per-instruction, despite each slot's fields being derived
independently, from different mechanisms (compute granularity for
matmul/SFU, HBM contiguity for DMA). `load-stationary` was never part of
this pattern — it's chunk/slice/K-or-V indexed, never head-indexed.

See [`schedule.md`](schedule.md) for the cycle-accurate hand-schedule
built against this ISA.
