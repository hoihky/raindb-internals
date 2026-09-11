---
title: "Chapter 9: Global Aggregates"
order: 9
---

# Chapter 9: Global Aggregates

A query like `SELECT SUM(amount) FROM sales WHERE region = 'us'` collapses an entire table (after filtering) into a single scalar. In relational algebra this is aggregation without a grouping key list — one implicit group containing every qualifying row. RainDB handles this path inside `VectorizedScanEngine` rather than spinning up a hash aggregator: there is exactly one group, so a map-reduce over columnar batches is both simpler and faster than maintaining a hash table with one entry.

This chapter is split into two parts. **Part I** develops the theory: relational algebra's γ operator, SQL aggregate semantics, algebraic properties that enable parallel partial aggregation, distributed map-reduce patterns, NULL and empty-set behavior, and SIMD reduction. **Part II** walks through RainDB's concrete implementation — `PartialAgg`, `ComputeAggregateAsync`, AVX2 via `AggregateIntrinsics`, and the `IAggregateQueryResult` API boundary.

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

RainDB Phase 1 ships `COUNT`, `SUM`, `MIN`, and `MAX`. You can compute an average yourself with `SUM(x) / COUNT(x)`; there is no dedicated `AVG` opcode yet.

### Set vs bag semantics

`SUM(amount)` over `(10, 10, 5)` is `25`. The engine does not deduplicate unless you ask it to with `COUNT(DISTINCT col)` or similar — not implemented in RainDB today.

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

A hash table keyed by "the one group" still pays for hashing, pointer chasing, and bucket management. When `|G| = 0`, fusion into the scan wins. `VectorizedScanEngine.ComputeAggregateAsync` is the fused path.

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
2. **Intra-batch SIMD** — AVX2 horizontal sum on dense, null-free `Float64` columns.

Combine stays serial. That is fine when batch count is tiny compared to row count.

## 9.6 NULL handling in COUNT, SUM, MIN, MAX

Predicates use three-valued logic (`TRUE`, `FALSE`, `UNKNOWN`). Aggregates use a simpler rule: NULL means **skip this value**, except `COUNT(*)` which counts rows regardless of column nulls.

### COUNT

| Form | Counts | NULL in column? |
|------|--------|-----------------|
| `COUNT(*)` | rows in group | irrelevant |
| `COUNT(col)` | rows where `col IS NOT NULL` | excluded |
| `COUNT(DISTINCT col)` | distinct non-null values | excluded (not in RainDB yet) |

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

RainDB's `PartialAgg.ContributingRows` and `HasMin`/`HasMax` encode this contract.

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

### Preconditions for safe SIMD sum

| Condition | Reason |
|-----------|--------|
| No NULLs | Null bitmap needs masking or gather |
| Dense row order | No selection-vector indirection |
| Fixed width | Known stride (`sizeof(double)`) |
| Associative FP caveat | Reordering changes rounding bits |

RainDB enables AVX2 only when `!col.HasNulls && selectedRows.IsEmpty && options.UseAvx2DoubleSum`.

### Gather vs contiguous load

A `WHERE` clause builds a **selection vector** of row indices. Contiguous SIMD loads assume row `i` lives at offset `i`. With filtering, you either gather indirectly or stay scalar. RainDB chooses scalar for filtered paths — gather setup often loses on small selections.

### Other SIMD opportunities (not yet in RainDB)

- `MIN`/`MAX` on floats via `vminpd`/`vmaxpd`
- Integer `SUM` with widening (`vpmovzx` + `vpaddq`)
- `COUNT` via popcount on known-non-null bitmaps

`AggregateIntrinsics` in RainDB.Core implements AVX2 `SumFloat64` today; min/max helpers exist but run scalar.

---

# Part II: RainDB Implementation

## 9.9 Where global aggregates live

Global aggregates are not a separate physical operator. The scan physical plan carries an optional `Aggregate` field; when it is set, `ExecuteAsync` routes to `ComputeAggregateAsync` instead of projecting batches.

```csharp
// src/RainDB.Query/Execution/VectorizedScanEngine.cs
public static async ValueTask<IQueryResult> ExecuteAsync(
    VectorizedScanPhysicalPlan plan,
    IColumnarTableSource table,
    IExecutionContext context)
{
    ArgumentNullException.ThrowIfNull(plan);
    ArgumentNullException.ThrowIfNull(table);
    ArgumentNullException.ThrowIfNull(context);
    ValidatePlan(plan, table);

    if (plan.Aggregate is { } agg)
        return await ComputeAggregateAsync(plan, table, agg, context).ConfigureAwait(false);

    var batches = await ProjectAllBatchesAsync(plan, table, context).ConfigureAwait(false);
    return new ColumnarMaterializedQueryResult(batches);
}
```

The result type changes: callers receive `IAggregateQueryResult` (wrapped in `IQueryResult`) rather than `IColumnarQueryResult`. The scalar value and a `ValueIsNull` flag encode SQL semantics at the API boundary.

`DefaultQueryExecutor` dispatches `VectorizedScanPhysicalPlan` to this engine without inspecting aggregate presence — the engine branches internally.

## 9.10 AggregateSpec and physical plan

`AggregateSpec` is defined alongside `VectorizedScanPhysicalPlan`:

```csharp
// src/RainDB.Query/Plans/VectorizedScanPhysicalPlan.cs
public readonly record struct AggregateSpec(int SourceColumnIndex, AggregateKind Kind);
```

| Field | Meaning |
|-------|---------|
| `SourceColumnIndex` | Column to measure; **-1** means `COUNT(*)` |
| `Kind` | `Count`, `Sum`, `Min`, or `Max` |

`AggregateKind` lives in `src/RainDB.Abstractions/Execution/AggregateKind.cs`:

```csharp
public enum AggregateKind
{
    None,
    Sum,
    Min,
    Max,
    /// <summary>COUNT(*) uses SourceColumnIndex -1; COUNT(col) uses the column index.</summary>
    Count,
}
```

Validation in `ValidatePlan` enforces type/kind compatibility before any parallel work begins:

- `Sum` requires `Int32`, `Int64`, or `Float64`
- `Min` / `Max` require `Float64` only
- `Count` accepts any column type (or -1 for star)
- A negative `SourceColumnIndex` is rejected unless `Kind == Count`

The SQL binder (`LogicalTableScanBinder` in `src/RainDB.Sql/Compilation/`) produces these specs from `SELECT SUM(x), COUNT(*), ...` without a `GROUP BY` clause.

### Execution options

```csharp
// src/RainDB.Query/Plans/VectorizedScanPhysicalPlan.cs
public readonly record struct VectorizedScanExecutionOptions
{
    public int MaxDegreeOfParallelism { get; init; }  // -1 = ProcessorCount
    public bool UseChannelScheduler { get; init; }
    public bool UseAvx2DoubleSum { get; init; }
}
```

## 9.11 ComputeAggregateAsync: map-reduce over batches

```csharp
// src/RainDB.Query/Execution/VectorizedScanEngine.cs
private static async ValueTask<IAggregateQueryResult> ComputeAggregateAsync(
    VectorizedScanPhysicalPlan plan,
    IColumnarTableSource table,
    AggregateSpec spec,
    IExecutionContext context)
{
    var batches = table.Batches;
    var n = batches.Count;
    var ct = context.CancellationToken;
    if (n == 0)
        return EmptyAggregate(spec, table.Schema);

    var dop = EffectiveDop(plan.Options.MaxDegreeOfParallelism);
    var partials = new PartialAgg[n];
    if (dop <= 1 || n == 1)
    {
        for (var i = 0; i < n; i++)
            partials[i] = AccumulateAggregateBatch(plan, batches[i], spec, plan.Options);
    }
    else if (plan.Options.UseChannelScheduler)
    {
        await RunChannelMorselsAsync(
            n, dop,
            i => partials[i] = AccumulateAggregateBatch(plan, batches[i], spec, plan.Options),
            ct).ConfigureAwait(false);
    }
    else
    {
        Parallel.For(
            0, n,
            new ParallelOptions { MaxDegreeOfParallelism = dop, CancellationToken = ct },
            i => partials[i] = AccumulateAggregateBatch(plan, batches[i], spec, plan.Options));
    }

    var combined = partials[0];
    for (var i = 1; i < n; i++)
        combined = PartialAgg.Combine(combined, partials[i], spec.Kind);

    var resultType = spec.SourceColumnIndex >= 0
        ? table.Schema.Columns[spec.SourceColumnIndex].Type
        : RainDbType.Int64;
    return combined.ToResult(resultType, spec.Kind);
}
```

**Map:** `AccumulateAggregateBatch` per batch index.

**Reduce:** sequential `PartialAgg.Combine` fold.

**Empty table:** `EmptyAggregate` before any partial allocation.

## 9.12 EmptyAggregate: SQL empty-input rules

```csharp
private static IAggregateQueryResult EmptyAggregate(AggregateSpec spec, TableSchema schema)
{
    var columnType = spec.SourceColumnIndex >= 0
        ? schema.Columns[spec.SourceColumnIndex].Type
        : RainDbType.Int64;
    return spec.Kind switch
    {
        AggregateKind.Count =>
            new AggregateQueryResult(RainDbType.Int64, 0d, 0L, 0, valueIsNull: false),
        AggregateKind.Sum when columnType == RainDbType.Float64 =>
            new AggregateQueryResult(RainDbType.Float64, 0d, 0L, 0, valueIsNull: true),
        AggregateKind.Sum =>
            new AggregateQueryResult(RainDbType.Int64, 0d, 0L, 0, valueIsNull: true),
        AggregateKind.Min or AggregateKind.Max when columnType == RainDbType.Float64 =>
            new AggregateQueryResult(RainDbType.Float64, 0d, 0L, 0, valueIsNull: true),
        _ => throw new InvalidOperationException("Unsupported empty aggregate."),
    };
}
```

| Aggregate | Empty table | `ValueIsNull` |
|-----------|-------------|---------------|
| `COUNT(*)` / `COUNT(col)` | `0` | `false` |
| `SUM` / `MIN` / `MAX` | placeholder | `true` |

## 9.13 PartialAgg: private accumulator struct

```csharp
private readonly struct PartialAgg
{
    public readonly long ContributingRows;
    public readonly double FloatSum;
    public readonly double FloatMin;
    public readonly double FloatMax;
    public readonly long IntSum;
    public readonly bool HasMin;
    public readonly bool HasMax;
    public readonly long CountAgg;
}
```

| Field | Used by |
|-------|---------|
| `ContributingRows` | `SUM`, `MIN`, `MAX` — non-null value count |
| `FloatSum` | `SUM` on `Float64` |
| `IntSum` | `SUM` on `Int32` / `Int64` |
| `FloatMin` / `HasMin` | `MIN` on `Float64` |
| `FloatMax` / `HasMax` | `MAX` on `Float64` |
| `CountAgg` | `COUNT(*)` and `COUNT(col)` |

One struct for all kinds avoids virtual dispatch. `readonly struct` enables stack allocation and copy-by-value combine without heap churn.

### Factory methods

**`FromCountStar`** — filtered row count only:

```csharp
public static PartialAgg FromCountStar(int selectedRowCount) =>
    new PartialAgg(0, 0d, 0d, 0d, 0L, false, false, selectedRowCount);
```

**`FromCountColumn`** — skips nulls via null bitmap:

```csharp
public static PartialAgg FromCountColumn(IColumnChunk col, ReadOnlySpan<int> selectedRows, int selectedCount)
{
    var nb = col.HasNulls ? col.NullBitmap.Span : ReadOnlySpan<byte>.Empty;
    long c = 0;
    for (var i = 0; i < selectedCount; i++)
    {
        var r = Row(selectedRows, i);
        if (!SelectionEvaluator.IsNull(nb, r, col.HasNulls))
            c++;
    }
    return new PartialAgg(0, 0d, 0d, 0d, 0L, false, false, c);
}
```

**`FromColumn`** — dispatches `Sum` / `Min` / `Max` by `col.PhysicalType`.

Helper `Row(selectedRows, i)` returns `i` when `selectedRows` is empty (dense iteration).

## 9.14 AccumulateAggregateBatch and filters

```csharp
private static PartialAgg AccumulateAggregateBatch(
    VectorizedScanPhysicalPlan plan,
    IColumnarBatch batch,
    AggregateSpec spec,
    VectorizedScanExecutionOptions options)
{
    if (spec.Kind == AggregateKind.Count && spec.SourceColumnIndex < 0)
        return AccumulateFilteredRowCount(plan, batch);

    if (spec.Kind == AggregateKind.Count)
    {
        // rent int[] selection buffer → FromCountColumn
    }

    // rent int[] selection buffer → FromColumn(measureCol, spec.Kind, sel, k, options)
}
```

When `plan.Filters` is non-empty, `SelectionEvaluator.FillSelectedRowsConjunctive` writes qualifying row indices into a rented `ArrayPool<int>` buffer. When no filters exist, `k = batch.RowCount` and `selectedRows` is empty — dense path.

Semantic distinction:

- **`COUNT(*)`** — counts rows surviving `WHERE`
- **`COUNT(col)`** — counts non-null `col` among those rows

## 9.15 PartialAgg.Combine

```csharp
public static PartialAgg Combine(PartialAgg a, PartialAgg b, AggregateKind kind) =>
    kind switch
    {
        AggregateKind.Sum => new PartialAgg(
            a.ContributingRows + b.ContributingRows,
            a.FloatSum + b.FloatSum,
            0d, 0d,
            a.IntSum + b.IntSum,
            false, false, 0),
        AggregateKind.Count => new PartialAgg(
            0, 0d, 0d, 0d, 0L, false, false,
            a.CountAgg + b.CountAgg),
        AggregateKind.Min => CombineMinMax(a, b, isMin: true),
        AggregateKind.Max => CombineMinMax(a, b, isMin: false),
        _ => throw new ArgumentOutOfRangeException(nameof(kind), kind, null),
    };
```

`CombineMinMax` handles identity (no extremum yet):

```csharp
if (!a.HasMin) return b;
if (!b.HasMin) return a;
return new PartialAgg(rows, 0d, Math.Min(a.FloatMin, b.FloatMin), 0d, 0L, true, false, 0);
```

If both sides are empty, combined partial has `HasMin == false` → `ToResult` emits NULL.

## 9.16 ToResult and AggregateQueryResult

```csharp
// src/RainDB.Query/Results/ColumnarAndAggregateResults.cs
public sealed class AggregateQueryResult : IAggregateQueryResult
{
    public AggregateQueryResult(
        RainDbType resultType,
        double float64Value,
        long int64Value,
        long contributingRowCount,
        bool valueIsNull = false)
    {
        ResultType = resultType;
        Float64Value = float64Value;
        Int64Value = int64Value;
        ContributingRowCount = contributingRowCount;
        ValueIsNull = valueIsNull;
        RowCount = 1;
    }

    public bool ValueIsNull { get; }
    public long ContributingRowCount { get; }
    // Float64Value / Int64Value — read based on ResultType
}
```

`ToResult` mapping:

```csharp
AggregateKind.Sum when columnType == RainDbType.Float64 =>
    new AggregateQueryResult(RainDbType.Float64, FloatSum, 0L, ContributingRows,
        valueIsNull: ContributingRows == 0),
AggregateKind.Min when columnType == RainDbType.Float64 =>
    new AggregateQueryResult(RainDbType.Float64, HasMin ? FloatMin : 0d, 0L, ContributingRows,
        valueIsNull: !HasMin),
```

`ContributingRows == 0` after combine means all-null or all-filtered measure column — not the empty-table path (handled by `EmptyAggregate`).

## 9.17 SUM on Float64: scalar and AVX2 paths

```csharp
private static PartialAgg SumFloat64(
    IColumnChunk col,
    ReadOnlySpan<int> selectedRows,
    int selectedCount,
    ReadOnlySpan<byte> nb,
    ReadOnlySpan<byte> values,
    VectorizedScanExecutionOptions options)
{
    if (selectedCount == 0)
        return new PartialAgg(0, 0d, 0d, 0d, 0L, false, false, 0);

    if (selectedRows.IsEmpty && !col.HasNulls && options.UseAvx2DoubleSum)
    {
        var sum = AggregateIntrinsics.SumFloat64(values, allowAvx2: true);
        return new PartialAgg(selectedCount, sum, 0d, 0d, 0L, false, false, 0);
    }

    double s = 0;
    long contrib = 0;
    for (var i = 0; i < selectedCount; i++)
    {
        var r = Row(selectedRows, i);
        if (SelectionEvaluator.IsNull(nb, r, col.HasNulls))
            continue;
        var bits = BinaryPrimitives.ReadInt64LittleEndian(
            values.Slice(r * sizeof(double), sizeof(double)));
        s += BitConverter.Int64BitsToDouble(bits);
        contrib++;
    }
    return new PartialAgg(contrib, s, 0d, 0d, 0L, false, false, 0);
}
```

AVX2 activates when: no selection vector, no nulls, `UseAvx2DoubleSum == true`.

Integer sums (`SumInt32`, `SumInt64`) always use scalar loops with null skipping; `Int32` widens into `long` accumulator.

## 9.18 AggregateIntrinsics.SumFloat64

```csharp
// src/RainDB.Core/Columnar/AggregateIntrinsics.cs
public static double SumFloat64(ReadOnlySpan<byte> valuesLittleEndian, bool allowAvx2 = true)
{
    if (valuesLittleEndian.Length % sizeof(double) != 0)
        throw new ArgumentException("Length must be multiple of 8.", nameof(valuesLittleEndian));
    var doubles = MemoryMarshal.Cast<byte, double>(valuesLittleEndian);
    if (doubles.IsEmpty)
        return 0d;
    if (allowAvx2 && Avx2.IsSupported && doubles.Length >= Vector256<double>.Count)
        return SumDoubleAvx2(doubles);
    return SumDoubleScalar(doubles);
}
```

AVX2 kernel:

```csharp
private static unsafe double SumDoubleAvx2(ReadOnlySpan<double> values)
{
    fixed (double* p = values)
    {
        var n = values.Length;
        var i = 0;
        var acc = Vector256<double>.Zero;
        var limit = n - (n % Vector256<double>.Count);
        for (; i < limit; i += Vector256<double>.Count)
            acc = Avx2.Add(acc, Avx2.LoadVector256(p + i));

        var lo = acc.GetLower();
        var hi = acc.GetUpper();
        var s128 = lo + hi;
        var sum = s128.GetElement(0) + s128.GetElement(1);
        for (; i < n; i++)
            sum += p[i];
        return sum;
    }
}
```

`MemoryMarshal.Cast<byte, double>` reinterprets the column buffer without copy. FP addition order differs from scalar path — acceptable for analytics; document for financial determinism requirements.

`MinFloat64` / `MaxFloat64` exist in `AggregateIntrinsics` but global MIN/MAX use inline `MinMaxFloat64` in `PartialAgg`.

## 9.19 MIN and MAX on Float64

```csharp
private static PartialAgg MinMaxFloat64(
    AggregateKind kind,
    ReadOnlySpan<int> selectedRows,
    int selectedCount,
    bool hasNulls,
    ReadOnlySpan<byte> nb,
    ReadOnlySpan<byte> values)
{
    double? cur = null;
    long contrib = 0;
    for (var i = 0; i < selectedCount; i++)
    {
        var r = Row(selectedRows, i);
        if (SelectionEvaluator.IsNull(nb, r, hasNulls))
            continue;
        var bits = BinaryPrimitives.ReadInt64LittleEndian(
            values.Slice(r * sizeof(double), sizeof(double)));
        var v = BitConverter.Int64BitsToDouble(bits);
        cur = cur.HasValue
            ? kind == AggregateKind.Min ? Math.Min(cur.Value, v) : Math.Max(cur.Value, v)
            : v;
        contrib++;
    }

    if (!cur.HasValue)
        return new PartialAgg(0, 0d, 0d, 0d, 0L, false, false, 0);
    // ...
}
```

## 9.20 Morsel parallelism

`EffectiveDop` maps `MaxDegreeOfParallelism`: negative → `Environment.ProcessorCount`, zero → 1, positive → explicit cap.

Three scheduling modes mirror non-aggregate projection:

1. Sequential loop (`dop <= 1 || n == 1`)
2. `Parallel.For` over batch indices
3. `RunChannelMorselsAsync` — bounded channel work queue

Each worker writes to `partials[i]` — no locks. `PartialAgg` is a value type; array slots are independent.

## 9.21 Global vs grouped aggregates

| Aspect | `PartialAgg` (global) | `AggregateAccumulator` (hash, Ch. 10) |
|--------|------------------------|----------------------------------------|
| Scope | One per batch → one combined | One per group key per batch |
| Combine | `PartialAgg.Combine` | `AggregateRowOps.Combine` |
| COUNT field | `CountAgg` | `Count` |
| Output | `IAggregateQueryResult` | `FixedWidthColumnChunk` in batch |
| Operator | `VectorizedScanEngine` | `HashAggregateEngine` |

`ShouldEmitAggregateNull` in hash aggregation mirrors `ToResult` null rules for per-group output columns.

## 9.22 COUNT semantics reference

| Scenario | `COUNT(*)` | `COUNT(col)` | `SUM(col)` |
|----------|------------|--------------|------------|
| Empty table | 0, not null | 0, not null | NULL |
| All rows filtered | 0, not null | 0, not null | NULL |
| Rows exist, all col null | row count | 0, not null | NULL |
| Normal | row count | non-null count | sum |

Tests: `SqlGroupByTests`, `Phase1ReadPathTests` in `tests/RainDB.Tests/`.

## 9.23 End-to-end trace

Query: `SELECT SUM(amount) FROM sales WHERE region_id = 3`

Physical plan: `VectorizedScanPhysicalPlan` with filter on `region_id`, `Aggregate = (SourceColumnIndex: amount_ix, Kind: Sum)`.

```
ExecuteAsync
  → ComputeAggregateAsync
      batch 0: AccumulateAggregateBatch → PartialAgg(IntSum=1200, ContributingRows=40)
      batch 1: AccumulateAggregateBatch → PartialAgg(IntSum=800, ContributingRows=25)
      Combine → PartialAgg(IntSum=2000, ContributingRows=65)
      ToResult → AggregateQueryResult(Int64, int64Value=2000, valueIsNull=false)
```

## 9.24 Summary

Global aggregation in RainDB is a disciplined map-reduce over immutable columnar batches:

1. **Plan** — `VectorizedScanPhysicalPlan.Aggregate` selects kind and column via `AggregateSpec`.
2. **Map** — `AccumulateAggregateBatch` produces `PartialAgg` per batch, honoring filters and COUNT vs COUNT(col).
3. **Reduce** — `PartialAgg.Combine` folds partials associatively.
4. **Materialize** — `ToResult` or `EmptyAggregate` exposes SQL-correct scalars via `ValueIsNull`.
5. **Accelerate** — `AggregateIntrinsics.SumFloat64` optional AVX2 when column is dense and null-free.

The design keeps aggregation colocated with the scan operator that already owns filter evaluation and batch iteration — avoiding hash table overhead when the group count is exactly one.
