---
title: "Chapter 11: Joins"
order: 11
---

# Chapter 11: Joins

RainDB implements **equi-joins** in `JoinOperator` (`src/RainDB.Query/Execution/JoinExecutionEngine.cs`) — matching rows from a probe (left) table to rows in a build (right) table where composite join keys are equal. Supported **logical** semantics include inner, left outer, right outer (normalized to left), and full outer. Two **physical** algorithms are available: **hash join** and **sort-merge join**, chosen at compile time by `HeuristicJoinAlgorithmSelector` unless overridden.

This chapter is split into two parts. **Part I** develops join theory. **Part II** covers `JoinPhysicalPlan`, `LogicalJoinSemantics`, probe-driven `ExecuteStreaming`, `JoinMatchChunkEmitter`, outer null padding, and integration with grouped aggregation.

---

# Part I: Theory

## 11.1 Relational algebra join (⋈)

Join combines two relations under a predicate. The **theta join** filters a Cartesian product:

```
R ⋈_θ S = σ_θ (R × S)
```

An **equi-join** uses conjunctions of equalities — `R.A = S.B` — the shape RainDB optimizes with hash and sort-merge algorithms.

## 11.2 Join types

| Join type | Keeps unmatched left? | Keeps unmatched right? | SQL syntax |
|-----------|----------------------|------------------------|------------|
| **Inner** | no | no | `INNER JOIN` |
| **Left outer** | yes (NULL-padded right) | no | `LEFT JOIN` |
| **Right outer** | no | yes (NULL-padded left) | `RIGHT JOIN` |
| **Full outer** | yes | yes | `FULL OUTER JOIN` |

**Inner:** emit `(r, s)` only when the predicate is TRUE.

**Outer:** preserve rows from the preserved side(s); missing columns from the other side become **NULL**.

RainDB implements all four equi-join semantics on the physical plan via `LogicalJoinSemantics` on `JoinPhysicalPlan`.

## 11.3 Nested-loop, hash join, sort-merge complexity

| Algorithm | Time (typical) | Space | Notes |
|-----------|----------------|-------|-------|
| Nested-loop | O(|R|×|S|) | O(1) | Not primary in RainDB |
| Hash join | O(|R|+|S|) avg | O(build) | Build hash on right, probe left |
| Sort-merge | O(n log n) per side | O(n) entries | Merge sorted key runs |

## 11.4 Build vs probe side

RainDB fixes naming in plans:

- `ProbeTableId` — left table (driving side in hash join)
- `BuildTableId` — right table (hash table side)

Duplicate keys on both sides produce a **many-to-many** expansion — one probe row pairs with every matching build row.

## 11.5 Equi-join vs theta join

RainDB supports multi-column equi-join with `GroupKey` (fixed-width) and `CompositeJoinKey` (UTF-8 payloads). Range and inequality joins are out of scope.

## 11.6 Join ordering and cardinality estimation

Multi-table join order and cardinality estimation affect plan quality in full optimizers. RainDB's SQL subset emits straightforward join trees; `HeuristicJoinAlgorithmSelector` uses **catalog row counts** and key shape, not histograms.

## 11.7 Semi-join and anti-join

`EXISTS` / `NOT EXISTS` patterns map to semi/anti semantics conceptually. RainDB exposes correlated subquery execution on scans and joins with dedicated filter resolution — not a separate semi-join physical operator.

## 11.8 NULL in join keys

`NULL = NULL` is **UNKNOWN**. Rows with any NULL join key component do not match in the hash or sort-merge index phases. **Outer** semantics still emit preserved-side rows when the join key is null or has no partner — via `JoinRowMatch.ProbeOnly` / `BuildOnly` (Part II).

Contrast **GROUP BY**, where NULL is a legitimate group key value.

---

# Part II: RainDB Implementation

## 11.9 LogicalJoinSemantics and physical plan

```csharp
// src/RainDB.Abstractions/Logical/LogicalJoinSemantics.cs
public enum LogicalJoinSemantics
{
    Inner,
    LeftOuter,
    RightOuter,
    FullOuter,
}
```

`JoinPhysicalPlan` (`src/RainDB.Query/Plans/JoinPhysicalPlan.cs`) carries:

- `PhysicalJoinAlgorithm Algorithm` — `Hash` or `SortMerge`
- `LogicalJoinSemantics Semantics` — default `Inner`
- Probe/build table ids, key column index arrays, output schema, optional column order, side filters, subquery specs

`LogicalJoinBinder.NormalizeRightOuterJoin` swaps left/right tables and keys so **RIGHT OUTER** becomes **LEFT OUTER** on the probe side — one outer implementation path.

## 11.10 HeuristicJoinAlgorithmSelector

`HeuristicJoinAlgorithmSelector` (`src/RainDB.Sql/Planning/HeuristicJoinAlgorithmSelector.cs`) is the default `IJoinAlgorithmSelector` wired from `SqlCompilationService`.

Selection order:

1. `PhysicalPlanningOptions.ForcedJoinAlgorithm` if set
2. `JoinAlgorithmPreference.PreferHash` or `PreferSortMerge` if set
3. **Auto:** estimate row counts from `IColumnarTableSource.Batches`; if either side is empty → hash; if `max(left,right)/min(left,right) ≤ 4` **and** the join has a **single** key column → **sort-merge**; otherwise **hash**

The chosen algorithm appears in physical `EXPLAIN` output on `JoinPhysicalPlan`.

## 11.11 JoinOperator: ExecuteAsync vs ExecuteStreaming

**Historical shape:** build a full `List<RowRefMatch>`, then materialize one wide batch.

**Current shape:** `ExecuteAsync` collects chunks by calling **`ExecuteStreaming`**, which never retains the entire match list at once.

```csharp
// src/RainDB.Query/Execution/JoinExecutionEngine.cs — JoinOperator.ExecuteAsync (conceptual)
var batches = new List<ColumnarBatch>();
ExecuteStreaming(plan, probeTable, buildTable, context, batches.Add,
    JoinMatchChunkEmitter.DefaultChunkRowCount, subFilters);
```

`ExecuteStreaming` validates tables, resolves correlated join subquery gates when needed, constructs a `JoinMatchChunkEmitter`, runs hash or sort-merge core, then `emitter.Flush()`.

### JoinMatchChunkEmitter

`JoinMatchChunkEmitter` (`src/RainDB.Query/Execution/Joining/JoinMatchChunkEmitter.cs`):

- `DefaultChunkRowCount = **8192**`
- Buffers `JoinRowMatch` records in `_pending`
- When count reaches `_chunkRowCount`, `JoinBatchMaterializer.Materialize` builds a `ColumnarBatch` and invokes the caller's `emitBatch` action
- Optional `shouldEmit` predicate filters correlated subquery matches before buffering

Peak memory scales with O(chunk rows × output width) instead of O(total matches).

## 11.12 JoinRowMatch and outer semantics

```csharp
// src/RainDB.Query/Execution/Joining/JoinRowMatch.cs
public bool HasLeft => LeftBatchIdx >= 0;
public bool HasRight => RightBatchIdx >= 0;
public static JoinRowMatch ProbeOnly(int leftBatchIdx, int leftRow) => new(leftBatchIdx, leftRow, -1, -1);
public static JoinRowMatch BuildOnly(int rightBatchIdx, int rightRow) => new(-1, -1, rightBatchIdx, rightRow);
```

Helper predicates on `JoinOperator`:

- `PreserveUnmatchedProbe` — `LeftOuter` or `FullOuter`
- `PreserveUnmatchedBuild` — `FullOuter` only

### Hash join probe logic

`EmitHashProbeMatchesFixed` / `Utf8`:

- NULL probe key → if left/full outer, emit `ProbeOnly`; else skip
- No build matches → if left/full outer, emit `ProbeOnly`; else skip
- Matches → emit full `JoinRowMatch` pairs; track matched build rows when `FullOuter`

After probing, `EmitUnmatchedBuildRows` emits `BuildOnly` for build rows never matched — **full outer** build preservation.

Sort-merge paths mirror the same semantics in `MergeSortedKeyRuns` / `MergeSortedCompositeRuns` — emitting probe-only, build-only, or paired matches when runs do not align.

### Null padding in materialization

`JoinBatchMaterializer` (`src/RainDB.Query/Execution/Joining/JoinBatchMaterializer.cs`) gathers columns per match. When `useProbeSide && !m.HasLeft` or `!useProbeSide && !m.HasRight`, it sets the output null bit and skips copying source bytes — SQL NULL padding for the missing side. UTF-8 columns use the same outer-null detection (`OuterProbeNullPresent` / `OuterBuildNullPresent`).

## 11.13 Hash join: fixed and UTF-8 keys

**Build:** `Dictionary<GroupKey, List<RowRef>>` or `Dictionary<CompositeJoinKey, List<RowRef>>`, skipping null keys and rows failing build-side filters.

**Probe:** for each probe row passing filters, lookup key and emit matches through the emitter.

Side filters and `IN` set filters are conjunctive per side via `RowPassesProbe` / `RowPassesBuild`.

## 11.14 Sort-merge join

Flatten non-null keys per side, sort with `GroupKeyComparer` or `CompositeJoinKeyComparer`, merge runs. Equal-key runs expand with nested loops (same cardinality as hash join). Outer semantics emit probe-only or build-only rows when one side exhausts before the other.

## 11.15 Validation

Table ids must match resolved catalog tables. Key types must match pairwise. Supported key types: fixed-width primitives plus `Utf8`.

## 11.16 Integration

| Consumer | Behavior |
|----------|----------|
| `DefaultQueryExecutor` | `_operators.Join.ExecuteAsync` |
| `SortTopNOperator.ExecuteJoinAsync` | Join then sort/limit on materialized batches |
| `GroupedJoinOperator` | `ExecuteStreaming` → incremental hash agg per chunk |

`GroupedJoinOperator` requires `JoinOperator` for the streaming entry point with subquery filter overload; it passes `DefaultChunkRowCount` explicitly.

## 11.17 Example: left outer hash join

Probe order with no matching customer:

```
EmitHashProbeMatches → JoinRowMatch.ProbeOnly(probeBatch, probeRow)
Materialize → customer columns NULL-padded in output batch chunk
```

## 11.18 Limitations

| Topic | Notes |
|-------|-------|
| Equi-join only | No range join |
| In-memory tables | No Grace partition spill in join |
| Correlated subqueries on joins | Limited; see implementation status docs |
| Match order | Hash join undefined; sort-merge follows key order |

## 11.19 Summary

`JoinOperator` delivers equi-joins with inner and outer semantics:

1. **Plan** — algorithm from `HeuristicJoinAlgorithmSelector`, semantics from `LogicalJoinSemantics` (RIGHT normalized to LEFT).
2. **Match** — hash or sort-merge, fixed or UTF-8 keys; skip null keys in indexes.
3. **Stream** — `ExecuteStreaming` + `JoinMatchChunkEmitter` at 8192 rows per batch.
4. **Pad** — `JoinRowMatch` probe-only/build-only rows → NULL columns on the absent side.
5. **Compose** — grouped aggregation consumes join chunks without a full materialized rowset.

Probe-driven streaming replaces the older pattern of allocating a complete match list before materialization, while `ExecuteAsync` remains a convenience that aggregates emitted chunks into `IColumnarQueryResult`.
