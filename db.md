# db.go Overview

`db.go` is the core file defining Pebble's main database type and its primary operations.

## Key Components

| Component | Purpose |
|-----------|---------|
| `d.mu` | Main mutex protecting version, memtable queue, compaction state |
| `d.commit` | Commit pipeline for batching/parallelizing writes |
| `d.readState` | Lock-free read path via atomic pointer swap |
| `d.mu.mem.queue` | Memtable queue (mutable at end, immutable waiting flush) |
| `d.mu.versions` | LSM version set (immutable snapshots of file metadata) |
| `d.mu.compact` | Compaction scheduling state |

## Design Highlights

1. **All writes go through Batch** - even single `Set()` creates a batch internally
2. **Read path is mostly lock-free** - `readState` provides consistent view without holding `d.mu`
3. **Iterator construction pools allocations** - `iterAlloc` + `sync.Pool` minimizes GC pressure
4. **Lazy combined iteration** - range key iterator only constructed when range keys exist

## Structure Overview

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
