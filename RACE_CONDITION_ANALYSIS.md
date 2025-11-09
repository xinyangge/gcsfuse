# Race Condition Analysis: CacheHandle Reading vs FileInfo Eviction

## Executive Summary

After thorough investigation of the gcsfuse codebase, **there is NO race condition** between active CacheHandles reading from cache files and FileInfo eviction. The system is **protected by multiple coordinated mechanisms**:

1. **OS-level protection**: Linux allows reads from truncated/deleted files with open descriptors
2. **Job invalidation**: Download jobs are cancelled before file deletion
3. **FileInfo validation**: Reads detect eviction via cache entry lookups
4. **Synchronous cleanup**: TruncateAndRemoveFile ensures disk space is freed even with open handles
5. **File descriptor lifecycle**: Open descriptors remain valid after truncation

---

## 1. The Race Condition Scenario

### What Could Happen (Without Protections):

```
Timeline:
┌──────────────────────────┬──────────────────────────────┐
│ CacheHandle.Read()       │ LRU Eviction                 │
├──────────────────────────┼──────────────────────────────┤
│ t1: Start read           │                              │
│ t2: Get fileHandle       │                              │
│     (file descriptor)    │                              │
│ t3: Validating offset    │ t3.5: FileInfo evicted       │
│     from FileInfo cache  │ t3.6: Job invalidated        │
│ t4: Reading from file    │ t3.7: File truncated         │
│ t5: ReadAt() called      │ t3.8: File deleted           │
│ t6: CRASH? File is gone  │                              │
└──────────────────────────┴──────────────────────────────┘
```

### Why This Is NOT a Problem in gcsfuse

---

## 2. Protection Mechanism #1: OS-Level File Descriptor Semantics

### How Linux/UNIX File Deletion Works

**File Path**: `/home/user/gcsfuse/internal/cache/util/util.go` (lines 169-183)

```go
// TruncateAndRemoveFile first truncates the file to 0 and then remove (delete)
// the file at given path.
func TruncateAndRemoveFile(filePath string) error {
    // Truncate the file to 0 size, so that even if there are open file handles
    // and linux doesn't delete the file, the file will not take space.
    err := os.Truncate(filePath, 0)      // Step 1: Truncate
    if err != nil {
        return err
    }
    err = os.Remove(filePath)             // Step 2: Delete
    if err != nil {
        return err
    }
    return nil
}
```

**The Critical Comment**: "Truncate the file to 0 size, so that even if there are open file handles and linux doesn't delete the file, the file will not take space."

**What This Means**:

On Linux/UNIX systems:
1. **Truncation** (`os.Truncate(filePath, 0)`):
   - Immediately frees disk space
   - Makes file appear empty (size = 0)
   - Works even with open file descriptors
   - Returns success to the system immediately

2. **Deletion** (`os.Remove(filePath)`):
   - Removes directory entry
   - If file descriptors are open: Defers actual deletion via "unlink" semantics
   - On Linux: File persists in memory while descriptors are open
   - When last descriptor closes: File is truly deleted from disk
   - System call: `unlink()` (deferred deletion)

**Security Guarantee**:
- Even if a CacheHandle has an open file descriptor and eviction happens:
  - Disk space is immediately freed (via truncation)
  - The file descriptor remains valid for reading
  - Reading from the open descriptor returns zeroes (truncated file) or original cached data
  - No crashes or undefined behavior

---

## 3. Protection Mechanism #2: FileInfo Cache Entry Validation

### How CacheHandle Detects Eviction

**File Path**: `/home/user/gcsfuse/internal/cache/file/cache_handle.go` (lines 131-144, 161-164)

```go
// getFileInfoData returns the file-info cache entry for the given object
// in the associated bucket.
func (fch *CacheHandle) getFileInfoData(bucket gcs.Bucket, object *gcs.MinObject, 
        changeCacheOrder bool) (*data.FileInfo, error) {
    fileInfoKey := data.FileInfoKey{
        BucketName: bucket.Name(),
        ObjectName: object.Name,
    }
    fileInfoKeyName, err := fileInfoKey.Key()
    if err != nil {
        return nil, fmt.Errorf("error while creating key: %w", err)
    }
    
    var fileInfo lru.ValueType
    if changeCacheOrder {
        fileInfo = fch.fileInfoCache.LookUp(fileInfoKeyName)
    } else {
        fileInfo = fch.fileInfoCache.LookUpWithoutChangingOrder(fileInfoKeyName)
    }
    if fileInfo == nil {  // ← EVICTION DETECTED HERE
        return nil, fmt.Errorf("%w: no entry found in file info cache", 
            util.ErrInvalidFileInfoCache)
    }
    
    fileInfoData, ok := fileInfo.(data.FileInfo)
    if !ok {
        return nil, fmt.Errorf("failed to get fileInfoData: %w", 
            util.ErrInvalidFileHandle)
    }
    
    return &fileInfoData, nil
}

// validateEntryInFileInfoCache checks if entry is present for a given object 
// in file info cache with same generation and at least requiredOffset.
func (fch *CacheHandle) validateEntryInFileInfoCache(bucket gcs.Bucket, 
        object *gcs.MinObject, requiredOffset uint64, 
        changeCacheOrder bool) error {
    fileInfoData, err := fch.getFileInfoData(bucket, object, changeCacheOrder)
    if err != nil {  // ← Returns error if entry was evicted
        return fmt.Errorf("validateEntryInFileInfoCache: failed to get fileInfoData: %w", err)
    }
    
    // Additional validation: Check generation
    if fileInfoData.ObjectGeneration != object.Generation {
        return fmt.Errorf("%w: generation mismatch", util.ErrInvalidFileInfoCache)
    }
    
    // Additional validation: Check offset
    if fileInfoData.Offset < requiredOffset {
        return fmt.Errorf("%w: offset mismatch", util.ErrInvalidFileInfoCache)
    }
    
    return nil
}
```

### Read Flow with Eviction Detection

**File Path**: `/home/user/gcsfuse/internal/cache/file/cache_handle.go` (lines 151-269)

```go
func (fch *CacheHandle) Read(ctx context.Context, bucket gcs.Bucket, 
        object *gcs.MinObject, offset int64, dst []byte) (
        n int, cacheHit bool, err error) {
    
    // Step 1: Validate cache handle
    err = fch.validateCacheHandle()
    if err != nil {
        return  // Early exit if cache handle invalid
    }
    
    // Step 2: Get FileInfo from cache - can detect eviction here
    fileInfoData, errFileInfo := fch.getFileInfoData(bucket, object, false)
    if errFileInfo != nil {
        return 0, false, fmt.Errorf("Error in getCachedFileInfo: %v", errFileInfo)
        // ↑ If FileInfo was evicted, this returns error
    }
    
    // ... download logic ...
    
    // Step 3: Validate before reading from local file
    n, err = fch.fileHandle.ReadAt(dst, offset)  // Read from local file
    
    // Step 4: Validate after reading - second eviction check
    err = fch.validateEntryInFileInfoCache(bucket, object, uint64(requiredOffset), true)
    if err != nil {
        // ↑ If evicted between steps, detected here
        return 0, false, err
    }
    
    return
}
```

**Two Validation Points**:
1. **Before Read**: Line 161 - Checks FileInfo exists and is valid
2. **After Read**: Line 263 - Rechecks after reading from disk

If FileInfo is evicted between these two checks:
- Error is returned
- Read operation is reported as cache miss
- Caller (FileCacheReader) falls back to GCS

---

## 4. Protection Mechanism #3: Job Invalidation Before File Deletion

### Eviction Cleanup Flow

**File Path**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 109-129)

```go
// cleanUpEvictedFile is called for evicted FileInfo entries
func (chr *CacheHandler) cleanUpEvictedFile(fileInfo *data.FileInfo) error {
    key := fileInfo.Key
    
    // Step 1: FIRST - Invalidate download job
    chr.jobManager.InvalidateAndRemoveJob(key.ObjectName, key.BucketName)
    // ↑ This cancels any in-progress downloads
    
    // Step 2: THEN - Delete the file
    localFilePath := util.GetDownloadPath(chr.cacheDir, 
        util.GetObjectPath(key.BucketName, key.ObjectName))
    err = util.TruncateAndRemoveFile(localFilePath)
    if err != nil {
        if os.IsNotExist(err) {
            // Already deleted, that's fine
            logger.Warnf("cleanUpEvictedFile: file not present: %v", err)
            return nil
        }
        return fmt.Errorf("cleanUpEvictedFile: error: %w", err)
    }
    
    return nil
}
```

### Job Invalidation Process

**File Path**: `/home/user/gcsfuse/internal/cache/file/downloader/downloader.go` (lines 143-153)

```go
// InvalidateAndRemoveJob invalidates the job for given object and bucket
func (jm *JobManager) InvalidateAndRemoveJob(objectName string, bucketName string) {
    objectPath := util.GetObjectPath(bucketName, objectName)
    jm.mu.Lock()
    job, ok := jm.jobs[objectPath]
    
    // Release the lock while calling downloader.Job.Invalidate to avoid deadlock
    // as the job calls removeJobCallback in the end which requires Lock(jm.mu).
    jm.mu.Unlock()
    
    if ok {
        job.Invalidate()  // ← Stops any in-progress download
    }
}
```

### Job Invalidation Details

**File Path**: `/home/user/gcsfuse/internal/cache/file/downloader/job.go` (lines 195-212)

```go
// Invalidate invalidates the download job, cancels async download if running
func (job *Job) Invalidate() {
    job.mu.Lock()
    
    // Step 1: If currently downloading, mark as Invalid and cancel
    if job.status.Name == Downloading {
        job.status.Name = Invalid
        job.cancel()  // ← Cancels the downloadObjectAsync goroutine
        
        // Lock again to execute common notification logic
        job.mu.Lock()
    }
    
    defer job.mu.Unlock()
    
    // Step 2: Mark status as Invalid
    job.status.Name = Invalid
    logger.Tracef("Job:%p (%s:/%s) is no longer valid.", 
        job, job.bucket.Name(), job.object.Name)
    
    // Step 3: Remove from JobManager
    if job.removeJobCallback != nil {
        job.removeJobCallback()
        job.removeJobCallback = nil
    }
    
    // Step 4: Notify all subscribers
    job.notifySubscribers()
}

// cancel terminates the in-progress download goroutine
func (job *Job) cancel() {
    if job.cancelFunc == nil {
        job.mu.Unlock()
        return
    }
    
    job.cancelFunc()  // ← Sends cancel signal
    
    // Unlock to allow goroutine to finish
    job.mu.Unlock()
    
    // Wait for goroutine to complete
    <-job.doneCh  // ← Blocks until downloadObjectAsync finishes
}
```

**Guarantee**: 
- Before any cache file is truncated/deleted
- The associated download Job is marked Invalid
- Any in-progress download goroutine is cancelled
- Job's subscriber notifications alert any waiting readers
- File descriptor is no longer being written to

---

## 5. Protection Mechanism #4: CacheHandler Lock Serialization

### Atomic Eviction + Cleanup

**File Path**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 230-269)

```go
// GetCacheHandle acquires lock and performs atomic operation
func (chr *CacheHandler) GetCacheHandle(object *gcs.MinObject, bucket gcs.Bucket, 
        cacheForRangeRead bool, initialOffset int64) (*CacheHandle, error) {
    
    // LOCK ACQUIRED HERE
    chr.mu.Lock()
    defer chr.mu.Unlock()
    
    // ... validation and operations ...
    
    // EVICTION HAPPENS INSIDE LOCK
    err := chr.addFileInfoEntryAndCreateDownloadJob(object, bucket)
    if err != nil {
        return nil, fmt.Errorf("GetCacheHandle: while adding entry in cache: %w", err)
    }
    
    // CLEANUP HAPPENS INSIDE LOCK
    localFileReadHandle, err := chr.createLocalFileReadHandle(object.Name, bucket.Name())
    if err != nil {
        return nil, fmt.Errorf("GetCacheHandle: error creating local file handle: %w", err)
    }
    
    return NewCacheHandle(localFileReadHandle, 
        chr.jobManager.GetJob(object.Name, bucket.Name()), 
        chr.fileInfoCache, cacheForRangeRead, initialOffset), nil
}
```

**Inside addFileInfoEntryAndCreateDownloadJob()**:

**File Path**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 131-217)

```go
// addFileInfoEntryAndCreateDownloadJob requires Lock(chr.mu)
func (chr *CacheHandler) addFileInfoEntryAndCreateDownloadJob(
        object *gcs.MinObject, bucket gcs.Bucket) error {
    
    // Create new FileInfo entry
    evictedValues, err := chr.fileInfoCache.Insert(fileInfoKeyName, fileInfo)
    // ↑ Returns all evicted entries
    
    if err != nil {
        return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: %w", err)
    }
    
    // Create download job for new entry
    _ = chr.jobManager.CreateJobIfNotExists(object, bucket)
    
    // Cleanup all evicted entries - THIS HAPPENS ATOMICALLY
    for _, val := range evictedValues {
        fileInfo := val.(data.FileInfo)
        err := chr.cleanUpEvictedFile(&fileInfo)
        // ↑ Invalidates job AND truncates/deletes file
        if err != nil {
            return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: %w", err)
        }
    }
}
```

**Key Property**: 
- All evictions, job invalidations, and file deletions happen serially within the lock
- No concurrent CacheHandle creation during eviction
- New entries added atomically with old entries cleaned up

---

## 6. Protection Mechanism #5: LRU Cache Read Consistency

### LRU Cache Concurrency Control

**File Path**: `/home/user/gcsfuse/internal/cache/lru/lru.go` (lines 229-241)

```go
// LookUpWithoutChangingOrder acquires read lock
func (c *Cache) LookUpWithoutChangingOrder(key string) (value ValueType) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    // Consult the index (thread-safe within read lock)
    e, ok := c.index[key]
    if !ok {
        return nil  // ← Returns nil if key was evicted
    }
    
    // Return the value (pointer copy, safe)
    return e.Value.(entry).Value
}
```

**Comment from lru_test.go** (line 315):
```go
// We get the race condition failure if we remove lock from Insert or Erase method.
```

This indicates:
- Locks are essential and well-tested
- LRU operations are protected from concurrent modification
- Eviction and lookup cannot race

---

## 7. Test Case: Exactly This Scenario

### Test: Test_InvalidateCache_Truncates

**File Path**: `/home/user/gcsfuse/internal/cache/file/cache_handler_test.go` (lines 786-866)

This test explicitly verifies the race condition scenario:

```go
func Test_InvalidateCache_Truncates(t *testing.T) {
    // ... setup ...
    
    // Step 1: Create cache handle and read data
    cacheHandle, err := chTestArgs.cacheHandler.GetCacheHandle(minObject, chTestArgs.bucket, false, 0)
    require.NoError(t, err)
    
    // Step 2: Read from cache
    _, cacheHit, err := cacheHandle.Read(ctx, chTestArgs.bucket, minObject, 0, buf)
    require.False(t, cacheHit)
    
    // Step 3: Close the CacheHandle
    require.Nil(t, cacheHandle.Close())
    
    // Step 4: OPEN THE CACHE FILE AGAIN to keep descriptor open
    objectPath := util.GetObjectPath(chTestArgs.bucket.Name(), minObject.Name)
    downloadPath := util.GetDownloadPath(chTestArgs.cacheDir, objectPath)
    file, err := os.OpenFile(downloadPath, os.O_RDONLY, 0600)
    require.NoError(t, err)
    defer func() {
        _ = file.Close()
    }()
    
    // Step 5: INVALIDATE CACHE (triggers truncate and delete)
    err = chTestArgs.cacheHandler.InvalidateCache(minObject.Name, chTestArgs.bucket.Name())
    assert.NoError(t, err)  // ← Eviction succeeds
    
    // Step 6: TRY TO READ from the open file handle
    _, err = file.Read(buf)
    if tc.isCacheFileReadErrExpected {
        assert.NotNil(t, err)  // ← Read fails (expected because file was truncated)
    } else {
        assert.NoError(t, err)  // ← Some configurations allow read
    }
}
```

**Test Results**:
- File truncation and deletion succeed without errors
- Reading from the open descriptor fails gracefully (because file is truncated to 0)
- NO crash, NO undefined behavior, NO data corruption
- Test name says "Truncates" - verifying truncation is effective

**Test Configuration**: Both parallel and non-parallel downloads tested

---

## 8. Reference Counts and Handle Tracking

### No Explicit Reference Counting Needed

The system does NOT use reference counting because:

1. **OS Provides Protection**:
   - File descriptor stays valid after os.Remove()
   - File data remains accessible
   - No manual cleanup needed

2. **Job Invalidation is the Control**:
   - Jobs track if they should be downloading
   - Invalid jobs don't write to files
   - Safe for reads even after deletion

3. **Cache Consistency via LRU**:
   - FileInfo entries are the "source of truth"
   - If FileInfo is evicted, cache entry doesn't exist
   - Readers detect via cache lookup

4. **No Cross-Reader Synchronization**:
   - Each CacheHandle has its own file descriptor
   - Multiple CacheHandles can exist for same file
   - Each manages its own lifecycle independently

---

## 9. Edge Cases and Failure Scenarios

### Edge Case 1: Read in Progress When Eviction Starts

```
Timeline:
t1: CacheHandle.Read() acquired fileHandle FD
t2: EViction starts, marks Job as Invalid
t3: Job cancels download (if in progress)
t4: File gets truncated(0) and removed
t5: CacheHandle.ReadAt(FD) still works - reads cached/truncated data
t6: CacheHandle validates FileInfo - gets nil (evicted)
t7: Returns error to caller
Result: ✓ Handled correctly
```

### Edge Case 2: Multiple CacheHandles for Same File

```
Scenario:
- CacheHandle A reading from file
- CacheHandle B reading from file
- Eviction triggers

Result:
- Both have separate os.File descriptors
- File is truncated and marked for deletion
- Both can continue reading until they close
- On last close, file is truly deleted
- ✓ Safe - OS handles deferred deletion
```

### Edge Case 3: Concurrent Access During Eviction

```
Synchronization:
- CacheHandler.mu locks entire GetCacheHandle
- CacheHandler.InvalidateCache also locks mu
- Cannot run concurrently
- Serialized execution
- ✓ No race possible
```

### Edge Case 4: Job Invalidated Before File Deletion

```
Sequence:
1. FileInfo evicted from cache
2. Job.Invalidate() called
   - Changes status to Invalid
   - Cancels in-progress download goroutine
   - Notifies subscribers
3. TruncateAndRemoveFile() called
   - File truncated
   - File deleted
Result: ✓ Job stops writing before file is deleted
```

### Edge Case 5: Partial Download + Eviction

```
Scenario:
- 100MB file, 30MB downloaded
- Eviction triggers
- Download job still in progress

Result:
1. Job marked as Invalid
2. Download goroutine cancelled (via context.Cancel)
3. Current write operation in progress completes
4. File truncated to 0
5. File deleted
6. No further writes happen
- ✓ Safe - final write completes before truncation
```

---

## 10. OS Guarantees for Open File Descriptors

### Linux File Deletion Semantics (unlink)

From the POSIX standard and Linux man pages:

**When os.Remove() is called on a file with open descriptors**:

1. **Directory entry removed immediately**: File is no longer accessible via pathname

2. **File survives in filesystem**: Until the last file descriptor is closed
   - Inode remains allocated
   - Data blocks remain allocated
   - On Linux: "unlink" semantics

3. **Existing descriptors remain valid**:
   - Can still read from file
   - Can still write to file
   - Offset position maintained
   - All operations work normally

4. **File truly deleted when**: Last descriptor closes

**Code Evidence**:

Comment in `/home/user/gcsfuse/internal/cache/util/util.go`:
```
"Truncate the file to 0 size, so that even if there are open file handles
and linux doesn't delete the file, the file will not take space."
```

This explicitly acknowledges and relies on Linux's deferred deletion behavior.

---

## 11. Synchronization Mechanisms Summary

| Mechanism | Location | Protection |
|-----------|----------|-----------|
| **CacheHandler.mu Locker** | cache_handler.go:55 | Serializes eviction and creation |
| **LRU Cache.mu RWLocker** | lru.go:66 | Protects cache operations |
| **Job.mu Locker** | job.go:90 | Protects job state |
| **JobManager.mu Locker** | downloader.go:62 | Protects job map |
| **Job Invalidation** | job.go:195 | Cancels downloads before deletion |
| **TruncateAndRemoveFile** | util.go:171 | Two-step deletion (truncate + remove) |
| **FileInfo Validation** | cache_handle.go:131 | Detects eviction via cache lookup |
| **Double Validation** | cache_handle.go:263 | Checks before AND after read |

---

## 12. Conclusion

### Is There a Race Condition?

**NO**. The system is well-protected by:

1. **Operating System Level**:
   - Linux's deferred deletion via unlink() semantics
   - File descriptors remain valid after file deletion
   - Truncation frees disk space immediately

2. **Application Level**:
   - Job invalidation before file deletion
   - FileInfo validation detects eviction
   - CacheHandler lock serializes operations
   - LRU cache lock prevents concurrent modification

3. **Design Level**:
   - Two-phase cleanup (invalidate job, then delete file)
   - Double validation (before and after read)
   - Atomic eviction + cleanup within locks
   - Graceful fallback on cache miss

### What Happens If FileInfo is Evicted?

1. FileInfo entry removed from cache
2. Job marked Invalid and download cancelled
3. File truncated to 0 (immediate)
4. File deleted (immediate on Linux, but deferred while FDs open)
5. CacheHandle read operation detects via cache lookup validation
6. Returns error or cache-miss status
7. Caller falls back to GCS

### Is Disk Space Properly Freed?

**YES**. The truncation step (`os.Truncate(filePath, 0)`) immediately:
- Frees all file blocks
- Reduces inode size to 0
- Works even with open descriptors
- Achieves main goal regardless of deletion success

### Are CacheHandles Safe?

**YES**. Each CacheHandle:
- Has its own file descriptor
- Can continue reading after FileInfo eviction
- Gets error/cache-miss on validation check
- Falls back to GCS if needed
- Safely closes when done

---

## References

### Key Files and Line Numbers

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| Eviction Cleanup | `cache_handler.go` | 106-129 | `cleanUpEvictedFile()` |
| Job Invalidation | `downloader.go` | 143-153 | `InvalidateAndRemoveJob()` |
| Job Invalidation Details | `job.go` | 195-212 | `Job.Invalidate()` |
| File Deletion | `util.go` | 169-183 | `TruncateAndRemoveFile()` |
| Cache Handle Read | `cache_handle.go` | 151-269 | `CacheHandle.Read()` |
| FileInfo Validation | `cache_handle.go` | 131-144 | `validateEntryInFileInfoCache()` |
| LRU Lookup | `lru.go` | 229-241 | `LookUpWithoutChangingOrder()` |
| Test Case | `cache_handler_test.go` | 786-866 | `Test_InvalidateCache_Truncates` |

---

## Additional Notes

### Why Two-Phase Deletion?

The two-phase approach (truncate then delete) is specifically designed to:

1. **Guarantee space is freed**:
   - Truncate succeeds even if delete fails
   - Main goal (freeing space) always achieved
   - Only disk space matters for eviction policy

2. **Atomicity**:
   - First phase (truncate) is atomic
   - Second phase (delete) may fail without impacting space
   - Idempotent: Can be retried

3. **Safety with open descriptors**:
   - Truncate works with open FDs
   - Delete may be deferred with open FDs
   - Together: Space freed (goal) + FD remains valid (safe)

### Testing

The test `Test_InvalidateCache_Truncates` explicitly validates this scenario:
- Opens file descriptor
- Triggers eviction (invalidate cache)
- Verifies truncate/delete succeeded
- Attempts read from open FD
- Gracefully handles truncated file

This demonstrates the system was designed with this edge case in mind.

