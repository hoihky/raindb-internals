---
title: "Chapter 14: Persistence"
order: 14
---

# Chapter 14: Persistence

Persistence is what separates a toy in-memory analytics library from a database you can trust across process restarts. This chapter begins with storage-engine architecture, durability theory, write-ahead logging, checkpointing, and columnar file formats used in industry. Part II walks RainDB's directory-backed persistence: `catalog.json`, batch codec byte layouts, hydration, and export/import snapshots.

---

# Part I — Concepts

## 14.1 Storage engine architecture

A **storage engine** owns durable reads and writes — to local SSDs, spinning disks, or remote object stores — while exposing tables and columns to the query layer. Typical designs stack responsibilities vertically:

```text
Query executor
      ↓
Table / catalog API
      ↓
Buffer manager (page cache, pin/unpin)
      ↓
Storage manager (segments, files, LSM levels)
      ↓
OS / device (block I/O, mmap, direct I/O)
```

### Responsibilities

| Component | Role |
|-----------|------|
| **Catalog** | Metadata: schemas, types, constraints, statistics |
| **Heap / segment store** | Row or column data on disk |
| **Index structures** | B-trees, zone maps, bloom filters (more common in OLTP) |
| **Transaction log** | Ordered change history for crash recovery |
| **Checkpoint / flush** | Persist dirty in-memory state |

Analytical engines often trim the stack: fewer secondary indexes, append-friendly writes, scan-oriented layouts. RainDB's MVP folds much of this into `RainDbFileDatabase` plus `MemoryTable` with append callbacks.

### Read path vs write path

**Writes** usually flow: validate incoming batch → append to an in-memory structure → encode to durable files if configured → refresh catalog metadata.

**Reads** usually flow: resolve table by name → walk segment or batch files → decode or map column data → hand batches to scan operators.

The perennial tradeoff is write throughput (batched sequential I/O) versus read efficiency (columnar layout, pruning metadata, caching).

## 14.2 Durability and the ACID "D"

Durability — the "D" in ACID — means **committed work survives process death**: power failure, kernel panic, or `kill -9`. It sits alongside, but is not the same as:

- **Atomicity** — a transaction's effects appear all at once or not at all (WAL is the usual mechanism)
- **Consistency** — invariants hold before and after each transaction
- **Isolation** — concurrent sessions do not observe torn partial updates

Embedded OLAP products span a spectrum from "durable after explicit flush" to full transactional logging. RainDB's MVP relies on temp-file rename without a documented fsync policy — treat durability as **best-effort** until Phase F (Chapter 15).

### Durability levels in practice

| Level | Mechanism | Tradeoff |
|-------|-----------|----------|
| **Process memory only** | No disk | Fastest; all data lost on exit |
| **Append + rename** | Atomic file replacement | Simple; small windows between related files |
| **fsync per commit** | Force media persistence | Higher latency; stronger guarantees |
| **WAL + checkpoint** | Log-first, batched data flush | Standard for production databases |

## 14.3 Write-ahead logging (WAL)

**Write-ahead logging** is the workhorse durability pattern in database systems. The invariant: **the log must reach durable storage before in-place data pages count as committed**.

### WAL record flow

```text
1. Client requests commit
2. Engine appends WAL record(s) describing the change
3. Log is forced to disk (fsync or group commit)
4. In-memory pages are updated (possibly lazily)
5. Client receives acknowledgment
```

After a crash, recovery **replays the WAL** from the latest checkpoint — redoing committed work and undoing transactions that never finished.

### Why log before data?

If data pages land on disk first and the process dies before the log record, the engine cannot tell which updates were committed. An append-only log supplies a total order of changes.

### WAL segment structure

Typical log files store:

- A monotonic **LSN** (log sequence number) per record
- A record kind: insert, update, delete, catalog mutation, checkpoint marker
- Payload bytes: table id, encoded batch, or page-level diff

Checkpoint records bound replay: they assert that all changes before LSN *X* are already reflected in data files.

### RainDB status

RainDB ships **without a WAL**. Batches and the catalog are written directly to files. Phase F targets WAL plus checkpoint support (Chapter 15).

## 14.4 Checkpointing

A **checkpoint** captures a coherent on-disk image so recovery can start near current state and older log segments can be discarded.

### Types

| Checkpoint style | Description |
|------------------|-------------|
| **Fuzzy** | Data pages flush while transactions continue; WAL brackets the fuzzy interval |
| **Sharp** | Pause writers, flush everything, resume — short replay afterward |
| **Incremental** | Flush only pages dirtied since the prior checkpoint |

Append-only OLAP designs often checkpoint by **sealing immutable segment files** and snapshotting catalog metadata — spiritually similar to RainDB's `FlushCatalog` plus numbered `.batch` files, but without WAL replay today.

### Checkpoint frequency tradeoffs

- More frequent checkpoints shorten recovery but add steady-state I/O
- Rare checkpoints defer checkpoint cost but lengthen WAL replay

## 14.5 Page-oriented vs log-structured storage

Two organizing principles cover most on-disk engines.

### Page-oriented (slotted pages)

Storage is carved into fixed **pages** (often 4–64 KB). B-tree indexes point at pages; updates rewrite pages in place, usually under WAL protection. PostgreSQL, SQLite, and classic row stores follow this shape.

**Strengths:** fine-grained random updates, mature locking models  
**Weaknesses:** write amplification on flash; wide column scans pay per-page overhead

### Log-structured (append-only)

New data appends to sequential **segments**; background compaction merges them. LSM-trees (LevelDB, RocksDB) are the textbook example. Many OLAP systems adopt **append-only column segments** without full LSM merge policies in embedded deployments.

**Strengths:** sequential write bandwidth; immutable segments ease concurrency  
**Weaknesses:** compaction work; reads may touch many files

### RainDB's model

RainDB appends **numbered batch files** per table — log-structured at segment granularity, without merge compaction. Each insert produces `00000N.batch`, aligning with embedded OLAP practice (local DuckDB files, early ClickHouse MergeTree parts, conceptually).

## 14.6 Columnar file formats in industry

Analytical engines lay out each column (or chunk) contiguously on disk. Formats vary in metadata depth and mmap suitability.

| System | Format idea | Notes |
|--------|-------------|-------|
| **Apache Parquet** | Row groups, column chunks, rich footer statistics | De facto interchange format |
| **Apache Arrow IPC** | Flat buffers tuned for alignment and zero-copy | In-memory and streaming |
| **DuckDB** | Proprietary database file with block manager | Tight integration with buffer pool |
| **ClickHouse** | MergeTree parts with per-column compressed `.bin` files | Column files inside immutable parts |
| **Snowflake** | Cloud micro-partitions | Immutable objects plus separate catalog service |

Shared design threads:

- **Typed column chunks** with known row counts
- **Per-column compression** (dictionary, RLE, bit-packing)
- **Min/max/null statistics** for pruning
- **Versioned headers** (magic bytes, format numbers) for evolution

RainDB's `RNBATCH1` batches and `RNBFCOL1` column files (Chapter 16) are deliberately minimal self-describing layouts — no compression or zone maps yet.

## 14.7 Catalog metadata on disk

The **catalog** is the authoritative map from logical names to physical storage: databases, schemas, tables, columns, types, and often statistics or ACLs.

### What catalogs store

- Stable table identifiers (GUIDs, OIDs)
- Column names, types, nullability
- Pointers to data files or byte ranges
- Schema generation counters for plan-cache invalidation

### Serialization choices

| Format | Used by | Tradeoff |
|--------|---------|----------|
| **JSON** | RainDB MVP, many prototypes | Easy to inspect; typically whole-file rewrite |
| **SQLite catalog tables** | DuckDB embedded mode | Transactional metadata updates |
| **Custom binary** | Large server engines | Compact; opaque without tooling |

Readers must never observe torn catalog state — achieve atomicity with temp-file rename or catalog records in the WAL.

RainDB persists `catalog.json` at `formatVersion: 1` and rewrites the full document on each `FlushCatalog`.

## 14.8 Crash recovery basics

Recovery answers one question after an unexpected shutdown: **what is safe to serve?**

### Recovery phases (WAL-based systems)

1. **Analysis** — scan the log from the last checkpoint; list dirty pages and in-flight transactions
2. **Redo** — reapply committed updates missing from data files
3. **Undo** — reverse effects of uncommitted transactions

### Failure modes without WAL

RainDB's MVP can encounter:

- A catalog entry pointing at a table directory with missing or partial batches
- A batch file on disk that the catalog never recorded (orphan segment)
- A crash mid-write to a `.tmp` staging file (final rename semantics usually preserve the prior file)

**Detection** happens at open time: magic validation, row-count checks, cross-checking catalog paths against disk. **Repair** is manual — export/import or restore from backup.

### Idempotent open

`RainDbFileDatabase.Open` materializes an empty database when `catalog.json` is absent; otherwise it hydrates tables and batches. Corrupt or unsupported format versions raise `InvalidDataException` or `NotSupportedException`.

## 14.9 Import, export, and snapshots

A **snapshot** freezes database state at a point in time — for backup, migration, reproducible tests, or cloning.

### Snapshot strategies

| Strategy | Description |
|----------|-------------|
| **File copy** | Duplicate the database directory while the engine is quiesced |
| **Export API** | Write a fresh consistent catalog plus data tree |
| **Volume snapshot** | Filesystem or cloud block snapshot (crash-consistent or coordinated) |

**Import** inverts export: read metadata, recreate tables, load segments. Imported tables may stay read-only or reconnect to live append hooks.

RainDB exposes `ExportCatalog` and `ImportCatalog` for snapshot workflows; import does not require a running engine instance.

---

# Part II — RainDB

An embedded OLAP engine is only useful if data survives process restarts. RainDB stores **catalog metadata as JSON** and **columnar batches as self-describing `RNBATCH1` files**, wiring append hooks into `MemoryTable` so in-memory tables and on-disk segments stay aligned. Atomic replace-on-write avoids torn committed files; optional mmap hydration avoids copying fixed-width columns on open.

## 14.10 Design goals

RainDB persistence optimizes for:

1. **Simplicity.** One JSON catalog plus numbered `.batch` files per table under `tables/{tableId}/`.
2. **Round-trip fidelity.** Supported chunk kinds encode and decode without loss (null bitmaps, UTF-8 layouts, dictionary Int32).
3. **Transparent append.** `MemoryTable.AppendBatch` invokes `IRainDbBatchPersistence.OnBatchAppended`, which writes the batch and refreshes the catalog.
4. **Hydration without rewrite.** `AppendHydratedBatch` loads existing segments without re-persisting them.
5. **Optional zero-copy fixed-width reads.** When `PreferMmapBatchHydration` is true, `RainDbBatchMmapReader` maps each batch file and slices kind-1 columns from the mapping (Chapter 16).

Still out of scope:

- Write-ahead logging (Phase F)
- Documented multi-writer concurrency
- Per-column incremental catalog updates (each `FlushCatalog` rewrites the full JSON document)

## 14.11 On-disk directory layout

A RainDB database root looks like this:

```text
database-root/
  catalog.json
  tables/
    {tableId-guid-N}/
      000000.batch
      000001.batch
      000002.batch
      ...
```

Constants on `RainDbFileDatabase`:

```csharp
// src/RainDB.Core/Persistence/RainDbFileDatabase.cs
public const string CatalogFileName = "catalog.json";
public const string TablesDirectoryName = "tables";
```

| Path | Purpose |
|------|---------|
| `catalog.json` | Table names, stable `TableId` GUIDs, column schemas |
| `tables/{id}/` | All batch segments for one table, named by zero-based append index |
| `*.batch` | Committed binary columnar batch (format v1, magic `RNBATCH1`) |
| `*.tmp` | Staging files during atomic writes — **ignored** on open and batch enumeration |

`TableId` is a 128-bit GUID stored without dashes in JSON (`"N"` format) and as the directory name under `tables/`.

Committed paths are recognized explicitly: `RainDbAtomicFileWriter.IsCommittedBatchFileName` accepts `######.batch` but rejects names ending in `.tmp` (so a crashed `000000.batch.tmp` is never loaded). Catalog reads use `TryReadCommittedCatalogJson`, which reads only `catalog.json`, not `catalog.json.tmp`.

## 14.12 RainDbAtomicFileWriter

All durable catalog and batch replacements go through `RainDbAtomicFileWriter`:

```csharp
// src/RainDB.Core/Persistence/RainDbAtomicFileWriter.cs
public void WriteAllText(string targetPath, string content)
{
    var tmp = targetPath + ".tmp";
    File.WriteAllText(tmp, content);
    File.Move(tmp, targetPath, overwrite: true);
}

public void WriteStream(string targetPath, Action<Stream> writeBody)
{
    var tmp = targetPath + ".tmp";
    using (var fs = File.Create(tmp))
        writeBody(fs);
    File.Move(tmp, targetPath, overwrite: true);
}
```

Crash mid-write leaves a `.tmp` sibling; the previous committed file (if any) remains visible. Readers never treat `.tmp` as authoritative.

## 14.13 RainDbFileDatabaseOptions

```csharp
// src/RainDB.Core/Persistence/RainDbFileDatabaseOptions.cs
public sealed class RainDbFileDatabaseOptions
{
    public bool PreferMmapBatchHydration { get; init; } = true;
    public long MappedBatchMemoryBudgetBytes { get; init; } = long.MaxValue;
    public MappedBatchBudgetExceededBehavior MappedBatchBudgetExceededBehavior { get; init; } =
        MappedBatchBudgetExceededBehavior.EvictColdBatches;
    public bool EnableInt32DictionaryEncoding { get; init; } = true;
}
```

| Option | Effect |
|--------|--------|
| `PreferMmapBatchHydration` | Try `RainDbBatchMmapReader.Open` per `.batch`; fall back to `ReadAllBytes` + `DecodeBatch` on IO errors |
| `MappedBatchMemoryBudgetBytes` | Charged bytes for registered mmap segments (`long.MaxValue` = unlimited) |
| `MappedBatchBudgetExceededBehavior` | `EvictColdBatches` (default) or `Fail` when registration would exceed budget |
| `EnableInt32DictionaryEncoding` | Passed to `RainDbBatchBinaryCodec` on append — may emit kind `4` for qualifying Int32 columns |

Pass options to `RainDbFileDatabase.Open(root, options)`.

## 14.14 Opening a persistent database

### 14.14.1 RainDbFileDatabase.Open

```csharp
public static RainDbFileDatabase Open(string rootDirectory, RainDbFileDatabaseOptions? options = null)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(rootDirectory);
    var root = Path.GetFullPath(rootDirectory);
    Directory.CreateDirectory(root);
    var catalog = new InMemoryCatalog();
    var db = new RainDbFileDatabase(root, catalog, options ?? new RainDbFileDatabaseOptions());
    db.HydrateFromDiskIfPresent();
    return db;
}
```

Steps:

1. Normalize path to full path (stable regardless of working directory)
2. Create root directory if missing (fresh database)
3. Construct empty `InMemoryCatalog` and mmap budget manager from options
4. If committed `catalog.json` exists, deserialize and load all tables + batches

### 14.14.2 RainDbEngine.OpenPersistent

The driver wraps file database + default engine collaborators:

```csharp
// src/RainDB.Driver/RainDbEngine.cs
public static RainDbEngine OpenPersistent(string directoryPath)
{
    var fileDb = RainDbFileDatabase.Open(directoryPath);
    return CreateDefault(fileDb.Catalog, fileDb);
}
```

`RainDbEngine.FileDatabase` holds a reference to the `RainDbFileDatabase` instance so it is not collected while the engine runs. Queries use `fileDb.Catalog` — the same catalog object mutated by hydration and new table registration.

### 14.14.3 HydrateFromDiskIfPresent

```csharp
private void HydrateFromDiskIfPresent()
{
    if (!RainDbAtomicFileWriter.TryReadCommittedCatalogJson(RootDirectory, out var json))
        return;
    lock (_ioLock)
    {
        var doc = JsonSerializer.Deserialize<RainDbCatalogDocument>(json, JsonOptions)
            ?? throw new InvalidDataException("catalog.json could not be deserialized.");
        // ... foreach table: MemoryTable with BatchPersistence: this ...
        LoadBatchesIntoTable(table, _options.PreferMmapBatchHydration);
        Catalog.Register(table);
    }
}
```

Important behaviors:

- **Persistence hook is wired on hydrated tables.** New appends after reopen persist as the next `######.batch`.
- **`TableId` is restored from catalog**, not regenerated — batch directory paths remain stable.
- **`_ioLock`** serializes hydration, catalog flush, and batch writes.
- **Mmap path** registers each mapped batch with `RainDbMappedBatchMemoryManager` and `MemoryTable.AttachMappedBatch`.

## 14.15 catalog.json format

### 14.15.1 Document types

```csharp
internal sealed class RainDbCatalogDocument
{
    public int FormatVersion { get; set; }
    public List<RainDbTableDocument> Tables { get; set; } = new();
}

internal sealed class RainDbTableDocument
{
    public string Id { get; set; } = "";
    public string Name { get; set; } = "";
    public List<RainDbColumnDocument> Columns { get; set; } = new();
}

internal sealed class RainDbColumnDocument
{
    public string Name { get; set; } = "";
    public string Type { get; set; } = "";
}
```

### 14.15.2 Example catalog.json

```json
{
  "formatVersion": 1,
  "tables": [
    {
      "id": "a1b2c3d4e5f6789012345678abcdef01",
      "name": "sales",
      "columns": [
        { "name": "region", "type": "Utf8" },
        { "name": "amount", "type": "Float64" }
      ]
    }
  ]
}
```

Serializer options:

```csharp
private static readonly JsonSerializerOptions JsonOptions = new()
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
    PropertyNameCaseInsensitive = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
};
```

### 14.15.3 Schema reconstruction

```csharp
private static TableSchema ToTableSchema(RainDbTableDocument t)
{
    var list = t.Columns ?? [];
    if (list.Count == 0)
        throw new InvalidDataException($"Table '{t.Name}' has no columns in catalog.");
    var cols = new ColumnDef[list.Count];
    for (var i = 0; i < list.Count; i++)
    {
        var c = list[i];
        if (!Enum.TryParse<RainDbType>(c.Type, ignoreCase: false, out var rt))
            throw new InvalidDataException($"Unknown column type '{c.Type}' for column '{c.Name}'.");
        cols[i] = new ColumnDef(c.Name, rt);
    }
    return new TableSchema(cols);
}
```

Column type strings must match `RainDbType` enum names exactly (`Utf8`, `Int32`, `Float64`, …). Case-sensitive parse is intentional — corrupt catalogs fail loudly.

### 14.15.4 FlushCatalog

```csharp
public void FlushCatalog()
{
    lock (_ioLock)
    {
        var doc = BuildCatalogDocument(Catalog);
        WriteCatalogAtomic(doc);
    }
}

private void WriteCatalogAtomic(RainDbCatalogDocument doc)
{
    var catalogPath = Path.Combine(RootDirectory, CatalogFileName);
    var json = JsonSerializer.Serialize(doc, JsonOptions);
    _atomicWriter.WriteAllText(catalogPath, json);
}
```

`BuildCatalogDocument` includes only `MemoryTable` entries. Table names are sorted case-insensitively for deterministic JSON. `ExportCatalog` uses the same atomic writer for catalog and batches.

## 14.16 Creating persistent tables

```csharp
public MemoryTable CreateMemoryTable(string name, TableSchema schema, TableId? id = null, bool strictVectorChunkRows = false)
{
    var opts = new MemoryTableOptions(strictVectorChunkRows, BatchPersistence: this);
    var table = new MemoryTable(name, schema, id, opts);
    Catalog.Register(table);
    FlushCatalog();
    return table;
}
```

Flow:

1. Construct `MemoryTable` with `BatchPersistence = this`
2. Register in catalog (in-memory)
3. Write `catalog.json` immediately so the table exists on disk even before first batch

Optional `TableId` parameter supports deterministic IDs in tests or migrations.

## 14.17 MemoryTable append hooks

### 14.17.1 AppendCore

```csharp
// src/RainDB.Core/Tables/MemoryTable.cs
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

**Rollback on persistence failure.** If disk write throws, the in-memory batch is removed so memory and disk cannot diverge silently within the process.

### 14.17.2 Append vs AppendHydratedBatch

| Method | Persists? | Use case |
|--------|-----------|----------|
| `AppendBatch` | Yes (if hook set) | Normal inserts |
| `AppendHydratedBatch` (internal) | No | Loading from disk |

```csharp
public void AppendBatch(IColumnarBatch batch) => AppendCore(batch, notifyPersistence: true);
internal void AppendHydratedBatch(IColumnarBatch batch) => AppendCore(batch, notifyPersistence: false);
```

Hydration loads existing files without rewriting them.

### 14.17.3 IRainDbBatchPersistence — batch then catalog

```csharp
void IRainDbBatchPersistence.OnBatchAppended(TableId tableId, string tableName, int zeroBasedBatchIndex, IColumnarBatch batch)
{
    var path = GetBatchPath(tableId, zeroBasedBatchIndex);
    lock (_ioLock)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);
        WriteBatchFile(path, batch);
        FlushCatalog();
    }
}
```

Each successful append writes the new `.batch` atomically, then rewrites `catalog.json`. The catalog does not list individual batch files (only table schema and id), but flushing keeps on-disk metadata consistent with registered tables after every durable segment — including batch count implied by filesystem layout and any future catalog fields.

Batch path pattern:

```csharp
private string GetBatchPath(TableId tableId, int zeroBasedBatchIndex) =>
    Path.Combine(RootDirectory, TablesDirectoryName, tableId.ToString(), $"{zeroBasedBatchIndex:D6}.batch");
```

Six-digit zero padding keeps lexical sort order aligned with append order.

### 14.17.4 Atomic batch write and codec options

```csharp
private void WriteBatchFile(string path, IColumnarBatch batch)
{
    var codecOptions = new RainDbBatchCodecOptions
    {
        EnableInt32DictionaryEncoding = _options.EnableInt32DictionaryEncoding,
    };
    _atomicWriter.WriteStream(path, fs => RainDbBatchBinaryCodec.WriteBatch(fs, batch, codecOptions));
}
```

## 14.18 RainDbBatchBinaryCodec overview

`RainDbBatchBinaryCodec` in `src/RainDB.Core/Persistence/RainDbBatchBinaryCodec.cs` implements batch format **version 1**, little-endian throughout.

Supported column kinds:

| Kind byte | Name | In-memory type |
|-----------|------|----------------|
| `1` | Fixed | `FixedWidthColumnChunk` |
| `2` | UTF-8 Arrow | `Utf8ColumnChunk` |
| `3` | UTF-8 length-prefixed | `Utf8LengthPrefixedColumnChunk` |
| `4` | Dict Int32 | `DictionaryEncodedInt32ColumnChunk` |

### 14.18.1 Batch header

```text
Offset  Size  Field
------  ----  -----
0       8     Magic "RNBATCH1"
8       4     formatVersion (uint32 = 1)
12      4     rowCount (int32)
16      4     columnCount (int32)
20      ...   column payloads (columnCount times)
```

Write path:

```csharp
public static void WriteBatch(Stream destination, IColumnarBatch batch)
{
    destination.Write(Magic);
    WriteU32(destination, FormatVersion);
    WriteI32(destination, batch.RowCount);
    WriteI32(destination, batch.Columns.Count);
    for (var i = 0; i < batch.Columns.Count; i++)
        WriteColumn(destination, batch.Columns[i], batch.RowCount);
}
```

Decode validates magic, version, row/column counts, consumes all columns, and asserts no trailing bytes:

```csharp
if (o != data.Length)
    throw new InvalidDataException($"Batch buffer has {data.Length - o} trailing byte(s) after column data.");
```

### 14.18.2 Column payloads by kind

**Kind 1 — fixed-width:** `kind`, `physicalType`, `hasNulls`, `rowCount`, `valuesLength`, raw values, optional null bitmap.

**Kind 2 — UTF-8 Arrow:** `kind`, `hasNulls`, `rowCount`, `offsetsLength`, offset table, `blobLength`, blob, optional null bitmap.

**Kind 3 — UTF-8 length-prefixed:** `kind`, `hasNulls`, `rowCount`, `payloadLength`, payload, optional null bitmap.

**Kind 4 — dictionary Int32 (`KindDictInt32`):** `kind`, `hasNulls`, `rowCount`, `dictLength`, dictionary `Int32` values, `indexWidthBytes` (1/2/4), `indicesLength`, index bytes, optional null bitmap.

On write, plain `FixedWidthColumnChunk` with `RainDbType.Int32` may be rewritten to kind `4` when `EnableInt32DictionaryEncoding` is true and `Int32DictionaryColumnEncoder.TryEncode` succeeds (heuristic: at least four rows, dictionary size ≤ 65535, encoded bytes strictly smaller than raw). Already-encoded `DictionaryEncodedInt32ColumnChunk` instances write kind `4` directly.

### 14.18.3 Int32DictionaryColumnEncoder (summary)

The encoder scans non-null values, builds a distinct-value map, chooses 1- or 2-byte indices when dictionary size permits, and compares total encoded size against `rowCount * 4`. Null rows still participate in the null bitmap; index slots for null rows are zero-filled and ignored by decode semantics for null bits.

### 14.18.4 EncodeBatch helper

```csharp
public static byte[] EncodeBatch(IColumnarBatch batch)
{
    using var ms = new MemoryStream();
    WriteBatch(ms, batch);
    return ms.ToArray();
}
```

Tests and in-memory round-trips use `EncodeBatch` / `DecodeBatch` without touching the filesystem.

## 14.19 Loading batches — mmap and fallback

```csharp
private void LoadBatchesIntoTable(MemoryTable table, bool preferMmap)
{
    var dir = Path.Combine(RootDirectory, TablesDirectoryName, table.Id.ToString());
    if (!Directory.Exists(dir))
        return;
    foreach (var file in Directory.GetFiles(dir, "*.batch").OrderBy(f => f, StringComparer.Ordinal))
    {
        if (!RainDbAtomicFileWriter.IsCommittedBatchFileName(Path.GetFileName(file)))
            continue;
        if (preferMmap)
        {
            try
            {
                var mapped = _mmapReader.Open(file);
                table.AppendHydratedBatch(mapped.Batch);
                _mappedBatchMemory.RegisterMappedBatch(table, table.Batches.Count - 1, mapped, file);
                continue;
            }
            catch (IOException) { }
            catch (UnauthorizedAccessException) { }
        }

        var bytes = File.ReadAllBytes(file);
        table.AppendHydratedBatch(RainDbBatchBinaryCodec.DecodeBatch(bytes));
    }
}
```

### RainDbBatchMmapReader and MappedColumnarBatch

`RainDbBatchMmapReader.Open(path)` memory-maps the entire `.batch` file, parses the `RNBATCH1` header in-place, and builds a `ColumnarBatch`. Fixed-width columns (kind 1) expose `Values` and null bitmaps as slices into the map. UTF-8 and dictionary Int32 columns copy their variable portions into managed arrays (see Chapter 16).

`MappedColumnarBatch` wraps:

- `Batch` — the hydrated `IColumnarBatch` held in `MemoryTable._batches`
- A private `IDisposable` that releases the mmap view when disposed

`MemoryTable.AttachMappedBatch(batchIndex, mapped)` keeps the handle alive while the table references mapped slices. **Scans must not outlive an evicted map** — the memory manager coordinates eviction.

`ImportCatalog` uses a static loader with `preferMmap: true` but does not attach the database memory manager (no budget/LRU until you open via `RainDbFileDatabase`).

## 14.20 RainDbMappedBatchMemoryManager

The manager implements `IMappedBatchScanObserver` and tracks resident mmap bytes per database.

```csharp
public void RegisterMappedBatch(MemoryTable table, int batchIndex, MappedColumnarBatch mapped, string batchFilePath)
{
    // Charge byte estimate (sum of column Values + NullBitmap lengths, min 4096)
    // If budget exceeded:
    //   Fail → throw InvalidOperationException
    //   EvictColdBatches → evict LRU tail until under budget
    table.AttachMappedBatch(batchIndex, mapped);
}

public void OnBatchScanned(TableId tableId, int batchIndex)
{
    // Move entry to front of LRU (touch)
}
```

**EvictColdBatches:** remove the least-recently scanned mapped batch, `DecodeBatch(ReadAllBytes(path))` into RAM, `ReplaceHydratedBatchAt`, dispose `MappedColumnarBatch`, `DetachMappedBatch`. SQL results stay correct; only the backing store changes from mmap to heap.

**Fail:** `RegisterMappedBatch` throws if adding the segment would exceed `MappedBatchMemoryBudgetBytes`.

`RainDbEngine` wires `fileDb.MappedBatchScanObserver` into `RainDbExecutionContext`; `VectorizedScanEngine` calls `OnBatchScanned` when a persistent table batch is scanned.

Expose metrics via `fileDb.MappedBatchMemory.ResidentBytes` and `BudgetBytes`.

## 14.21 Export and import

### 14.21.1 ExportCatalog

Creates a fresh on-disk snapshot from any `ICatalog`:

```csharp
public static void ExportCatalog(ICatalog catalog, string rootDirectory)
{
    var root = Path.GetFullPath(rootDirectory);
    Directory.CreateDirectory(root);
    var tablesRoot = Path.Combine(root, TablesDirectoryName);
    if (Directory.Exists(tablesRoot))
        Directory.Delete(tablesRoot, recursive: true);
    Directory.CreateDirectory(tablesRoot);

    var doc = new RainDbCatalogDocument { FormatVersion = 1 };
    foreach (var name in catalog.TableNames.OrderBy(n => n, StringComparer.OrdinalIgnoreCase))
    {
        // ... build table entry, write all batches ...
        for (var i = 0; i < col.Batches.Count; i++)
        {
            var batchPath = Path.Combine(tableDir, $"{i:D6}.batch");
            writer.WriteStream(batchPath, fs => RainDbBatchBinaryCodec.WriteBatch(fs, col.Batches[i]));
        }
    }
    writer.WriteAllText(catalogPath, JsonSerializer.Serialize(doc, JsonOptions));
}
```

Export **wipes** existing `tables/` under the target root. Batch writes use the same atomic stream helper as live appends.

### 14.21.2 ImportCatalog

```csharp
public static InMemoryCatalog ImportCatalog(string rootDirectory)
{
    if (!RainDbAtomicFileWriter.TryReadCommittedCatalogJson(root, out var json))
        throw new FileNotFoundException(...);
    // MemoryTable without BatchPersistence; LoadBatchesIntoTableStatic(..., preferMmap: true)
    return catalog;
}
```

Import returns an in-memory catalog **without** automatic persistence or mmap budget tracking.

## 14.22 End-to-end persistence test pattern

From `src/RainDB.Tests/RainDbPersistenceTests.cs`:

```csharp
[Fact]
public async Task OpenPersistent_append_then_reopen_restores_batches()
{
    var root = Path.Combine(Path.GetTempPath(), "raindb_persist_" + Guid.NewGuid().ToString("N"));
    var engine = RainDbEngine.OpenPersistent(root);
    var fileDb = engine.FileDatabase!;
    var table = fileDb.CreateMemoryTable("sales", schema);
    table.AppendBatch(new ColumnarBatch(2, chunks));

    await using var r1 = await engine.ExecuteSqlAsync("SELECT COUNT(*) FROM sales");
    // assert count == 2

    var engine2 = RainDbEngine.OpenPersistent(root);
    await using var r2 = await engine2.ExecuteSqlAsync("SELECT COUNT(*) FROM sales");
    // assert count == 2 after reopen
}
```

Codec unit test covers all three UTF-8/fixed chunk kinds independently of filesystem.

## 14.23 IO locking model

`RainDbFileDatabase` uses a single `_ioLock` for:

- Hydration
- Catalog flush
- Batch append writes

Concurrent appends from multiple threads will serialize on this lock. Concurrent **reads** through `MemoryTable.Batches` during append are safe for RainDB's single-writer assumption but not a documented multi-writer contract.

## 14.24 Memory and performance characteristics

| Operation | Default behavior | Fallback |
|-----------|------------------|----------|
| Open / hydrate | mmap `.batch`; kind-1 columns zero-copy in process | `ReadAllBytes` + `DecodeBatch` on mmap failure |
| Append | Atomic batch write + `FlushCatalog` | — |
| Catalog update | Full JSON rewrite (atomic) | — |
| Query | Scan mapped or heap chunks; LRU touch on mmap batches | Evicted batches are fully decoded in RAM |

Dictionary Int32 columns still allocate index/dictionary bytes on mmap hydrate; accessing `Values` may materialize a full Int32 array (Chapter 4).

## 14.25 Durability semantics

What you can rely on today:

- `RainDbAtomicFileWriter` temp-rename for catalog and batches; `.tmp` files ignored on read
- Catalog flush after each persisted batch append (same lock as batch write)
- In-memory rollback if persistence throws after `AppendBatch`
- Stable `TableId` and batch indices across reopen

What is **not** fully specified yet:

- fsync / media flush policy
- Automated repair for orphan or missing batch files

WAL and stronger cross-file transactional guarantees remain Phase F (Chapter 15).

## 14.26 Relationship to RNBFCOL1 column files

`ColumnarFixedWidthFileFormat` (`RNBFCOL1`, Chapter 16) mmap **one column per file**. `RNBATCH1` mmap **one multi-column segment per file** and only avoids copying kind-1 payloads. Sidecar column files remain optional tooling — batch persistence is the primary table storage path.

## 14.27 Operational recipes

- **Backup:** `RainDbFileDatabase.ExportCatalog(engine.Catalog, path)` or copy the database root while idle
- **Clone in-memory → disk:** `ExportCatalog` then `RainDbEngine.OpenPersistent`
- **Inspect batch:** `RainDbBatchBinaryCodec.DecodeBatch(File.ReadAllBytes("tables/{id}/000000.batch"))`
- **Fresh DB:** `RainDbEngine.OpenPersistent(tempDir)` + `FileDatabase.CreateMemoryTable`

## 14.28 Errors and format evolution

| Failure | Exception |
|---------|-----------|
| Bad batch magic | `InvalidDataException` |
| Unsupported format version | `InvalidDataException` / `NotSupportedException` |
| Schema mismatch on append | `ArgumentException` from `MemoryTable` |
| Unknown column type in catalog | `InvalidDataException` |
| Export non-columnar table | `InvalidOperationException` |

When changing on-disk formats: bump `formatVersion`, keep decode for prior version or ship a migration tool, add `RainDbPersistenceTests` round-trips, and assign new column kind bytes without reusing old IDs.

## 14.29 Summary

**Part I** covered storage engine layering, WAL and checkpoint theory, page vs log-structured organizations, industry columnar formats, catalog durability, crash recovery, and snapshot import/export concepts. **Part II** described RainDB's current directory-backed engine: `RainDbAtomicFileWriter` commits `catalog.json` and `######.batch` files while ignoring `*.tmp`; `RainDbFileDatabaseOptions` controls mmap hydration, Int32 dictionary encoding on write, and mmap memory budget (`EvictColdBatches` vs `Fail`); `RainDbBatchMmapReader` and `MappedColumnarBatch` tie batch lifetime to `MemoryTable` and `RainDbMappedBatchMemoryManager`; scans refresh LRU via `IMappedBatchScanObserver`. See Chapter 16 for mmap mechanics and the distinction from per-column `RNBFCOL1` files.
