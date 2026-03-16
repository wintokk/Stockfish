# Stockfish 18 — Complete Architecture Analysis

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Module-by-Module Algorithm Breakdown](#module-by-module-algorithm-breakdown)
3. [Validation of "NPS vs Depth" Claims](#validation-of-nps-vs-depth-claims)
4. [Can Stockfish 18 Be Tuned to Search "Wider/Slower"?](#can-stockfish-18-be-tuned-to-search-widerslower)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              UCI LAYER                                 │
│  uci.cpp/h  ←→  ucioption.cpp/h  ←→  engine.cpp/h                    │
│  (Protocol)     (Options mgmt)       (Orchestrator)                   │
└─────────────────────┬───────────────────────────────────────────────────┘
                      │
          ┌───────────▼───────────┐
          │    SEARCH ENGINE      │
          │    search.cpp/h       │
          │                       │
          │  ┌─────────────────┐  │
          │  │Iterative Deepen │  │
          │  │  ┌───────────┐  │  │
          │  │  │Alpha-Beta │  │  │
          │  │  │  Search    │  │  │
          │  │  │  ┌──────┐ │  │  │
          │  │  │  │QSearch│ │  │  │
          │  │  │  └──────┘ │  │  │
          │  │  └───────────┘  │  │
          │  └─────────────────┘  │
          └───┬────┬────┬────┬────┘
              │    │    │    │
    ┌─────────▼┐ ┌─▼──┐│  ┌─▼────────────┐
    │Time Mgmt │ │ TT ││  │  Move Picker  │
    │timeman.  │ │tt. ││  │  movepick.    │
    │cpp/h     │ │cpp ││  │  cpp/h        │
    └──────────┘ └────┘│  └──────┬────────┘
                       │         │
              ┌────────▼──┐  ┌───▼──────────┐
              │ Threading │  │Move Generator│
              │ thread.   │  │ movegen.     │
              │ cpp/h     │  │ cpp/h        │
              └───────────┘  └──────────────┘
                       │
          ┌────────────▼─────────────┐
          │      EVALUATION          │
          │    evaluate.cpp/h        │
          │  ┌─────────────────────┐ │
          │  │  NNUE Neural Net    │ │
          │  │  nnue/              │ │
          │  │  ├─ network.cpp/h   │ │
          │  │  ├─ architecture.h  │ │
          │  │  ├─ accumulator.h   │ │
          │  │  ├─ features/       │ │
          │  │  │  ├─half_ka_v2_hm │ │
          │  │  │  └─full_threats  │ │
          │  │  └─ layers/         │ │
          │  │     ├─ affine.h     │ │
          │  │     └─ sqr_clipped  │ │
          │  │        _relu.h      │ │
          │  └─────────────────────┘ │
          └──────────────────────────┘
                       │
          ┌────────────▼─────────────┐
          │    BOARD REPRESENTATION   │
          │  position.cpp/h          │
          │  bitboard.cpp/h          │
          │  types.h                 │
          └──────────────────────────┘

  Supporting: misc.cpp/h, memory.cpp/h, numa.h, score.cpp/h,
              tune.cpp/h, history.h, syzygy/, benchmark.cpp/h
```

---

## Module-by-Module Algorithm Breakdown

### 1. `types.h` — Core Type Definitions

Defines the fundamental types used everywhere:
- **Value** (`int`): Evaluation scores. `VALUE_MATE = 32000`, `VALUE_INFINITE = 32001`.
- **Move** (16-bit): Bits 0-5 = destination, 6-11 = origin, 12-13 = promotion piece, 14-15 = special flags (normal/promotion/en-passant/castling).
- **Depth** (`int`): Search depth in plies.
- **Piece values**: Pawn=208, Knight=781, Bishop=825, Rook=1276, Queen=2538 (internal centipawn units).

### 2. `bitboard.cpp/h` — Bitboard Infrastructure

Chess board represented as 64-bit integers (one bit per square).

**Key algorithms:**
- **Magic bitboards** for sliding piece attacks (bishops, rooks). Precomputed lookup tables indexed by `(occupied_squares * magic_number) >> shift`. ~800KB of attack tables.
- **PEXT** (Parallel Bit Extract) used on BMI2-capable CPUs for faster sliding piece lookups.
- Precomputed tables: `LineBB[64][64]` (lines through two squares), `BetweenBB[64][64]` (squares between), `PseudoAttacks[8][64]`.
- `popcount()`, `lsb()`, `msb()` use compiler intrinsics (`__builtin_ctzll`, `_tzcnt_u64`).

### 3. `position.cpp/h` — Board Representation

Maintains the full game state:

**Data structures:**
- `board[64]`: Piece-per-square mailbox.
- `byTypeBB[8]`: Bitboards per piece type (pawn, knight, bishop, rook, queen, king, all).
- `byColorBB[2]`: Bitboards per color.
- `StateInfo`: Incrementally updated — Zobrist hash key, castling rights, en passant square, rule50, checkers bitboard, pinners, blockers.

**Key algorithms:**
- `do_move()` / `undo_move()`: Applies/reverts moves with incremental Zobrist hash updates, O(1) per move.
- **Repetition detection**: Uses Cuckoo hashing tables (initialized from Zobrist keys) to detect positions reachable by reversible move sequences without generating all moves.
- `see_ge()`: Static Exchange Evaluation — iterative "swap" algorithm that determines if a capture sequence results in material gain ≥ threshold, used extensively for pruning.

### 4. `movegen.cpp/h` — Move Generation

Generates pseudo-legal moves by type:
- `CAPTURES`: All captures + queen promotions.
- `QUIETS`: All non-captures + underpromotions.
- `EVASIONS`: Check evasion moves only.
- `NON_EVASIONS`: All moves (captures + quiets).
- `LEGAL`: Filters pseudo-legal for legality.

**Algorithm:** Template-based generation per piece type. Pawn moves handled separately (pushes, double-pushes, captures, en passant, promotions). AVX512 SIMD optimization for bulk move creation on supported CPUs.

### 5. `movepick.cpp/h` — Move Ordering

**Staged move generation** — only generates moves of each type when previous stages are exhausted:

```
Stage Pipeline (Main Search):
MAIN_TT → CAPTURE_INIT → GOOD_CAPTURE → QUIET_INIT → GOOD_QUIET → BAD_CAPTURE → BAD_QUIET

Stage Pipeline (Evasions):
EVASION_TT → EVASION_INIT → EVASION

Stage Pipeline (Quiescence):
QSEARCH_TT → QCAPTURE_INIT → QCAPTURE
```

**Scoring heuristics:**
- **Captures**: `7 × PieceValue[captured] + captureHistory[piece][to][capturedType]`
- **Quiets**: `2 × mainHistory + 2 × pawnHistory + continuationHistory[0..5] + checkBonus + threatPenalty`
- Good captures filtered by SEE: `pos.see_ge(move, -value/18)`
- Quiet moves sorted by `partial_insertion_sort()` with depth-dependent threshold (`-3560 × depth`).

### 6. `history.h` — History Heuristics

Multiple layered history tables guide move ordering:

| Table | Indexed By | Purpose |
|-------|-----------|---------|
| ButterflyHistory | [color][move] | General quiet move success |
| LowPlyHistory | [ply][move] | Root-near move success |
| CaptureHistory | [piece][to][captured] | Capture ordering |
| ContinuationHistory | [prevPiece][prevTo][piece][to] | Move-pair correlations |
| PawnHistory | [pawnStructureHash][piece][to] | Pawn-structure-aware ordering |

**Update formula** (gravity toward zero): `val += bonus - val × |bonus| / D`

### 7. `tt.cpp/h` — Transposition Table

- **Entry size**: 10 bytes (key16 + depth8 + genBound8 + move16 + value16 + eval16).
- **Cluster size**: 3 entries + 2 bytes padding = 32 bytes (cache-line aligned).
- **Replacement policy**: Prefers entries from current generation; among same-generation entries, prefers deeper searches. Old entries replaced first.
- **Prefetching**: `_mm_prefetch()` called before probe for cache warm-up.

### 8. `evaluate.cpp/h` + `nnue/` — NNUE Evaluation

**Dual-network system:**
- **Big network** (1024→31→32→1): Full-strength eval for balanced positions.
- **Small network** (128→15→32→1): Fast eval when `|simple_eval| > 962` (large material imbalance).
- If small net returns `|nnue| < 277` (unclear position), falls back to big net.

**Architecture per network:**
- **Feature transformer**: Converts position to feature vector using HalfKAv2_hm features (king-piece relationships, 22528 input features) or FullThreats features.
- **Hidden layers**: Affine → Squared Clipped ReLU + Clipped ReLU → Affine → Clipped ReLU → Affine → Output.
- **8 sub-networks** (LayerStacks): Selected by `bucket = (piece_count - 1) / 4`.
- **Incremental accumulator updates**: Feature changes propagated through `do_move` without full recomputation. "Finny Tables" cache accumulators per king square.

**Final eval formula** (`evaluate.cpp:65-87`):
```
nnue = (125 × psqt + 131 × positional) / 128
material = 534 × num_pawns + non_pawn_material
v = (nnue × (77871 + material) + optimism × (7191 + material)) / 77871
v -= v × rule50_count / 199    // fifty-move dampening
```

### 9. `search.cpp/h` — The Search Engine (Core)

#### Iterative Deepening (lines 258-544)

```
for rootDepth = 1 to MAX_PLY:
    for each PV line (pvIdx = 0 to MultiPV):
        set aspiration window around previous score
        repeat:
            search<Root>(pos, alpha, beta, rootDepth)
            if fail-low:  widen alpha, delta += delta/3
            if fail-high: widen beta,  delta += delta/3
        until score within window
    time management: should we search deeper?
```

#### Alpha-Beta Search (lines 615-1486)

Template parameterized by NodeType ∈ {Root, PV, NonPV}.

**Step-by-step algorithm:**

| Step | Name | Line | Description |
|------|------|------|-------------|
| 1 | Init | 623 | If depth ≤ 0, drop to quiescence search |
| 2 | Draw check | 664 | Detect draws by repetition, 50-move rule, insufficient material |
| 3 | Mate distance | 679 | Prune if mate already found closer to root |
| 4 | TT probe | 693 | Look up position in transposition table |
| 5 | Tablebase | 753 | Probe Syzygy endgame tablebases |
| 6 | Static eval | 712 | Evaluate position with NNUE; compute "improving" flag |
| 7 | **Razoring** | 870 | If eval < alpha − 507 − 312×depth², go straight to qsearch |
| 8 | **Futility pruning** | 876 | If eval − margin ≥ beta at depth < 16, return early |
| 9 | **Null move pruning** | 892 | Skip own turn; if opponent can't exploit it, prune. R = 7 + depth/3 |
| 10 | **IIR** | 932 | Reduce depth by 1 if no TT move found at depth ≥ 6 |
| 11 | **ProbCut** | 935 | Try captures with reduced window; if holds, prune |
| 12 | Move loop | 983 | Iterate through moves from MovePicker |
| 13 | **Singular extension** | 1119 | Extend search for uniquely good TT moves (up to +3 plies) |
| 14 | **Late Move Pruning** | 1053 | Skip quiets when moveCount ≥ (3 + depth²) / (2 − improving) |
| 15 | **Futility (parent)** | 1100 | Prune quiets where staticEval + margin ≤ alpha |
| 16 | **SEE pruning** | 1075 | Prune captures/quiets with bad static exchange evaluation |
| 17 | **History pruning** | 1088 | Prune quiets with history < −3826 × depth |
| 18 | **LMR** | 1230 | Search later moves at reduced depth; re-search at full depth if promising |
| 19 | Full search | 1263 | Full-depth search for non-LMR moves |
| 20 | Update | 1330 | Update TT, histories, killers on cutoff |

#### Late Move Reductions (LMR) Detail

The reduction formula (`search.cpp:606-607, 1042, 1190-1228`):

```
Base:     reductions[i] = 2809/128 × ln(i)     // logarithmic table

r = reductions[depth] × reductions[moveCount]   // base reduction
r -= delta × 576 / rootDelta                     // aspiration window factor
r += !improving × r × 217 / 512                 // non-improving bonus
r += 1182                                        // constant offset

// Then adjusted by 15+ factors including:
r -= PvNode × 1013                               // less reduction in PV
r += cutNode × 3582                              // more reduction at cut nodes
r -= statScore × 454 / 4096                      // history-based adjustment
r += allNode × r × 276 / (256×depth + 254)       // expected ALL nodes

Effective depth: d = clamp(newDepth - r/1024, 1, newDepth + 2)
```

#### Quiescence Search (lines 1496-1733)

Searches only captures and checks (at depth 0) to reach a "quiet" position before evaluating. Uses stand-pat (return static eval if ≥ beta), futility pruning, and SEE pruning.

### 10. `timeman.cpp/h` — Time Management

**Two modes:**
1. **Basetime + increment**: `optScale` grows logarithmically with ply and time.
2. **Moves-in-time** (tournament): `optScale` divided by remaining moves.

**Dynamic adjustment at root** (`search.cpp:486-527`):
```
totalTime = optimum × fallingEval × reduction × bestMoveInstability × highBestMoveEffort

where:
  fallingEval     = penalty if position eval is dropping (0.57 to 1.70)
  reduction       = sigmoid based on best-move stability across depths
  bestMoveInstability = 1.02 + 2.14 × bestMoveChanges / numThreads
  highBestMoveEffort  = 0.76 if >93.3% of nodes go to best move, else 1.0
```

**Continue to next depth?** `increaseDepth = (elapsed ≤ totalTime × 0.50)` — line 527.

### 11. `thread.cpp/h` — Parallel Search

- **Lazy SMP**: All threads search the same position independently with slightly different search parameters (aspiration window delta varies by thread index).
- Threads communicate only through the shared transposition table.
- Main thread controls time management; helper threads only search.
- `get_best_thread()`: Selects the thread whose root move achieved the best score at the deepest completed depth.
- NUMA-aware: Threads bound to NUMA nodes, with per-NUMA shared histories and network replicas.

### 12. `uci.cpp/h` + `ucioption.cpp/h` — UCI Protocol

Standard UCI protocol implementation. Key options:

| Option | Default | Range | Effect |
|--------|---------|-------|--------|
| Threads | 1 | 1-1024 | Number of search threads |
| Hash | 16 | 1-... | Transposition table size (MB) |
| MultiPV | 1 | 1-500 | Number of principal variations |
| Skill Level | 20 | 0-20 | Playing strength (20=full) |
| Move Overhead | 10 | 0-5000 | Time buffer for communication (ms) |
| nodestime | 0 | 0-10000 | Nodes-per-millisecond mode |
| UCI_LimitStrength | false | — | Enable Elo-limited play |
| UCI_Elo | 1320 | 1320-3190 | Target Elo when limited |
| Ponder | false | — | Think on opponent's time (+25%) |
| EvalFile | nn-*.nnue | — | Big NNUE network file |
| EvalFileSmall | nn-*.nnue | — | Small NNUE network file |

### 13. `tune.cpp/h` — Tuning Infrastructure

SPSA-based parameter tuning framework. The `TUNE(...)` macro registers internal constants as UCI options with auto-computed ranges, enabling automated tuning via tools like `fishtest`. Parameters update through UCI `setoption` commands.

### 14. `syzygy/` — Endgame Tablebases

Probes Syzygy endgame tablebases for positions with ≤ 7 pieces. Returns WDL (win/draw/loss) and DTZ (distance to zeroing move) for perfect endgame play.

---

## Validation of "NPS vs Depth" Claims

The screenshots contain these claims from @chessifyai:

### Claim 1: "Stockfish uses selective search — it doesn't explore all lines equally. It focuses on promising continuations and prunes weaker ones."

**CONFIRMED by code.** This is the central design principle of Stockfish's search. The code contains at least **10 distinct pruning/reduction techniques** (see Step 7-18 in the search algorithm above). Key evidence:

- **Razoring** (`search.cpp:873`): Entire subtrees skipped when eval is far below alpha.
- **Null Move Pruning** (`search.cpp:892-925`): Skips moves entirely if position is so good that passing still wins.
- **Late Move Pruning** (`search.cpp:1054`): After checking `(3 + depth²)/(2-improving)` moves, remaining quiets are **never generated**.
- **LMR** (`search.cpp:1230-1261`): Later moves searched at reduced depth — e.g., at depth 16, move #30 might be searched at effective depth 4-6.
- **History pruning** (`search.cpp:1088-1090`): Moves with historically poor results pruned if `history < -3826 × depth`.

### Claim 2: "With lower NPS, the engine has to prune more aggressively, which can cause it to miss important ideas or tactical resources."

**PARTIALLY CONFIRMED, but the mechanism is INDIRECT.** The code does NOT adjust pruning aggressiveness based on NPS. All pruning thresholds are **hard-coded constants** (e.g., razoring margin of 507, futility multiplier of 77, LMP threshold of `(3+depth²)/(2-improving)`). NPS does not appear in any pruning formula.

However, the claim is effectively true through an **indirect mechanism**:
- Lower NPS → less time budget → fewer iterations of iterative deepening → shallower maximum depth reached.
- At any given depth, the same pruning rules apply regardless of NPS.
- But with more time (higher NPS), the search reaches deeper iterations where:
  - The TT is richer with more entries, improving move ordering and enabling better cutoffs.
  - More threads fill the TT with diverse positions (Lazy SMP effect).
  - Singular extensions fire more accurately with better TT data.

So it's not that lower NPS prunes "more aggressively" in terms of thresholds — it's that lower NPS means less total work done, fewer TT entries, and shallower depth, all of which compound.

### Claim 3: "With higher NPS, the engine explores significantly more nodes in the same amount of time, allowing it to evaluate more variations and often find critical tactical refutations earlier."

**CONFIRMED.** This is straightforwardly true. Time management (`timeman.cpp:132`) allocates a fixed time budget per move. Higher NPS = more nodes explored = deeper search = more variations evaluated. The time management doesn't adjust for NPS — it uses wall-clock time (or node-time if `nodestime` option is set).

### Claim 4: "Higher-speed Stockfish might even reach the same depth more slowly, but that's often because it's exploring wider — leading to a more accurate evaluation."

**NOT DIRECTLY CONFIRMED by code — this is a mischaracterization.** The search tree width at a given depth is determined entirely by the pruning parameters, which are constants. A faster machine doesn't search "wider" at the same depth — it searches **exactly the same way** at the same depth but **reaches deeper depths** in the same time.

What IS true: with a richer transposition table (from more threads or more time), the effective branching factor can change because:
- Better move ordering from TT hits leads to more cutoffs (narrower in some branches).
- But also more singular extensions fire, extending some lines deeper.
- The net effect can make certain depths take longer per iteration as the search finds more interesting lines to extend.

### Claim 5: "Depth alone doesn't tell the full story — evaluation quality depends on how thoroughly that depth was searched."

**CONFIRMED.** The concept of "selective depth" (`selDepth`) vs "nominal depth" (`rootDepth`) is built into the code:
- `rootDepth` is the iterative deepening depth (increments by 1 each iteration).
- `selDepth` (selective depth) is the maximum ply actually reached in any line (`search.cpp` tracks this).
- Due to LMR, some lines at depth 20 may only be searched 4-6 plies deep for later moves.
- Due to singular extensions, some critical lines may be searched 3+ plies deeper than nominal.
- More nodes searched (higher NPS × time) means more re-searches of LMR moves that failed high, more singular extensions firing, and better TT data — all of which make the search at a given depth more "thorough."

---

## Can Stockfish 18 Be Tuned to Search "Wider/Slower"?

The question: Can you configure Stockfish to explore more candidate moves (wider) at each depth before moving to the next depth, effectively trading search depth for search width?

### Short Answer

**Yes, partially**, through several mechanisms — but none of them correspond to the NPS-based effect discussed in the screenshots. There is **no UCI option to directly control pruning aggressiveness**.

### Available Tuning Mechanisms

#### 1. `MultiPV` (UCI Option) — **Most Direct**
```
setoption name MultiPV value 5
```
- **Effect**: Forces the engine to find the N best moves, not just the single best.
- **Code** (`search.cpp:301-309`): Outer loop `for (pvIdx = 0; pvIdx < multiPV; ++pvIdx)` searches each PV line fully at each depth.
- **Impact**: Dramatically wider search. With MultiPV=5, Stockfish must find 5 distinct good continuations. This catches tactical resources that single-PV might prune.
- **Downside**: Reaches much shallower depth in the same time. Each PV line requires a full root search.

#### 2. `go depth N` (UCI Command) — **Indirect Width Control**
```
go depth 20
```
- **Effect**: Forces search to exactly depth N regardless of time. At a fixed depth, higher NPS means more nodes searched → richer TT → effectively wider search at that depth.
- **How it helps**: At depth 20 with 1M NPS vs 30M NPS, the 30M NPS version fills the TT with 30× more positions. Moves that would have been pruned at low NPS might get re-searched because the TT provides better bounds.

#### 3. `go nodes N` (UCI Command) — **Fixed Computation Budget**
```
go nodes 100000000
```
- **Effect**: Search exactly N nodes regardless of depth reached.
- **Impact**: Higher NPS reaches deeper in fewer wall-clock seconds but searches exactly the same tree.

#### 4. `Threads` (UCI Option) — **Effective Width Increase**
```
setoption name Threads value 16
```
- **Effect**: Each thread searches the root position independently with slightly different parameters (delta offset by `threadIdx % 8` in aspiration window, `search.cpp:353`).
- **Impact**: Different threads explore different parts of the search tree. The shared TT means threads benefit from each other's discoveries. More threads = effectively wider search at each depth.
- **Evidence**: This is why Stockfish at 30M NPS (16 threads) vs 1M NPS (1 thread) finds different evaluations at the same depth — the multi-threaded version has a much richer TT.

#### 5. `Hash` (UCI Option) — **Indirect Width**
```
setoption name Hash value 4096
```
- **Effect**: Larger TT means fewer hash collisions → more positions cached → better move ordering → fewer wasted re-searches.
- **Impact**: Marginally wider effective search due to better TT hit rates.

#### 6. Source Code Modification — **Full Control** (requires recompilation)

If you truly want to search "wider," you would modify the pruning constants:

```cpp
// search.cpp:1054 — Late Move Pruning threshold
// ORIGINAL: moveCount >= (3 + depth * depth) / (2 - improving)
// WIDER:    moveCount >= (6 + depth * depth) / (2 - improving)  // 2x more moves tried

// search.cpp:873 — Razoring margin
// ORIGINAL: eval < alpha - 507 - 312 * depth * depth
// WIDER:    eval < alpha - 1014 - 624 * depth * depth  // 2x harder to trigger

// search.cpp:1089 — History pruning
// ORIGINAL: history < -3826 * depth
// WIDER:    history < -7652 * depth  // 2x more lenient
```

However, **these constants have been tuned over years via `fishtest`** (millions of games). Making search wider generally loses Elo because the depth loss outweighs the width gain. The current balance is near-optimal for playing strength.

#### 7. `nodestime` (UCI Option) — **Nodes-as-Time Mode**
```
setoption name nodestime value 1000
```
- **Effect**: Converts time management to use nodes instead of wall-clock time. `1000` means "1000 nodes per millisecond."
- **Impact**: Makes time management NPS-independent. If set lower than actual NPS, the engine gets more real time to search → effectively deeper search per "time unit."
- **Code** (`timeman.cpp:72-82`): `availableNodes = npmsec × limits.time[us]`, then all time management uses node counts.

### The Real Answer to the Screenshot's Question

The discussion in the screenshots compares Stockfish at 1 MN/s vs 30 MN/s reaching different evaluations at the same depth. The code confirms this happens because:

1. **Time management stops search** (`search.cpp:517`): `if (elapsed > min(totalTime, maximum))`. At 30 MN/s, the engine gets more iterations before time runs out.

2. **Lazy SMP enriches the TT**: More threads = more TT entries = better move ordering at every node = fewer critical moves pruned.

3. **LMR re-searches are NPS-sensitive**: When a reduced-depth search fails high (`search.cpp:1246`), a full-depth re-search happens. With more NPS, these re-searches complete within the time budget. With less NPS, the search might stop before completing re-searches at the current depth.

4. **Aspiration window failures take time** (`search.cpp:407-416`): Each fail-high/fail-low requires re-searching with a wider window. More NPS means more re-searches can complete.

**To directly make Stockfish "search wider at a given depth"**, the most practical approaches are:
- Use `MultiPV 3-5` to force exploration of multiple candidate moves
- Use `go depth N` with high NPS hardware to ensure thorough search at that depth
- Increase `Threads` and `Hash` for richer transposition table coverage
- There is **no single "width" knob** — the pruning is deeply integrated into the search with dozens of interacting parameters

### What You CANNOT Do

- **No "search aggressiveness" slider**: All pruning margins are compile-time constants, not UCI options.
- **No "width vs depth" tradeoff parameter**: The `TUNE()` macro system exists for developer tuning via fishtest, but these are not exposed as UCI options.
- **NPS itself is not controllable**: It's a property of your hardware. You can artificially limit it with `nodestime` to simulate slower hardware, but this makes the engine weaker, not "wider."
