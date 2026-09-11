---
title: "Chapter 13: SQL Compilation"
order: 13
---

# Chapter 13: SQL Compilation

SQL compilation is the bridge between declarative queries and executable plans. Every OLAP engine that accepts SQL text must transform human-readable statements into structures the runtime can execute efficiently. This chapter first explains how database compilers work in general — lexer through code generation, grammars, intermediate representations, and prepared statements — then walks RainDB's staged pipeline from `SqlLexer` through physical plan lowering.

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

An optimizer rewrites logical plans into semantically equivalent but cheaper shapes. Classic **rule-based** passes push predicates toward scans and prune unused columns. **Cost-based** optimizers rank alternatives with cardinality estimates drawn from table statistics. RainDB currently skips a dedicated optimizer and lowers logical nodes directly; Chapter 15 outlines what a fuller pass would add.

### Physical planning and code generation

Physical planning picks concrete algorithms: hash versus sort-merge join, full sort versus bounded top-N, scan parallelism. **Code generation** may materialize an interpreted operator tree (RainDB's approach today), emit vectorized bytecode (common in Velox- and DuckDB-style engines), or lower to LLVM IR (ClickHouse, HyPer). Whatever form it takes, the output is what the runtime executes.

### Where RainDB sits today

RainDB runs lex → parse → bind → physical plan. There is no separate optimizer stage and no bytecode tier — compiled queries become `IPhysicalPlan` instances ready for the executor.

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

RainDB's accepted subset is implied by `SelectParser` and locked down by `StrictSqlSubset` tests.

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

Physical IR states **how** to compute: `VectorizedScanPhysicalPlan(tableId, columnIndices, filters, ...)`, `HashJoinPhysicalPlan(...)`. Catalog handles, algorithm choices, and execution knobs live here.

### RainDB's two IR layers

RainDB defines **logical nodes** under `RainDB.Abstractions/Logical` (`LogicalTableScan`, `LogicalInnerJoin`) and **physical plans** under `RainDB.Query/Plans`. There is no public AST type hierarchy — `SelectParser` emits logical IR directly. That shortcut suits a narrow grammar; larger systems usually retain an explicit AST for tooling and parse-level `EXPLAIN`.

## 13.4 Name resolution and binding

**Name resolution** traverses lexical scopes and maps each identifier to exactly one catalog object:

- Table aliases introduced in `FROM`
- Qualified (`sales.amount`) or bare (`amount`) column references
- Columns visible from enclosing scopes in correlated subqueries

Inner scopes shadow outer ones; join predicates may legally reference columns from either side.

**Binding** records the resolved meaning in a form execution can use:

```text
"sales".amount  →  TableId {guid}, column index 1, RainDbType.Float64
```

Bound artifacts depend on the live catalog. Identical SQL can compile differently after a schema change. Production engines track **schema versions** so cached plans can be discarded safely (see prepared statements below).

### Common binding errors

| Error | Cause |
|-------|-------|
| Unknown table | Identifier absent from catalog |
| Unknown column | Name not in referenced table schema |
| Ambiguous column | Bare name matches multiple joined tables |
| Invalid qualifier | `WHERE other.col` when only `sales` is in scope |

RainDB's strict subset rejects several ambiguous patterns during parsing — for example, bare column names in grouped join queries.

## 13.5 Semantic analysis

After names resolve, semantic passes enforce typing and structural SQL rules.

### Type checking

- Predicates need compatible operand types (or well-defined casts)
- `SUM` expects numeric operands
- Join keys must pair columns of compatible types

### Aggregate and grouping rules

Standard SQL expects every non-aggregated `SELECT` expression either to appear in `GROUP BY` or to be functionally determined by the grouping key (engines differ on how far they take the latter). Catching violations at compile time avoids silent wrong answers at runtime.

### Set operation compatibility

`UNION` branches must align in column count and type. `INSERT` rows must fit the destination schema.

### Side effects and volatility

Volatile functions cannot be folded at compile time. RainDB's current subset exposes no user-defined functions.

The output of semantic analysis is a **bound logical plan** eligible for optimization or direct lowering.

## 13.6 Query optimization overview

Optimization explores equivalent plans and picks one with lower estimated cost. Two broad strategies dominate practice.

### Rule-based optimization (RBO)

A pipeline (or fixpoint loop) of deterministic rewrites:

- **Predicate pushdown** — relocate filters next to scans
- **Projection pruning** — drop columns no operator downstream reads
- **Constant folding** — reduce `1 + 2` at compile time
- **Join reordering** (where semantics permit) — commute or regroup joins

Rules are quick, predictable, and testable without statistics.

### Cost-based optimization (CBO)

Each candidate plan receives an estimated cost built from **statistics**:

- Base table row counts
- Distinct-value counts (NDV) per column
- Skew-capturing histograms
- Join selectivity models

Search may use dynamic programming over join orders (System R lineage) or greedy heuristics common in large-scale OLAP. CBO pays off most when many tables join and cardinalities are skewed.

### RainDB's position

RainDB lowers logical plans with fixed physical choices — hash join by default, no predicate pushdown yet. Chapter 15 covers statistics and cost-based planning as future work.

## 13.7 Prepared statements

**Prepared statements** decouple compilation from repeated execution:

1. **Prepare** — parse, bind, and optionally optimize once; retain a plan template
2. **Execute** — bind parameter values and run the stored plan

Benefits include:

- **Amortized compile cost** — the same shape runs millions of times in OLTP-style workloads
- **Injection resistance** — parameters are data, not concatenated text
- **Plan stability** — execution shape stays fixed until statistics or schema drift

### Parameter binding

Placeholders stand in for literals: `WHERE amount > ?` supplies `1.0` at execute time. Physical filters can store values in dedicated slots (`ColumnCompareFilter.ImmediateBits`) without re-resolving column indices.

### Plan cache invalidation

Reuse is unsafe when:

- A referenced table's schema changes (columns added, dropped, or retyped)
- Statistics refresh materially alters cost estimates (in CBO engines)
- Operator semantics change across engine versions

A common pattern attaches a `SchemaVersion` to each table, records versions at prepare time, and revalidates before execute.

### RainDB status

RainDB parses and binds on every `ExecuteSqlAsync` call today. `MemoryTable.SchemaVersion` is in place for a future cache. The async `CompileAsync` signature leaves room for disk-backed catalog access and cache I/O.

---

# Part II — RainDB

RainDB exposes SQL as the primary query surface for embedded analytics. The compiler is deliberately staged: **lex** the text into tokens, **parse** tokens into logical IR that still uses table and column *names*, **bind** those names against the live catalog to produce column indices and physical types, then **lower** the bound logical plan into `IPhysicalPlan` nodes that the query executor already understands.

## 13.8 Why three stages?

OLAP engines separate parsing from binding for three practical reasons:

1. **Catalog independence.** Tests can parse SQL into `LogicalPlan` without constructing tables. The parser never touches `ICatalog`.
2. **Shared lowering.** `LogicalTableScanBinder` and `LogicalJoinBinder` are reused by `StrictSqlSubset.CompilePhysicalPlan` and `DefaultSqlCompiler` alike. A future LINQ compiler targets the same logical nodes.
3. **Fail-fast semantics.** Unsupported SQL is rejected at compile time via `SqlCompileException`, not at runtime with silent wrong answers. RainDB's strict subset prefers a hard error over guessing.

The end-to-end path from `RainDbEngine.ExecuteSqlAsync` looks like this:

```text
SQL string
  → SqlLexer + SelectParser        (RainDB.Sql/Parsing)
  → LogicalPlan / ILogicalRoot     (RainDB.Abstractions/Logical)
  → LogicalTableScanBinder or
    LogicalJoinBinder              (RainDB.Sql/Compilation)
  → IPhysicalPlan                  (RainDB.Query/Plans)
  → DefaultQueryExecutor           (RainDB.Query/Execution)
```

```csharp
// src/RainDB.Driver/RainDbEngine.cs
public async ValueTask<IQueryResult> ExecuteSqlAsync(string sql, CancellationToken cancellationToken = default)
{
    var ctx = CreateSession(cancellationToken);
    var plan = await SqlCompiler.CompileAsync(sql, Catalog, cancellationToken).ConfigureAwait(false);
    return await Executor.ExecuteAsync(plan, ctx).ConfigureAwait(false);
}
```

Compilation is synchronous inside `CompileAsync` today (it returns `ValueTask.FromResult`), but the async signature leaves room for disk-backed catalog lookups or plan-cache I/O later.

## 13.9 The strict SQL subset

`StrictSqlSubset` is both documentation and a test-friendly entry point. It exposes parse-only and parse-and-bind helpers without requiring `ISqlCompiler`:

```csharp
// src/RainDB.Sql/StrictSqlSubset.cs
public static class StrictSqlSubset
{
    public static LogicalPlan ParseLogicalPlan(string sql) => SqlParser.Parse(sql);

    public static IPhysicalPlan CompilePhysicalPlan(
        string sql,
        ICatalog catalog,
        VectorizedScanExecutionOptions scanOptions = default) =>
        CompileRoot(ParseLogicalPlan(sql).Root, catalog, scanOptions);
}
```

Supported shapes today:

| Feature | Supported | Notes |
|---------|-----------|-------|
| `SELECT *` or column list | Yes | `*` is mutually exclusive with explicit aggregates in grouped queries |
| `FROM` single table | Yes | |
| `INNER JOIN … ON` equi-keys | Yes | Keys must be `table.column = table.column`; multiple keys ANDed |
| `WHERE` | Yes | Conjunctive (`AND` only); one comparison per conjunct |
| `GROUP BY` + aggregates | Yes | `SUM`, `MIN`, `MAX`, `COUNT`, `COUNT(*)` |
| Global aggregate (no `GROUP BY`) | Yes | Exactly one aggregate in `SELECT` |
| `ORDER BY` / `LIMIT` | Partial | Only on non-grouped, non-global-aggregate scans and joins |
| `DISTINCT`, `HAVING`, subqueries | No | Parser throws `SqlCompileException` |
| `LEFT`/`RIGHT`/`FULL` join | No | Only `INNER JOIN` tokenized as keyword |

When you add SQL surface area, extend the parser first, then logical IR if needed, then binders, then executors — in that order.

## 13.10 SqlLexer: hand-written tokenization

`SqlLexer` lives in `src/RainDB.Sql/Parsing/SqlLexer.cs`. It is intentionally small: ASCII identifiers, `--` line comments, and a fixed keyword set. There is no Unicode identifier support; table and column names must be ASCII letters, digits, and underscore.

### 13.10.1 Token model

```csharp
internal enum SqlTokenKind
{
    EndOfFile,
    Identifier,
    Star,
    Comma,
    LParen,
    RParen,
    Eq, Ne, Lt, Le, Gt, Ge,
    Semicolon,
    Dot,
    StringLiteral,
    Number,
    KwSelect, KwFrom, KwWhere,
    KwInner, KwJoin, KwOn, KwAnd,
}

internal readonly record struct SqlToken(SqlTokenKind Kind, int Start, int Length);
```

Each token records **start offset and length** into the original source string. Error messages reference positions; `Lexeme` reconstructs the substring without allocating until needed:

```csharp
public ReadOnlySpan<char> Lexeme(in SqlToken t) => _src.AsSpan(t.Start, t.Length);
```

### 13.10.2 Save/restore for speculative parsing

`SelectParser.TryParseAggregationCall` uses lexer checkpointing. It advances past a function-looking identifier; if `(` does not follow, it rewinds:

```csharp
var savePos = _lexer.Save();
var saveTok = _cur;
Advance();
if (_cur.Kind != SqlTokenKind.LParen)
{
    _lexer.Restore(savePos);
    _cur = saveTok;
    return false;
}
```

This pattern avoids a separate "is this an aggregate call?" grammar production. The lexer exposes `Save()` / `Restore(int)` as thin wrappers over `_pos`.

### 13.10.3 Literals, keywords, and operators

**Strings** use single quotes with SQL-style `''` escaping; unterminated strings throw `SqlCompileException`. **Numbers** accept optional sign, integer part, and optional fraction. **Booleans** (`TRUE`/`FALSE`) are recognized in `ParseLiteral`, not in the lexer.

`ClassifyKeyword` uppercases identifiers on the stack for `SELECT`, `FROM`, `WHERE`, `INNER`, `JOIN`, `ON`, `AND`. Words like `GROUP`, `ORDER`, and `LIMIT` stay as `Identifier` tokens — `SelectParser` matches them with `LexemeEqualsIgnoreCase`.

Comparison operators: `=`, `!=`/`<>` → `Ne`, `<`, `<=`, `>`, `>=`. Line comments (`--`) are skipped; there is no block comment syntax.

## 13.11 SqlParser and SelectParser

`SqlParser.Parse` trims input, constructs `SqlLexer` and private `SelectParser`, and returns `LogicalPlan`:

```csharp
// src/RainDB.Sql/Parsing/SqlParser.cs
public static LogicalPlan Parse(string sql)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(sql);
    var trimmed = sql.Trim();
    var lexer = new SqlLexer(trimmed);
    var parser = new SelectParser(lexer, trimmed);
    return parser.Parse();
}
```

`SelectParser` is recursive descent with a single current token `_cur`. The top-level `Parse()` method encodes the grammar's major branches.

### 13.11.1 Parse order

Every statement follows:

1. `SELECT` → `ParseSelectItems`
2. `FROM` → `ParseFromClause` (table or inner join)
3. Optional `WHERE`, `GROUP BY`, `ORDER BY`, `LIMIT` depending on shape
4. `ExpectEnd` (optional trailing `;`, then EOF)

### 13.11.2 SELECT list shapes

`ParseSelectItems` returns a list of `LogicalSelectListItem` and sets `starOnly`:

- `SELECT *` → empty list, `starOnly = true`
- Column projections → `LogicalColumnProjection`
- Aggregates → `LogicalAggregationCall` via `TryParseAggregationCall`

`DISTINCT` is explicitly rejected:

```csharp
if (_cur.Kind == SqlTokenKind.Identifier && LexemeEqualsIgnoreCase(_cur, "DISTINCT"))
    throw new SqlCompileException("DISTINCT is not supported in the strict SQL subset.");
```

### 13.11.3 FROM clause: single table vs join

```csharp
private FromClause ParseFromClause()
{
    var left = ExpectIdentifier("table name");
    if (_cur.Kind != SqlTokenKind.KwInner)
        return new SingleTableFrom(left);

    Advance();
    Expect(SqlTokenKind.KwJoin, "JOIN");
    var right = ExpectIdentifier("table name");
    Expect(SqlTokenKind.KwOn, "ON");
    // ... ParseJoinEquiConditions ...
    return new JoinFrom(left, right, leftKeys, rightKeys);
}
```

Join keys must equate one qualified column from the left table with one from the right. `MapJoinPair` accepts either column order and normalizes into left/right key lists.

### 13.11.4 WHERE: conjunctive predicates

`TryParseWhereClause` requires `WHERE` then parses `ParseWherePredicate`, then zero or more `AND` conjuncts:

```csharp
private SimpleWhereClause ParseWherePredicate()
{
    // optional table.column qualifier
    var op = ParseCompareOp();
    var lit = ParseLiteral();
    return new SimpleWhereClause
    {
        QualifierTableName = qualifier,
        ColumnName = column,
        Operator = op,
        Literal = lit,
    };
}
```

Each conjunct becomes one `SimpleWhereClause` in a list. There is no `OR`, no parentheses, no expression trees.

### 13.11.5 GROUP BY validation

When `GROUP BY` is present on a single-table scan, the parser builds:

```csharp
return new LogicalPlan(new LogicalTableScan
{
    TableName = table,
    WhereConjuncts = whereConjuncts,
    GroupByColumns = groupByColsSingle,
    SelectList = selectItems,
});
```

`ValidateGroupedSelect` enforces SQL grouping rules at parse time: every bare column in `SELECT` must appear in `GROUP BY`. Aggregates are validated by `ValidateAggregationCall`.

For joins with `GROUP BY`, qualified columns are mandatory (`table.column`) because unqualified names are ambiguous.

### 13.11.6 Non-grouped branches and sort/limit

Non-grouped scans choose among `SELECT *` (`Projection = null`), a lone global aggregate, or a column projection list; mixing bare columns with aggregates without `GROUP BY` is an error. Join queries mirror the same split on `LogicalInnerJoin`.

`RejectOrderByLimitAfterGrouped` rejects `ORDER BY`/`LIMIT` on grouped or global-aggregate queries. Otherwise the parser attaches `OrderBy` and `Limit` to the logical node; the binder lowers them to `SortTopNPhysicalPlan`.

## 13.12 Logical IR

Logical nodes live in `RainDB.Abstractions/Logical`. They use **names**, not indices.

### 13.12.1 LogicalTableScan

```csharp
// src/RainDB.Abstractions/Logical/LogicalTableScan.cs
public sealed class LogicalTableScan : ILogicalRoot
{
    public required string TableName { get; init; }
    public IReadOnlyList<LogicalColumnProjection>? Projection { get; init; }
    public IReadOnlyList<LogicalColumnProjection>? GroupByColumns { get; init; }
    public IReadOnlyList<LogicalSelectListItem>? SelectList { get; init; }
    public IReadOnlyList<SimpleWhereClause>? WhereConjuncts { get; init; }
    public LogicalAggregate? Aggregate { get; init; }
    public IReadOnlyList<LogicalSortKey>? OrderBy { get; init; }
    public int? Limit { get; init; }
}
```

`Explain()` on logical nodes produces human-readable plan text for debugging and future `EXPLAIN` SQL support.

### 13.12.2 Other logical nodes

`SimpleWhereClause` stores optional `QualifierTableName`, `ColumnName`, `ScalarCompareOp`, and `SqlLiteral` (kind + raw text; coercion happens in the binder). `LogicalInnerJoin` carries left/right table names, parallel qualified join key lists, optional `WhereConjuncts`, projection or grouped select list, and optional sort/limit. Keeping `ILogicalRoot` in Abstractions lets tests assert on parsed plans without referencing `RainDB.Sql`.

## 13.13 DefaultSqlCompiler

The production compiler is a thin switch over logical roots:

```csharp
// src/RainDB.Sql/Compilation/DefaultSqlCompiler.cs
public sealed class DefaultSqlCompiler : ISqlCompiler
{
    private readonly VectorizedScanExecutionOptions _defaultScanOptions;

    public ValueTask<IPhysicalPlan> CompileAsync(string sql, ICatalog catalog, CancellationToken cancellationToken = default)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(sql);
        ArgumentNullException.ThrowIfNull(catalog);
        cancellationToken.ThrowIfCancellationRequested();
        var logical = SqlParser.Parse(sql);
        IPhysicalPlan plan = logical.Root switch
        {
            LogicalTableScan s => LogicalTableScanBinder.BindAndLower(s, catalog, _defaultScanOptions),
            LogicalInnerJoin j => LogicalJoinBinder.BindAndLower(j, catalog, PhysicalJoinAlgorithm.Hash, _defaultScanOptions),
            _ => throw new InvalidOperationException($"Unsupported logical root {logical.Root.GetType().Name}."),
        };
        return ValueTask.FromResult(plan);
    }
}
```

Notable details:

- **Hash join is the default** algorithm passed to `LogicalJoinBinder`. Phase B roadmap work will make this heuristic-driven.
- **`VectorizedScanExecutionOptions`** flows into every physical plan (parallelism, channel scheduler, AVX2 sum). Inject options at construction time for session-level tuning.

## 13.14 LogicalTableScanBinder overview

`LogicalTableScanBinder.BindAndLower` is the heart of single-table lowering:

```csharp
// src/RainDB.Sql/Compilation/LogicalTableScanBinder.cs
public static IPhysicalPlan BindAndLower(
    LogicalTableScan scan,
    ICatalog catalog,
    VectorizedScanExecutionOptions scanOptions = default)
{
    if (!catalog.TryGetTable(scan.TableName, out var ts) || ts is null)
        throw new SqlCompileException($"Table '{scan.TableName}' does not exist in the catalog or is not a columnar table.");
    if (ts is not IColumnarTableSource colTable)
        throw new SqlCompileException($"Table '{scan.TableName}' is registered but is not a columnar table source.");

    if (scan.GroupByColumns is { Count: > 0 })
        return BindHashAggregate(scan, colTable, ts.Schema, scanOptions);

    return BindVectorizedScan(scan, colTable, ts.Schema, scanOptions);
}
```

Binding steps common to both paths:

1. Resolve `TableName` → `IColumnarTableSource` and `TableSchema`
2. Validate `WHERE` table qualifiers reference the scanned table only
3. Build `ColumnCompareFilter[]` from `WhereConjuncts`
4. Emit the appropriate physical plan with `TableId`, indices, filters, and options

## 13.15 BindVectorizedScan

`BindVectorizedScan` handles `SELECT *`, column projections, global aggregates, and sort/limit wrapping.

### 13.15.1 Output column indices

```csharp
if (scan.Projection is null)
{
    outputIndices = new int[colCount];
    for (var i = 0; i < colCount; i++)
        outputIndices[i] = i;
}
else
{
    outputIndices = new int[scan.Projection.Count];
    for (var i = 0; i < scan.Projection.Count; i++)
    {
        var p = scan.Projection[i];
        ValidateProjectionTableQualifier(p, scan.TableName);
        outputIndices[i] = ResolveColumn(schema, p.ColumnName, scan.TableName);
    }
}
```

`ResolveColumn` is case-insensitive linear search over `TableSchema.Columns`. A future catalog index would replace this hot path for wide schemas.

### 13.15.2 Global aggregate branch

When `scan.Aggregate` is set:

```csharp
if (a.Kind == AggregateKind.Count && a.ColumnName is null)
    spec = new AggregateSpec(-1, AggregateKind.Count);
else if (a.Kind == AggregateKind.Count)
    spec = new AggregateSpec(ResolveColumn(schema, a.ColumnName!, scan.TableName), AggregateKind.Count);
else
    spec = new AggregateSpec(ResolveColumn(schema, a.ColumnName!, scan.TableName), a.Kind);
```

`AggregateSpec.SourceColumnIndex == -1` means `COUNT(*)`. `ValidateAggregate` enforces type rules (`SUM` only on numeric types, `MIN`/`MAX` on `Float64` today).

### 13.15.3 SortTopN wrapping

If the scan is not a global aggregate and has `ORDER BY` or `LIMIT`:

```csharp
var scanPlan = new VectorizedScanPhysicalPlan(colTable.Id, outputIndices, filters, aggregate, scanOptions);
if (scan.Aggregate is not null)
    return scanPlan;
if (scan.OrderBy is not { Count: > 0 } && scan.Limit is null)
    return scanPlan;

var sortSpecs = scan.OrderBy is { Count: > 0 } ob
    ? BuildTableSortKeySpecs(schema, scan.TableName, ob)
    : Array.Empty<SortKeyPhysicalSpec>();
return new SortTopNPhysicalPlan(colTable.Id, outputIndices, filters, sortSpecs, scan.Limit, scanOptions);
```

`BuildTableSortKeySpecs` rejects non-sortable types (only fixed-width and `Utf8` today).

## 13.16 BindHashAggregate

Grouped queries lower to `HashAggregatePhysicalPlan`.

### 13.16.1 Group key indices

```csharp
var groupIndices = new int[scan.GroupByColumns!.Count];
for (var i = 0; i < scan.GroupByColumns.Count; i++)
{
    var p = scan.GroupByColumns[i];
    ValidateProjectionTableQualifier(p, scan.TableName);
    groupIndices[i] = ResolveColumn(schema, p.ColumnName, scan.TableName);
}
```

### 13.16.2 SELECT list → output slots

The binder walks `SelectList` in output order, building parallel arrays of `AggregateSpec` and `HashAggregateOutputSlot`:

```csharp
case LogicalColumnProjection col:
    if (!keyOrdinal.TryGetValue(NormalizeGroupKey(col, scan.TableName), out var ko))
        throw new SqlCompileException(
            $"Column '{ExplainProj(col)}' is not listed in GROUP BY for table '{scan.TableName}'.");
    slots.Add(new HashAggregateOutputSlot(HashAggregateOutputColumnKind.GroupKey, ko));
    break;
case LogicalAggregationCall agg:
    aggs.Add(ToAggregateSpec(schema, agg, scan.TableName));
    slots.Add(new HashAggregateOutputSlot(HashAggregateOutputColumnKind.Aggregate, aggs.Count - 1));
    break;
```

`NormalizeGroupKey` uses an internal `\u001f` separator between table qualifier and column name so `"sales\u001fregion"` keys stay unambiguous.

### 13.16.3 Physical plan emission

```csharp
return new HashAggregatePhysicalPlan(
    colTable.Id,
    groupIndices,
    aggs.ToArray(),
    slots.ToArray(),
    filters,
    scanOptions);
```

Filters are identical `ColumnCompareFilter[]` used by vectorized scan — hash aggregation applies them per batch before accumulating groups.

## 13.17 Filter building: WHERE → ColumnCompareFilter

Filter construction is shared between table scan and join binders via `BuildColumnCompareFilters` and `BuildColumnCompareFilter`.

### 13.17.1 Conjunctive array

```csharp
internal static ColumnCompareFilter[]? BuildColumnCompareFilters(
    IReadOnlyList<SimpleWhereClause>? conjuncts,
    TableSchema schema,
    string tableName)
{
    if (conjuncts is null or { Count: 0 })
        return null;
    var arr = new ColumnCompareFilter[conjuncts.Count];
    for (var i = 0; i < conjuncts.Count; i++)
        arr[i] = BuildColumnCompareFilter(conjuncts[i], schema, tableName);
    return arr;
}
```

`null` means no filter. Non-null arrays are ANDed at execution time.

### 13.17.2 ColumnCompareFilter structure

```csharp
// src/RainDB.Query/Plans/VectorizedScanPhysicalPlan.cs
public readonly record struct ColumnCompareFilter(
    int ColumnIndex,
    ScalarCompareOp Op,
    long ImmediateBits,
    byte[]? Utf8LiteralBytes = null);
```

Fixed-width columns store the comparison value in `ImmediateBits`:

- `Int32` / `Int64` / `Boolean` → integer bits in the low bits of `long`
- `Float64` → `BitConverter.DoubleToInt64Bits(d)`

UTF-8 columns use `Utf8LiteralBytes` and only support `Eq` and `Ne`:

```csharp
if (wt == RainDbType.Utf8)
{
    if (where.Operator is not (ScalarCompareOp.Eq or ScalarCompareOp.Ne))
        throw new SqlCompileException(
            $"WHERE on UTF-8 column '{where.ColumnName}' (table '{tableName}') supports only '=' and '!=' or '<>' with a string literal.");
    if (where.Literal.Kind != SqlLiteralKind.String)
        throw new SqlCompileException(
            $"UTF-8 column '{where.ColumnName}' (table '{tableName}') requires a single-quoted string literal.");
    var bytes = Encoding.UTF8.GetBytes(where.Literal.Text);
    return new ColumnCompareFilter(wi, where.Operator, 0, bytes);
}
```

### 13.17.3 Literal coercion

`CoerceLiteralToImmediateBits` is strict — no implicit casts. Integer columns reject float literals; float columns accept integer text by widening to `double` then storing `BitConverter.DoubleToInt64Bits`. Execution kernels never parse strings at runtime.

### 13.17.4 Table qualifier validation

```csharp
internal static void ValidateWhereTableQualifier(SimpleWhereClause? where, string scannedTableName)
{
    if (where?.QualifierTableName is { } q && !q.Equals(scannedTableName, StringComparison.OrdinalIgnoreCase))
        throw new SqlCompileException(
            $"WHERE references table '{q}' but the FROM clause scans '{scannedTableName}' only.");
}
```

Single-table scans reject `other_table.col` in `WHERE` even if `other_table` exists in the catalog.

## 13.18 Execution: how filters run

After lowering, `SelectionEvaluator` in `RainDB.Query/Vectorized` consumes `ColumnCompareFilter[]`:

```csharp
internal static int FillSelectedRowsConjunctive(
    IColumnarBatch batch,
    ReadOnlySpan<ColumnCompareFilter> filters,
    Span<int> dest)
{
    var count = FillSelectedRows(batch.Columns[filters[0].ColumnIndex], filters[0], dest);
    for (var f = 1; f < filters.Length; f++)
    {
        var col = batch.Columns[filters[f].ColumnIndex];
        count = filters[f].Utf8LiteralBytes is not null || col.PhysicalType == RainDbType.Utf8
            ? IntersectUtf8(col, filters[f], dest, count)
            : FixedWidthSelectionKernels.IntersectSelectedIndices(col, filters[f], dest, count);
    }
    return count;
}
```

The first predicate fills a dense selection vector; later predicates intersect in place. This is the compile-time `AND` of `WHERE` realized at vectorized execution time.

## 13.19 LogicalJoinBinder (brief)

Join lowering mirrors scan binding:

- Resolve both tables and validate key column types match
- Split `WHERE` conjuncts to probe/build sides via `ResolveJoinWhere`
- Build `JoinPhysicalPlan` or `GroupedJoinPhysicalPlan`
- Optionally wrap with `SortTopNPhysicalPlan`

`LogicalTableScanBinder.BuildColumnCompareFilters` is reused for per-side filters. When reading join code, look for the same `ColumnCompareFilter` and `ValidateWhereTableQualifier` patterns.

## 13.20 Error handling

All compile failures throw `SqlCompileException`. Parser errors include token positions; binder errors reference table/column names and unsupported type combinations. `IsReservedWord` blocks using `SELECT`, `FROM`, and other keywords as identifiers. Tests in `src/RainDB.Tests/SqlStrictSubsetCompilerTests.cs` and `SqlGroupByTests.cs` lock accepted shapes and error messages.

## 13.21 Worked example: grouped aggregate

SQL:

```sql
SELECT region, SUM(amount)
FROM sales
WHERE amount > 1.0
GROUP BY region
```

**Parse** → `LogicalTableScan` with:

- `WhereConjuncts`: one `SimpleWhereClause` (`amount`, `Gt`, float literal `1.0`)
- `GroupByColumns`: `[region]`
- `SelectList`: `[region projection, SUM(amount) call]`

**Bind** → `HashAggregatePhysicalPlan`:

- `TableId` from catalog entry `sales`
- `GroupKeyColumnIndices`: `[index of region]`
- `Aggregates`: `[AggregateSpec(amount_index, Sum)]`
- `OutputColumns`: group key slot 0, aggregate slot 0
- `Filters`: `[ColumnCompareFilter(amount_index, Gt, double_bits)]`

**Execute** → `HashAggregateEngine.ExecuteAsync` scans batches, applies filters per batch, accumulates partial hash maps, merges, sorts keys, materializes output batch.

## 13.22 LINQ and second front ends

`DefaultLinqCompiler` currently returns `ExplainOnlyPhysicalPlan`. The intended architecture is `Expression tree → Logical IR → existing binders → IPhysicalPlan`. See Chapter 15 for LINQ and plan-cache roadmap detail.

## 13.23 Checklist: adding a new SQL feature

1. **Lexer** — new tokens only if syntactic keywords are required
2. **SelectParser** — new productions; extend logical IR fields if needed
3. **Logical Explain** — update `Explain()` for debuggability
4. **Binder** — validation + lowering to existing or new physical plans
5. **Executor** — implement operator if no plan exists yet
6. **Tests** — parser unit tests, binder error tests, end-to-end SQL tests
7. **Book / Programming Guide** — document the new surface

## 13.24 Summary

**Part I** covered the general SQL compilation pipeline: lexer, parser, semantic analysis, logical and physical IR, optimization families, and prepared statements. **Part II** showed how RainDB implements a strict staged translator — `SqlLexer` and `SelectParser` produce logical IR; `LogicalTableScanBinder` and `LogicalJoinBinder` resolve catalog names and lower `WHERE` conjuncts to `ColumnCompareFilter` arrays consumed directly by vectorized execution. Extending SQL means extending this pipeline while keeping logical IR as the shared contract between SQL, LINQ, and programmatic planners.
