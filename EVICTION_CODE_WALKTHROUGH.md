# GCS Fuse Eviction Flow - Code Walkthrough

This document provides a detailed line-by-line walkthrough of the eviction process with actual code from the repository.

---

## 1. Cache Initialization

**File**: `/home/user/gcsfuse/internal/fs/fs.go` (lines 251-279)

### Code:
```go
func createFileCacheHandler(serverCfg *ServerConfig) (fileCacheHandler *file.CacheHandler, err error) {
	var sizeInBytes uint64
	// -1 means unlimited size for cache, the underlying LRU cache doesn't handle
	// -1 explicitly, hence we pass MaxUint64 as capacity in that case.
	if serverCfg.NewConfig.FileCache.MaxSizeMb == -1 {
		sizeInBytes = math.MaxUint64
	} else {
		sizeInBytes = uint64(serverCfg.NewConfig.FileCache.MaxSizeMb) * cacheutil.MiB
	}
	fileInfoCache := lru.NewCache(sizeInBytes)  // <-- Creates LRU cache with size limit

	cacheDir := string(serverCfg.NewConfig.CacheDir)
	// Adding a new directory inside cacheDir to keep file-cache separate from
	// metadata cache if and when we support storing metadata cache on disk in
	// the future.
	cacheDir = path.Join(cacheDir, cacheutil.FileCache)

	filePerm := cacheutil.DefaultFilePerm
	dirPerm := cacheutil.DefaultDirPerm

	cacheDirErr := cacheutil.CreateCacheDirectoryIfNotPresentAt(cacheDir, dirPerm)
	if cacheDirErr != nil {
		return nil, fmt.Errorf("createFileCacheHandler: while creating file cache directory: %w", cacheDirErr)
	}

	jobManager := downloader.NewJobManager(fileInfoCache, filePerm, dirPerm, cacheDir, serverCfg.SequentialReadSizeMb, &serverCfg.NewConfig.FileCache, serverCfg.MetricHandle)
	fileCacheHandler = file.NewCacheHandler(fileInfoCache, jobManager, cacheDir, filePerm, dirPerm, serverCfg.NewConfig.FileCache.ExcludeRegex, serverCfg.NewConfig.FileCache.IncludeRegex)
	return
}
```

### Key Points:
1. **Size conversion**: `MaxSizeMb * MiB` converts user config to bytes
2. **Special case**: `-1` means unlimited (MaxUint64)
3. **LRU creation**: Single LRU cache created for all files
4. **JobManager**: Created with reference to fileInfoCache
5. **CacheHandler**: Created with fileInfoCache and jobManager

---

## 2. Getting Cache Handle (Trigger Point)

**File**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 230-269)

### Code:
```go
func (chr *CacheHandler) GetCacheHandle(
    object *gcs.MinObject,
    bucket gcs.Bucket,
    cacheForRangeRead bool,
    initialOffset int64) (*CacheHandle, error) {
	
    chr.mu.Lock()  // <-- Serialize cache access
    defer chr.mu.Unlock()

	// Check if file should be excluded from cache
	if chr.shouldExcludeFromCache(bucket, object) {
		return nil, util.ErrFileExcludedFromCacheByRegex
	}

	// If cacheForRangeRead is set to False, initialOffset is non-zero (i.e. random read)
	// and entry for file doesn't already exist in fileInfoCache then no need to
	// create file in cache.
	if !cacheForRangeRead && initialOffset != 0 {
		fileInfoKey := data.FileInfoKey{
			BucketName: bucket.Name(),
			ObjectName: object.Name,
		}
		fileInfoKeyName, err := fileInfoKey.Key()
		if err != nil {
			return nil, fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: while creating key: %v", fileInfoKeyName)
		}

		fileInfo := chr.fileInfoCache.LookUpWithoutChangingOrder(fileInfoKeyName)
		if fileInfo == nil {
			return nil, fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: %w", util.ErrCacheHandleNotRequiredForRandomRead)
		}
	}

	err := chr.addFileInfoEntryAndCreateDownloadJob(object, bucket)  // <-- THIS TRIGGERS EVICTION
	if err != nil {
		return nil, fmt.Errorf("GetCacheHandle: while adding the entry in the cache: %w", err)
	}

	localFileReadHandle, err := chr.createLocalFileReadHandle(object.Name, bucket.Name())
	if err != nil {
		return nil, fmt.Errorf("GetCacheHandle: while creating local-file read handle: %w", err)
	}

	return NewCacheHandle(localFileReadHandle, chr.jobManager.GetJob(object.Name, bucket.Name()), chr.fileInfoCache, cacheForRangeRead, initialOffset), nil
}
```

### Key Points:
1. **Lock**: All cache operations protected by `chr.mu`
2. **Exclusion check**: Regex filters applied first
3. **Random read optimization**: Skip caching if not needed
4. **Key calling point**: `addFileInfoEntryAndCreateDownloadJob()` - this is where eviction happens

---

## 3. Adding Entry (Eviction Trigger)

**File**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 140-217)

### Code:
```go
func (chr *CacheHandler) addFileInfoEntryAndCreateDownloadJob(object *gcs.MinObject, bucket gcs.Bucket) error {
	fileInfoKey := data.FileInfoKey{
		BucketName: bucket.Name(),
		ObjectName: object.Name,
	}
	fileInfoKeyName, err := fileInfoKey.Key()
	if err != nil {
		return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: while creating key: %v", fileInfoKeyName)
	}

	addEntryToCache := false
	fileInfo := chr.fileInfoCache.LookUpWithoutChangingOrder(fileInfoKeyName)  // <-- Check existing
	if fileInfo == nil {
		addEntryToCache = true
	} else {
		// Throw an error, if there is an entry in the file-info cache and cache file doesn't
		// exist locally.
		filePath := util.GetDownloadPath(chr.cacheDir, util.GetObjectPath(bucket.Name(), object.Name))
		_, err := os.Stat(filePath)
		if err != nil && os.IsNotExist(err) {
			return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: %w: %s", util.ErrFileNotPresentInCache, filePath)
		}

		// Evict object in cache if the generation of object in cache is different
		// from the generation of object in inode (we can't compare generations and
		// decide to evict or not because generations are not always increasing:
		// https://cloud.google.com/storage/docs/metadata#generation-number)
		// Also, invalidate the cache if download job has failed or not invalid.
		fileInfoData := fileInfo.(data.FileInfo)
		
		// If offset in file info cache is less than object size and there is no
		// reference to download job then it means the job has failed.
		existingJob := chr.jobManager.GetJob(object.Name, bucket.Name())
		shouldInvalidate := (existingJob == nil) && (fileInfoData.Offset < fileInfoData.FileSize)  // <-- Job failure check
		
		if (!shouldInvalidate) && (existingJob != nil) {
			existingJobStatus := existingJob.GetStatus().Name
			shouldInvalidate = (existingJobStatus == downloader.Failed) || (existingJobStatus == downloader.Invalid)  // <-- Job status check
		}
		
		if (fileInfoData.ObjectGeneration != object.Generation) || shouldInvalidate {  // <-- Generation or job-based eviction
			erasedVal := chr.fileInfoCache.Erase(fileInfoKeyName)  // <-- Manual erase (not LRU-based)
			if erasedVal != nil {
				erasedFileInfo := erasedVal.(data.FileInfo)
				err := chr.cleanUpEvictedFile(&erasedFileInfo)  // <-- CLEANUP
				if err != nil {
					return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: while performing post eviction of %s object error: %w", erasedFileInfo.Key.ObjectName, err)
				}
			}
			addEntryToCache = true
		}
	}

	if addEntryToCache {
		fileInfo = data.FileInfo{
			Key:              fileInfoKey,
			ObjectGeneration: object.Generation,
			Offset:           0,
			FileSize:         object.Size,
		}

		evictedValues, err := chr.fileInfoCache.Insert(fileInfoKeyName, fileInfo)  // <-- LRU-BASED EVICTION HAPPENS HERE
		if err != nil {
			return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: while inserting into the cache: %w", err)
		}
		
		// Create download job for new entry added to cache.
		_ = chr.jobManager.CreateJobIfNotExists(object, bucket)
		
		// PROCESS EVICTED VALUES
		for _, val := range evictedValues {  // <-- Process LRU-evicted entries
			fileInfo := val.(data.FileInfo)
			err := chr.cleanUpEvictedFile(&fileInfo)  // <-- CLEANUP each evicted entry
			if err != nil {
				return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: while performing post eviction of %s object error: %w", fileInfo.Key.ObjectName, err)
			}
		}
	} else {
		// Move this entry on top of LRU.
		_ = chr.fileInfoCache.LookUp(fileInfoKeyName)  // <-- Update LRU order for existing
	}

	return nil
}
```

### Key Points:
1. **Three eviction triggers**:
   - Generation change: `fileInfoData.ObjectGeneration != object.Generation`
   - Job missing: `existingJob == nil && fileInfoData.Offset < fileInfoData.FileSize`
   - Job failed: `existingJobStatus == downloader.Failed || existingJobStatus == downloader.Invalid`

2. **Manual vs LRU eviction**:
   - Manual: `Erase()` for generation/job-based
   - LRU: `Insert()` returns evicted entries

3. **Cleanup**: Called for each evicted entry

---

## 4. LRU Cache Insert (Core Algorithm)

**File**: `/home/user/gcsfuse/internal/cache/lru/lru.go` (lines 148-184)

### Code:
```go
func (c *Cache) Insert(key string, value ValueType) ([]ValueType, error) {
	if value == nil {
		return nil, ErrInvalidEntry
	}

	valueSize := value.Size()
	if valueSize > c.maxSize {  // <-- VALIDATION: Entry larger than cache
		return nil, ErrInvalidEntrySize
	}

	c.mu.Lock()
	defer c.mu.Unlock()

	e, ok := c.index[key]
	if ok {
		// Update an entry if already exist.
		c.currentSize -= e.Value.(entry).Value.Size()
		c.currentSize += valueSize
		e.Value = entry{key, value}
		c.entries.MoveToFront(e)  // <-- Mark as MRU
	} else {
		// Add the entry if already doesn't exist.
		e := c.entries.PushFront(entry{key, value})
		c.index[key] = e
		c.currentSize += valueSize
	}

	var evictedValues []ValueType
	
	// *** EVICTION LOOP - THE CORE ALGORITHM ***
	for c.currentSize > c.maxSize {  // <-- While over limit
		evictedValues = append(evictedValues, c.evictOne())  // <-- Remove LRU entry
	}

	return evictedValues, nil  // <-- Return evicted entries for cleanup
}
```

### Key Points:
1. **Validation**: Entry must be non-nil and fit in cache
2. **Update or insert**: Existing entries moved to front
3. **Eviction loop**: Keeps running until cache size <= maxSize
4. **Evict from back**: `evictOne()` removes least recently used entry
5. **Return values**: Evicted entries returned for cleanup

---

## 5. LRU Eviction Algorithm

**File**: `/home/user/gcsfuse/internal/cache/lru/lru.go` (lines 128-139)

### Code:
```go
func (c *Cache) evictOne() ValueType {
	e := c.entries.Back()  // <-- Get LRU entry (back of list)
	key := e.Value.(entry).Key

	evictedEntry := e.Value.(entry).Value
	c.currentSize -= evictedEntry.Size()  // <-- Update size

	c.entries.Remove(e)  // <-- Remove from linked list
	delete(c.index, key)  // <-- Remove from hash map

	return evictedEntry  // <-- Return for cleanup
}
```

### Key Points:
1. **LRU entry**: At back of doubly-linked list
2. **Size update**: Immediately subtract from currentSize
3. **Two removals**: From linked list AND from hash map
4. **Return value**: FileInfo returned for cleanup

### Data Structure Visualization:
```
MOST RECENTLY USED (Front)                    LEAST RECENTLY USED (Back)
        ↓                                              ↓
    ┌────────┐  ┌────────┐  ┌────────┐      ┌────────┐
    │Entry 1 │↔→│Entry 2 │↔→│Entry 3 │↔→...↔→│Entry N │
    └────────┘  └────────┘  └────────┘      └────────┘
        ↑
    Access updates:
    - Remove from list
    - Re-insert at front
    - MoveToFront()
```

---

## 6. Cleanup - Job Invalidation

**File**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 109-129)

### Code:
```go
func (chr *CacheHandler) cleanUpEvictedFile(fileInfo *data.FileInfo) error {
	key := fileInfo.Key
	_, err := key.Key()
	if err != nil {
		return fmt.Errorf("cleanUpEvictedFile: while creating key: %w", err)
	}

	// STEP 1: Invalidate download job
	chr.jobManager.InvalidateAndRemoveJob(key.ObjectName, key.BucketName)

	// STEP 2: Truncate and delete cache file
	localFilePath := util.GetDownloadPath(chr.cacheDir, util.GetObjectPath(key.BucketName, key.ObjectName))
	err = util.TruncateAndRemoveFile(localFilePath)
	if err != nil {
		if os.IsNotExist(err) {
			logger.Warnf("cleanUpEvictedFile: file was not present at the time of clean up: %v", err)
			return nil  // <-- Treat "not exist" as success
		}
		return fmt.Errorf("cleanUpEvictedFile: error while cleaning up file: %s, error: %w", localFilePath, err)
	}

	return nil
}
```

### Key Points:
1. **Two-step cleanup**:
   - Step 1: Job invalidation
   - Step 2: File truncation and deletion

2. **Error handling**: "File not found" treated as success (idempotent)

---

## 7. Job Invalidation

**File**: `/home/user/gcsfuse/internal/cache/file/downloader/downloader.go` (lines 143-153)

### Code:
```go
func (jm *JobManager) InvalidateAndRemoveJob(objectName string, bucketName string) {
	objectPath := util.GetObjectPath(bucketName, objectName)
	jm.mu.Lock()
	job, ok := jm.jobs[objectPath]
	
	// *** DEADLOCK PREVENTION ***
	// Release the lock while calling downloader.Job.Invalidate to avoid deadlock
	// as the job calls removeJobCallback in the end which requires Lock(jm.mu).
	jm.mu.Unlock()  // <-- UNLOCK BEFORE CALLING Invalidate()
	
	if ok {
		job.Invalidate()  // <-- Calls removeJobCallback() which needs jm.mu
	}
}
```

### Critical Comment:
> Release the lock while calling downloader.Job.Invalidate to avoid deadlock
> as the job calls removeJobCallback in the end which requires Lock(jm.mu).

This is a **critical deadlock prevention pattern**.

---

## 8. Job Status Change

**File**: `/home/user/gcsfuse/internal/cache/file/downloader/job.go` (lines 195-212)

### Code:
```go
func (job *Job) Invalidate() {
	job.mu.Lock()
	if job.status.Name == Downloading {
		job.status.Name = Invalid  // <-- Mark as Invalid
		job.cancel()  // <-- Cancel async download

		// Lock again to execute common notification logic.
		job.mu.Lock()
	}
	defer job.mu.Unlock()
	job.status.Name = Invalid
	logger.Tracef("Job:%p (%s:/%s) is no longer valid.", job, job.bucket.Name(), job.object.Name)
	
	if job.removeJobCallback != nil {
		job.removeJobCallback()  // <-- Removes from JobManager.jobs
		job.removeJobCallback = nil
	}
	job.notifySubscribers()  // <-- Notify waiters
}
```

### Key Points:
1. **Status change**: `Downloading` → `Invalid`
2. **Cancel download**: Terminates async goroutine
3. **Callback execution**: Removes from JobManager map
4. **Notification**: All subscribers notified of invalidation

---

## 9. File Truncation and Deletion

**File**: `/home/user/gcsfuse/internal/cache/util/util.go` (lines 169-183)

### Code:
```go
func TruncateAndRemoveFile(filePath string) error {
	// Truncate the file to 0 size, so that even if there are open file handles
	// and linux doesn't delete the file, the file will not take space.
	err := os.Truncate(filePath, 0)  // <-- FREES DISK SPACE
	if err != nil {
		return err
	}
	err = os.Remove(filePath)  // <-- REMOVES FILE ENTRY
	if err != nil {
		return err
	}
	return nil
}
```

### Key Points:
1. **Two-phase cleanup**:
   - Truncate: Frees disk space immediately
   - Remove: Removes file entry

2. **Why truncate first**: Even if file handles are open or delete fails, space is freed

3. **Linux behavior**: `os.Remove()` is `unlink()` - file deletion deferred if handles open

---

## 10. Complete Eviction Example Sequence

### Scenario:
Cache max: 1024 MiB, current: 850 MiB
Insert new file: 300 MiB (Total would be: 1150 MiB, need to evict 126 MiB)

### Execution Trace:
```
1. CacheHandler.GetCacheHandle() called
   └─ Lock acquired (chr.mu)
   
2. addFileInfoEntryAndCreateDownloadJob() called
   └─ Create FileInfo for 300 MiB file
   
3. fileInfoCache.Insert(key, fileInfo) called
   └─ currentSize = 850 MiB + 300 MiB = 1150 MiB
   └─ maxSize = 1024 MiB
   └─ Need to evict: 1150 - 1024 = 126 MiB
   
4. Eviction loop begins:
   ├─ Iteration 1: evictOne()
   │  └─ Remove entry X (100 MiB) from back
   │  └─ currentSize = 1050 MiB
   │  └─ Add to evictedValues[]
   │
   ├─ Iteration 2: evictOne()
   │  └─ Remove entry Y (30 MiB) from back
   │  └─ currentSize = 1020 MiB
   │  └─ Add to evictedValues[]
   │
   └─ Loop exits (currentSize <= maxSize)
   
5. Insert() returns evictedValues = [X (100 MiB), Y (30 MiB)]

6. Process evicted entries:
   ├─ For entry X:
   │  └─ cleanUpEvictedFile() called
   │  └─ JobManager.InvalidateAndRemoveJob()
   │  └─ Job.Invalidate() changes status to Invalid
   │  └─ TruncateAndRemoveFile() removes local file
   │
   └─ For entry Y:
      └─ cleanUpEvictedFile() called
      └─ JobManager.InvalidateAndRemoveJob()
      └─ Job.Invalidate() changes status to Invalid
      └─ TruncateAndRemoveFile() removes local file

7. Unlock (chr.mu) released

RESULT: Cache state after eviction
├─ 300 MiB (new file)
├─ All remaining files from LRU order (total ≤ 1024 MiB)
└─ Entries X and Y completely cleaned up
```

---

## 11. Edge Case: Concurrent Access During Eviction

**Scenario**: CacheHandle reads while FileInfo is being evicted

### Code Flow:

**Reader Thread**:
```go
// From cache_handle.go - CacheHandle.Read()
fileInfoData, errFileInfo := fch.getFileInfoData(bucket, object, false)
if errFileInfo != nil {
    return 0, false, fmt.Errorf("%w Error in getCachedFileInfo: %v", util.ErrInvalidFileInfoCache, errFileInfo)
}
```

**Eviction Thread** (simultaneously):
```go
// From cache_handler.go - cleanUpEvictedFile()
fileInfo := val.(data.FileInfo)
err := chr.cleanUpEvictedFile(&fileInfo)
```

**Result**:
- Reader's `getFileInfoData()` will NOT find entry (evicted from cache)
- Returns `ErrInvalidFileInfoCache`
- Caller falls back to GCS read
- No data corruption (file handle might still exist)

**Thread Safety**:
- LRU cache has RWLocker
- CacheHandler has Locker
- Job has Locker
- No deadlock due to careful lock release (see Invalidate pattern)

---

## Summary: Key Code Sections

| Operation | File | Lines | Key Code |
|-----------|------|-------|----------|
| **Cache init** | `fs.go` | 255-260 | `lru.NewCache(sizeInBytes)` |
| **Get handle** | `cache_handler.go` | 230-269 | `addFileInfoEntryAndCreateDownloadJob()` |
| **Add entry** | `cache_handler.go` | 140-217 | `fileInfoCache.Insert(key, fileInfo)` |
| **LRU insert** | `lru.go` | 148-184 | Eviction loop: `for c.currentSize > c.maxSize` |
| **Evict LRU** | `lru.go` | 128-139 | `evictOne()` removes from back |
| **Cleanup** | `cache_handler.go` | 109-129 | Two-step: invalidate job + delete file |
| **Job invalidate** | `downloader.go` | 143-153 | Unlock before calling `Invalidate()` |
| **Job status** | `job.go` | 195-212 | Change status to Invalid, cancel download |
| **File cleanup** | `util.go` | 171-183 | Truncate then remove |

