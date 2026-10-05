---
title: "Chapter 10: Hash Aggregation"
order: 10
---

# Chapter 10: Hash Aggregation

When a query includes `GROUP BY`, RainDB builds a hash map from **group keys** to **aggregate accumulators**, processes each source batch in parallel to produce partial maps, merges them into a global map, sorts keys for deterministic output, optionally applies **HAVING**, and materializes columnar result batches.

This chapter is split into two parts. **Part I** covers hash-table-based grouping theory. **Part II** traces RainDB's `HashAggregateOperator`, `GroupHavingEvaluator`, extended `MIN`/`MAX` support, the streaming grouped-join merge path, and the metrics-only spill hook.

---

# Part I: Theory

## 10.1 Hash table based grouping

`GROUP BY` asks: which rows belong together? The answer is a **group key** — the tuple of values from the grouping columns. A hash map indexes key → running totals:

```
key  →  accumulator state (SUM, COUNT, MIN, ...)
```

Per input row: read key columns, hash and probe, insert fresh accumulator on first sight, update measures for every aggregate in the SELECT list.

### Multiset semantics

Input is a **bag**: duplicate keys are expected. Ten rows with `region = 'us'` collapse to one output row; `COUNT(*)` reports 10 for that bucket.

### Hash function requirements

`h(key)` should spread keys evenly, stay deterministic, and mix every component of a composite key. Skew creates hot buckets and long chains.

## 10.2 Open addressing vs closed addressing

**Closed addressing** (separate chaining) — each bucket points at a chain of entries. .NET `Dictionary<K,V>` uses this model.

**Open addressing** — all entries in one array; probe on collision. Often more cache-friendly at high load factors.

RainDB Phase 1 uses `Dictionary<GroupKey, AggregateAccumulator[]>` (and `CompositeJoinKey` for UTF-8 keys) for implementation speed.

## 10.3 Hash aggregation vs sort aggregation

**Hash aggregation:** scan → hash into map → (optional) sort keys → emit. Best when many distinct keys fit in memory.

**Sort aggregation:** scan → sort by group key → scan runs → accumulate. Best when input is already ordered on keys or external sort is acceptable.

RainDB groups with **hash aggregation**, then **sorts keys** for deterministic output. It does not use sort-aggregation as the primary grouping strategy.

## 10.4 Memory bounds and spill

Memory scales with distinct groups × (key footprint + accumulator state). **Grace hash aggregation** partitions by `hash(key) mod P`, processes one partition at a time, and merges partition outputs.

RainDB's `SpillPartialEntryThreshold` on `HashAggregatePhysicalPlan` can call `ISpillWriter.SpillChunkAsync` with a JSON metrics blob when a partial dictionary grows large. The operator **still finishes in memory** — instrumentation for profiling and future real spill.

## 10.5 Multi-column group keys

Composite keys `(k₁, k₂, …)` need combined hash and equality. **Grouping** treats NULL as a real group value — `(NULL, 1)` groups with `(NULL, 1)`. That differs from **join** semantics where `NULL = NULL` is unknown.

RainDB packs fixed-width components into `ulong[] Parts` with `uint NullMask` — bit `i` set when column `i` is SQL NULL.

## 10.6 Variable-length keys

Strings use **copy-on-insert** in `CompositeJoinKey.Utf8Payloads` via `CompositeJoinKeyBuilder`. Equality is byte `SequenceEqual`. High-cardinality UTF-8 `GROUP BY` amplifies memory.

## 10.7 Output ordering of GROUP BY

SQL does not require sorted `GROUP BY` output. RainDB sorts after merge with `GroupKeyComparer` / `CompositeJoinKeyComparer` before materialization — policy for tests and predictable CLI behavior.

User `ORDER BY` on grouped results can fuse into `GroupedSortTopNPhysicalPlan` (Chapter 12) instead of a separate sort over a bare hash-aggregate result.

---

# Part II: RainDB Implementation

## 10.8 HashAggregatePhysicalPlan

```csharp
// src/RainDB.Query/Plans/HashAggregatePhysicalPlan.cs
public sealed class HashAggregatePhysicalPlan : IPhysicalPlan
{
    public int[] GroupKeyColumnIndices { get; }
    public AggregateSpec[] Aggregates { get; }
    public HashAggregateOutputSlot[] OutputColumns { get; }
    public ColumnCompareFilter[]? Filters { get; }
    public GroupOutputCompareFilter[]? HavingFilters { get; }
    public VectorizedScanExecutionOptions Options { get; }
    public int SpillPartialEntryThreshold { get; }
}
```

`HavingFilters` carries post-aggregate predicates bound from SQL `HAVING`. `OutputColumns` reorder keys and aggregates to match the `SELECT` list.

## 10.9 HashAggregateOperator entry

`HashAggregateOperator` (`src/RainDB.Query/Execution/HashAggregateEngine.cs`) implements `IHashAggregateOperator` and `IHashAggregateGroupingSupport`.

Flow:

1. Validate plan against table schema.
2. Resolve uncorrelated `IN` / `EXISTS` subquery filters when present.
3. Branch to fixed-width `GroupKey` path or `CompositeJoinKey` path when any group key column is `Utf8`.
4. Parallel `AccumulateBatch` per source batch → `MergePartials`.
5. Optional spill **metrics** when threshold exceeded.
6. `SortKeys` → `MaterializeOutput` → **`ApplyHavingIfNeeded`**.

Empty grouped input returns a zero-row batch (unlike global `COUNT(*)` which returns one row with `0`).

## 10.10 GroupKey encoding

`GroupKey` (`src/RainDB.Query/Execution/FixedWidthGroupKey.cs`) owns `ulong[] Parts` and `uint NullMask`. `FixedWidthGroupKeyBuilder.BuildKey` sets null bits and copies physical little-endian bits into scratch, then into an owned array for dictionary stability.

`PhysicalValueToULong` supports `Int32`, `Int64`, `Float64`, and `Boolean`. Float keys compare by IEEE bit pattern via `FixedWidthKeyCompare`.

## 10.11 AccumulateBatch and parallelism

Each parallel worker owns an isolated `Dictionary`. Cross-batch duplicate keys merge in `MergePartials` with `AggregateRowOps.Combine` per aggregate slot.

Three scheduling modes match the scan operator: sequential, `Parallel.For`, or channel scheduler via `RunChannelMorselsAsync`.

## 10.12 CompositeJoinKey UTF-8 path

When `AnyUtf8GroupKey` is true, `ExecuteWithCompositeKeysAsync` mirrors the fixed-width flow with `AccumulateBatchComposite`, `MergePartialsComposite`, `SortCompositeKeys`, and `MaterializeOutputComposite`. `DeepClone` on merge prevents shared mutable key payloads.

## 10.13 AggregateRowOps and MIN/MAX types

`AggregateAccumulator` (`src/RainDB.Query/Execution/HashAggregateEngine.cs`, nested `AggregateRowOps`) extends the global `PartialAgg` shape with:

- Integer min/max: `Int32Min`/`Int32Max`, `Int64Min`/`Int64Max`
- Float min/max: `FloatMin`/`FloatMax` with `HasMin`/`HasMax`
- UTF-8 min/max: `Utf8MinBytes` / `Utf8MaxBytes` with lexicographic `Utf8Compare`

`AddRow` skips null measure cells via the shared `SelectionEvaluator`. `Combine` merges partials associatively; `CombineMinMax` for integers and UTF-8 follows the same identity rules as float extrema.

Plan validation allows `Min`/`Max` when the source column type is `Int32`, `Int64`, `Float64`, or `Utf8`:

```csharp
// Validate excerpt — materialize switch
case AggregateKind.Min or AggregateKind.Max when columnType is RainDbType.Float64 or RainDbType.Int32 or RainDbType.Int64 or RainDbType.Utf8:
```

`MaterializeUtf8AggregateColumn` emits length-prefixed UTF-8 blobs for per-group `MIN`/`MAX` on string measures. `ShouldEmitAggregateNull` sets aggregate null bits when no non-null inputs contributed (`ContributingRows == 0` for sum, `!HasMin` / `!HasMax` for extrema, never for `COUNT`).

### Grouped null semantics (output)

| Aggregate | All measure values NULL in group |
|-----------|----------------------------------|
| `SUM` | NULL bit set |
| `MIN` / `MAX` | NULL bit set |
| `COUNT(col)` | `0`, not null |
| `COUNT(*)` | row count in group, not null |

This matches global `IAggregateQueryResult.ValueIsNull` rules, but encoded as column null bitmaps in a `ColumnarBatch`.

## 10.14 HAVING: GroupHavingEvaluator

After materialization, `ApplyHavingIfNeeded` invokes `GroupHavingEvaluator.Apply` (`src/RainDB.Query/Vectorized/GroupHavingEvaluator.cs`) when `HavingFilters` is non-empty.

The evaluator scans each output row and tests `GroupOutputCompareFilter` conjuncts against **post-aggregate output columns** (indices refer to the grouped result schema, not source table columns). Supported comparisons:

- Fixed-width: `Int32`, `Int64`, `Float64` with `ScalarCompareOp` and immediate operands
- `Utf8`: equality / inequality against a literal byte span

Rows that fail any predicate are dropped. Survivors are gathered into a new `ColumnarBatch` via `ProjectGather` with an explicit row selection — no re-hash of source data.

SQL binding (`GroupedHavingBinder` in the SQL compilation layer) produces `HavingFilters` from `HAVING` clauses on grouped queries.

## 10.15 Spill hook (metrics only)

When `context.SpillWriter.IsEnabled`, `SpillPartialEntryThreshold > 0`, and a partial dictionary entry count crosses the threshold, the operator writes a small UTF-8 JSON line describing `hash_agg_partial` (or `hash_agg_partial_utf8` on the composite path). Execution still completes in memory.

## 10.16 GroupedJoinOperator: streaming merge

`GroupedJoinPhysicalPlan` no longer materializes the full join into an ephemeral table before grouping. `GroupedJoinOperator` (`src/RainDB.Query/Execution/GroupedJoinOperator.cs`) pipelines join output directly into hash aggregation:

1. Allocate a global `Dictionary<GroupKey, AggregateAccumulator[]>` (or composite variant).
2. Call `JoinOperator.ExecuteStreaming` with a callback that receives each join output `ColumnarBatch`.
3. For each batch: `AccumulateBatchForGrouped` → `MergePartialIntoGlobal`.
4. `MaterializeFromGlobalAsync` produces the final grouped result.

`ExecuteStreaming` uses `JoinMatchChunkEmitter.DefaultChunkRowCount` (**8192** rows) so peak memory stays bounded by chunk size rather than a monolithic match list (Chapter 11).

Fixed-width and UTF-8 group keys pick `ExecuteFixedWidthAsync` vs `ExecuteCompositeAsync` based on `IHashAggregateGroupingSupport.UsesCompositeGroupKeys`.

## 10.17 Integration with executor

`DefaultQueryExecutor` dispatches `HashAggregatePhysicalPlan` to `_operators.HashAggregate` and `GroupedJoinPhysicalPlan` to `_operators.GroupedJoin`.

`GroupedSortTopNPhysicalPlan` runs hash aggregate first, then sorts the grouped batch through a second `SortTopNPhysicalPlan` over an ephemeral table (Chapter 12).

## 10.18 Example trace

```sql
SELECT region, SUM(amount), MIN(score), MAX(name)
FROM sales
WHERE active = 1
GROUP BY region
HAVING SUM(amount) > 100;
```

```
HashAggregateOperator.ExecuteAsync
  parallel partial maps per batch
  MergePartials → SortKeys → MaterializeOutput
  GroupHavingEvaluator.Apply on SUM(amount) > 100
```

## 10.19 Limitations

| Topic | Notes |
|-------|-------|
| In-memory grouping | Spill hook is metrics-only |
| Sort-aggregation | Hash-only primary path |
| SIMD grouped sums | Per-row `AggregateRowOps`, no AVX2 in hash agg |
| HAVING | Compares materialized aggregate columns; not arbitrary expressions on every SQL feature |

## 10.20 Summary

Hash aggregation in RainDB:

1. **Encode** keys as `GroupKey` or `CompositeJoinKey`
2. **Accumulate** per batch in isolated hash maps
3. **Merge** with associative `AggregateRowOps.Combine`
4. **Sort** keys for deterministic order
5. **Materialize** with SQL-correct null bits on aggregates, including `MIN`/`MAX` on integers, floats, and UTF-8
6. **Filter** with `GroupHavingEvaluator` when `HAVING` is present
7. **Stream** from joins via `GroupedJoinOperator` without retaining the full join rowset

UTF-8 keys reuse join infrastructure (`CompositeJoinKey`, `CompositeJoinKeyBuilder`, `CompositeJoinKeyComparer`) so equality semantics stay aligned across `GROUP BY` and `JOIN`.
