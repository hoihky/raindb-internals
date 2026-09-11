---
title: "Chapter 11: Joins"
order: 11
---

# Chapter 11: Joins

RainDB implements **inner equi-joins** in `JoinExecutionEngine` — matching rows from a probe (left) table to rows in a build (right) table where composite join keys are equal. Two algorithms are available: **hash join** (build a hash table on the right, probe from the left) and **sort-merge join** (sort both sides by key, merge matching runs). String keys route through `CompositeJoinKey`; fixed-width keys use `GroupKey`.

This chapter is split into two parts. **Part I** develops join theory: relational algebra ⋈, join types, algorithm complexity, build vs probe, equi-join vs theta join, join ordering, cardinality estimation, and semi/anti joins. **Part II** walks RainDB's `JoinPhysicalPlan`, both algorithms in both key modes, `RowRefMatch`, and `MaterializeMatches`.

---

# Part I: Theory

## 11.1 Relational algebra join (⋈)

Join combines two relations under a predicate. The general **theta join** filters a Cartesian product:

```
R ⋈_θ S = σ_θ (R × S)
```

`×` pairs every row of R with every row of S; σ keeps pairs where θ holds.

A **natural join** is the special case where θ equates every attribute that appears in both relations — rare in hand-written SQL, but the same underlying idea.

### Equi-join

When θ is a conjunction of equalities — `R.A = S.B AND R.C = S.D` — you have an **equi-join**. OLAP engines optimize this shape heavily because keys hash and partition cleanly.

Inequality predicates (`R.A > S.B`) are **theta** or **range** joins. They usually need nested loops, interval structures, or inequality-aware merge — not a simple hash table.

RainDB Phase 2 implements **inner equi-join** only: one or more column pairs, matching types, equality predicate.

### Join as composition

Plans represent join as a binary node:

```
        ⋈
       / \
   scan_R scan_S
```

Multi-table queries become **join trees** — left-deep, right-deep, or bushy. Tree shape drives intermediate row counts and memory (see §11.6).

## 11.2 Join types

SQL names several join flavors by what happens to non-matching rows:

| Join type | Keeps unmatched left? | Keeps unmatched right? | SQL syntax |
|-----------|----------------------|------------------------|------------|
| **Inner** | no | no | `INNER JOIN` / `JOIN` |
| **Left outer** | yes (NULL-padded right) | no | `LEFT JOIN` |
| **Right outer** | no | yes | `RIGHT JOIN` |
| **Full outer** | yes | yes | `FULL OUTER JOIN` |
| **Cross** | all pairs | all pairs | `CROSS JOIN` |

### Inner join semantics

Emit `(r, s)` only when the predicate evaluates to **TRUE**. Unmatched probe rows disappear.

### Outer join semantics

Preserve rows from one side even without a partner. Missing columns from the other side become **NULL**. Implementation must enumerate non-matches and widen the output schema.

RainDB ships **inner join** today — no outer variants.

### Cross join

No predicate: every left row pairs with every right row — `|R| × |S|` output. Expensive and rare in hand-written SQL; optimizers sometimes introduce it during decorrelation.

## 11.3 Nested-loop, hash join, sort-merge complexity

Three algorithms cover most join workloads.

### Nested-loop join (NLJ)

```
for each row r in R:
    for each row s in S:
        if predicate(r, s): emit (r, s)
```

| Metric | Value |
|--------|-------|
| Time | O(|R| × |S|) worst case |
| Space | O(1) beyond inputs |
| Best for | tiny inner relation, index on inner |

**Block nested-loop** reads S in cache-sized chunks. **Index nested-loop** probes a B-tree per outer row — O(|R| × log|S|).

RainDB does not use NLJ as a primary equi-join path; hash and sort-merge handle the supported cases.

### Hash join (HJ)

**Build:** load the (typically smaller) relation into a hash table keyed by join columns.

**Probe:** walk the other relation, hash each key, look up matches, emit pairs.

| Metric | Value |
|--------|-------|
| Time | O(|R| + |S|) average |
| Space | O(|S|) for hash table |
| Requirement | equi-join predicate |

When the build side exceeds RAM, **Grace hash join** partitions both inputs — same family as hash-aggregation spill.

RainDB implements in-memory hash join in `ComputeHashMatchesFixed` / `ComputeHashMatchesUtf8` with `Dictionary<Key, List<RowRef>>`.

### Sort-merge join (SMJ)

1. Sort both inputs on join keys.
2. Walk two cursors in merge fashion.
3. On equal keys, cross-product the **runs** of duplicates.

| Metric | Value |
|--------|-------|
| Time | O(|R| log|R| + |S| log|S|) |
| Space | O(|R| + |S|) for sorted entry lists |
| Best for | pre-sorted inputs, large sequential scans |

RainDB's `ComputeSortMergeMatchesFixed` flattens, sorts in memory, then merges.

### Algorithm selection summary

| Condition | Typical choice |
|-----------|----------------|
| One side small, fits in memory | Hash join |
| Both sides large, sorted / clustered on key | Sort-merge |
| Non-equi predicate | Nested-loop or specialized |
| Streaming / online | Nested-loop or merge on sorted streams |

RainDB picks the algorithm at **compile time** via `PhysicalJoinAlgorithm` — no runtime cost model yet.

## 11.4 Build vs probe side

Classic hash-join vocabulary:

| Side | Role | Typical relation |
|------|------|------------------|
| **Build** (right) | Construct hash table | Smaller relation |
| **Probe** (left) | Lookup each row | Larger relation |

Put the smaller input on the build side to shrink table size and build time.

### Probe-centric view

Some papers call the driving relation "probe" regardless of size. RainDB's plan names are fixed:

- `ProbeTableId` — left table
- `BuildTableId` — right table

The binder assigns roles today. Swapping when left ≫ right is an optimizer job (Chapter 13).

### Many-to-many matches

Duplicate keys on both sides mean one probe row can match many build rows (and vice versa). Hash join stores `List<RowRef>` per key; sort-merge expands equal-key **runs** with nested loops.

## 11.5 Equi-join vs theta join

| Type | Predicate form | RainDB support |
|------|----------------|----------------|
| Equi-join | `a = b AND c = d` | ✓ |
| Theta join | `a > b`, `a BETWEEN b AND c` | ✗ |
| Cross join | none | ✗ (explicit) |
| Semi-join | exists subquery pattern | ✗ |

Equi-join keys hash. Theta predicates generally need all-pairs comparison or specialized structures.

### Multi-column equi-join

`(a₁, a₂) = (b₁, b₂)` is one tuple equality, not two independent filters. Engines pack components into a single hashable key — RainDB uses `GroupKey` and `CompositeJoinKey`.

## 11.6 Join ordering problem

Joining `k` tables means choosing a binary tree over `k` leaves. Bushy trees explode combinatorially. **Which edges to evaluate first** and **which algorithm per edge** dominate optimizer runtime and plan quality.

### Left-deep vs bushy trees

```
Left-deep:        Bushy:
    ⋈                 ⋈
   /  \               / \
  ⋈   T3            ⋈   ⋈
 / \               / \ / \
T1 T2             T1 T2 T3 T4
```

System R–style optimizers often prefer left-deep plans: one growing intermediate on the left spine.

### Cost factors

- Cardinality of intermediate results
- Memory for hash tables
- Existing sort orders on keys
- Network shuffle cost in distributed engines

RainDB's SQL subset emits simple join plans without dynamic-programming reordering.

## 11.7 Cardinality estimation intro

Optimizers need row-count estimates at every node to rank plans.

### Basic statistics

- `n_R` — rows in R
- `n_S` — rows in S
- `d_A` — distinct values of join column A

### Simple equi-join estimate (independence assumption)

```
|R ⋈ S| ≈ |R| × |S| / max(d_A, d_S)
```

Uniform keys, independent distributions — often wrong under skew.

### Selectivity

Filters scale counts: `|σ_p(R)| ≈ sel(p) × |R|`.

RainDB Phase 1 knows exact in-memory table sizes for embedded workloads — no histogram CE yet. Zone maps and HyperLogLog sketches are plausible next steps.

### Skew and bad estimates

A few hot keys with massive fanout blow up hash buckets. **Adaptive join** re-partitions at runtime when estimates lie.

## 11.8 Semi-join and anti-join concepts

Not every join widens the schema.

### Semi-join (EXISTS)

```
R ⋉ S = { r ∈ R | ∃ s ∈ S : θ(r, s) }
```

Output keeps **R's columns** only. At most one row per `r` even if many `s` match — the pattern behind `EXISTS (SELECT 1 FROM S WHERE ...)`.

### Anti-join (NOT EXISTS)

```
R ▷ S = { r ∈ R | ¬∃ s ∈ S : θ(r, s) }
```

Rows of R with no partner in S — `NOT EXISTS` semantics.

### Implementation

Common recipes:

- **Hash semi-join** — stop at first hit in the build table
- **Mark join** — hash join plus duplicate suppression on the probe side

RainDB does not expose semi/anti joins in SQL. `INNER JOIN` materializes every matching pair; duplicate keys on both sides duplicate output rows.

### NULL in join keys

`NULL = NULL` is **UNKNOWN**, not TRUE. Any NULL component in the join key disqualifies the row:

```
(1, NULL) ⋈ (1, NULL) → no row
```

RainDB skips rows with `key.NullMask != 0` on both build and probe. Contrast with `GROUP BY`, where NULL is a legitimate group value.

---

# Part II: RainDB Implementation

## 11.9 JoinPhysicalPlan

```csharp
// src/RainDB.Query/Plans/JoinPhysicalPlan.cs
public enum PhysicalJoinAlgorithm
{
    Hash,
    SortMerge,
}

public sealed class JoinPhysicalPlan : IPhysicalPlan
{
    public PhysicalJoinAlgorithm Algorithm { get; }
    public TableId ProbeTableId { get; }      // left
    public TableId BuildTableId { get; }       // right
    public int[] ProbeKeyColumnIndices { get; }
    public int[] BuildKeyColumnIndices { get; }
    public TableSchema OutputSchema { get; }
    public JoinOutputColumnRef[]? OutputColumnOrder { get; }
    public ColumnCompareFilter[]? ProbeSideFilters { get; }
    public ColumnCompareFilter[]? BuildSideFilters { get; }
}
```

`JoinOutputColumnRef` selects output columns:

```csharp
public readonly record struct JoinOutputColumnRef(bool IsProbe, int ColumnIndex);
```

`Explain()` renders `HashJoin` or `SortMergeJoin` with key indices and optional filters.

Construction requires matching key arity: `probeKeyColumnIndices.Length == buildKeyColumnIndices.Length > 0`.

## 11.10 Engine entry point

```csharp
// src/RainDB.Query/Execution/JoinExecutionEngine.cs
public static ValueTask<IQueryResult> ExecuteAsync(
    JoinPhysicalPlan plan,
    IColumnarTableSource probeTable,
    IColumnarTableSource buildTable,
    IExecutionContext context)
{
    Validate(plan, probeTable, buildTable);

    var utf8JoinKeys = JoinKeysIncludeUtf8(probeSchema, plan.ProbeKeyColumnIndices);
    List<RowRefMatch> matches = plan.Algorithm switch
    {
        PhysicalJoinAlgorithm.Hash => utf8JoinKeys
            ? ComputeHashMatchesUtf8(plan, probeBatches, buildBatches, probeSchema, buildSchema, ct)
            : ComputeHashMatchesFixed(plan, probeBatches, buildBatches, ct),
        PhysicalJoinAlgorithm.SortMerge => utf8JoinKeys
            ? ComputeSortMergeMatchesUtf8(plan, probeBatches, buildBatches, probeSchema, buildSchema, ct)
            : ComputeSortMergeMatchesFixed(plan, probeBatches, buildBatches, probeSchema, ct),
        _ => throw new ArgumentOutOfRangeException(nameof(plan)),
    };

    var batch = MaterializeMatches(plan, probeBatches, buildBatches, probeSchema, buildSchema, matches);
    return new ValueTask<IQueryResult>(new ColumnarMaterializedQueryResult([batch]));
}
```

Synchronous after match computation — all matches in `List<RowRefMatch>`, then materialize. No streaming join output.

`DefaultQueryExecutor` resolves probe and build tables from catalog by `TableId`.

## 11.11 Internal row references

```csharp
private readonly record struct RowRef(int BatchIdx, int RowIdx);

private readonly record struct RowRefMatch(
    int LeftBatchIdx, int LeftRow,
    int RightBatchIdx, int RightRow);
```

| Type | Purpose |
|------|---------|
| `RowRef` | One row in one batch (hash table values) |
| `RowRefMatch` | One output row: paired left and right locations |

Materialization walks `RowRefMatch` once per output column.

## 11.12 Validation

```csharp
private static void Validate(JoinPhysicalPlan plan, IColumnarTableSource probe, IColumnarTableSource build)
{
    if (plan.ProbeTableId != probe.Id || plan.BuildTableId != build.Id)
        throw new ArgumentException("Physical join table ids do not match supplied tables.");

    // key indices in range
    // filter indices in range per side
    // output schema non-empty
    // output column order length matches schema

    for (var i = 0; i < plan.ProbeKeyColumnIndices.Length; i++)
    {
        var pt = probe.Schema.Columns[plan.ProbeKeyColumnIndices[i]].Type;
        var bt = build.Schema.Columns[plan.BuildKeyColumnIndices[i]].Type;
        if (pt != bt)
            throw new ArgumentException($"Join key part {i} type mismatch {pt} vs {bt}.");
        if (pt != RainDbType.Utf8 && !ColumnTypeSizes.IsFixedWidth(pt))
            throw new ArgumentException($"Join key column type {pt} is not supported for equi-join.");
    }
}
```

Rules:

1. Table IDs match resolved catalog tables
2. Key types match pairwise
3. Supported types: fixed-width primitives + `Utf8`

## 11.13 NULL key semantics

Both algorithms skip rows where any join key component is NULL:

```csharp
var key = FixedWidthGroupKeyBuilder.BuildKey(batch, row, plan.BuildKeyColumnIndices, scratch);
if (key.NullMask != 0)
    continue;
```

SQL three-valued logic: null keys never match. Differs from `GROUP BY` where NULL is a valid group key.

## 11.14 Hash join: fixed-width keys

### Build phase

```csharp
private static List<RowRefMatch> ComputeHashMatchesFixed(
    JoinPhysicalPlan plan,
    IReadOnlyList<IColumnarBatch> probeBatches,
    IReadOnlyList<IColumnarBatch> buildBatches,
    CancellationToken ct)
{
    var dict = new Dictionary<GroupKey, List<RowRef>>();
    var scratch = new ulong[plan.BuildKeyColumnIndices.Length];
    for (var bi = 0; bi < buildBatches.Count; bi++)
    {
        ct.ThrowIfCancellationRequested();
        var batch = buildBatches[bi];
        for (var row = 0; row < batch.RowCount; row++)
        {
            if (!RowPassesAll(batch, plan.BuildSideFilters, row))
                continue;
            var key = FixedWidthGroupKeyBuilder.BuildKey(batch, row, plan.BuildKeyColumnIndices, scratch);
            if (key.NullMask != 0)
                continue;
            if (!dict.TryGetValue(key, out var list))
            {
                list = [];
                dict[key] = list;
            }
            list.Add(new RowRef(bi, row));
        }
    }
    // probe phase ...
}
```

### Probe phase

```csharp
var matches = new List<RowRefMatch>();
for (var bi = 0; bi < probeBatches.Count; bi++)
{
    var batch = probeBatches[bi];
    for (var row = 0; row < batch.RowCount; row++)
    {
        if (!RowPassesAll(batch, plan.ProbeSideFilters, row))
            continue;
        var key = FixedWidthGroupKeyBuilder.BuildKey(batch, row, plan.ProbeKeyColumnIndices, scratch);
        if (key.NullMask != 0)
            continue;
        if (!dict.TryGetValue(key, out var list))
            continue;
        foreach (var br in list)
            matches.Add(new RowRefMatch(bi, row, br.BatchIdx, br.RowIdx));
    }
}
return matches;
```

### RowPassesAll

```csharp
private static bool RowPassesAll(IColumnarBatch batch, ColumnCompareFilter[]? filters, int row)
{
    if (filters is null || filters.Length == 0)
        return true;
    foreach (var f in filters)
    {
        if (!SelectionEvaluator.RowMatchesFilter(batch.Columns[f.ColumnIndex], f, row))
            return false;
    }
    return true;
}
```

Side filters are conjunctive (`AND`).

## 11.15 Hash join: UTF-8 keys

`ComputeHashMatchesUtf8` uses `Dictionary<CompositeJoinKey, List<RowRef>>` and `CompositeJoinKeyBuilder.Build(schema, batch, row, keyIndices)`.

Schema required to distinguish numeric vs UTF-8 components. Equality via `CompositeJoinKey.Equals` — byte `SequenceEqual` on payloads.

Memory: each key owns copied UTF-8 bytes — significant for high-cardinality string join keys on large build tables.

## 11.16 Sort-merge join: fixed-width keys

```csharp
private static List<RowRefMatch> ComputeSortMergeMatchesFixed(
    JoinPhysicalPlan plan,
    IReadOnlyList<IColumnarBatch> probeBatches,
    IReadOnlyList<IColumnarBatch> buildBatches,
    TableSchema probeSchema,
    CancellationToken ct)
{
    var comparer = new GroupKeyComparer(probeSchema, plan.ProbeKeyColumnIndices);
    var left = FlattenNonNullFixedKeys(probeBatches, plan.ProbeKeyColumnIndices, plan.ProbeSideFilters, ct);
    var right = FlattenNonNullFixedKeys(buildBatches, plan.BuildKeyColumnIndices, plan.BuildSideFilters, ct);

    left.Sort((a, b) => comparer.Compare(a.Key, b.Key));
    right.Sort((a, b) => comparer.Compare(a.Key, b.Key));

    return MergeSortedKeyRuns(left, right, comparer, ct);
}
```

`SortEntryFixed` carries key + `(BatchIdx, RowIdx)`.

### Merge algorithm

```csharp
while (i < left.Count && j < right.Count)
{
    var c = comparer.Compare(left[i].Key, right[j].Key);
    if (c < 0) { i++; continue; }
    if (c > 0) { j++; continue; }

    // equal keys: expand runs
    var iStart = i;
    while (i < left.Count && comparer.Compare(left[i].Key, left[iStart].Key) == 0)
        i++;
    var jStart = j;
    while (j < right.Count && comparer.Compare(right[j].Key, right[jStart].Key) == 0)
        j++;

    for (var ii = iStart; ii < i; ii++)
        for (var jj = jStart; jj < j; jj++)
            matches.Add(new RowRefMatch(
                left[ii].BatchIdx, left[ii].RowIdx,
                right[jj].BatchIdx, right[jj].RowIdx));
}
```

Run expansion produces same semantics as hash join nested loop over `List<RowRef>`.

## 11.17 Sort-merge: UTF-8 keys

`ComputeSortMergeMatchesUtf8` uses `CompositeJoinKeyComparer`, `SortEntryUtf8`, and `MergeSortedCompositeRuns` — parallel structure to fixed-width merge.

`FlattenNonNullCompositeKeys` requires `TableSchema` for `CompositeJoinKeyBuilder.Build`.

## 11.18 MaterializeMatches

```csharp
private static ColumnarBatch MaterializeMatches(
    JoinPhysicalPlan plan,
    IReadOnlyList<IColumnarBatch> probeBatches,
    IReadOnlyList<IColumnarBatch> buildBatches,
    TableSchema probeSchema,
    TableSchema buildSchema,
    List<RowRefMatch> matches)
{
    var n = matches.Count;
    if (n == 0)
    {
        var empty = new IColumnChunk[totalCols];
        for (var c = 0; c < totalCols; c++)
            empty[c] = EmptyColumnChunk(outSchema.Columns[c].Type);
        return new ColumnarBatch(0, empty);
    }
    // materialize columns ...
}
```

### Default column order

When `OutputColumnOrder` is null: all probe columns, then all build columns.

### Explicit column order

```csharp
for (var ocol = 0; ocol < order.Length; ocol++)
{
    var r = order[ocol];
    cols[ocol] = r.IsProbe
        ? MaterializeOneColumn(probeBatches, typ, r.ColumnIndex, matches, useProbeSide: true)
        : MaterializeOneColumn(buildBatches, typ, r.ColumnIndex, matches, useProbeSide: false);
}
```

Supports `SELECT a.x, b.y, a.z FROM a JOIN b ...`.

## 11.19 MaterializeOneColumn

Fixed-width gather:

```csharp
for (var o = 0; o < n; o++)
{
    var m = matches[o];
    var bi = useProbeSide ? m.LeftBatchIdx : m.RightBatchIdx;
    var ri = useProbeSide ? m.LeftRow : m.RightRow;
    var col = batches[bi].Columns[colIndex];
    if (SelectionEvaluator.IsNull(srcNb, ri, col.HasNulls))
    {
        anyNull = true;
        SetNullBit(outNb.AsSpan(), o);
        continue;
    }
    col.Values.Span.Slice(ri * w, w).CopyTo(outVals.AsSpan(o * w, w));
}
```

UTF-8: `MaterializeUtf8Column` builds offset + blob layout. `ReadUtf8Payload` handles `Utf8ColumnChunk` and `Utf8LengthPrefixedColumnChunk`.

## 11.20 Hash vs sort-merge comparison

| Aspect | Hash join | Sort-merge join |
|--------|-----------|-----------------|
| Build structure | `Dictionary<Key, List<RowRef>>` | Sorted `List<SortEntry>` per side |
| Probe | O(1) average lookup | Two-pointer O(n+m) |
| Duplicates | List per key | Run expansion |
| Memory | Hash table + lists | Two full entry lists |
| Match order | Undefined (probe order) | Sort order |

Both produce identical match **sets** for inner equi-join; ordering may differ.

## 11.21 Integration with sort and grouped join

`SortTopNEngine.ExecuteJoinAsync` (Chapter 12) calls join first, then sorts materialized output — join engine does not sort.

`GroupedJoinPhysicalPlan` composes join + `HashAggregateEngine` via `EphemeralColumnarTableSource` — join output fed directly into grouped aggregation.

## 11.22 FixedWidthKeyCompare

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

Float comparison decodes IEEE bits — consistent with hash equality and sort order.

## 11.23 Cancellation

Build, probe, flatten, and merge loops call `ct.ThrowIfCancellationRequested()`. Materialization does not check per row — matches already computed.

## 11.24 Example trace

Tables:

- `orders(order_id, customer_id)` — probe
- `customers(customer_id, name)` — build

Plan: hash join on `customer_id`, no filters.

```
Build hash:
  customer_id=1 → [RowRef(0,0), RowRef(0,5)]
  customer_id=2 → [RowRef(0,1)]

Probe orders:
  order (cust=1) → 2 matches
  order (cust=3) → no match
  order (cust=2) → 1 match

matches = [
  RowRefMatch(0, 3, 0, 0),
  RowRefMatch(0, 3, 0, 5),
  RowRefMatch(0, 7, 0, 1),
]

MaterializeMatches → 3-row ColumnarBatch
```

## 11.25 Limitations

| Limitation | Notes |
|------------|-------|
| Inner join only | No outer variants |
| Equi-join only | No range joins |
| In-memory | Full match list resident |
| No bloom filter | Every probe hits hash table |
| No runtime algorithm pick | Plan-time `PhysicalJoinAlgorithm` |
| Match order | Undefined for hash join |

## 11.26 SQL compilation and samples

The SQL compiler (`DefaultSqlCompiler` in `src/RainDB.Sql/Compilation/`) binds `INNER JOIN` syntax to `JoinPhysicalPlan` or nested `JoinSortTopNPhysicalPlan` / `GroupedJoinPhysicalPlan` when `ORDER BY` or `GROUP BY` follow.

Sample SQL in `samples/sql/07_join_select_list.sql` demonstrates explicit output column ordering — the binder emits `JoinOutputColumnRef[]` matching the `SELECT` list rather than default left-then-right layout.

`LogicalInnerJoin` in `src/RainDB.Abstractions/Logical/LogicalInnerJoin.cs` is the logical plan node; `LogicalJoinBinder` resolves table aliases and key column names to schema indices before physical plan construction.

Tests in `tests/RainDB.Tests/JoinExecutionTests.cs` and `PhysicalPlanCorrectnessTests.cs` verify hash and sort-merge paths produce identical row sets for inner equi-join on fixed-width and UTF-8 keys.

## 11.27 Summary

`JoinExecutionEngine` implements a complete inner equi-join pipeline:

1. **Validate** table IDs, key types, filters, output schema
2. **Match** via hash or sort-merge, fixed or UTF-8 keys
3. **Skip** null join keys per SQL semantics
4. **Materialize** wide columnar output by gathering from source batches

`RowRef` and `RowRefMatch` decouple matching from projection — the same match list feeds arbitrary output column orderings without re-running the join.
