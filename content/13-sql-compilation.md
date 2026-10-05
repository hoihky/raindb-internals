---
title: "Chapter 13: SQL Compilation"
order: 13
---

# Chapter 13: SQL Compilation

SQL compilation is the bridge between declarative queries and executable plans. Every OLAP engine that accepts SQL text must transform human-readable statements into structures the runtime can execute efficiently. This chapter first explains how database compilers work in general — lexer through code generation, grammars, intermediate representations, optimization, and prepared statements — then walks RainDB's staged pipeline from parsing through physical lowering and caching.

---

# Part I — Concepts

## 13.1 Compiler phases: from text to execution

Turning SQL into runnable work is a pipeline of distinct transformations. Engines differ in detail, but the stages recur often enough to be worth naming:

```text
SQL text
  → Lexical analysis (characters → tokens)
  → Syntax analysis (tokens → parse tree / AST)
  → Semantic analysis (types, scopes, catalog binding)
  → Logical planning (relational algebra IR)
  → Optimization (rule-based or cost-based rewrites)
  → Physical planning (algorithms, layouts)
  → Code generation (interpreted tree, bytecode, or native code)
  → Execution
```

### Lexical analysis

The scanner walks the input left to right and emits **tokens**: reserved words, identifiers, numeric and string literals, comparison operators, and delimiters. SQL lexers are frequently hand-coded because token boundaries are awkward for generic generators — multi-character operators, keyword/identifier overlap, and literal edge cases all need special cases.

Whitespace and comments are skipped. Each token carries a source offset so later phases can report precise errors.

### Syntax analysis (parsing)

The parser checks whether the token stream matches a **grammar** and builds an **abstract syntax tree (AST)** whose nodes mirror language constructs (`SelectStatement`, `BinaryExpression`) without carrying parse-only scaffolding.

Implementations range from **recursive descent** (readable, easy to extend clause by clause) to **table-driven LR/GLR** parsers generated from yacc-style specs. Statement-level SQL is mostly amenable to LR parsing; dialect forks and optional clauses still push many teams toward hybrid or hand-tuned front ends.

### Binding and semantic analysis

Syntax alone does not guarantee meaning. Semantic passes ask catalog-backed questions:

- Is `sales` a known table?
- Does `amount` resolve to a column on that table?
- Can `amount` be compared to `'hello'`?
- May `region` appear in `SELECT` without a matching `GROUP BY` entry?

**Name resolution** ties identifiers to catalog objects. **Binding** stamps stable handles — column ordinals, physical types — onto tree nodes. Ambiguity and missing objects should fail at compile time rather than at execution.

### Optimization

An optimizer rewrites logical plans into semantically equivalent but cheaper shapes. Classic **rule-based** passes push predicates toward scans and prune unused columns. **Cost-based** optimizers rank alternatives with cardinality estimates drawn from table statistics. RainDB implements a **rule-based logical rewrite pipeline** and **join algorithm heuristics**; a full cost model with histograms remains future work.

### Physical planning and code generation

Physical planning picks concrete algorithms: hash versus sort-merge join, full sort versus bounded top-N, scan parallelism. **Code generation** may materialize an interpreted operator tree (RainDB's approach), emit vectorized bytecode, or lower to LLVM IR. RainDB's output is **`IPhysicalPlan`** instances consumed by **`DefaultQueryExecutor`**.

### Where RainDB sits today

RainDB runs **parse → logical optimize → physical bind → execute**, with optional **plan caching** and **prepared statements** for parameterized SQL. There is no separate bytecode tier — compiled queries become physical plan graphs ready for the operator suite.

## 13.2 Context-free grammars and SQL

In the Chomsky hierarchy, **context-free grammars (CFGs)** allow productions with a single non-terminal on the left and any mix of terminals and non-terminals on the right. SQL `SELECT` syntax in standards and textbooks is usually written this way.

A toy fragment might look like:

```text
<select_statement> ::= SELECT <select_list> FROM <table_ref> <where_clause>? <group_clause>? <order_clause>? <limit_clause>?
<where_clause>     ::= WHERE <boolean_expr>
<boolean_expr>     ::= <comparison> | <boolean_expr> AND <boolean_expr>
```

CFGs alone cannot enforce every SQL rule. Constraints such as "aggregates may appear only in grouped or fully aggregated contexts" belong in semantic analysis, not in the context-free layer.

### Ambiguity and precedence

Expression grammars need explicit precedence: `AND` typically binds tighter than `OR`; arithmetic tighter than comparisons. Generator tools encode this in precedence tables; recursive-descent code encodes it by call nesting (`ParseOr` delegates to `ParseAnd`, which delegates to `ParseComparison`).

### Why grammars matter to implementers

- A grammar document is the acceptance contract for the SQL surface.
- Regression suites often freeze **golden trees** for both valid and rejected inputs.
- New syntax usually lands as new productions first, then semantic rules, then execution support.

RainDB's accepted subset is implied by the parser and locked down by tests and **`StrictSqlSubset`** helpers.

## 13.3 AST vs intermediate representation (IR)

The parse tree and the execution plan answer different questions and should not be treated as interchangeable.

| Aspect | AST (parse tree) | IR (logical / physical plan) |
|--------|------------------|------------------------------|
| Mirrors surface syntax | Closely | No — algebra-oriented |
| Name usage | Identifiers as written | Logical: names; Physical: ordinals and ids |
| Evaluation order | Implicit | Explicit tree or DAG |
| Primary rewrite target | Rarely | Optimizer's main workspace |
| Shared across front ends | SQL-specific | Reusable by SQL, LINQ, APIs |

### AST characteristics

A node like `BinaryExpression(>, ColumnRef(amount), Literal(1.0))` records that the user wrote `amount > 1.0`. It does not, by itself, disambiguate which table owns `amount` when several are visible — binding supplies that.

### Logical IR

Logical IR states **what** to compute in relational terms: `TableScan(sales)`, `Filter(amount > 1.0)`, `HashAggregate(groupBy=[region], agg=[sum(amount)])`. Human-readable names typically remain. SQL, LINQ, and programmatic planners can all target the same logical layer.

### Physical IR

Physical IR states **how** to compute: `VectorizedScanPhysicalPlan`, `JoinPhysicalPlan`, `DerivedTableScanPhysicalPlan`, and related records. Catalog handles, algorithm choices, and execution knobs live here.

### RainDB's two IR layers

RainDB defines **logical nodes** under `RainDB.Logical` (`LogicalTableScan`, `LogicalInnerJoin`, `LogicalUnionAll`, `LogicalDerivedTableScan`, …) and **physical plans** under `RainDB.Query/Plans`. The parser emits logical IR directly (no public AST type hierarchy), which keeps the front end small; larger systems often retain an explicit AST for tooling.

## 13.4 Name resolution and binding

**Name resolution** traverses lexical scopes and maps each identifier to exactly one catalog object:

- Table aliases introduced in `FROM`
- Qualified (`sales.amount`) or bare (`amount`) column references
- Derived tables and subquery scopes

**Binding** records the resolved meaning in a form execution can use:

```text
"sales".amount  →  TableId {guid}, column index 1, RainDbType.Float64
```

Bound artifacts depend on the live catalog. Identical SQL can compile differently after a schema change. Production engines track **schema versions** or **fingerprints** so cached plans can be discarded safely.

### Common binding errors

| Error | Cause |
|-------|-------|
| Unknown table | Identifier absent from catalog |
| Unknown column | Name not in referenced table schema |
| Ambiguous column | Bare name matches multiple joined tables |
| Invalid qualifier | `WHERE other.col` when only `sales` is in scope |

RainDB rejects many ambiguous patterns during parsing or binding with **`SqlCompileException`**.

## 13.5 Semantic analysis

After names resolve, semantic passes enforce typing and structural SQL rules.

### Type checking

- Predicates need compatible operand types (or well-defined casts via **`ScalarExpressionBindingPipeline`**)
- `SUM` expects numeric operands
- Join keys must pair columns of compatible types

### Aggregate and grouping rules

Standard SQL expects every non-aggregated `SELECT` expression either to appear in `GROUP BY` or to be functionally determined by the grouping key. RainDB validates grouped select lists at parse time and in binders; **`HAVING`** filters are bound separately after aggregation.

### Set operation compatibility

`UNION` / `UNION ALL` branches must align in column count and type. The union binder checks schema compatibility before emitting **`UnionAllPhysicalPlan`** or distinct wrappers.

### Side effects and volatility

Volatile functions cannot be folded at compile time. RainDB's subset exposes no user-defined functions.

The output of semantic analysis is a **bound logical plan** (or optimized logical template for prepared SQL) eligible for physical lowering.

## 13.6 Query optimization overview

Optimization explores equivalent plans and picks one with lower estimated cost. Two broad strategies dominate practice.

### Rule-based optimization (RBO)

A pipeline of deterministic rewrites:

- **Predicate pushdown / partition** — relocate filters toward scans and join sides
- **Projection pruning** — drop columns no operator downstream reads
- **Limit validation** — ensure sort/limit combinations are legal for grouped queries

Rules are quick, predictable, and testable without statistics.

### Cost-based optimization (CBO)

Each candidate plan receives an estimated cost built from **statistics**. RainDB does not yet implement histogram-based CBO; join **algorithm** choice uses **row-count heuristics** instead.

### RainDB's position

**`LogicalRewritePipeline`** applies ordered rules (`JoinPredicatePartitionRule`, table/join projection pruning, limit pushdown validation). **`HeuristicJoinAlgorithmSelector`** chooses hash vs sort-merge join from estimated row counts and planning options. A full cost model remains on the development roadmap.

## 13.7 Prepared statements

**Prepared statements** decouple compilation from repeated execution:

1. **Prepare** — parse, optimize, and retain a logical template (and parameter metadata)
2. **Execute** — bind parameter values, lower to a physical plan, run

Benefits include:

- **Amortized compile cost** — optimize once, bind many times
- **Injection resistance** — parameters are data, not concatenated text
- **Plan stability** — execution shape stays fixed until schema drift

### Parameter binding

Placeholders such as `@min_amount` in `WHERE` are collected at prepare time. At execute time, **`LogicalParameterBinder`** substitutes values into the optimized logical plan before physical binding.

### Plan cache invalidation

Reuse is unsafe when catalog schema or table **`SchemaVersion`** changes. **`CatalogSchemaFingerprint`** hashes table ids, schema versions, and column counts for cache keys alongside SQL text.

### RainDB status

**`ISqlCompiler.PrepareAsync`** returns **`PreparedSqlStatement`**. **`DefaultSqlCompiler.CompileAsync`** uses **`CompiledSqlCache`** for parameter-free SQL keyed by text + fingerprint. Parameterized statements bypass the physical plan cache and rebind on each execute.

---

# Part II — RainDB

RainDB exposes SQL as the primary query surface for embedded analytics. The production path is **`DefaultSqlCompiler`** delegating to **`SqlCompilationService`**, which orchestrates parse, logical rewrite, join heuristics, and **`LogicalPlanCompiler`** physical binding. Tests and samples may also call **`StrictSqlSubset`**, which runs the same service stack without implementing **`ISqlCompiler`**.

## 13.8 End-to-end compilation pipeline

```text
SQL string
  → LogicalPlanCompiler.Parse (SqlParser / SelectParser)
  → LogicalPlan (ILogicalRoot + optional ExplainLevel)
  → LogicalRewritePipeline.Optimize
  → HeuristicJoinAlgorithmSelector (when root is LogicalInnerJoin)
  → LogicalPlanCompiler.CompilePhysical (binder graph)
  → IPhysicalPlan
  → DefaultQueryExecutor + IQueryOperatorSuite
```

**Facade:** **`SqlCompilationService`** owns collaborators:

| Collaborator | Role |
|--------------|------|
| `LogicalPlanCompiler` | Parse + bind logical roots to physical plans |
| `LogicalRewritePipeline` | Rule-based logical rewrites |
| `IJoinAlgorithmSelector` | Default **`HeuristicJoinAlgorithmSelector`** |
| `LogicalParameterBinder` | `@param` substitution for prepared SQL |
| `SqlExplainFormatter` | Text for EXPLAIN bundles |
| `PhysicalPlanningOptions` | Forced join algorithm, preferences |

Public entry from the engine:

```csharp
public async ValueTask<IQueryResult> ExecuteSqlAsync(string sql, CancellationToken cancellationToken = default)
{
    var ctx = CreateSessionWithExecutor(cancellationToken);
    var plan = await SqlCompiler.CompileAsync(sql, Catalog, cancellationToken).ConfigureAwait(false);
    return await Executor.ExecuteAsync(plan, ctx).ConfigureAwait(false);
}
```

## 13.9 SqlCompilationService in detail

**`Optimize`** runs **`LogicalRewritePipeline`** on a copy of the parsed plan (explain level preserved separately on the original when needed).

**`CompilePhysical`**:

1. Optimizes the logical plan.
2. Selects **`PhysicalJoinAlgorithm`** when the root is **`LogicalInnerJoin`**.
3. Calls **`LogicalPlanCompiler.CompilePhysical`** with scan options and the chosen algorithm.
4. If the parsed plan carried an **`ExplainLevel`**, wraps the executable plan in **`ExplainBundlePhysicalPlan`** with formatted logical and physical text instead of returning the executable alone.

**`CompileBoundLogical`** skips re-optimization — used after parameter binding on a prepared template.

**`ContainsParameters`** inspects the logical tree for parameter nodes so **`DefaultSqlCompiler`** can avoid incorrect caching.

Default rewrite rules (in order):

```text
JoinPredicatePartitionRule
TableScanProjectionPruningRule
JoinProjectionPruningRule
LimitPushdownValidationRule
```

## 13.10 LogicalPlanCompiler

**`LogicalPlanCompiler`** is the **physical binding hub**. It owns an instance **`SqlParser`** and composes specialized binders:

| Binder | Logical root |
|--------|----------------|
| `LogicalTableScanBinder` | `LogicalTableScan` |
| `LogicalJoinBinder` | `LogicalInnerJoin` |
| `LogicalUnionAllBinder` | `LogicalUnionAll` |
| `LogicalDerivedTableScanBinder` | `LogicalDerivedTableScan` |
| `UncorrelatedSubqueryBinder` | Used from table/join binders for `IN` / `EXISTS` |

**`CompilePhysical`** dispatches on **`ILogicalRoot`**:

```csharp
public IPhysicalPlan CompilePhysical(
    ILogicalRoot root,
    ICatalog catalog,
    VectorizedScanExecutionOptions scanOptions = default,
    PhysicalJoinAlgorithm joinAlgorithm = PhysicalJoinAlgorithm.Hash) =>
    root switch
    {
        LogicalTableScan s => _tableScanBinder.BindAndLower(s, catalog, scanOptions, joinAlgorithm),
        LogicalDerivedTableScan d => _derivedBinder.BindAndLower(d, catalog, scanOptions, joinAlgorithm),
        LogicalInnerJoin j => _joinBinder.BindAndLower(j, catalog, joinAlgorithm, scanOptions),
        LogicalUnionAll u => _unionBinder.BindAndLower(u, catalog, scanOptions, joinAlgorithm),
        _ => throw new InvalidOperationException($"Unsupported logical root {root.GetType().Name}."),
    };
```

**`Parse(string sql)`** exposes parse-only access for tests and **`DefaultSqlCompiler`**.

Lowering choices include **`VectorizedScanPhysicalPlan`**, **`HashAggregatePhysicalPlan`**, join and sort composites, **`DerivedTableScanPhysicalPlan`**, **`DistinctPhysicalPlan`**, **`UnionAllPhysicalPlan`**, and grouped sort/join plans — see Chapter 7 for execution semantics.

## 13.11 HeuristicJoinAlgorithmSelector

When **`PhysicalPlanningOptions.ForcedJoinAlgorithm`** is unset, **`SelectAuto`** estimates row counts from **`IColumnarTableSource.Batches`**:

- If either side is missing or empty → prefer **hash**.
- If max/min row ratio ≤ **4** and join keys are single-column → **sort-merge**.
- Otherwise → **hash**.

Options can force **`PreferHash`** or **`PreferSortMerge`**. The chosen algorithm is stored on **`JoinPhysicalPlan`** and appears in physical **`Explain()`** output.

## 13.12 DefaultSqlCompiler: cache and ISqlCompiler

**`DefaultSqlCompiler`** wraps **`SqlCompilationService`** plus:

- **`CatalogSchemaFingerprint`** — computes a stable **`long`** from catalog table names, ids, **`SchemaVersion`**, and column counts.
- **`CompiledSqlCache`** — concurrent dictionary keyed by `(sql, fingerprint)`.

**`CompileAsync`** flow:

1. Parse via **`_compilation.PhysicalCompiler.Parse`**.
2. If no parameters and cache hit → return cached **`IPhysicalPlan`**.
3. Else **`CompilePhysical`**.
4. Store in cache when parameter-free and result is not **`ExplainBundlePhysicalPlan`**.

**`PrepareAsync`** flow:

1. Parse SQL.
2. **`Optimize`** → store optimized logical template.
3. Collect parameter names via **`LogicalParameterBinder`**.
4. Return **`PreparedSqlStatement`** holding template + compilation service reference.

## 13.13 PreparedSqlStatement

**`PreparedSqlStatement`** implements **`IPreparedSqlStatement`**:

- **`CompileAsync(catalog, parameters)`** — bind parameters into the optimized template, then **`CompileBoundLogical`** (physical bind only).
- **`ExecuteAsync(catalog, executor, context, parameters)`** — compile + **`executor.ExecuteAsync`**.

Prepare amortizes **parse + logical optimization**; each execute pays **parameter bind + physical bind** (and benefits from current catalog state). Schema drift is handled by re-preparing or by fingerprint mismatch on the ad hoc **`CompileAsync`** cache path.

## 13.14 EXPLAIN levels and ExplainBundlePhysicalPlan

The parser attaches **`SqlExplainLevel`** to **`LogicalPlan`** when the statement begins with **`EXPLAIN`**, **`EXPLAIN LOGICAL`**, or **`EXPLAIN PHYSICAL`**.

**`SqlCompilationService`** formats optimized logical text and bound physical text through **`SqlExplainFormatter`**, then returns **`ExplainBundlePhysicalPlan`**. Execution yields **`ExplainTextQueryResult`** — not an empty row set. Explain plans are excluded from **`CompiledSqlCache`**.

## 13.15 ScalarExpressionBindingPipeline

Scalar expressions in **`WHERE`**, **`SELECT`**, and **`ORDER BY`** (where supported) are bound by **`ScalarExpressionBindingPipeline`** inside **`LogicalTableScanBinder`** (and related binders). It validates table references, infers types, and lowers arithmetic, **`CAST`**, **`CASE`**, and comparisons to row evaluators used during vectorized execution.

Strict **`SimpleWhereClause`** conjuncts remain the fast path for literal comparisons; expression predicates extend the subset documented in implementation status Phase D1+.

## 13.16 StrictSqlSubset vs LogicalPlanCompiler

| Aspect | `StrictSqlSubset` | `LogicalPlanCompiler` / `DefaultSqlCompiler` |
|--------|-------------------|-----------------------------------------------|
| Purpose | Test and sample entry without `ISqlCompiler` | Production compiler + cache + prepare |
| Parse | `Shared.Compiler.Parse` | Same parser instance on compiler |
| Optimize + bind | `new SqlCompilationService(physicalCompiler: _compiler)` | Shared service inside `DefaultSqlCompiler` |
| Caching | None | `CompiledSqlCache` + fingerprint |
| Prepare | Not exposed | `PrepareAsync` |

Static helpers **`ParseLogicalPlan`** / **`CompilePhysicalPlan`** forward to **`StrictSqlSubset.Shared`**. **`CompilePhysicalPlan`** runs the **full** optimize + bind pipeline — not parse-only binders from older Phase 1 docs.

## 13.17 Parsing layer (lexer and SelectParser)

**`SqlLexer`** tokenizes identifiers, literals, operators, and keywords. **`SelectParser`** is recursive descent, producing **`LogicalPlan`** with roots for scans, joins, unions, derived tables, explain prefixes, and expanded SQL features (distinct, having, outer joins, subqueries) as implemented in the current parser.

Errors throw **`SqlCompileException`** with source positions where available.

Detailed grammar walkthroughs for the original strict subset (conjunctive **`WHERE`**, inner join keys, grouped select validation) remain valid for core shapes; consult **`samples/sql`** and phase tests for the full feature matrix.

## 13.18 Logical → physical lowering (selected patterns)

**Single-table scan:** **`LogicalTableScanBinder`** resolves catalog entries, builds **`ColumnCompareFilter`** arrays and expression predicates, chooses **`VectorizedScanPhysicalPlan`**, **`HashAggregatePhysicalPlan`**, or **`SortTopNPhysicalPlan`** / grouped sort composites.

**Joins:** **`LogicalJoinBinder`** resolves probe/build tables, partitions **`WHERE`**, selects join algorithm argument, emits **`JoinPhysicalPlan`**, **`GroupedJoinPhysicalPlan`**, sort/join composites, or outer-join semantics per **`LogicalJoinSemantics`**.

**Derived tables:** **`LogicalDerivedTableScanBinder`** emits **`DerivedTableScanPhysicalPlan`** with inner subquery plan + outer plan — executed via overlay catalog (Chapter 7).

**Union:** **`LogicalUnionAllBinder`** builds **`UnionAllPhysicalPlan`** or distinct-wrapped variants.

Filter literals still pack into **`ImmediateBits`** / **`Utf8LiteralBytes`** for vectorized selection; expression filters use bound evaluators.

## 13.19 Worked example: parameterized compile path

SQL:

```sql
SELECT region, SUM(amount)
FROM sales
WHERE amount > @threshold
GROUP BY region
```

**Prepare:** parse → optimize → template with parameter node `@threshold`.

**Execute** with `@threshold = 1.0`:

1. **`BindParameters`** on optimized logical plan.
2. **`CompileBoundLogical`** → **`HashAggregatePhysicalPlan`** with filter immediate bits for `1.0`.
3. Executor → hash aggregate operator.

**Ad hoc `CompileAsync`** without parameters would cache the physical plan until **`CatalogSchemaFingerprint`** changes.

## 13.20 LINQ and second front ends

**`DefaultLinqCompiler`** remains a stub relative to SQL. The intended architecture is expression tree → shared logical IR → **`SqlCompilationService`** / **`LogicalPlanCompiler`**. SQL is the reference front end today.

## 13.21 Checklist: adding a new SQL feature

1. **Lexer / parser** — tokens and productions; extend logical IR if needed.
2. **Logical Explain** — update **`Explain()`** on logical nodes.
3. **Rewrite rules** — if equivalences apply, add **`ILogicalRewriteRule`** implementations.
4. **`LogicalPlanCompiler` binders** — validation + physical lowering.
5. **Physical plan + operator** — if no existing plan fits, extend **`IQueryOperatorSuite`** and executor dispatch.
6. **Tests** — parser, binder, end-to-end SQL, optimizer rule tests where relevant.
7. **Book / Programming Guide** — document the surface.

## 13.22 Summary

**Part I** covered the general SQL compilation pipeline, IR layers, optimization families, and prepared statements. **Part II** showed RainDB's current architecture: **`SqlCompilationService`** coordinates **`LogicalRewritePipeline`**, join heuristics, and **`LogicalPlanCompiler`** binding; **`DefaultSqlCompiler`** adds **`CompiledSqlCache`** and **`PrepareAsync`**; EXPLAIN SQL returns **`ExplainBundlePhysicalPlan`**; scalar expressions flow through **`ScalarExpressionBindingPipeline`**. **`StrictSqlSubset`** mirrors the production bind path for tests without the compiler interface. Extending SQL means extending this pipeline while keeping logical IR the shared contract between front ends and the vectorized executor.
