# GCSFuse Components - Quick Reference Guide

## Component Summary Table

| Aspect | FileHandle | CacheHandle | FileInfo |
|--------|-----------|-------------|----------|
| **Purpose** | FUSE file handle abstraction | Local cache file coordination | Cache metadata storage |
| **Scope** | Per file open (per handle) | Per read instance | Per cached object (global) |
| **Lifetime** | open() → close() | First cache read → error/close | First cache access → LRU eviction |
| **Location** | Temporary (fs.handles map) | Temporary (FileCacheReader) | Persistent (LRU cache) |
| **Thread Safe** | Yes (InvariantMutex) | No (per-instance) | Yes (LRU internal locks) |
| **Key Responsibility** | Read/write orchestration | Download coordination | Metadata & generation tracking |

## File Locations Reference

```
FileHandle:
  Definition:    /internal/fs/handle/file.go:37-78
  Constructor:   /internal/fs/handle/file.go:81-97
  Destroy:       /internal/fs/handle/file.go:104-115
  Read:          /internal/fs/handle/file.go:244-302
  ReadWithReadManager: /internal/fs/handle/file.go:164-237
  
CacheHandle:
  Definition:    /internal/cache/file/cache_handle.go:30-52
  Constructor:   /internal/cache/file/cache_handle.go:54-64
  Read:          /internal/cache/file/cache_handle.go:151-269
  Close:         /internal/cache/file/cache_handle.go:290-301
  IsSequential:  /internal/cache/file/cache_handle.go:273-287

FileInfo:
  Definition:    /internal/cache/data/file_info.go:26-61
  Key Type:      /internal/cache/data/file_info.go:26-36
  Key Generation:/internal/cache/data/file_info.go:34-44

CacheHandler:
  Definition:    /internal/cache/file/cache_handler.go:38-62
  GetCacheHandle:/internal/cache/file/cache_handler.go:230-269
  Create FileInfo: /internal/cache/file/cache_handler.go:131-217

FileSystem Integration:
  OpenFile:      /internal/fs/fs.go:2775-2813 (FileHandle creation)
  ReadFile:      /internal/fs/fs.go:2816-2876 (FileHandle usage)
  ReleaseFile:   /internal/fs/fs.go:2993-3010 (FileHandle destruction)
  FS Init:       /internal/fs/fs.go:147-277 (CacheHandler creation)

FileInode Integration:
  Registration:  /internal/fs/inode/file.go:433-437
  Deregistration:/internal/fs/inode/file.go:440-459
```

## Key Code Patterns

### Creating FileHandle (OpenFile)
```go
// From: /internal/fs/fs.go:2803
fs.handles[handleID] = handle.NewFileHandle(
    in,                           // *FileInode
    fs.fileCacheHandler,          // *CacheHandler (can be nil)
    fs.cacheFileForRangeRead,     // bool
    fs.metricHandle,              // MetricHandle
    openMode,                     // util.OpenMode
    fs.newConfig,                 // *cfg.Config
    fs.bufferedReadWorkerPool,    // WorkerPool
    fs.globalMaxReadBlocksSem,    // *Semaphore
)
```

### Creating CacheHandle (First Cache Read)
```go
// From: /internal/cache/file/cache_handler.go:230-269
cacheHandle, err := cacheHandler.GetCacheHandle(
    object,              // *gcs.MinObject
    bucket,              // gcs.Bucket
    cacheFileForRangeRead, // bool
    initialOffset,       // int64
)
// Returns:
// - New CacheHandle with file opened, job created, FileInfo added
// - Error if excluded from cache, generation mismatch, etc.
```

### Creating FileInfo
```go
// From: /internal/cache/file/cache_handler.go:191-196
fileInfo := data.FileInfo{
    Key: FileInfoKey{
        BucketName: bucket.Name(),
        ObjectName: object.Name,
    },
    ObjectGeneration: object.Generation,  // From GCS
    Offset:           0,                  // Download starts here
    FileSize:         object.Size,        // From GCS
}

// Then inserted into LRU cache via:
// evictedValues, err := chr.fileInfoCache.Insert(fileInfoKeyName, fileInfo)
```

### Reading via CacheHandle
```go
// From: /internal/cache/file/cache_handle.go:151-269
n, cacheHit, err := cacheHandle.Read(
    ctx,              // context.Context
    bucket,           // gcs.Bucket
    object,           // *gcs.MinObject
    offset,           // int64
    dst,              // []byte buffer
)
// Does:
// 1. Determines if read is sequential
// 2. Coordinates Job download
// 3. Reads from local file
// 4. Validates FileInfo
```

### FileInfo Key Generation
```go
// From: /internal/cache/data/file_info.go:38-44
fileInfoKey := data.FileInfoKey{
    BucketName: "my-bucket",
    ObjectName: "path/to/file.txt",
    BucketCreationTime: time.Unix(1609459200, 0),
}
key, err := fileInfoKey.Key()
// Generates: "my-bucket1609459200path/to/file.txt"
```

### FileHandle Destruction
```go
// From: /internal/fs/fs.go:2993-3010
fs.mu.Lock()
fileHandle := fs.handles[op.Handle].(*handle.FileHandle)
delete(fs.handles, op.Handle)
fs.mu.Unlock()

fileHandle.Lock()
defer fileHandle.Unlock()
fileHandle.Destroy()  // Cleans up reader/readManager
```

### FileInfo Eviction Cleanup
```go
// From: /internal/cache/file/cache_handler.go:106-129
func (chr *CacheHandler) cleanUpEvictedFile(fileInfo *data.FileInfo) error {
    key := fileInfo.Key
    // 1. Invalidate and remove download job
    chr.jobManager.InvalidateAndRemoveJob(key.ObjectName, key.BucketName)
    // 2. Truncate and delete local cache file
    localFilePath := util.GetDownloadPath(chr.cacheDir, 
        util.GetObjectPath(key.BucketName, key.ObjectName))
    return util.TruncateAndRemoveFile(localFilePath)
}
```

## Important Constants

```go
// ReadChunkSize - threshold for sequential vs random read detection
// From: /internal/cache/file/downloader/job.go:49
const ReadChunkSize = 8 * cacheutil.MiB  // 8 MB

// Job status states
// From: /internal/cache/file/downloader/job.go:39-47
const (
    NotStarted  jobStatusName = "NotStarted"
    Downloading jobStatusName = "Downloading"
    Completed   jobStatusName = "Completed"
    Failed      jobStatusName = "Failed"
    Invalid     jobStatusName = "Invalid"
)
```

## Common Error Scenarios

### 1. Generation Mismatch
```
Scenario: Object updated in GCS (new generation)
Detection: cacheHandle.validateEntryInFileInfoCache()
Action:   - Old FileInfo evicted from cache
          - Old Job invalidated & download cancelled
          - New FileInfo created for new generation
          - New Job created for download
```

### 2. Cache Capacity Exceeded
```
Scenario: LRU cache full, new FileInfo needs space
Detection: LRU.Insert() capacity check
Action:   - Oldest FileInfo evicted
          - cleanUpEvictedFile() called
          - Associated Job invalidated
          - Local cache file deleted
```

### 3. Reader Generation Invalid
```
Scenario: Inode becomes dirty (write or external update)
Detection: FileHandle.isValidReader() returns false
Action:   - Old reader destroyed
          - New reader created with current generation
          - Next read uses new reader
```

### 4. Job Download Failure
```
Scenario: Download from GCS fails (network error, etc.)
Detection: CacheHandle.shouldReadFromCache() checks jobStatus.Err
Action:   - cacheHit = false
          - Fall back to direct GCS read
          - For sequential: may retry
          - For random: skip cache, read from GCS
```

### 5. Cache File Deleted Externally
```
Scenario: Cache file deleted outside of gcsfuse
Detection: CacheHandler.createLocalFileReadHandle() fails
Result:   - ErrFileNotPresentInCache returned
          - FileInfo evicted from cache
          - Fall back to GCS read
```

## Configuration Impact

### FileHandle
- `sequentialReadSizeMb`: Size of read chunks from GCS
  - Larger = better for sequential workloads
  - Smaller = better for random workloads
- `EnableNewReader`: Use ReadManager (true) vs RandomReader (false)
- `bufferedReadWorkerPool`: Thread pool size for prefetch

### CacheHandle
- `cacheFileForRangeRead`: Whether to download on random reads
  - true: Downloads initiated for all reads
  - false: Downloads only for sequential reads
- `MaxReadAhead`: How much to pre-download ahead of current read

### FileInfo/LRU
- `CacheMaxSize`: Maximum bytes for file cache
  - Larger = more files cached
  - Smaller = more evictions
- `MaxParallelDownloads`: Concurrent download limit
  - Higher = more parallelism
  - Lower = less resource usage

## Concurrency Model

### Lock Ordering
```
Always acquire locks in this order to prevent deadlock:
1. FileSystem.mu (if needed)
2. FileInode.mu (if needed)
3. FileHandle.mu
4. CacheHandler.mu
5. JobManager.mu
6. Job.mu
```

### Shared Resources
```
Shared Across Multiple FileHandles:
- FileInode (one inode per file)
- CacheHandler (one per filesystem)
- JobManager (one per filesystem)
- LRU Cache (one per filesystem)
- Individual Jobs (shared download progress)

Per-FileHandle Resources:
- FileHandle itself
- Reader/ReadManager (lazy created, destroyed on invalidation)

Per-Read Resources:
- CacheHandle (created per read attempt)
- Local file OS handle
- Download job reference (not exclusive)
```

## Performance Characteristics

### Best Case (Sequential Read, Cache Hit)
```
Time: O(download_time)
Steps:
1. Sequential detected → waitForDownload=true
2. Job.Download() blocks until data available
3. fileHandle.ReadAt() from local file
4. No GCS latency (data already present)
```

### Worst Case (Random Read, Cache Miss, No Prefetch)
```
Time: O(gcs_latency)
Steps:
1. Random detected → waitForDownload=false (if cacheFileForRangeRead=false)
2. Job.Download() returns immediately
3. CacheHandle.Read() falls back to GCS
4. Direct GCS API read (full network latency)
```

### Memory Usage
```
Per FileHandle:
- ~1 KB (struct overhead)
- Reader/ReadManager: ~10-100 KB (depending on buffering)

Per CacheHandle:
- ~1 KB (struct overhead)
- OS file handle: minimal

Per Job:
- ~10 KB (struct overhead)
- Download buffers: configurable (typically 1-8 MB)

Per FileInfo:
- ~200 bytes (struct overhead)
- In LRU cache key: variable (bucket + object name)

Aggregate:
- LRU cache: configurable total size (e.g., 100 MB)
- Buffered read blocks: limited by globalMaxReadBlocksSem
- Download concurrency: limited by maxParallelismSem
```

## Testing Entry Points

### Unit Tests
- `/internal/fs/handle/file_test.go` - FileHandle tests
- `/internal/cache/file/cache_handle_test.go` - CacheHandle tests
- `/internal/cache/data/file_info_test.go` - FileInfo tests
- `/internal/cache/file/cache_handler_test.go` - CacheHandler tests

### Integration Tests
- `/tools/integration_tests/` - Full integration scenarios

### Mock Objects
- `fake.NewFakeBucket()` - Mock bucket for testing
- `fake.FakeStorage` - Mock GCS storage
- Test JobManager/Job with mocked downloads

## Debugging Tips

### Enable Trace Logging
```go
// For FileCacheReader operations
// logger.Tracef() calls in file_cache_reader.go

// For CacheHandle.Read()
// Check validation errors and cache hit status

// For Job operations
// Job state transitions logged
```

### Inspect Runtime State
```go
// Check FileHandle.reader/readManager existence
fh.mu.Lock()
readerExists := fh.reader != nil
readManagerExists := fh.readManager != nil
fh.mu.Unlock()

// Check Job status
jobStatus := job.GetStatus()
// {Name: "Downloading", Offset: 5000000, Error: nil}

// Check LRU cache entry
fileInfo := fh.fileInfoCache.LookUpWithoutChangingOrder(key)
// Shows if entry exists and its state
```

### Common Issues to Check
1. **Generation mismatch**: Monitor object.Generation changes
2. **Cache exhaustion**: Watch LRU eviction statistics
3. **Download hangs**: Check Job state and subscriber notifications
4. **Lock contention**: Monitor mutex acquisition times
5. **Memory leaks**: Ensure CacheHandle.Close() called on errors

## Version Notes

Current implementation uses:
- **New ReadManager path** (recommended) - prefer EnableNewReader=true
- **FileCacheReader** - handles cache coordination
- **LRU Cache** - for FileInfo storage
- **Async Jobs** - for non-blocking downloads
- **Generation-based invalidation** - detects external object changes

Legacy path still supported:
- **RandomReader path** - direct reader access (being phased out)

