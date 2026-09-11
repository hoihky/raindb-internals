---
title: Building an OLAP Database Engine from Scratch
order: 0
---

# Building an OLAP Database Engine from Scratch

**Using RainDB as a working example**

This book teaches you how to design and implement an embedded columnar OLAP database engine step by step. RainDB is an experimental .NET engine that stores data in columnar batches, executes queries with vectorized operators, and compiles a strict SQL subset into physical plans. Every chapter walks through one layer of the stack.

## Theory-first structure

Each chapter follows the same two-part layout:

| Part | Purpose |
|------|---------|
| **Part I — Concepts and Theory** | General database-engine ideas — relational algebra, storage layouts, compiler stages, OS I/O, and patterns from production OLAP systems — without tying the explanation to RainDB source code |
| **Part II — RainDB Implementation** | RainDB's concrete realization of those ideas: types, file paths, on-disk layouts, and annotated walkthroughs of the current codebase |

Start with Part I to learn the problem and the usual industry answers; continue to Part II to see how RainDB applies them. Part I stays language-neutral (pseudocode and diagrams where helpful). Part II cites real C# with repository-relative paths such as `src/RainDB.Core/Columnar/ColumnarBatch.cs`.

## Who this book is for

You should be comfortable reading C# and basic data-structure code. Prior exposure to relational databases helps, but you do not need to have built a query engine before. By the end, you should understand:

- Why OLAP engines use columnar storage and batch-oriented execution
- How modules in RainDB depend on each other
- How batches, chunks, null bitmaps, and catalogs fit together
- How scan, filter, project, aggregate, join, and sort operators work internally
- How SQL text becomes a physical plan and runs on the engine
- How directory-backed persistence and mmap column files relate to in-memory storage
- What production OLAP engines add beyond an embedded prototype (cost-based optimization, spill, WAL, observability)

## How RainDB is organized

RainDB splits responsibilities across six projects:

| Project | Role |
|---------|------|
| **RainDB.Abstractions** | Contracts: catalog, columnar model, execution interfaces, logical IR |
| **RainDB.Core** | Storage: `MemoryTable`, column chunks, buffer pools, file persistence, mmap I/O |
| **RainDB.Query** | Execution: physical plans, vectorized engines, query results |
| **RainDB.Sql** | SQL lexer, parser, binders, compiler |
| **RainDB.Linq** | Expression-tree compiler (stub today; same physical IR target) |
| **RainDB** (Driver) | `RainDbEngine` composition root |

The dependency rule is simple: outer layers depend on inner abstractions, never the reverse.

## Chapter map

Each chapter corresponds to one implementation step you would take when building RainDB from an empty repository.

| Chapter | Topic | Part I focus |
|---------|-------|--------------|
| [1. OLAP Fundamentals](01-olap-fundamentals.md) | Column vs row storage, batches, vectorization | Star schema, ETL, Amdahl, volcano vs vectorized, column-store history |
| [2. Architecture and Modules](02-architecture-and-modules.md) | Solution layout, `RainDbEngine`, extension points | DBMS layering, logical vs physical plans, Cascades, composition root |
| [3. Columnar Storage Model](03-columnar-storage-model.md) | Schemas, types, batch invariants | Zone maps, cache hierarchy, append-only storage, NULL semantics |
| [4. Chunk Data Structures](04-chunk-data-structures.md) | Fixed-width and UTF-8 chunks, null bitmaps | Encodings (plain/dictionary/RLE), Arrow layout, immutability |
| [5. Catalog and Tables](05-catalog-and-tables.md) | `TableId`, `MemoryTable`, append validation | System catalog, table identity, schema as contract, append-only vs MVCC |
| [6. Buffer Management](06-buffer-management.md) | `HybridBufferPool`, aligned allocation | Memory hierarchy, pools, SIMD alignment, LOH, bandwidth |
| [7. Execution Pipeline](07-execution-pipeline.md) | Physical plans, `DefaultQueryExecutor` | Volcano/vectorized models, pull vs push, pipelining vs materialization |
| [8. Vectorized Scan, Filter, Project](08-vectorized-scan-filter-project.md) | Selection vectors, morsel parallelism | Relational σ and π, selectivity, predicate pushdown, gather/scatter |
| [9. Global Aggregates](09-global-aggregates.md) | Partial aggregation, SIMD sum | γ operator, associative aggregates, map-reduce, empty-set semantics |
| [10. Hash Aggregation](10-hash-aggregation.md) | Group keys, hash maps, merge | Hash vs sort grouping, spill concept, composite keys |
| [11. Join Execution](11-joins.md) | Hash join, sort-merge join | Join algebra, NLJ/HJ/SMJ, build vs probe, join ordering |
| [12. Sort and Limit](12-sort-and-limit.md) | Top-N, ORDER BY | External sort, Top-K heaps, stable sort semantics |
| [13. SQL Compilation](13-sql-compilation.md) | Lexer, parser, binders | Compiler phases, CFG, AST vs IR, prepared statements |
| [14. Persistence](14-persistence.md) | Catalog JSON, batch codec | WAL, checkpointing, crash recovery, columnar file formats |
| [15. Future Extensions](15-future-extensions.md) | Roadmap and stubs | Production OLAP requirements, CBO, statistics, observability |
| [16. mmap I/O](16-mmap-io.md) | `RNBFCOL1` column files | Virtual memory, page cache, zero-copy, mmap vs read |

## How to read this book

Read chapters in order for the full build narrative. Later chapters assume earlier concepts.

- **Part I in every chapter** explains the general theory you need before reading RainDB code.
- **Part II in every chapter** walks through RainDB's current implementation with source references.
- **Chapter 15** is best read after 13–16 when you want the full roadmap map, not just today's code.

RainDB is actively developed. Sections marked **(planned)** describe interfaces or stubs that exist today but are not fully implemented. Chapter 15 collects extension points; Chapter 16 documents mmap code that is implemented but not yet wired into default persistence hydration.

## A minimal end-to-end example

Before diving into internals, here is the public surface the engine exposes after all layers are wired:

```csharp
using RainDB;
using RainDB.Core.Columnar;
using RainDB.Core.Tables;
using RainDB.Schema;

var engine = RainDbEngine.CreateDefault();

var schema = new TableSchema([
    new ColumnDef("region", RainDbType.Utf8),
    new ColumnDef("amount", RainDbType.Float64),
]);
var table = new MemoryTable("sales", schema);
// ... append columnar batches ...
engine.Catalog.Register(table);

await using var rows = await engine.ExecuteSqlAsync(
    "SELECT region, amount FROM sales WHERE amount > 0.5");
```

For durable storage, replace `CreateDefault()` with `RainDbEngine.OpenPersistent("./my-data")` — see Chapter 14.

Everything that happens between `ExecuteSqlAsync` and the returned rows is what this book explains.
