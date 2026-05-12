# ClickHouse — Query Execution

![DA-IICT](https://img.shields.io/badge/DA--IICT-Big%20Data%20Engineering-blue?style=flat)
![Semester](https://img.shields.io/badge/Semester-2-blue?style=flat)
![ClickHouse](https://img.shields.io/badge/Topic-ClickHouse-informational?style=flat)
![Built From Source](https://img.shields.io/badge/Built%20From-Source-blue?style=flat)
![Status](https://img.shields.io/badge/Status-Complete-success?style=flat)

> Not a tutorial. Not documentation. A reverse-engineering journal of how one of the world's fastest database engines executes queries — from source code to broken experiments.

---

While MergeTree handles the storage of data on disk, **ClickHouse's Query Execution Pipeline** is what actually processes that data at breakneck speeds. ClickHouse uses a **vectorized query execution engine**, meaning it processes data in chunks (blocks of columns) rather than row-by-row, keeping the CPU cache hot and maximizing SIMD instructions.

The execution engine is built around a few core principles:

| Principle | What It Means |
|-----------|---------------|
| **Pipeline Processors** | Queries are translated into a Directed Acyclic Graph (DAG) of Processors (e.g., Read, Filter, Expression, Aggregation) that pass data chunks to each other. |
| **Massive Parallelism** | Operations like reading from disk and filtering are heavily parallelized out-of-the-box using multiple execution streams. |
| **Early Filtering & Optimizations** | Expressions are aggressively optimized (e.g., constant folding) during query planning, and filters are pushed down as early as possible to minimize data movement. |

---

### Phase 1 — Read the Source Code
We analyzed the `src/Processors/` and `src/Interpreters/` directories of the ClickHouse source and traced the critical code paths of query execution:

- **The Execution Path:** SQL Parser → AST → `ExpressionAnalyzer` (Constant Folding & Optimization) → Query Plan → DAG of Processors → Execution via `ReadFromMergeTree` → `FilterTransform` → `ExpressionTransform` → Result.

### Phase 2 — Break Things on Purpose
We modified the ClickHouse C++ source directly to disable or alter core execution pipeline behaviors, then measured the impact on query performance using `system.query_log`.

| Experiment | Source File Modified | What We Changed |
|---|---|---|
| **Exp 1: Disable Parallelization** | `Processors/QueryPlan/ReadFromMergeTree.cpp` | Hardcoded `num_streams = 1` to force single-threaded data reading. |
| **Exp 2: Disable Filter Pushdown** | `Processors/Transforms/FilterTransform.cpp` | Forced early return in `doTransform()` to skip early row filtration. |
| **Exp 3: Disable Expression Optimizer** | `Interpreters/ExpressionAnalyzer.cpp` | Changed `allowEarlyConstantFolding()` to always `return false;`. |
| **Exp 4: Extra Expression Evaluation** | `Processors/Transforms/ExpressionTransform.cpp` | Duplicated `expression->execute(...)` to force the engine to evaluate expressions twice. |

### Phase 3 — The Autopsy
Every performance degradation was traced back to the exact C++ function in the execution pipeline. The experiments prove why ClickHouse's pipeline optimizations are absolutely vital for its speed.

---

## Repository Structure
```text
ClickHouse-Query-Execution/
│
├── 📁 raw/                              # ClickHouse source code
│   └── ClickHouse/
│       ├── src/Processors/              # The C++ source files for Pipeline Processors
│       └── src/Interpreters/            # The C++ source files for Query Analysis
│
├── 📁 code-notes/                       # Our analysis and documentation
│   └── execution_pipeline.md            # Query Plan to Processor DAG explanation
│
├── 📁 experiments/
│   ├── QEE1.png                         # Code modification for Parallelization
│   ├── QEE2.png                         # Code modification for Filter Pushdown
│   ├── QEE3.png                         # Code modification for Expression Optimizer
│   └── QEE4.png                         # Code modification for Extra Evaluation
│
├── 📁 Screenshots/                      # Chart images for each experiment result
│   ├── Screenshot 2026-05-12 013922.png # Exp 1 Results
│   ├── Screenshot 2026-05-12 014246.png # Exp 2 Results
│   ├── Screenshot 2026-05-12 014532.png # Exp 3 Results
│   └── Screenshot 2026-05-12 014652.png # Exp 4 Results
│
└── README.md
```

---

## Source Code Changes — What We Modified and Why
This table summarizes every modification made to the ClickHouse C++ source during the project.

| File | Location in Source | Our Modification | Experiment |
|------|--------------------|------------------|------------|
| `ReadFromMergeTree.cpp` | `src/Processors/QueryPlan/` | `const size_t num_streams = 1;` — Disabled parallel streams when generating execution pipes. | **Exp 1** |
| `FilterTransform.cpp` | `src/Processors/Transforms/` | Added an early `return;` in `doTransform()` to prevent `chunk.setColumns()` from doing actual filtering work early on. | **Exp 2** |
| `ExpressionAnalyzer.cpp` | `src/Interpreters/` | Hardcoded `return false;` in `allowEarlyConstantFolding()` to block query optimization. | **Exp 3** |
| `ExpressionTransform.cpp` | `src/Processors/Transforms/` | Added a duplicate call to `expression->execute(block, num_rows, ...)` inside `transform()`. | **Exp 4** |

---

## System Requirements
> **Warning:** Building ClickHouse from source requires significant resources.

| Component | Requirement |
|-----------|-------------|
| Operating System | Linux (Ubuntu 20.04+) or macOS. Windows users should use **WSL2**. |
| RAM | Minimum 8 GB or more strongly recommended for building |
| Disk Space |** ~70 GB** free (ClickHouse source + build artifacts) |
| Tools | `git`, `cmake`, `ninja`, `clang-14` |
| Build Time |Initial build: approximately **12-14 hours**. Incremental rebuild after source change: approximately **45-60 minutes** |
---

## Build Instructions
> These steps are required to reproduce the experiments by modifying the ClickHouse C++ source code.

```bash
# Step 1 — Install build dependencies (Ubuntu)
sudo apt-get install -y cmake ninja-build clang-14 libssl-dev

# Step 2 — Clone & Navigate into the ClickHouse source directory
git clone --recursive https://github.com/ClickHouse/ClickHouse.git raw/ClickHouse
cd raw/ClickHouse

# Step 3 — Apply the source code modification for your chosen experiment

# Step 4 — Create the build directory and compile
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo -G Ninja
ninja clickhouse-server clickhouse-client
```

> **Important:** Before building for a new experiment, revert any changes from the previous one to avoid interference between results.

---

## Running Each Experiment

| Experiment | What Was Tested |
|---|---|
| **Exp 1: Disable Parallelization** | Running a query with `num_streams = 1` vs default parallel streams. |
| **Exp 2: Disable Filter Pushdown** | Observing the cost of blocking early row filtration inside the processor pipeline. |
| **Exp 3: Disable Expression Optimizer** | Running a query with heavy constant expressions without the `ExpressionAnalyzer` folding them. |
| **Exp 4: Extra Expression Evaluation** | Forcing the `ExpressionTransform` processor to compute the expressions twice per chunk. |

---

## Experiment Results
<br/>

**Exp 1 — Parallel Disabled vs Enabled**
<img src="Screenshot 2026-05-12 013922.png" alt="Exp 1: Disable Parallelization" height="150" style="display:block; margin: 5px 0;" />
*Query execution time jumped from 811 ms to 1422 ms (1.75x slower) when reading 50M rows.*

<br/>

**Exp 2 — Filter Early vs Filter Late (Pushdown Blocked)**
<img src="Screenshot 2026-05-12 014246.png" alt="Exp 2: Disable Filter Pushdown" height="150" style="display:block; margin: 5px 0;" />
*Skipping the early filter transform increased execution time from 48 ms to 87 ms.*

<br/>

**Exp 3 — Expression Optimized vs Not Simplified**
<img src="Screenshot 2026-05-12 014532.png" alt="Exp 3: Disable Expression Optimizer" height="150" style="display:block; margin: 5px 0;" />
*When early constant folding was blocked, the query took 87 ms instead of 34 ms (2.5x slower).*

<br/>

**Exp 4 — Normal vs Extra Expression Evaluation**
<img src="Screenshot 2026-05-12 014652.png" alt="Exp 4: Extra Expression Evaluation" height="150" style="display:block; margin: 5px 0;" />
*Evaluating expressions twice per chunk doubled the query time (40 ms to 81 ms) and processed significantly more bytes.*

---

## Numbers That Surprised Us
| Finding | Value | Experiment |
|---------|-------|------------|
| Slower execution without parallel streams | **1.75× slower** (1422ms vs 811ms) | Exp 1 |
| Slower execution when blocking filter pushdown | **1.8× slower** (87ms vs 48ms) | Exp 2 |
| Slower execution without early constant folding | **2.5× slower** (87ms vs 34ms) | Exp 3 |
| Double expression evaluation overhead | **~2× slower**, processed **144 MiB** (vs 67 MiB) | Exp 4 |

---

## Conclusion
This project reverse-engineered the ClickHouse Query Execution Pipeline by modifying four core C++ processors and analyzers.

| Experiment | Property Exposed | Observed Impact |
|---|---|---|
| Exp 1: Disable Parallelization | Queries scale linearly with execution streams | Single stream execution increased query time by 75% |
| Exp 2: Disable Filter Pushdown | Early pruning is critical for pipeline efficiency | Delaying filters almost doubled execution time |
| Exp 3: Disable Expression Optimizer | Query analysis (AST constant folding) saves CPU | Unoptimized complex expressions slowed queries by 2.5× |
| Exp 4: Extra Expression Evaluation | Transform processors dictate actual processing cost | Extra computation per chunk doubled the runtime and memory processed |

ClickHouse's execution speed is not just about the MergeTree format on disk; it is equally dependent on its **vectorized Processor DAG**, **massive stream parallelism**, and **early optimization/filtering** at the plan level. Disabling any of these mechanisms results in immediate and measurable performance degradation.

---
