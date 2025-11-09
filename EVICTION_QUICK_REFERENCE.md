# GCS Fuse Eviction Flow - Quick Reference

## Quick Summary

The gcsfuse caching system uses **Least Recently Used (LRU) eviction** to manage cache size. When the cache exceeds `max-size-mb`, the least recently accessed FileInfo entries are evicted, triggering cleanup of associated cache files and download jobs.

---

## What Gets Evicted

| Item | Type | Location |
|------|------|----------|
| **FileInfo** | Metadata struct | In-memory LRU cache |
| **Cache Files** | Disk files | `{CACHE_DIR}/gcsfuse-file-cache/` |
| **Download Jobs** | Job structs | JobManager memory |

---

## What Triggers Eviction

| Trigger | Location | Details |
|---------|----------|---------|
| **LRU** | `lru.go:128-139` | Evict when cache size > maxSize |
| **Generation Change** | `cache_handler.go:177` | Object has new version in GCS |
| **Failed Job** | `cache_handler.go:175` | Job status is Failed or Invalid |
| **Manual** | `cache_handler.go:275` | `InvalidateCache()` called |
| **Prefix Cleanup** | `lru.go:272` | `EraseEntriesWithGivenPrefix()` |

---

## Quick Eviction Sequence

```
1. Insert into cache (cache_handler.go:198)
   └─> fileInfoCache.Insert(key, fileInfo)

2. LRU eviction loop (lru.go:179-181)
   └─> While cache.currentSize > maxSize:
       └─> evictOne() removes LRU entry

3. Process evicted entries (cache_handler.go:204-210)
   └─> For each FileInfo:
       └─> cleanUpEvictedFile()

4. Cleanup steps (cache_handler.go:109-129)
   ├─ JobManager.InvalidateAndRemoveJob()
   │  └─ Job.Invalidate()
   └─ TruncateAndRemoveFile()
      ├─ os.Truncate(path, 0)
      └─ os.Remove(path)
```

---

## Key Components

| Component | File | Purpose |
|-----------|------|---------|
| **LRU Cache** | `lru/lru.go` | Core eviction algorithm (doubly-linked list + hash map) |
| **CacheHandler** | `file/cache_handler.go` | Coordinates eviction, cleanup, generation checks |
| **JobManager** | `downloader/downloader.go` | Manages download job lifecycle |
| **Job** | `downloader/job.go` | Async download of GCS object |
| **CacheHandle** | `file/cache_handle.go` | Read interface to cached file |
| **FileInfo** | `data/file_info.go` | Metadata (generation, offset, size) |

---

## Configuration Parameters

| Parameter | File | Type | Meaning |
|-----------|------|------|---------|
| `max-size-mb` | `cfg/config.go:432` | int64 | Cache size limit in MiB (-1 = unlimited) |
| `exclude-regex` | `cfg/config.go:424` | string | Pattern to exclude from cache |
| `include-regex` | `cfg/config.go:428` | string | Pattern to include in cache |
| `cache-file-for-range-read` | `cfg/config.go:414` | bool | Cache range reads? |

**Initialization**: `fs.go:255-260` converts MaxSizeMb to bytes and creates LRU cache

---

## Interaction Model

```
FileInfo Lifecycle:
  Creation → LRU Cache Insertion → Access (Updates LRU order)
             ↓
  Eviction ← LRU becomes least recently used
             ↓
  Cleanup ← Job invalidated + File deleted
```

**CacheHandle**:
- Created with reference to FileInfo cache
- Reads from local cache file
- Detects eviction via FileInfo lookup failure
- Falls back to GCS if FileInfo evicted

**File Handle**:
- Open file handle persists through eviction
- File truncated to 0 (frees space immediately)
- Deletion deferred if handles still open
- CacheHandle.Read() detects eviction on next access

---

## Cleanup Process (8 Steps)

```
Step 1: LRU eviction triggered
        └─> evictOne() removes from linked list and hash map

Step 2: FileInfo cleanup initiated
        └─> cleanUpEvictedFile() called

Step 3: Job invalidation
        └─> JobManager.InvalidateAndRemoveJob()

Step 4: Job state changed to Invalid
        └─> Job.Invalidate() cancels download

Step 5: Download cancelled
        └─> Cancel async download goroutine

Step 6: Callback removes job from JobManager
        └─> removeJobCallback() executed

Step 7: File truncated
        └─> os.Truncate(path, 0)

Step 8: File deleted
        └─> os.Remove(path)
```

---

## Edge Cases & Special Handling

| Case | Handling | Location |
|------|----------|----------|
| **Open file handle during eviction** | Truncate frees space; delete deferred | `util.go:171-183` |
| **Eviction during read** | CacheHandle detects via FileInfo lookup | `cache_handle.go:131-144` |
| **Large entry > maxSize** | Rejected with `ErrInvalidEntrySize` | `lru.go:156` |
| **Multiple rapid evictions** | Loop until cache size OK | `lru.go:179-181` |
| **Generation mismatch** | Evict old, re-cache with new generation | `cache_handler.go:177` |
| **Failed job detection** | Check job status or missing job + partial offset | `cache_handler.go:172-176` |
| **Concurrent access** | Protected by RWLocker (LRU), Locker (Handler/Job) | Multiple files |
| **Graceful shutdown** | All jobs invalidated on unmount | `downloader.go:158-171` |

---

## Test Cases (Examples)

**Location**: `lru_test.go`

- `ExpiresLeastRecentlyUsed()` (line 104): LRU eviction
- `TestMultipleEviction()` (line 139): Multiple entries evicted
- `Overwrite()` (line 125): Update with size change
- `TestEraseCacheWithGivenPrefix()` (line 171): Batch deletion
- `TestWhenEntrySizeMoreThanCacheMaxSize()` (line 153): Oversized entry

---

## Critical Code Sections

| What | File | Lines | What to Know |
|------|------|-------|--------------|
| **Eviction trigger** | `lru.go` | 179-181 | While loop until size OK |
| **LRU removal** | `lru.go` | 128-139 | Takes from back of list |
| **Job invalidation** | `job.go` | 195-212 | Changes status, cancels download |
| **File cleanup** | `util.go` | 171-183 | Truncate then remove |
| **Generation check** | `cache_handler.go` | 177 | If different, evict + re-cache |
| **Cache initialization** | `fs.go` | 255-260 | Creates LRU with size limit |
| **Deadlock prevention** | `downloader.go` | 147-149 | Unlock before calling Invalidate() |

---

## Performance Notes

- **LRU operations**: O(1) (doubly-linked list + hash map)
- **Eviction cost**: Proportional to number of entries evicted
- **File cleanup**: Sequential (truncate then remove)
- **Lock contention**: Multiple locks (LRU, Handler, JobManager, Job)
- **Deadlock prevention**: Careful lock ordering and releases

---

## Related Documentation

- **FileHandle Lifecycle**: See COMPONENT_LIFECYCLE_ANALYSIS.md
- **CacheHandle Lifecycle**: See COMPONENT_LIFECYCLE_ANALYSIS.md
- **FileInfo Lifecycle**: See COMPONENT_LIFECYCLE_ANALYSIS.md
- **Architecture Overview**: See COMPONENT_ARCHITECTURE_SUMMARY.md

---

## Quick Debug Tips

1. **Check cache size**: Monitor currentSize vs maxSize in LRU cache
2. **Verify eviction**: Look for "Job invalid" logs in downloader
3. **Check generation**: Compare ObjectGeneration in FileInfo vs GCS
4. **Verify cleanup**: Confirm cache files deleted from disk
5. **Test failures**: Job status check (Failed vs Invalid)
6. **Concurrent issues**: Check lock acquisition order

