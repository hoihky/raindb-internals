---
title: "Chapter 10: Hash Aggregation"
order: 10
---

# Chapter 10: Hash Aggregation

When a query includes `GROUP BY`, RainDB cannot fold all rows into a single scalar. Instead, `HashAggregateEngine` builds a hash map from **group keys** to **aggregate accumulators**, processes each source batch in parallel to produce partial maps, merges them into a global map, sorts keys for deterministic output, and materializes columnar result batches.

This chapter is split into two parts. **Part I** covers hash-table-based grouping theory: open vs closed addressing, hash aggregation vs sort aggregation, memory bounds and spill, multi-column keys, variable-length keys, and GROUP BY output ordering. **Part II** traces RainDB's implementation — `HashAggregatePhysicalPlan`, `GroupKey`, `CompositeJoinKey`, `MergePartials`, `AggregateRowOps`, and the spill instrumentation hook.

---

# Part I: Theory

## 10.1 Hash table based grouping

`GROUP BY` asks a simple question: "which rows belong together?" The answer is a **group key** — the tuple of values from the grouping columns. A hash map is the standard index from key to running totals:

```
key  →  accumulator state (SUM, COUNT, MIN, ...)
```

Per input row:

1. Read the key columns.
2. Hash the key and probe the table.
3. Insert a fresh accumulator if this key is new.
4. Feed measure columns into the accumulator slot(s).

When the scan finishes, each distinct key becomes one output row with finalized aggregates.

### Multiset semantics

Input is a **bag**: duplicate keys are expected. Ten rows with `region = 'us'` collapse to one output row; `COUNT(*)` reports 10 for that bucket.

### Hash function requirements

`h(key)` should spread keys evenly, stay deterministic across runs, and mix every component of a composite key. Skewed hash functions create **hot buckets** — long chains and worst-case O(n) behavior inside an otherwise O(1) structure.

## 10.2 Open addressing vs closed addressing

Collision resolution splits into two families.

### Closed addressing (separate chaining)

Each bucket points at a chain (or tree) of entries that share the same bucket index. .NET `Dictionary<K,V>` is in this camp.

| Pros | Cons |
|------|------|
| Simple deletion | Pointer chasing, cache misses |
| Load factor can exceed 1 | Extra allocation per entry |
| Tolerates clustering | Memory overhead for small maps |

### Open addressing

All entries sit in one array. On collision, probe forward (linear, quadratic, or double hashing) until an empty slot appears.

| Pros | Cons |
|------|------|
| Cache-friendly contiguous memory | Clustering (linear probing) |
| No per-entry pointers | Deletion needs tombstones |
| Good for SIMD-friendly layouts | Load factor must stay below ~0.7 |

### Engine choice

High-performance analytical engines often build custom open-addressing tables with power-of-two sizing (bitmask instead of modulo). RainDB Phase 1 uses `Dictionary<GroupKey, AggregateAccumulator[]>` — BCL closed addressing — for speed of implementation. A future columnar open-addressing table with inline accumulators is a natural upgrade path.

## 10.3 Hash aggregation vs sort aggregation

Two physical recipes implement the same logical `GROUP BY`.

### Hash aggregation

```
Scan rows → hash into map → (optional) sort keys → emit
```

- **Best when:** many distinct keys, no helpful input order, one pass is enough
- **Memory:** O(distinct groups)
- **Time:** O(n) average insert per row

### Sort aggregation

```
Scan rows → sort by group key → scan sorted runs → accumulate per run → emit
```

- **Best when:** data already sorted on group keys, or hash memory is impossible but external sort is acceptable
- **Memory:** O(n) for the sort buffer (or disk-backed runs)
- **Time:** O(n log n) for the sort

### Hybrid decisions

| Factor | Favors hash | Favors sort |
|--------|-------------|-------------|
| Pre-sorted input on group keys | | ✓ |
| Low cardinality (few groups) | ✓ | |
| Memory budget tight | | ✓ (external sort) |
| Multiple aggregates | ✓ (one pass) | ✓ (one pass after sort) |
| Need ordered output | sort phase either way | natural after sort-agg |

RainDB groups with **hash aggregation** (`HashAggregateEngine`), then **sorts keys** for deterministic output. It does not use sort-aggregation as the primary grouping strategy.

## 10.4 Memory bounds and spill

Memory scales with **distinct groups** × (key footprint + accumulator state). Exceed `work_mem` or the operator budget and something has to give.

### Grace hash aggregation (spill to disk)

1. Partition rows by `hash(key) mod P` into P temp files.
2. Process one partition at a time — each fits in RAM.
3. Write partition results to temp storage.
4. Merge partition outputs.

### Partial spill strategies

- Flush the whole table and start fresh when a threshold trips
- **Metrics-only hooks** (RainDB today) — log that a threshold was crossed without actually spilling

`SpillPartialEntryThreshold` on `HashAggregatePhysicalPlan` calls `ISpillWriter.SpillChunkAsync` with a JSON metrics blob when a partial dictionary grows large. The operator **still finishes in memory**; this is scaffolding for real spill later.

### Memory estimation

```
memory ≈ |G| × (key_bytes + agg_state_bytes + hash_overhead)
```

High-cardinality UTF-8 `GROUP BY` hurts: RainDB copies each distinct string into `CompositeJoinKey` owned payloads.

## 10.5 Multi-column group keys

A composite key is `(k₁, k₂, …, kₙ)`. Hash and equality must consider every component:

```
hash = H( H(k₁) ⊕ H(k₂) ⊕ … ⊕ H(kₙ) )
```

Equality walks components left to right. **Grouping** treats NULL as a real group value — `(NULL, 1)` groups with `(NULL, 1)`. That differs from **join** semantics, where `NULL = NULL` is unknown.

RainDB packs fixed-width components into `ulong[] Parts` with a `uint NullMask` — bit `i` set when column `i` is SQL NULL.

### Lexicographic ordering

Sorted output compares:

1. `NullMask` (which columns are null)
2. Component 0 with type-specific rules
3. Component 1, and so on

Deterministic for tests; full SQL `ORDER BY` null placement may differ.

## 10.6 Handling variable-length keys

`Int32`, `Float64`, and `Boolean` fit in machine words — cheap hash, cheap compare.

Strings need a strategy:

| Approach | Description |
|----------|-------------|
| **Copy-on-insert** | Store payload bytes in the key object (RainDB) |
| **String interning / dictionary** | Map payload to integer id, hash the id |
| **Prefix hash** | Hash first N bytes + length (collision risk) |

`CompositeJoinKey` keeps `byte[]?[] Utf8Payloads` — owned copies from `CompositeJoinKeyBuilder`. Equality is `SequenceEqual` on byte spans.

**Trade-off:** correct, stable dictionary keys versus memory amplification when cardinality is high.

## 10.7 Output ordering of GROUP BY

The SQL standard does **not** promise sorted `GROUP BY` output. Engines sort anyway for regression tests, merge-join consumers, and predictable CLI behavior.

RainDB sorts after merge with `Array.Sort` and `GroupKeyComparer` / `CompositeJoinKeyComparer` before `MaterializeOutput`. That is **policy**, not spec.

User `ORDER BY` on grouped results would today route through `SortTopNEngine` (Chapter 12) as a separate step — not a fused hash+sort operator yet.

---

# Part II: RainDB Implementation

## 10.8 HashAggregatePhysicalPlan

```csharp
// src/RainDB.Query/Plans/HashAggregatePhysicalPlan.cs
public sealed class HashAggregatePhysicalPlan : IPhysicalPlan
{
    public TableId TableId { get; }
    public int[] GroupKeyColumnIndices { get; }
    public AggregateSpec[] Aggregates { get; }
    public HashAggregateOutputSlot[] OutputColumns { get; }
    public ColumnCompareFilter[]? Filters { get; }
    public VectorizedScanExecutionOptions Options { get; }
    public int SpillPartialEntryThreshold { get; }
}
```

Construction constraints:

- At least one group key column and one aggregate
- `OutputColumns` defaults to keys first, then aggregates (or explicit reorder for `SELECT` list)
- `SpillPartialEntryThreshold` defaults to `0` (spill hook disabled)

`Explain()` example:

```
HashAggregate(table=...) KEYS[0,1] AGGS[Sum(2),Count(*)] FILTER[col3>5]
```

### Output slot layout

```csharp
public enum HashAggregateOutputColumnKind { GroupKey, Aggregate }

public readonly record struct HashAggregateOutputSlot(
    HashAggregateOutputColumnKind Kind, int Ordinal);
```

`Ordinal` indexes into `GroupKeyColumnIndices` or `Aggregates` depending on `Kind`. Supports `SELECT k, SUM(v), k` reordering without changing accumulation.

`AggregateSpec` (from `VectorizedScanPhysicalPlan.cs`):

```csharp
public readonly record struct AggregateSpec(int SourceColumnIndex, AggregateKind Kind);
```

## 10.9 Engine entry and dual code paths

```csharp
// src/RainDB.Query/Execution/HashAggregateEngine.cs
public static async ValueTask<IQueryResult> ExecuteAsync(
    HashAggregatePhysicalPlan plan,
    IColumnarTableSource table,
    IExecutionContext context)
{
    ValidatePlan(plan, table);

    if (AnyUtf8GroupKey(plan, table.Schema))
        return await ExecuteWithCompositeKeysAsync(plan, table, context).ConfigureAwait(false);

    var batches = table.Batches;
    var n = batches.Count;
    // ...
    var partials = new Dictionary<GroupKey, AggregateAccumulator[]>[n];
    // parallel AccumulateBatch per batch index
    var global = MergePartials(partials, plan.Aggregates);
    var sortedKeys = SortKeys(global.Keys, schema, plan.GroupKeyColumnIndices);
    var outBatch = MaterializeOutput(sortedKeys, global, plan, schema);
    return new ColumnarMaterializedQueryResult([outBatch]);
}
```

`AnyUtf8GroupKey` scans `GroupKeyColumnIndices` — any `RainDbType.Utf8` column routes to `CompositeJoinKey` path (shared with join infrastructure). Mixed fixed-width and UTF-8 keys in one `GROUP BY` use the composite path.

### Empty table

```csharp
if (n == 0)
{
    var emptyCols = MaterializeEmptyOutput(plan, schema);
    return new ColumnarMaterializedQueryResult([new ColumnarBatch(0, emptyCols)]);
}
```

Empty grouped query: **zero output rows** (unlike global `COUNT(*)` which returns one row with `0`).

## 10.10 GroupKey and FixedWidthGroupKeyBuilder

```csharp
// src/RainDB.Query/Execution/FixedWidthGroupKey.cs
internal sealed class GroupKey : IEquatable<GroupKey>
{
    public GroupKey(ulong[] parts, uint nullMask)
    {
        Parts = parts;
        NullMask = nullMask;
    }

    public ulong[] Parts { get; }
    public uint NullMask { get; }
}
```

| Field | Role |
|-------|------|
| `Parts` | One `ulong` per key column, raw little-endian bits |
| `NullMask` | Bit `i` set when key column `i` is SQL NULL |

`Equals` compares `NullMask` then `Parts` element-wise. `GetHashCode` mixes both. SQL NULL group keys are **valid** — NULL groups with NULL.

### BuildKey from a row

```csharp
internal static class FixedWidthGroupKeyBuilder
{
    public static GroupKey BuildKey(
        IColumnarBatch batch, int row, int[] keyIndices, ulong[] scratch)
    {
        uint mask = 0;
        for (var i = 0; i < keyIndices.Length; i++)
        {
            var col = batch.Columns[keyIndices[i]];
            var nb = col.HasNulls ? col.NullBitmap.Span : ReadOnlySpan<byte>.Empty;
            if (SelectionEvaluator.IsNull(nb, row, col.HasNulls))
            {
                mask |= 1u << i;
                scratch[i] = 0;
                continue;
            }
            scratch[i] = PhysicalValueToULong(col, row);
        }

        var owned = new ulong[keyIndices.Length];
        scratch.AsSpan(0, keyIndices.Length).CopyTo(owned);
        return new GroupKey(owned, mask);
    }
}
```

**Scratch reuse:** `AccumulateBatch` rents `ulong[]` from `ArrayPool` per batch. Each `GroupKey` **owns** copied `ulong[]` for dictionary stability.

### PhysicalValueToULong

```csharp
public static ulong PhysicalValueToULong(IColumnChunk col, int row)
{
    var values = col.Values.Span;
    return col.PhysicalType switch
    {
        RainDbType.Int32 => (ulong)(uint)BinaryPrimitives.ReadInt32LittleEndian(
            values.Slice(row * sizeof(int), sizeof(int))),
        RainDbType.Int64 => (ulong)BinaryPrimitives.ReadInt64LittleEndian(
            values.Slice(row * sizeof(long), sizeof(long))),
        RainDbType.Float64 => (ulong)BinaryPrimitives.ReadInt64LittleEndian(
            values.Slice(row * sizeof(double), sizeof(double))),
        RainDbType.Boolean => values[row] != 0 ? 1UL : 0UL,
        _ => throw new InvalidOperationException($"Unexpected key physical type {col.PhysicalType}."),
    };
}
```

Float keys compare by IEEE-754 bit pattern via `FixedWidthKeyCompare` — not numeric tolerance.

## 10.11 AccumulateBatch (fixed-width path)

```csharp
private static Dictionary<GroupKey, AggregateAccumulator[]> AccumulateBatch(
    IColumnarBatch batch,
    HashAggregatePhysicalPlan plan,
    CancellationToken cancellationToken)
{
    var specs = plan.Aggregates;
    var aggCount = specs.Length;
    var dict = new Dictionary<GroupKey, AggregateAccumulator[]>();
    var rent = ArrayPool<int>.Shared.Rent(batch.RowCount);
    try
    {
        // filter → selection buffer
        var scratch = ArrayPool<ulong>.Shared.Rent(plan.GroupKeyColumnIndices.Length);
        try
        {
            for (var i = 0; i < k; i++)
            {
                var row = sel.IsEmpty ? i : sel[i];
                var key = FixedWidthGroupKeyBuilder.BuildKey(
                    batch, row, plan.GroupKeyColumnIndices, scratch);
                if (!dict.TryGetValue(key, out var accs))
                {
                    accs = new AggregateAccumulator[aggCount];
                    dict[key] = accs;
                }
                for (var a = 0; a < aggCount; a++)
                {
                    ref var slot = ref accs[a];
                    var spec = specs[a];
                    if (spec.Kind == AggregateKind.Count && spec.SourceColumnIndex < 0)
                        AggregateRowOps.AddCountStar(ref slot);
                    else if (spec.Kind == AggregateKind.Count)
                        AggregateRowOps.AddCountColumn(ref slot, batch.Columns[spec.SourceColumnIndex], row);
                    else
                        AggregateRowOps.AddRow(ref slot, batch.Columns[spec.SourceColumnIndex], spec.Kind, row);
                }
            }
        }
        finally { ArrayPool<ulong>.Shared.Return(scratch); }
        return dict;
    }
    finally { ArrayPool<int>.Shared.Return(rent); }
}
```

Parallel workers each own a `Dictionary` — no cross-thread contention. Cross-batch duplicate keys merge in `MergePartials`.

## 10.12 CompositeJoinKey UTF-8 path

```csharp
// src/RainDB.Query/Execution/JoinCompositeKey.cs
internal sealed class CompositeJoinKey : IEquatable<CompositeJoinKey>
{
    public CompositeJoinKey(uint nullMask, ulong[] numericParts, byte[]?[] utf8Payloads)
    {
        NullMask = nullMask;
        NumericParts = numericParts;
        Utf8Payloads = utf8Payloads;
    }

    public uint NullMask { get; }
    public ulong[] NumericParts { get; }
    public byte[]?[] Utf8Payloads { get; }
}
```

Per key column index `i`:

- Fixed-width → `NumericParts[i]`, `Utf8Payloads[i]` null
- `Utf8` → copied payload in `Utf8Payloads[i]`
- NULL → `NullMask` bit set

`CompositeJoinKeyBuilder.Build` in the same file handles `Utf8ColumnChunk` and `Utf8LengthPrefixedColumnChunk`.

### DeepClone on merge

```csharp
global[kv.Key.DeepClone()] = merged;
```

Prevents shared mutable key arrays between partial and global dictionaries.

`ExecuteWithCompositeKeysAsync` mirrors fixed-width flow: parallel `AccumulateBatchComposite`, `MergePartialsComposite`, `SortCompositeKeys`, `MaterializeOutputComposite`.

## 10.13 MergePartials

```csharp
private static Dictionary<GroupKey, AggregateAccumulator[]> MergePartials(
    Dictionary<GroupKey, AggregateAccumulator[]>[] partials,
    AggregateSpec[] specs)
{
    var aggCount = specs.Length;
    var global = new Dictionary<GroupKey, AggregateAccumulator[]>();
    for (var bi = 0; bi < partials.Length; bi++)
    {
        foreach (var kv in partials[bi])
        {
            if (!global.TryGetValue(kv.Key, out var merged))
            {
                merged = new AggregateAccumulator[aggCount];
                for (var j = 0; j < aggCount; j++)
                    merged[j] = kv.Value[j];
                global[new GroupKey(kv.Key.Parts.ToArray(), kv.Key.NullMask)] = merged;
            }
            else
            {
                for (var j = 0; j < aggCount; j++)
                    merged[j] = AggregateRowOps.Combine(merged[j], kv.Value[j], specs[j].Kind);
            }
        }
    }
    return global;
}
```

**Key copying:** `Parts.ToArray()` on insert — defensive copy.

**Per-slot combine:** each `AggregateSpec` at index `j` uses its own `Kind`.

Complexity: O(total entries in partial maps) ≤ O(batches × distinct groups).

## 10.14 AggregateRowOps

```csharp
internal struct AggregateAccumulator
{
    public long ContributingRows;
    public long Count;
    public double FloatSum;
    public double FloatMin;
    public double FloatMax;
    public long IntSum;
    public bool HasMin;
    public bool HasMax;
}
```

Parallel to `PartialAgg` (Chapter 9) with `Count` instead of `CountAgg`.

### AddCountStar / AddCountColumn

```csharp
public static void AddCountStar(ref AggregateAccumulator acc) => acc.Count++;

public static void AddCountColumn(ref AggregateAccumulator acc, IColumnChunk col, int row)
{
    var nb = col.HasNulls ? col.NullBitmap.Span : ReadOnlySpan<byte>.Empty;
    if (!SelectionEvaluator.IsNull(nb, row, col.HasNulls))
        acc.Count++;
}
```

### AddRow

```csharp
public static void AddRow(ref AggregateAccumulator acc, IColumnChunk col, AggregateKind kind, int row)
{
    var nb = col.HasNulls ? col.NullBitmap.Span : ReadOnlySpan<byte>.Empty;
    if (SelectionEvaluator.IsNull(nb, row, col.HasNulls))
        return;

    var values = col.Values.Span;
    switch (kind)
    {
        case AggregateKind.Sum when col.PhysicalType == RainDbType.Float64:
            // acc.FloatSum += v; acc.ContributingRows++
        case AggregateKind.Sum when col.PhysicalType == RainDbType.Int32:
            // acc.IntSum += i32
        case AggregateKind.Min when col.PhysicalType == RainDbType.Float64:
            // HasMin / FloatMin tracking
        // ...
    }
}
```

### Combine

```csharp
public static AggregateAccumulator Combine(AggregateAccumulator a, AggregateAccumulator b, AggregateKind kind) =>
    kind switch
    {
        AggregateKind.Count => new AggregateAccumulator { Count = a.Count + b.Count },
        AggregateKind.Sum => new AggregateAccumulator
        {
            ContributingRows = a.ContributingRows + b.ContributingRows,
            FloatSum = a.FloatSum + b.FloatSum,
            IntSum = a.IntSum + b.IntSum,
        },
        AggregateKind.Min => CombineMinMax(a, b, isMin: true),
        AggregateKind.Max => CombineMinMax(a, b, isMin: false),
        _ => throw new ArgumentOutOfRangeException(nameof(kind), kind, null),
    };
```

`CombineMinMax` matches `PartialAgg.CombineMinMax` — identity for empty sides.

## 10.15 Sorting group keys

```csharp
private static GroupKey[] SortKeys(
    Dictionary<GroupKey, AggregateAccumulator[]>.KeyCollection keys,
    TableSchema schema,
    int[] keyIndices)
{
    var arr = new GroupKey[keys.Count];
    keys.CopyTo(arr, 0);
    Array.Sort(arr, new GroupKeyComparer(schema, keyIndices));
    return arr;
}
```

`GroupKeyComparer`:

```csharp
public int Compare(GroupKey? x, GroupKey? y)
{
    var c = x.NullMask.CompareTo(y.NullMask);
    if (c != 0) return c;

    for (var i = 0; i < _keyIndices.Length; i++)
    {
        var t = _schema.Columns[_keyIndices[i]].Type;
        c = ComparePart(t, x.Parts[i], y.Parts[i]);
        if (c != 0) return c;
    }
    return 0;
}
```

`ComparePart` delegates to `FixedWidthKeyCompare.ComparePart` — shared with join ordering.

`CompositeJoinKeyComparer` adds UTF-8 `SequenceCompareTo` on payload bytes.

## 10.16 MaterializeOutput

```csharp
private static ColumnarBatch MaterializeOutput(
    GroupKey[] sortedKeys,
    Dictionary<GroupKey, AggregateAccumulator[]> global,
    HashAggregatePhysicalPlan plan,
    TableSchema schema)
{
    var rowCount = sortedKeys.Length;
    var cols = new List<IColumnChunk>();
    foreach (var outSlot in plan.OutputColumns)
    {
        switch (outSlot.Kind)
        {
            case HashAggregateOutputColumnKind.GroupKey:
                var schemaCol = schema.Columns[plan.GroupKeyColumnIndices[outSlot.Ordinal]];
                cols.Add(MaterializeKeyColumn(sortedKeys, outSlot.Ordinal, schemaCol.Type, rowCount));
                break;
            case HashAggregateOutputColumnKind.Aggregate:
                var spec = plan.Aggregates[outSlot.Ordinal];
                cols.Add(MaterializeAggregateColumn(sortedKeys, global, outSlot.Ordinal, spec, schema, rowCount));
                break;
        }
    }
    return new ColumnarBatch(rowCount, cols);
}
```

### MaterializeKeyColumn

Walks sorted keys, sets null bits from `NullMask`, writes physical bytes via `WritePhysical` (little-endian). UTF-8 uses `MaterializeUtf8KeyColumn` — offset array + contiguous blob.

### ShouldEmitAggregateNull

```csharp
private static bool ShouldEmitAggregateNull(AggregateSpec spec, AggregateAccumulator acc)
{
    return spec.Kind switch
    {
        AggregateKind.Sum => acc.ContributingRows == 0,
        AggregateKind.Min => !acc.HasMin,
        AggregateKind.Max => !acc.HasMax,
        AggregateKind.Count => false,
        _ => false,
    };
}
```

For a group where all measure values are null: `SUM` → null bit; `COUNT(col)` → `0` without null; `COUNT(*)` → row count.

```csharp
var hasNulls = anyNull;
if (spec.Kind == AggregateKind.Count)
    hasNulls = false;
```

## 10.17 Spill hook

```csharp
if (context.SpillWriter.IsEnabled && plan.SpillPartialEntryThreshold > 0)
{
    for (var i = 0; i < n; i++)
    {
        if (partials[i].Count >= plan.SpillPartialEntryThreshold)
        {
            var payload = Encoding.UTF8.GetBytes(
                $"{{\"op\":\"hash_agg_partial\",\"batch\":{i},\"entries\":{partials[i].Count}}}\n");
            await context.SpillWriter.SpillChunkAsync(payload, ct).ConfigureAwait(false);
        }
    }
}
```

UTF-8 path emits `hash_agg_partial_utf8`. Conditions:

1. `IExecutionContext.SpillWriter.IsEnabled`
2. `SpillPartialEntryThreshold > 0`
3. Partial dictionary entry count ≥ threshold

Operator still completes in memory — instrumentation for profiling and future spill.

`ISpillWriter` defined in `src/RainDB.Abstractions/Execution/ISpillWriter.cs`.

## 10.18 Validation and parallelism

`ValidatePlan` checks:

- Table id match
- Key types: fixed-width or `Utf8` only
- Filter column indices in range
- Aggregate type rules (`Min`/`Max` on `Float64` only; `Sum` on `Int32`/`Int64`/`Float64`)

Parallelism: same three modes as `VectorizedScanEngine` — sequential, `Parallel.For`, channel scheduler via `RunChannelMorselsAsync`. Merge and materialize are sequential.

## 10.19 GroupedJoinPhysicalPlan integration

`DefaultQueryExecutor` handles `GroupedJoinPhysicalPlan`:

```csharp
// src/RainDB.Query/Execution/DefaultQueryExecutor.cs
var joinResult = await JoinExecutionEngine.ExecuteAsync(grouped.Join, probeCols2, buildCols2, context);
var ephemeral = new EphemeralColumnarTableSource(
    grouped.Aggregate.TableId, "_grouped_join_", grouped.Join.OutputSchema, colResult.Batches);
return await HashAggregateEngine.ExecuteAsync(grouped.Aggregate, ephemeral, context);
```

Join output becomes input to hash aggregation — `GROUP BY` on joined columns without persisting intermediate results.

## 10.20 FixedWidthKeyCompare

```csharp
// src/RainDB.Query/Execution/FixedWidthKeyCompare.cs
internal static int ComparePart(RainDbType type, ulong xa, ulong xb) =>
    type switch
    {
        RainDbType.Int32 => ((int)(uint)xa).CompareTo((int)(uint)xb),
        RainDbType.Int64 => ((long)xa).CompareTo((long)xb),
        RainDbType.Float64 => CompareFloat64Bits(xa, xb),
        RainDbType.Boolean => ((xa != 0) ? 1 : 0).CompareTo(xb != 0 ? 1 : 0),
        _ => xa.CompareTo(xb),
    };
```

Shared across hash aggregate sort, join sort-merge, and composite key compare.

## 10.21 Example trace

```sql
SELECT region, SUM(amount), COUNT(*)
FROM sales
WHERE active = 1
GROUP BY region;
```

```
ExecuteAsync
  batch 0: partial map { "us" → [sum=100, count=5], "eu" → [sum=200, count=3] }
  batch 1: partial map { "us" → [sum=50, count=2], "ap" → [sum=80, count=1] }
  MergePartials → global { us: [150, 7], eu: [200, 3], ap: [80, 1] }
  SortKeys → [ap, eu, us] (lexicographic on region column)
  MaterializeOutput → ColumnarBatch(3 rows, [region, sum, count])
```

## 10.22 Tests and limitations

Tests: `HashAggregatePhysicalTests`, `SqlGroupByTests` in `tests/RainDB.Tests/`.

| Limitation | Notes |
|------------|-------|
| No `DISTINCT` aggregates | `COUNT(DISTINCT x)` not supported |
| No `AVG` builtin | Use `SUM/COUNT` |
| In-memory only | Spill hook is metrics-only |
| No sort-aggregation | Hash-only grouping |
| SIMD | Per-row `AggregateRowOps`, no AVX2 grouped sum |

## 10.23 Summary

Hash aggregation in RainDB follows a classic partial-aggregate pattern:

1. **Encode** row keys as `GroupKey` or `CompositeJoinKey`
2. **Accumulate** per batch in isolated hash maps using `AggregateRowOps`
3. **Merge** partials with associative `Combine`
4. **Sort** keys for deterministic output order
5. **Materialize** column chunks with SQL-correct null bits on aggregates

The UTF-8 path reuses join key infrastructure (`CompositeJoinKey`, `CompositeJoinKeyBuilder`, `CompositeJoinKeyComparer`), keeping equality semantics consistent across `GROUP BY` and `JOIN` operators.
