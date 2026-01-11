### Pebble Overview

`db.go` is the core file defining Pebble's main database type and its primary operations.

**Key Components:**

| Component        | Purpose                                                         |
| ---------------- | --------------------------------------------------------------- |
| `d.mu`           | Main mutex protecting version, memtable queue, compaction state |
| `d.commit`       | Commit pipeline for batching/parallelizing writes               |
| `d.readState`    | Lock-free read path via atomic pointer swap                     |
| `d.mu.mem.queue` | Memtable queue (mutable at end, immutable waiting flush)        |
| `d.mu.versions`  | LSM version set (immutable snapshots of file metadata)          |
| `d.mu.compact`   | Compaction scheduling state                                     |

**Design Highlights:**

1. **All writes go through Batch** - even single `Set()` creates a batch internally
2. **Read path is mostly lock-free** - `readState` provides consistent view without holding `d.mu`
3. **Iterator construction pools allocations** - `iterAlloc` + `sync.Pool` minimizes GC pressure
4. **Lazy combined iteration** - range key iterator only constructed when range keys exist

**Structure Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│                      INTERFACES (71-226)                    │
│  Reader: Get, NewIter, NewIterWithContext, Close            │
│  Writer: Apply, Set, Delete, Merge, RangeKey ops, etc.      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    DB STRUCT (254-552)                      │
│  - Atomics: memTableCount, logSize, diskAvailBytes          │
│  - Core: cacheHandle, dirname, opts, comparer funcs         │
│  - Storage: objProvider, fileCache, newIters                │
│  - Concurrency: commit pipeline, readState                  │
│  - d.mu protected state:                                    │
│      - formatVers, versions, log, mem, compact              │
│      - snapshots, tableStats, tableValidation               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 WRITE PATH (583-947)                            │
│  Set, Delete, DeleteSized, SingleDelete, DeleteRange,           │
│  Merge, LogData, RangeKeySet/Unset/Delete, Apply                │
│  └─> All funnel through Batch + commit pipeline                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 READ PATH / ITERATORS (949-1255)                │
│  iterAlloc: pooled allocation for iterator components           │
│  newIter: constructs merged iterator over memtables + LSM       │
│  finishInitializingIter: sets up point/range key iteration      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              SNAPSHOTS & LIFECYCLE (1441-1693)                  │
│  NewBatch, NewIndexedBatch, NewIter, NewSnapshot,               │
│  NewEventuallyFileOnlySnapshot, Close                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              COMPACTION & FLUSH (1695-1936)                     │
│  Compact: manual compaction over key range                      │
│  Flush/AsyncFlush: force memtable to SST                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    METRICS (1938+)                              │
│  Metrics(): aggregates stats from all subsystems                │
└─────────────────────────────────────────────────────────────────┘
```

---

### Iterator Overview

The `Iterator` struct (iterator.go:199-338) is Pebble's user-facing iterator over key/value pairs.

**Core Architecture:**

```
┌──────────────────────────────────────────────────────────────────┐
│                     USER-FACING ITERATOR                         │
│  SeekGE, SeekLT, First, Last, Next, Prev, Key, Value, Close      │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                    INTERLEAVING ITER (optional)                  │
│  Merges point keys with range keys when both are requested       │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                       MERGING ITER                               │
│  Heap-based merge of all levels (batch, memtables, L0-L6)        │
│  Handles MERGE key type, tombstones, sequence number filtering   │
└──────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
   ┌──────────┐        ┌───────────┐        ┌───────────┐
   │ batchIter│        │ memtable  │        │ levelIter │
   │ (pending │        │  iters    │        │  (L0-L6)  │
   │  writes) │        │           │        │           │
   └──────────┘        └───────────┘        └───────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   ┌──────────┐        ┌───────────┐        ┌───────────┐
   │ Batch.   │        │d.mu.mem.  │        │d.mu.      │
   │ index    │        │queue      │        │versions   │
   │          │        │           │        │+fileCache │
   │          │        │           │        │           │
   │ skiplist │        │ skiplist  │        │ SSTs      │
   │ owned by │        │ owned by  │        │ owned by  │
   │ Batch    │        │ DB        │        │ DB        │
   └──────────┘        └───────────┘        └───────────┘
```

**Disk access chain for levelIter:**

```
levelIter → fileCache → objProvider → opts.FS (vfs.FS) → disk
```

- `d.objProvider` (db.go:294) - manages SST file access
- `d.fileCache` (db.go:298) - caches open SST readers
- `vfs.FS` interface - actual filesystem I/O (swappable for testing via `vfs.NewMem()`)

**Key Fields:**

| Field                   | Purpose                                                |
| ----------------------- | ------------------------------------------------------ |
| `iter`                  | Current internal iterator (point-only or interleaved)  |
| `pointIter`             | The point-key merging iterator                         |
| `merging`               | Pointer to the underlying `mergingIter`                |
| `rangeKey`              | Range key iteration state (lazy, may be nil)           |
| `readState` / `version` | Snapshot of LSM state (one or the other)               |
| `batch`                 | State for indexed batch iteration                      |
| `pos`                   | Position relative to internal iterator (cur/next/prev) |
| `iterValidityState`     | Valid, Exhausted, or AtLimit                           |
| `key`, `value`          | Current key/value exposed to user                      |
| `seqNum`                | Snapshot sequence number for visibility                |

**Main Methods:**

| Category        | Methods                                                           |
| --------------- | ----------------------------------------------------------------- |
| **Positioning** | `SeekGE`, `SeekLT`, `SeekPrefixGE`, `First`, `Last`               |
| **Stepping**    | `Next`, `Prev`, `NextPrefix`                                      |
| **Limited**     | `SeekGEWithLimit`, `NextWithLimit`, `PrevWithLimit`               |
| **Access**      | `Key`, `Value`, `LazyValue`, `Valid`, `Error`                     |
| **Range Keys**  | `RangeBounds`, `RangeKeys`, `RangeKeyChanged`, `HasPointAndRange` |
| **Lifecycle**   | `Close`, `Clone`, `SetBounds`, `SetOptions`                       |

**Design Highlights:**

1. **Lazy value fetching** - Values can be stored in blob files; `LazyValue` defers fetch until needed
2. **Position tracking** - `iterPos` tracks whether internal iterator is at current key or advanced past it
3. **Seek optimizations** - Tracks `lastPositioningOp` to enable fast no-op seeks and `trySeekUsingNext`
4. **Range key masking** - Point keys can be masked by range keys with matching suffix
5. **Lazy combined iteration** - Range key iterator only constructed when range keys encountered
6. **Pooled allocation** - Embedded in `iterAlloc` for reuse via `sync.Pool`


### FileCache Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         FileCache                               │
│  Shared across all DBs on a node                                │
│  - refs: reference count                                        │
│  - counts: sstables/blobFiles counters                          │
│  - c: genericcache.Cache[fileCacheKey, fileCacheValue]          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ one per DB
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      fileCacheHandle                            │
│  Per-DB handle to the shared FileCache                          │
│  - fileCache: pointer to shared FileCache                       │
│  - objProvider: access to SST files on disk                     │
│  - blockCacheHandle: for caching SST blocks                     │
│  - readerOpts: sstable.ReaderOptions                            │
│  - iterCount: tracks open iterators (leak detection)            │
└─────────────────────────────────────────────────────────────────┘
```

**Key Data Structures:**

| Type | Purpose |
|------|---------|
| `fileCacheKey` | Cache key: `{handle, fileNum, fileType}` |
| `fileCacheValue` | Cached value: `{reader, isShared, closeHook}` |
| `iterSet` | Bundle of iterators: `{point, rangeDeletion, rangeKey}` |

**Main Operations:**

```
fileCacheHandle.newIters(file, opts, kinds)
        │
        ▼
findOrCreateTable(file)  ──────────────────┐
        │                                  │ cache miss
        │ cache hit                        ▼
        │                           openFile(fileNum)
        │                                  │
        │                                  ▼
        │                           objProvider.OpenForReading()
        │                                  │
        │                                  ▼
        │                           sstable.NewReader() or
        │                           blob.NewFileReader()
        │                                  │
        ▼                                  │
fileCacheValue ◄───────────────────────────┘
        │
        ▼
Create requested iterators (point/rangeDel/rangeKey)
```

**FileCache vs Block Cache:**

| Cache           | What it caches                                                                                                                |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **FileCache**   | Open file readers (`sstable.Reader`, `blob.FileReader`) - includes file handles, parsed metadata, index blocks, filter blocks |
| **Block Cache** | Actual data blocks read from SSTs (the key-value data itself). Block cache is passed into `fileCacheHandle` via `cacheHandle *cache.Handle`                                                                 |

**Both are shared across DBs on a node:**

```
┌──────────────────────────────────────────────────────┐
│                       Node                           │
│  ┌─────────────┐              ┌─────────────┐        │
│  │ FileCache   │              │ Block Cache │        │
│  │ (readers)   │              │ (data)      │        │
│  └──────┬──────┘              └──────┬──────┘        │
│         │           shared           │               │
│    ┌────┴────┬──────────────────┬────┴────┐          │
│    ▼         ▼                  ▼         ▼          │
│  ┌───┐     ┌───┐              ┌───┐     ┌───┐        │
│  │DB1│     │DB2│              │DB1│     │DB2│        │
│  └───┘     └───┘              └───┘     └───┘        │
└──────────────────────────────────────────────────────┘
```

Each DB gets its own `fileCacheHandle` and `cache.Handle`, but they point to shared underlying caches.

---

### Block Cache Overview

The block cache (`internal/cache/cache.go`) caches actual data blocks read from SST files.

```
┌─────────────────────────────────────────────────────────────────┐
│                           Cache                                 │
│  - refs: reference count                                        │
│  - maxSize: total cache size                                    │
│  - shards: []shard (4 × NumCPUs for concurrency)               │
│  - idAlloc: allocates unique IDs for Handles                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ one per DB
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Handle                                 │
│  Per-DB namespace into the shared Cache                         │
│  - cache: pointer to shared Cache                               │
│  - id: handleID (namespace for file numbers)                    │
└─────────────────────────────────────────────────────────────────┘
```

**Key Design Points:**

| Aspect | Details |
|--------|---------|
| **Sharding** | `4 × GOMAXPROCS` shards to reduce lock contention |
| **Eviction** | Clock-PRO algorithm (per shard, independent) |
| **Key** | `(handleID, fileNum, offset)` triple |
| **Manual memory** | Uses `C.malloc/free` to avoid GC pressure |
| **Reference counting** | Values are refcounted; freed when count hits 0 |

**Main Operations:**

| Method | Purpose |
|--------|---------|
| `New(size)` | Create cache with auto-calculated shard count |
| `NewHandle()` | Create per-DB handle (namespace) |
| `Handle.Get(fileNum, offset, ...)` | Retrieve cached block |
| `Handle.Set(fileNum, offset, value)` | Store block in cache |
| `Handle.EvictFile(fileNum)` | Remove all blocks for a file (on SST deletion) |

**Memory Management:**

The cache uses manual memory management to reduce GC overhead:

```
┌────────────────────────────────────────┐
│         Go Heap (GC managed)           │
│  - Cache struct                        │
│  - shard structs                       │
│  - map entries                         │
└────────────────────────────────────────┘
                  │
                  │ pointers to
                  ▼
┌────────────────────────────────────────┐
│      C Heap (manual, via malloc)       │
│  - Actual block data ([]byte)          │
│  - Freed when refcount → 0             │
└────────────────────────────────────────┘
```

This was critical for CockroachDB - keeping large block data outside Go's GC significantly reduced GC pause times.

---
