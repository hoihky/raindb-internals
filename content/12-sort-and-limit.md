---
title: "Chapter 12: Sort and Limit"
order: 12
---

# Chapter 12: Sort and Limit

`ORDER BY` and `LIMIT` in RainDB are handled by `SortTopNEngine` — an in-memory operator that collects row locations from columnar batches, optionally sorts them with a multi-column comparer, truncates to a limit, and gathers output columns into a single result batch.

This chapter is split into two parts. **Part I** covers sorting theory in query engines: external vs in-memory sort, stable vs unstable algorithms, ORDER BY semantics, Top-K algorithms, sort vs hash for grouping, and sort as a physical operator. **Part II** traces RainDB's `SortTopNPhysicalPlan`, `ExecuteTableAsync`, `ExecuteJoinAsync`, `RowLoc`, and gather materialization.

---

# Part I: Theory

## 12.1 Sorting in query engines

Sort shows up everywhere once you leave pure scan-filter-project:

- `ORDER BY` in user queries
- Sort-aggregation as a `GROUP BY` alternative
- Sort-merge join
- Window functions (`ROW_NUMBER`, `RANK`) — partition, then order
- `DISTINCT` via sort + adjacent dedup
- Set ops with duplicate removal

It is often the most expensive operator in an analytical plan because it touches every row and compares wide keys repeatedly.

### In-memory sort

When the working set fits the operator memory budget (`work_mem`, buffer pool):

1. Collect row handles (or keys + row ids)
2. Run a comparison sort — typically O(n log n)
3. Emit sorted output or materialize a batch

**Introsort** — what .NET `Array.Sort` uses — blends quicksort, heapsort fallback, and insertion sort on tiny partitions for O(n log n) worst case.

`SortTopNEngine` sorts a `RowLoc[]` with `RowLocComparer` entirely in memory.

### External sort (disk-backed)

When n exceeds RAM:

**Phase 1 — run generation**

- Read memory-sized chunks
- Sort each chunk in RAM
- Write sorted **runs** to temp files

**Phase 2 — merge**

- K-way merge with a heap of size k (fan-in)
- Stream globally sorted output

Still O(n log n) comparisons; I/O stays O(n) if merge width is tuned. **Replacement selection** can lengthen initial runs beyond one memory chunk, cutting merge passes.

RainDB Phase 1 has **no external sort** — everything must fit in process memory. `ISpillWriter` or mmap temp files are the likely extension points.

### Sort keys vs row payloads

Compare skinny records, not whole rows:

```
sort key: (salary DESC, name ASC)
payload:  (batch_idx, row_idx)   ← RainDB RowLoc
```

Sort permutes indices; a later **gather** reads column values. Wide VARCHAR columns never move during comparisons.

## 12.2 Stable vs unstable sort

**Stable** means equal keys keep their original relative order.

| Property | Stable sort | Unstable sort |
|----------|-------------|---------------|
| Equal keys | original order preserved | order undefined |
| Implementation | merge sort, timsort | quicksort (typical) |
| Use case | implicit tie-break via input order | raw speed |

SQL does not mandate tie order for `ORDER BY k` alone — ties are implementation-defined unless you add more keys.

.NET `Array.Sort` with `IComparer<T>` is **not stable** for reference types. RainDB does not depend on stability.

For deterministic ties, list explicit secondary keys:

```sql
ORDER BY salary DESC, employee_id ASC
```

## 12.3 ORDER BY semantics

`ORDER BY` defines a sort order over result rows — almost a total order once you fix NULL placement.

### Multi-column ordering

Lexicographic: compare column 1; on tie, compare column 2; repeat.

```
ORDER BY a ASC, b DESC
```

Sort by `a` ascending; within equal `a`, sort by `b` descending.

### NULL placement

SQL:2003 adds `NULLS FIRST` / `NULLS LAST` per key. Defaults differ:

| DBMS | Default NULL order ASC |
|------|-------------------------|
| PostgreSQL | NULLS LAST |
| Oracle | NULLS LAST |
| SQL Server | NULLs first (clustered index legacy) |

RainDB `RowLocComparer` puts **NULLs first** on ascending keys (`na` returns `-1` when only the left side is null). `DESC` negates the final comparison, so nulls tend to sort last on descending keys.

`NULLS FIRST` / `NULLS LAST` syntax is not exposed yet.

### Expressions and aliases

Full SQL allows sorting expressions and SELECT aliases. RainDB binds `SortKeyPhysicalSpec` to concrete column indices — expression keys need an upstream projection node.

### ORDER BY without LIMIT

Legal and common. Cost is O(n log n) compare work plus O(n) gather. RainDB pays full sort even when `LIMIT` is tiny — no Top-K shortcut yet.

## 12.4 Top-K algorithms

`LIMIT K` after `ORDER BY` only needs the best K rows, not a full ranking.

### Naive approach

Sort all n rows, slice the first K — O(n log n). RainDB does this today.

### Heap-based Top-K

Keep a size-K heap (min-heap for "top K largest", max-heap for "top K smallest"):

```
for each row r:
    if heap.size < K: push r
    else if r > heap.min(): pop min, push r
```

- Time: O(n log K)
- Space: O(K)

When K ≪ n (`LIMIT 10` on a billion rows), heaps win by a lot.

### Quickselect

Partition like quicksort but recurse only toward the K-th element — O(n) average, O(n²) worst.

Some engines use this for `LIMIT` without `ORDER BY` (any K rows).

### Partial sort (nth_element)

`std::partial_sort` — only the first K positions are fully ordered — O(n log K).

### Top-K with ORDER BY on multiple columns

The heap entry must carry the full sort key tuple; the comparer matches `RowLocComparer`.

### Distributed Top-K

Each shard returns local top-K; the coordinator merges with care — global bounds matter when keys are partitioned.

Future RainDB work: heap path when `plan.Limit` is set and `Limit << row_count`.

## 12.5 Sort vs hash for grouping

`GROUP BY` has two classic implementations:

| Aspect | Hash aggregation | Sort aggregation |
|--------|------------------|------------------|
| Passes | 1 over data | Sort + 1 scan of sorted data |
| Memory | O(groups) | O(n) for sort |
| Output order | Unordered unless extra sort | Sorted by group key |
| Spill | Grace hash partition | External sort runs |

**Sort aggregation:**

1. Sort rows by group key
2. Scan sorted array; accumulate within each equal-key **run**
3. Emit one row per run

Wins when data arrives pre-sorted or hash memory is impossible but external sort is OK. Loses when groups are numerous and random — hash often needs only one pass.

RainDB hashes (Chapter 10), then sorts keys for deterministic output — not sort-aggregation as the primary path.

### When sort beats hash for GROUP BY

- Hash table cannot fit, but sort can spill
- Input already ordered on group keys
- You need grouped output in key order without a second sort

## 12.6 Sort as physical operator

Volcano/Cascades treat **Sort** as a unary operator:

```
Sort(keys, limit?)
  └── child operator
```

Notable properties:

- **Blocking** — usually must drain the child before emitting row 1 (unless incremental sort applies)
- **Interesting order** — elide sort when the child already delivers the required key order
- **Limit pushdown** — Top-N can use a heap instead of full sort

RainDB fuses sort and limit in `SortTopNPhysicalPlan` and `JoinSortTopNPhysicalPlan` rather than chaining separate `Sort` and `Limit` nodes.

### Interesting orders

If the child is sorted on a prefix of the `ORDER BY` keys, **incremental sort** only sorts within equal-prefix groups.

Example: data clustered on `region`; query `ORDER BY region, revenue` — sort inside each region block only.

RainDB does not detect interesting orders; it always full-sorts when keys are present.

### Sort + join interaction

`JoinSortTopNPhysicalPlan` runs:

```
Join → materialize all matches → Sort → Limit
```

`LIMIT` does not prune the join — every match is built first. **Top-K join** pushdown is an obvious future optimization.

---

# Part II: RainDB Implementation

## 12.7 Physical plans

### SortTopNPhysicalPlan

```csharp
// src/RainDB.Query/Plans/SortTopNPhysicalPlan.cs
public sealed class SortTopNPhysicalPlan : IPhysicalPlan
{
    public TableId TableId { get; }
    public int[] OutputColumnIndices { get; }
    public ColumnCompareFilter[]? Filters { get; }
    public SortKeyPhysicalSpec[] SortKeys { get; }
    public int? Limit { get; }
    public VectorizedScanExecutionOptions Options { get; }
}
```

Constructor rejects `limit < 1` when specified.

`Explain()` example:

```
SortTopN(table=...) PROJECT[0,2] KEYS[2D,0A] FILTER[col1>5] LIMIT(100)
```

### JoinSortTopNPhysicalPlan

```csharp
// src/RainDB.Query/Plans/JoinSortTopNPhysicalPlan.cs
public sealed class JoinSortTopNPhysicalPlan : IPhysicalPlan
{
    public JoinPhysicalPlan Join { get; }
    public SortKeyPhysicalSpec[] SortKeys { get; }
    public int? Limit { get; }
}
```

### SortKeyPhysicalSpec

```csharp
public readonly record struct SortKeyPhysicalSpec(int ColumnIndex, bool Descending);
```

When `SortKeys` is empty, rows keep collection order (batch index, then row index) — `LIMIT` without `ORDER BY`.

## 12.8 ExecuteTableAsync

```csharp
// src/RainDB.Query/Execution/SortTopNEngine.cs
public static ValueTask<IQueryResult> ExecuteTableAsync(
    SortTopNPhysicalPlan plan,
    IColumnarTableSource table,
    IExecutionContext context)
{
    ArgumentNullException.ThrowIfNull(plan);
    ArgumentNullException.ThrowIfNull(table);
    ArgumentNullException.ThrowIfNull(context);
    if (plan.TableId != table.Id)
        throw new ArgumentException("SortTopN plan table id does not match source.", nameof(table));
    ValidateSortKeys(table.Schema, plan.SortKeys);
    ValidateOutputAndFilters(table.Schema, plan.OutputColumnIndices, plan.Filters);

    var batches = table.Batches;
    var ct = context.CancellationToken;
    var rows = CollectFilteredRows(batches, plan.Filters, ct);
    if (plan.SortKeys.Length > 0)
        Array.Sort(rows, new RowLocComparer(table.Schema, plan.SortKeys, batches));

    var take = plan.Limit is { } lim ? Math.Min(lim, rows.Length) : rows.Length;
    var batch = MaterializeRows(batches, table.Schema, rows.AsSpan(0, take), plan.OutputColumnIndices);
    return new ValueTask<IQueryResult>(new ColumnarMaterializedQueryResult([batch]));
}
```

Pipeline:

1. **Collect** — `RowLoc[]` with optional filters
2. **Sort** — `Array.Sort` if sort keys present
3. **Limit** — slice first `take` elements
4. **Materialize** — gather into one `ColumnarBatch`

`DefaultQueryExecutor` dispatches `SortTopNPhysicalPlan` to this method.

## 12.9 ExecuteJoinAsync

```csharp
public static async ValueTask<IQueryResult> ExecuteJoinAsync(
    JoinSortTopNPhysicalPlan plan,
    IColumnarTableSource probeTable,
    IColumnarTableSource buildTable,
    IExecutionContext context)
{
    var joinRes = await JoinExecutionEngine.ExecuteAsync(
        plan.Join, probeTable, buildTable, context).ConfigureAwait(false);
    if (joinRes is not IColumnarQueryResult col)
        throw new InvalidOperationException("Join must return columnar result.");
    var batches = col.Batches;
    var schema = plan.Join.OutputSchema;
    ValidateSortKeys(schema, plan.SortKeys);
    var rows = CollectAllRows(batches, context.CancellationToken);
    if (plan.SortKeys.Length > 0)
        Array.Sort(rows, new RowLocComparer(schema, plan.SortKeys, batches));

    var take = plan.Limit is { } lim ? Math.Min(lim, rows.Length) : rows.Length;
    var outIx = new int[schema.Columns.Count];
    for (var i = 0; i < outIx.Length; i++)
        outIx[i] = i;
    var batch = MaterializeRows(batches, schema, rows.AsSpan(0, take), outIx);
    return new ColumnarMaterializedQueryResult([batch]);
}
```

Differences from table path:

- Join executes first — full materialization
- No sort-engine filters (join side filters already applied)
- All join output columns projected (`outIx[i] = i`)

Performance: join cost dominates when match count is large; `LIMIT` does not reduce join work.

## 12.10 RowLoc: indirect row addressing

```csharp
private readonly struct RowLoc
{
    public RowLoc(int batchIdx, int rowIdx)
    {
        BatchIdx = batchIdx;
        RowIdx = rowIdx;
    }

    public int BatchIdx { get; }
    public int RowIdx { get; }
}
```

Sorting permutes `RowLoc` values without moving column data — **sort indirect indices** pattern.

After sort, `MaterializeRows` reads `batches[loc.BatchIdx].Columns[col][loc.RowIdx]` — one gather per output column.

Comparison to join's `RowRefMatch`: `RowLoc` addresses single table; `RowRefMatch` pairs left and right.

## 12.11 CollectFilteredRows

```csharp
private static RowLoc[] CollectFilteredRows(
    IReadOnlyList<IColumnarBatch> batches,
    ColumnCompareFilter[]? filters,
    CancellationToken ct)
{
    var rent = ArrayPool<int>.Shared;
    var total = 0;
    foreach (var b in batches)
        total += b.RowCount;
    var tmp = rent.Rent(Math.Max(total, 16));
    try
    {
        var list = new List<RowLoc>(total);
        for (var bi = 0; bi < batches.Count; bi++)
        {
            ct.ThrowIfCancellationRequested();
            var batch = batches[bi];
            if (filters is { Length: > 0 } fa)
            {
                var k = SelectionEvaluator.FillSelectedRowsConjunctive(
                    batch, fa, tmp.AsSpan(0, batch.RowCount));
                for (var i = 0; i < k; i++)
                    list.Add(new RowLoc(bi, tmp[i]));
            }
            else
            {
                for (var r = 0; r < batch.RowCount; r++)
                    list.Add(new RowLoc(bi, r));
            }
        }
        return list.ToArray();
    }
    finally
    {
        rent.Return(tmp);
    }
}
```

**Capacity hint:** `List<RowLoc>(total)` pre-allocates when unfiltered.

**Filter path:** rented `int[]` selection buffer per batch — same as scan/hash aggregate.

**Dense path:** `r = 0 .. RowCount-1` without selection buffer.

**Pre-sort order:** increasing batch index, then row index.

### CollectAllRows

```csharp
private static RowLoc[] CollectAllRows(IReadOnlyList<IColumnarBatch> batches, CancellationToken ct)
{
    var list = new List<RowLoc>();
    for (var bi = 0; bi < batches.Count; bi++)
    {
        ct.ThrowIfCancellationRequested();
        var batch = batches[bi];
        for (var r = 0; r < batch.RowCount; r++)
            list.Add(new RowLoc(bi, r));
    }
    return list.ToArray();
}
```

Join output typically one batch; code handles multiple.

## 12.12 RowLocComparer

```csharp
private sealed class RowLocComparer : IComparer<RowLoc>
{
    public int Compare(RowLoc x, RowLoc y)
    {
        foreach (var spec in _keys)
        {
            var c = CompareAtColumn(spec.ColumnIndex, x, y);
            if (c != 0)
                return spec.Descending ? -c : c;
        }
        return 0;
    }
}
```

Lexicographic multi-column compare with per-key `Descending` flip.

### NULL ordering

```csharp
private int CompareAtColumn(int colIx, RowLoc a, RowLoc b)
{
    var na = colA.HasNulls && SelectionEvaluator.IsNull(colA.NullBitmap.Span, a.RowIdx, true);
    var nb = colB.HasNulls && SelectionEvaluator.IsNull(colB.NullBitmap.Span, b.RowIdx, true);
    if (na && nb) return 0;
    if (na) return -1;
    if (nb) return 1;

    return t switch
    {
        RainDbType.Utf8 => CompareUtf8(colA, a.RowIdx, colB, b.RowIdx),
        RainDbType.Int32 => ReadI32(colA, a.RowIdx).CompareTo(ReadI32(colB, b.RowIdx)),
        RainDbType.Int64 => ReadI64(colA, a.RowIdx).CompareTo(ReadI64(colB, b.RowIdx)),
        RainDbType.Float64 => ReadF64(colA, a.RowIdx).CompareTo(ReadF64(colB, b.RowIdx)),
        RainDbType.Boolean => ReadBool(colA, a.RowIdx).CompareTo(ReadBool(colB, b.RowIdx)),
        _ => 0,
    };
}
```

NULLs first in ASC; DESC inverts final non-zero comparison.

### UTF-8 comparison

```csharp
private static int CompareUtf8(IColumnChunk ca, int ra, IColumnChunk cb, int rb)
{
    ReadOnlySpan<byte> sa = ca switch
    {
        Utf8ColumnChunk u => u.Values.Span[u.Offsets.Span[ra]..u.Offsets.Span[ra + 1]],
        Utf8LengthPrefixedColumnChunk lp => lp.GetPayloadSpan(ra),
        _ => throw new InvalidOperationException(),
    };
    // symmetric for sb
    return sa.SequenceCompareTo(sb);
}
```

Byte-wise lexicographic (UTF-8 code unit order, not Unicode collation). Consistent with join key equality.

### Fixed-width reads

Little-endian `BinaryPrimitives` / `BitConverter` for `Float64` — matches storage layout elsewhere.

## 12.13 LIMIT semantics

```csharp
var take = plan.Limit is { } lim ? Math.Min(lim, rows.Length) : rows.Length;
var batch = MaterializeRows(batches, table.Schema, rows.AsSpan(0, take), plan.OutputColumnIndices);
```

| `Limit` | Behavior |
|---------|----------|
| `null` | All qualifying rows (after sort) |
| `N ≥ 1` | First N in sort order |
| `N > rows.Length` | All rows (`Math.Min`) |

`LIMIT` without `ORDER BY` — arbitrary prefix in batch/row collection order.

**No `OFFSET`** in physical plan yet.

Constructor throws if `limit < 1` when provided.

## 12.14 MaterializeRows

```csharp
private static ColumnarBatch MaterializeRows(
    IReadOnlyList<IColumnarBatch> batches,
    TableSchema schema,
    ReadOnlySpan<RowLoc> rows,
    ReadOnlySpan<int> outputColumnIndices)
{
    var n = rows.Length;
    var cols = new IColumnChunk[outputColumnIndices.Length];
    for (var c = 0; c < outputColumnIndices.Length; c++)
    {
        var colIx = outputColumnIndices[c];
        var t = schema.Columns[colIx].Type;
        cols[c] = t == RainDbType.Utf8
            ? GatherUtf8Column(batches, colIx, rows)
            : GatherFixedWidthColumn(batches, colIx, t, rows);
    }
    return new ColumnarBatch(n, cols);
}
```

Output always **one batch** — consolidation regardless of input batch count.

Supports `SELECT name FROM t ORDER BY salary` — `OutputColumnIndices` may differ from sort keys.

## 12.15 GatherFixedWidthColumn

```csharp
private static IColumnChunk GatherFixedWidthColumn(
    IReadOnlyList<IColumnarBatch> batches,
    int colIx,
    RainDbType type,
    ReadOnlySpan<RowLoc> rows)
{
    var w = ColumnTypeSizes.FixedWidthBytes(type);
    var values = new byte[checked(rows.Length * w)];
    var nbBytes = ColumnTypeSizes.NullBitmapBytes(rows.Length);
    var nb = nbBytes > 0 ? new byte[nbBytes] : Array.Empty<byte>();
    var anyNull = false;
    for (var o = 0; o < rows.Length; o++)
    {
        var loc = rows[o];
        var col = batches[loc.BatchIdx].Columns[colIx];
        var r = loc.RowIdx;
        if (SelectionEvaluator.IsNull(srcNb, r, col.HasNulls))
        {
            anyNull = true;
            SetNull(nb, o);
            continue;
        }
        col.Values.Span.Slice(r * w, w).CopyTo(values.AsSpan(o * w, w));
    }
    return new FixedWidthColumnChunk(type, rows.Length, values, nb, anyNull);
}
```

Same gather pattern as `JoinExecutionEngine.MaterializeOneColumn`.

## 12.16 GatherUtf8Column

```csharp
private static IColumnChunk GatherUtf8Column(
    IReadOnlyList<IColumnarBatch> batches, int colIx, ReadOnlySpan<RowLoc> rows)
{
    var offsets = new int[rows.Length + 1];
    using var blob = new MemoryStream();
    // ...
    for (var o = 0; o < rows.Length; o++)
    {
        offsets[o] = (int)blob.Length;
        // null check → SetNull
        // else append payload bytes
    }
    offsets[rows.Length] = (int)blob.Length;
    return new Utf8ColumnChunk(rows.Length, offsets, blob.ToArray(), nb, anyNull);
}
```

Arrow-style offset + blob UTF-8 layout.

## 12.17 Validation

### Sort keys

```csharp
private static void ValidateSortKeys(TableSchema schema, SortKeyPhysicalSpec[] keys)
{
    foreach (var k in keys)
    {
        if ((uint)k.ColumnIndex >= (uint)schema.Columns.Count)
            throw new ArgumentException($"Sort key column index {k.ColumnIndex} is out of range.");
        var t = schema.Columns[k.ColumnIndex].Type;
        if (t != RainDbType.Utf8 && !ColumnTypeSizes.IsFixedWidth(t))
            throw new NotSupportedException($"ORDER BY on type {t} is not supported.");
    }
}
```

Same type restrictions as group keys and join keys.

### Output and filters

Bounds-checks `OutputColumnIndices` and filter column indices against schema.

## 12.18 SQL compilation mapping

Binder produces `SortTopNPhysicalPlan` for single-table `ORDER BY` / `LIMIT`, or `JoinSortTopNPhysicalPlan` when sort follows join.

Example:

```sql
SELECT a, b FROM t WHERE c > 0 ORDER BY b DESC, a LIMIT 100
```

Maps to:

- `Filters`: `c > 0`
- `SortKeys`: `[{b, Descending}, {a, Ascending}]`
- `OutputColumnIndices`: indices of `a`, `b`
- `Limit`: `100`

Global aggregates bypass sort engine. `GROUP BY` key sort happens in `HashAggregateEngine.SortKeys` — separate from `SortTopNEngine`.

## 12.19 Complexity and memory

Let R = qualifying rows, C = output columns, K = sort keys.

| Phase | Time | Extra memory |
|-------|------|--------------|
| Collect | O(R) | `RowLoc[R]` |
| Sort | O(R log R) × compare cost | in-place on array |
| Materialize | O(take × C) | output buffers |

Each compare may read K columns from random batches — cache-unfriendly for large R.

For `LIMIT k` with large R: heap Top-K would be O(R log k) — not implemented.

## 12.20 Comparison to hash aggregate sorting

| | `HashAggregateEngine.SortKeys` | `SortTopNEngine` |
|--|----------------------------------|------------------|
| Input | `GroupKey` in hash map | `RowLoc` across batches |
| Purpose | Deterministic group order | User `ORDER BY` |
| Comparator | `GroupKeyComparer` | `RowLocComparer` |
| Output | In-place key array → materialize | Gather → new batch |

Both use `FixedWidthKeyCompare` / UTF-8 `SequenceCompareTo` for column parts.

## 12.21 Cancellation

`CollectFilteredRows` and `CollectAllRows` check `CancellationToken` per batch. Sort and materialize do not — once collection completes, remaining work runs to completion.

## 12.22 Example: filtered sort with limit

Table `employees`, 3 batches × 1000 rows:

```sql
SELECT name, salary FROM employees
WHERE dept = 'eng'
ORDER BY salary DESC
LIMIT 5
```

```
CollectFilteredRows → 590 RowLoc entries
Array.Sort(RowLocComparer, salary DESC)
take = min(5, 590) = 5
MaterializeRows → ColumnarBatch(5, [name_chunk, salary_chunk])
```

## 12.23 Example: join then sort

```sql
SELECT o.id, c.name
FROM orders o JOIN customers c ON o.cid = c.id
ORDER BY o.id
LIMIT 50
```

```
JoinExecutionEngine → ColumnarBatch (all matches)
CollectAllRows → RowLoc[]
Array.Sort by o.id
take = 50
MaterializeRows (all columns)
```

Join cost is independent of LIMIT.

## 12.24 Empty results

Filters eliminate all rows → `RowLoc[0]` → empty `ColumnarBatch` with typed zero-row chunks.

`Limit = 0` rejected at plan construction (`limit < 1`).

## 12.25 Future optimizations

| Optimization | Benefit |
|--------------|---------|
| Heap Top-K when `Limit << R` | O(R log K) vs O(R log R) |
| External sort + spill | Larger-than-RAM datasets |
| LIMIT pushdown into join | Avoid full match materialization |
| Key normalization before sort | Sort encoded keys once, not per-compare column reads |
| Stable sort (Timsort) | Predictable tie ordering |
| `OFFSET` support | Pagination |

## 12.26 Summary

`SortTopNEngine` implements in-memory sort and limit over columnar data:

1. **`RowLoc`** — indirect row references across batches
2. **`CollectFilteredRows`** — apply `WHERE`, build row list
3. **`RowLocComparer`** — multi-column sort with NULL and DESC handling
4. **`MaterializeRows`** — gather projected columns into one output batch
5. **`ExecuteJoinAsync`** — join via `JoinExecutionEngine`, then same sort/limit path

The design separates **ordering** (permute indices) from **projection** (gather values) — the same separation used in analytics engines that sort selection vectors before vectorized materialization.
