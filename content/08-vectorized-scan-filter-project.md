---
title: "Chapter 8: Vectorized Scan, Filter, and Project"
order: 8
---

# Chapter 8: Vectorized Scan, Filter, and Project

Selection (σ) and projection (π) show up in almost every analytical query. Columnar engines implement them as **scan** (pull batches from storage), **filter** (evaluate predicates), and **project** (copy output columns) — usually fused into one vectorized pipeline per batch. Parallel workers chew through **morsels** (independent batch units) while a fixed output slot per batch index keeps results deterministic.

**Part I** covers the algebra, selectivity, predicate pushdown, selection vectors, conjunctive predicates, short-circuit behavior, gather/scatter mechanics, and morsel-driven parallelism. **Part II** follows RainDB's `VectorizedScanEngine`, `SelectionEvaluator`, `FixedWidthSelectionKernels`, and `ProjectGather` through the source tree.

---

# Part I: Concepts and Theory

## 8.1 Selection (σ) and projection (π)

Relational algebra models query fragments as operations on relations.

### Selection

**Selection** σ_P(R) returns the subset of R where predicate P is true:

```
σ_{amount > 100}(Sales)
```

Key properties:

- **Arity unchanged** — same columns in and out
- **Row count non-increasing** — |σ_P(R)| ≤ |R|
- **Projection commute** — pushing π below σ is safe only when the predicate references columns you keep

SQL maps selection to `WHERE`.

### Projection

**Projection** π_C(R) keeps columns in C and drops the rest:

```
π_{region, amount}(Sales)
```

Key properties:

- **Column count may shrink** — row count does not grow (duplicate elimination aside)
- **Set vs bag** — relational π is a set; SQL retains duplicate rows unless `DISTINCT`

SQL maps projection to the `SELECT` column list (excluding aggregates).

### Composition

The usual pattern:

```
π_{region, amount}( σ_{amount > 100}( Sales ) )
```

Run **filter before project** when you can — copying fewer rows saves bandwidth downstream.

### Pseudocode: algebra to operators

```
function execute(query):
    rel = scan(table)
    rel = select(rel, predicate)   // σ
    rel = project(rel, columns)    // π
    return rel
```

RainDB implements this fusion in `ProcessOneBatch`: `SelectionEvaluator` first, then `ProjectGather`.

---

## 8.2 Selectivity and predicate pushdown

**Selectivity** measures how aggressively a predicate thins the input:

```
selectivity(P) = |σ_P(R)| / |R|
```

On uniform data across 10 countries, `country = 'US'` yields selectivity ≈ 0.1.

### Why selectivity matters

| Selectivity | Filter impact |
|-------------|---------------|
| High (0.9) | Little row reduction; scan dominates |
| Low (0.01) | Early shrink; cheap downstream operators |

Optimizers feed selectivity into join ordering, index choice, and broadcast-versus-shuffle decisions.

### Predicate pushdown

**Predicate pushdown** places filters as close to the data as possible — scan operator, storage layer, or remote worker — so upstream operators see fewer rows.

```
Without pushdown:
  Scan(all rows) → Join → Filter

With pushdown:
  Scan → Filter → Join   (only matching rows join)
```

Columnar systems commonly:

- Evaluate predicates **inside scan** on each batch
- Hand **filters to I/O** (zone maps, row-group min/max skip whole stripes)
- Skip Parquet row groups using **column statistics**

RainDB Phase 1 runs filters inside `VectorizedScanEngine` per batch — logical pushdown without storage-level zone maps.

### Conjunctive selectivity estimation

Assuming independent predicates P and Q:

```
selectivity(P AND Q) ≈ selectivity(P) × selectivity(Q)
```

Correlated columns break the product rule; production optimizers use histograms and multi-column stats. RainDB emits conjuncts in binder order and does not reorder them yet.

---

## 8.3 Selection vectors in columnar execution

Row stores evaluate predicates on assembled **row structs**. Columnar engines build **selection vectors** — dense lists of passing row indices.

### Dense index list

Filter a batch of n rows, then record survivors:

```
selectedRows = [0, 3, 7, 7, ...]   // length = selectedCount
selectedCount = k
```

Each downstream operator uses `selectedRows[o]` to index every column chunk at source row `r`.

### Versus bitmasks

| Representation | Pros | Cons |
|----------------|------|------|
| Bitmask (n bits) | Compact when k ≈ n | Often must convert to indices before gather |
| Dense indices (k ints) | Ready for gather loops | 4 bytes per surviving row |

RainDB rents a dense `int[]` from `ArrayPool<int>` and passes it straight into `ProjectGather`.

### Pseudocode

```
function filterBatch(batch, pred, dest: int[]):
    count = 0
    for r in 0 .. batch.rowCount-1:
        if pred(batch, r):
            dest[count++] = r
    return count
```

### Identity fast path

No filter means no index list — row `o` maps implicitly to source row `o`. RainDB sets `useRowSelection: false` when `plan.Filters` is empty.

---

## 8.4 Conjunctive predicates (CNF)

`WHERE` clauses usually chain predicates with **AND** — a **conjunction**. Full CNF is AND of OR clauses; the everyday case is a flat AND list:

```
WHERE amount > 100 AND region = 'US' AND active = true
```

### Sequential intersection algorithm

1. Run the first predicate → fill buffer, count = k₁
2. For each later predicate, **compact in place** — keep indices that still pass, count = k₂ ≤ k₁

```
indices = filter(col_amount, >100, buffer)     // k=5000
indices = intersect(col_region, ='US', buffer, k)  // k=500
indices = intersect(col_active, =true, buffer, k)  // k=450
```

In-place intersection avoids k-sized temporary bitmasks.

RainDB entry point: `SelectionEvaluator.FillSelectedRowsConjunctive`.

### Disjunction (OR)

`OR` needs set union or merged bitmasks — harder and not in RainDB Phase 1's `ColumnCompareFilter[]` AND focus.

### Reordering conjuncts

Classic optimizer advice: evaluate the cheapest or most selective predicate first to shrink work on later conjuncts. RainDB keeps textual order until a cost model arrives.

---

## 8.5 Short-circuit evaluation

**Short-circuit** logic stops as soon as the boolean result is determined.

### SQL three-valued logic

SQL distinguishes **TRUE, FALSE, UNKNOWN (NULL)**. `WHERE` retains only TRUE.

```
NULL AND FALSE  → FALSE
TRUE OR UNKNOWN → TRUE
```

For `P AND Q`, a FALSE `P` means `Q` can be skipped in row-at-a-time evaluation.

### Columnar batch evaluation

Vectorized engines typically evaluate **one predicate across all candidates** per step, not literal row-by-row short-circuit. **Conjunctive intersection** still mimics the benefit: each step runs on a shrinking index list.

Row-at-a-time:

```
if not P(row): continue
if not Q(row): continue
```

Columnar conjunctive:

```
candidates = all rows
candidates = filter P on candidates
candidates = filter Q on candidates   // |Q| = |candidates|, not |all|
```

RainDB's fill is **column-at-a-time**; shrinking `count` between conjuncts is the performance lever.

### NULL in comparisons

`NULL = 5` is UNKNOWN, so the row drops out of `WHERE`. RainDB's `SelectionEvaluator.IsNull` excludes null cells before comparison — they never land in the selection vector.

---

## 8.6 Gather and scatter in vectorized engines

Columnar layout stores values **per column**. Filtering yields sparse indices; projection must **gather** into dense output buffers.

### Gather

Copy selected elements into contiguous output:

```
for o in 0 .. selectedCount-1:
    r = selectedRows[o]
    outValues[o] = srcValues[r]
```

Fixed-width types: per-row copy or SIMD gather. UTF-8: copy payloads and rebuild offsets — costlier.

### Scatter

Write values to non-contiguous destinations — gather's inverse:

```
for o in 0 .. selectedCount-1:
    dest[selectedRows[o]] = values[o]
```

Hash builds and in-place filters use scatter. RainDB's scan path **gathers into fresh pooled chunks** so input batches stay read-only.

### Full-column copy fast path

When every row passes (`selectedCount == rowCount` without a selection vector), one bulk copy suffices:

```
outValues.copyFrom(srcValues)
```

RainDB: `ProjectGather.CopyEntireFixedWidthColumn`.

---

## 8.7 Morsel-driven parallelism

**Morsel-driven parallelism** (Leis et al., 2014) hands small table partitions (**morsels**) to worker threads for dynamic load balance.

### Static versus dynamic morsels

| Strategy | Behavior |
|----------|----------|
| Static | Pre-partition batch indices — simple, skew-prone |
| Dynamic queue | Workers dequeue next morsel — better balance |

RainDB defaults to **one batch per morsel**, optionally feeding batch indices through a **`Channel<int>`** when `UseChannelScheduler` is enabled.

### Deterministic output with parallel morsels

Never let parallel workers append to one shared list — order would depend on scheduling. **Fixed slots:**

```
output[batchIndex] = process(table.batches[batchIndex])
```

Workers may finish in any order; `output[i]` always corresponds to input batch `i`.

### Morsel granularity trade-off

| Morsel size | Pros | Cons |
|-------------|------|------|
| Small (1K rows) | Fine load balance | Scheduler overhead |
| Large (1M rows) | Low overhead | Skew on uneven batches |

RainDB morsel size equals **ingested batch size** — `VectorChunkLimits` sets the knob.

### Pipeline within morsel

Each morsel runs **filter → project** (or a partial aggregate) independently — embarrassingly parallel until a global combine step.

---

## 8.8 Integration with global aggregates

`SELECT SUM(x) FROM t WHERE p` filters inside each batch, accumulates **partial aggregates** per batch, then **combines in fixed order**:

```
partials[i] = sum(filter(batch[i], p))
total = sum(partials[0..n-1])   // ordered combine
```

`VectorizedScanEngine` shares the morsel scheduler between project and aggregate paths.

---

## 8.9 Summary: Part I checklist

1. **σ and π** — filter rows, subset columns; prefer filter before project.
2. **Selectivity** — drives cost models; push predicates toward scan.
3. **Selection vectors** — dense indices feeding gather loops.
4. **CNF / AND** — in-place intersection of index lists.
5. **Short-circuit** — shrink candidates between conjuncts; NULL rows excluded.
6. **Gather/scatter** — gather into new columns; bulk copy when unfiltered.
7. **Morsels** — parallel batches with index-stable output slots.

---

# Transition to Part II: RainDB Implementation

Phase 1 scan/filter/project/aggregate lives in **`VectorizedScanEngine`** over **`IColumnarTableSource.Batches`**. **`SelectionEvaluator`** builds conjunctive selection vectors. **`FixedWidthSelectionKernels`** vectorizes `Int32` equality. **`ProjectGather`** materializes **`PooledFixedWidthColumnChunk`** outputs. Parallelism uses **`Parallel.For`** or a bounded **`Channel<int>`** scheduler writing deterministic `outArr[i]` slots.

---

# Part II: RainDB Implementation

## 8.11 Entry point and control flow

```csharp
// src/RainDB.Query/Execution/VectorizedScanEngine.cs
public static class VectorizedScanEngine
{
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
}
```

**Two top-level paths:**

1. **Aggregate path** — `plan.Aggregate != null` → `ComputeAggregateAsync` → `IAggregateQueryResult`
2. **Project path** — no aggregate → `ProjectAllBatchesAsync` → `ColumnarMaterializedQueryResult`

Both paths share batch iteration, parallelism options, and filter evaluation via `SelectionEvaluator`.

---

## 8.12 ValidatePlan: fail fast before parallel work

```csharp
private static void ValidatePlan(VectorizedScanPhysicalPlan plan, IColumnarTableSource table)
{
    if (plan.TableId != table.Id)
        throw new ArgumentException("Physical plan table id does not match resolved table.", nameof(table));

    var colCount = table.Schema.Columns.Count;
    foreach (var idx in plan.OutputColumnIndices)
    {
        if ((uint)idx >= (uint)colCount)
            throw new ArgumentException($"Output column index {idx} is out of range.", nameof(plan));
    }

    if (plan.Filters is { } fa)
    {
        foreach (var f in fa)
        {
            if ((uint)f.ColumnIndex >= (uint)colCount)
                throw new ArgumentException("Filter column index is out of range.", nameof(plan));
        }
    }

    if (plan.Aggregate is { } a)
    {
        // bounds + supported AggregateKind/type pairs
    }
}
```

Unsupported type/kind pairs throw `NotSupportedException` before any parallel worker starts — avoiding partial results from mid-query failures.

---

## 8.13 ProjectAllBatchesAsync: morsel parallelism

```csharp
private static async ValueTask<IReadOnlyList<IColumnarBatch>> ProjectAllBatchesAsync(
    VectorizedScanPhysicalPlan plan,
    IColumnarTableSource table,
    IExecutionContext context)
{
    var batches = table.Batches;
    var n = batches.Count;
    if (n == 0)
        return Array.Empty<IColumnarBatch>();

    var dop = EffectiveDop(plan.Options.MaxDegreeOfParallelism);
    var outArr = new IColumnarBatch[n];
    var ct = context.CancellationToken;

    if (dop <= 1 || n == 1)
    {
        for (var i = 0; i < n; i++)
            outArr[i] = ProcessOneBatch(plan, batches[i], context);
    }
    else if (plan.Options.UseChannelScheduler)
    {
        await RunChannelMorselsAsync(n, dop,
            i => outArr[i] = ProcessOneBatch(plan, batches[i], context), ct)
            .ConfigureAwait(false);
    }
    else
    {
        Parallel.For(0, n,
            new ParallelOptions { MaxDegreeOfParallelism = dop, CancellationToken = ct },
            i => outArr[i] = ProcessOneBatch(plan, batches[i], context));
    }

    return outArr;
}
```

**Morsel model:** Each batch is an independent **morsel** — filter + project with no cross-batch state. Output batch `i` depends only on input batch `i`.

**Deterministic ordering:** `outArr[i]` assignment by index preserves table batch order in the result list even when parallel workers finish out of order.

**Empty table:** Returns `Array.Empty<IColumnarBatch>()` without allocating `outArr`.

---

## 8.14 Channel morsel scheduler

```csharp
private static async ValueTask RunChannelMorselsAsync(
    int batchCount,
    int dop,
    Action<int> body,
    CancellationToken cancellationToken)
{
    var ch = Channel.CreateBounded<int>(
        new BoundedChannelOptions(Math.Max(16, dop * 4))
        {
            SingleWriter = true,
            SingleReader = false,
            FullMode = BoundedChannelFullMode.Wait,
        });

    var workers = new Task[dop];
    for (var w = 0; w < dop; w++)
    {
        workers[w] = Task.Run(async () =>
        {
            while (await ch.Reader.WaitToReadAsync(cancellationToken).ConfigureAwait(false))
            {
                while (ch.Reader.TryRead(out var idx))
                    body(idx);
            }
        }, cancellationToken);
    }

    for (var i = 0; i < batchCount; i++)
        await ch.Writer.WriteAsync(i, cancellationToken).ConfigureAwait(false);

    ch.Writer.Complete();
    await Task.WhenAll(workers).ConfigureAwait(false);
}
```

| Option | Value | Rationale |
|--------|-------|-----------|
| Capacity | `max(16, dop * 4)` | Back-pressure |
| `SingleWriter` / multi-reader | | Workers compete for batch indices |

**vs `Parallel.For`:** Channels integrate cleanly with async cancellation; `Parallel.For` has lower overhead for uniform CPU-bound loops.

The aggregate path (`ComputeAggregateAsync`) reuses the same scheduler with `partials[i] = AccumulateAggregateBatch(...)`.

---

## 8.15 ProcessOneBatch: filter then project

```csharp
private static IColumnarBatch ProcessOneBatch(
    VectorizedScanPhysicalPlan plan,
    IColumnarBatch batch,
    IExecutionContext context)
{
    context.CancellationToken.ThrowIfCancellationRequested();
    var rent = ArrayPool<int>.Shared.Rent(batch.RowCount);
    try
    {
        var span = rent.AsSpan(0, batch.RowCount);
        int selected;
        var hasFilters = plan.Filters is { Length: > 0 };
        if (hasFilters)
            selected = SelectionEvaluator.FillSelectedRowsConjunctive(
                batch, plan.Filters!, span);
        else
            selected = batch.RowCount;

        return ProjectGather.Project(
            batch,
            plan.OutputColumnIndices.AsSpan(),
            useRowSelection: hasFilters,
            selectedRows: hasFilters ? span[..selected] : ReadOnlySpan<int>.Empty,
            selectedCount: selected,
            context.BufferPool,
            context.AlignedBufferPool);
    }
    finally
    {
        ArrayPool<int>.Shared.Return(rent);
    }
}
```

**`useRowSelection` semantics:**

- `true` when any filter present — `ProjectGather` reads `selectedRows[o]` to find source row
- `false` when no filters — `selectedCount == batch.RowCount`, gather uses identity mapping

**Note:** When filters exist but select all rows, `selected == batch.RowCount` and `useRowSelection` is still `true` — gather uses the explicit index list, not the identity fast path inside `GatherFixedWidth`.

---

## 8.16 SelectionEvaluator overview

```csharp
// src/RainDB.Query/Vectorized/SelectionEvaluator.cs
/// <summary>
/// Builds dense selection vectors (matching row indices) for filters.
/// Fixed-width predicates use column-wise compare kernels; conjunctive AND
/// intersects successive selections in-place.
/// </summary>
internal static class SelectionEvaluator
```

### Null bitmap helper

```csharp
internal static bool IsNull(ReadOnlySpan<byte> nullBitmap, int row, bool hasNulls)
{
    if (!hasNulls)
        return false;
    return (nullBitmap[row >> 3] & (1 << (row & 7))) != 0;
}
```

Bit `1` means SQL NULL (RainDB column chunk convention).

### Conjunctive AND: FillSelectedRowsConjunctive

```csharp
internal static int FillSelectedRowsConjunctive(
    IColumnarBatch batch,
    ReadOnlySpan<ColumnCompareFilter> filters,
    Span<int> dest)
{
    var n = batch.RowCount;
    if (filters.Length == 0)
    {
        for (var i = 0; i < n; i++)
            dest[i] = i;
        return n;
    }

    if (dest.Length < n)
        throw new ArgumentException("Selection buffer too small.", nameof(dest));

    var count = FillSelectedRows(batch.Columns[filters[0].ColumnIndex], filters[0], dest);
    for (var f = 1; f < filters.Length; f++)
    {
        var col = batch.Columns[filters[f].ColumnIndex];
        count = filters[f].Utf8LiteralBytes is not null || col.PhysicalType == RainDbType.Utf8
            ? IntersectUtf8(col, filters[f], dest, count)
            : FixedWidthSelectionKernels.IntersectSelectedIndices(
                col, filters[f], dest, count);
    }

    return count;
}
```

**Algorithm:**

1. First predicate: fill `dest[0..count)` with matching row indices
2. Each subsequent predicate: **compact in-place** — survivors only

UTF-8 intersection uses scalar `RowMatchesFilter` per candidate row.

### Single predicate: FillSelectedRows

```csharp
internal static int FillSelectedRows(IColumnChunk column, ColumnCompareFilter filter, Span<int> dest)
{
    if (filter.Utf8LiteralBytes is not null)
    {
        if (column.PhysicalType != RainDbType.Utf8)
            throw new ArgumentException("UTF-8 literal filter requires a UTF-8 column.");
        return FillUtf8Selected(column, filter, dest);
    }

    if (column.PhysicalType == RainDbType.Utf8)
        throw new NotSupportedException("UTF-8 column requires a string literal predicate.");

    return FixedWidthSelectionKernels.FillSelectedIndices(column, filter, dest);
}
```

### RowMatchesFilter (scalar fallback)

Used for UTF-8 and intersection fallback. Supports `Int32`, `Int64`, `Float64`, `Boolean` with full comparison operator set. NULL cells never match equality/range filters in `WHERE`.

### UTF-8 equality

```csharp
private static bool Utf8PayloadEquals(IColumnChunk column, int row, ReadOnlySpan<byte> literal)
{
    return column switch
    {
        Utf8ColumnChunk utf8 => Utf8ArrowRowEquals(utf8, row, literal),
        Utf8LengthPrefixedColumnChunk lp => lp.GetPayloadSpan(row).SequenceEqual(literal),
        _ => throw new NotSupportedException(...),
    };
}
```

**Limitations:** Only `=`, `!=`, `<>` with single-quoted literals. No `LIKE`, range, or collation.

---

## 8.17 FixedWidthSelectionKernels: SIMD Int32 equality

```csharp
// src/RainDB.Query/Vectorized/FixedWidthSelectionKernels.cs
internal static class FixedWidthSelectionKernels
{
    internal static int FillSelectedIndices(
        IColumnChunk column, ColumnCompareFilter filter, Span<int> dest)
    {
        // dispatch by PhysicalType → FillInt32 / FillInt64 / FillFloat64 / FillBool
    }
}
```

### Int32 fill with vectorized equality fast path

```csharp
private static int FillInt32(
    ReadOnlySpan<byte> values,
    ReadOnlySpan<byte> nb,
    bool hasNulls,
    ScalarCompareOp op,
    int imm,
    Span<int> dest)
{
    var ints = MemoryMarshal.Cast<byte, int>(values);
    if (!hasNulls && op == ScalarCompareOp.Eq && Vector128.IsHardwareAccelerated)
        return FillInt32EqVectorized(ints, imm, dest);

    var count = 0;
    for (var i = 0; i < ints.Length; i++)
    {
        if (SelectionEvaluator.IsNull(nb, i, hasNulls))
            continue;
        if (CompareInt32(ints[i], imm, op))
            dest[count++] = i;
    }
    return count;
}
```

**Fast path preconditions:**

- No nulls (`!hasNulls`)
- Equality operator (`ScalarCompareOp.Eq`)
- `Vector128.IsHardwareAccelerated`

### FillInt32EqVectorized

```csharp
private static int FillInt32EqVectorized(ReadOnlySpan<int> values, int imm, Span<int> dest)
{
    var count = 0;
    var i = 0;
    var vImm = Vector128.Create(imm);
    ref var start = ref MemoryMarshal.GetReference(values);
    var limit = values.Length - (values.Length % Vector128<int>.Count);

    for (; i < limit; i += Vector128<int>.Count)
    {
        var block = Vector128.LoadUnsafe(ref Unsafe.Add(ref start, i));
        var eq = Vector128.Equals(block, vImm);
        for (var j = 0; j < Vector128<int>.Count; j++)
        {
            if (eq.GetElement(j) != 0)
                dest[count++] = i + j;
        }
    }

    for (; i < values.Length; i++)
    {
        if (values[i] == imm)
            dest[count++] = i;
    }

    return count;
}
```

**Step-by-step:** cast values to `ReadOnlySpan<int>`, broadcast immediate with `Vector128.Create`, compare in `Vector128<int>.Count` lanes via `LoadUnsafe`/`Equals`, extract set lanes to indices, scalar tail for remainder.

### IntersectSelectedIndices for Int32

```csharp
private static int IntersectInt32(
    ReadOnlySpan<byte> values,
    ReadOnlySpan<byte> nb,
    bool hasNulls,
    ScalarCompareOp op,
    int imm,
    Span<int> selectedRows,
    int count)
{
    var ints = MemoryMarshal.Cast<byte, int>(values);
    var write = 0;
    for (var r = 0; r < count; r++)
    {
        var i = selectedRows[r];
        if (SelectionEvaluator.IsNull(nb, i, hasNulls))
            continue;
        if (CompareInt32(ints[i], imm, op))
            selectedRows[write++] = i;
    }
    return write;
}
```

Second and later predicates touch **candidate rows only** — critical when the first predicate is highly selective. Intersect path does not currently use SIMD — candidate count may be small.

### Other fixed-width types

`Int64`, `Float64`, and `Boolean` use scalar fill and intersect loops. Extension point: `FillInt64EqVectorized` when profiling warrants.

---

## 8.18 ProjectGather: projection after selection

```csharp
// src/RainDB.Query/Vectorized/ProjectGather.cs
internal static class ProjectGather
{
    internal static ColumnarBatch Project(...)
    {
        var cols = new IColumnChunk[outputColumnIndices.Length];
        for (var c = 0; c < outputColumnIndices.Length; c++)
        {
            cols[c] = GatherColumn(
                batch.Columns[outputColumnIndices[c]],
                useRowSelection, selectedRows, selectedCount,
                bufferPool, alignedBufferPool);
        }
        return new ColumnarBatch(selectedCount, cols);
    }
}
```

Output row count is `selectedCount`, not `batch.RowCount`.

### Gather routing

```csharp
private static IColumnChunk GatherColumn(...)
{
    if (source.PhysicalType == RainDbType.Utf8)
    {
        if (source is Utf8ColumnChunk utf8)
            return GatherUtf8Arrow(utf8, ...);
        if (source is Utf8LengthPrefixedColumnChunk lp)
            return GatherUtf8LengthPrefixed(lp, ...);
        throw new NotSupportedException("Unknown UTF-8 chunk implementation.");
    }
    return GatherFixedWidth(source, ..., bufferPool, alignedBufferPool);
}
```

### Fixed-width gather paths

**Path A — identity / no filter (`CopyEntireFixedWidthColumn`):**

When `!useRowSelection && selectedCount == source.RowCount`, bulk `CopyTo` of values and null bitmap.

**Path B — selective gather without nulls:**

```csharp
for (var o = 0; o < selectedCount; o++)
{
    var r = RowAt(useRowSelection, selectedRows, o);
    srcValues.Slice(r * w, w).CopyTo(outValues.Slice(o * w, w));
}
```

**Path C — selective gather with nulls:** per output row, set null bit or copy value bytes.

```csharp
private static int RowAt(bool useRowSelection, ReadOnlySpan<int> selectedRows, int o) =>
    useRowSelection ? selectedRows[o] : o;
```

### UTF-8 gather paths

**Arrow (`GatherUtf8Arrow`):** rebuild offsets + blob. **Length-prefixed:** write length + payload per row. UTF-8 gathers use managed allocations (`List<byte>`, `ToArray`) — not pooled in Phase 1.

---

## 8.19 Global aggregate path

When `plan.Aggregate` is set, `ComputeAggregateAsync`:

1. Builds `PartialAgg[n]` per batch (same morsel parallelism)
2. Applies filters via `SelectionEvaluator`
3. Combines partials in **batch index order**
4. Returns `AggregateQueryResult`

`COUNT(*)` counts selected rows only. `COUNT(col)` skips nulls. Unfiltered non-null `Float64` `SUM` may use `UseAvx2DoubleSum` + `AggregateIntrinsics.SumFloat64`.

```csharp
var combined = partials[0];
for (var i = 1; i < n; i++)
    combined = PartialAgg.Combine(combined, partials[i], spec.Kind);
```

Ordered combine preserves deterministic floating-point aggregation semantics versus arbitrary thread reduction order.

---

## 8.20 Performance characteristics

| Stage | Dominant cost | Optimizations |
|-------|---------------|---------------|
| Filter (Int32 =) | Memory bandwidth | SIMD `Vector128.Equals` |
| Filter (other ops) | Per-row branch | Selective intersection on AND chain |
| Filter (UTF-8) | Byte compare | Scalar only |
| Project fixed-width | memcpy | Full-column copy fast path |
| Project UTF-8 | Alloc + copy | Not pooled |
| Parallelism | CPU cores | Morsel per batch, deterministic merge |

`VectorizedSelectionPerformanceTests` benchmarks selection kernels.

---

## 8.21 Correctness and integration

**Edge cases:**

- Empty or fully filtered batches → zero-row output chunks
- NULL cells never match equality predicates
- Filters present but select every row → `useRowSelection` still true
- Disposed `PooledFixedWidthColumnChunk` throws on property access

**Pipeline:**

```
SqlCompiler → VectorizedScanPhysicalPlan
DefaultQueryExecutor → catalog resolve → ValidatePlan
Per batch: SelectionEvaluator → ProjectGather
ColumnarMaterializedQueryResult (DisposeAsync returns pools)
```

---

## 8.22 Quick reference

| Component | Path |
|-----------|------|
| `VectorizedScanEngine` | `src/RainDB.Query/Execution/VectorizedScanEngine.cs` |
| `VectorizedScanPhysicalPlan` | `src/RainDB.Query/Plans/VectorizedScanPhysicalPlan.cs` |
| `SelectionEvaluator` | `src/RainDB.Query/Vectorized/SelectionEvaluator.cs` |
| `FixedWidthSelectionKernels` | `src/RainDB.Query/Vectorized/FixedWidthSelectionKernels.cs` |
| `ProjectGather` | `src/RainDB.Query/Vectorized/ProjectGather.cs` |
| `ColumnarMaterializedQueryResult` | `src/RainDB.Query/Results/ColumnarAndAggregateResults.cs` |
| Performance tests | `tests/RainDB.Tests/VectorizedSelectionPerformanceTests.cs` |

The vectorized scan path is the foundation for RainDB's analytical read performance: batches in, selections in the middle, pooled columnar batches out — with optional parallel morsels and SIMD where the type system and nullability allow.
