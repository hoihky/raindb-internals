---
title: "Chapter 9: Global Aggregates"
order: 9
---

# Chapter 9: Global Aggregates

A query like `SELECT SUM(amount) FROM sales WHERE region = 'us'` collapses an entire table (after filtering) into a single scalar. In relational algebra this is aggregation without a grouping key list — one implicit group containing every qualifying row. RainDB handles this path inside the vectorized scan operator rather than spinning up a hash aggregator: there is exactly one group, so a map-reduce over columnar batches is both simpler and faster than maintaining a hash table with one entry.

This chapter is split into two parts. **Part I** develops the theory: relational algebra's γ operator, SQL aggregate semantics, algebraic properties that enable parallel partial aggregation, distributed map-reduce patterns, NULL and empty-set behavior, and SIMD reduction. **Part II** walks through RainDB's concrete implementation — `PartialAgg`, `ComputeAggregateAsync`, optional SIMD via `ColumnarAggregateIntrinsics`, operator wiring through `VectorizedScanOperator`, and the `IAggregateQueryResult` API boundary.

---

# Part I: Theory

## 9.1 Relational algebra aggregation (γ)

Once you have selection, projection, and join, the next operator most query engines add is **grouped aggregation**, denoted γ in relational algebra. You read it as "group by G, then apply aggregates F":

```
γ_{G; F₁, F₂, …, Fₙ}(R)
```

`G` names the grouping columns. Each `Fᵢ` is an aggregate over some expression built from `R`'s attributes.

The special case that matters here is **G = ∅**. No columns participate in grouping, so the engine treats the entire input as one bucket. Every surviving row feeds the same accumulator. The answer is always a single tuple — one number per aggregate in the SELECT list.

| Notation | SQL equivalent | Result cardinality |
|----------|----------------|-------------------|
| `γ_{}; SUM(x)(R)` | `SELECT SUM(x) FROM R` | 1 row |
| `γ_{a}; SUM(x)(R)` | `SELECT a, SUM(x) FROM R GROUP BY a` | ≤ distinct `a` |
| `γ_{}; COUNT(*)(R)` | `SELECT COUNT(*) FROM R` | 1 row, never NULL |

γ is an extension operator: it consumes a bag of rows and emits a bag of group rows. With an empty group list, that bag has size 0 or 1. SQL almost always returns one row for scalar aggregates even when the filtered input is empty — RainDB follows that contract via `IAggregateQueryResult`.

### Pipeline placement

Logically, a global aggregate query is a thin stack:

```
π_{result_cols}( γ_{}; AGG(expr)( σ_{predicate}(R) ) )
```

Filter first, aggregate second, project last. RainDB collapses this into `VectorizedScanPhysicalPlan` with an optional `Aggregate` field. When `|G| = 0`, there is no reason to allocate a separate physical operator — the scan loop already touches every row.

## 9.2 Aggregate functions in the SQL standard

SQL aggregates consume a **multiset** of values drawn from the group. Duplicates count unless you write `DISTINCT` inside the function. The standard catalog includes:

| Function | Operand | Result type (typical) | Ignores NULLs? |
|----------|---------|----------------------|----------------|
| `COUNT(*)` | — | exact integer | N/A (counts rows) |
| `COUNT(expr)` | any | exact integer | yes |
| `SUM(expr)` | numeric | same or wider numeric | yes |
| `AVG(expr)` | numeric | numeric | yes |
| `MIN(expr)` | comparable | same as expr | yes |
| `MAX(expr)` | comparable | same as expr | yes |

RainDB ships `COUNT`, `SUM`, `MIN`, and `MAX` on the global scan path. `MIN`/`MAX` on global aggregates are validated for `Float64` measure columns at plan time (grouped hash aggregation supports a wider type set — see Chapter 10). You can compute an average with `SUM(x) / COUNT(x)`; there is no dedicated `AVG` opcode.

### Set vs bag semantics

`SUM(amount)` over `(10, 10, 5)` is `25`. The engine does not deduplicate unless you ask with `COUNT(DISTINCT col)` on the grouped path.

### Filter interaction

`WHERE` trims rows **before** aggregation. `HAVING` trims groups **after**. A query without `GROUP BY` only ever sees `WHERE`:

```sql
SELECT SUM(x) FROM t WHERE y > 0;
-- γ_{}; SUM(x)( σ_{y>0}(t) )
```

RainDB evaluates `ColumnCompareFilter[]` conjunctively while scanning batches. Rows that fail the predicate never reach the aggregate kernel.

## 9.3 Algebraic properties

Parallel engines only work if you can split the input, summarize each chunk independently, and merge summaries later. That requires **decomposability**: `whole = Combine(chunk₀, chunk₁, …)`.

### Associativity

⊕ is associative when `(a ⊕ b) ⊕ c = a ⊕ (b ⊕ c)`.

| Aggregate | Combine operation | Associative? |
|-----------|-------------------|--------------|
| `SUM` | addition | yes (modulo overflow / FP rounding) |
| `COUNT` | addition | yes |
| `MIN` | minimum | yes |
| `MAX` | maximum | yes |
| `AVG` | needs `(sum, count)` pair | associative on the pair |

Associativity is what lets you fold partials in a tree, a thread pool, or a coordinator merge without caring about order.

### Commutativity

When `a ⊕ b = b ⊕ a`, merge order is irrelevant. `SUM`, `COUNT`, `MIN`, and `MAX` all commute on their partial representations. RainDB's `PartialAgg.Combine` therefore does not need to replay batches in ingestion order.

### Distributivity (limited)

Addition distributes across disjoint partitions: `SUM(A ∪ B) = SUM(A) + SUM(B)` when A and B share no rows. `COUNT` behaves the same way.

`MIN` and `MAX` are not additive, but they **do** form idempotent semilattices: `min(a, b)` and `max(a, b)` are associative and commutative if you track whether each side has seen a real value. A partial that only observed nulls must not contribute a fake numeric zero when merged with a partial that saw data.

Practical partial state for extrema:

1. Best value so far (once one exists)
2. `has_value` flag — distinguishes "never saw a non-null" from "saw zero"

### Decomposable aggregate functions (DAF)

Literature calls aggregates **decomposable** when a fixed-size partial plus a `Combine` function suffices. Every RainDB Phase 1 aggregate qualifies. `MEDIAN`, `PERCENTILE_CONT`, and `MODE` do not — they need order statistics or unbounded state.

## 9.4 Map-reduce pattern

Partitioned global aggregation is textbook map-reduce with a constant key:

```
         ┌─────────┐     ┌─────────┐     ┌─────────┐
Input ──►│ Map P₀  │     │ Map P₁  │ ... │ Map Pₙ  │
         └────┬────┘     └────┬────┘     └────┬────┘
              │ partial₀      │ partial₁      │ partialₙ
              └───────────────┼───────────────┘
                              ▼
                        ┌───────────┐
                        │  Reduce   │  Combine(partials)
                        └─────┬─────┘
                              ▼
                         final scalar
```

**Map:** each partition scans its rows and emits a compact `PartialAgg`.

**Reduce:** `Combine` folds partials. RainDB uses a serial fold today; nothing stops a parallel tree reduction.

The pattern matches distributed frameworks, except the group key is always the empty tuple.

### Why not hash for global agg?

A hash table keyed by "the one group" still pays for hashing, pointer chasing, and bucket management. When `|G| = 0`, fusion into the scan wins. `VectorizedScanOperator.ComputeAggregateAsync` is the fused path.

## 9.5 Partial aggregation in distributed systems

In a cluster, each node emits one partial per aggregate; the coordinator merges them. Network cost is **O(1)** per aggregate per node, not **O(rows)**.

Design the partial carefully:

| Concern | Implication |
|---------|-------------|
| Fixed-size partial | Stack allocation, SIMD-friendly structs |
| Associative combine | Any merge order valid |
| Identity element | Empty partition contributes neutral state |
| Null awareness | All-null partition ≠ partition with values |

RainDB mirrors this inside one process: each columnar batch is a partition, `PartialAgg[n]` holds `n` local summaries.

### Two-level parallelism

1. **Morsel parallelism** — process batches on different threads (`Parallel.For` or channel workers).
2. **Intra-batch SIMD** — AVX2 horizontal sum/min/max on dense, null-free columns when scan options allow.

Combine stays serial. That is fine when batch count is tiny compared to row count.

## 9.6 NULL handling in COUNT, SUM, MIN, MAX

Predicates use three-valued logic (`TRUE`, `FALSE`, `UNKNOWN`). Aggregates use a simpler rule: NULL means **skip this value**, except `COUNT(*)` which counts rows regardless of column nulls.

### COUNT

| Form | Counts | NULL in column? |
|------|--------|-----------------|
| `COUNT(*)` | rows in group | irrelevant |
| `COUNT(col)` | rows where `col IS NOT NULL` | excluded |
| `COUNT(DISTINCT col)` | distinct non-null values | grouped path only |

`COUNT(expr)` is never NULL. Zero rows → `0`.

### SUM, MIN, MAX

Skip NULL operands. If nothing remains, emit **NULL** — not zero for `SUM`, not an arbitrary sentinel for `MIN`/`MAX`.

### Implementation pattern

Track how many non-null values contributed (`contributing_row_count` or `HasMin`/`HasMax`):

```
if contributing_row_count == 0:
    emit NULL (except COUNT variants)
else:
    emit computed value
```

RainDB's `PartialAgg.ContributingRows` and `HasMin`/`HasMax` encode this contract on the global path. Grouped aggregates write null bits on output column chunks using the same rules (Chapter 10).

## 9.7 Empty set semantics

Four cases show up constantly in tests and bug reports:

| Situation | Rows after WHERE | Non-null measures | COUNT(*) | SUM(x) |
|-----------|------------------|-------------------|----------|--------|
| Empty table | 0 | — | 0 | NULL |
| All filtered out | 0 | — | 0 | NULL |
| Rows exist, all x NULL | >0 | 0 | row count | NULL |
| Normal | >0 | >0 | row count | numeric |

Scalar aggregate queries return **one row** even when the input is empty. RainDB exposes `IAggregateQueryResult` with `RowCount = 1` and `ValueIsNull` set correctly — not an empty columnar batch.

Grouped queries behave differently (Chapter 10): no input rows means no groups, so zero output rows.

## 9.8 SIMD reduction theory

SIMD applies one instruction to multiple lanes at once — four `double` values in an AVX2 `Vector256<double>`, for example.

### Horizontal reduction

To sum `n` doubles:

1. **Vector phase** — load four values per iteration, add into a vector accumulator.
2. **Horizontal fold** — collapse accumulator lanes to one scalar.
3. **Scalar tail** — handle `n % 4` leftovers.

Still O(n) work, but higher throughput. On wide analytic columns, memory bandwidth usually caps you before ALU throughput.

### Preconditions for safe SIMD aggregates

| Condition | Reason |
|-----------|--------|
| No NULLs | Null bitmap needs masking or gather |
| Dense row order | No selection-vector indirection |
| Fixed width | Known stride (`sizeof(double)` / `sizeof(int)`) |
| Associative FP caveat | Reordering changes rounding bits |

RainDB enables SIMD only when `!col.HasNulls && selectedRows.IsEmpty` and the matching `VectorizedScanExecutionOptions` flag is set (`UseAvx2DoubleSum`, `UseAvx2DoubleMinMax`, or `UseAvx2IntegerSum`).

### Gather vs contiguous load

A `WHERE` clause builds a **selection vector** of row indices. Contiguous SIMD loads assume row `i` lives at offset `i`. With filtering, you either gather indirectly or stay scalar. RainDB chooses scalar for filtered paths — gather setup often loses on small selections.

### SIMD coverage in RainDB

`ColumnarAggregateIntrinsics` in `src/RainDB.Core/Columnar/` implements `SumFloat64`, `MinFloat64`, `MaxFloat64` (AVX2 when available), plus `SumInt32` (Vector128 lanes) and `SumInt64` (AVX2). The scan operator calls these through `QueryOperatorDependencies.Aggregates` when the fast-path preconditions hold.

---

# Part II: RainDB Implementation

## 9.9 Operator model and entry point

Physical execution is organized around **instance operator classes** injected through `IQueryOperatorSuite` / `DefaultQueryOperatorSuite` into `DefaultQueryExecutor`. Global aggregation lives in `VectorizedScanOperator` (`src/RainDB.Query/Execution/VectorizedScanEngine.cs`), which implements `IVectorizedScanOperator`.

Shared columnar helpers — selection, projection, join materialization, sort top-N, group keys, and **`IColumnarAggregateIntrinsics`** — are bundled in internal `QueryOperatorDependencies` (`src/RainDB.Query/Execution/Operators/QueryOperatorDependencies.cs`). The default aggregate implementation is `ColumnarAggregateIntrinsics`.

When `VectorizedScanPhysicalPlan.Aggregate` is set, `ExecuteAsync` routes to `ComputeAggregateAsync` instead of projecting batches:

```csharp
// src/RainDB.Query/Execution/VectorizedScanEngine.cs — VectorizedScanOperator.ExecuteAsync
if (plan.Aggregate is { } agg)
    return await ComputeAggregateAsync(plan, table, agg, context).ConfigureAwait(false);
```

Callers receive `IAggregateQueryResult` (as `IQueryResult`) rather than `IColumnarQueryResult`. The scalar value and `ValueIsNull` encode SQL semantics at the API boundary.

## 9.10 AggregateSpec and execution options

```csharp
// src/RainDB.Query/Plans/VectorizedScanPhysicalPlan.cs
public readonly record struct AggregateSpec(int SourceColumnIndex, AggregateKind Kind);
```

| Field | Meaning |
|-------|---------|
| `SourceColumnIndex` | Column to measure; **-1** means `COUNT(*)` |
| `Kind` | `Count`, `Sum`, `Min`, or `Max` |

`AggregateKind` is defined in `src/RainDB.Abstractions/Execution/AggregateKind.cs`. `ValidatePlan` enforces type/kind compatibility before parallel work:

- `Sum` → `Int32`, `Int64`, or `Float64`
- `Min` / `Max` → **`Float64` only** on the global scan path
- `Count` → any column type, or index -1 for star

The SQL binder (`LogicalTableScanBinder` in `src/RainDB.Sql/Compilation/`) produces these specs from `SELECT` without `GROUP BY`.

```csharp
// src/RainDB.Query/Plans/VectorizedScanPhysicalPlan.cs
public readonly record struct VectorizedScanExecutionOptions
{
    public int MaxDegreeOfParallelism { get; init; }  // -1 = ProcessorCount
    public bool UseChannelScheduler { get; init; }
    public bool UseAvx2DoubleSum { get; init; }
    public bool UseAvx2DoubleMinMax { get; init; }
    public bool UseAvx2IntegerSum { get; init; }
}
```

`UseAvx2DoubleMinMax` gates full-column, null-free `Min`/`Max` on `Float64`. `UseAvx2IntegerSum` gates the same shape of fast path for `Sum` on `Int32`/`Int64`.

## 9.11 ComputeAggregateAsync: map-reduce over batches

`ComputeAggregateAsync` allocates one `PartialAgg` per source batch, fills them in parallel (when `MaxDegreeOfParallelism` allows), then folds with `PartialAgg.Combine` and `ToResult`.

Empty table short-circuits to `EmptyAggregate` before any partial work — `COUNT` returns 0 with `ValueIsNull: false`; `SUM`/`MIN`/`MAX` return placeholders with `ValueIsNull: true`.

Scheduling mirrors non-aggregate projection: sequential loop, `Parallel.For`, or `RunChannelMorselsAsync` when `UseChannelScheduler` is true. Each worker writes to `partials[i]` with no locking.

## 9.12 PartialAgg accumulator

`PartialAgg` is a private `readonly struct` inside `VectorizedScanOperator` holding:

- `ContributingRows` — non-null operands for `SUM` / `MIN` / `MAX`
- `FloatSum`, `IntSum` — numeric sums
- `FloatMin`, `FloatMax`, `HasMin`, `HasMax` — `Float64` extrema
- `CountAgg` — `COUNT(*)` and `COUNT(col)`

`Combine` is associative per `AggregateKind`. `CombineMinMax` treats a side with `HasMin == false` as identity so an all-null batch does not poison a merge.

## 9.13 Per-batch accumulation and filters

`AccumulateAggregateBatch` distinguishes `COUNT(*)` (filtered row count) from `COUNT(col)` (non-null count via null bitmap). When `plan.Filters` is non-empty, `SelectionEvaluator.FillSelectedRowsConjunctive` fills a rented `int[]` selection buffer; when filters are absent, iteration is dense (`selectedRows` empty, `k = batch.RowCount`).

`FromColumn` dispatches `Sum` / `Min` / `Max` by physical type. Integer sums widen into `long` in `IntSum`.

## 9.14 SIMD fast paths

Fast paths share the same predicate: **no selection vector, no nulls, option flag enabled**.

**Float64 sum** — `deps.Aggregates.SumFloat64(values, allowAvx2: true)` when `UseAvx2DoubleSum`.

**Float64 min/max** — `MinFloat64` / `MaxFloat64` when `UseAvx2DoubleMinMax`:

```csharp
// src/RainDB.Query/Execution/VectorizedScanEngine.cs — MinMaxFloat64 (excerpt)
if (selectedRows.IsEmpty && !col.HasNulls && options.UseAvx2DoubleMinMax)
{
    var x = kind == AggregateKind.Min
        ? deps.Aggregates.MinFloat64(values, allowAvx2: true)
        : deps.Aggregates.MaxFloat64(values, allowAvx2: true);
    // PartialAgg with HasMin/HasMax set, ContributingRows = selectedCount
}
```

**Int32 / Int64 sum** — `SumInt32` / `SumInt64` when `UseAvx2IntegerSum`.

Otherwise scalar loops skip nulls via `deps.Selection.IsNull`. Filtered batches always take the scalar path.

`ColumnarAggregateIntrinsics` (`src/RainDB.Core/Columnar/ColumnarAggregateIntrinsics.cs`) uses AVX2 `Add`/`Min`/`Max` on `Vector256<double>` with a scalar tail. Integer `SumInt32` uses Vector128 loads; `SumInt64` uses AVX2 add. Hardware absence or `allowAvx2: false` falls back to scalar loops in the same class.

## 9.15 ToResult and AggregateQueryResult

`AggregateQueryResult` (`src/RainDB.Query/Results/ColumnarAndAggregateResults.cs`) always reports `RowCount = 1`. `ToResult` sets `ValueIsNull` when:

- `SUM`: `ContributingRows == 0`
- `MIN`/`MAX`: `!HasMin` / `!HasMax`
- `COUNT`: never null

`EmptyAggregate` handles zero batches before any partial exists — distinct from "rows exist but all measures are null."

## 9.16 Global vs grouped aggregates

| Aspect | Global (`PartialAgg`) | Grouped (`AggregateAccumulator`, Ch. 10) |
|--------|------------------------|------------------------------------------|
| Scope | One combined result | One accumulator per group key |
| `MIN`/`MAX` types | `Float64` only (scan validation) | `Int32`, `Int64`, `Float64`, `Utf8` |
| NULL output | `IAggregateQueryResult.ValueIsNull` | Null bits on aggregate columns in `ColumnarBatch` |
| Operator | `VectorizedScanOperator` | `HashAggregateOperator` |

Grouped null semantics: `ShouldEmitAggregateNull` in hash aggregation mirrors `ToResult` — all-null measure values in a group yield NULL for `SUM`/`MIN`/`MAX`, while `COUNT(col)` stays `0` without a null bit and `COUNT(*)` counts rows in the group.

## 9.17 End-to-end trace

Query: `SELECT SUM(amount) FROM sales WHERE region_id = 3`

Physical plan: `VectorizedScanPhysicalPlan` with filter on `region_id`, `Aggregate = (amount_ix, Sum)`.

```
DefaultQueryExecutor → _operators.Scan.ExecuteAsync
  → ComputeAggregateAsync
      per batch: AccumulateAggregateBatch → PartialAgg
      Combine → ToResult → AggregateQueryResult
```

## 9.18 Summary

Global aggregation in RainDB is map-reduce over immutable columnar batches, fused into the scan operator:

1. **Plan** — `AggregateSpec` on `VectorizedScanPhysicalPlan`.
2. **Map** — `PartialAgg` per batch with filter-aware selection.
3. **Reduce** — `PartialAgg.Combine`.
4. **Materialize** — `ToResult` / `EmptyAggregate` with SQL-correct `ValueIsNull`.
5. **Accelerate** — `ColumnarAggregateIntrinsics` behind `UseAvx2DoubleSum`, `UseAvx2DoubleMinMax`, and `UseAvx2IntegerSum` on dense null-free columns.

The design avoids hash-table overhead when the group count is exactly one, while sharing intrinsics and selection code with the rest of the operator suite via `QueryOperatorDependencies`.
