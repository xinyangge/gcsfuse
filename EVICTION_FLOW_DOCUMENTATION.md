# GCS Fuse Eviction Flow Documentation

## Overview

The gcsfuse caching system implements a Least Recently Used (LRU) eviction policy to manage cache size and ensure the system doesn't consume unlimited disk space. This document provides a comprehensive analysis of what gets evicted, what triggers eviction, the step-by-step process, and all involved components.

---

## Table of Contents

1. [What Gets Evicted](#what-gets-evicted)
2. [What Triggers Eviction](#what-triggers-eviction)
3. [Eviction Process - Step by Step](#eviction-process---step-by-step)
4. [Components Involved](#components-involved)
5. [Interaction with FileHandle, CacheHandle, and FileInfo](#interaction-with-filehandle-cachehandle-and-fileinfo)
6. [Cleanup Process](#cleanup-process)
7. [Eviction Policies and Configurations](#eviction-policies-and-configurations)
8. [Edge Cases and Special Handling](#edge-cases-and-special-handling)
9. [Diagrams and Sequences](#diagrams-and-sequences)

---

## What Gets Evicted

### 1. FileInfo Entries (Primary Eviction Target)
- **Type**: `data.FileInfo` structs stored in the LRU cache
- **File Location**: `/home/user/gcsfuse/internal/cache/data/file_info.go`
- **Structure**:
  ```go
  type FileInfo struct {
      Key              FileInfoKey       // Bucket name, object name, bucket creation time
      ObjectGeneration int64             // GCS generation number
      Offset           uint64            // Download progress (bytes downloaded)
      FileSize         uint64            // Total object size
  }
  ```
- **Size Calculation**: File size (ObjectSize) is used to determine cache occupancy

### 2. Cache Files (Disk Resources)
- **Type**: Local files stored in the cache directory
- **Location**: `{CACHE_DIR}/gcsfuse-file-cache/{BUCKET_NAME}/{OBJECT_NAME}`
- **File Permissions**: Read-only (0600) after creation
- **Associated Resources**: Download jobs referencing these files

### 3. Download Jobs
- **Type**: `downloader.Job` structs
- **Lifecycle**: Created on first access, invalidated when evicted
- **State**: Can be in NotStarted, Downloading, Completed, Failed, or Invalid state

### 4. Metadata (Type Cache and Stat Cache)
- **Separate caches**: Also use LRU but are independently configured
- **Not in scope of this file cache eviction document**

---

## What Triggers Eviction

### 1. Cache Size Limit (Primary Trigger)
- **Configuration Parameter**: `file-cache.max-size-mb`
- **File Location**: `/home/user/gcsfuse/cfg/config.go` (line 432)
- **Default**: Typically configured by user
- **Special Value**: `-1` means unlimited cache size (converted to MaxUint64 internally)
- **File Reference**: `/home/user/gcsfuse/internal/fs/fs.go` (lines 255-260)

**Trigger Logic**:
```go
// When inserting into cache:
if serverCfg.NewConfig.FileCache.MaxSizeMb == -1 {
    sizeInBytes = math.MaxUint64  // Unlimited
} else {
    sizeInBytes = uint64(serverCfg.NewConfig.FileCache.MaxSizeMb) * cacheutil.MiB
}
fileInfoCache := lru.NewCache(sizeInBytes)
```

### 2. Object Generation Change
- **Trigger**: Object in GCS has different generation number than cached version
- **File Location**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 163-177)
- **Reason**: GCS can create new versions of objects; old cached versions become stale
- **Note**: Generation numbers don't always increase (see GCS documentation)

**Detection Logic**:
```go
if fileInfoData.ObjectGeneration != object.Generation {
    // Evict old entry, add new one
    erasedVal := chr.fileInfoCache.Erase(fileInfoKeyName)
    addEntryToCache = true
}
```

### 3. Failed or Invalid Download Jobs
- **Trigger**: Download job has failed or was explicitly invalidated
- **Detection**: Job status is `Failed` or `Invalid`
- **File Location**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 172-176)

**Detection Logic**:
```go
shouldInvalidate := (existingJob == nil) && (fileInfoData.Offset < fileInfoData.FileSize)
if (!shouldInvalidate) && (existingJob != nil) {
    existingJobStatus := existingJob.GetStatus().Name
    shouldInvalidate = (existingJobStatus == downloader.Failed) || 
                       (existingJobStatus == downloader.Invalid)
}
```

### 4. Manual Invalidation
- **Trigger**: Explicit call to `CacheHandler.InvalidateCache()`
- **File Location**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 275-297)
- **Use Cases**: File deletion, explicit cache clearing

### 5. Prefix-based Cleanup (Batch Eviction)
- **Trigger**: `EraseEntriesWithGivenPrefix()` called on cache
- **File Location**: `/home/user/gcsfuse/internal/cache/lru/lru.go` (lines 272-278)
- **Use Cases**: Directory deletion (all files under directory prefix removed)

**Function**:
```go
func (c *Cache) EraseEntriesWithGivenPrefix(prefix string) {
    for key := range c.index {
        if strings.HasPrefix(key, prefix) {
            c.Erase(key)
        }
    }
}
```

---

## Eviction Process - Step by Step

### High-Level Flow

```
New File Access Request
         ↓
Check Cache Size + New Object Size
         ↓
    ┌─ If Size OK: Just insert
    │
    └─ If Size Exceeded:
         ↓
    Evict LRU entries one by one
    until cache is at or below limit
         ↓
    Insert new entry
```

### Detailed Step-by-Step Process

#### **Step 1: Determine Need for Eviction**
- **Location**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 230-269)
- **File**: `CacheHandler.GetCacheHandle()`
- **Code**:
```go
func (chr *CacheHandler) GetCacheHandle(
    object *gcs.MinObject, 
    bucket gcs.Bucket, 
    cacheForRangeRead bool, 
    initialOffset int64) (*CacheHandle, error) {
    
    chr.mu.Lock()
    defer chr.mu.Unlock()
    
    // Check if file should be excluded from cache
    if chr.shouldExcludeFromCache(bucket, object) {
        return nil, util.ErrFileExcludedFromCacheByRegex
    }
    
    // Add entry (may trigger eviction)
    err := chr.addFileInfoEntryAndCreateDownloadJob(object, bucket)
    // ... continue
}
```

#### **Step 2: Prepare FileInfo Entry**
- **Location**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 140-217)
- **File**: `CacheHandler.addFileInfoEntryAndCreateDownloadJob()`
- **Actions**:
  1. Create `FileInfoKey` from bucket and object name
  2. Check if entry already exists in cache
  3. Validate generation and job status
  4. Prepare new `FileInfo` struct

**Key Code Section**:
```go
fileInfo = data.FileInfo{
    Key:              fileInfoKey,
    ObjectGeneration: object.Generation,
    Offset:           0,
    FileSize:         object.Size,
}

evictedValues, err := chr.fileInfoCache.Insert(fileInfoKeyName, fileInfo)
```

#### **Step 3: Insert into LRU Cache and Trigger Eviction**
- **Location**: `/home/user/gcsfuse/internal/cache/lru/lru.go` (lines 148-184)
- **File**: `Cache.Insert()`
- **Algorithm**:
  1. Check if value is non-nil
  2. Check if value size <= maxSize
  3. If key exists: update entry and move to front
  4. If new key: add to front of linked list
  5. Add to currentSize
  6. **Evict LRU entries until currentSize <= maxSize**

**Eviction Logic**:
```go
func (c *Cache) Insert(key string, value ValueType) ([]ValueType, error) {
    // Validation
    if value == nil {
        return nil, ErrInvalidEntry
    }
    if valueSize > c.maxSize {
        return nil, ErrInvalidEntrySize
    }
    
    c.mu.Lock()
    defer c.mu.Unlock()
    
    // Update or insert
    e, ok := c.index[key]
    if ok {
        // Update existing
        c.currentSize -= e.Value.(entry).Value.Size()
        c.currentSize += valueSize
        e.Value = entry{key, value}
        c.entries.MoveToFront(e)  // Mark as most recently used
    } else {
        // New entry
        e := c.entries.PushFront(entry{key, value})
        c.index[key] = e
        c.currentSize += valueSize
    }
    
    var evictedValues []ValueType
    
    // *** EVICTION HAPPENS HERE ***
    // Evict LRU entries from back of list
    for c.currentSize > c.maxSize {
        evictedValues = append(evictedValues, c.evictOne())
    }
    
    return evictedValues, nil
}

// evictOne removes least recently used entry
func (c *Cache) evictOne() ValueType {
    e := c.entries.Back()  // Get LRU entry
    key := e.Value.(entry).Key
    
    evictedEntry := e.Value.(entry).Value
    c.currentSize -= evictedEntry.Size()  // Update size
    
    c.entries.Remove(e)   // Remove from linked list
    delete(c.index, key)  // Remove from index
    
    return evictedEntry
}
```

#### **Step 4: Process Evicted Entries**
- **Location**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 204-210)
- **File**: `CacheHandler.addFileInfoEntryAndCreateDownloadJob()`
- **Actions**: For each evicted FileInfo:

```go
for _, val := range evictedValues {
    fileInfo := val.(data.FileInfo)
    err := chr.cleanUpEvictedFile(&fileInfo)
    if err != nil {
        return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: while performing post eviction of %s object error: %w", 
            fileInfo.Key.ObjectName, err)
    }
}
```

#### **Step 5: Clean Up Evicted File Resources**
- **Location**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 109-129)
- **File**: `CacheHandler.cleanUpEvictedFile()`
- **Actions**:
  1. Get object path from FileInfoKey
  2. Invalidate and remove download job from JobManager
  3. Truncate local cache file to 0 bytes
  4. Delete local cache file

**Code**:
```go
func (chr *CacheHandler) cleanUpEvictedFile(fileInfo *data.FileInfo) error {
    key := fileInfo.Key
    _, err := key.Key()
    if err != nil {
        return fmt.Errorf("cleanUpEvictedFile: while creating key: %w", err)
    }
    
    // 1. Invalidate download job
    chr.jobManager.InvalidateAndRemoveJob(key.ObjectName, key.BucketName)
    
    // 2. Truncate and delete cache file
    localFilePath := util.GetDownloadPath(chr.cacheDir, 
        util.GetObjectPath(key.BucketName, key.ObjectName))
    err = util.TruncateAndRemoveFile(localFilePath)
    if err != nil {
        if os.IsNotExist(err) {
            logger.Warnf("cleanUpEvictedFile: file was not present at the time of clean up: %v", err)
            return nil
        }
        return fmt.Errorf("cleanUpEvictedFile: error while cleaning up file: %s, error: %w", 
            localFilePath, err)
    }
    
    return nil
}
```

#### **Step 6: Invalidate Download Job**
- **Location**: `/home/user/gcsfuse/internal/cache/file/downloader/downloader.go` (lines 143-153)
- **File**: `JobManager.InvalidateAndRemoveJob()`
- **Code**:
```go
func (jm *JobManager) InvalidateAndRemoveJob(objectName string, bucketName string) {
    objectPath := util.GetObjectPath(bucketName, objectName)
    jm.mu.Lock()
    job, ok := jm.jobs[objectPath]
    jm.mu.Unlock()  // Release before calling Invalidate to avoid deadlock
    if ok {
        job.Invalidate()
    }
}
```

#### **Step 7: Job Invalidation**
- **Location**: `/home/user/gcsfuse/internal/cache/file/downloader/job.go` (lines 195-212)
- **File**: `Job.Invalidate()`
- **Actions**:
  1. Change job status to "Invalid"
  2. Cancel in-progress download goroutine (if running)
  3. Call removeJobCallback to remove from JobManager
  4. Notify all subscribers

**Code**:
```go
func (job *Job) Invalidate() {
    job.mu.Lock()
    if job.status.Name == Downloading {
        job.status.Name = Invalid
        job.cancel()  // Cancel async download
        
        // Lock again to execute common notification logic
        job.mu.Lock()
    }
    defer job.mu.Unlock()
    job.status.Name = Invalid
    logger.Tracef("Job:%p (%s:/%s) is no longer valid.", job, job.bucket.Name(), job.object.Name)
    
    if job.removeJobCallback != nil {
        job.removeJobCallback()
        job.removeJobCallback = nil
    }
    job.notifySubscribers()
}
```

#### **Step 8: File Truncation and Deletion**
- **Location**: `/home/user/gcsfuse/internal/cache/util/util.go` (lines 169-183)
- **File**: `TruncateAndRemoveFile()`
- **Purpose**: Even if file handles are open, truncation releases disk space
- **Code**:
```go
func TruncateAndRemoveFile(filePath string) error {
    // First truncate to free disk space even with open handles
    err := os.Truncate(filePath, 0)
    if err != nil {
        return err
    }
    
    // Then delete the file
    err = os.Remove(filePath)
    if err != nil {
        return err
    }
    return nil
}
```

---

## Components Involved

### 1. LRU Cache (`internal/cache/lru/lru.go`)
- **Purpose**: Maintains in-memory cache with LRU eviction policy
- **Key Methods**:
  - `NewCache(maxSize)` - Create cache with size limit
  - `Insert(key, value)` - Add/update entry, returns evicted values
  - `LookUp(key)` - Retrieve entry, marks as most recently used
  - `LookUpWithoutChangingOrder(key)` - Retrieve without changing order
  - `Erase(key)` - Remove entry
  - `EraseEntriesWithGivenPrefix(prefix)` - Batch removal
- **Data Structure**: Doubly-linked list + hash map for O(1) operations

**Key Files**:
- `/home/user/gcsfuse/internal/cache/lru/lru.go` (279 lines)
- `/home/user/gcsfuse/internal/cache/lru/lru_test.go` (tests)

### 2. Cache Handler (`internal/cache/file/cache_handler.go`)
- **Purpose**: Manages FileInfo entries and coordinates eviction cleanup
- **Key Methods**:
  - `GetCacheHandle()` - Get or create cache handle
  - `InvalidateCache()` - Manually invalidate cache entry
  - `cleanUpEvictedFile()` - Perform cleanup for evicted entries
  - `addFileInfoEntryAndCreateDownloadJob()` - Add entry to cache
  - `shouldExcludeFromCache()` - Check regex filters
- **Responsibilities**:
  - Detect generation changes
  - Detect failed jobs
  - Trigger cleanup on eviction

**Key Files**:
- `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (334 lines)
- `/home/user/gcsfuse/internal/cache/file/cache_handler_test.go`

### 3. Cache Handle (`internal/cache/file/cache_handle.go`)
- **Purpose**: Provides read interface to cached file
- **Key Methods**:
  - `Read()` - Read from cache
  - `Close()` - Close file handle
- **Lifecycle**: Created with reference to FileInfo cache, can detect eviction

**Key Files**:
- `/home/user/gcsfuse/internal/cache/file/cache_handle.go` (301 lines)

### 4. Download Job Manager (`internal/cache/file/downloader/downloader.go`)
- **Purpose**: Manages lifecycle of download jobs
- **Key Methods**:
  - `NewJobManager()` - Create job manager
  - `CreateJobIfNotExists()` - Create or get job
  - `GetJob()` - Retrieve job
  - `InvalidateAndRemoveJob()` - Invalidate job
  - `Destroy()` - Clean up all jobs on unmount

**Key Files**:
- `/home/user/gcsfuse/internal/cache/file/downloader/downloader.go` (172 lines)

### 5. Download Job (`internal/cache/file/downloader/job.go`)
- **Purpose**: Manages async download of GCS object
- **Key Methods**:
  - `NewJob()` - Create job
  - `Invalidate()` - Cancel job
  - `Download()` - Request download
  - `GetStatus()` - Get current status
- **States**: NotStarted, Downloading, Completed, Failed, Invalid

**Key Files**:
- `/home/user/gcsfuse/internal/cache/file/downloader/job.go` (500+ lines)

### 6. FileInfo Data Structure (`internal/cache/data/file_info.go`)
- **Purpose**: Stores cache metadata
- **Fields**:
  - `Key`: FileInfoKey (bucket, object, bucket creation time)
  - `ObjectGeneration`: GCS generation number
  - `Offset`: Download progress
  - `FileSize`: Total file size
- **Size Method**: Returns FileSize for LRU size calculation

**Key Files**:
- `/home/user/gcsfuse/internal/cache/data/file_info.go` (62 lines)

### 7. File System (`internal/fs/fs.go`)
- **Purpose**: Creates and manages cache handler and job manager
- **Key Functions**:
  - `createFileCacheHandler()` - Initialize cache
  - Unmount cleanup
- **Configuration**: Reads MaxSizeMb and other settings

**Key Files**:
- `/home/user/gcsfuse/internal/fs/fs.go` (lines 251-279)

---

## Interaction with FileHandle, CacheHandle, and FileInfo

### FileInfo Lifecycle

```
┌─────────────────────────────────────────┐
│           FileInfo Lifecycle            │
└─────────────────────────────────────────┘

1. CREATION
   ├─ Object accessed via CacheHandle
   ├─ CacheHandler creates FileInfoKey
   ├─ FileInfo struct initialized with:
   │  ├─ ObjectGeneration from GCS object
   │  ├─ Offset = 0 (download not started)
   │  └─ FileSize from object.Size
   └─ Inserted into LRU cache

2. UPDATE (During Download)
   ├─ Job.updateStatusOffset() called
   ├─ FileInfo.Offset updated with downloaded bytes
   ├─ Update via fileInfoCache.UpdateWithoutChangingOrder()
   └─ LRU order preserved (no MRU change)

3. ACCESS (During Read)
   ├─ CacheHandle.Read() called
   ├─ FileInfo retrieved via LookUp() or LookUpWithoutChangingOrder()
   ├─ If LookUp(): entry moves to front (marked as MRU)
   ├─ Generation checked against current object
   └─ Offset checked against required read offset

4. EVICTION (LRU + Triggers)
   ├─ LRU: Entry becomes least recently used
   ├─ Size: Cache size exceeds MaxSizeMb
   ├─ Generation: Object has new generation
   ├─ Job: Job failed or invalidated
   ├─ Manual: InvalidateCache() called
   └─ Prefix: EraseEntriesWithGivenPrefix() called

5. CLEANUP
   ├─ cleanUpEvictedFile() called
   ├─ Job.Invalidate() called
   ├─ Local cache file truncated and deleted
   └─ FileInfo removed from LRU cache
```

### CacheHandle Interaction

```
┌─────────────────────────────────────────┐
│       CacheHandle Interaction           │
└─────────────────────────────────────────┘

CacheHandle Fields:
├─ fileHandle: *os.File           // Local cache file handle
├─ fileDownloadJob: *Job          // Reference to download job
├─ fileInfoCache: *lru.Cache      // Reference to FileInfo cache
└─ Various metadata for tracking reads

During Read:
1. CacheHandle.Read() called with offset
2. getFileInfoData() retrieves FileInfo from cache
   ├─ Checks if entry still in cache
   ├─ Detects if evicted (generation mismatch, not found)
   └─ Returns error if cache entry invalid
3. validateEntryInFileInfoCache() checks:
   ├─ Entry exists with matching generation
   ├─ Download offset >= required offset
   └─ Updates LRU order on successful validation
4. If validation fails:
   ├─ ErrInvalidFileInfoCache returned
   ├─ CacheHandle must be closed
   └─ New CacheHandle created on retry

Edge Case - Eviction During Read:
1. CacheHandle.Read() in progress
2. FileInfo entry evicted from cache
3. Next validation check fails
4. Returns ErrInvalidFileInfoCache
5. Read operation falls back to GCS
```

### FileHandle Lifecycle

```
┌─────────────────────────────────────────┐
│        FileHandle Lifecycle             │
└─────────────────────────────────────────┘

1. File opened from user perspective
   ├─ Creates inode.FileInode
   └─ Associated with one or more file handles

2. Read requested
   ├─ FileSystem calls Open
   ├─ CacheHandler.GetCacheHandle() creates cache handle
   ├─ Local file opened via createLocalFileReadHandle()
   └─ *os.File returned in CacheHandle

3. During sequential read
   ├─ CacheHandle tracks read pattern
   ├─ Determines if sequential or random read
   └─ Download job starts for sequential reads

4. During eviction
   ├─ Local file truncated and deleted
   ├─ *os.File still may have open references
   ├─ Truncation frees disk space immediately
   └─ File delete completes when references close

5. CacheHandle close
   ├─ Closes *os.File
   ├─ Does NOT invalidate FileInfo
   ├─ Download job continues if running
   └─ New CacheHandle can be created later
```

---

## Cleanup Process

### Detailed Cleanup Sequence

```
┌──────────────────────────────────────────────────────────────┐
│         Eviction-Triggered Cleanup Sequence                 │
└──────────────────────────────────────────────────────────────┘

Step 1: LRU Cache eviction triggered
   └─> Cache.evictOne() called
       ├─ Removes entry from linked list (back)
       ├─ Removes entry from hash map
       ├─ Returns FileInfo value
       └─ Returns to CacheHandler

Step 2: FileInfo cleanup initiated
   └─> CacheHandler receives evicted FileInfo
       ├─ Extracts FileInfoKey
       ├─ Logs eviction
       └─ Calls cleanUpEvictedFile()

Step 3: Job invalidation
   └─> JobManager.InvalidateAndRemoveJob() called
       ├─ Gets job reference
       ├─ Calls Job.Invalidate()
       │  ├─ Changes status to "Invalid"
       │  ├─ Cancels any in-progress download
       │  ├─ Calls removeJobCallback()
       │  │  └─> JobManager removes from jobs map
       │  └─ Notifies all subscribers
       └─ Job removed from JobManager.jobs

Step 4: File system cleanup
   └─> TruncateAndRemoveFile() called
       ├─ os.Truncate(path, 0)
       │  ├─ Frees disk space immediately
       │  ├─ Works even with open file handles
       │  └─ File now appears empty
       └─ os.Remove(path)
          ├─ On Linux: removes directory entry
          ├─ File still exists if handles open (unlink)
          ├─ File fully deleted when handles close
          └─ Cache directory remains

Step 5: Completion
   └─ FileInfo completely cleaned up
      ├─ No entry in LRU cache
      ├─ Job invalidated and removed
      ├─ Local file deleted or truncated
      └─ Resources released
```

### Cleanup Error Handling

```
Errors During Cleanup (from cache_handler.go lines 106-129):

1. File not present
   ├─ Logged as warning
   ├─ Treated as success
   └─ Cleanup continues

2. Stat error
   ├─ Returned as error
   └─ Cleanup fails, logged

3. Truncate error
   ├─ Returned as error
   └─ Cleanup fails

4. Remove error (after truncate)
   ├─ Returned as error
   └─ Cleanup fails
   └─ Note: Disk space already freed by truncate

Error Handling Philosophy:
- If file already gone: OK (idempotent)
- If truncate succeeds: Space freed (main goal achieved)
- If remove fails: File still inaccessible to cache
```

### Graceful Shutdown Cleanup

```
FileSystem.Destroy() called (line 1650):
├─> CacheHandler.Destroy()
│   └─> JobManager.Destroy()
│       ├─ Gets all jobs
│       ├─ For each job: Job.Invalidate()
│       │  ├─ Cancels download
│       │  ├─ Calls removeJobCallback()
│       │  └─ Notifies subscribers
│       └─ All jobs cleaned up
│
└─> FileInfo cache (in-memory)
    └─ Not explicitly destroyed
       (will be garbage collected)
```

---

## Eviction Policies and Configurations

### 1. LRU (Least Recently Used) Policy

**Algorithm**:
- Most recently accessed entry: Front of linked list
- Least recently used entry: Back of linked list
- On access: Move entry to front (via MoveToFront)
- On eviction: Remove from back (via evictOne)

**Access Patterns**:
- Explicit access via `LookUp()`: Moves to front
- Non-blocking access via `LookUpWithoutChangingOrder()`: No movement
- Update via `UpdateWithoutChangingOrder()`: No movement (used during download progress)

**Implementation**:
- Data structure: Doubly-linked list + hash map
- Time complexity: O(1) for all operations
- File location: `/home/user/gcsfuse/internal/cache/lru/lru.go`

### 2. Size-Based Configuration

**Parameter**: `file-cache.max-size-mb`
**Location**: `/home/user/gcsfuse/cfg/config.go` line 432
**Type**: `int64`

**Semantics**:
- Value > 0: Cache size limit in MiB
- Value = -1: Unlimited cache (converted to MaxUint64)
- Value = 0: Invalid (would have maxSize = 0)

**Usage**:
```go
// From fs.go lines 255-260
if serverCfg.NewConfig.FileCache.MaxSizeMb == -1 {
    sizeInBytes = math.MaxUint64
} else {
    sizeInBytes = uint64(serverCfg.NewConfig.FileCache.MaxSizeMb) * cacheutil.MiB
}
fileInfoCache := lru.NewCache(sizeInBytes)
```

**Example**:
- Config: `max-size-mb: 1024`
- Result: Cache size limit = 1024 * 1024 * 1024 = 1 GiB
- When FileInfo entries total > 1 GiB: LRU eviction triggered

### 3. Generation-Based Policy

**Trigger**: Object generation changes in GCS
**File**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` lines 163-177

**Logic**:
```go
if fileInfoData.ObjectGeneration != object.Generation {
    // Evict and re-cache
}
```

**Rationale**:
- GCS can create new versions of objects
- Old cached data becomes invalid
- Must re-download from new generation

**Important Note** (from source):
```
// Evict object in cache if the generation of object in cache is different
// from the generation of object in inode (we can't compare generations and
// decide to evict or not because generations are not always increasing:
// https://cloud.google.com/storage/docs/metadata#generation-number)
```

### 4. Job Status-Based Policy

**Trigger**: Download job failed or invalidated
**File**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` lines 172-176

**Logic**:
```go
shouldInvalidate := (existingJob == nil) && (fileInfoData.Offset < fileInfoData.FileSize)
if (!shouldInvalidate) && (existingJob != nil) {
    existingJobStatus := existingJob.GetStatus().Name
    shouldInvalidate = (existingJobStatus == downloader.Failed) || 
                       (existingJobStatus == downloader.Invalid)
}
```

**Detection**:
1. Job missing AND partial download: Indicates job failed
2. Job exists AND status is Failed/Invalid: Explicit failure

### 5. Filter-Based Policy (Include/Exclude Regex)

**Parameters**:
- `file-cache.exclude-regex`: Pattern to exclude from cache
- `file-cache.include-regex`: Pattern to include in cache

**File**: `/home/user/gcsfuse/internal/cache/file/cache_handler.go` lines 314-333

**Logic**:
```go
func (chr *CacheHandler) shouldExcludeFromCache(bucket gcs.Bucket, object *gcs.MinObject) bool {
    if chr.includeRegex == nil && chr.excludeRegex == nil {
        return false  // No filters, cache everything
    }
    
    objectName := path.Join(bucket.Name(), bucket.GCSName(object))
    
    // Exclude if matches exclude pattern
    if chr.excludeRegex != nil && chr.excludeRegex.MatchString(objectName) {
        return true
    }
    
    // Exclude if include pattern exists and doesn't match
    if chr.includeRegex != nil && !chr.includeRegex.MatchString(objectName) {
        return true
    }
    
    return false
}
```

**Precedence**: Exclude takes precedence over Include

### 6. Batch Eviction (Prefix-Based)

**Method**: `EraseEntriesWithGivenPrefix()`
**File**: `/home/user/gcsfuse/internal/cache/lru/lru.go` lines 272-278

**Use Case**: Directory deletion (all objects under directory evicted)

**Implementation**:
```go
func (c *Cache) EraseEntriesWithGivenPrefix(prefix string) {
    for key := range c.index {
        if strings.HasPrefix(key, prefix) {
            c.Erase(key)
        }
    }
}
```

### 7. Download-Related Configurations

**Other relevant config parameters**:
- `file-cache.cache-file-for-range-read`: Whether to cache range reads
- `file-cache.download-chunk-size-mb`: Size of each download request
- `file-cache.max-parallel-downloads`: Concurrent download limit

---

## Edge Cases and Special Handling

### 1. File Handle Remains Open During Eviction

**Scenario**: CacheHandle has open file handle, FileInfo is evicted

**Behavior**:
```go
// From util.go - TruncateAndRemoveFile()
err := os.Truncate(filePath, 0)  // Frees space immediately
err = os.Remove(filePath)         // May defer if handles open
```

**Result**:
- Disk space immediately freed
- File appears empty to cache system
- Existing file handle still valid for reading
- File fully deleted when handle closes

**Handling in CacheHandle.Read()**:
```go
// From cache_handle.go lines 131-144
err = fch.validateEntryInFileInfoCache(bucket, object, requiredOffset, changeCacheOrder)
if err != nil {
    return 0, false, err  // Detects eviction via entry absence
}
```

### 2. Concurrent Access During Eviction

**Synchronization**:
- `LRU cache`: Protected by RWLocker (`mu`)
- `CacheHandler`: Protected by Locker (`mu`) 
- `JobManager`: Protected by Locker (`mu`)
- `Job`: Protected by Locker (`mu`) with invariant checking

**Deadlock Prevention** (from downloader.go lines 147-149):
```go
func (jm *JobManager) InvalidateAndRemoveJob(objectName string, bucketName string) {
    jm.mu.Lock()
    job, ok := jm.jobs[objectPath]
    // Release the lock while calling downloader.Job.Invalidate to avoid deadlock
    // as the job calls removeJobCallback in the end which requires Lock(jm.mu).
    jm.mu.Unlock()
    if ok {
        job.Invalidate()
    }
}
```

### 3. Entry Larger Than Cache Capacity

**Scenario**: FileInfo.FileSize > cache.maxSize

**Handling** (from lru.go line 156):
```go
if valueSize > c.maxSize {
    return nil, ErrInvalidEntrySize
}
```

**Result**: Entry rejected, error returned

### 4. Multiple Rapid Evictions

**Scenario**: Very large file added when cache nearly full

**Behavior** (from lru.go lines 179-181):
```go
var evictedValues []ValueType
for c.currentSize > c.maxSize {
    evictedValues = append(evictedValues, c.evictOne())
}
```

**Result**:
- Multiple entries evicted in loop
- Each eviction triggers cleanup
- Test case: `TestMultipleEviction()` in lru_test.go

### 5. Generation Change Race Condition

**Scenario**: Object generation changed between checks

**Handling** (from cache_handler.go lines 151-188):
```go
fileInfo := chr.fileInfoCache.LookUpWithoutChangingOrder(fileInfoKeyName)
if fileInfo == nil {
    addEntryToCache = true
} else {
    // Check file still exists
    _, err := os.Stat(filePath)
    if err != nil && os.IsNotExist(err) {
        return fmt.Errorf("addFileInfoEntryAndCreateDownloadJob: %w: %s", 
            util.ErrFileNotPresentInCache, filePath)
    }
    
    // Check generation
    fileInfoData := fileInfo.(data.FileInfo)
    if fileInfoData.ObjectGeneration != object.Generation {
        // Evict old entry, add new one
        erasedVal := chr.fileInfoCache.Erase(fileInfoKeyName)
        if erasedVal != nil {
            // ... cleanup
        }
        addEntryToCache = true
    }
}
```

### 6. Failed Download Job Recovery

**Scenario**: Download job fails partway through

**Detection** (from cache_handler.go lines 172-176):
```go
shouldInvalidate := (existingJob == nil) && (fileInfoData.Offset < fileInfoData.FileSize)
if (!shouldInvalidate) && (existingJob != nil) {
    existingJobStatus := existingJob.GetStatus().Name
    shouldInvalidate = (existingJobStatus == downloader.Failed) || 
                       (existingJobStatus == downloader.Invalid)
}
```

**Recovery**:
1. Old entry evicted
2. FileInfo removed from cache
3. Local file truncated and deleted
4. On next access: New job created, fresh download starts

### 7. Prefix Eviction with Partial Cache

**Scenario**: Directory deleted, some files already evicted by LRU

**Behavior** (from lru_test.go lines 199-214):
```go
// Test: TestEraseCacheWithGivenPrefixWithSomeEntriesEvictedDueToCacheSize
t.cache.EraseEntriesWithGivenPrefix("a")
// Result: Only entries still in cache are removed
```

**Implementation**:
```go
func (c *Cache) EraseEntriesWithGivenPrefix(prefix string) {
    for key := range c.index {
        if strings.HasPrefix(key, prefix) {
            c.Erase(key)
        }
    }
}
```

**Result**: Only entries currently in cache are removed; already-evicted entries unaffected

### 8. Unmount with Active Downloads

**Scenario**: FileSystem.Destroy() called while downloads in progress

**Handling** (from downloader.go lines 155-171):
```go
func (jm *JobManager) Destroy() {
    jm.mu.Lock()
    jobs := make([]*Job, 0, len(jm.jobs))
    for _, job := range jm.jobs {
        jobs = append(jobs, job)
    }
    jm.mu.Unlock()
    
    for _, job := range jobs {
        job.Invalidate()  // Cancels download goroutine
    }
}
```

**Result**:
- All jobs invalidated
- All downloads cancelled
- All goroutines terminate
- Clean shutdown

### 9. CRC Validation (Optional)

**Configuration**: `file-cache.enable-crc`

**Purpose**: Verify downloaded data integrity

**Note**: CRC is calculated but not used for eviction decisions in current implementation

### 10. O_DIRECT Flag (Optional)

**Configuration**: `file-cache.enable-o-direct`

**Purpose**: Bypass kernel page cache for direct I/O

**Impact**: Reduces memory pressure, affects eviction indirectly

---

## Diagrams and Sequences

### Diagram 1: Cache Data Structures

```
┌─────────────────────────────────────────────────────────┐
│                    LRU Cache                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─ Linked List (Doubly-Linked) ─────────────────────┐ │
│  │                                                    │ │
│  │  [MRU]                                      [LRU] │ │
│  │    ▲                                          ▲   │ │
│  │    │                                          │   │ │
│  │  ┌─────┐  ┌─────┐  ┌─────┐      ┌─────┐   │ │ │
│  │  │Entry│─→│Entry│─→│Entry│─→...─→│Entry│─┘ │ │
│  │  └─────┘  └─────┘  └─────┘      └─────┘     │ │
│  │    │        │        │            │          │ │
│  │    ▼        ▼        ▼            ▼          │ │
│  │  Key1    Key2      Key3          KeyN        │ │
│  │                                              │ │
│  └──────────────────────────────────────────────┘ │
│                        │                          │
│                        │ Indexed by                │
│                        ▼                          │
│  ┌─ Hash Map ────────────────────────────────────┐ │
│  │                                               │ │
│  │  Key1 ──────→ [pointer to entry]              │ │
│  │  Key2 ──────→ [pointer to entry]              │ │
│  │  Key3 ──────→ [pointer to entry]              │ │
│  │  ...                                          │ │
│  │  KeyN ──────→ [pointer to entry]              │ │
│  │                                               │ │
│  └───────────────────────────────────────────────┘ │
│                                                    │
│  currentSize: 854 MiB / maxSize: 1024 MiB         │
│  entries: 127 (LRU count)                         │
│                                                    │
└─────────────────────────────────────────────────────┘
```

### Diagram 2: FileInfo Cache Structure

```
┌─────────────────────────────────────────────────────────┐
│              FileInfo Cache Structure                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Each Entry contains FileInfo with:                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ FileInfo {                                      │   │
│  │   Key: FileInfoKey {                           │   │
│  │     BucketName: "my-bucket"                    │   │
│  │     ObjectName: "path/to/file.txt"             │   │
│  │     BucketCreationTime: 2023-01-01 12:00:00   │   │
│  │   }                                             │   │
│  │   ObjectGeneration: 1672534800                 │   │
│  │   Offset: 524288000 (500 MB downloaded)        │   │
│  │   FileSize: 1048576000 (1 GB total)            │   │
│  │ }                                               │   │
│  │                                                 │   │
│  │ Size() → Returns FileSize (1 GB)               │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Key Format: "{BucketName}{BucketCreationTimeUnix}     │
│              {ObjectName}"                            │
│                                                         │
│  Example: "my-bucket1672534800path/to/file.txt"       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Diagram 3: Eviction Flow Sequence

```
┌─────────────────────────────────────────────────────────────┐
│            Eviction Flow - Sequence Diagram               │
└─────────────────────────────────────────────────────────────┘

User Application
  │
  ├─→ Read from file
  │
  └─→ FileSystem.Read()
      │
      └─→ inode.Read()
          │
          └─→ CacheHandler.GetCacheHandle()
              │
              ├─ Check shouldExcludeFromCache() [line 234-237]
              │
              ├─ Check cache size [line 242-256]
              │
              └─→ addFileInfoEntryAndCreateDownloadJob() [line 258]
                  │
                  ├─ Create FileInfoKey [line 141-144]
                  │
                  ├─ Look up existing entry [line 151]
                  │
                  ├─ If entry exists:
                  │  ├─ Check generation [line 177]
                  │  ├─ Check job status [line 172-176]
                  │  └─ If mismatch: Erase old entry [line 178]
                  │
                  ├─ Create FileInfo [line 191-196]
                  │
                  └─→ fileInfoCache.Insert(key, fileInfo) [line 198]
                      │
                      ├─ Validation checks [line 151-158]
                      │
                      ├─ Update or add entry [line 163-175]
                      │
                      └─ EVICTION LOOP [line 179-181]:
                         │
                         └─→ While cache.currentSize > cache.maxSize:
                             │
                             └─→ Cache.evictOne() [line 128-139]
                                 │
                                 ├─ Get LRU entry from back [line 129]
                                 ├─ Update currentSize [line 133]
                                 ├─ Remove from linked list [line 135]
                                 ├─ Remove from hash map [line 136]
                                 └─ Return FileInfo ← Returns to Insert()
                                     │
                                     └─ Collected in evictedValues []
                      │
                      └─ Return evictedValues [line 183]
                          │
                          └─ Back to addFileInfoEntryAndCreateDownloadJob()
                              │
                              └─→ Cleanup loop [line 204-210]
                                  │
                                  └─→ For each evicted FileInfo:
                                      │
                                      └─→ cleanUpEvictedFile() [line 206]
                                          │
                                          ├─ jobManager.InvalidateAndRemoveJob()
                                          │  [line 116]
                                          │  │
                                          │  └─→ Job.Invalidate() [line 151]
                                          │      │
                                          │      ├─ Change status to Invalid
                                          │      ├─ Cancel async download
                                          │      ├─ Call removeJobCallback()
                                          │      │   └─ Remove from jobs map
                                          │      └─ Notify subscribers
                                          │
                                          └─ TruncateAndRemoveFile(path)
                                             [line 119]
                                             │
                                             ├─ os.Truncate(path, 0)
                                             │  └─ Frees disk space
                                             └─ os.Remove(path)
                                                └─ Deletes file entry
                              │
                              └─ Return to caller

  ← Back to User Application
```

### Diagram 4: Component Interaction Map

```
┌───────────────────────────────────────────────────────────┐
│           Component Interaction Map                      │
└───────────────────────────────────────────────────────────┘

                      FileSystem
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   CacheHandler    JobManager        FileSystem
      │                 │
      ├─────┬───────────┤
      │     │           │
      ▼     ▼           ▼
    LRU   File      Download
   Cache  System      Job
    │      │           │
    │      ▼           │
    │   Local Cache    │
    │   Directory      │
    │                  │
    └──────┬───────────┘
           │
           ├──→ FileInfo (in LRU cache)
           │
           ├──→ CacheHandle (provides read interface)
           │
           └──→ Download Progress (Job status)


Data Flow During Eviction:
━━━━━━━━━━━━━━━━━━━━━━━━

  LRU Cache
     │
     ├─ Insert triggers eviction loop
     │
     ├─ evictOne() removes LRU entry
     │
     ├─ Returns FileInfo to Insert()
     │
     └─ Insert() collects evictedValues[]
           │
           └─ Returns to CacheHandler
               │
               └─ Cleanup loop calls cleanUpEvictedFile()
                   │
                   ├─ JobManager.InvalidateAndRemoveJob()
                   │     │
                   │     └─ Job.Invalidate()
                   │
                   └─ util.TruncateAndRemoveFile()
                         │
                         ├─ os.Truncate()
                         │
                         └─ os.Remove()
```

### Diagram 5: State Transitions

```
┌────────────────────────────────────────────────────────┐
│        Job and FileInfo State Transitions             │
└────────────────────────────────────────────────────────┘

Job States:
━━━━━━━━━

  ┌─────────────┐
  │ NotStarted  │
  └──────┬──────┘
         │
         ├─ Download() called
         │
         ▼
  ┌─────────────┐
  │ Downloading │◄──────────┐
  └──────┬──────┘           │
         │                  │
         ├─ Error           │
         │ or               ├─ Download progress update
         ├─ Invalidate()    │
         │ or               │
         ├─ Cancel()        │
         │                  │
         ▼                  │
  ┌─────────────┐           │
  │   Failed    │ ──────────┘ (May retry)
  └─────────────┘
         │
         ├─ Invalidate() called
         │
         ▼
  ┌─────────────┐
  │  Invalid    │ ◄──────────┐
  └─────────────┘            │
         │                   │
         └───────────────────┘
                (From Downloading or Failed)

FileInfo States:
━━━━━━━━━━━━━━

  ┌─────────────┐
  │   Created   │
  └──────┬──────┘
         │
         ├─ Inserted into LRU cache
         │
         ▼
  ┌─────────────────────┐
  │ In-Cache (Most/LRU) │◄─┐ On access:
  └─────────┬───────────┘  │ LookUp() → MRU
            │              │ (moved to front)
            │ ─────────────┘
            │
            ├─ Generation change
            ├─ Job failed/invalid
            ├─ Cache size exceeded
            ├─ Manual invalidation
            │
            ▼
  ┌──────────────────────┐
  │ Evicted (Cleanup)    │
  └──────────┬───────────┘
             │
             ├─ Job invalidated
             ├─ File truncated/deleted
             │
             ▼
  ┌──────────────────────┐
  │ Fully Cleaned Up     │
  └──────────────────────┘
```

### Diagram 6: Time-Based Eviction Trigger

```
┌──────────────────────────────────────────────────────┐
│       Cache Size Evolution & Eviction Timeline      │
└──────────────────────────────────────────────────────┘

Cache MaxSize: 1024 MiB
━━━━━━━━━━━━━━━━━━━━

1000 MiB ┤
         │
 900 MiB ┤   ┌─────────────┐
         │   │ File A      │
         │   │ (800 MiB)   │
 800 MiB ┤   │  ┌────────┐ │
         │   │  │MRU     │ │  ┌──────────────┐
         │   │  │Most     │  │ File B added │
         │   │  │Recent  │  │ (400 MiB)    │
 700 MiB ┤   │  └────────┘ │  └──────────────┘
         │   │             │
 600 MiB ┤   │ File A      │ ┌───────────────┐
         │   │ (800 MiB)   │ │ LRU Position: │
         │   │             │ │ File A (800)  │
 500 MiB ┤   │             │ │ File B (400)  │
         │   │   ┌────────┐ │ │ Total: 1200   │
         │   │   │   LRU  │ │ └───────────────┘
         │   │   │Least   │ │
         │   │   │Recent │ │  EVICTION TRIGGERED!
         │   │   └────────┘ │  ─────────────────
         │   └─────────────┘ │  Must free space
         │
   0 MiB ┤

    Time: t=0           t=1 (Insert File B)        t=2 (After Eviction)
                                                   
                                                   ┌─────────────┐
                                                   │ File B      │
                                                   │ (400 MiB)   │
                                                   │ MRU ┌─────┐ │
                                                   │     │ New │ │
                                                   │     └─────┘ │
                                                   │             │
                                                   │ File A      │
                                                   │ Evicted!    │
                                                   │             │
                                                   │ Total: 400  │
                                                   └─────────────┘
```

---

## Summary: Key Takeaways

1. **What Gets Evicted**: FileInfo entries (metadata) + associated cache files (data) + download jobs

2. **What Triggers Eviction**:
   - LRU when cache size exceeds MaxSizeMb
   - Generation change (new object version)
   - Failed download jobs
   - Manual invalidation (InvalidateCache)
   - Prefix-based cleanup (directory deletion)

3. **Eviction Process**: LRU (least recently used) → cleanup old entry → invalidate job → truncate/delete file

4. **Key Components**: LRU Cache (algorithm), CacheHandler (coordination), Job Manager (download control), FileInfo (metadata)

5. **Interaction Model**:
   - FileInfo represents cached object metadata
   - CacheHandle provides read interface
   - Job manages download progress
   - Eviction invalidates all three

6. **Cleanup**: Atomic operation with error recovery (truncate frees space even if delete fails)

7. **Configurations**:
   - `max-size-mb`: Size limit (-1 for unlimited)
   - `exclude-regex` / `include-regex`: Filter patterns
   - `cache-file-for-range-read`: Range read caching
   - Other download configurations

8. **Edge Cases**: Handled carefully (open file handles, concurrent access, generation races, failed jobs)

9. **Graceful Shutdown**: All jobs invalidated and cancelled on unmount

---

## File References

| Component | File Path | Lines | Purpose |
|-----------|-----------|-------|---------|
| LRU Cache | `/home/user/gcsfuse/internal/cache/lru/lru.go` | 1-279 | Core eviction algorithm |
| LRU Tests | `/home/user/gcsfuse/internal/cache/lru/lru_test.go` | Multiple | Test eviction scenarios |
| Cache Handler | `/home/user/gcsfuse/internal/cache/file/cache_handler.go` | 1-334 | Coordination and cleanup |
| Cache Handle | `/home/user/gcsfuse/internal/cache/file/cache_handle.go` | 1-301 | Read interface |
| Job Manager | `/home/user/gcsfuse/internal/cache/file/downloader/downloader.go` | 1-172 | Job lifecycle management |
| Download Job | `/home/user/gcsfuse/internal/cache/file/downloader/job.go` | 1-500+ | Job implementation |
| FileInfo | `/home/user/gcsfuse/internal/cache/data/file_info.go` | 1-62 | Data structure |
| Config | `/home/user/gcsfuse/cfg/config.go` | 410-437 | Configuration structs |
| FileSystem | `/home/user/gcsfuse/internal/fs/fs.go` | 250-279 | Initialization |
| Utilities | `/home/user/gcsfuse/internal/cache/util/util.go` | 1-286 | Cleanup utilities |

