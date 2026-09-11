---
title: "Chapter 15: Future Extensions"
order: 15
---

# Chapter 15: Future Extensions

RainDB is an embedded OLAP prototype with a real execution stack and explicit stubs for production features. This chapter opens with what mature analytical engines require — cost-based optimization, statistics, spill, observability, and format evolution — at a conceptual level. Part II maps those ideas onto RainDB's phased roadmap, existing interfaces, and extension hooks in the repository.

---

# Part I — Concepts

## 15.1 What production OLAP engines require

A working prototype that scans in-memory column batches proves core ideas; shipping analytics software demands more. Whether the engine embeds in-process like DuckDB or runs as a cluster service like ClickHouse or Snowflake, mature products converge on overlapping capabilities beyond raw SQL and storage.

### Architectural patterns (conceptual)

| Engine | Distinctive pattern |
|--------|---------------------|
| **DuckDB** | In-process vectorized execution, strong compression, integrated buffer management, portable single-file storage |
| **ClickHouse** | MergeTree append-only parts, per-column compression, sharding, materialized views |
| **Snowflake** | Separated storage and compute, immutable micro-partitions, elastic cloud warehouses |

The implementations differ, but the **underlying problems** recur:

1. **Reuse compiled plans** — pay compile cost once, execute often
2. **Bound memory** — spill hash structures and sorts when RAM is insufficient
3. **Reduce I/O** — prune columns, compress data, mmap cold segments, cache hot pages
4. **Observe execution** — traces, metrics, slow-query logs
5. **Evolve formats safely** — versioned files with backward-compatible readers

RainDB's roadmap (Part II) orders work by dependency: tighten in-memory operators before expanding SQL; harden storage before WAL; collect statistics before cost-based join ordering.

## 15.2 Cost-based optimization theory

**Cost-based optimization (CBO)** ranks logically equivalent plans by estimated resource use — pages read, CPU time, memory footprint — instead of relying solely on fixed rewrite rules.

### Plan space

Joining *N* tables admits *N*! orderings. Practical optimizers explore a subset using:

- **Dynamic programming** (System R heritage, PostgreSQL-style) — exploit optimal substructure over join subsets
- **Greedy or heuristic search** — repeatedly merge the cheapest pair (common at warehouse scale)
- **Genetic search** — evolutionary exploration for very large join graphs (historically in SQL Server)

### Cost model ingredients

```text
cost(scan) ≈ pages_to_read × seq_page_cost + rows × cpu_per_row
cost(hash_join) ≈ build_side_pages + probe_side_pages + hash_table_memory
cost(sort) ≈ n log n × comparison_cost  (or spill-aware variant)
```

Accurate **selectivity estimates** for filters and joins dominate plan quality: what fraction of rows survives each predicate?

### Statistics dependency

Without statistics, CBO collapses to uniform-distribution guesses. Production systems run `ANALYZE` (or equivalent) to gather:

- Per-table row counts
- Per-column NDV (number of distinct values)
- Min/max for range filters
- Histograms where distributions skew

RainDB has no CBO yet — `DefaultSqlCompiler` always picks hash join. Phase B introduces heuristics; Phase E feeds statistics into better choices.

## 15.3 Statistics and histograms

### Table-level statistics

- **Row count** — base cardinality or estimated post-filter size
- **Average row width** — bytes per row for memory budgeting

### Column-level statistics

| Statistic | Use |
|-----------|-----|
| **NDV** | Join cardinality estimates; hash table sizing |
| **Null fraction** | Selectivity of `IS NULL` / `IS NOT NULL` |
| **Min / max** | Range pruning; zone-map style skipping |
| **Histogram** | Skewed domains (e.g., most rows tagged `US`) |

### Histogram types

- **Equi-width** — buckets cover equal value ranges
- **Equi-depth** — buckets hold roughly equal row counts (better under skew)

Given `WHERE region = 'EU'`, the optimizer locates the matching bucket and estimates hits as `table_rows × bucket_fraction`.

### Maintenance

Bulk loads and heavy updates stale statistics. Engines schedule `ANALYZE` automatically or on demand. Plan caches should drop entries when statistic generations change materially.

## 15.4 Spill to disk algorithms

Hash aggregation and hash join assume the working set fits in RAM. When it does not, engines **spill** partitions to temporary storage and finish in multiple passes.

### Hash aggregation spill

1. Hash group keys into *k* partitions
2. Retain some partitions in memory; write others to disk
3. Reprocess spilled partitions recursively (possibly re-partitioning with larger *k*)

**Grace hash join** applies the same partition idea so only one build/probe pair must reside in memory at once.

### External sort

When an in-memory sort exceeds budget:

1. **Run generation** — sort manageable chunks, emit sorted runs to disk
2. **K-way merge** — combine runs with a bounded heap

When only the top *k* rows matter, a size-*k* heap can avoid a full external sort — RainDB Phase A2 targets this for `ORDER BY … LIMIT`.

### Spill file format

Spill files are usually columnar or length-prefixed binary with partition metadata. Merge passes should be **deterministic** so OLAP results stay reproducible.

RainDB wires `ISpillWriter` today but only emits JSON telemetry — durable spill lands in Phase E2.

## 15.5 Observability in databases

Running analytics blind is expensive. Production databases instrument both compile and execute paths.

### Tracing

End-to-end traces stitch parse, bind, optimize, and per-operator execute spans. On .NET, `ActivitySource` pairs naturally with OpenTelemetry exporters.

High-value attributes: hashed SQL fingerprint (not raw text when PII is a concern), referenced tables, row counts, operator kinds, bytes spilled.

### Metrics

Typical counters and histograms:

- Query throughput, compile latency, execute latency
- Plan-cache and buffer-pool hit rates
- Spill volume, bytes scanned
- Active queries and queue depth

### Slow query log

Queries above a latency threshold log redacted SQL, plan text, and phase timings — separating bad plans from resource starvation.

### Profiling

Per-operator CPU sampling (`EXPLAIN ANALYZE` in PostgreSQL) reveals where time goes. Pipeline-oriented OLAP engines may attribute stalls to I/O versus CPU.

RainDB targets Phase F3 observability (`ActivitySource` hooks appear in roadmap documentation).

## 15.6 Format versioning strategy

On-disk layouts survive many releases. A deliberate versioning policy prevents silent corruption and eases upgrades.

### Principles

1. **Magic plus version in every header** — unknown versions fail fast with a clear error
2. **Never recycle type/kind identifiers** — only append new enum values
3. **Ship N−1 readers** — support the previous format for at least one release cycle
4. **Decouple catalog version from payload version** — self-describing segment files can evolve independently of `catalog.json`
5. **Provide migration paths** — export/import or offline `migrate` for breaking changes
6. **Reserve header bytes** — compression codec ids, checksums, feature flags

### Compatibility matrix (example)

| Change | Strategy |
|--------|----------|
| New column chunk kind | Bump batch `formatVersion`; older readers error explicitly |
| Optional catalog field | Ignore unknown JSON keys (forward compatible) |
| Different null-bitmap packing | New magic or version 2; retain version 1 decoder |

RainDB today: `formatVersion: 1` in `catalog.json`, `RNBATCH1` batch magic, `RNBFCOL1` mmap columns. Part II maps extension hooks on the roadmap.

## 15.7 Query scheduling and resource governance

Not every query should grab all cores and all RAM. A **scheduler** allocates threads, memory, and I/O bandwidth across concurrent work.

### Workload classes

| Class | Typical policy |
|-------|----------------|
| Interactive | Cap latency and DOP; prioritize small queries |
| Batch ETL | Favor throughput; tolerate spill; schedule off-peak |
| Maintenance | Background `ANALYZE`, compaction, checkpoint |

RainDB exposes parallelism through `VectorizedScanExecutionOptions` today; a future **resource manager** would reconcile per-query DOP with a global memory ceiling and spill pressure.

### Admission control

When budgets are exhausted, engines **queue** or **reject** new queries instead of risking OOM. Track `queries_queued` and `queries_rejected_resource_limit`.

### Cancellation

Long scans should respect `CancellationToken` — RainDB threads tokens through `ExecuteSqlAsync` and operator loops. Hosted systems also cancel on client disconnect or administrative `KILL QUERY`.

## 15.8 Distributed and disaggregated patterns (conceptual)

Cluster OLAP partitions tables across machines. RainDB remains single-process, but the vocabulary matters for later extension.

### Shared-nothing partitioning

Each node holds a shard. Plans gain **exchange** operators — shuffle, broadcast — to move batches between workers. ClickHouse distributed tables and Snowflake micro-partitions instantiate this at different scales.

### Disaggregated storage

Stateless compute nodes read immutable objects from S3-like storage while a catalog service maps tables to partition lists. Elastic scale and cheap durability trade against remote-read latency unless caching is aggressive.

### Why embedded engines defer distribution

Networking, fault tolerance, and distributed consistency dwarf single-node engineering. Proving correctness and performance locally first — DuckDB followed this arc — is deliberate. RainDB's roadmap stays single-process until Phase G ecosystem goals justify cluster modes.

## 15.9 Adaptive and feedback-driven execution

**Adaptive query processing** adjusts plans mid-flight using observed cardinalities:

1. If a hash-join build side explodes past estimates, switch strategy or spill sooner
2. Reorder join inputs after the first scan reveals true row counts
3. Choose broadcast versus shuffle in distributed plans based on runtime sizes

Adaptive switches need **checkpointable operator state** and safe replanning boundaries — substantial complexity. RainDB schedules this for Phase E4, after real spill (E2) and statistics (E3). Without baseline estimates, runtime feedback has little to compare against.

## 15.10 Security and multi-tenancy hooks

Serving SQL to untrusted callers — even through an embedded host — demands guardrails:

- **Parser fuzzing** — malformed input must not hang or crash the process
- **Query cost caps** — limits on rows scanned, spill bytes, compile time
- **Tenant isolation** — per-tenant catalog views (distant future for RainDB)

These concerns span the roadmap's security track. They need not block core engine milestones but must be designed before exposing arbitrary SQL endpoints.

---

# Part II — RainDB

RainDB is an actively developed embedded OLAP prototype. Much of the execution stack is real — columnar storage, vectorized scan, hash aggregation, inner joins, sort/limit, strict SQL compilation, and directory-backed persistence. Other surfaces are **interfaces and stubs** that document where the engine is headed without pretending the feature ships today.

This part collects those extension points: the phased development roadmap, spill infrastructure, the LINQ compiler stub, plan-cache hooks, column encodings, observability plans, and a practical guide for extending this book when new phases land.

## 15.11 How to read the roadmap

`docs/Development-Roadmap.md` sequences work by **dependency**, not calendar quarters. Phases are labeled A through G:

| Phase | Theme | Depends on |
|-------|-------|------------|
| **A** | Execution efficiency (in-memory core) | Current operator baseline |
| **B** | Planning and compile-once semantics | Logical IR + physical plans |
| **C** | Storage, I/O, memory budget | Persistence MVP (Chapter 14) |
| **D** | SQL and analytics surface | A + B for performance headroom |
| **E** | Advanced analytics (windows, real spill, stats) | A, B, partial C |
| **F** | Production durability and concurrency | C |
| **G** | Ecosystem (LINQ, Parquet, API stability) | D + stable formats |

Guiding principles (unchanged across phases):

- One physical-plan IR for SQL, LINQ, and programmatic planners
- Columnar batches as the unit of parallel work (morsels)
- Deterministic OLAP ordering where semantics require it
- Minimize allocations on hot paths; isolate unsafe SIMD behind clear boundaries

```text
Phase A (fast in-memory ops)
    ↓
Phase B (rewrite + prepared plans) ──→ Phase D (more SQL) ──→ Phase E (windows + spill + stats)
    ↓                                      ↑
Phase C (mmap + encodings + durability) ───┘
    ↓
Phase F (WAL / MVCC)
    ↓
Phase G (LINQ + formats + API stability)
```

Phases **A** and **C** may overlap (mmap helps file-backed scans), but **D** should not outrun **A** for heavy operators — new SQL that exposes slow join/sort paths only widens the performance gap.

## 15.12 Phase A — Execution efficiency

**Goal:** Make existing operators fast and memory-safe at scale before adding much new SQL.

### A1. Vectorized selection and projection — implemented

Documented in Chapter 8. `SelectionEvaluator` intersects dense row-index vectors; `ProjectGather` uses pooled output chunks.

### A2. True top-N and sort discipline — planned

Today `SortTopNPhysicalPlan` may sort more rows than necessary. Target: partial sort / heap top-N when `LIMIT k` is small, retaining full sort only when global order is required.

**Exit criteria:** `ORDER BY … LIMIT k` memory bounded by O(k), not O(n).

### A3. Join and grouped-join memory model — planned

Reduce peak RSS by streaming join output and feeding join batches directly into hash aggregation instead of materializing full match lists.

### A4. SIMD and native boundaries — partial

`AggregateIntrinsics` and AVX2 double sum exist. Extend min/max, integer sums, hash combine helpers. Optional `RainDB.Native` only when C# intrinsics are insufficient.

### A5. Micro-benchmarks — planned

BenchmarkDotNet projects for scan, filter+project, hash agg, hash join, sort/top-N with documented baselines.

**Phase A out of scope:** New SQL constructs, WAL, Parquet, cost-based optimizer.

## 15.13 Phase B — Planning and compile-once

**Goal:** Reduce per-query overhead without a full cost-based optimizer.

### B1. Logical rewrite (rule-based optimizer v0)

Single pass over logical IR:

- Predicate pushdown to scan/join sides
- Projection pruning (drop unread columns early)
- Limit pushdown where legal

Keep rules testable with golden logical plans before/after rewrite.

### B2. Physical choices (heuristics)

Today `DefaultSqlCompiler` hard-codes hash join:

```csharp
LogicalInnerJoin j => LogicalJoinBinder.BindAndLower(j, catalog, PhysicalJoinAlgorithm.Hash, _defaultScanOptions),
```

Future: choose hash vs sort-merge using sorted-input hints or row-count estimates. `IPhysicalPlan.Explain()` must show the choice.

### B3. Prepared execution and plan cache

**Compile once:** parse + bind + optional rewrite → cached `IPhysicalPlan`.

Cache key candidates:

- Normalized SQL text
- Catalog snapshot id or schema version set
- `VectorizedScanExecutionOptions` fingerprint

**Invalidate** when any referenced table's `SchemaVersion` changes:

```csharp
// src/RainDB.Core/Tables/MemoryTable.cs
public int SchemaVersion => Volatile.Read(ref _schemaVersion);

public event EventHandler<SchemaVersionChangedEventArgs>? SchemaVersionChanged;

public int BumpSchemaVersion()
{
    var v = Interlocked.Increment(ref _schemaVersion);
    SchemaVersionChanged?.Invoke(this, new SchemaVersionChangedEventArgs(v));
    return v;
}
```

Prepared statement API sketch (not implemented):

```csharp
// (planned) src/RainDB.Sql/Compilation/PreparedStatement.cs
public sealed class PreparedStatement
{
    public IPhysicalPlan Plan { get; }
    public int CompiledAtSchemaVersion { get; }
    public void BindParameters(IReadOnlyDictionary<string, object> values);
}
```

Second execution of the same prepared statement should skip `SqlParser.Parse` and binder work; only parameter slots in `ColumnCompareFilter` change.

### B4. Explain and introspection

Structured `EXPLAIN` returning logical + physical text. `LogicalTableScan.Explain()` and `HashAggregatePhysicalPlan.Explain()` already produce strings — SQL `EXPLAIN` would surface them.

Optional `EXPLAIN ANALYZE` with operator timers (can start as no-op stubs).

**Phase B out of scope:** Histogram join ordering, adaptive mid-query re-optimization.

## 15.14 Phase C — Storage, I/O, memory budget

**Goal:** Stop treating durability as "load everything into `MemoryTable` byte arrays."

### C1. mmap-first column segments — designed, partial code

`ColumnarFixedWidthMmapReader` and `ColumnarFixedWidthFileFormat` exist in `RainDB.Core/IO/` (Chapter 16). Integration work:

- Hydration maps batch files instead of `File.ReadAllBytes` where format allows
- Scan engine reads through mapped spans without extra copy

### C2. Buffer manager v0

Central policy for mapped regions, pin/unpin, configurable memory cap, naive LRU eviction per table.

### C3. Column encodings

Dictionary encoding and lightweight integer compression on write; transparent decode in scan.

Extension requirements:

| Layer | Change |
|-------|--------|
| Chunk types | New `IColumnChunk` implementations or encoded wrappers |
| `RainDbBatchBinaryCodec` | New kind bytes; bump batch `formatVersion` |
| Scan kernels | Decode to pooled buffers or operate on encoded data |
| Tests | Round-trip + correctness on encoded fixtures |

Example future kind byte (illustrative, not in repo):

```text
KindDictionaryUtf8 = 4
  → indices column + dictionary blob + null bitmap
```

### C4. Durability hardening (pre-WAL)

Formalize ordering: batch durable before catalog references it (or defined recovery on open). Crash tests between catalog and batch writes.

**Phase C out of scope:** Full WAL/MVCC (Phase F).

## 15.15 Phase D — SQL and analytics surface

Sequence inside Phase D (each slice: parser → logical IR → binder → physical plan → tests):

| Slice | Features |
|-------|----------|
| D1 | Scalar expressions: arithmetic, `CASE`, casts |
| D2 | `HAVING`, `DISTINCT`, `COUNT(DISTINCT)`, broader `MIN`/`MAX` types |
| D3 | Outer joins, `UNION ALL` / `UNION` |
| D4 | Subqueries: `IN`, `EXISTS` (uncorrelated first) |
| D5 | `ORDER BY` / `LIMIT` with `GROUP BY` |
| D6 | `DATE`/`TIMESTAMP`, `COALESCE`, `LIKE` |

**Exit criteria:** Programming Guide strict SQL section updated; `samples/sql/` grows per milestone.

**Out of scope:** Window functions (Phase E), nested struct/array types.

## 15.16 Phase E — Advanced analytics

### E1. Window functions

`ROW_NUMBER`, `RANK`, `SUM() OVER (PARTITION BY … ORDER BY …)` with default framing rules. Builds on sort infrastructure from Chapter 12.

### E2. External spill (real)

Extend `ISpillWriter` beyond metrics JSON to partition and merge aggregate/join state under memory pressure.

### E3. Statistics

`ANALYZE table`: row counts, NDV, min/max; optional histograms feeding Phase B heuristics.

### E4. Adaptive execution (optional)

Runtime cardinality feedback to choose spill or join strategy — only after E2 and E3 exist.

## 15.17 Phase F — Production durability and concurrency

### F1. WAL + checkpoint

Append-only log for batch/catalog changes; periodic checkpoint to segment files; recovery on open.

### F2. MVCC or snapshot reads

Single-writer multi-reader is a reasonable first target for analytics workloads.

### F3. Observability

- `ActivitySource` spans for compile and execute phases
- Slow-query threshold and structured plan logging
- Integration with .NET diagnostics (`dotnet-counters`, OpenTelemetry exporters)

Sketch:

```csharp
// (planned) src/RainDB.Query/Runtime/QueryActivitySource.cs
internal static class QueryActivitySource
{
    public static readonly ActivitySource Source = new("RainDB.Query", "1.0.0");

    public static Activity? StartCompile(string sql) =>
        Source.StartActivity("raindb.compile", ActivityKind.Internal);
}
```

**Exit criteria:** Crash during write recovers to last committed state; concurrency model documented.

## 15.18 Phase G — Ecosystem

### G1. LINQ provider (see §15.20)

### G2. Ingest and interchange

CSV and Parquet load paths; optional Arrow IPC interop aligned with `Utf8ColumnChunk` layouts.

### G3. API stability

Version physical plan and on-disk formats once logical IR stabilizes after Phase D; publish migration notes.

## 15.19 ISpillWriter today

Spill is an execution hook, not a finished feature:

```csharp
// src/RainDB.Abstractions/Execution/ISpillWriter.cs
public interface ISpillWriter
{
    bool IsEnabled { get; }
    ValueTask SpillChunkAsync(ReadOnlyMemory<byte> chunk, CancellationToken cancellationToken = default);
}

public sealed class NoOpSpillWriter : ISpillWriter
{
    public static NoOpSpillWriter Instance { get; } = new();
    public bool IsEnabled => false;
    public ValueTask SpillChunkAsync(ReadOnlyMemory<byte> chunk, CancellationToken cancellationToken = default) =>
        ValueTask.CompletedTask;
}
```

`RainDbEngine` accepts optional `ISpillWriter`; default is `NoOpSpillWriter.Instance`:

```csharp
SpillWriter = spillWriter ?? NoOpSpillWriter.Instance;
```

`IExecutionContext.SpillWriter` flows into operators. `HashAggregatePhysicalPlan` exposes `SpillPartialEntryThreshold`:

```csharp
/// When positive and IExecutionContext.SpillWriter is enabled, partial maps with at least this many
/// groups invoke ISpillWriter.SpillChunkAsync with a UTF-8 metrics payload (operator still completes in-memory).
public int SpillPartialEntryThreshold { get; }
```

`HashAggregateEngine` emits JSON metrics when threshold exceeded:

```csharp
if (context.SpillWriter.IsEnabled && plan.SpillPartialEntryThreshold > 0)
{
    for (var i = 0; i < n; i++)
    {
        if (partials[i].Count >= plan.SpillPartialEntryThreshold)
        {
            var payload = Encoding.UTF8.GetBytes(
                "{\"op\":\"hash_agg_partial\",\"batch\":" + i +
                ",\"entries\":" + partials[i].Count + "}\n");
            await context.SpillWriter.SpillChunkAsync(payload, ct).ConfigureAwait(false);
        }
    }
}
```

Phase E2 will define a binary spill chunk format and external merge. Until then, custom `ISpillWriter` implementations can log, meter, or tee to files for experiments.

## 15.20 LINQ compiler stub

```csharp
// src/RainDB.Linq/Compilation/DefaultLinqCompiler.cs
public sealed class DefaultLinqCompiler : ILinqCompiler
{
    public ValueTask<IPhysicalPlan> CompileAsync(Expression expression, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(expression);
        cancellationToken.ThrowIfCancellationRequested();
        IPhysicalPlan plan = new ExplainOnlyPhysicalPlan($"LINQ expression: {expression.NodeType}");
        return ValueTask.FromResult(plan);
    }
}
```

`ExplainOnlyPhysicalPlan` is a placeholder:

```csharp
// src/RainDB.Query/Plans/ExplainOnlyPhysicalPlan.cs
public sealed class ExplainOnlyPhysicalPlan : IPhysicalPlan
{
    public ExplainOnlyPhysicalPlan(string label) => Label = label;
    public string Label { get; }
    public string Explain(string indent = "") => $"{indent}{Label}";
}
```

### Target architecture

```text
IQueryable<T>
  → Expression tree (System.Linq.Expressions)
  → RainDB logical IR (LogicalTableScan, LogicalInnerJoin, …)
  → LogicalTableScanBinder / LogicalJoinBinder (reuse)
  → IPhysicalPlan
  → DefaultQueryExecutor
```

### Implementation sketch

1. **Provider root** — `RainDbQueryable<T>` holds `RainDbEngine` + `ICatalog` + root expression
2. **Visitor** — translate `Where`, `Select`, `GroupBy`, `Join` method calls to logical nodes
3. **Name resolution** — map CLR property names to column names (attribute or convention)
4. **Errors** — `LinqCompileException` for unsupported expression shapes (mirroring `SqlCompileException`)

### Why reuse binders?

SQL and LINQ differ only at the front. Once logical IR is built, catalog binding and physical lowering are identical. Tests can share golden physical plans for equivalent SQL and LINQ queries.

## 15.21 Plan cache design notes

No plan cache ships yet. Recommended design when implementing Phase B3:

### Cache entry

```csharp
// (illustrative)
sealed class CachedPlan
{
    public required IPhysicalPlan Plan { get; init; }
    public required IReadOnlyDictionary<TableId, int> SchemaVersionsAtCompile { get; init; }
    public required string NormalizedSql { get; init; }
}
```

### Invalidation

On `SchemaVersionChanged`, drop entries referencing that `TableId`. On catalog unregister, drop entries referencing removed tables.

### Parameter binding

Physical plans store `ColumnCompareFilter` with literal bits at compile time. Prepared execution replaces `ImmediateBits` / `Utf8LiteralBytes` per execution without re-binding column indices.

### Thread safety

Compiler cache should be concurrent dictionary with copy-on-write plan trees (plans are immutable after construction today).

## 15.22 Column encodings roadmap

Encodings reduce disk and memory footprint for repetitive or low-cardinality data.

### Dictionary encoding

Store unique values once; column holds integer indices. Natural fit for UTF-8 region codes and enum-like fields.

Scan options:

- Decode entire vector to pooled chunk per morsel (simpler)
- Compare in encoded space for equality filters (harder)

### Integer compression

Bit-packing, delta encoding, or lightweight schemes (e.g., frame-of-reference) for sorted or clustered integer columns.

### Format versioning

Bump `RainDbBatchBinaryCodec` format version when adding kinds. `RainDbCatalogDocument.formatVersion` may stay at 1 if batch files self-describe encoding.

### Interaction with mmap

Encoded columns may not mmap to a simple `FixedWidthColumnChunk` layout. Either:

- mmap compressed payload + decode buffer per scan, or
- mmap decompressed column files written at checkpoint time

## 15.23 Observability roadmap

### Compile path

Spans: `raindb.parse`, `raindb.bind`, `raindb.rewrite` (future). Tags: `sql.hash`, table names (careful with PII).

### Execute path

Per-operator activities: `VectorizedScan`, `HashAggregate`, `HashJoin`, `SortTopN`. Attributes: `table_id`, `row_count`, `dop`.

### Metrics

Counters: queries compiled, cache hits/misses, spill bytes written, batches scanned.

### Slow query log

Engine-level threshold in `RainDbEngineOptions` (future): log SQL + `plan.Explain()` when execution exceeds N ms.

## 15.24 Cross-cutting tracks

From the roadmap "all phases" table:

| Track | Intent |
|-------|--------|
| **Testing** | Golden SQL files, property tests for optimizer rules, spill and persistence integration tests |
| **Documentation** | Keep README, Implementation Status, RainDB-Internals, and this book aligned |
| **Security** | Fuzz SQL parser; bound work for hosted SQL scenarios |

## 15.25 Success metrics (product-level)

| Milestone | Indicator |
|-----------|-----------|
| Embedded analytics MVP | Prepared SQL + top-N + mmap scans on 100M rows within RAM budget |
| BI-shaped SQL | Expressions, `HAVING`, outer joins, `UNION ALL` with tests |
| Larger than RAM | Spilling hash agg/join passes correctness suite |
| Production embed | WAL recovery + single-writer / multi-reader documented |

## 15.26 Mapping original implementation phases

`docs/Implementation-Status.md` Phase 0–5 map to the lettered roadmap:

| Implementation phase | Roadmap |
|---------------------|---------|
| Phase 0–1 (foundations, read path) | Complete; Phase A extends |
| Phase 2 (agg, join, sort) | Complete in-memory; A + E2 mature |
| Phase 2b (directory persistence) | MVP complete; C + F harden |
| Phase 3 (planning & SQL) | B + D |
| Phase 4 (LINQ) | G1 |
| Phase 5 (durability & stats) | C4, E3, F |

## 15.27 How to extend this book

When a roadmap phase ships substantial code, add material without restructuring earlier chapters.

### New chapter vs appendix

| Change size | Book action |
|-------------|-------------|
| New operator (e.g., window) | New chapter with `order` after 16, or appendix under `content/appendix/` |
| Incremental improvement (top-N heap) | Expand existing chapter (e.g., 12) |
| New storage layer (WAL) | New chapter after persistence / mmap |
| Stub → real feature | Remove **(planned)** markers; add worked example |

MDWeb navigation uses front matter `order` field. Keep `order: 0` on `index.md`.

### Chapter template

Each new chapter should include:

1. Front matter (`title`, `order`)
2. Part I problem statement (theory) + Part II RainDB implementation
3. Architecture diagram or flow
4. RainDB types and file paths (relative to repo root)
5. Code walkthrough with real snippets
6. Interaction with prior chapters
7. Tests to read in `src/RainDB.Tests/`
8. **(planned)** section only while feature is partial

### Suggested future chapters

| Topic | Roadmap | Builds on |
|-------|---------|-----------|
| Spill and external sort | E2 | Ch. 10, `ISpillWriter` |
| mmap storage integration | C1 | Ch. 14, Ch. 16 |
| WAL and MVCC | F | Ch. 14 |
| LINQ provider | G1 | Ch. 13 |
| Cost-based optimizer | B2+ | Ch. 7, 13 |
| Window functions | E1 | Ch. 12 |
| Column encodings | C3 | Ch. 4, 14 |
| Observability | F3 | Ch. 7 |

### Keeping index.md current

Update the chapter table in `content/index.md` when adding chapters. Note the theory-first structure when a major expansion lands.

### Code citation convention

Cite paths relative to RainDB repository root:

```text
src/RainDB.Core/IO/ColumnarFixedWidthMmapReader.cs
```

Not absolute machine paths. Match existing chapters 1–12 style.

## 15.28 Contributing a roadmap feature — checklist

1. Read `docs/Development-Roadmap.md` exit criteria for the phase
2. Implement with tests first or in parallel
3. Update `docs/Implementation-Status.md`
4. Add Programming Guide section if user-visible
5. Extend this book (chapter or section)
6. If on-disk format changes, bump version constants and document migration

## 15.29 What not to do

- Do not add SQL syntax in Phase D without execution headroom from Phase A
- Do not implement real spill without defining merge correctness tests
- Do not fork physical plans for LINQ — use logical IR
- Do not break batch format version without decode fallback or export tool

## 15.30 Summary

**Part I** surveyed production OLAP requirements: cost-based optimization, statistics and histograms, spill algorithms, observability, format versioning, scheduling, and adaptive execution concepts. **Part II** mapped those ideas onto RainDB's phased roadmap — `ISpillWriter` and `SpillPartialEntryThreshold` wire telemetry today; `DefaultLinqCompiler` marks the LINQ insertion point; `MemoryTable.SchemaVersion` awaits a plan cache; Phases C and G add mmap, encodings, WAL, and host integration. When features land, extend this book using the chapter map in §15.27 — the narrative from OLAP fundamentals through SQL compilation and persistence remains the spine; new chapters attach at the leaves.
