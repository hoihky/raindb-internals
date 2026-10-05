---
title: "Chapter 17: Advanced SQL and Analytics"
order: 17
---

# Chapter 17: Advanced SQL and Analytics

Beyond scans, joins, and simple aggregates, analytical SQL adds expression evaluation, set operations, subqueries, and deduplication. These features interact with the compiler (binding and rewriting), the catalog (name resolution across nested scopes), and the executor (materialization versus per-row apply). This chapter explains the relational ideas first, then shows how RainDB implements Phase B planning hooks and Phase D analytics SQL on top of the vectorized stack from earlier chapters.

---

# Part I — Concepts

## 17.1 Prepared statements and plan reuse

Every ad hoc query pays the full compile cost: tokenize, parse, bind to the catalog, rewrite the logical plan, and lower to physical operators. **Prepared statements** amortize that work. The client sends SQL once with **parameter placeholders**; the engine stores a compiled plan keyed by normalized text and a **catalog fingerprint** (schema versions of referenced tables). Later executions only bind new literal values into slots already carved out in the physical plan.

Invalidation is catalog-driven: when a table bumps `SchemaVersion`, any cached entry that referenced that `TableId` must be dropped so column ordinals and types stay consistent. Parameters in RainDB appear as `@name` in `WHERE` predicates today; the binder replaces them before execution without re-parsing the statement.

Prepared execution is not a separate runtime — it is the same `IPhysicalPlan` tree the executor already understands, plus a cache and a parameter-binding pass.

## 17.2 Rule-based optimization (without a cost model)

A **cost-based optimizer** ranks plans using statistics. A **rule-based optimizer** applies deterministic rewrite rules that preserve semantics but shrink work: push filters toward scans, drop unused columns from intermediate results, validate that `LIMIT` is legal on the shaped plan, and partition join predicates into equi-join keys versus residual filters.

Rules run in a fixed order over **logical IR**, before physical algorithm selection. Each rule is small and testable: given a logical subtree, emit an equivalent subtree. RainDB's pipeline does not yet estimate cardinalities; join hash versus sort-merge is chosen by **heuristics** (row-count ratio), not by a cost formula. That split — rewrite at the logical layer, heuristics at physical bind — is a common stepping stone toward full CBO.

## 17.3 Subqueries: correlated and uncorrelated

A **subquery** is a nested `SELECT` used as a value, a table source, or a boolean filter.

**Uncorrelated** subqueries do not reference columns from an outer query. The engine can evaluate them once, materialize the result (often a single column or an empty/non-empty flag), and reuse that snapshot for every outer row. `WHERE col IN (SELECT …)` and `WHERE EXISTS (SELECT …)` are classic uncorrelated forms when the inner query is self-contained.

**Correlated** subqueries reference outer row values (e.g. `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)`). Semantically the inner query must be re-evaluated **per outer row** (or per join match), because the correlation predicate changes with each outer binding. Engines implement this with nested-loop **apply**: for each outer row, run the inner plan with extra equality filters wired from outer cells.

Correlated evaluation is expensive; uncorrelated forms should be detected and **decorrelated** or executed once when possible. RainDB resolves uncorrelated `IN`/`EXISTS` before the main scan; correlated forms on single-table scans use per-row apply. Correlated subqueries that must see both sides of a join as outer context remain a documented gap.

## 17.4 UNION versus UNION ALL

`UNION ALL` **concatenates** result sets from two or more branches with compatible column types and names. Duplicate rows from different branches are all retained; no global deduplication runs.

`UNION` (without `ALL`) asks for the **set union**: duplicate rows across the combined stream appear once. Implementations typically concatenate branches (`UNION ALL`) and then run a **distinct** operator on the full row shape, or use hash-based deduplication while merging.

Schema checks at compile time ensure branch outputs align column count and types; runtime concatenation preserves batch order from each branch in sequence.

## 17.5 DISTINCT and COUNT(DISTINCT)

`SELECT DISTINCT` removes duplicate **rows** from the projection — equality is defined on all selected columns, including null handling per type.

`COUNT(DISTINCT col)` counts unique non-null values of one column within each aggregation group (or globally). Hash aggregation can maintain a per-group set or a single global set for one distinct column; multi-column `COUNT(DISTINCT …)` generalizes to composite keys.

Full-row `DISTINCT` after large joins can be memory-heavy because it materializes unique signatures; OLAP engines often prefer grouping keys explicitly when the user knows the grain.

## 17.6 Outer join semantics

**Inner join** keeps only matching row pairs. **Outer joins** retain non-matching rows from one or both sides, padding missing columns with **NULL**.

- **LEFT OUTER**: all probe-side rows; unmatched build columns are null.
- **RIGHT OUTER**: symmetric to left; many engines normalize RIGHT to LEFT by swapping inputs.
- **FULL OUTER**: probe-only rows, build-only rows, and matches — with null padding on the opposite side when a key appears on only one input.

Equi-join predicates (equality on key columns) drive hash or sort-merge match logic; outer semantics add **null-extended** output rows for keys with no partner. Full outer join is implemented as the union of left-outer probe rows, left-outer build rows (after swap), and inner matches, with careful deduplication of the pure match set where algorithms overlap.

## 17.7 Overlay catalogs and derived tables

`FROM (SELECT …) AS alias` introduces a **derived table**: a transient relation with its own column list. The compiler must:

1. Compile the subquery to a physical plan.
2. **Materialize** its batches at runtime.
3. Register the result under a stable **ephemeral** `TableId` and alias.
4. Re-bind the outer `FROM` clause against a catalog that **overlays** the ephemeral table on the base catalog.

An **overlay catalog** delegates lookups: ephemeral names and ids resolve locally; everything else falls through to the persistent catalog. Outer plans (joins, filters, sorts on the derived table) then use the same scan and join binders as base tables. This avoids special-case "subquery scan" operators in every binder while keeping name resolution uniform.

---

# Part II — RainDB

Phase D (analytics SQL) and Phase B (planning) extend the pipeline described in Chapter 13. Compilation still flows through `SqlCompilationService`: parse → `LogicalRewritePipeline` → physical bind with `HeuristicJoinAlgorithmSelector` → `DefaultQueryExecutor`. New logical nodes (`LogicalUnionAll`, `LogicalDerivedTableScan`, scalar expressions in filters and projections) lower to physical plans executed by dedicated engines or executor branches.

## 17.8 Scalar expressions (Phase D1)

`ScalarExpressionBindingPipeline` binds arithmetic on `Int32` and `Float64`, `CAST`, `CASE`, and comparisons. Bound expressions appear in:

- `WHERE` — combined with existing column compare filters and `AND` chains.
- `SELECT` — computed output columns gathered after scan or join.
- `ORDER BY` — sort keys that are expressions, not only column ordinals.

The binder produces physical evaluators reused by `SelectionEvaluator`, projection gather paths, and sort comparators. Unsupported types or shapes fail at compile time with `SqlCompileException`.

Example scripts in the repository's `samples/sql` folder (`08_select_expressions.sql`, `09_case_expression.sql`) exercise projection and conditional logic.

## 17.9 HAVING and broader MIN/MAX (Phase D2)

Post-aggregate filtering uses `GroupedHavingBinder` and `GroupHavingEvaluator` on hash aggregation output. `MIN` and `MAX` extend to `Int32`, `Int64`, `Float64`, and `Utf8` group aggregates, not only `Float64` globals.

Samples: `10_having_aggregate_filter.sql`, `11_min_max_by_type.sql`. Automated tests live in `PhaseD1D2Tests.cs`.

## 17.10 LEFT JOIN and UNION ALL (Phase D3)

`LogicalJoinSemantics` on equi-joins marks inner, left, right, or full outer behavior. Hash and sort-merge join engines null-pad the non-preserved side when a key has no match. RIGHT outer joins are normalized to LEFT with swapped inputs during planning.

`LogicalUnionAll` lowers through `LogicalUnionAllBinder` to `UnionAllPhysicalPlan`, which requires at least two child plans and identical output schemas:

```csharp
// src/RainDB.Query/Plans/UnionAllPhysicalPlan.cs
public sealed class UnionAllPhysicalPlan : IPhysicalPlan
{
    public UnionAllPhysicalPlan(IPhysicalPlan[] inputs, TableSchema outputSchema) { ... }
    public IPhysicalPlan[] Inputs { get; }
    public TableSchema OutputSchema { get; }
}
```

`DefaultQueryExecutor` runs each branch, appends all batches into one `ColumnarMaterializedQueryResult`, and preserves branch order.

Samples: `12_left_join.sql`, `13_union_all.sql`. Tests: `PhaseD3Tests.cs`.

## 17.11 Uncorrelated subqueries and derived tables (Phase D4)

### SubqueryFilterResolver

`WHERE` predicates `IN`, `NOT IN`, `EXISTS`, and `NOT EXISTS` attach **physical specs** to scan, join, aggregate, and sort plans. Before the main operator runs, `SubqueryFilterResolver`:

- Detects **uncorrelated** specs (empty correlation list).
- Executes the nested plan once via the nested executor.
- For `IN`, builds a `ScalarValueSet` and replaces the spec with a `ColumnInSetFilter` on the outer column.
- For `EXISTS`, if the inner result is empty (respecting `NOT EXISTS`), short-circuits the whole query to no rows when appropriate.

Correlated specs are left in the plan for per-row handling (Phase D5 on scans).

```text
Compile:  VectorizedScanPhysicalPlan + InSubqueries / ExistsSubqueries
Execute:  SubqueryFilterResolver.ResolveAsync → resolved IN sets + leftover correlated specs
          → VectorizedScanEngine applies column filters and correlated apply
```

Uncorrelated subqueries work on **table scans and joins** (join resolver mirrors the scan path). Samples: `14_subquery_in_exists.sql`.

### DerivedTableScanPhysicalPlan and OverlayCatalog

`FROM (SELECT …) alias` becomes `LogicalDerivedTableScan` → `DerivedTableScanPhysicalPlan`:

```csharp
// src/RainDB.Query/Plans/DerivedTableScanPhysicalPlan.cs
public sealed class DerivedTableScanPhysicalPlan : IPhysicalPlan
{
    public IPhysicalPlan Subquery { get; }
    public TableId EphemeralTableId { get; }
    public string Alias { get; }
    public TableSchema DerivedSchema { get; }
    public IPhysicalPlan OuterPlan { get; }
}
```

Execution in `DefaultQueryExecutor.ExecuteDerivedTableAsync`:

1. Run `Subquery` and collect batches.
2. Wrap them in `EphemeralColumnarTableSource` with a fresh `TableId`.
3. Build `OverlayCatalog` over `context.Catalog` so `TryGetTable` resolves the alias.
4. Run `OuterPlan` under a scoped `RainDbExecutionContext` pointing at the overlay.

`OverlayCatalog` in `RainDB.Core/Catalog/OverlayCatalog.cs` shadows names and ids from overlays, then delegates to the base catalog. `Register` still mutates the base catalog only — derived tables are never persisted.

Sample: `15_derived_table.sql`. Tests: `PhaseD4Tests.cs`.

## 17.12 DISTINCT, UNION, outer joins, correlated apply, grouped sort (Phase D5)

### DistinctPhysicalPlan and DistinctEngine

`SELECT DISTINCT` and `UNION` (deduplicating) wrap a child plan in `DistinctPhysicalPlan`. `DistinctOperator` materializes the child, walks every row, and tracks **row signatures** in a `HashSet`. Unseen signatures are buffered and emitted as new columnar batches.

```csharp
// src/RainDB.Query/Plans/DistinctPhysicalPlan.cs
public sealed class DistinctPhysicalPlan : IPhysicalPlan
{
    public IPhysicalPlan Input { get; }
    public TableSchema OutputSchema { get; }
    public string Explain(string indent = "") =>
        $"{indent}Distinct\n{Input.Explain(indent + "  ")}";
}
```

`LogicalUnionAllBinder` stacks `UnionAllPhysicalPlan` and optionally wraps `DistinctPhysicalPlan` when the SQL keyword is `UNION` without `ALL`.

`COUNT(DISTINCT col)` is handled in hash aggregation via distinct counting on the aggregate column. Samples: `19_select_distinct_region.sql`, `17_union_distinct.sql`, `20_count_distinct_group.sql`.

### FULL OUTER and RIGHT joins

RIGHT and FULL equi-joins extend the same join engines with semantics described in §17.6. Sample: `18_full_outer_join.sql`.

### CorrelatedSubqueryExecutor

When correlation bindings are present, `SubqueryFilterResolver` does not materialize the inner query globally. Instead `CorrelatedSubqueryExecutor` runs an **apply** loop:

- For each outer row (or join match), merge correlation equalities into a copy of the inner `VectorizedScanPhysicalPlan` filters.
- Execute the inner scan; for `EXISTS`, any row suffices; for `IN`, build a value set and test membership of the outer column.

Inner plans for correlated subqueries are currently restricted to **single-table scans** — nested joins inside the correlated subquery are not supported. Join outer rows pass probe/build batch coordinates through `CorrelatedOuterRow`.

Correlated `EXISTS` / `IN` on **joins** as outer context is still unsupported; only single-table scan outers and join-match paths wired in `JoinExecutionEngine` are implemented.

Sample: `21_correlated_not_exists.sql`, `16_d5_analytics.sql`. Tests: `PhaseD5Tests.cs`.

### GROUP BY with ORDER BY and LIMIT

`GroupedSortTopNPhysicalPlan` runs hash aggregation, materializes grouped output into an ephemeral table (again via `OverlayCatalog`), then applies `SortTopNPhysicalPlan` on the grouped result. This reuses sort infrastructure from Chapter 12 without duplicating grouped sort logic inside the aggregate engine.

## 17.13 Planning and introspection (Phase B1–B4)

### B1 — LogicalRewritePipeline

`SqlCompilationService.Optimize` runs `LogicalRewritePipeline` with default rules: join predicate partition, projection pruning on scans and joins, limit validation. Rules implement `ILogicalRewriteRule` and apply in order (single pass today).

### B2 — HeuristicJoinAlgorithmSelector

When lowering `LogicalInnerJoin`, `HeuristicJoinAlgorithmSelector` picks `PhysicalJoinAlgorithm.Hash` or `SortMerge` from estimated row counts. Physical `EXPLAIN` output includes the chosen algorithm on `JoinPhysicalPlan`.

### B3 — Prepared SQL and CompiledSqlCache

`ISqlCompiler.PrepareAsync` compiles SQL with `@param` placeholders. `CompiledSqlCache` keys entries by normalized SQL and `CatalogSchemaFingerprint`; `MemoryTable.SchemaVersion` changes invalidate affected tables. `DefaultSqlCompiler` stores non-parameterized plans in the cache after first compile.

### B4 — EXPLAIN bundle

SQL statements `EXPLAIN`, `EXPLAIN LOGICAL`, and `EXPLAIN PHYSICAL` set `LogicalPlan.ExplainLevel`. Instead of executing data operators, compilation returns `ExplainBundlePhysicalPlan` with formatted logical and physical text. `DefaultQueryExecutor` maps that plan to `ExplainTextQueryResult`.

```csharp
// src/RainDB.Query/Plans/ExplainBundlePhysicalPlan.cs
public sealed class ExplainBundlePhysicalPlan : IPhysicalPlan
{
    public string LogicalText { get; }
    public string PhysicalText { get; }
    public SqlExplainLevel Level { get; }
}
```

`EXPLAIN ANALYZE` with per-operator timers is not implemented; see Chapter 15 for roadmap placement.

## 17.14 End-to-end compile and execute flow

```text
ExecuteSqlAsync / PrepareAsync
  → SqlParser → LogicalPlan
  → LogicalRewritePipeline.Optimize
  → LogicalPlanCompiler + HeuristicJoinAlgorithmSelector → IPhysicalPlan
       (may be ExplainBundlePhysicalPlan)
  → DefaultQueryExecutor
       ├─ SubqueryFilterResolver (uncorrelated IN/EXISTS)
       ├─ VectorizedScan / Join / HashAggregate / SortTopN / …
       ├─ UnionAllPhysicalPlan (concat batches)
       ├─ DistinctPhysicalPlan → DistinctOperator
       ├─ DerivedTableScanPhysicalPlan → OverlayCatalog + outer plan
       └─ CorrelatedSubqueryExecutor (per-row on scans)
```

## 17.15 Sample SQL and tests

The RainDB repository ships incremental `samples/sql` scripts numbered by milestone: expression and `CASE` (08–09), `HAVING` and typed `MIN`/`MAX` (10–11), left join and `UNION ALL` (12–13), uncorrelated subqueries (14), derived tables (15), combined D5 analytics (16), deduplicating `UNION` (17), full outer join (18), `SELECT DISTINCT` (19), `COUNT(DISTINCT)` with `GROUP BY` (20), correlated `NOT EXISTS` (21), and `GROUP BY` with `WHERE IN` (22). `RainDB.AnalyticsDemo` copies these scripts for local runs.

Read alongside:

| Area | Tests |
|------|--------|
| Scalars + HAVING | `tests/RainDB.Tests/PhaseD1D2Tests.cs` |
| Outer join + UNION ALL | `PhaseD3Tests.cs` |
| Subqueries + derived tables | `PhaseD4Tests.cs` |
| DISTINCT, UNION, correlated, grouped sort | `PhaseD5Tests.cs` |
| Rewrite, prepare, EXPLAIN | `PhaseBSqlPlanningTests.cs` |

## 17.16 Limitations and related chapters

- **Correlated subqueries on join outer context** — not supported; compile or execute paths throw or reject unsupported shapes.
- **Cost-based optimization, statistics, WAL, LINQ** — roadmap items in Chapter 15.
- **Window functions** — Phase E; not in the strict SQL subset yet.
- **Spill** — hash agg may emit metrics via `ISpillWriter`; full external spill is future work.

Prior chapters supply the operators this layer composes: vectorized scan (8), aggregation (9–10), joins (11), sort/limit (12), compilation basics (13), persistence and mmap (14–16).

## 17.17 Summary

**Part I** covered prepared plans, rule-based rewrites, subquery correlation, set operations, distinct semantics, outer joins, and overlay catalogs as relational concepts. **Part II** mapped RainDB's Phase D analytics surface — scalars, `HAVING`, outer joins, `UNION ALL`/`UNION`, uncorrelated and correlated subqueries, derived tables, and grouped sort/top-N — onto concrete physical plans (`DistinctPhysicalPlan`, `UnionAllPhysicalPlan`, `DerivedTableScanPhysicalPlan`, `ExplainBundlePhysicalPlan`) and execution helpers (`SubqueryFilterResolver`, `CorrelatedSubqueryExecutor`, `OverlayCatalog`), with Phase B planning and `EXPLAIN` integrated into the same compile path.
