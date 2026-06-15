# ORDO — Technical Specification (v0.1, draft)

> **ORDO** — *Ordered Reduction / Depth-Obliged* proof-of-work. A PoW whose
> per-attempt work is a **forced-depth, integer-only, memory-latency-floored
> evaluation of a generated word-level circuit**, whose anti-parallel property is
> grounded in the **P-completeness of the Circuit Value Problem (CVP)**.
>
> **Status:** draft design spec, not consensus-ready. Companion to
> [`proposals.md`](./proposals.md) (see §3b, "Proposal 5 — ORDO"). This document
> expands that sketch into an implementable specification with an honest security
> analysis.

> © Hairetikos (Natoshi Sakamoto) @github 2026
---

## Motivation & goal — toward a single-thread-bound, "fair-luck" PoW

ORDO is designed to be **ASIC- and GPU-resistant** *and* **hard to parallelize
across threads/cores**, so that mining-lottery luck is distributed as evenly as
possible across different CPUs — rewarding the metric on which modern CPUs are
most *alike*, not the one on which specialized hardware runs away.

**The wager: single-thread performance has stagnated.** Clock speeds plateaued in
the mid-2000s with the end of Dennard scaling; since then we have gained *cores*,
SIMD width, and accelerators — not *GHz*. A high-end server core is not
dramatically faster, *per dependent instruction*, than a modest desktop core; they
are all, roughly, mediocre at one long serial chain of work. A PoW **bound by
single-thread speed** therefore levels the field: it cannot be won by piling on
cores, GPU lanes, or ASIC pipelines — only by the one quantity that has barely
moved in twenty years and is similar across machines.

Put aspirationally: **we want to see 10 GHz CPUs.** Until the industry delivers
single-thread speed again, a single-thread-bound PoW keeps the mining reward
pointed at the metric we wish vendors would optimize, instead of subsidizing
warehouses of parallel silicon.

**The honest tension (read §2 and §12, do not skip).** "Single-threaded by design"
and "fair lottery" pull against each other:

- A *truly* serial-by-design workload is **ORDO-VDF** (no grinding, §12): one
  forced-serial chain everyone must walk. It defeats multithreading by
  construction — but it has **no lottery**: the chain is deterministic, so the
  fastest single-thread stepper wins *every* block → centralization.
- A *lottery* needs **grinding** (independent nonce attempts, §9), which restores
  fair variance — but grinding is embarrassingly parallel across nonces, so
  multicore/GPU *does* help, and you only *approximate* single-threadedness by
  **saturating a shared resource** (memory bandwidth/capacity) so extra cores stop
  paying off (§10.2).

ORDO's stance: ship **grinding ORDO** for a real lottery, and lean on the borrowed
memory subsystem to *flatten* multicore scaling so that, in practice, throughput
tracks single-thread speed far more than core count. Whether that flattening is
actually achievable is an empirical question — it is exactly what the Stage-0 PoC
(§14) is built to measure *before* we commit to the design.

---

## Inspiration — workloads that are serial *by design*

The design draws on real systems whose core step resists parallelization. They
share one shape: a **left-fold with a non-associative combiner**,

> `sₙ = f(sₙ₋₁, xₙ)` — where the state `s` is large and `f` is non-associative and
> non-invertible.

The standard way to parallelize a fold is the **parallel prefix-scan**, which
*requires the combiner to be associative* (so subranges can be combined in any
order). Strip associativity away and no scan exists: the fold's critical path is
the **entire input**, and it is serial by construction. **ORDO's backbone (§5) is
deliberately such a fold** — each step's primary input is the previous step's
output, mixed by a non-associative word-gate. The examples below are the existence
proofs that this pattern is real, ubiquitous, and stubbornly single-threaded.

### clang/LLVM full (monolithic) LTO — the final link stage
With `-flto` (full LTO) each translation unit is compiled to **LLVM bitcode**, not
native objects. At link time the LTO plugin **merges all bitcode into one
monolithic module** and runs **whole-program interprocedural optimization**
(cross-module inlining, global DCE, IPSCCP, whole-program devirtualization) then
codegen on that single module. The IPO stage is **inherently serial**: the
optimizations are **order-dependent and mutate one shared IR graph** — inlining A
into B changes B, which changes whether B should be inlined into C, which changes
the call graph the next decision reads. The analyses need *global visibility of a
single mutable module*, so they run on one thread and scale with the whole program
(the classic "LTO link pegs one core for minutes and eats all the RAM").

The proof that *this* is the bottleneck: **ThinLTO** was invented to remove it. It
keeps modules separate, performs only a tiny **serial "thin-link"** over compact
per-module **summaries** + a function index, then does the heavy
optimization/import **per-module in parallel backends** — confining the serial
work to a small summary merge. Full LTO trades that parallelism away for maximal
whole-program visibility; that trade *is* the single-threaded final link.

### Bitcoin Core IBD — connecting the chain (researched in `validation.cpp`)
Initial Block Download has a cleanly split personality:
- **Parallel:** block *download* (many peers) and **script/signature verification**
  via `CCheckQueue` (`-par=N` worker threads). `-assumevalid` even *skips*
  signature checks for historical blocks, so the parallelizable work is largely
  elided during a modern sync.
- **Serial by design:** **connecting each block to the UTXO set.** The path is
  `ActivateBestChain` → `ActivateBestChainStep` → `ConnectTip` → `ConnectBlock`,
  one block at a time; `ConnectBlock` applies every transaction to the
  `CCoinsViewCache` via `UpdateCoins` (`SpendCoin` inputs, `AddCoin` outputs).
  Block *N+1*'s inputs must be validated against the **exact** UTXO set left by
  block *N* (to prove referenced outputs exist and are unspent), so the UTXO set is
  a single evolving state folded over the chain in order:
  `utxo_n = apply(utxo_{n-1}, block_n)`. You cannot connect *N+1* before *N*. Even
  with scripts skipped and blocks pre-fetched, IBD stays bottlenecked on this
  serial fold (+ its DB/cache work) — which is why it does not keep speeding up
  with more cores.

### More instances of the same pattern
- **VDFs / repeated squaring** (`y = x^(2^t) mod N`) — the canonical sequential
  function; the basis of ORDO-VDF (§12).
- **Hash chains / iterated hashing** (`H(H(H(…)))`) — each step needs the last.
- **CBC-mode encryption** — `cₙ = E(cₙ₋₁ ⊕ pₙ)`; encryption is serial (decryption
  and CTR mode are parallel — the contrast is instructive: the *chaining* is what
  forbids parallelism).
- **RNN recurrence** — `hₜ = f(hₜ₋₁, xₜ)`; the exact serialism Transformers
  abandoned to parallelize training. A modern, vivid example of the trade.
- **Single-stream DEFLATE/gzip decompression** — back-references read previously
  emitted output, so one stream decompresses serially.
- **Gauss–Seidel / SOR relaxation** — each update consumes the just-updated
  neighbor (see RELAX in [`proposals.md`](./proposals.md) §3b).
- **P-complete problems** — the formal home of "inherently sequential" (§3): not
  in NC unless NC = P. ORDO's theoretical backstop.

---

## 0. What is novel here, and what is borrowed (read this first)

Be honest about provenance — it is the only way to reason about security.

- **Novel (ORDO's actual contribution):**
  1. The per-attempt computation is a **forced-depth-`L` serial circuit** — its
     critical path is `L` *by construction* (the "backbone rule", §5). This floors
     **per-attempt latency** at `L × (DRAM round-trip)` regardless of how much
     parallel compute an attacker throws at a *single* attempt.
  2. That seriality is justified by a **complexity-theoretic** argument, not a
     statistical one: evaluating the circuit is an instance of **CVP, which is
     P-complete**, so no polylog-depth parallel evaluation exists unless **NC = P**
     (§3). RandomX's anti-parallel claim is statistical ("a CPU happens to be the
     best executor"); ORDO's is structural.
  3. The core is **integer-only and bit-exact** (§11) — no floating point — so
     cross-platform consensus determinism is *far* easier than RandomX (which must
     pin FP rounding modes). This is a concrete engineering win.
  4. A **low-power profile** falls out for free: a serial, latency-bound core
     spends most cycles *stalled on memory*, not burning watts.

- **Borrowed (deliberately, because reinventing it would be reckless):** the
  **two-tier memory subsystem** is RandomX-class — a large read-only **epoch
  dataset** (capacity gate + latency-floored reads) plus a small per-attempt
  **read-write scratchpad** sized to defeat GPU caches. These techniques are
  battle-tested; ORDO reuses the *rationale* rather than pretending to invent new
  memory-hardness.

**One-line summary:** *ORDO = a forced-serial, integer, CVP-grounded core layered
on RandomX-class memory-hardness.* The novelty is the **within-attempt
seriality + theoretical footing + determinism**; the memory tiering is proven art.

---

## 1. Threat model recap (the two enemies)

From `proposals.md` §2, restated because the security analysis (§10) leans on it:

- **Enemy A — intra-attempt parallelism** (speeding up *one* attempt with many
  lanes/pipeline stages; GPUs, deep ASIC pipelines). **ORDO's distinctive defense:
  the depth-`L` backbone + CVP P-completeness.**
- **Enemy B — inter-attempt parallelism** (running many *independent* attempts;
  many cores/cards/chips). **Cannot be abolished for a grinding PoW** (nonces are
  parallel by definition). ORDO *approximates* single-threadedness via the
  borrowed memory subsystem: capacity gating + a 2 MiB RW scratchpad per attempt
  that thrashes shared caches, flattening multicore/GPU scaling.

The honest combined claim ORDO can defend:

> **Throughput ≤ (attempts that fit in fast memory) ÷ (L × DRAM_latency).**
> Both the numerator (capacity/cache) and the denominator (serial latency floor)
> are pinned to *commodity DRAM physics*, leaving an ASIC/GPU little room.

---

## 2. Notation & primitives

| Symbol | Meaning |
|---|---|
| `m` | machine word width = **64 bits** |
| `H(·)` | cryptographic hash, **Keccak-256** (domain-separated) |
| `PRG(seed)` | XOF stream, **BLAKE3** in XOF mode (fast, parallel-fill for dataset) |
| `D[0..ND-1]` | epoch **dataset**, read-only, `ND` words of `m` bits |
| `S[0..NS-1]` | per-attempt **scratchpad**, read-write, `NS` words |
| `R[0..W-1]` | per-attempt **register file**, `W` live words (the backbone state) |
| `L` | steps per attempt (= circuit **depth** = critical-path length) |
| `PROG[0..L-1]` | the per-attempt **program** (static opcode/selector schedule) |
| `target` | 256-bit difficulty target; win iff `attempt_result < target` |

All multi-byte values are **little-endian**. All arithmetic in the core is on
unsigned 64-bit words with explicit wraparound (mod 2⁶⁴). `ND`, `NS`, `W` are
powers of two so addressing is masking (`x & (N-1)`), never division.

Domain separation: every hash/PRG call is prefixed with an ASCII tag
(`"ORDO/v0.1/lane"`, `"ORDO/v0.1/dataset"`, `"ORDO/v0.1/final"`, …) to prevent
cross-protocol and cross-stage collisions and to bind the version.

---

## 3. The computational model and the anti-parallel claim

### 3.1 Model

An ORDO attempt is the evaluation of a **bounded-width word-level circuit**:

- **Width** = `W` (the register file; default 8 → 512 bits of live state).
- **Depth** = `L` (the number of steps; the critical path).
- Each "gate" is a 64-bit word operation (`GATE[op]`, §6) — a *bundle* of ~64
  boolean gates. **We operate at word granularity on purpose:** literal 1-bit
  gates are trivial for an ASIC (one gate delay), so per-gate work must be a full
  word op whose latency is dominated by a **DRAM access**, which silicon cannot
  beat (§10.3).

The unrolled execution trace of `L` steps *is* a boolean circuit of depth
`Θ(L)`. Evaluating it = solving an instance of the **Circuit Value Problem**.

### 3.2 The claim (precise, with its limits)

> **Claim.** CVP is **P-complete** (Ladner, 1975). Therefore, under the standard
> conjecture **NC ≠ P**, there is no algorithm that evaluates a worst-case ORDO
> circuit in **polylogarithmic parallel depth** with polynomially many
> processors. Intra-attempt parallelism (Enemy A) cannot give *asymptotic* depth
> reduction.

**What this does NOT say (paranoia, stated up front):**

1. P-completeness is **worst-case**. It does not, by itself, prove a *random*
   ORDO instance is hard. We close this gap **operationally** with the **backbone
   rule** (§5): we *construct* every instance to have a guaranteed critical path
   of length `L`, so seriality does not depend on average-case conjectures.
2. It forbids **NC-style** (polylog-depth) speedups, **not** constant-factor or
   modest *polynomial* speedups with many processors. ORDO caps the *exponent* of
   any parallel attack, not every gain.
3. The DRAM-latency floor (§10.3), not the theorem, is what makes the *constant*
   in "`L × latency`" hard to shrink. The theorem and the floor are complementary:
   the theorem stops you from collapsing depth; the floor stops you from making
   each unit of depth cheaper.

This layered, hedged argument is the whole point of ORDO — it replaces "trust us,
a CPU is best" with "here is a theorem, here is a construction, and here is a
physics floor."

---

## 4. Memory subsystem (borrowed, RandomX-class)

Two tiers, with distinct jobs:

### 4.1 Epoch dataset `D` (large, read-only)

- **Size:** `ND = 2²⁸` words = **2 GiB** (default; tunable up — the capacity gate).
- **Lifetime:** depends on an **epoch key** `K_epoch` (the block hash from
  `E` blocks ago, default `E = 2048` ≈ a few days). Regenerated once per epoch.
- **Generation:** `D` is filled by a **memory-hard expansion** of `K_epoch`
  (Argon2d-style data-dependent fill, or RandomX's superscalar generator). Two
  modes:
  - **Full mode:** store all 2 GiB → fast reads. (Miners.)
  - **Light mode:** store a 32–64 MiB cache, recompute cells on demand at a large
    slowdown. (Verifiers / low-RAM nodes.) This is the TMTO knob: light mode is
    *allowed* but ~10–30× slower, so rational miners use full mode.
- **Role:** capacity gate (ASIC needs real DRAM; few datasets fit on a GPU) +
  source of **data-dependent, cache-missing reads** (the latency floor).

### 4.2 Per-attempt scratchpad `S` (small, read-write)

- **Size:** `NS = 2¹⁸` words = **2 MiB** (default) — sized to live in **L2/L3**.
- **Init:** filled from `PRG("ORDO/v0.1/scratch" ‖ lane_seed)` at attempt start.
- **Role:** the **GPU killer**. Each attempt needs its *own* 2 MiB of low-latency,
  frequently read-written memory. Thousands of concurrent attempts (what a GPU
  wants) need thousands × 2 MiB of *L2-class* memory, which GPUs do **not** have
  (their L2 is a few MiB shared). It is also **read-modify-written** every step
  (anti-TMTO within an attempt).

> **Why both tiers are required (paranoid note).** A big read-only dataset *alone*
> is **not** GPU-resistant: all attempts in an epoch share `D`, so a GPU loads it
> once and hides its latency behind thousands of threads. The **per-attempt 2 MiB
> RW scratchpad** is what actually defeats the GPU, exactly as in RandomX. ORDO
> keeps both and *adds* the serial backbone on top.

---

## 5. The backbone rule (ORDO's depth guarantee)

To make seriality unconditional (not reliant on average-case hardness), the
program is generated so that **step `i` consumes a value produced by step `i−1`**:

> **Backbone rule.** For every `i ≥ 1`, the *primary source register* of step `i`
> is the *destination register* of step `i−1`: `PROG[i].ra := PROG[i−1].rd`.
> The other selectors (`rb`, `rd`, `op`) are pseudo-random from `PRG(lane_seed)`.

This forces a dependency chain `step₀ → step₁ → … → step_{L-1}` of length `L`.
The longest path of the unrolled circuit is therefore **≥ L**: no scheduler can
evaluate the attempt in fewer than `L` dependent steps. The random `rb`/`rd`/`op`
and the data-dependent memory reads supply *irregularity, memory-hardness, and
control-flow divergence* on top of the guaranteed depth.

*(Generalization, deferred to §15: dropping the strict backbone in favor of a
width-`W` circuit with only a **probabilistic** long path is where CVP
P-completeness would carry the full load. v0.1 keeps the backbone for a
deterministic guarantee.)*

---

## 6. Word-gates `GATE[op]`

`op ∈ {0..15}`, drawn from the program stream. A representative set (final set is
a tuning decision; all are **integer, deterministic**):

| op | name | definition (all mod 2⁶⁴) | purpose |
|----|------|--------------------------|---------|
| 0 | ADD | `rd + ra + d` | diffusion |
| 1 | SUB | `rd - ra - s` | diffusion |
| 2 | XOR | `rd ^ ra ^ d` | linear mix |
| 3 | MUL | `rd * (ra | 1)` | nonlinearity |
| 4 | MULHI | `hi64(rd * d)` | stresses wide multiplier |
| 5 | ROTL | `rotl(rd, ra & 63) ^ d` | bit mixing |
| 6 | AND | `rd ^ (ra & d)` | nonlinear |
| 7 | ANDN | `rd ^ (ra & ~d)` | nonlinear |
| 8 | IMUL_ADD | `rd*0x9E3779B97F4A7C15 + ra` | avalanche |
| 9 | **MUX** | `(d & 1) ? ra : (rd + s)` | **data-dependent control flow** |
| 10 | DIV | `rd / (ra | 1)` (unsigned) | stresses ASIC/GPU divider |
| 11 | REM | `rd % (d | 1)` | divider stress |
| 12 | BSWAP_ADD | `bswap(rd) + ra` | byte-perm |
| 13 | POPMIX | `rd ^ (popcount(d) * ra)` | irregular |
| 14 | AESR | `aes_enc_round(rd, d)` | substitution (AES-NI on CPU) |
| 15 | CLMUL | `clmul_lo(rd, ra) ^ d` | carryless mul |

Notes:
- **`MUX` (op 9)** and the **data-dependent register swap** (§7) are the
  control-flow-divergence sources: on a GPU, neighboring lanes take different
  branches → SIMT serialization.
- **`DIV`/`REM`** deliberately use the integer divider — slow and area-expensive
  to replicate at scale, weakening the ASIC throughput argument.
- **No floating point anywhere** → bit-exact across CPUs (§11).

---

## 7. The STEP function (normative pseudocode)

Per-attempt state: `R[0..W-1]` (init from `lane_seed`), `S` (init, §4.2), and the
read-only `D`. `SELECT_MASK` is a fixed constant (e.g. `0x3F` → ~1/64 swap rate).

```text
STEP(i):
    op = PROG[i].op;  ra = PROG[i].ra;  rb = PROG[i].rb;  rd = PROG[i].rd
    # (PROG[i].ra == PROG[i-1].rd by the backbone rule, §5)

    # 1) data-dependent dataset read  -> latency floor + capacity gate
    da = R[ra] & (ND - 1)
    d  = D[da]

    # 2) data-dependent scratchpad read-modify-write -> anti-TMTO, GPU-cache killer
    sa = (R[rb] >> 3) & (NS - 1)
    s  = S[sa]
    S[sa] = s + R[rd]                    # mutate (wraparound)

    # 3) the word-gate (backbone: new R[rd] depends on old R[rd], R[ra], d, s)
    R[rd] = GATE[op](R[rd], R[ra], d, s)

    # 4) data-dependent control-flow divergence (branchy; hostile to SIMT)
    if (R[rd] & SELECT_MASK) == 0:
        swap(R[ra], R[rb])
```

Per step: **2 dependent memory touches** (one big-`D` read, one `S` RMW) + one
word-gate + a data-dependent branch. The big-`D` read is the dominant cost (a
cache-missing dependent load); its address `R[ra]` is only known after the
previous step → **no prefetch, no reordering** → the latency floor bites.

---

## 8. An ORDO attempt

```text
ATTEMPT(header_prefix, nonce) -> result:
    lane_seed = H("ORDO/v0.1/lane" ‖ header_prefix ‖ nonce)
    R[0..W-1] = PRG("ORDO/v0.1/regs"   ‖ lane_seed)         # 8 words
    PROG      = gen_program(lane_seed)                      # L entries, backbone rule
    S         = PRG("ORDO/v0.1/scratch" ‖ lane_seed)        # 2 MiB
    for i in 0..L-1: STEP(i)
    result    = H("ORDO/v0.1/final" ‖ R[0..W-1] ‖ fold(S))  # 256-bit
    return result
```

- `gen_program` streams `PRG(lane_seed)` to fill `op`, `rb`, `rd` for each step,
  then applies the **backbone rule** to set each `ra`.
- `fold(S)` binds the final scratchpad: hash 64 PRG-selected 4 KiB blocks of `S`
  (cheap, ~256 KiB hashed) so the result depends on `S`'s mutated state without
  hashing all 2 MiB. (Full-`S` hash is an option if profiling allows.)
- **Win** iff `result < target`.

The attempt is a pure deterministic function of `(header_prefix, nonce, K_epoch)`.

---

## 9. Block-level protocol

- **Header** (consensus fields): `prev_hash`, `merkle_root`, `timestamp`, `bits`
  (compact target), `epoch_ref` (identifies `K_epoch`), and the search `nonce`.
- **`header_prefix`** = the header serialized with `nonce` zeroed (so the nonce is
  the only search freedom). `lane_seed` binds to it (§8).
- **Mining (grinding lottery):** iterate `nonce`; for each, run `ATTEMPT`; submit
  when `result < target`. Each attempt is one independent Bernoulli trial, so the
  classic lottery holds: `P(win) ∝ attempts/sec` with the usual geometric
  variance (solo miners still occasionally win).
- **Difficulty / retarget:** `target = 2²⁵⁶ / difficulty`; standard
  moving-window retarget to hold a target block interval. `L` is *fixed by
  consensus* (it sets per-attempt latency); difficulty floats via `target`.
- **Epoch:** `K_epoch` = block hash from `E` blocks ago; dataset `D` rotates with
  it. A short warm-up overlap (mine on old + new dataset around the boundary)
  avoids a stall while `D` regenerates.

> ORDO v0.1 is a **grinding** PoW (nonce search), *not* a pure VDF — this is a
> deliberate choice to keep a real lottery. The no-grinding ORDO-VDF variant and
> why it sacrifices the lottery are in §12.

---

## 10. Security analysis

### 10.1 Within an attempt (Enemy A) — ORDO's distinctive defense
- **Backbone (§5):** critical path = `L`; a single attempt cannot finish in fewer
  than `L` dependent steps. Throwing parallel ALUs at *one* attempt buys nothing.
- **CVP P-completeness (§3):** assures no clever NC-style parallel evaluation
  exists for the general circuit, so the backbone guarantee is not an artifact of
  a naive construction.
- **Consequence:** **per-attempt latency is floored** at `L × t_step`, where
  `t_step ≥` one DRAM round-trip. This is the property RandomX does *not*
  explicitly provide.

### 10.2 Across attempts (Enemy B) — inherited, approximate
- **Capacity gate:** 2 GiB dataset → a GPU fits few datasets; but since `D` is
  shared per epoch, the real limiter is...
- **2 MiB RW scratchpad per attempt:** thousands of concurrent attempts need
  thousands of L2-class scratchpads; GPUs lack that low-latency capacity, so
  concurrency collapses (the RandomX mechanism). 
- **Result:** multicore/GPU throughput scaling is **sub-linear and flattens** once
  L2/L3 and memory bandwidth saturate → *approximate* single-threadedness, with
  the lottery intact. **Not** a proof of single-threadedness (impossible for
  grinding, see `proposals.md` §2) — an economic flattening.

### 10.3 ASIC / FPGA
- **DRAM latency floor:** the dominant per-step cost is a dependent cache-missing
  load. No ASIC beats commodity DRAM *latency* (the "memory wall": DRAM latency
  has been ~50–90 ns for two decades; HBM improves *bandwidth*, not latency). An
  ASIC must use the same DRAM and so inherits the same `t_step`.
- **General-ALU requirement:** the per-attempt program differs every nonce
  (`lane_seed`-derived), so no fixed circuit can be hardwired; the chip needs a
  general word-ALU + JIT/interpreter (the RandomX "optimal ASIC ≈ CPU" argument),
  *plus* an integer divider and AES/CLMUL units (ops 10/11/14/15) that cost area.
- **Modeled speedup target:** **< ~3×** over a mainstream CPU (vs. 10⁶× for
  SHA-256). Honest: not zero, but economically uninteresting.

### 10.4 GPU latency-hiding — the trap, and why ORDO survives it
A GPU normally hides memory latency behind thousands of threads. That defeats a
*shared-read-only, latency-bound* workload. ORDO blocks it with the **per-attempt
2 MiB RW scratchpad**: to run `N` concurrent attempts the GPU needs `N × 2 MiB` of
*fast* memory; beyond a few dozen it spills to slow VRAM and the access pattern
(random RMW) wrecks coalescing. So the GPU cannot deploy enough concurrency to
hide the `D`-read latency. (Same reason RandomX is GPU-hostile.)

### 10.5 TMTO
- **Dataset:** light mode (recompute from 32–64 MiB cache) is *allowed* but
  ~10–30× slower → rational actors store the full 2 GiB. No free time-memory win.
- **Scratchpad:** RMW + data-dependent addressing (Argon2d property) defeats
  recompute-on-demand within an attempt.

### 10.6 Precompute / grinding shortcuts
- `lane_seed = H(header_prefix ‖ nonce)` is unpredictable → no per-attempt
  precompute. Epoch rotation (`E` blocks) caps any "precompute the dataset
  forever" advantage to one epoch.
- The backbone + cryptographic `fold(S)` and final hash leave no algebraic
  shortcut to the result short of running the chain.

### 10.7 Verification DoS
- Verify = **one re-executed attempt** (≈ `L × t_step` ≈ 5–15 ms at defaults) on a
  node holding `D` (full or light mode). Cheap relative to mining millions of
  attempts, but **not free** — at very large `L` this is a knob to watch. The
  optional **checkpoint/Merkle spot-check** from HELIX (`proposals.md` §3) can make
  verification sub-linear if `L` is pushed high; v0.1 uses plain re-execution.

### 10.8 Botnets (the CPU-PoW curse — be paranoid)
CPU-friendliness invites malware mining. ORDO's deterrent is the **2 GiB
dataset**: shared hosting, containers, and low-RAM zombies can't hold it (light
mode is too slow to be worth it), and a 2 GiB resident footprint is conspicuous to
endpoint security. Not a cure — a tax.

### 10.9 Economic-security caveat (low power = different model)
A latency-bound, low-watt PoW shifts security from *energy burned* toward *capital
and opportunity cost* (RAM-hours and serial time you can't reuse). Attack cost is
"rent enough RAM and CPUs," not "buy megawatts." Defensible and arguably
healthier, but it is a **different threat model** and must be stated to the
community, not buried.

---

## 11. Determinism & cross-platform reproducibility (a hard requirement)

Consensus forks on *any* nondeterminism. ORDO is designed for determinism:

- **Integer-only core.** No floating point anywhere in `STEP`/`GATE` → no rounding
  modes, no FMA-contraction differences, no x87/SSE/NEON divergence. *This is a
  deliberate advantage over RandomX.*
- **Defined word semantics:** all ops mod 2⁶⁴; `DIV/REM` guarded by `| 1` (never
  divide by zero); `MULHI` = high 64 of the 128-bit product; `ROTL` by `ra & 63`;
  `AESR` = one AES encryption round with `d` as round key (AES-NI and software
  must agree — they do, AES is exactly specified); `CLMUL` = carryless multiply
  low 64.
- **Fixed endianness** (little-endian) for all serialization and PRG output.
- **Pinned primitives:** Keccak-256 and BLAKE3 with fixed parameters and the
  domain tags of §2.
- **Reference vectors** (§14) are consensus-binding: any implementation that
  disagrees on a vector is wrong by definition.

---

## 12. Variant: ORDO-VDF (no grinding) — and why the lottery breaks

For maximal Enemy-B resistance one can *abolish grinding*: fix `lane_seed =
H(header_prefix)` (no nonce), run **one** very long ORDO chain, and define the
block as valid at the **first step `i`** where `H(R) < target`; the proof is
`(i, checkpoint_openings)` verified by the HELIX checkpoint/spot-check scheme
(`proposals.md` §3, §10.7).

- **Gain:** there is exactly **one** chain for everyone → no inter-attempt
  parallelism at all → Enemy B *defeated by construction*; verification becomes
  sub-linear via checkpoints.
- **Loss — the showstopper:** the chain is deterministic, so the *first crossing*
  is at a fixed `i` for everyone; the winner is simply **whoever steps fastest** →
  a deterministic speed race → **centralization**, no lottery.
- **Partial fix:** `K` header-seeded lanes (`proposals.md` §3, "HELIX") reintroduce
  variance but cap per-machine parallelism at `K`; `K` is the
  fairness↔anti-parallel dial. A standalone CPU-PoW cannot get a *clean* lottery
  from a pure VDF (cf. Chia, which buys its lottery from Proof-of-Space and uses
  the VDF only for delay).

**Recommendation:** ship **grinding ORDO (§9)** for a real lottery; keep ORDO-VDF
as a research track for chains willing to pair it with an external lottery.

---

## 13. Parameters

| Param | Symbol | Default | Range / notes |
|---|---|---|---|
| Word width | `m` | 64 bit | fixed |
| Register file | `W` | 8 words | 4–16; bounded width |
| Dataset size | `ND` | 2 GiB | 1–8 GiB; **capacity gate** (raise to harden vs GPU/ASIC, watch RAM) |
| Scratchpad | `NS` | 2 MiB | 1–4 MiB; **must exceed GPU per-thread fast memory**, fit CPU L2/L3 |
| Steps/attempt | `L` | 2¹⁷ ≈ 131072 | sets per-attempt latency (~5–15 ms) & verify cost; **consensus-fixed** |
| Epoch length | `E` | 2048 blocks | dataset rotation cadence |
| Light-mode cache | — | 32–64 MiB | TMTO knob; ~10–30× slower |
| Swap rate mask | `SELECT_MASK` | `0x3F` | control-flow divergence frequency |

Tuning targets (the numbers a PoC must hit, §14 of `proposals.md`): GPU **slower**
than a mainstream CPU per unit work; multicore scaling **sub-linear & flattening**
by a handful of cores; modeled ASIC speedup **< ~3×**; verify **< ~15 ms**.

---

## 14. Reference implementation & PoC plan

Reuse this repo's proven methodology: a **CPU reference model as the oracle**,
**differential testing** between reference and optimized executor, and
**re-verify every solution before submit** (the amdminer invariants transfer).

- **Stage 0 — falsify the floor (1 weekend).** Microbench a dependent
  `D`-read + `S`-RMW STEP loop. Measure: (a) ns/step on CPU (is it DRAM-latency
  floored?), (b) multicore scaling curve (does it flatten?), (c) a quick GPU port
  (is it sub-CPU?). **If the floor/flattening don't materialize, stop and
  redesign.** Measure before building (this repo's first rule).
- **Stage 1 — reference core.** Integer `STEP`/`GATE`, `gen_program` + backbone,
  scratchpad init/fold, `ATTEMPT`, target check. Pure portable C, bit-exact.
- **Stage 2 — determinism gate.** Cross-platform test vectors (x86-64 + aarch64)
  for `ATTEMPT(seed,nonce) → result`. **Make-or-break** for consensus.
- **Stage 3 — dataset + epochs.** Memory-hard `D` generation (full + light mode),
  epoch rotation, verifier.
- **Stage 4 — optimized executor.** JIT the per-attempt program; differential-test
  against the reference (== amdminer `--sim-test` discipline). Benchmark vs CPU
  reference, attempt a GPU port to confirm hostility.
- **Stage 5 — adversarial review.** TMTO, FPGA/ASIC cost model for the STEP,
  GPU-concurrency feasibility, verification-DoS at large `L`, botnet, lottery
  variance. Independent review before any mainnet talk.

Suggested PoC layout (under this directory):

```
research/anti-parallel-pow/
  proposals.md          # the survey (this spec's parent)
  ordo-spec.md          # this document
  poc/                  # Stage 0–4 code
    bench_step.c        # Stage 0 microbenchmark (CPU)
    bench_step.cl       # Stage 0 GPU port
    ordo_ref.c          # reference core (oracle)
    ordo_vectors.json   # consensus test vectors
    README.md
```

---

## 15. Open problems

1. **Backbone-free depth.** Can we drop the strict backbone and rely on CVP
   P-completeness for a width-`W` circuit with only a *probabilistic* long path —
   gaining more irregularity (GPU-hostility) without losing the depth guarantee?
   Needs a generator with a *provable* (or tightly measured) critical-path
   distribution.
2. **Average-case hardness.** Bridge worst-case P-completeness to the hardness of
   *random* ORDO instances (the perennial gap in complexity-based crypto).
3. **Sub-linear verification without HELIX checkpoints** — an IVC/folding proof
   that is *not* itself GPU-acceleratable (else it re-centralizes proving).
4. **Enemy B is only approximated.** Quantify the multicore flattening precisely
   across microarchitectures; find the smallest `NS` that still defeats GPUs.
5. **`L` vs. verification-DoS vs. lottery granularity** — the three-way tension in
   choosing `L`.

---

## 16. Changelog

- **v0.1 (draft):** initial spec — grinding ORDO with backbone-forced depth,
  integer-only core, RandomX-class two-tier memory, honest security analysis,
  ORDO-VDF variant, PoC plan.
