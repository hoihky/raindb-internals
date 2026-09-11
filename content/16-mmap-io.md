---
title: "Chapter 16: mmap I/O"
order: 16
---

# Chapter 16: mmap I/O

Memory-mapped I/O is how analytical engines avoid copying entire column files into the heap before scanning them. This chapter begins with virtual memory, the `mmap` syscall, page caches, and zero-copy philosophy. Part II documents RainDB's `RNBFCOL1` column file format, `ColumnarFixedWidthMmapReader`, and integration plans with batch persistence.

---

# Part I — Concepts

## 16.1 Virtual memory and paging

Operating systems give each process a **virtual address space** — often 48 bits wide on x86-64 — far larger than installed RAM. The **MMU** walks **page tables** to translate virtual addresses into **physical frames**.

### Pages and page frames

Addressable memory is sliced into fixed **pages** (commonly 4 KB; huge-page modes use 2 MB or 1 GB). A **page frame** is the physical RAM backing one page.

```text
Virtual address  →  page table lookup  →  physical frame (or page fault)
```

### Why virtual memory matters for databases

- **Isolation** — one process cannot read another's mappings
- **Overcommit** — virtual size can exceed RAM; pages materialize on fault
- **Shared mappings** — multiple processes mapping one file share physical frames
- **mmap** — file bytes appear at virtual addresses without an explicit `read()` buffer per access

Engines use these properties to treat column files as lazily populated byte ranges.

## 16.2 mmap syscall semantics

**mmap** links a virtual range to a file region (or anonymous memory). POSIX supplies `mmap`; Windows offers `MapViewOfFile` with similar semantics.

### Basic operation

```c
void *addr = mmap(NULL, length, PROT_READ, MAP_SHARED, fd, offset);
```

- **`fd`** — open file descriptor
- **`offset`** — byte position in the file (typically page-aligned)
- **`length`** — span to map
- **`PROT_READ` / `PROT_WRITE`** — permitted access
- **`MAP_SHARED`** — writable changes may propagate to other mappers and eventually to the file

The call returns a virtual base pointer. Touching `addr[i]` on a cold page raises a **page fault**; the kernel loads the page from disk into the cache and wires the frame.

### munmap

`munmap(addr, length)` destroys the mapping. Further accesses fault or crash. Engines must align unmap with query lifetimes.

### madvise and msync

- **`madvise`** — performance hints (`MADV_SEQUENTIAL` for scans, `MADV_DONTNEED` to drop cached pages)
- **`msync`** — flush dirty mapped pages (writable maps only)

RainDB maps column files read-only for scans — no `msync` on the query hot path.

## 16.3 Page cache and buffer cache

The kernel **page cache** retains recently touched file pages in RAM. Both `read()` and mmap faults usually hit cached pages on repeat access.

### Unified cache

On Linux, mmap and buffered `read()` share backing store for the same file offsets — warming one path benefits the other.

### Buffer pool vs OS page cache

Classic servers maintain an application **buffer pool** (InnoDB, PostgreSQL `shared_buffers`) when they need explicit pin/unpin policies, direct I/O to avoid double caching, or non-default page sizes.

Many embedded OLAP engines lean on the **OS cache via mmap** for cold files and reserve application pools for query intermediates — RainDB's `HybridBufferPool` backs operator output, not file pages.

### Eviction

Under memory pressure the kernel evicts cold cache pages. Valid mmap mappings remain; the next touch refaults from disk. Read-only analytics must accept **non-deterministic** cache residency.

## 16.4 Demand paging

**Demand paging** defers physical reads until first access.

```text
mmap(1 TB file)  →  virtual mapping created instantly (no disk I/O)
scan first row   →  fault on page 0  →  kernel reads 4 KB from disk
scan row 10000   →  fault on another page  →  another 4 KB read
```

Sequential column scans benefit from kernel **readahead** prefetching the next pages.

### Implications for OLAP

- **Mapping is cheap** — virtual size does not equal immediate I/O
- **Cold scans pay fault latency** — first pass touches many pages
- **Warm repeats are fast** — data already resident in cache
- **Random probes fault widely** — poor spatial locality hurts

## 16.5 Translation lookaside buffer (TLB)

The **TLB** caches recent virtual→physical translations. Every load/store needs a translation; TLB misses trigger a page-table walk.

### TLB pressure

Scanning very large mapped regions can **thrash the TLB** when the working set spans many small pages. Mitigations include:

- **Huge pages** — one TLB entry covers more bytes
- **Sequential access patterns** — friendlier to hardware prefetch
- **Fewer distinct mappings** — reduce mapping churn

Effects often show up between hundreds of megabytes and a few gigabytes — measure on deployment hardware.

## 16.6 Zero-copy I/O philosophy

**Zero-copy** here means application code consumes file bytes without an extra userspace copy between kernel buffers and algorithm input.

### read() path (copy)

```text
disk → kernel page cache → copy to userspace buffer → application reads buffer
```

Allocating a managed `byte[]` and copying again adds a second userspace pass.

### mmap path (zero-copy in userspace)

```text
disk → kernel page cache → mapped virtual pages → application reads via pointer/span
```

No userspace `memcpy` on initial load; SIMD kernels can read mapped spans directly.

### sendfile and other zero-copy APIs

Networking stacks expose `sendfile` and friends to pipe file pages into sockets without bouncing through userspace. HTTP exporters of Arrow or Parquet benefit; RainDB's mmap work targets local scan, not wire protocols.

### When zero-copy wins

- Large fixed-width columns read sequentially
- Datasets bigger than RAM with a hot subset in cache
- Multiple readers sharing the same on-disk files

### When zero-copy loses

- Tiny files where setup dominates
- Workloads that rewrite every value in userspace
- Environments with weak or broken mmap semantics — validate in target containers

## 16.7 mmap vs read() tradeoffs

| Factor | mmap | read() / pread |
|--------|------|-----------------|
| Initial setup | Create mapping; pages load on fault | Allocate buffer; syscall reads now |
| Memory accounting | Virtual size immediate; physical RAM grows with faults | Heap buffer size known up front |
| Random access | Fault per touched page | Explicit seek + read |
| Portability | Solid on major desktop OSes; edge cases differ | Ubiquitous |
| Error handling | Truncated files may SIGBUS mid-access on some OSes | Syscall returns error codes |
| Lifetime | Mapping must outlive consumers | Buffer ownership is independent |
| GC interaction | Custom `MemoryManager<byte>` or unsafe pointers in .NET | Plain `byte[]` |

### .NET specifics

`MemoryMappedFile` and `MemoryMappedViewAccessor` wrap the OS APIs. `SafeMemoryMappedViewHandle.AcquirePointer` yields native pointers for span-based consumption. `MemoryMarshal.TryGetArray` typically fails on mapped memory — by design.

## 16.8 Column file layout design principles

mmap-friendly column files share several layout conventions.

### 1. Fixed header size

Place magic, version, row count, and type at known offsets; validate before handing out pointers.

### 2. Self-describing payload lengths

The header records `nullBytes` and `valuesBytes`; readers confirm `fileSize >= header + nullBytes + valuesBytes`.

### 3. Alignment (optional)

Pad value arrays to 8- or 64-byte boundaries for SIMD. RainDB `RNBFCOL1` v1 packs tightly without alignment padding.

### 4. Immutable segments

Sealed append-only files let readers map while writers produce new segments elsewhere.

### 5. Separate metadata from payload

Table and column names live in the catalog, keeping on-disk payloads small and reload-friendly.

### 6. Versioned magic

Distinct magic strings (`RNBFCOL1`) identify families; a `uint32` version field reserves space for future compression flags.

### 7. No pointers in file

Store offsets from file base — files must remain position-independent.

## 16.9 Lifetime management of mapped memory

Correctness around mapping lifetime is the sharpest mmap footgun.

### Rules

1. **Never unmap while spans are live** — otherwise use-after-free
2. **Avoid deleting or replacing files under an active map** — platform-dependent failure modes
3. **Dispose in order** — release acquired pointer, then accessor, then `MemoryMappedFile`
4. **Pin through async queries** — keep `ColumnarFixedWidthMmapReader` alive until `IQueryResult` disposal

### Reference counting pattern

```text
Table holds List<MmapHandle>
  each handle: reader + ref count
  query acquires handle, releases on dispose
  eviction unmaps only when ref count == 0
```

RainDB Phase C2 will add formal pin/unpin in the buffer manager. Until then, tests must scope readers manually.

### GC and finalizers

Do not depend on finalizers for prompt unmap — call `Dispose` explicitly. Long-lived maps of huge files inflate virtual size statistics even when pages are cold.

---

# Part II — RainDB

RainDB's default persistence path (Chapter 14) reads entire batch files into managed `byte[]` arrays during hydration. That is correct and simple, but it copies every fixed-width value through the heap before the scan engine touches it. For large analytic tables where numeric columns dominate, **memory-mapped column files** offer a zero-copy alternative: the operating system maps file pages into the process address space, and scan kernels read values directly from mapped spans.

## 16.10 Scope and limitations

The mmap stack today supports **only fixed-width** `IColumnChunk` columns:

- `Boolean`, `Int32`, `Int64`, `Float64` (per `ColumnTypeSizes.IsFixedWidth`)
- **Not** `Utf8` or variable-length types in this file format

Magic string: `RNBFCOL1` (RainDB Fixed COLumn, version 1).

UTF-8 columns remain in `RainDbBatchBinaryCodec` batch files until a variable-length mmap layout is designed (Phase C, likely separate from fixed-width column files).

## 16.11 Why mmap for OLAP?

| Approach | Pros | Cons |
|----------|------|------|
| `File.ReadAllBytes` + `FixedWidthColumnChunk` | Simple; full ownership of bytes; works everywhere | 2× memory pressure during hydrate (file cache + managed copy); copy cost on load |
| mmap + `MappedFixedWidthColumnChunk` | Zero-copy scan; OS page cache shared across processes | Lifetime tied to `MemoryMappedFile`; unsafe pointer management; fixed layout only |
| In-memory append-only batches | Fastest writes; no IO during query | Lost on process exit without persistence |

RainDB's Phase C1 goal: **wire mmap chunks into table storage** so file-backed tables query without hydrating full byte arrays. The types in this chapter are the building blocks; `LoadBatchesIntoTable` still uses `ReadAllBytes` until integration lands.

## 16.12 ColumnarFixedWidthFileFormat — on-disk layout

`ColumnarFixedWidthFileFormat` in `src/RainDB.Core/IO/ColumnarFixedWidthFileFormat.cs` defines a single-column file.

```csharp
public static class ColumnarFixedWidthFileFormat
{
    public const int HeaderSizeBytes = 32;
    public static ReadOnlySpan<byte> Magic => "RNBFCOL1"u8;
}
```

### 16.12.1 File structure overview

```text
┌────────────────────────────────────────┐
│  Header (32 bytes)                     │
├────────────────────────────────────────┤
│  Null bitmap (optional, nullBytes)     │
├────────────────────────────────────────┤
│  Values (rowCount * typeWidth bytes)   │
└────────────────────────────────────────┘
```

Total file size:

```text
total = 32 + nullBytes + valuesBytes
nullBytes = hasNulls ? NullBitmapBytes(rowCount) : 0
valuesBytes = rowCount * FixedWidthBytes(physicalType)
```

### 16.12.2 Header — byte by byte

All multi-byte integers are **little-endian**.

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | `magic` | ASCII `R N B F C O L 1` (`0x52 0x4E 0x42 0x46 0x43 0x4F 0x4C 0x31`) |
| 8 | 4 | `version` | `uint32` format version; must be `1` |
| 12 | 4 | `rowCount` | `int32` number of rows; must be ≥ 0 |
| 16 | 1 | `physicalType` | `byte` cast of `RainDbType` enum |
| 17 | 1 | `flags` | Bit 0: `hasNulls` (1 = null bitmap present) |
| 18 | 2 | `_reserved` | Not written explicitly; remain zero in `WriteFile` |
| 20 | 4 | `nullBytes` | `int32` length of null bitmap section (0 if no nulls) |
| 24 | 4 | `valuesBytes` | `int32` length of values section |
| 28 | 4 | `_reserved2` | Implicitly zero (header array default) |

The writer uses a 32-byte `header` array and fills known fields:

```csharp
Magic.CopyTo(header);
BinaryPrimitives.WriteUInt32LittleEndian(header.AsSpan(8), 1);
BinaryPrimitives.WriteInt32LittleEndian(header.AsSpan(12), rowCount);
header[16] = (byte)chunk.PhysicalType;
header[17] = (byte)(chunk.HasNulls ? 1 : 0);
BinaryPrimitives.WriteInt32LittleEndian(header.AsSpan(20), nullBytes);
BinaryPrimitives.WriteInt32LittleEndian(header.AsSpan(24), valuesBytes);
```

Bytes at offsets 18–19 and 28–31 are zero unless future versions assign them (e.g., checksum, compression codec id).

### 16.12.3 Payload layout

After the header:

```text
payloadStart = 32

if hasNulls:
  [payloadStart .. payloadStart + nullBytes)     → null bitmap
  [payloadStart + nullBytes .. payloadStart + nullBytes + valuesBytes) → values

else:
  [payloadStart .. payloadStart + valuesBytes)   → values only
```

Null bitmap packing matches in-memory chunks: one bit per row, LSB-first within each byte (`ColumnTypeSizes.NullBitmapBytes`).

Values are tightly packed little-endian physical representation (same as `FixedWidthColumnChunk.Values`).

### 16.12.4 WriteFile implementation

```csharp
public static void WriteFile(string path, IColumnChunk chunk)
{
    if (!ColumnTypeSizes.IsFixedWidth(chunk.PhysicalType))
        throw new ArgumentException("Only fixed-width columns can be written with this format.", nameof(chunk));

    var rowCount = chunk.RowCount;
    var w = ColumnTypeSizes.FixedWidthBytes(chunk.PhysicalType);
    var nullBytes = chunk.HasNulls ? ColumnTypeSizes.NullBitmapBytes(rowCount) : 0;
    var valuesBytes = rowCount * w;
    // ... build header ...
    using var fs = new FileStream(path, FileMode.Create, FileAccess.Write, FileShare.None);
    fs.Write(header);
    if (nullBytes > 0)
        fs.Write(chunk.NullBitmap.Span[..nullBytes]);
    fs.Write(chunk.Values.Span[..valuesBytes]);
}
```

**Preconditions enforced at write time:**

- Chunk is fixed-width
- `chunk.Values` span is at least `valuesBytes`
- If `hasNulls`, null bitmap span is at least `nullBytes`

**Not embedded in file:** column name, table id, batch id — the path or catalog must carry metadata. Column files are dumb payloads.

### 16.12.5 Example hex dump (conceptual)

`Float64` column, 2 rows, no nulls, values `1.0` and `2.0`:

```text
Offset  Hex (abbreviated)     Meaning
------  -----------------     -------
0x00    52 4E 42 46 43 4F 4C 31  Magic RNBFCOL1
0x08    01 00 00 00             version = 1
0x0C    02 00 00 00             rowCount = 2
0x10    ??                      RainDbType.Float64 byte
0x11    00                      flags: no nulls
0x14    00 00 00 00             nullBytes = 0
0x18    10 00 00 00             valuesBytes = 16 (2 * 8)
0x20    00 00 00 00 00 00 F0 3F  1.0 IEEE754 LE
0x28    00 00 00 00 00 00 00 40  2.0 IEEE754 LE
```

## 16.13 ColumnarFixedWidthMmapReader.Open

`ColumnarFixedWidthMmapReader` maps an existing file read-only and exposes `Chunk` as `IColumnChunk`.

### 16.13.1 Open sequence

```csharp
public static ColumnarFixedWidthMmapReader Open(string path)
{
    var mmf = MemoryMappedFile.CreateFromFile(path, FileMode.Open, null, 0, MemoryMappedFileAccess.Read);
    var accessor = mmf.CreateViewAccessor(0, 0, MemoryMappedFileAccess.Read);
    // validate header + slice payload
    return new ColumnarFixedWidthMmapReader(mmf, accessor, manager, chunk);
}
```

`CreateViewAccessor(0, 0, Read)` maps the **entire file** (0 size means to end of file).

### 16.13.2 Validation steps (mirrors WriteFile)

```csharp
if (capacity > int.MaxValue)
    throw new NotSupportedException("Mapped column file exceeds int.MaxValue bytes.");

if (!span.StartsWith(ColumnarFixedWidthFileFormat.Magic))
    throw new InvalidDataException("Unexpected column file magic.");

var version = BinaryPrimitives.ReadUInt32LittleEndian(span.Slice(8, 4));
if (version != 1)
    throw new InvalidDataException($"Unsupported column file version {version}.");

var rowCount = BinaryPrimitives.ReadInt32LittleEndian(span.Slice(12, 4));
if (rowCount < 0)
    throw new InvalidDataException("Invalid row count.");

var type = (RainDbType)span[16];
if (!ColumnTypeSizes.IsFixedWidth(type))
    throw new InvalidDataException("Mapped column type must be fixed-width.");

var hasNulls = (span[17] & 1) != 0;
var nullBytes = BinaryPrimitives.ReadInt32LittleEndian(span.Slice(20, 4));
var valuesBytes = BinaryPrimitives.ReadInt32LittleEndian(span.Slice(24, 4));
var w = ColumnTypeSizes.FixedWidthBytes(type);

if (valuesBytes != checked(rowCount * w))
    throw new InvalidDataException("Values byte length does not match row count and type width.");

var expectedNull = hasNulls ? ColumnTypeSizes.NullBitmapBytes(rowCount) : 0;
if (nullBytes != expectedNull)
    throw new InvalidDataException("Null bitmap size does not match header flags / row count.");

var total = ColumnarFixedWidthFileFormat.HeaderSizeBytes + nullBytes + valuesBytes;
if (span.Length < total)
    throw new InvalidDataException("Column file truncated.");
```

Strict validation prevents silent misreads if files are hand-edited or truncated.

### 16.13.3 Slicing mapped memory

```csharp
var payloadStart = ColumnarFixedWidthFileFormat.HeaderSizeBytes;
ReadOnlyMemory<byte> nullMem;
ReadOnlyMemory<byte> valuesMem;
if (nullBytes == 0)
{
    nullMem = ReadOnlyMemory<byte>.Empty;
    valuesMem = bytes.Slice(payloadStart, valuesBytes);
}
else
{
    nullMem = bytes.Slice(payloadStart, nullBytes);
    valuesMem = bytes.Slice(payloadStart + nullBytes, valuesBytes);
}

var chunk = new MappedFixedWidthColumnChunk(type, rowCount, hasNulls, nullMem, valuesMem);
```

`bytes` is `ReadOnlyMemory<byte>` from `MmapBytesMemoryManager` covering the **whole file**. Slices are offsets into the map — no allocation for value data.

### 16.13.4 Dispose lifecycle

```csharp
public void Dispose()
{
    _manager.ReleasePointer();
    _accessor.Dispose();
    _mmf.Dispose();
}
```

Callers must dispose the reader (or wrap in `using`) before deleting or replacing the underlying file on disk. Undefined behavior if the file shrinks while mapped.

## 16.14 MmapBytesMemoryManager — unsafe pointer bridge

.NET's `ReadOnlyMemory<byte>` over mmap requires a custom `MemoryManager<byte>`:

```csharp
private sealed unsafe class MmapBytesMemoryManager : MemoryManager<byte>
{
    private readonly MemoryMappedViewAccessor _accessor;
    private byte* _pointer;
    private readonly int _length;

    public MmapBytesMemoryManager(MemoryMappedViewAccessor accessor, int length)
    {
        _accessor = accessor;
        _length = length;
        _pointer = null;
        accessor.SafeMemoryMappedViewHandle.AcquirePointer(ref _pointer);
        if (_pointer == null)
            throw new InvalidOperationException("Failed to acquire memory-mapped pointer.");
    }

    public override Span<byte> GetSpan() => new(_pointer, _length);
    public override MemoryHandle Pin(int elementIndex = 0) => new(_pointer + elementIndex);
    public override void Unpin() { }

    protected override bool TryGetArray(out ArraySegment<byte> segment)
    {
        segment = default;
        return false;  // cannot expose as managed array
    }

    public void ReleasePointer()
    {
        if (_pointer != null)
        {
            _accessor.SafeMemoryMappedViewHandle.ReleasePointer();
            _pointer = null;
        }
    }

    protected override void Dispose(bool disposing) => ReleasePointer();
}
```

### Why unsafe?

`MemoryMappedViewAccessor` exposes a native pointer via `SafeMemoryMappedViewHandle.AcquirePointer`. The manager wraps that pointer as a `Span<byte>` for interop with existing kernels that expect `ReadOnlyMemory<byte>` on chunks.

### Pin/Unpin

`Pin` returns a pointer offset for callers using `fixed` or interop. `Unpin` is empty because the mapping stays valid until dispose — there is no movable GC object behind the pointer.

### TryGetArray returns false

Forces consumers down the span path. Code that assumes `MemoryMarshal.TryGetArray` will fail — by design, so accidental heap copies are not silently introduced.

## 16.15 MappedFixedWidthColumnChunk — zero-copy IColumnChunk

```csharp
private sealed class MappedFixedWidthColumnChunk : IColumnChunk
{
    public MappedFixedWidthColumnChunk(
        RainDbType type,
        int rowCount,
        bool hasNulls,
        ReadOnlyMemory<byte> nullBitmap,
        ReadOnlyMemory<byte> values)
    {
        PhysicalType = type;
        RowCount = rowCount;
        HasNulls = hasNulls;
        NullBitmap = nullBitmap;
        Values = values;
    }

    public RainDbType PhysicalType { get; }
    public int RowCount { get; }
    public bool HasNulls { get; }
    public ReadOnlyMemory<byte> NullBitmap { get; }
    public ReadOnlyMemory<byte> Values { get; }
}
```

Implements the same `IColumnChunk` surface as `FixedWidthColumnChunk`. Scan and filter code paths that read via `column.Values.Span` and `SelectionEvaluator.IsNull(nullBitmap, row, hasNulls)` work unchanged.

### Zero-copy scan path

```text
SQL query
  → VectorizedScanPhysicalPlan
  → VectorizedScanEngine.ProcessOneBatch
  → batch.Columns[i] is MappedFixedWidthColumnChunk
  → FixedWidthSelectionKernels read Values.Span (mapped)
  → ProjectGather copies only selected rows to pooled output (if projection narrows rows)
```

**Full column scan without filter** may still avoid copying values if the engine passes mapped spans through. **Projection with selection** copies matching rows into pooled chunks — that is expected; mmap saves the *hydration* copy, not necessarily all query-time gathers.

### Comparison to FixedWidthColumnChunk

| Property | `FixedWidthColumnChunk` | `MappedFixedWidthColumnChunk` |
|----------|-------------------------|-------------------------------|
| Backing store | `byte[]` or pooled memory | mmap file |
| Lifetime | GC / pool return | `ColumnarFixedWidthMmapReader` dispose |
| Writable | Can construct for append | Read-only |
| Scan API | `IColumnChunk` | `IColumnChunk` |

## 16.16 End-to-end usage example

Write a column file from an in-memory chunk, then mmap it back:

```csharp
using RainDB.Core.Columnar;
using RainDB.Core.IO;
using RainDB.Schema;

// Build in-memory chunk
var values = new byte[16]; // 2 x Float64
BitConverter.TryWriteBytes(values.AsSpan(0), 1.0);
BitConverter.TryWriteBytes(values.AsSpan(8), 2.0);
var chunk = new FixedWidthColumnChunk(RainDbType.Float64, 2, values, ReadOnlyMemory<byte>.Empty, hasNulls: false);

var path = Path.Combine(Path.GetTempPath(), "amount.col");
ColumnarFixedWidthFileFormat.WriteFile(path, chunk);

using var reader = ColumnarFixedWidthMmapReader.Open(path);
var mapped = reader.Chunk;
Assert.Equal(2, mapped.RowCount);
Assert.Equal(RainDbType.Float64, mapped.PhysicalType);
// mapped.Values.Span reads directly from file mapping
```

Use in a `ColumnarBatch` for engine tests (manual composition):

```csharp
var batch = new ColumnarBatch(mapped.RowCount, new IColumnChunk[] { mapped });
// Pass to operators that accept IColumnarBatch — ensure reader stays alive for batch lifetime
```

**Lifetime rule:** Keep `ColumnarFixedWidthMmapReader` alive while any scan references `mapped`.

## 16.17 Integration roadmap with persistence

Current persistence (Chapter 14) uses monolithic `.batch` files. mmap integration options:

### Option A — sidecar column files per batch

```text
tables/{tableId}/
  000000.batch          # manifest or UTF-8 columns only
  000000.col0           # mmap Float64 amount
  000000.col1           # mmap Int32 quantity
```

Hydration:

1. Read small `.batch` manifest listing column types and sidecar paths
2. mmap each fixed-width sidecar → `MappedFixedWidthColumnChunk`
3. Load UTF-8 columns from batch body as today

### Option B — column-major table directory

```text
tables/{tableId}/
  meta.json
  columns/
    amount.rnbfcol
    quantity.rnbfcol
    region.batch-utf8
```

Append becomes column-append or new segment files per column. Harder for atomic multi-column append; better for read-heavy workloads.

### Option C — hybrid hydrate

On `LoadBatchesIntoTable`:

```csharp
// (future sketch)
if (File.Exists(fixedWidthSidecar))
    chunks[i] = ColumnarFixedWidthMmapReader.Open(sidecar).Chunk;
else
    chunks[i] = DecodeColumnFromBatch(bytes, i);
```

Minimal change surface: only replace `ReadAllBytes` + `DecodeBatch` for eligible columns.

### Catalog changes

`catalog.json` might gain optional `storageMode: "mmap" | "batch"` per table. Default remains batch for compatibility.

### Buffer manager (Phase C2)

Long-lived maps need pin/unpin when memory cap exceeded:

```csharp
// (planned)
interface IMappedColumnHandle
{
    IColumnChunk Chunk { get; }
    void Pin();
    void Unpin();
}
```

`ColumnarFixedWidthMmapReader` becomes internal to handle implementation.

## 16.18 When to use mmap vs in-memory bytes

### Prefer mmap when

- Table (or column) size exceeds comfortable RAM fraction
- Multiple processes read same data (OS shares page cache)
- Workload is scan-heavy, append-rare
- Column is fixed-width numeric (fits `RNBFCOL1` today)
- Latency of first touch can be lazy (page faults on scan)

### Prefer in-memory bytes when

- Dataset fits easily in RAM and you want predictable performance
- High append rate (mmap files require rewrite or segment append policy)
- UTF-8 or mixed-type batches without sidecar decomposition
- Need writable chunks for in-query transforms
- Running on platforms with weak mmap semantics (some containers — test first)

### Hybrid strategy (recommended product direction)

| Tier | Storage |
|------|---------|
| Hot recent batches | In-memory `MemoryTable` segments |
| Cold sealed batches | mmap column files |
| UTF-8 dimensions | Batch codec or future UTF-8 mmap layout |

RainDB has not implemented tiering yet; Phase C1 starts with "hydrate to mmap instead of copy" for whole fixed-width columns.

## 16.19 Interaction with RainDbBatchBinaryCodec

| Format | Magic | Granularity | UTF-8 | mmap-friendly |
|--------|-------|-------------|-------|---------------|
| Batch v1 | `RNBATCH1` | Whole batch, all columns | Yes | Poor (decode copies) |
| Column v1 | `RNBFCOL1` | Single fixed-width column | No | Yes |

Converting batch → column files (export tool):

```csharp
// (illustrative migration utility)
void ExportFixedColumns(IColumnarBatch batch, string dir)
{
    for (var i = 0; i < batch.Columns.Count; i++)
    {
        var c = batch.Columns[i];
        if (ColumnTypeSizes.IsFixedWidth(c.PhysicalType))
            ColumnarFixedWidthFileFormat.WriteFile(Path.Combine(dir, $"col{i}.rnbfcol"), c);
    }
}
```

Round-trip for fixed-width: `WriteFile` → `Open` → compare spans.

## 16.20 Error handling and security

| Condition | Result |
|-----------|--------|
| Truncated file | `InvalidDataException` |
| Wrong magic | `InvalidDataException` |
| `valuesBytes` mismatch | `InvalidDataException` |
| Non-fixed-width type byte | `InvalidDataException` |
| File > 2GB on 32-bit | `NotSupportedException` (int length) |

For hostile files, validation before pointer exposure prevents out-of-bounds slice on header fields. Do not mmap untrusted paths without size caps at open time in hosted scenarios.

## 16.21 Testing recommendations

When integration lands, add tests for:

1. WriteFile → Open → row count and value equality
2. Null bitmap round-trip with `hasNulls = true`
3. Dispose reader → ensure no use-after-free if scan held spans (document as unsupported)
4. Persistence reopen with mmap chunks + SQL `SELECT SUM(col)`
5. Mixed batch: one mmap column + one UTF-8 in-memory column in same batch

Property: `valuesBytes == rowCount * FixedWidthBytes(type)` for all valid files.

## 16.22 Performance notes

- **First scan** after open may page-fault; subsequent scans hit OS cache
- **SIMD kernels** work on mapped spans if pinned spans are contiguous (they are)
- **No extra copy** on hydrate vs `ReadAllBytes` + new `FixedWidthColumnChunk`
- **Projection** still allocates pooled output for selected rows — mmap does not eliminate gather copies

Benchmark compare (manual, Phase A5):

```text
Hydrate 1M-row Float64 column:
  ReadAllBytes + DecodeBatch  vs  ColumnarFixedWidthMmapReader.Open
Scan SUM(amount) where amount > 0:
  in-memory chunk vs mapped chunk (should be near parity after warm-up)
```

## 16.23 Future header versions

Reserved header bytes (18–19, 28–31) enable:

- `version = 2` with compression codec id in reserved bytes
- Checksum at offset 28
- Row count as `int64` for very large columns (would require header size bump)

Readers must reject unknown versions (as today for `version != 1`).

## 16.24 Summary

**Part I** covered virtual memory, mmap semantics, page cache interaction, demand paging, TLB effects, zero-copy philosophy, mmap vs `read()` tradeoffs, column file design principles, and mapped memory lifetime rules. **Part II** showed how RainDB implements these ideas: `ColumnarFixedWidthFileFormat` writes a 32-byte `RNBFCOL1` header; `ColumnarFixedWidthMmapReader.Open` validates and constructs `MappedFixedWidthColumnChunk` via unsafe `MmapBytesMemoryManager`; scan pipelines consume mapped chunks through the same `IColumnChunk` interface as heap-backed columns. UTF-8 and multi-column batches still use `RainDbBatchBinaryCodec` until Phase C wires sidecar column files and buffer management into `RainDbFileDatabase`.
