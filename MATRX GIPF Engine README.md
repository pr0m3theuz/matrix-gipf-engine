# MATRX GIPF Engine

A high-performance game engine, AI using adversial search, and desktop interface written in Kotlin for **GIPF** featuring the complete **Project GIPF MATRX** expansion potentials.

---

## Overview

**MATRX GIPF** combines the core mechanics of Kris Burm's abstract strategy game *GIPF* with special "potential" pieces from the other games in the GIPF project:

- **GIPF**: Core pieces (capturing all 3 of the opponent's GIPF pieces—or leaving the opponent with no legal moves—wins the game).
- **TAMSK**: Grants an extra move when occupying the central board node as a potential.
- **ZERTZ**: Jumps over adjacent pieces along a line into an immediate empty spot.
- **YINSH**: Moves diagonally across empty intersections without pushing.
- **DVONN**: Stacks on top of an adjacent piece (supporting multi-layer piece stacks).
- **PUNCT**: Slides and jumps over a piece at distance to land directly behind it.

This repository provides a bitboard-accelerated engine (`:core`), a parallelized batch runner for AI experiments with deterministic replay (`:cli`), and an interactive GUI built with Compose Multiplatform (`:desktop`).

---

## Key Features

### Bitboard & Bit-Packed Architecture
- **64-Bit Bitboards (`ULong`)**: Separate bitboard masks per player and piece type, plus multi-layer `ULongArray` bitboards for `DVONN` and `PUNCT` stacks.
- **32-Bit Packed Moves & Pieces (`UInt`)**: Zero-allocation move encoding packing origin, target, direction, move type, piece type, color, and potential status into primitive unsigned integers.
- **Zobrist Hashing & Transposition Table**: Incremental/fast 64-bit state hashing with generation-aware transposition table probing (`EXACT`, `ALPHA`, `BETA` bounds).

### AI Search Agents
- **Minimax (Negamax with Alpha-Beta Pruning)**
  - Iterative deepening with configurable fixed depth or millisecond time controls
  - Quiescence Search (`qSearch`) to mitigate horizon effects during piece captures and removals
  - Move ordering via Transposition Table best-move, Killer Moves, Capture Moves, and History Heuristic tables
  - Dedicated `resolveBoardRemovals` search for 4-in-a-row line removal decisions
- **Monte Carlo Tree Search (MCTS)**
  - Configurable rollout iteration budgets or time-bounded search
  - **RAVE** (Rapid Action Value Estimation / All-Moves-As-First)
  - **First Play Urgency (FPU)** and **Progressive Widening (PW)**
- **Telemetry & Experience Collection**
  - Per-turn tracking of nodes searched, maximum depth, **Average Branching Factor (ABF)**, and **Effective Branching Factor (EBF)**
  - Game record export in JSONL and search statistics in CSV, plus HDF5 experience buffer serialization (`jhdf`) and Jupyter notebooks (`analyse_games.ipynb`, `analyse_search.ipynb`) for analysis.

---

## Project Structure

```text
matrx-gipf-engine/
├── core/                         # Core game engine, bitboards, AI agents, and test suite
│   └── src/
│       ├── main/kotlin/org/example/
│       │   ├── agents/           # Agent abstractions
│       │   ├── ai/               # Evaluation functions, MCTS, and Minimax + Quiescence search
│       │   ├── engine/           # Turn phases, game loop, Zobrist hashing, experience collector
│       │   └── model/            # Bitboard, move generation, state encoding, piece & player models
│       └── test/kotlin/          # Unit, property-based (jqwik), and fuzz (Jazzer) tests
├── cli/                          # Command-line runner and parallel batch experiment harness
│   └── src/main/kotlin/org/example/
│       ├── CLIOptions.kt                 # CLI flag definitions and parser
│       ├── Main.kt                       # Single-game execution entry point
│       └── generateReproducibleGames.kt  # Parallel batch runner & deterministic crash replay
├── desktop/                      # Compose Multiplatform desktop application
│   └── src/main/kotlin/org/example/ui/   # Hexagonal board UI, ViewModel, and theme
├── run_base_experiments.sh       # Base AI matchup experiment suite
├── run_experiments.sh            # Difficulty tier matrix experiment script
├── run-mcts-rave-experiments.sh  # MCTS Standard vs. RAVE ablation matrix script
└── run-minimax_mcts-time_control.sh # Minimax vs. MCTS depth/iteration matrix script
```

---

## Prerequisites

- **JDK 21** (configured via Gradle JVM Toolchain)
- **Gradle 8+** (included via `./gradlew` wrapper)

---

## Building the Project

Build all modules and generate standalone fat JARs:

```bash
./gradlew build
```

To build only the CLI standalone JAR:

```bash
./gradlew :cli:fatJar
```

---

## Usage

### 1. Desktop GUI

Launch the Compose Multiplatform desktop interface:

```bash
./gradlew :desktop:run
```

### 2. CLI Single-Game Runner

Run a single default match via the `:cli` module:

```bash
./gradlew :cli:run
```

### 3. Reproducible Batch Runner & Replay

The `GenerateReproducibleGamesKt` entry point runs parallel batches of games with deterministic per-game seeds derived from a master seed (`baseSeed`). If any game crashes or exceeds time/turn guards, its exception trace and serialized `State` JSON are dumped to `output/failures/` and logged to `output/failures.jsonl`.

#### Batch Mode Example

```bash
java -Xmx16g -cp cli/build/libs/cli-1.0-SNAPSHOT-standalone.jar org.example.GenerateReproducibleGamesKt \
  -b -g 500 -p 8 -s 12345 \
  -m mcts -mi 500 -mrv \
  -M minimax -mdp 5 \
  -css
```

#### Replay a Specific Game

```bash
java -cp cli/build/libs/cli-1.0-SNAPSHOT-standalone.jar org.example.GenerateReproducibleGamesKt \
  -r -G 42 -s 12345 -v true \
  -m mcts -mi 500 -mrv \
  -M minimax -mdp 5
```

#### CLI Options Reference

| Short | Long | Description |
| :--- | :--- | :--- |
| `-b` | `--batch` | Run a multi-threaded batch of games |
| `-r` | `--replay` | Replay a single game deterministically |
| `-g <COUNT>` | `--total-games <COUNT>` | Total number of games to run in batch mode (default: `50`) |
| `-G <ID>` | `--game-id <ID>` | Game ID to replay when using `-r` |
| `-p <THREADS>` | `--parallelism <THREADS>` | Number of worker threads (default: `cores - 2`) |
| `-s <SEED>` | `--seed <SEED>` | Base RNG seed for reproducibility |
| `-v <BOOL>` | `--verbose <BOOL>` | Print board and state summaries every turn (`true`/`false`) |
| `-css` | `--collect-search-statistics` | Record branching factor and node statistics to CSV |
| `-h` | `--help` | Display help message |

#### Agent Configuration Flags

Use lowercase `-m*` / `--agent-1-*` for **Agent 1 (White)** and uppercase `-M*` / `--agent-2-*` for **Agent 2 (Black)**:

| White (Agent 1) | Black (Agent 2) | Argument | Description |
| :--- | :--- | :--- | :--- |
| `-m`, `--agent-1-model` | `-M`, `--agent-2-model` | `mcts` \| `minimax` \| `nn` | AI search model |
| `-ms`, `--agent-1-strength` | `-MS`, `--agent-2-strength` | `random` \| `greedy` \| `easy` \| `medium` \| `hard` | Preset difficulty tier |
| `-mdp`, `--agent-1-minimax-depth` | `-Mdp`, `--agent-2-minimax-depth` | `<INT>` | Minimax search depth (default: `1`) |
| `-mi`, `--agent-1-mcts-iterations` | `-MI`, `--agent-2-mcts-iterations` | `<INT>` | MCTS rollout iterations per move |
| `-mtc`, `--agent-1-time-control` | `-Mtc`, `--agent-2-time-control` | *(flag)* | Enable time-bounded search |
| `-mtd`, `--agent-1-time-duration` | `-Mtd`, `--agent-2-time-duration` | `<MS>` | Time budget per move in milliseconds |
| `-mrv`, `--agent-1-mcts-rave` | `-MRV`, `--agent-2-mcts-rave` | *(flag)* | Enable RAVE in MCTS |
| `-mfpu`, `--agent-1-mcts-fpu` | `-Mfpu`, `--agent-2-mcts-fpu` | *(flag)* | Enable First Play Urgency in MCTS |
| `-mpw`, `--agent-1-mcts-pw` | `-Mpw`, `--agent-2-mcts-pw` | *(flag)* | Enable Progressive Widening in MCTS |

---

## Testing

The `:core` module includes unit tests, property-based tests (`jqwik`), and fuzz tests (`Jazzer`) covering bitboard operations, move generation equivalence, line scoring, piece conservation invariants, MCTS selection, and Minimax board removals.

```bash
./gradlew :core:test
```
