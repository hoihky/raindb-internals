---
title: "Chapter 12: Sort and Limit"
order: 12
---

# Chapter 12: Sort and Limit

`ORDER BY` and `LIMIT` in RainDB are handled by `SortTopNOperator` (`src/RainDB.Query/Execution/SortTopNEngine.cs`) — collect row locations from columnar batches, choose between full sort and **bounded heap top-K**, gather projected columns into a result batch. Grouped queries use fused physical plans that hash-aggregate first, then sort the grouped output.

This chapter is split into two parts. **Part I** covers sorting theory. **Part II** traces `SortTopNPhysicalPlan`, `SchemaRowLocationComparer`, `BoundedTopKHeap`, expression sort keys, and `GroupedSortTopNPhysicalPlan` / `GroupedJoinSortTopNPhysicalPlan`.

---

# Part I: Theory

## 12.1 Sorting in query engines

Sort appears in `ORDER BY`, sort-merge join, deterministic `GROUP BY` output, window functions, and duplicate removal. It is often expensive because it touches every row and compares keys repeatedly.

### In-memory sort

Collect row handles, comparison-sort (typically O(n log n)), emit or gather. .NET `Array.Sort` with a custom comparer is the full-sort backbone in RainDB.

### External sort

When data exceeds RAM: generate sorted runs, k-way merge. RainDB Phase 1 has **no external sort** — working sets must fit in process memory.

### Sort keys vs row payloads

Compare skinny `(batch_idx, row_idx)` records — **RowLocation** in RainDB — not full row payloads. Sort permutes indices; gather reads column values afterward.

## 12.2 Stable vs unstable sort

SQL does not mandate tie order for `ORDER BY k` alone. .NET `Array.Sort` with `IComparer<T>` is not stable. Add explicit secondary keys for deterministic ties.

## 12.3 ORDER BY semantics

Multi-column order is lexicographic with per-key `ASC`/`DESC`. RainDB puts **NULLs first** on ascending keys in `SchemaRowLocationComparer`; `DESC` negates the comparison so nulls tend toward the end on descending keys. `NULLS FIRST` / `NULLS LAST` syntax is not exposed yet.

## 12.4 Top-K algorithms

`LIMIT k` after `ORDER BY` needs only the best k rows, not a full ranking.

| Approach | Time | Space |
|----------|------|-------|
| Full sort + slice | O(n log n) | O(n) |
| Bounded heap | O(n log k) | O(k) |
| Quickselect | O(n) avg | O(1) extra |

When **k ≪ n**, a heap wins. RainDB uses `BoundedTopKHeap` in that regime (Part II).

## 12.5 Sort vs hash for grouping

RainDB hashes for `GROUP BY` (Chapter 10), then may sort grouped output for user `ORDER BY` via fused plans — not sort-aggregation as the primary grouping strategy.

## 12.6 Sort as physical operator

RainDB fuses sort and limit in `SortTopNPhysicalPlan`, `JoinSortTopNPhysicalPlan`, `GroupedSortTopNPhysicalPlan`, and `GroupedJoinSortTopNPhysicalPlan` rather than separate Sort + Limit nodes.

`LIMIT` without `ORDER BY` truncates in collection order (batch index, then row index).

---

# Part II: RainDB Implementation

## 12.7 Physical plans

### SortTopNPhysicalPlan

Single-table scan with optional filters, projection indices, sort keys, and limit.

### JoinSortTopNPhysicalPlan

Nested `JoinPhysicalPlan` plus sort keys and limit on the join output schema.

### GroupedSortTopNPhysicalPlan

```csharp
// src/RainDB.Query/Plans/GroupedSortTopNPhysicalPlan.cs
public sealed class GroupedSortTopNPhysicalPlan : IPhysicalPlan
{
    public HashAggregatePhysicalPlan Aggregate { get; }
    public TableSchema OutputSchema { get; }
    public SortKeyPhysicalSpec[] SortKeys { get; }
    public int? Limit { get; }
}
```

`LogicalTableScanBinder` emits this when a single-table query has `GROUP BY` with `ORDER BY` and/or `LIMIT`.

### GroupedJoinSortTopNPhysicalPlan

```csharp
// src/RainDB.Query/Plans/GroupedJoinSortTopNPhysicalPlan.cs
public sealed class GroupedJoinSortTopNPhysicalPlan : IPhysicalPlan
{
    public GroupedJoinPhysicalPlan GroupedJoin { get; }
    public TableSchema OutputSchema { get; }
    public SortKeyPhysicalSpec[] SortKeys { get; }
    public int? Limit { get; }
}
```

Emitted from `LogicalJoinBinder` when join + `GROUP BY` + sort/limit appear together.

`DefaultQueryExecutor` runs aggregate/grouped-join first, then sorts via `SortGroupedOutputAsync` — wrapping the grouped batch in an `EphemeralColumnarTableSource` and executing a `SortTopNPhysicalPlan` over the grouped output schema.

## 12.8 SortKeyPhysicalSpec and expression keys

```csharp
// src/RainDB.Query/Plans/SortTopNPhysicalPlan.cs
public readonly record struct SortKeyPhysicalSpec(
    int ColumnIndex = -1,
    bool Descending = false,
    BoundInt32RowExpression? Int32SortExpression = null,
    BoundFloat64RowExpression? Float64SortExpression = null);
```

Column-index keys reference positions in the row schema (table columns for scans; join or grouped output columns for fused plans).

**Expression keys:** `LogicalTableScanBinder.BuildTableSortKeySpecs` binds `LogicalSortKey.SortExpression` through `ScalarExpressionBindingPipeline` into `Int32SortExpression` or `Float64SortExpression` on the physical spec. Grouped sort specs (`BuildGroupedSortSpecs`, join variants) map `ORDER BY` to **output column indices** after aggregate projection when the sort key is a plain column; expression order-by on grouped queries follows the same binding pipeline where supported.

`SchemaRowLocationComparer` evaluates expression keys per compare:

```csharp
// src/RainDB.Query/Execution/Sorting/SchemaRowLocationComparer.cs — Compare (excerpt)
var c = spec.Int32SortExpression is { } i32
    ? CompareInt32Expression(i32, x, y)
    : spec.Float64SortExpression is { } f64
        ? CompareFloat64Expression(f64, x, y)
        : CompareAtColumn(spec.ColumnIndex, x, y);
```

Null expression results sort NULL-first like column nulls.

## 12.9 SortTopNOperator table path

`ExecuteTableAsync`:

1. Resolve subquery filters (`IN` / `EXISTS`); deny-all → empty result.
2. `CollectFilteredRows` (or correlated variant) → `RowLocation[]`.
3. `_deps.SortTopNSelection.SelectInSortOrder` — sort / top-K selection.
4. `MaterializeRows` gather into one `ColumnarBatch`.

`RowLocation` (`BatchIndex`, `RowIndex`) replaces the older `RowLoc` name in the sort subsystem — same indirect addressing pattern.

## 12.10 SortTopNRowSelector and BoundedTopKHeap

`SortTopNRowSelector` (`src/RainDB.Query/Execution/Sorting/SortTopNRowSelector.cs`) centralizes ordering logic:

| Condition | Behavior |
|-----------|----------|
| No sort keys | `TruncateWithoutSort` — apply `LIMIT` only |
| Sort keys, no limit | `Array.Sort` entire `RowLocation[]` |
| Sort keys + limit, k == n | Full sort |
| Sort keys + limit, k < n | **`BoundedTopKHeap.Select`** then sort the k survivors |

```csharp
// SortTopNRowSelector.SelectInSortOrder (excerpt)
var comparer = new SchemaRowLocationComparer(schema, sortKeys, batches, _selection);
if (limit is not { } k) { Array.Sort(rows, comparer); return rows; }
k = Math.Min(k, rows.Length);
if (k == rows.Length) { Array.Sort(rows, comparer); return rows; }
var top = _heap.Select(rows, k, comparer);
Array.Sort(top, comparer);
return top;
```

### BoundedTopKHeap

`BoundedTopKHeap` (`src/RainDB.Query/Execution/Sorting/BoundedTopKHeap.cs`) retains the **k best** rows per sort order using a **max-heap of size k**:

- Root holds the **worst** row among the current top-k (largest per comparer).
- Scan each candidate; skip if not better than root; else replace root and sift down.
- Time **O(n log k)**, memory **O(k)** row locations.
- Returns unsorted heap contents; `SortTopNRowSelector` runs a final `Array.Sort` on k elements for output order.

When `LIMIT` is large relative to row count, the selector falls back to full sort — no heap overhead.

Tests in `tests/RainDB.Tests/SortTopNHeapTests.cs` exercise ascending/descending integer keys through `SchemaRowLocationComparer`.

## 12.11 SchemaRowLocationComparer

Shared comparer for:

- Multi-column keys with per-key `Descending` flip
- NULL-first column compares on `Int32`, `Int64`, `Float64`, `Boolean`, `Utf8` (byte `SequenceCompareTo`)
- Bound expression compares via `TryGetInt32` / `TryGetFloat64` on row batches

Used by both full sort and heap selection so **ORDER BY** semantics stay consistent.

## 12.12 Join and grouped execution paths

**`ExecuteJoinAsync`** (`JoinSortTopNPhysicalPlan`): `_join.ExecuteAsync` materializes join batches, collects all `RowLocation`s, then `SelectInSortOrder`. Join cost is **not** reduced by `LIMIT` — every match is produced first.

**Grouped paths:** `ExecuteGroupedSortTopNAsync` / `ExecuteGroupedJoinSortTopNAsync` run hash aggregate (or grouped join) completely, then sort the grouped result batch with the same `SortTopNRowSelector` pipeline.

## 12.13 MaterializeRows and validation

`MaterializeRows` gathers fixed-width and UTF-8 columns into a single output batch. `ValidateSortKeys` rejects unsupported types and out-of-range indices.

`Limit < 1` is rejected at plan construction.

## 12.14 SQL compilation mapping

| SQL shape | Physical plan |
|-----------|----------------|
| `ORDER BY` / `LIMIT` on one table | `SortTopNPhysicalPlan` |
| Join + sort/limit | `JoinSortTopNPhysicalPlan` |
| `GROUP BY` + sort/limit | `GroupedSortTopNPhysicalPlan` |
| Join + `GROUP BY` + sort/limit | `GroupedJoinSortTopNPhysicalPlan` |

Expression `ORDER BY` on scans binds to `Int32SortExpression` / `Float64SortExpression` in `BuildTableSortKeySpecs`.

## 12.15 Complexity and memory

Let R = qualifying rows, K = limit, C = output columns.

| Phase | Time | Extra memory |
|-------|------|--------------|
| Collect | O(R) | `RowLocation[R]` |
| Full sort | O(R log R) | in-place on array |
| Heap top-K | O(R log K) | O(K) |
| Final k-sort | O(K log K) | — |
| Materialize | O(take × C) | output buffers |

## 12.16 Comparison to hash aggregate key sort

| | `HashAggregateOperator` key sort | `SortTopNOperator` |
|--|-----------------------------------|---------------------|
| Input | `GroupKey` in hash map | `RowLocation` across batches |
| Purpose | Deterministic group order | User `ORDER BY` |
| Top-K | No | `BoundedTopKHeap` when k < R |

Both reuse `FixedWidthKeyCompare` / UTF-8 byte compare for column-shaped keys.

## 12.17 Future optimizations

External sort, `OFFSET`, LIMIT pushdown into join, and interesting-order detection remain extension points. Heap top-K for grouped plans already applies whenever `GroupedSortTopNPhysicalPlan.Limit` is set and k is smaller than the group count.

## 12.18 Summary

`SortTopNOperator` separates **ordering** from **projection**:

1. **`RowLocation`** — indirect references across batches
2. **`SortTopNRowSelector`** — full sort or `BoundedTopKHeap` when `LIMIT k` and k ≪ rows
3. **`SchemaRowLocationComparer`** — column and expression keys, NULL-first, ASC/DESC
4. **`MaterializeRows`** — gather into one batch
5. **Fused grouped plans** — `GroupedSortTopNPhysicalPlan` and `GroupedJoinSortTopNPhysicalPlan` sort aggregate output without re-scanning base tables

The bounded heap path delivers O(n log k) selection for small limits — the behavior called out in benchmark baselines after top-N heap selection landed in the engine.
