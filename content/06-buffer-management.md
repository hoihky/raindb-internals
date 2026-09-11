---
title: "Chapter 6: Buffer Management"
order: 6
---

# Chapter 6: Buffer Management

Vectorized engines chew through **short-lived allocations** at a furious rate: selection index arrays, projected column buffers, hash buckets, sort runs. Calling `new byte[n]` inside every inner loop shifts the bottleneck from computation to the garbage collector. Buffer management — deliberate policies for acquiring, aligning, reusing, and releasing memory — is therefore a first-class design problem in OLAP, not a polish pass at the end.

**Part I** walks the memory hierarchy, allocator families, SIMD alignment rules, .NET Large Object Heap behavior, and why bandwidth — not CPU — often caps scan throughput. **Part II** connects those ideas to RainDB's `HybridBufferPool`, `PooledAlignedMemoryOwner`, and the dual-pool pattern in `ProjectGather` and `VectorizedScanEngine`.

---

# Part I: Concepts and Theory

## 6.1 The memory hierarchy

Hardware presents storage as a **pyramid**: each tier is bigger and slower than the one above. Engines that treat all memory as equally fast leave performance on the table.

```
         ┌─────────────┐
         │  Registers  │  ~1 cycle, tens of values
         ├─────────────┤
         │ L1 cache    │  ~4 cycles, 32–64 KiB per core
         ├─────────────┤
         │ L2 cache    │  ~12 cycles, 256 KiB–1 MiB per core
         ├─────────────┤
         │ L3 cache    │  ~40 cycles, shared 8–64 MiB
         ├─────────────┤
         │ DRAM        │  ~100–300 cycles, GB–TB
         ├─────────────┤
         │ SSD / NVMe  │  ~10–100 µs, TB
         ├─────────────┤
         │ HDD         │  ~5–10 ms, TB
         └─────────────┘
```

### Registers and SIMD lanes

**Registers** hold the operands of individual instructions. SIMD widens them so one instruction touches many elements at once:

| ISA extension | Typical register width | Elements (32-bit int) |
|---------------|------------------------|----------------------|
| SSE / Vector128 | 128 bits | 4 |
| AVX2 / Vector256 | 256 bits | 8 |
| AVX-512 | 512 bits | 16 |

Columnar kernels load contiguous values into SIMD registers, compare or add across lanes, and write results. **Register pressure** caps how many columns you can process in one loop — a key reason engines batch rows into fixed-width **vectors** (often 1024–65536 rows).

### Cache lines

Caches fetch **cache lines** — typically **64 bytes** at a time. A `Float64` load that straddles a line boundary may pull two lines. Columnar layout helps: scanning 8-byte values sequentially tends to use each line once with high utilization.

**False sharing** happens when two cores write different fields that share a line; each write evicts the line for the other core. Padding hot counters to 128 bytes (two lines) isolates them — RainDB exposes `SimdAlignment.CacheLine128` for that pattern.

### DRAM

DRAM is the working set for in-memory OLAP. When data exceeds RAM, operators **spill** to disk (hash partitions, sort runs). RainDB's `ISpillWriter` port anticipates future hash-aggregate spill paths.

### Disk

Even on NVMe, random I/O is orders of magnitude slower than sequential DRAM bandwidth. Column pruning and batch sizing exist partly to keep hot columns in RAM and reads sequential on disk.

---

## 6.2 Why allocation matters in tight loops

Filter and project one batch of 65,536 rows across three `Float64` columns at 50% selectivity:

```
Per batch (naive allocation):
  selection int[65536]           ≈ 256 KiB
  3 × projected Float64 columns  ≈ 3 × 50% × 65536 × 8 ≈ 768 KiB
  Total ≈ 1 MiB allocated and discarded
```

Across 200 batches that is **~200 MiB** of garbage per query, mostly Gen0/Gen1 churn. The CPU spends cycles on:

1. Zeroing fresh arrays
2. Tracing references during GC
3. Promoting survivors across generations
4. Allocator metadata traffic that evicts useful cache lines

### Allocation inside inner loops

An **inner loop** that runs per row, per column, or per batch should avoid the general allocator unless there is no alternative:

```
// BAD: allocates every iteration
for each batch:
    for each column:
        output = new byte[rowCount * width]

// GOOD: rent once per batch/column, return after use
for each batch:
    buf = pool.rent(rowCount * width)
    try:
        ... compute ...
    finally:
        pool.return(buf)
```

The goal is **amortization**: pay allocation cost once per batch (or query), then reuse through a pool.

### Throughput versus latency

OLAP workloads optimize **throughput** (rows per second). Millisecond GC pauses may be tolerable for batch jobs but ruin interactive tail latency. Pooling cuts collection frequency and makes pauses more predictable.

---

## 6.3 Arena and bump allocators

An **arena** (region allocator, bump allocator) hands out memory from a pre-reserved block by advancing an offset:

```
Arena block: [████████████████████████████████]
              ^offset
              bump by N bytes → fast pointer add

reset(): offset = 0  →  entire arena reclaimed at once
```

### Characteristics

| Property | Arena behavior |
|----------|----------------|
| Individual free | Not supported (or rarely) |
| Deallocation | Reset entire arena or drop arena object |
| Speed | Near-constant time per allocation |
| Fragmentation | None within arena lifetime |
| Thread safety | Usually one arena per thread |

### Use cases in query engines

- **Expression evaluation** — scratch strings and structs for one query
- **Parser / planner** — AST nodes for one compile
- **Per-query scratch** — reset when the query finishes

### Pseudocode

```
class Arena {
    block: byte[]
    offset: int = 0

    allocate(n: int) -> Span<byte> {
        if offset + n > block.length: grow()
        result = block[offset .. offset+n]
        offset += n
        return result
    }

    reset() { offset = 0 }
}
```

Arenas pair naturally with **compile-once, run-many** and **one query, one arena** lifetimes. RainDB Phase 1 leans on **`ArrayPool`** instead of a custom arena — simpler integration with `IMemoryOwner<byte>` and chunk disposal — but arena-like semantics appear in nested `PooledBufferOwner` lifetimes tied to chunk `Dispose`.

---

## 6.4 Object pools versus array pools

Managed runtimes offer two dominant pooling styles:

### Object pool

Reuses **heap objects** (class instances):

```
pool.acquire() → Widget instance (possibly reused)
pool.release(widget) → reset state, return to pool
```

Best for expensive-to-build objects: `StringBuilder`, connection wrappers, protocol buffers with headers.

**Cost:** Object headers, reference-type GC tracking, and reset logic to clear sensitive state.

### Array pool

Reuses **primitive arrays** (`byte[]`, `int[]`) without allocating a wrapper per rent:

```
buf = ArrayPool<byte>.Shared.Rent(minimumLength)
// ... use buf ...
ArrayPool<byte>.Shared.Return(buf, clearArray: false)
```

.NET's `System.Buffers.ArrayPool<T>` targets high-throughput server code. Rented arrays may be **longer** than requested — track logical length separately.

| Aspect | Object pool | Array pool |
|--------|-------------|------------|
| Unit | Class instance | `T[]` |
| GC pressure | Lower than alloc-per-use | Minimal |
| Type safety | Strong | Raw arrays |
| Typical OLAP use | Result wrappers | Column bytes, selection indices |

RainDB routes column bytes through **`ArrayPool<byte>`** via `HybridBufferPool` and selection indices through **`ArrayPool<int>.Shared`** in `VectorizedScanEngine` — the byte pool abstraction does not map cleanly onto typed `int[]`.

---

## 6.5 SIMD alignment requirements (32/64 byte)

**Aligned SIMD loads** require the address to be a multiple of the vector width:

```
Address % 32 == 0   →  aligned Vector256 load (typical AVX2)
Address % 16 == 0   →  aligned Vector128 load
```

### Misaligned loads

x86-64 often tolerates unaligned SIMD with a latency penalty. ARM may fault or slow-path depending on the instruction. Portable engines align hot buffers defensively.

### Alignment versus array object header

In .NET, the **first element** of a `byte[]` is not guaranteed 32-byte aligned — object header and length prefix skew the address. The standard fix is sub-slicing inside a larger rent:

```
rented = pool.rent(needed + alignment - 1)
offset = align_up(address_of(rented[0]), 32)
aligned_slice = rented[offset .. offset+needed]
```

RainDB's `HybridBufferPool.RentAligned` implements exactly this pattern.

### 32 versus 64 byte

| Alignment | Rationale |
|-----------|-----------|
| 32 bytes | `Vector256<double>` / AVX2 |
| 64 bytes | AVX-512, some cache-line isolation strategies |

RainDB defaults to **32 bytes** (`SimdAlignment.Vector256`). Requests below 32 are rejected — under-alignment defeats the purpose.

---

## 6.6 Large Object Heap (LOH) in .NET GC

On 64-bit runtimes, the GC routes large arrays to a separate heap.

### LOH threshold

Objects **≥ ~85,000 bytes** land on the **Large Object Heap**:

- Collected less often than Gen0/Gen1
- Historically not compacted (fragmentation risk); modern .NET can compact LOH on demand
- Higher alloc and collection cost than small objects

### Implications for column buffers

```
65536 rows × 8 bytes (Float64) = 524,288 bytes  →  LOH
65536 rows × 4 bytes (Int32)   = 262,144 bytes  →  LOH
8192 rows × 8 bytes            =  65,536 bytes  →  below LOH (Gen2 candidate after survival)
```

Strict vector sizing (`64K–1M rows`) often pushes column buffers into LOH territory. **Pooling** does not avoid LOH placement but **reuses** those arrays across queries, eliminating repeated alloc/free cycles.

### `ArrayPool` and LOH

`ArrayPool<byte>.Shared` maintains buckets including large sizes. Returned arrays stay pooled until memory pressure evicts them. RainDB's `HybridBufferPool` comment notes intent to keep **typical** rents under LOH when possible — null bitmaps at 8 KiB for 64K rows sit well below the threshold.

### LOH mitigation strategies

1. **Smaller vector widths** — trade loop overhead for Gen1-sized buffers
2. **Pooling** — amortize LOH allocation cost
3. **Native memory / unmanaged** — `NativeMemory.AlignedAlloc` bypasses GC (some engines; RainDB stays managed in Phase 1)
4. **ArrayPool clear policy** — `Return(buf, clearArray: false)` on hot paths avoids zeroing megabytes

---

## 6.7 Memory bandwidth as the OLAP bottleneck

Once allocation noise is under control, **memory bandwidth** often caps scan throughput — the **memory wall**.

### Roofline intuition

Peak performance is bounded by:

```
min(peak_compute_ops, memory_bandwidth × operational_intensity)
```

Scan/filter/project on cold columns has **low operational intensity**: few arithmetic ops per byte loaded from DRAM.

### Column pruning effect

Reading 2 of 50 columns uses roughly 4% of the row-oriented bandwidth — columnar storage makes pruning a physical win, not just a logical one.

### SIMD helps compute, not bandwidth

SIMD cuts instruction count for compares and sums but still **loads every value byte** from memory (unless compressed encoding shrinks the payload). At saturation, cores stall waiting for cache/DRAM fills.

### Bandwidth-conscious design checklist

| Technique | Bandwidth impact |
|-----------|------------------|
| Columnar layout | Read only needed columns |
| Batch processing | Sequential access, prefetch friendly |
| Selection before project | Gather fewer rows into output |
| Compression | Fewer bytes from disk/DRAM (decode cost trade-off) |
| Pool reuse | Less allocator traffic, better cache residency |

RainDB's filter-then-project pipeline (`SelectionEvaluator` → `ProjectGather`) shrinks output materialization when predicates are selective.

---

## 6.8 Ownership and lifetime models (theory)

Buffer strategies must answer **who returns memory when**:

| Model | Rule |
|-------|------|
| Caller-owned | Rent and return in same function |
| Owner object | `IMemoryOwner<T>.Dispose()` returns buffer |
| Query-scoped | Dispose query result → all column chunks release |
| Arena-scoped | Reset arena at query end |

Mixing models invites use-after-free or pool leaks. RainDB ties **`PooledFixedWidthColumnChunk`** disposal to **`ColumnarMaterializedQueryResult.DisposeAsync`**.

---

## 6.9 Summary: Part I checklist

1. **Hierarchy** — registers/SIMD → cache lines → DRAM → disk; know where bottlenecks move.
2. **Tight loops** — no per-iteration `new`; rent/return or bump within batch scope.
3. **Arenas** — fast bulk reclaim; great for query-scoped scratch.
4. **Array pools** — preferred for column bytes and index arrays in .NET OLAP.
5. **Alignment** — sub-slice within padded pool rent for SIMD loads.
6. **LOH** — large vectors go LOH; pool to amortize.
7. **Bandwidth** — often limits scans after allocation is fixed; filter early, project fewer bytes.

---

# Transition to Part II: RainDB Implementation

RainDB tackles scan-path allocation with a two-tier model: **`IBufferPool`** for general byte rents and **`IAlignedBufferPool`** for SIMD-safe slices, both implemented by **`HybridBufferPool`**. **`ProjectGather`** rents aligned value buffers and byte null bitmaps per projected column. **`VectorizedScanEngine`** pulls selection vectors from **`ArrayPool<int>.Shared`** separately from the byte pool. Column chunks implement **`IDisposable`** so pooled memory returns when query results are disposed.

---

# Part II: RainDB Implementation

## 6.10 The allocation problem on scan hot paths

Consider `SELECT region, amount FROM sales WHERE amount > 100` over a table with 200 batches of 65,536 rows each. For each batch, `VectorizedScanEngine.ProcessOneBatch`:

1. Rents an `int[]` selection buffer (row indices passing the filter)
2. Calls `ProjectGather.Project`, which for each output column may rent aligned value bytes and optional null bitmap bytes
3. Returns the `int[]` to `ArrayPool<int>.Shared`
4. Returns byte buffers when `PooledFixedWidthColumnChunk` is disposed

Without pooling, step 2 alone might allocate `selectedCount × columnWidth × numColumns` bytes per batch. For three `Float64` columns and 50% selectivity, that is roughly 24 KiB per batch × 200 batches ≈ 4.8 MB of Gen0/Gen1 garbage per query, plus alignment padding waste.

RainDB's design goal: **keep hot-path allocations predictable** and **return buffers promptly** so `ArrayPool` can reuse them across queries in the same process.

---

## 6.11 IBufferPool: the general byte rent contract

```csharp
// src/RainDB.Abstractions/Memory/IBufferPool.cs
/// <summary>
/// Low-latency byte buffers for I/O and vector materialization (DIP: mock in tests).
/// Prefer lengths under the LOH threshold (~85KB on 64-bit CLR) when renting from ArrayPool{T}.
/// </summary>
public interface IBufferPool
{
    byte[] Rent(int minimumLength);
    void Return(byte[] buffer, bool clearArray = false);
}
```

**Dependency inversion:** Operators depend on `IBufferPool`, not `ArrayPool<byte>.Shared` directly. Tests can inject a counting or failing pool. `RainDbEngine` wires a single `HybridBufferPool` instance as both `BufferPool` and `AlignedBufferPool`.

**`clearArray`:** Forwarded to `ArrayPool.Return`. RainDB typically passes `false` on hot paths — projected data overwrites the rented region; clearing would waste CPU and bandwidth.

---

## 6.12 ArrayPoolBufferPool: thin adapter

When aligned sub-allocation is not needed, the adapter adds zero indirection beyond a field store:

```csharp
// src/RainDB.Core/Memory/ArrayPoolBufferPool.cs
public sealed class ArrayPoolBufferPool : IBufferPool
{
    private readonly ArrayPool<byte> _pool;

    public ArrayPoolBufferPool(ArrayPool<byte>? pool = null) =>
        _pool = pool ?? ArrayPool<byte>.Shared;

    public byte[] Rent(int minimumLength) => _pool.Rent(minimumLength);

    public void Return(byte[] buffer, bool clearArray = false) =>
        _pool.Return(buffer, clearArray);
}
```

`HybridBufferPool` implements `IBufferPool` with identical rent/return logic. The engine uses `HybridBufferPool` exclusively so one object satisfies both interfaces.

---

## 6.13 IAlignedBufferPool and SimdAlignment constants

```csharp
// src/RainDB.Abstractions/Memory/IAlignedBufferPool.cs
public interface IAlignedBufferPool
{
    /// <param name="minimumByteLength">Usable byte length (not including alignment padding).</param>
    /// <param name="alignment">Power of two, at least SimdAlignment.Vector256 (32) for AVX2 loads.</param>
    IMemoryOwner<byte> RentAligned(int minimumByteLength, int alignment = SimdAlignment.Vector256);
}
```

```csharp
// src/RainDB.Abstractions/Memory/SimdAlignment.cs
public static class SimdAlignment
{
    /// <summary>Vector256&lt;T&gt; requires 32-byte alignment for aligned loads.</summary>
    public const int Vector256 = 32;

    /// <summary>Two cache lines — useful for false-sharing avoidance on write-heavy buffers.</summary>
    public const int CacheLine128 = 128;
}
```

**Why 32 bytes?** `System.Runtime.Intrinsics.Vector128<int>` and future `Vector256` loads benefit from addresses divisible by the vector width. Misaligned loads work on x86-64 but can incur penalties; aligned `RentAligned` guarantees the **start** of the returned `Memory<byte>` slice is suitable for `Vector128.LoadUnsafe`.

**`IMemoryOwner<byte>`:** Buffers are returned through `Dispose()` on the owner, not manual `IBufferPool.Return`. This ties buffer lifetime to column chunk lifetime (`PooledFixedWidthColumnChunk`).

---

## 6.14 HybridBufferPool: full implementation

```csharp
// src/RainDB.Core/Memory/HybridBufferPool.cs
public sealed class HybridBufferPool : IBufferPool, IAlignedBufferPool
{
    private readonly ArrayPool<byte> _pool;

    public HybridBufferPool(ArrayPool<byte>? pool = null) =>
        _pool = pool ?? ArrayPool<byte>.Shared;

    public byte[] Rent(int minimumLength) => _pool.Rent(minimumLength);

    public void Return(byte[] buffer, bool clearArray = false) =>
        _pool.Return(buffer, clearArray);

    /// <summary>
    /// Rents a sub-slice of a pooled array aligned to alignment (default 32 for SimdAlignment.Vector256).
    /// Uses ArrayPool{T} so buffers stay under LOH for typical OLAP vector sizes (&lt;85KB).
    /// </summary>
    public IMemoryOwner<byte> RentAligned(int minimumByteLength, int alignment = SimdAlignment.Vector256)
    {
        if (minimumByteLength < 0)
            throw new ArgumentOutOfRangeException(nameof(minimumByteLength));
        if (alignment < SimdAlignment.Vector256 || (alignment & (alignment - 1)) != 0)
            throw new ArgumentException(
                $"Alignment must be a power of two >= {SimdAlignment.Vector256}.", nameof(alignment));

        var pad = alignment - 1;
        var rented = _pool.Rent(checked(minimumByteLength + pad));
        ref var r0 = ref rented[0];
        unsafe
        {
            var addr = (nuint)Unsafe.AsPointer(ref r0);
            var mis = addr % (nuint)alignment;
            var off = mis == 0 ? 0 : (int)((nuint)alignment - mis);
            if (off + minimumByteLength > rented.Length)
            {
                _pool.Return(rented);
                throw new InvalidOperationException(
                    "Rented buffer could not satisfy alignment; increase padding.");
            }

            return new PooledAlignedMemoryOwner(rented, off, minimumByteLength, _pool);
        }
    }
}
```

This is the complete production implementation — no hidden helpers. Understanding it requires walking through the alignment math in §6.15.

---

## 6.15 Alignment mathematics

Given:

- `minimumByteLength` — usable bytes the caller needs
- `alignment` — power of two (default 32)
- `pad = alignment - 1` — maximum skew before the first array element

**Step 1 — rent oversized array:**

```
rentedLength >= minimumByteLength + pad
```

`ArrayPool` may return an array longer than requested; the extra space is essential. Example: need 40 usable bytes at 32-byte alignment. Rent `40 + 31 = 71` bytes; pool might return 128.

**Step 2 — compute address of `rented[0]`:**

```csharp
var addr = (nuint)Unsafe.AsPointer(ref rented[0]);
```

Pinning is not required for managed arrays in modern .NET when using `Unsafe.AsPointer` on a `ref` to the first element — the GC tracks interior pointers for the duration of the unsafe block's logical use; the `IMemoryOwner` keeps the array alive.

**Step 3 — misalignment:**

```csharp
var mis = addr % (nuint)alignment;
var off = mis == 0 ? 0 : (int)((nuint)alignment - mis);
```

- If `addr` is already divisible by 32, `off = 0`.
- Otherwise, skip forward to the next aligned boundary within the array.

**Worked example:** Suppose `addr % 32 == 8`. Then `mis = 8`, `off = 32 - 8 = 24`. The aligned slice starts at `rented[24]`.

**Step 4 — feasibility check:**

```csharp
if (off + minimumByteLength > rented.Length)
```

If the pool returned exactly `minimumByteLength + pad` and `off` is large, the slice might not fit. The implementation returns the oversized array to the pool and throws — this should be rare because `ArrayPool` typically over-allocates to power-of-two bucket sizes.

**Step 5 — expose slice:**

`PooledAlignedMemoryOwner` stores `(rented, off, minimumByteLength)` and exposes `new Memory<byte>(rented, off, minimumByteLength)`.

**Validation:** Tests pin the span and assert `addr % 32 == 0`:

```csharp
// tests/RainDB.Tests/ColumnarAndCatalogTests.cs
using var owner = pool.RentAligned(40);
unsafe {
    fixed (byte* p = owner.Memory.Span) {
        Assert.Equal(0u, (nuint)p % 32);
    }
}
```

Alignments below 32 are rejected (`RentAligned(8, 16)` throws).

---

## 6.16 PooledAlignedMemoryOwner: full implementation

```csharp
// src/RainDB.Core/Memory/PooledAlignedMemoryOwner.cs
internal sealed class PooledAlignedMemoryOwner : IMemoryOwner<byte>
{
    private byte[]? _rented;
    private readonly int _start;
    private readonly int _length;
    private readonly ArrayPool<byte> _pool;

    public PooledAlignedMemoryOwner(byte[] rented, int start, int length, ArrayPool<byte> pool)
    {
        _rented = rented;
        _start = start;
        _length = length;
        _pool = pool;
    }

    public Memory<byte> Memory
    {
        get
        {
            var r = _rented;
            if (r is null)
                throw new ObjectDisposedException(nameof(PooledAlignedMemoryOwner));
            return new Memory<byte>(r, _start, _length);
        }
    }

    public void Dispose()
    {
        var r = Interlocked.Exchange(ref _rented, null);
        if (r is not null)
            _pool.Return(r);
    }
}
```

**Lifetime rules:**

- Only the **owner** returns the backing array — callers must not retain the `byte[]` after dispose.
- `Interlocked.Exchange` makes double-dispose safe (second dispose is a no-op).
- The **entire** rented array returns to the pool, including alignment prefix bytes — those bytes are "wasted" within the pooled object but reused on the next rent.

**Disposal chain:** `ColumnarMaterializedQueryResult.DisposeAsync` disposes `IDisposable` columns. `PooledFixedWidthColumnChunk.Dispose` disposes value and null `IMemoryOwner<byte>` instances. Until the query result is disposed, buffers stay rented — important for callers iterating `IColumnarQueryResult.Batches`.

---

## 6.17 Large Object Heap (LOH) considerations in RainDB

On 64-bit .NET, the LOH threshold is approximately **85,000 bytes**. Objects larger than this are:

- Allocated on the LOH, which is collected less frequently
- Not compacted by default (though .NET 5+ can compact on demand)
- Expensive to allocate and release under memory pressure

RainDB's comment in `HybridBufferPool.RentAligned` explicitly targets **sub-LOH** vector sizes where practical. Typical batch projections:

| Scenario | Rows | Type | Value bytes |
|----------|------|------|-------------|
| Half batch | 32K | Int32 | 128 KiB |
| Full strict min | 64K | Float64 | 512 KiB |
| Small test batch | 1K | Int32 | 4 KiB |

A 64K-row `Float64` column projection is **512 KiB** — above LOH. RainDB still uses `ArrayPool` for these; pooled large arrays may still land on LOH buckets in practice, but **reuse** across queries amortizes allocation cost versus `new byte[n]` every time.

**Mitigation strategies in the codebase:**

1. **Batch sizing policy** (`VectorChunkLimits`) keeps production vectors in a predictable range
2. **Pooling** avoids allocation churn even when individual rents are large
3. **Aligned rent padding** adds at most `alignment - 1` extra bytes — negligible relative to column data

For null bitmaps, size is `ceil(rowCount / 8)` — 8 KiB for 64K rows, well under LOH.

---

## 6.18 Wiring pools through IExecutionContext

```csharp
// src/RainDB.Abstractions/Execution/IExecutionContext.cs
public interface IExecutionContext
{
    ICatalog Catalog { get; }
    IBufferPool BufferPool { get; }
    IAlignedBufferPool AlignedBufferPool { get; }
    ISpillWriter SpillWriter { get; }
    CancellationToken CancellationToken { get; }
}
```

```csharp
// src/RainDB.Query/Runtime/RainDbExecutionContext.cs
public sealed class RainDbExecutionContext : IExecutionContext
{
    public RainDbExecutionContext(
        ICatalog catalog,
        IBufferPool bufferPool,
        IAlignedBufferPool alignedBufferPool,
        ISpillWriter spillWriter,
        CancellationToken cancellationToken = default)
    { ... }
}
```

```csharp
// src/RainDB.Driver/RainDbEngine.cs
var buffers = new HybridBufferPool();
return new RainDbEngine(catalog, buffers, buffers, executor, ...);
```

The same `HybridBufferPool` instance serves both interfaces — one pool, two views. `DefaultQueryExecutor` validates `context.AlignedBufferPool` is non-null before dispatch.

---

## 6.19 ProjectGather: dual-pool usage

`ProjectGather` is the primary consumer of aligned buffers in the scan pipeline:

```csharp
// src/RainDB.Query/Vectorized/ProjectGather.cs
internal static ColumnarBatch Project(
    IColumnarBatch batch,
    ReadOnlySpan<int> outputColumnIndices,
    bool useRowSelection,
    ReadOnlySpan<int> selectedRows,
    int selectedCount,
    IBufferPool bufferPool,
    IAlignedBufferPool alignedBufferPool)
{
    var cols = new IColumnChunk[outputColumnIndices.Length];
    for (var c = 0; c < outputColumnIndices.Length; c++)
    {
        cols[c] = GatherColumn(
            batch.Columns[outputColumnIndices[c]],
            useRowSelection, selectedRows, selectedCount,
            bufferPool, alignedBufferPool);
    }
    return new ColumnarBatch(selectedCount, cols);
}
```

### Fixed-width gather with selection

```csharp
var valueBytes = checked(selectedCount * w);
var valuesOwner = alignedBufferPool.RentAligned(valueBytes);
var outValues = valuesOwner.Memory.Span[..valueBytes];

if (source.HasNulls)
{
    var nbBytes = ColumnTypeSizes.NullBitmapBytes(selectedCount);
    var nullOwner = CreateNullOwner(bufferPool, bufferPool.Rent(nbBytes), nbBytes);
    // ... copy non-null values, set null bits ...
    return new PooledFixedWidthColumnChunk(
        type, selectedCount, valuesOwner, valueBytes, nullOwner, nbBytes, anyNull);
}

// dense copy loop
for (var o = 0; o < selectedCount; o++)
{
    var r = RowAt(useRowSelection, selectedRows, o);
    srcValues.Slice(r * w, w).CopyTo(outValues.Slice(o * w, w));
}
return new PooledFixedWidthColumnChunk(
    type, selectedCount, valuesOwner, valueBytes,
    nullBitmapOwner: null, nullBitmapBytes: 0, hasNulls: false);
```

**Pool split rationale:**

| Buffer | Pool | Why |
|--------|------|-----|
| Value column | `IAlignedBufferPool` | SIMD-friendly; may feed vectorized kernels |
| Null bitmap | `IBufferPool` | Byte-level bit ops; alignment unnecessary |

### Full-column copy fast path

When the filter passes all rows (`!useRowSelection && selectedCount == source.RowCount`):

```csharp
private static IColumnChunk CopyEntireFixedWidthColumn(...)
{
    var valueBytes = checked(rowCount * w);
    var valuesOwner = alignedBufferPool.RentAligned(valueBytes);
    srcValues.CopyTo(valuesOwner.Memory.Span[..valueBytes]);
    // null bitmap via bufferPool.Rent if needed
}
```

This avoids per-row index indirection when `WHERE` is absent or tautological.

### UTF-8 gather paths

UTF-8 columns **do not** use the aligned pool today:

```csharp
if (source.PhysicalType == RainDbType.Utf8)
{
    if (source is Utf8ColumnChunk utf8)
        return GatherUtf8Arrow(utf8, ...);
    if (source is Utf8LengthPrefixedColumnChunk lp)
        return GatherUtf8LengthPrefixed(lp, ...);
}
```

`GatherUtf8Arrow` builds `int[]` offsets and `List<byte>` blob — heap allocations acceptable for variable-length data where SIMD gather is not yet implemented. Null bitmaps use `new byte[nbBytes]` for small outputs.

---

## 6.20 PooledFixedWidthColumnChunk: ownership bridge

```csharp
// src/RainDB.Query/Vectorized/PooledFixedWidthColumnChunk.cs
internal sealed class PooledFixedWidthColumnChunk : IColumnChunk, IDisposable
{
    private IMemoryOwner<byte>? _valuesOwner;
    private IMemoryOwner<byte>? _nullBitmapOwner;

    public void Dispose()
    {
        _valuesOwner?.Dispose();
        _valuesOwner = null;
        _nullBitmapOwner?.Dispose();
        _nullBitmapOwner = null;
    }
}
```

`Values` and `NullBitmap` throw `ObjectDisposedException` after dispose — tested in `RobustnessAndEdgeCaseTests.Pooled_column_chunk_throws_after_dispose`.

**Invariant:** `valueBytes == rowCount * FixedWidthBytes(type)` enforced at construction.

---

## 6.21 ArrayPool<int> in VectorizedScanEngine

Selection indices are `int`, not `byte`. The scan engine uses **`ArrayPool<int>.Shared`** directly — not `IBufferPool`:

```csharp
// src/RainDB.Query/Execution/VectorizedScanEngine.cs
private static IColumnarBatch ProcessOneBatch(...)
{
    var rent = ArrayPool<int>.Shared.Rent(batch.RowCount);
    try
    {
        var span = rent.AsSpan(0, batch.RowCount);
        int selected;
        var hasFilters = plan.Filters is { Length: > 0 };
        if (hasFilters)
            selected = SelectionEvaluator.FillSelectedRowsConjunctive(
                batch, plan.Filters!, span);
        else
            selected = batch.RowCount;

        return ProjectGather.Project(
            batch, plan.OutputColumnIndices.AsSpan(),
            useRowSelection: hasFilters,
            selectedRows: hasFilters ? span[..selected] : ReadOnlySpan<int>.Empty,
            selectedCount: selected,
            context.BufferPool, context.AlignedBufferPool);
    }
    finally
    {
        ArrayPool<int>.Shared.Return(rent);
    }
}
```

**Why a separate pool?**

- `IBufferPool` is byte-oriented — casting `int[]` through `MemoryMarshal` would be awkward and error-prone
- `ArrayPool<int>.Shared` is optimized for exactly this pattern
- Selection buffers are **ephemeral per batch** — returned before `ProjectGather` completes, while projected columns outlive the batch processing

**Rent sizing:** `Rent(batch.RowCount)` may return an array longer than `batch.RowCount`; only `span[..batch.RowCount]` is used for filling. After filter, `span[..selected]` is passed to `ProjectGather`.

The same `ArrayPool<int>` pattern appears in `AccumulateAggregateBatch` and `AccumulateFilteredRowCount` for filtered aggregates.

**LOH note:** A 64K-row `int[]` is 256 KiB — above LOH. Pooling still helps; the array is returned immediately after each batch.

---

## 6.22 ProjectGather's nested PooledBufferOwner

Null bitmaps use a small nested owner wrapping `IBufferPool`:

```csharp
private sealed class PooledBufferOwner : IMemoryOwner<byte>
{
    private readonly IBufferPool _pool;
    private byte[]? _buffer;

    public void Dispose()
    {
        var b = Interlocked.Exchange(ref _buffer, null);
        if (b is not null)
            _pool.Return(b);
    }
}
```

This mirrors `PooledAlignedMemoryOwner` but without alignment offset — same dispose idiom, different pool interface method.

---

## 6.23 Parallel batch processing and pool thread safety

`VectorizedScanEngine` processes batches in parallel via `Parallel.For` or channel workers. `ArrayPool<byte>.Shared` and `ArrayPool<int>.Shared` are thread-safe. Each parallel task:

1. Rents its own `int[]` selection buffer
2. Rents its own aligned value owners via `ProjectGather`
3. Returns `int[]` before the task completes
4. Leaves `PooledFixedWidthColumnChunk` owners alive until query result disposal

No cross-thread sharing of a single rented buffer occurs — avoiding data races without locks on the pool.

---

## 6.24 Mental model

```
  VectorizedScanEngine.ProcessOneBatch
           │
           ├─ ArrayPool<int>.Shared  ──► selection indices (per batch, returned immediately)
           │
           └─ ProjectGather.Project
                    │
                    ├─ IAlignedBufferPool ──► PooledAlignedMemoryOwner ──► value bytes
                    │
                    └─ IBufferPool ──► PooledBufferOwner ──► null bitmap bytes
                              │
                              ▼
                    PooledFixedWidthColumnChunk (disposed with query result)
```

Buffer management is not an optimization bolt-on — it is how RainDB keeps the Phase 1 scan/project path viable for multi-batch analytical queries without allocating fresh column backing stores for every operator invocation.

---

## 6.25 Quick reference

| Component | Path |
|-----------|------|
| `IBufferPool` | `src/RainDB.Abstractions/Memory/IBufferPool.cs` |
| `IAlignedBufferPool` | `src/RainDB.Abstractions/Memory/IAlignedBufferPool.cs` |
| `SimdAlignment` | `src/RainDB.Abstractions/Memory/SimdAlignment.cs` |
| `HybridBufferPool` | `src/RainDB.Core/Memory/HybridBufferPool.cs` |
| `PooledAlignedMemoryOwner` | `src/RainDB.Core/Memory/PooledAlignedMemoryOwner.cs` |
| `ArrayPoolBufferPool` | `src/RainDB.Core/Memory/ArrayPoolBufferPool.cs` |
| `ProjectGather` | `src/RainDB.Query/Vectorized/ProjectGather.cs` |
| `PooledFixedWidthColumnChunk` | `src/RainDB.Query/Vectorized/PooledFixedWidthColumnChunk.cs` |
| Scan `ArrayPool<int>` usage | `src/RainDB.Query/Execution/VectorizedScanEngine.cs` |

Chapter 8 details how selection indices feed `ProjectGather`; Chapter 7 shows how execution context delivers pools to every engine.
