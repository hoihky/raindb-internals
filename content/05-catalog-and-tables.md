---
title: "Chapter 5: Catalog and Tables"
order: 5
---

# Chapter 5: Catalog and Tables

Before any row can be read or written, the engine must answer two questions: **what objects exist**, and **where their bytes live**. The **catalog** — also called the system catalog, metadata layer, or data dictionary — is the authoritative registry of tables, views, columns, types, and related metadata. The **table** is the unit of storage and scan: a named collection of rows organized according to a **schema**.

This chapter is split into two parts. **Part I** develops the conceptual foundation: how commercial databases structure catalogs, why stable table identity matters, how schemas function as contracts, and why analytical engines often favor append-only storage over in-place mutation. **Part II** maps those ideas onto RainDB's concrete types — `ICatalog`, `TableId`, `MemoryTable`, persistence hooks, and the ephemeral table wrapper used in grouped-join pipelines.

---

# Part I: Concepts and Theory

## 5.1 What is a database catalog?

The catalog sits between SQL and storage. It records **what objects exist** and **how to reach them** — not the user rows themselves (except when system tables double as metadata). Its job is to hold **descriptions** and the mapping from logical names to physical locations.

Every catalog needs at least:

| Metadata element | Purpose |
|------------------|---------|
| Object name | Human-readable identifier (`sales`, `customers`) |
| Object kind | Table, view, index, sequence, … |
| Column definitions | Name, logical type, nullability, defaults |
| Storage location | File path, segment id, partition key |
| Access control | Owner, grants (in multi-user systems) |
| Statistics | Row counts, distinct counts, min/max (for optimization) |

Consider `SELECT region FROM sales`. The engine looks up `sales` in the catalog, confirms `region` is a valid column, finds where batches live, and only then opens storage. Name resolution happens at compile time (**binding**); execution works from the resolved handle, not the string `sales`.

### Catalog as a dependency-inversion boundary

Treat the catalog as a **port**: compilation and execution depend on an interface, not a concrete storage backend. That boundary separates **what** (table identity, schema) from **how** (in-memory list today, mmap file tomorrow). Scan code should not care which backend backs a handle as long as the handle shape is uniform.

### Pseudocode: minimal catalog contract

```
interface Catalog {
    names(): string[]
    lookupByName(name: string) -> TableHandle | NOT_FOUND
    lookupById(id: ObjectId) -> TableHandle | NOT_FOUND
    register(table: TableHandle)
}
```

RainDB's `ICatalog` mirrors this contract, substituting `TableId` for a generic `ObjectId`.

---

## 5.2 System catalogs in SQL databases

Production SQL engines keep a **system catalog** — metadata about the database itself, usually hidden behind `INFORMATION_SCHEMA` or `pg_catalog`. You rarely `SELECT` from it directly, yet every DDL statement and planner decision flows through it.

### PostgreSQL

PostgreSQL persists catalog state in ordinary heap tables whose names start with `pg_`:

| System table | Contents |
|--------------|----------|
| `pg_class` | Relations (tables, indexes, views); includes `relname`, `relkind`, `relfilenode` |
| `pg_attribute` | Columns per relation; type oid, nullability, storage |
| `pg_type` | Data types |
| `pg_namespace` | Schemas (`public`, `pg_catalog`, …) |

`CREATE TABLE foo (...)` inserts into `pg_class` and `pg_attribute`. The planner consults these relations during parse and plan. Each row in `pg_class` carries an **`oid`** — the canonical internal id for that relation.

### SQL Server

SQL Server surfaces the same information through **catalog views** (`sys.tables`, `sys.columns`, `sys.types`) backed by internal tables in `mssqlsystemresource`. The `object_id` column plays the same role as PostgreSQL's `oid`.

### Common themes

Different products, same architectural habits:

1. **Dual access** — human names for SQL, numeric ids for plans and on-disk layout.
2. **Schema versioning** — DDL bumps a generation counter or forces plan-cache eviction.
3. **Bootstrap problem** — system tables must exist before user tables can be registered.
4. **Read-mostly during queries** — OLAP sessions assume catalog entries stay stable for the query lifetime.

RainDB's `InMemoryCatalog` is a stripped-down version of this idea: names, ids, and schemas only — no views, indexes, or ACLs yet.

---

## 5.3 Table identity: OID versus name

SQL says `FROM sales`. Internally, the engine should not hash that string on every operator step. It needs a **stable identity** that survives `RENAME TABLE` and fits cheaply in plan nodes.

### Why names alone are insufficient

| Problem with names only | Consequence |
|-------------------------|-------------|
| `ALTER TABLE sales RENAME TO revenue` | Compiled plans storing `"sales"` become stale |
| Case sensitivity (`Sales` vs `sales`) | Ambiguity across platforms |
| String comparison cost | Every operator step would compare UTF-8 names |
| Temporary / intermediate results | No natural name in user namespace |

### Object identifiers (OIDs)

PostgreSQL stamps each catalog row with an **`oid`** (historically 32-bit; often wider now). Storage files and cached plans reference the oid, not `relname`. A rename changes only the name column; oid stays put, so prepared plans keep working.

Comparable mechanisms elsewhere:

| System | Stable id |
|--------|-----------|
| PostgreSQL | `pg_class.oid` |
| SQL Server | `object_id` |
| Oracle | `DATA_OBJECT_ID` / rowid components |
| DuckDB | Internal table index in catalog |
| RainDB | `TableId` (`Guid`) |

### Name as alias, id as truth

Hold this picture:

```
  User-facing layer          Engine-internal layer
  ─────────────────          ─────────────────────
  "sales"  ──bind──►  TableId(3fa85f64-5717-...)
                           │
                           ├── physical plan
                           ├── persistence directory
                           └── statistics cache key
```

**Binding** is a compile-time step: resolve `FROM sales` once, embed `TableId` in the plan. **Execution** never re-parses the name. A future rename updates the name index; plans and disk paths keyed by id stay valid.

### Pseudocode: binding at compile time

```
function compileFromClause(name: string, catalog: Catalog) -> BoundTable {
    handle = catalog.lookupByName(name)
    if handle == NOT_FOUND:
        throw "table does not exist"
    return BoundTable(id = handle.id, schema = handle.schema)
}

function executeScan(plan: ScanPlan, catalog: Catalog) {
    table = catalog.lookupById(plan.tableId)  // not by name
    for batch in table.batches:
        scan(batch)
}
```

RainDB's binder produces `TableId` values that land in `VectorizedScanPhysicalPlan` and related plan records.

---

## 5.4 Schema as contract

A **schema** is the agreement between writers (ingest, ETL, `INSERT`) and readers (queries, joins, aggregates). It fixes **which columns exist, in what order, with which types**.

### Schema components

| Component | Contract role |
|-----------|-----------------|
| Column names | Bind SQL identifiers (`SELECT amount`) |
| Column types | Determine operators, widths, comparison semantics |
| Column order | Defines physical index in columnar batches |
| Nullability | Three-valued logic in predicates (when enforced) |
| Constraints | PRIMARY KEY, CHECK, FOREIGN KEY (OLTP-heavy; optional in OLAP) |

### Schema as versioned contract

Schemas drift. New columns appear; types widen; labels change. Engines attach a **generation counter** so caches know when compiled artifacts are stale:

```
SchemaVersion = 1  →  plan cache entry valid
ALTER ADD COLUMN   →  SchemaVersion = 2  →  invalidate plan cache
```

Without that signal, a cached plan might still assume three columns while the table now has four — a recipe for out-of-bounds reads or silent wrong answers.

### Validation at the boundary

Catch structural mismatches at **ingest**, before bad data enters storage:

```
function appendBatch(table, batch):
    if not table.schema.matches(batch):
        throw "schema mismatch"
    table.segments.append(batch)
```

`MatchesBatch` verifies column count, per-column row counts, and physical types. It does not enforce business rules like `amount > 0` — that is policy. Shape checking is not optional.

### Logical versus physical schema

| Layer | What it describes |
|-------|-------------------|
| Logical schema | Names and types visible to SQL |
| Physical schema | Encoding (Arrow, length-prefixed UTF-8), compression, sort order |

RainDB stores the logical view in `TableSchema` (`ColumnDef` list). Physical layout is chosen when batches are built — `FixedWidthColumnChunk`, `Utf8ColumnChunk`, and so on.

---

## 5.5 Append-only table design in analytics

OLAP engines commonly model a table as an **append-only chain of sealed segments** instead of updatable row pages.

### Segment model

```
Table "events"
├── Segment 0  (batch / stripe / row group)  — immutable after seal
├── Segment 1
├── Segment 2
└── ...
```

**Append** seals a new segment at the tail. **Read** walks the chain in order, often in parallel. The naive design never mutates a sealed segment.

### Why analytics favors append-only

| Benefit | Explanation |
|---------|-------------|
| Scan simplicity | Iterators walk a list; no free-space map |
| Parallelism | Each segment is an independent **morsel** |
| Compression | Immutable blobs compress better (no update holes) |
| Durability | Write-once segments map cleanly to files (`000000.batch`, `000001.batch`) |
| Cache behavior | Sequential reads through cold segments are predictable |

Parquet row groups, DuckDB segments, and ClickHouse parts all rhyme with this layout, even when SQL exposes `UPDATE`.

RainDB Phase 1 stops at the simple case: `MemoryTable` is a `List<IColumnarBatch>` with append only — no update or delete API.

### Pseudocode: append-only table

```
class AppendOnlyTable {
    schema: Schema
    segments: List<Batch> = []

    append(batch: Batch) {
        assert schema.matches(batch)
        segments.add(batch)   // never modify segments[i] in place
    }

    scan(): Iterator<Batch> {
        return segments.iterator()
    }
}
```

---

## 5.6 MVCC versus append-only: a contrast

**MVCC** and **append-only storage** attack different problems. Seeing the split explains why RainDB's heap table is append-only and MVCC-free.

### MVCC (OLTP-oriented)

MVCC lets readers and writers overlap by keeping **several row versions** alive at once:

```
Row id=42:
  Version A  (xmin=100, xmax=150)  amount=50   ← visible to txn 101-149
  Version B  (xmin=150, xmax=∞)  amount=75   ← visible to txn 150+
```

PostgreSQL chains versions inside heap pages. Snapshots filter on transaction id. `UPDATE` appends a new version; `VACUUM` reclaims dead tuples.

| MVCC strength | MVCC cost |
|---------------|-----------|
| Concurrent read/write | Version chains, bloat, vacuum overhead |
| Row-level isolation | Per-row metadata (xmin, xmax, ctid) |
| Fine-grained locking | Complex visibility rules |

### Append-only (OLAP-oriented)

Append-only layouts skip per-row version chains:

| Append-only strength | Append-only cost |
|---------------------|------------------|
| Simple scan loops | No cheap single-row update |
| Immutable segments | Eventual compaction needed |
| Excellent batch I/O | Stale rows until compaction |

Many OLAP deployments are **read-heavy** during query windows. Appending new segments while readers scan the current list is far simpler than full MVCC — readers see the segment list at scan open; writers only extend the tail.

### When each model fits

| Workload | Favored model |
|----------|---------------|
| Bank transfers, inventory | MVCC row store |
| Log ingestion, metrics, fact tables | Append-only columnar |
| Mixed HTAP | Hybrid (e.g., dual store, or Delta/Iceberg versioning) |

RainDB targets embedded OLAP: one process, analytics-heavy, append through `AppendBatch`. No txn ids, no row versions, no vacuum.

### Versioning without MVCC

**Schema version** (`SchemaVersion`) is not MVCC — it invalidates plans on DDL, not on DML. Row visibility is simply **every batch currently in the list**; cross-writer snapshot isolation is not implemented yet.

---

## 5.7 Table handles in query execution

Once compilation finishes, operators stop caring about SQL names. They carry **table handles** — resolved catalog entries plus whatever metadata the operator needs locally.

### Handle contents (conceptual)

```
TableHandle {
    id: ObjectId
    schema: Schema
    schemaVersion: int
    storage: StorageInterface   // batches, files, mmap region
}
```

### Resolution pipeline

```
Compile time:
  SQL "FROM sales"  →  BoundTable(id, schema snapshot)

Execute time:
  plan.tableId  →  catalog.lookupById  →  IColumnarTableSource
                                              │
                                              └── .Batches for scan
```

### Handle lifetime

| Handle kind | Lifetime |
|-------------|----------|
| Catalog-registered table | Process / database lifetime |
| Ephemeral intermediate | Single query operator chain |
| Prepared plan reference | Until schema version invalidates |

**Ephemeral** handles let a join feed a hash aggregate without polluting the user namespace with `"_temp_join_7"`.

### Defense in depth

Before scanning, execution checks `plan.tableId == resolvedTable.id`. That catches the case where lookup succeeded but the wrong `IColumnarTableSource` instance was passed downstream.

### Operator interface segregation

Not every consumer needs batch pointers:

| Interface level | Provides | Consumer |
|-----------------|----------|----------|
| Metadata only | id, name, schema | Binder, EXPLAIN |
| Columnar storage | + batch list | Scan, aggregate, join |

RainDB separates `ITableSource` (metadata) from `IColumnarTableSource` (+ `Batches`). A row-oriented adapter could implement only the former; Phase 1 operators require columnar sources.

---

## 5.8 Summary: Part I checklist

Before diving into RainDB source, you should be able to answer:

1. **What does a catalog do?** — Maps names and ids to metadata and storage locations.
2. **Why both name and id?** — Names for SQL; ids for stable plans and disk paths.
3. **What is schema-as-contract?** — Structural checks at ingest; generation bumps on DDL.
4. **Why append-only in OLAP?** — Cheap scans, easy parallelism, compression-friendly immutability.
5. **How does MVCC differ?** — Row-level versions for concurrent OLTP; OLAP often omits them.
6. **What is a table handle?** — The resolved catalog entry operators consume at runtime.

---

# Transition to Part II: RainDB Implementation

Part II turns the abstractions above into RainDB types. The catalog port is **`ICatalog`**; the default registry is **`InMemoryCatalog`** with dual indexes. Tables are **`MemoryTable`** objects — append-only **`IColumnarBatch`** lists checked against **`TableSchema`**. Plans embed **`TableId`**, not display names. Durability hooks go through **`IRainDbBatchPersistence`**, with in-memory rollback if a write fails. Grouped joins wrap intermediate output in **`EphemeralColumnarTableSource`** so hash aggregation can run without a catalog entry.

The following sections document every public type, registration rules, append paths, and how execution resolves tables — with paths relative to the RainDB repository root.

---

# Part II: RainDB Implementation

## 5.10 Why two identifiers: name and TableId

RainDB's storage surface is deliberately small: a **catalog** maps human-readable names to **table sources**, and the primary concrete source is **`MemoryTable`** — an append-only list of columnar batches. Nothing in this layer knows about SQL parsing, join algorithms, or SIMD kernels. That separation is intentional. The catalog is a **dependency-inversion port** (`ICatalog`) that execution resolves at runtime; physical plans carry stable **`TableId`** values so compiled operators do not depend on display names.

SQL and application code refer to tables by **name** (`sales`, `customers`). Compiled physical plans, persistence directories, and cross-session identity need something that survives renames and avoids string comparisons on hot paths. RainDB assigns each table a **`TableId`** — a thin wrapper around `Guid`:

```csharp
// src/RainDB.Abstractions/Catalog/TableId.cs
public readonly record struct TableId(Guid Value)
{
    public static TableId New() => new(Guid.NewGuid());
    public static TableId From(Guid g) => new(g);
    public override string ToString() => Value.ToString("N");
}
```

**Design notes:**

- `TableId` is a `readonly record struct`, so it is cheap to pass by value and usable as a dictionary key without heap allocation for the id itself.
- `ToString()` uses the `"N"` format (32 hex digits, no dashes), which matches on-disk directory names under `tables/{tableId}/` in `RainDbFileDatabase`.
- `TableId.New()` is called from `MemoryTable` when the caller does not supply an explicit id — typical for ad-hoc in-memory tables.
- `TableId.From(Guid)` is used when **hydrating** from `catalog.json`, where the id was serialized at export time.

Physical plans store `TableId`, not names. The SQL binder resolves `FROM sales` to a `TableId` once at compile time; `DefaultQueryExecutor` then calls `context.Catalog.TryGetTable(plan.TableId, ...)`. A future `ALTER TABLE ... RENAME` could update the name index while leaving plan identity intact.

---

## 5.11 The table source interfaces

RainDB splits "table metadata" from "columnar storage layout" using the **interface segregation principle**:

```csharp
// src/RainDB.Abstractions/Catalog/ITableSource.cs
public interface ITableSource
{
    TableId Id { get; }
    string Name { get; }
    TableSchema Schema { get; }
    int SchemaVersion { get; }
}
```

Every catalog entry exposes identity, logical schema, and a **schema generation counter**. Execution may cache plans or statistics keyed by `(TableId, SchemaVersion)`; when the logical schema changes, bumping the version signals invalidation without requiring a new `TableId`.

Columnar scans need batch access:

```csharp
// src/RainDB.Abstractions/Catalog/IColumnarTableSource.cs
public interface IColumnarTableSource : ITableSource
{
    IReadOnlyList<IColumnarBatch> Batches { get; }
}
```

Not every future table implementation must be columnar (for example, a row-oriented adapter could implement only `ITableSource`). Scan, hash-aggregate, join, and sort operators require `IColumnarTableSource`; `DefaultQueryExecutor` throws if resolution yields a non-columnar source.

---

## 5.12 ICatalog: the registry port

```csharp
// src/RainDB.Abstractions/Catalog/ICatalog.cs
public interface ICatalog
{
    IReadOnlyCollection<string> TableNames { get; }
    IReadOnlyCollection<TableId> TableIds { get; }
    bool TryGetTable(string name, out ITableSource? table);
    bool TryGetTable(TableId id, out ITableSource? table);
    void Register(ITableSource table);
}
```

The contract is intentionally minimal:

- **Dual lookup** — by case-insensitive name (SQL) and by `TableId` (plans).
- **No unregister** — tables live for the catalog lifetime; dropping tables is not yet a first-class API.
- **Read-only enumeration** — `TableNames` and `TableIds` support tooling and export without exposing internal maps.

SQL compilation receives `ICatalog` so the binder can resolve `FROM` clauses. Execution receives the same catalog through `IExecutionContext.Catalog`. Tests inject `InMemoryCatalog` directly; `RainDbEngine.CreateDefault()` constructs one at the composition root.

---

## 5.13 InMemoryCatalog: concurrent dual indexing

The default implementation maintains two `ConcurrentDictionary` instances:

```csharp
// src/RainDB.Core/Catalog/InMemoryCatalog.cs
public sealed class InMemoryCatalog : ICatalog
{
    private readonly ConcurrentDictionary<string, ITableSource> _byName =
        new(StringComparer.OrdinalIgnoreCase);
    private readonly ConcurrentDictionary<TableId, ITableSource> _byId = new();

    public IReadOnlyCollection<string> TableNames => _byName.Keys.ToArray();
    public IReadOnlyCollection<TableId> TableIds => _byId.Keys.ToArray();

    public bool TryGetTable(string name, out ITableSource? table) =>
        _byName.TryGetValue(name, out table);

    public bool TryGetTable(TableId id, out ITableSource? table) =>
        _byId.TryGetValue(id, out table);

    public void Register(ITableSource table)
    {
        ArgumentNullException.ThrowIfNull(table);
        if (!_byName.TryAdd(table.Name, table))
            throw new InvalidOperationException($"Table name already registered: {table.Name}");
        if (!_byId.TryAdd(table.Id, table))
        {
            _byName.TryRemove(table.Name, out _);
            throw new InvalidOperationException($"Table id already registered: {table.Id}");
        }
    }
}
```

**Registration semantics:**

1. Name insert first. If the name collides, registration fails immediately — no partial state.
2. Id insert second. If the id collides (extremely rare unless misusing `TableId.From`), the name entry is **rolled back** via `TryRemove`. The catalog never exposes a table under one index without the other.
3. Case-insensitive names match SQL's typical default behavior (`StringComparer.OrdinalIgnoreCase`).

**Concurrency:** `ConcurrentDictionary` allows concurrent reads during queries while another thread registers a table. RainDB does not yet support concurrent `AppendBatch` on the same `MemoryTable` from multiple writers; catalog concurrency covers read-heavy OLAP sessions with occasional DDL-like registration.

**Snapshot cost:** `TableNames` and `TableIds` allocate new arrays on every access (`Keys.ToArray()`). This is acceptable for catalog introspection, not for per-query hot paths.

---

## 5.14 TableSchema and batch validation

Before a batch enters `MemoryTable`, `TableSchema.MatchesBatch` validates structural compatibility:

```csharp
// src/RainDB.Abstractions/Schema/TableSchema.cs
public bool MatchesBatch(IColumnarBatch batch)
{
    ArgumentNullException.ThrowIfNull(batch);
    if (batch.Columns.Count != Columns.Count || batch.RowCount < 0)
        return false;
    if (batch.RowCount == 0)
        return batch.Columns.All(c => c.RowCount == 0);
    for (var i = 0; i < Columns.Count; i++)
    {
        var col = batch.Columns[i];
        if (col.RowCount != batch.RowCount)
            return false;
        if (col.PhysicalType != Columns[i].Type)
            return false;
    }
    return true;
}
```

`MemoryTable.AppendCore` throws `ArgumentException` when `MatchesBatch` returns false. This catches:

- Wrong column count
- Per-chunk row count mismatch within a batch
- Physical type mismatch (e.g., `Int64` chunk in an `Int32` column)

`MatchesBatch` does **not** validate null bitmap correctness, UTF-8 offset monotonicity, or vector chunk sizing — those are the chunk implementation's responsibility. Strict row-count policy is optional via `MemoryTableOptions`.

---

## 5.15 MemoryTable: append-only columnar segments

`MemoryTable` is the workhorse in-memory store:

```csharp
// src/RainDB.Core/Tables/MemoryTable.cs
public sealed class MemoryTable : ITableSource, IColumnarTableSource
{
    private readonly List<IColumnarBatch> _batches = new();
    private int _schemaVersion = 1;

    public MemoryTable(string name, TableSchema schema, TableId? id = null,
        MemoryTableOptions options = default)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        Name = name;
        Schema = schema;
        Id = id ?? TableId.New();
        Options = options;
    }

    public TableId Id { get; }
    public string Name { get; }
    public TableSchema Schema { get; }
    public MemoryTableOptions Options { get; }
    public int SchemaVersion => Volatile.Read(ref _schemaVersion);

    public event EventHandler<SchemaVersionChangedEventArgs>? SchemaVersionChanged;

    public IReadOnlyList<IColumnarBatch> Batches => _batches;

    public long RowCount
    {
        get
        {
            long sum = 0;
            foreach (var b in _batches)
                sum += b.RowCount;
            return sum;
        }
    }
}
```

**Storage model:**

- Batches are stored in a `List<IColumnarBatch>` in **insertion order**. Scans iterate `table.Batches` sequentially; there is no automatic compaction or sorting at the storage layer.
- `RowCount` is computed on demand by summing batch row counts — O(number of batches), not O(1).
- The table does not own chunk memory lifetimes beyond holding references; pooled chunks from queries are unrelated to table storage.

---

## 5.16 AppendBatch vs AppendHydratedBatch

Two public/internal entry points share one implementation:

```csharp
public void AppendBatch(IColumnarBatch batch) =>
    AppendCore(batch, notifyPersistence: true);

internal void AppendHydratedBatch(IColumnarBatch batch) =>
    AppendCore(batch, notifyPersistence: false);

private void AppendCore(IColumnarBatch batch, bool notifyPersistence)
{
    ArgumentNullException.ThrowIfNull(batch);
    if (!Schema.MatchesBatch(batch))
        throw new ArgumentException("Batch does not match this table's schema.", nameof(batch));
    VectorChunkLimits.ValidateRowCount(batch.RowCount, Options.StrictVectorChunkRows);
    _batches.Add(batch);
    if (notifyPersistence && Options.BatchPersistence is { } persistence)
    {
        try
        {
            persistence.OnBatchAppended(Id, Name, _batches.Count - 1, batch);
        }
        catch
        {
            _batches.RemoveAt(_batches.Count - 1);
            throw;
        }
    }
}
```

### User append path (`AppendBatch`)

Application code and ingest pipelines call `AppendBatch`. The sequence is:

1. Null-check the batch
2. Schema validation
3. Optional strict vector sizing (see §5.18)
4. Add to `_batches`
5. If `Options.BatchPersistence` is set, invoke `OnBatchAppended`

### Hydration path (`AppendHydratedBatch`)

When loading from disk, the batch **already exists** on storage. Calling the persistence hook would write the same bytes twice (or create duplicate files). `RainDbFileDatabase.LoadBatchesIntoTable` decodes each `.batch` file and calls `AppendHydratedBatch`:

```csharp
// src/RainDB.Core/Persistence/RainDbFileDatabase.cs (excerpt)
foreach (var file in Directory.GetFiles(dir, "*.batch").OrderBy(f => f, StringComparer.Ordinal))
{
    var bytes = File.ReadAllBytes(file);
    var batch = RainDbBatchBinaryCodec.DecodeBatch(bytes);
    table.AppendHydratedBatch(batch);
}
```

Hydration uses `ImportCatalog` tables without persistence hooks; `Open` wires `BatchPersistence: this` so subsequent user appends persist.

---

## 5.17 Persistence hook rollback

The persistence integration follows a **two-phase append** pattern: memory first, then durable mirror. If the mirror fails, memory state is rolled back.

```csharp
// src/RainDB.Abstractions/Persistence/IRainDbBatchPersistence.cs
public interface IRainDbBatchPersistence
{
    void OnBatchAppended(TableId tableId, string tableName,
        int zeroBasedBatchIndex, IColumnarBatch batch);
}
```

`RainDbFileDatabase` implements this interface:

```csharp
void IRainDbBatchPersistence.OnBatchAppended(
    TableId tableId, string tableName, int zeroBasedBatchIndex, IColumnarBatch batch)
{
    var path = GetBatchPath(tableId, zeroBasedBatchIndex);
    lock (_ioLock)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);
        WriteBatchFile(path, batch);
    }
}
```

**Rollback behavior** is verified in tests:

```csharp
// tests/RainDB.Tests/RobustnessAndEdgeCaseTests.cs (conceptual)
var table = new MemoryTable("t", schema,
    options: new MemoryTableOptions(BatchPersistence: new ThrowingPersistence()));
Assert.Throws<IOException>(() => table.AppendBatch(batch));
Assert.Equal(0, table.RowCount);
Assert.Empty(table.Batches);
```

If `OnBatchAppended` throws **any** exception, `AppendCore` removes the batch just added (`RemoveAt(_batches.Count - 1)`) and rethrows. Invariants after failure:

- `RowCount` unchanged
- `Batches` empty (for first append) or same as before
- No orphan batch file (write failed before completion, or never started)

This is not a full transactional log — catalog.json is flushed separately on `CreateMemoryTable` — but **in-memory and per-batch file state stay consistent** for the append that failed.

**Ordering:** `zeroBasedBatchIndex` is `_batches.Count - 1` after the add, so file names `000000.batch`, `000001.batch`, … align with list indices.

---

## 5.18 MemoryTableOptions and vector chunk policy

```csharp
// src/RainDB.Core/Tables/MemoryTableOptions.cs
public readonly record struct MemoryTableOptions(
    bool StrictVectorChunkRows = false,
    IRainDbBatchPersistence? BatchPersistence = null)
{
    public static MemoryTableOptions Default => default;
}
```

| Flag | Default | Effect |
|------|---------|--------|
| `StrictVectorChunkRows` | `false` | When `true`, enforces DuckDB-style batch sizes |
| `BatchPersistence` | `null` | Optional hook after successful in-memory append |

Strict sizing delegates to `VectorChunkLimits`:

```csharp
// src/RainDB.Core/Columnar/VectorChunkLimits.cs
public const int MinRows = 64 * 1024;   // 65,536
public const int MaxRows = 1024 * 1024; // 1,048,576

public static void ValidateRowCount(int rowCount, bool enforce)
{
    if (!enforce) return;
    if (rowCount == 0) return;
    if (rowCount < MinRows || rowCount > MaxRows)
        throw new ArgumentOutOfRangeException(...);
}
```

Tests confirm: a 2-row batch fails under strict mode; a 64K-row batch succeeds. Production ingest pipelines should enable strict mode for cache-friendly vector widths; unit tests leave it off for small fixtures.

`RainDbFileDatabase.CreateMemoryTable` passes `new MemoryTableOptions(strictVectorChunkRows, BatchPersistence: this)`.

---

## 5.19 Schema version and cache invalidation

```csharp
public int BumpSchemaVersion()
{
    var v = Interlocked.Increment(ref _schemaVersion);
    SchemaVersionChanged?.Invoke(this, new SchemaVersionChangedEventArgs(v));
    return v;
}
```

- Initial version is **1** (not 0), so "never bumped" is distinguishable from "bumped once to 2".
- `SchemaVersion` is read with `Volatile.Read` for cross-thread visibility without locking.
- `BumpSchemaVersion` is reserved for future `ALTER TABLE` / migration workflows; the current SQL subset does not call it automatically.
- `SchemaVersionChangedEventArgs` carries `NewVersion` for external caches (plan cache, statistics).

**Interaction with plans:** Physical plans capture `TableId` at compile time. They do not embed `SchemaVersion` today. A production plan cache should validate `(TableId, SchemaVersion)` before reuse. `EphemeralColumnarTableSource` hardcodes `SchemaVersion => 0` because ephemeral results are single-query intermediates.

---

## 5.20 EphemeralColumnarTableSource

Grouped join queries execute a join, then aggregate over the join output without registering a catalog table:

```csharp
// src/RainDB.Query/Execution/EphemeralColumnarTableSource.cs
internal sealed class EphemeralColumnarTableSource : IColumnarTableSource
{
    public EphemeralColumnarTableSource(
        TableId id, string name, TableSchema schema,
        IReadOnlyList<IColumnarBatch> batches)
    {
        Id = id;
        Name = name;
        Schema = schema;
        Batches = batches;
    }

    public TableId Id { get; }
    public string Name { get; }
    public TableSchema Schema { get; }
    public int SchemaVersion => 0;
    public IReadOnlyList<IColumnarBatch> Batches { get; }
}
```

`DefaultQueryExecutor` still uses ephemerals where a second operator needs a table-shaped input:

- **`DerivedTableScanPhysicalPlan`** — materialize the subquery, wrap batches in `EphemeralColumnarTableSource`, register it through **`OverlayCatalog`** on a scoped session, then run the outer plan.
- **`GroupedSortTopNPhysicalPlan` / grouped sort after join** — aggregate output is wrapped temporarily so `SortTopNOperator` can run over a uniform `IColumnarTableSource`.

**Grouped join without a giant intermediate table:** `GroupedJoinOperator` (see Chapter 11) streams join output batches into hash aggregation instead of building one full join result and re-wrapping it as a table. The older join-then-ephemeral-then-agg pattern remains a useful mental model for derived tables, not for the default grouped-join fast path.

```csharp
// src/RainDB.Core/Catalog/OverlayCatalog.cs — overlay wins on name/id lookup
public bool TryGetTable(string name, out ITableSource? table)
{
    if (_byName.TryGetValue(name, out table))
        return true;
    return _base.TryGetTable(name, out table);
}
```

**Why this exists:**

- Hash and sort operators expect an `IColumnarTableSource`, not a raw `IQueryResult`.
- Derived-table aliases must resolve in the catalog for the outer `FROM` clause without persisting data.
- `SchemaVersion => 0` on ephemerals signals that plan caches should not treat them like durable tables.

---

## 5.21 End-to-end catalog flows

### In-memory engine bootstrap

```csharp
// src/RainDB.Driver/RainDbEngine.cs
public static RainDbEngine CreateDefault()
{
    var catalog = new InMemoryCatalog();
    var buffers = new HybridBufferPool();
    return new RainDbEngine(catalog, buffers, buffers,
        new DefaultQueryExecutor(), ...);
}
```

Register a table, append data, query:

```csharp
var schema = new TableSchema([new ColumnDef("region", RainDbType.Utf8)]);
var table = new MemoryTable("sales", schema);
catalog.Register(table);
table.AppendBatch(myBatch);
// SQL: SELECT ... FROM sales  → binder resolves "sales" → table.Id → VectorizedScanPhysicalPlan
```

### Persistent open / hydrate / append

```csharp
var engine = RainDbEngine.OpenPersistent("/data/mydb");
// Hydrates catalog.json + *.batch files via AppendHydratedBatch
var table = engine.FileDatabase!.CreateMemoryTable("events", schema);
table.AppendBatch(batch); // → OnBatchAppended → tables/{id}/000000.batch
```

### Export / import without persistence hook

`RainDbFileDatabase.ExportCatalog` / `ImportCatalog` copy catalog metadata and batch files; imported tables have no `BatchPersistence` until re-wired to a file database.

---

## 5.22 Catalog resolution in execution

Every physical operator that reads a table follows the same resolution pattern:

```csharp
if (!context.Catalog.TryGetTable(plan.TableId, out var ts)
    || ts is not IColumnarTableSource cols)
    throw new InvalidOperationException(
        $"Columnar table {plan.TableId} was not found in the catalog.");
```

Failures mean:

- Table was never registered
- Table was registered but is not columnar
- Plan references a stale `TableId` from a different catalog instance

`VectorizedScanEngine.ValidatePlan` additionally asserts `plan.TableId == table.Id` when the executor passes the resolved source — defense in depth against caller bugs.

---

## 5.23 Testing patterns

`ColumnarAndCatalogTests` verifies dual lookup by name and id, schema version events, persistence rollback, and strict chunk policy (`RobustnessAndEdgeCaseTests`, `RainDbPersistenceTests`).

---

## 5.24 Design trade-offs and future directions

| Decision | Rationale | Limitation |
|----------|-----------|------------|
| Append-only batches | Simple scans, easy persistence | No in-place updates or deletes |
| Dual dictionary catalog | Fast name and id lookup | Registration rollback on id collision |
| Synchronous persistence hook | Simple error model | Append latency tied to disk |
| `SchemaVersion` event | Extensibility for plan cache | Compiler cache uses catalog fingerprint today |
| `EphemeralColumnarTableSource` | Derived tables, grouped sort | Grouped join often streams without holding full join rowset |
| `OverlayCatalog` | Scoped derived-table names | Extra catalog layer per derived-table query |

Plausible extensions: `UnregisterTable`, weak-reference plan cache keyed by schema version, asynchronous WAL with memory-table staging, and catalog entries that are views rather than physical `MemoryTable` instances.

---

## 5.25 Mental model

```
  SQL / API                    Physical plan
      │                              │
      ▼                              ▼
  ICatalog.TryGetTable(name)    TableId in plan
      │                              │
      └──────────┬───────────────────┘
                 ▼
         IColumnarTableSource
                 │
                 ▼
      MemoryTable.Batches[i]
                 │
                 ▼
      VectorizedScanEngine / Join / Aggregate ...
```

The catalog answers **what** and **where**; `MemoryTable` answers **how rows are stored**; `TableId` and `SchemaVersion` answer **whether a compiled plan still applies**. Persistence hooks and hydration paths ensure the in-memory story matches on-disk segments without double-writing during load.

---

## 5.26 Quick reference

| Type | Location | Role |
|------|----------|------|
| `TableId` | `src/RainDB.Abstractions/Catalog/TableId.cs` | Stable table identity |
| `ICatalog` | `src/RainDB.Abstractions/Catalog/ICatalog.cs` | Registry port |
| `InMemoryCatalog` | `src/RainDB.Core/Catalog/InMemoryCatalog.cs` | Default registry |
| `ITableSource` | `src/RainDB.Abstractions/Catalog/ITableSource.cs` | Metadata surface |
| `IColumnarTableSource` | `src/RainDB.Abstractions/Catalog/IColumnarTableSource.cs` | Batch storage surface |
| `MemoryTable` | `src/RainDB.Core/Tables/MemoryTable.cs` | Columnar heap table |
| `MemoryTableOptions` | `src/RainDB.Core/Tables/MemoryTableOptions.cs` | Strict sizing + persistence |
| `IRainDbBatchPersistence` | `src/RainDB.Abstractions/Persistence/IRainDbBatchPersistence.cs` | Append hook |
| `EphemeralColumnarTableSource` | `src/RainDB.Query/Execution/EphemeralColumnarTableSource.cs` | Join→aggregate bridge |

Understanding this layer is prerequisite for execution (Chapter 7) and vectorized scan (Chapter 8): every read path ultimately enumerates `IColumnarTableSource.Batches` resolved through `ICatalog`.
