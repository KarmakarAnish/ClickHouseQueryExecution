# ClickHouse — Query Execution

![DA-IICT](https://img.shields.io/badge/DA--IICT-Big%20Data%20Engineering-blue?style=flat)
![Semester](https://img.shields.io/badge/Semester-2-blue?style=flat)
![ClickHouse](https://img.shields.io/badge/Topic-ClickHouse-informational?style=flat)
![Built From Source](https://img.shields.io/badge/Built%20From-Source-blue?style=flat)
![Status](https://img.shields.io/badge/Status-Complete-success?style=flat)

> Not a tutorial. Not documentation.  
> A reverse-engineering journal of how one of the world's fastest database engines executes queries — from source code to controlled experiments.

---

## Problem Statement

ClickHouse is a high-performance analytical database designed to execute large SQL queries over columnar data with very low latency. In OLAP workloads, the main challenge is not only storing large volumes of data, but also reading, filtering, transforming, aggregating, and returning results efficiently.

This project studies the **ClickHouse Query Execution Pipeline**: the part of ClickHouse that takes a SQL query and turns it into executable processors that scan data, apply expressions, filter rows, aggregate results, sort, limit, and return the final output.

While MergeTree handles the storage of data on disk, **ClickHouse's Query Execution Pipeline** is what actually processes that data at breakneck speeds. ClickHouse uses a **vectorized query execution engine**, meaning it processes data in chunks/blocks of columns rather than row-by-row.

The execution engine is built around a few core principles:

| Principle | What It Means |
|-----------|---------------|
| **Pipeline Processors** | Queries are translated into a graph of processors/transforms such as read, expression, filter, aggregation, sorting, and limit. |
| **Massive Parallelism** | Read streams and many pipeline processors can run in parallel, then merge later when required. |
| **Early Filtering & Optimizations** | ClickHouse tries to reduce unnecessary work early using optimizations such as constant folding, column pruning, PREWHERE movement, and plan/pipeline optimizations. |

---

## Project Scope

This project focuses on **ClickHouse Query Execution**.

The main execution path studied is:

```text
SQL text
-> Parser
-> AST
-> Analyzer
-> Query Tree
-> Planner
-> Query Plan
-> Query Pipeline
-> Processors
-> Pipeline Executor
-> Result
```

This project does **not** deeply analyze:

- Columnar storage internals
- MergeTree parts, marks, granules, sparse indexes, or background merges

Those are treated only as shallow read-source details because the focus is the query execution pipeline.

---

## Query Execution Flow

```mermaid
flowchart LR
    A["SQL Text"] --> B["Parser"]
    B --> C["AST"]
    C --> D["Analyzer"]
    D --> E["Query Tree"]
    E --> F["Planner"]
    F --> G["Query Plan"]
    G --> H["Query Pipeline"]
    H --> I["Processors"]
    I --> J["Pipeline Executor"]
    J --> K["Result"]
```

---

## High-Level Source File Trace

The goal is to connect each major execution phase to the source areas responsible for that phase.

| Execution Stage | Main Source Area / File | Role in the Query Path |
|---|---|---|
| SQL parsing | `src/Parsers/ParserSelectQuery.cpp` | Parses SQL text and creates the initial AST representation of the query. |
| AST representation | `src/Parsers/ASTSelectQuery.*` and related AST classes | Stores the syntactic structure of the SQL query before semantic analysis. |
| Analyzer / query-tree construction | `src/Interpreters/InterpreterSelectQueryAnalyzer.cpp` | Converts the parsed query into a richer query tree and prepares it for planning. |
| Analyzer passes and rewrites | `src/Analyzer/` and related interpreter/analyzer files | Applies semantic resolution and query rewrites such as column pruning, expression simplification, and query normalization. |
| Planning | `src/Planner/` and `src/Processors/QueryPlan/` | Converts the analyzed query tree into a logical/physical query plan made of plan steps. |
| MergeTree reading | `src/Processors/QueryPlan/ReadFromMergeTree.cpp` | Builds the read stage for MergeTree tables and decides how many read streams are used. |
| PREWHERE optimization | `src/Storages/MergeTree/MergeTreeWhereOptimizer.cpp` | Tries to move suitable filters from `WHERE` into `PREWHERE` so unnecessary columns/rows can be avoided earlier. |
| Expression analysis | `src/Interpreters/ExpressionAnalyzer.cpp` | Handles expression actions and early expression simplifications such as constant folding. |
| Expression execution | `src/Processors/Transforms/ExpressionTransform.cpp` | Executes expression actions over blocks/chunks during pipeline execution. |
| Processor abstraction | `src/Processors/IProcessor.h` | Defines the processor model used by ClickHouse pipelines. |
| Pipeline execution | `src/Processors/Executors/PipelineExecutor.cpp` | Schedules and runs processors until the query result is produced. |

The important idea is that ClickHouse does not directly execute SQL text. It gradually lowers the query through multiple internal forms:

```text
SQL text -> AST -> Query Tree -> Query Plan -> Query Pipeline -> Processors
```

Each lowering stage removes ambiguity and prepares the query for efficient execution.

---

## Repository Structure

```text
ClickHouse-Query-Execution/
│
├── Experiments/
│   ├── QEE1.png       # Code modification for read parallelism
│   ├── QEE2-I.png     # Code modification for PREWHERE pushdown in function 1
│   ├── QEE2-II.png    # Code modification for PREWHERE pushdown in function 2
│   ├── QEE3.png       # Code modification for early constant folding
│   └── QEE4.png       # Code modification for extra expression evaluation
│
├── Graphs/
│   ├── GE1.png        # Bar Diagram for result of read parallelism
│   ├── GE2.png        # Bar Diagram for result of PREWHERE pushdown
│   ├── GE3.png        # Bar Diagram for result of early constant folding
│   └── GE4.png        # Bar Diagram for result of extra expression evaluation
│
├── Results/           # Result images for each experiment
│   ├── QER1.png       # Exp 1 results
│   ├── QER2.png       # Exp 2 results
│   ├── QER3.png       # Exp 3 results
│   └── QER4.png       # Exp 4 results
│
└── README.md
```

---

## Source Code Experiments

We modified ClickHouse C++ source code directly to isolate important execution behaviors, then compared performance using query metrics such as `query_ms`, `read_bytes`, `read_rows`, `selected_rows`, and `selected_marks`.

| Experiment | Source File Modified | Interpretation |
|---|---|---|
| **Exp 1: Restrict Read Parallelism** | `src/Processors/QueryPlan/ReadFromMergeTree.cpp` | Restricts MergeTree read streams, so the same data is processed with less read parallelism. |
| **Exp 2: Disable Filter Pushdown** | `src/Storages/MergeTree/MergeTreeWhereOptimizer.cpp` | Prevents automatic movement of suitable `WHERE` conditions into `PREWHERE`. The filter remains correct, but early PREWHERE filtering is blocked. |
| **Exp 3: Disable Early Constant Folding** | `src/Interpreters/ExpressionAnalyzer.cpp` | Prevents early simplification of constant expressions, increasing CPU-side expression work. |
| **Exp 4: Force Extra Expression Evaluation** | `src/Processors/Transforms/ExpressionTransform.cpp` | Adds extra expression work on a copied block, then discards the result so output correctness and read volume remain unchanged. |

---

## Exact Source Code Changes

### Experiment 1 — Restrict Read Parallelism

**File:**

```text
src/Processors/QueryPlan/ReadFromMergeTree.cpp
```

**Change:** force the number of read streams to one.

```cpp
/// Query Execution Experiment 1: Restrict read parallelism.
const size_t num_streams = 1;
```

**Experiment outcome:**

| Metric | Parallelism On | Parallelism Off |
|---|---:|---:|
| `query_ms` | 811 | 1422 |
| `read_rows` | 50 mil | 50 mil |
| `read_bytes` | 1.30 GB | 1.30 GB |

![](./Graphs/GE1.png)

---

### Experiment 2 — Disable Filter Pushdown

**File:**

```text
src/Storages/MergeTree/MergeTreeWhereOptimizer.cpp
```

**Function 1:**

```cpp
void MergeTreeWhereOptimizer::optimize(
    SelectQueryInfo & select_query_info,
    const ContextPtr & context) const
```

**Change:** add an early return at the top of the function.

```cpp
void MergeTreeWhereOptimizer::optimize(
    SelectQueryInfo & select_query_info,
    const ContextPtr & context) const
{
    /// Query Execution Experiment 2:
    /// Disable automatic WHERE -> PREWHERE pushdown.
    /// The filter remains in WHERE, so correctness is preserved.
    /// ClickHouse cannot use PREWHERE early column filtering for this query.
    return;

    auto & select = select_query_info.query->as<ASTSelectQuery &>();
    if (!select.where() || select.prewhere())
        return;

    // existing original code continues below
}
```

**Function 2:**

```cpp
MergeTreeWhereOptimizer::FilterActionsOptimizeResult
MergeTreeWhereOptimizer::optimize(
    const ActionsDAG & filter_dag,
    const std::string & filter_column_name,
    const ContextPtr & context,
    bool is_final)
```

**Change:** add an early return at the top of this overload too.

```cpp
MergeTreeWhereOptimizer::FilterActionsOptimizeResult
MergeTreeWhereOptimizer::optimize(
    const ActionsDAG & filter_dag,
    const std::string & filter_column_name,
    const ContextPtr & context,
    bool is_final)
{
    /// Query Execution Experiment 2:
    /// Disable ActionsDAG-based filter-to-PREWHERE optimization as well.
    return {};

    WhereOptimizerContext where_optimizer_context;

    // existing original code continues below
}
```

**Why this is the correct code change:**

This change does **not** remove the filter. It only prevents ClickHouse from moving the filter from `WHERE` into `PREWHERE`. The query result remains correct, but ClickHouse loses the early-filtering advantage of PREWHERE.

**Experiment Outcome:**

| Metric | Pushdown Used | Pushdown Blocked |
|---|---:|---:|
| `query_ms` | 10 | 87 |
| `read_bytes` | 5.2 MB | 67.21 MB |
| `read_rows` / `selected_rows` | 250k | 5 mil |

![](./Graphs/GE2.png)

---

### Experiment 3 — Disable Early Constant Folding

**File:**

```text
src/Interpreters/ExpressionAnalyzer.cpp
```

**Function:**

```cpp
allowEarlyConstantFolding(...)
```

**Change:** force early constant folding to be disabled.

```cpp
/// Query Execution Experiment 3: Disable early constant folding.
return false;
```

**Experiment Outcome:**

| Metric | Expression Optimized | Expression Not Optimized |
|---|---:|---:|
| `query_ms` | 34 | 87 |
| `read_rows` | ~5 mil | ~5 mil |
| `read_bytes` | 67.21 MB | 67.21 MB |
| `selected_marks` | 617 | 617 |

![](./Graphs/GE3.png)

This does not disable every expression optimization in ClickHouse. It specifically disables **early constant folding**.

---

### Experiment 4 — Force Extra Expression Evaluation Cleanly

**File:**

```text
src/Processors/Transforms/ExpressionTransform.cpp
```

**Function:**

```cpp
void ExpressionTransform::transform(Chunk & chunk)
```

**Change:** run the extra expression evaluation on a copied block and discard the result.

```cpp
void ExpressionTransform::transform(Chunk & chunk)
{
    size_t num_rows = chunk.getNumRows();
    auto block = getInputPort().getHeader().cloneWithColumns(chunk.detachColumns());

    /// Query Execution Experiment 4:
    /// Force extra expression evaluation.
    /// Run the same expression once on a copied block and discard the result.
    /// This adds CPU expression work without changing the real output block.
    {
        auto extra_block = block;
        size_t extra_num_rows = num_rows;
        expression->execute(
            extra_block,
            extra_num_rows,
            false,
            false,
            [this]() { return isCancelled(); });
    }

    /// Normal real expression evaluation.
    expression->execute(
        block,
        num_rows,
        false,
        false,
        [this]() { return isCancelled(); });

    chunk.setColumns(block.getColumns(), num_rows);

    if (updater)
        updater->recordOutputChunk(chunk, block);
}
```

**Experiment Outcome:**

| Metric | Extra Expression Evaluation | Normal Expression |
|---|---:|---:|
| `query_ms` | 81 | 43 |
| `read_rows` | ~5 mil | ~5 mil |
| `read_bytes` | 67.21 MB | 67.21 MB |
| `selected_rows` | ~5 mil | ~5 mil |
| `selected_marks` | 617 | 617 |

![](./Graphs/GE4.png)

---

## System Requirements

> **Warning:** Building ClickHouse from source requires significant resources.

| Component | Requirement |
|-----------|-------------|
| Operating System | Linux Ubuntu 20.04+ or macOS. Windows users should use **WSL2**. |
| RAM | Minimum 8 GB; more is strongly recommended for building. |
| Disk Space | **~70 GB** free for ClickHouse source and build artifacts. |
| Tools | `git`, `cmake`, `ninja`, `clang-14` or compatible Clang toolchain. |
| Build Time | Initial build: **12-14 hours**. Incremental rebuild after a small source change takes **45-60 minutes**. |

---

## Build Instructions

> These steps are required to reproduce the experiments by modifying the ClickHouse C++ source code.

```bash
# Step 1 — Install build dependencies on Ubuntu
sudo apt-get install -y cmake ninja-build clang-14 libssl-dev

# Step 2 — Clone and navigate into ClickHouse source
git clone --recursive https://github.com/ClickHouse/ClickHouse.git raw/ClickHouse
cd raw/ClickHouse

# Step 3 — Apply the source code modification for one experiment only
# from the above or from the photos given in this repo.

# Step 4 — Create the build directory and compile
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo -G Ninja
ninja clickhouse-server clickhouse-client
```

> **Important:** Before building for a new experiment, revert any changes from the previous experiment. Do not mix two experiment patches in the same binary unless intentionally testing interaction effects.

---

## Running Each Experiment

| Experiment | What Was Tested | Clean Result Pattern |
|---|---|---|
| **Exp 1: Restrict Read Parallelism** | `num_streams = 1` vs default parallel streams. | Same data read, slower query time. |
| **Exp 2: Disable Filter Pushdown** | Automatic PREWHERE movement disabled vs enabled. | More read work and slower query time when pushdown is blocked. |
| **Exp 3: Disable Early Constant Folding** | Heavy constant expressions with and without early folding. | Same read volume, higher CPU/query time. |
| **Exp 4: Extra Expression Evaluation** | Extra expression execution on copied block vs normal execution. | Same read volume, higher CPU/query time. |

---

## Numbers That Matter

| Experiment | Finding | Value |
|---------|-------|------------|
| Exp 1 | Slower execution without parallel read streams | **1.75× slower**: 1422 ms vs 811 ms |
| Exp 2 | Blocking early PREWHERE filtering increases read work | **~67.21 MiB vs ~5.20 MiB** read and **87ms vs 10ms** |
| Exp 3 | Blocking early constant folding increases CPU time | **2.5× slower**: 87 ms vs 34 ms |
| Exp 4 | Extra expression evaluation adds CPU overhead | **~2× slower**: 81 ms vs 43 ms |

---

## Failure Analysis

The experiments expose how ClickHouse behaves when execution assumptions are disturbed. This section answers the stress/failure-style questions directly.

### 1. What happens when data size increases significantly?

When the data size grows, the scan stage must read more blocks, the expression/filter stages must process more rows, and aggregation or sorting stages may need more memory. If the query is scan-heavy, parallel read streams become important because a single stream may underuse available CPU and I/O bandwidth.

This is connected to Experiment 1. When read parallelism was restricted, ClickHouse read the same number of rows and bytes, but runtime increased from 811 ms to 1422 ms. This suggests that large scans depend strongly on parallel reading and pipeline throughput.

Expected behavior at larger scale:

- `read_rows` and `read_bytes` increase with data volume.
- Query time increases if the query must scan more blocks.
- More read streams can improve throughput until CPU, memory bandwidth, or disk I/O becomes the bottleneck.
- Memory-heavy operators such as aggregation, sorting, and joins may fail if memory limits are reached.

### 2. What happens under skew?

Skew means the data or keys are unevenly distributed. For example, in a `GROUP BY`, one key may appear far more often than others. Under skew, some processors or aggregation states may receive much more work than others.

Possible effects:

- Parallel workers may become imbalanced.
- Some aggregation states may become much larger.
- Runtime may be dominated by the slowest overloaded processor.
- Memory pressure can increase if one group or partition becomes very large.

ClickHouse's pipeline model helps parallelize execution, but skew can still reduce the benefit of parallelism because not all streams perform equal work.

### 3. What happens if an execution component becomes expensive?

Experiment 4 simulates this by forcing extra expression evaluation. The query reads the same amount of data, but runtime increases because the expression transform does extra CPU work.

This shows that query performance is not determined only by storage reads. Even after data is read efficiently, CPU-heavy expression evaluation can become a bottleneck.

### 4. What assumptions does this system rely on?

ClickHouse query execution relies on several practical assumptions:

- Queries are usually analytical and scan many rows but relatively fewer columns.
- Columnar reads and early filtering can reduce unnecessary work.
- Blocks/chunks are large enough to benefit from vectorized processing.
- Parallelism improves throughput when hardware resources are available.
- Optimizer decisions such as PREWHERE movement and constant folding are usually beneficial.
- Intermediate memory usage remains within configured resource limits.

If these assumptions fail, performance can degrade. For example, if filters cannot be pushed down, ClickHouse may read more data. If expressions are expensive, CPU can dominate. If data is skewed, parallel execution may become imbalanced.

---

## What the Experiments Demonstrate

| Experiment | System Behavior Isolated | Main Lesson |
|---|---|---|
| Exp 1 | Read parallelism | Parallel streams improve throughput for large scans. |
| Exp 2 | PREWHERE filter pushdown | Early filtering reduces unnecessary read work. |
| Exp 3 | Early constant folding | Analyzer-level simplification reduces repeated CPU work. |
| Exp 4 | ExpressionTransform CPU cost | Execution transforms can become bottlenecks even when read volume is unchanged. |

Together, these experiments show that ClickHouse performance comes from the interaction of storage-aware reading, query optimization, pipeline parallelism, and efficient CPU execution.

---

## Conclusion

This project reverse-engineered the ClickHouse Query Execution Pipeline by tracing the source code at a file level and modifying selected execution paths.

| Experiment | Property Exposed | Observed / Expected Impact |
|---|---|---|
| Exp 1: Restrict Read Parallelism | Read-stream parallelism matters for throughput | Single-stream reading increased query time while reading the same amount of data. |
| Exp 2: Disable Filter Pushdown | Early filtering reduces unnecessary read work | Blocking PREWHERE movement caused more data to be read and increased runtime. |
| Exp 3: Disable Early Constant Folding | Analyzer-level expression simplification saves CPU | Unoptimized constant expressions slowed execution without changing scan volume. |
| Exp 4: Extra Expression Evaluation | ExpressionTransform contributes direct CPU cost | Extra expression work increased runtime while read volume should remain unchanged. |

ClickHouse's execution speed is not only about the MergeTree storage format. It also depends heavily on its staged query lowering process, vectorized processor pipeline, read parallelism, analyzer optimizations, PREWHERE filtering, and efficient expression transforms.

The main system-design insight is that ClickHouse gains speed by doing less unnecessary work as early as possible and by executing the remaining work in parallel, vectorized blocks. The tradeoff is implementation complexity: the query passes through several internal representations, and performance depends on many interacting components.

---
