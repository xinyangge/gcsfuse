# GCSFuse Component Lifecycle: FileHandle, CacheHandle, and FileInfo

## Executive Summary

This document provides a comprehensive analysis of three critical components in the gcsfuse codebase:
1. **FileHandle** - Manages file operations and orchestrates reading/writing
2. **CacheHandle** - Manages local file cache access and download coordination
3. **FileInfo** - Stores metadata about cached files (generation, offset, size)

These components work together to enable efficient file caching and I/O in gcsfuse.

---

## 1. FileHandle

### 1.1 Definition & Structure

**File:** `/home/user/gcsfuse/internal/fs/handle/file.go` (lines 37-78)

```go
type FileHandle struct {
	inode *inode.FileInode                    // Reference to the underlying file inode
	mu syncutil.InvariantMutex                // Mutex for thread safety
	reader gcsx.RandomReader                  // Random reader for file content
	readManager gcsx.ReadManager              // Read manager for optimized reading
	fileCacheHandler *file.CacheHandler       // Handler for file cache operations
	cacheFileForRangeRead bool                // Whether to cache range reads
	metricHandle metrics.MetricHandle         // For metrics collection
	openMode util.OpenMode                   // Read/Write/Append mode
	config *cfg.Config                        // Mount configuration
	bufferedReadWorkerPool workerpool.WorkerPool // Worker pool for buffered reads
	globalMaxReadBlocksSem *semaphore.Weighted   // Limits total read blocks
}
```

### 1.2 Creation Flow

**Creation Point:** `/home/user/gcsfuse/internal/fs/fs.go` (line 2803)

```go
// In OpenFile operation
fs.handles[handleID] = handle.NewFileHandle(
	in,                              // FileInode
	fs.fileCacheHandler,             // CacheHandler
	fs.cacheFileForRangeRead,        // bool
	fs.metricHandle,                 // MetricHandle
	openMode,                        // util.OpenMode
	fs.newConfig,                    // Config
	fs.bufferedReadWorkerPool,       // WorkerPool
	fs.globalMaxReadBlocksSem        // Semaphore
)
```

**Constructor:** `/home/user/gcsfuse/internal/fs/handle/file.go` (lines 81-97)

The `NewFileHandle()` function:
1. Creates a new FileHandle struct with provided parameters
2. **Registers itself with the FileInode** - Line 93: `fh.inode.RegisterFileHandle(fh.openMode == util.Read)`
3. Creates an InvariantMutex with checkInvariants callback
4. Returns the initialized FileHandle

### 1.3 Triggering Operations

FileHandle creation is triggered by FUSE OpenFile operation, which occurs when:
- User opens a file in read mode (`open()`)
- User opens a file in write/append mode (`open()` with write flags)
- Multiple readers/writers can have concurrent FileHandles for the same inode

### 1.4 Component Interactions

#### FileHandle ↔ FileInode
- **Registration:** When created, registers with inode (line 93)
- **Deregistration:** When destroyed, deregisters from inode (line 107)
- **Write tracking:** Inode tracks count of write-mode FileHandles (FileInode.writeHandleCount)

#### FileHandle ↔ CacheHandler
- FileHandle holds reference to CacheHandler (line 60)
- Used in two ways:
  1. **RandomReader path** - Line 276: `gcsx.NewRandomReader(..., fh.fileCacheHandler, ...)`
  2. **ReadManager path** - Line 198: Passed to ReadManager creation

#### FileHandle ↔ ReadManager/Reader
- **Lazy initialization:** Reader/ReadManager created on first read operation
- **Read flow:** Lines 160-237 (ReadWithReadManager) and 244-302 (Read)
- **Invalidation:** Reader/ReadManager destroyed if inode becomes dirty

### 1.5 Lifecycle: Creation → Usage → Destruction

**Phase 1: Creation (OpenFile)**
```
1. FUSE kernel calls fs.OpenFile()
2. fs.OpenFile() creates FileHandle via NewFileHandle()
3. FileHandle registers with FileInode
4. Returns HandleID to kernel
```

**Phase 2: Usage (ReadFile/WriteFile)**
```
1. FUSE kernel calls fs.ReadFile() with HandleID
2. fs retrieves FileHandle from handles map
3. FileHandle.Read() or FileHandle.ReadWithReadManager() called
4. On first read:
   - Reader/ReadManager created with CacheHandler reference
   - Inode lock checked to ensure it's not dirty
5. Reader validates object generation matches inode
6. If generation mismatch, old reader destroyed, new one created
```

**Phase 3: Destruction (ReleaseFileHandle)**
```
Location: /home/user/gcsfuse/internal/fs/fs.go (lines 2993-3010)

Steps:
1. Kernel calls ReleaseFileHandle when file is closed
2. fs.mu.Lock() - acquire filesystem lock
3. FileHandle removed from fs.handles map (line 3001)
4. fs.mu.Unlock()
5. fh.Lock() - acquire FileHandle lock (line 3005)
6. fh.Destroy() called (line 3007)
   a. Deregister from inode
   b. Destroy reader if exists
   c. Destroy readManager if exists
7. fh.Unlock()
```

### 1.6 Key Methods

**Read Methods:**
- `Read()` - Lines 244-302: Read using RandomReader (legacy path)
- `ReadWithReadManager()` - Lines 164-237: Read using ReadManager (new path)

**Lifecycle Methods:**
- `Destroy()` - Lines 104-115: Clean up resources
- `Lock()/Unlock()` - Lines 122-128: Thread safety

**Helper Methods:**
- `isValidReader()` - Lines 371-380: Check if reader is at correct generation
- `isValidReadManager()` - Lines 345-354: Check if readManager is at correct generation
- `destroyReader()` - Lines 359-365: Safely destroy reader
- `destroyReadManager()` - Lines 333-339: Safely destroy readManager

---

## 2. CacheHandle

### 2.1 Definition & Structure

**File:** `/home/user/gcsfuse/internal/cache/file/cache_handle.go` (lines 30-52)

```go
type CacheHandle struct {
	fileHandle *os.File                   // OS file handle to local cache file
	fileDownloadJob *downloader.Job       // Reference to async download job
	fileInfoCache *lru.Cache              // Reference to fileInfo LRU cache
	cacheFileForRangeRead bool            // Whether to cache range reads
	isSequential bool                     // Is current read sequential?
	prevOffset int64                      // Previous read offset for sequential detection
}
```

### 2.2 Creation Flow

**Creation Point:** `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 230-269)

The `GetCacheHandle()` method:

```go
func (chr *CacheHandler) GetCacheHandle(
	object *gcs.MinObject,
	bucket gcs.Bucket,
	cacheForRangeRead bool,
	initialOffset int64) (*CacheHandle, error) {
	
	// Step 1: Check if file should be excluded
	if chr.shouldExcludeFromCache(bucket, object) {
		return nil, util.ErrFileExcludedFromCacheByRegex
	}
	
	// Step 2: Skip creation for random reads if cacheFileForRangeRead=false
	if !cacheForRangeRead && initialOffset != 0 {
		// Return nil if file not already in cache
	}
	
	// Step 3: Add FileInfo entry to cache and create download job
	err := chr.addFileInfoEntryAndCreateDownloadJob(object, bucket)
	
	// Step 4: Create local file handle
	localFileReadHandle, err := chr.createLocalFileReadHandle(object.Name, bucket.Name())
	
	// Step 5: Create and return CacheHandle
	return NewCacheHandle(localFileReadHandle, job, fileInfoCache, 
		cacheForRangeRead, initialOffset), nil
}
```

**Constructor:** `/home/user/gcsfuse/internal/cache/file/cache_handle.go` (lines 54-64)

```go
func NewCacheHandle(localFileHandle *os.File, fileDownloadJob *downloader.Job,
	fileInfoCache *lru.Cache, cacheFileForRangeRead bool, initialOffset int64) *CacheHandle {
	return &CacheHandle{
		fileHandle:            localFileHandle,
		fileDownloadJob:       fileDownloadJob,
		fileInfoCache:         fileInfoCache,
		cacheFileForRangeRead: cacheFileForRangeRead,
		isSequential:          initialOffset == 0,
		prevOffset:            initialOffset,
	}
}
```

### 2.3 Triggering Operations

CacheHandle creation is triggered by:

1. **RandomReader path** - `/home/user/gcsfuse/internal/gcsx/random_reader.go` (line 269)
   - When RandomReader.ReadAt() is called and cache is available

2. **FileCacheReader path** - `/home/user/gcsfuse/internal/gcsx/file_cache_reader.go` (line 129)
   - When FileCacheReader.tryReadingFromFileCache() is called
   - More common path with new read infrastructure

3. **ReadManager path** - `/home/user/gcsfuse/internal/gcsx/read_manager/read_manager.go`
   - ReadManager internally uses FileCacheReader which creates CacheHandle

### 2.4 Component Interactions

#### CacheHandle ↔ FileInfo (via LRU Cache)
- FileInfo entries stored in LRU cache with key format: `{bucketName}{bucketCreationTime}{objectName}`
- **Lookup:** `validateEntryInFileInfoCache()` - Line 131-144
  - Checks generation matches
  - Checks offset >= requiredOffset
- **Update:** Every successful read updates LRU order - Line 263

#### CacheHandle ↔ Download Job
- **Job lifecycle:** Job can be in states: NotStarted, Downloading, Completed, Failed, Invalid
- **Job interaction:**
  - Sequential reads: Wait for download - `fileDownloadJob.Download(ctx, requiredOffset, waitForDownload=true)`
  - Random reads (if cacheFileForRangeRead=true): Wait for download conditionally
  - Random reads (if cacheFileForRangeRead=false): Don't wait for download
- **Job status:** `fch.fileDownloadJob.GetStatus()` returns JobStatus with current offset

#### CacheHandle ↔ Local File
- Reads from local cached file via `fileHandle.ReadAt()` - Line 242
- File is managed by CacheHandler/JobManager lifecycle

### 2.5 Lifecycle: Creation → Usage → Destruction

**Phase 1: Creation**
```
1. FileCacheReader.tryReadingFromFileCache() called
2. Calls CacheHandler.GetCacheHandle(object, bucket, cacheForRangeRead, offset)
3. CacheHandler:
   a. Adds FileInfo entry to LRU cache (creates or reuses)
   b. Creates/retrieves download Job from JobManager
   c. Creates OS file handle to local cache file
4. Returns new CacheHandle with all references set
```

**Phase 2: Usage (Read)**
```
Location: /home/user/gcsfuse/internal/cache/file/cache_handle.go (lines 151-269)

Steps:
1. CacheHandle.Read(ctx, bucket, object, offset, dst) called
2. Determine if read is sequential - IsSequential() (lines 273-287)
3. Calculate required offset for download
4. If fileDownloadJob exists:
   a. Get job status
   b. For sequential: waitForDownload=true
   c. For random: waitForDownload=cacheFileForRangeRead
   d. Call Job.Download(ctx, requiredOffset, waitForDownload)
   e. Job downloads data asynchronously/synchronously
5. Call fileHandle.ReadAt(dst, offset) to read from local file
6. Update FileInfo LRU entry order (lookUp with changeCacheOrder=true)
7. Return bytes read and cache hit status
```

**Phase 3: Destruction (Close)**
```
Location: /home/user/gcsfuse/internal/cache/file/cache_handle.go (lines 290-301)

Steps:
1. CacheHandle.Close() called when:
   - FileCacheReader re-initializes on error
   - FileCacheReader is destroyed
2. Close the fileHandle (os.File)
3. Set fileHandle to nil
4. Download Job continues independently (managed by JobManager)
```

### 2.6 Key Methods

**Read Methods:**
- `Read()` - Lines 151-269: Main read operation with job coordination
- `IsSequential()` - Lines 273-287: Detect read pattern

**Validation Methods:**
- `shouldReadFromCache()` - Lines 80-89: Check if job status allows reading
- `validateEntryInFileInfoCache()` - Lines 131-144: Validate FileInfo entry
- `getFileInfoData()` - Lines 95-124: Retrieve FileInfo from cache

**Lifecycle Methods:**
- `Close()` - Lines 290-301: Clean up file handle

---

## 3. FileInfo

### 3.1 Definition & Structure

**File:** `/home/user/gcsfuse/internal/cache/data/file_info.go` (lines 26-61)

```go
// Key for FileInfo cache entries
type FileInfoKey struct {
	BucketName         string      // GCS bucket name
	BucketCreationTime time.Time   // Bucket creation timestamp
	ObjectName         string      // GCS object name
}

// Metadata about a cached file
type FileInfo struct {
	Key              FileInfoKey  // Reference to bucket and object
	ObjectGeneration int64        // GCS object generation number
	Offset           uint64       // How much has been downloaded (cache offset)
	FileSize         uint64       // Total size of the object
}
```

### 3.2 Key Generation

**Method:** `FileInfoKey.Key()` - Lines 34-36 and `GetFileInfoKeyName()` - Lines 38-44

```go
// Creates cache key as: "{bucketName}{bucketCreationTime.Unix()}{objectName}"
// Example: "my-bucket1609459200my-file.txt"
func (fik FileInfoKey) Key() (string, error) {
	return GetFileInfoKeyName(fik.ObjectName, fik.BucketCreationTime, fik.BucketName)
}

func GetFileInfoKeyName(objectName string, bucketCreationTime time.Time, bucketName string) (string, error) {
	if bucketName == "" || objectName == "" {
		return "", errors.New(InvalidKeyAttributes)
	}
	unixTimeString := fmt.Sprintf("%d", bucketCreationTime.Unix())
	return bucketName + unixTimeString + objectName, nil
}
```

### 3.3 Creation Flow

**Primary Creation:** `/home/user/gcsfuse/internal/cache/file/cache_handler.go` (lines 131-217)

In `addFileInfoEntryAndCreateDownloadJob()`:

```go
// Step 1: Create FileInfoKey
fileInfoKey := data.FileInfoKey{
	BucketName: bucket.Name(),
	ObjectName: object.Name,
}

// Step 2: Create FileInfo
fileInfo := data.FileInfo{
	Key:              fileInfoKey,
	ObjectGeneration: object.Generation,    // Current generation from GCS
	Offset:           0,                    // Download starts at 0
	FileSize:         object.Size,          // Total object size
}

// Step 3: Insert into cache
evictedValues, err := chr.fileInfoCache.Insert(fileInfoKeyName, fileInfo)

// Step 4: Create download job if new
if addEntryToCache {
	_ = chr.jobManager.CreateJobIfNotExists(object, bucket)
}
```

### 3.4 Triggering Operations

FileInfo creation triggered by:

1. **First file read** - When GetCacheHandle called for new object
2. **Generation change** - When object generation changes in GCS
3. **Cache miss** - When object not in FileInfo cache
4. **Cache eviction** - When LRU cache reaches capacity

### 3.5 Component Interactions

#### FileInfo ↔ CacheHandle
- **Read path:** CacheHandle validates FileInfo before/after reads
- **Validation:** `validateEntryInFileInfoCache()` ensures:
  - Generation matches current object generation
  - Cached offset >= required offset
- **Update:** FileInfo.Offset updated by Job as download progresses

#### FileInfo ↔ Download Job
- **Offset tracking:** Job.GetStatus() returns current download offset
- **FileInfo.Offset:** Updated from JobStatus.Offset
- **Generation:** FileInfo.ObjectGeneration must match object.Generation

#### FileInfo ↔ LRU Cache
- **Storage:** FileInfo stored as value in LRU cache
- **Key:** Generated from bucket name, creation time, object name
- **Eviction:** When cache is full, oldest entries evicted
- **Cleanup:** Evicted FileInfo triggers job invalidation and file deletion

### 3.6 Lifecycle: Creation → Usage → Destruction

**Phase 1: Creation**
```
1. CacheHandler.GetCacheHandle() called for object
2. CacheHandler.addFileInfoEntryAndCreateDownloadJob() called
3. Checks if FileInfo exists in cache:
   a. If exists and valid (same generation): reuse
   b. If different generation: evict old, create new
   c. If doesn't exist: create new
4. FileInfo created with:
   - ObjectGeneration from current GCS object
   - Offset=0 (download starts at beginning)
   - FileSize from object.Size
5. Insert into LRU cache - may evict oldest entries
```

**Phase 2: Usage (Read Operations)**
```
1. CacheHandle.Read() called
2. CacheHandle.validateEntryInFileInfoCache() checks:
   a. FileInfo exists in cache
   b. Generation matches object.Generation
   c. Offset >= requiredOffset
3. If validation fails:
   a. Fall back to GCS
   b. Or return error
4. FileInfo.Offset updated by Job as download progresses
5. LRU lookup called to update entry recency
```

**Phase 3: Destruction (LRU Eviction)**
```
Location: /home/user/gcsfuse/internal/cache/file/cache_handler.go (lines 106-129)

Triggered when:
1. LRU cache exceeds capacity
2. New FileInfo needs to be inserted
3. Existing FileInfo generation differs from object
4. Job failed or was invalidated

Steps:
1. LRU cache evicts oldest FileInfo entry
2. CacheHandler.cleanUpEvictedFile() called with evicted FileInfo:
   a. Get object name and bucket name from FileInfo.Key
   b. Call JobManager.InvalidateAndRemoveJob()
   c. Call util.TruncateAndRemoveFile() on local cache file
3. Job invalidation:
   a. Job state set to Invalid
   b. Async download cancelled if in progress
   c. Job removed from JobManager.jobs map
4. Local file truncated and deleted
```

### 3.7 Key Methods

**Key Management:**
- `Key()` - Line 34-36: Generate cache key
- `GetFileInfoKeyName()` - Lines 38-44: Create key string

**Accessor:**
- `Size()` - Lines 53-55: Return FileSize

### 3.8 State Tracking

FileInfo tracks download progress:

```
Initial state:
  FileInfo { Offset: 0, FileSize: 100MB }
  Job.Status { Offset: 0, Name: NotStarted }

During sequential read (0, 10MB):
  -> Job downloads to offset 10MB
  -> Job.Status { Offset: 10MB, Name: Downloading }
  -> FileInfo.Offset updated to 10MB (indirectly via Job)

Completed:
  FileInfo { Offset: 100MB, FileSize: 100MB }
  Job.Status { Offset: 100MB, Name: Completed }
```

---

## 4. Interaction Flows

### 4.1 Complete Read Flow (New ReadManager Path)

```
FUSE Read Request
    ↓
fs.ReadFile()
    ↓
FileHandle.ReadWithReadManager()
    ↓
ReadManager.ReadAt()  [/internal/gcsx/read_manager/read_manager.go]
    ├→ Try FileCacheReader first
    │   ↓
    │   FileCacheReader.tryReadingFromFileCache()
    │   ↓
    │   [First read for file]
    │       ↓
    │       CacheHandler.GetCacheHandle()
    │       ├→ CacheHandler.addFileInfoEntryAndCreateDownloadJob()
    │       │  ├→ Create/retrieve FileInfo entry in LRU cache
    │       │  └→ Create/retrieve download Job
    │       └→ Create local file OS handle
    │           ↓
    │       CacheHandle created and stored
    │   ↓
    │   [Subsequent reads or first read]
    │       ↓
    │       CacheHandle.Read()
    │       ├→ Determine if sequential read
    │       ├→ Get Job status
    │       ├→ Job.Download(requiredOffset)
    │       │  └→ Download data to local file
    │       ├→ Read from local file handle
    │       └→ Validate FileInfo in cache
    │           ↓
    │       Return cached data (CacheHit=true)
    │
    └→ Fall back to GCS reader if cache miss
        └→ Return GCS data (CacheHit=false)
```

### 4.2 File Lifecycle During Cache Eviction

```
LRU Cache Full
    ↓
New FileInfo needs space
    ↓
LRU evicts oldest FileInfo entry
    ↓
CacheHandler.cleanUpEvictedFile(evictedFileInfo)
    ├→ Job.Invalidate()
    │   ├→ Set state to Invalid
    │   ├→ Cancel async download if running
    │   └→ Remove from JobManager
    ├→ Truncate and delete local cache file
    └→ FileInfo removed from LRU
```

### 4.3 Generation Change Flow

```
Object re-uploaded to GCS (new generation)
    ↓
Application reads file
    ↓
CacheHandler.GetCacheHandle()
    ↓
Checks FileInfo in cache
    ↓
FileInfo.ObjectGeneration != object.Generation
    ↓
Old FileInfo evicted
    ├→ Existing Job invalidated
    ├→ Local cache file deleted
    └→ Download job cancelled
    ↓
New FileInfo created with new generation
    ↓
New download Job created for new generation
```

### 4.4 FileHandle Lifecycle Within Read Operation

```
FileHandle.Read() called
    ↓
Acquire lock: fh.mu.RLock()
    ↓
Check if reader valid: fh.isValidReader()
    ├→ reader != nil AND
    ├→ reader.Object().Generation == inode.SourceGeneration().Object
    │
    ├→ YES → Use existing reader
    │         ↓
    │         reader.ReadAt(offset)
    │
    └→ NO → Create new reader
            ├→ fh.mu.Lock() (upgrade to write lock)
            ├→ Destroy old reader if exists
            ├→ Create new RandomReader with:
            │   ├→ inode.Source() (current object)
            │   ├→ fileCacheHandler (for cache access)
            │   └→ other config
            ├→ fh.mu.RUnlock() (downgrade to read lock)
            └→ reader.ReadAt(offset)
    ↓
Release lock: fh.mu.RUnlock()
    ↓
Return data
```

---

## 5. State Machines

### 5.1 FileHandle State

```
NOT_CREATED
    ↓ [fs.OpenFile()]
CREATED (reader=nil, readManager=nil)
    ↓ [First read operation]
    ├→ READING (reader created OR readManager created)
    │   ├→ [Inode becomes dirty]
    │   └→ READING_INVALIDATED (reader/readManager destroyed)
    │       ↓ [Next read operation]
    │       └→ READING (new reader/readManager created)
    ↓ [fs.ReleaseFileHandle()]
DESTROYED (all resources cleaned up)
```

### 5.2 CacheHandle State

```
NOT_CREATED
    ↓ [FileCacheReader.tryReadingFromFileCache()]
CREATED (fileHandle open, Job reference set)
    ├→ READING (read operations in progress)
    │   └→ [Error detected]
    │       └→ INVALID (fileHandle closed, set to nil)
    │           ↓
    │           Next error → Create new CacheHandle
    └→ CLOSED (fileHandle.Close() called)
        └→ DESTROYED
```

### 5.3 FileInfo State (in Cache)

```
NOT_CREATED
    ↓ [First access to new object]
IN_CACHE (LRU managed)
    ├→ VALID (generation matches, offset valid)
    │   └→ [Generation changes]
    │       └→ INVALIDATED (marked for eviction)
    ├→ MODIFIED (Offset updated as download progresses)
    └→ [LRU eviction]
        └→ EVICTED (removed from cache, resources cleaned)
```

### 5.4 Download Job State

```
NOT_STARTED
    ↓ [GetCacheHandle() creates job]
Downloading [async download in progress]
    ├→ [Data reaches required offset]
    │   └→ Subscribers notified (CacheHandle.Read() can proceed)
    ├→ [Download completes]
    │   └→ COMPLETED
    ├→ [Download fails]
    │   └→ FAILED
    └→ [Job invalidated]
        └→ INVALID
```

---

## 6. Concurrency Considerations

### 6.1 FileHandle Concurrency
- **Mutex:** `mu` (InvariantMutex) protects reader/readManager
- **Lock ordering:** Always acquire FileInode lock before FileHandle lock
- **Invariant:** If reader != nil, reader.CheckInvariants() doesn't panic

### 6.2 CacheHandle Concurrency
- **Not concurrent:** CacheHandle is per-read-instance (FileCacheReader holds one)
- **Job concurrency:** Job is shared across CacheHandles via JobManager
- **Download coordination:** Job uses channels for subscriber notifications

### 6.3 FileInfo Concurrency
- **LRU Cache protection:** Cache uses internal locks
- **Generation checks:** Atomic reads of object.Generation from inode

### 6.4 JobManager Concurrency
- **Mutex:** `mu` (Locker) protects jobs map
- **Callback pattern:** Jobs remove themselves via callback to avoid deadlock
- **Semaphore:** maxParallelismSem limits concurrent downloads

---

## 7. Error Handling & Edge Cases

### 7.1 FileHandle Errors
- **Invalid reader generation:** Detected by `isValidReader()`, old reader destroyed
- **Inode dirty:** Falls back to direct inode read
- **EOF handling:** Distinguishes between io.EOF and io.ErrUnexpectedEOF

### 7.2 CacheHandle Errors
- **Job failures:** `shouldReadFromCache()` checks job error
- **Cache invalidation:** `IsCacheHandleInvalid()` triggers cleanup
- **File not in cache:** ErrFileNotPresentInCache triggers fallback
- **Generation mismatch:** `validateEntryInFileInfoCache()` catches it

### 7.3 FileInfo Errors
- **Uninitialized key:** Returns InvalidKeyAttributes error
- **Evicted entries:** cleanUpEvictedFile handles cleanup

### 7.4 Generation Changes
- **Lazy detection:** Checked on each read operation
- **Full rebuild:** New reader/readManager created with new generation
- **Cache invalidation:** Old FileInfo evicted, new one created

---

## 8. Configuration & Tuning

### 8.1 CacheHandle Configuration
- **cacheFileForRangeRead:** Controls whether random reads trigger downloads
- **sequential detection:** ReadChunkSize (8MiB) defines sequential threshold

### 8.2 FileHandle Configuration
- **sequentialReadSizeMb:** Size of GCS read chunks
- **bufferedReadWorkerPool:** Async prefetch thread pool
- **globalMaxReadBlocksSem:** Limits total memory for buffered reads

### 8.3 FileInfo/Job Configuration
- **LRU cache size:** Limits number of cached objects
- **maxParallelDownloads:** Limits concurrent job downloads
- **file permissions:** Configurable via FilePerm/DirPerm

---

## 9. References to Key Code Locations

### Component Files
| Component | File | Lines |
|-----------|------|-------|
| FileHandle | `/internal/fs/handle/file.go` | 37-385 |
| CacheHandle | `/internal/cache/file/cache_handle.go` | 30-301 |
| FileInfo | `/internal/cache/data/file_info.go` | 26-61 |
| CacheHandler | `/internal/cache/file/cache_handler.go` | 32-333 |
| JobManager | `/internal/cache/file/downloader/downloader.go` | 31-171 |
| Job | `/internal/cache/file/downloader/job.go` | 53-141 |
| FileCacheReader | `/internal/gcsx/file_cache_reader.go` | 40-212 |
| ReadManager | `/internal/gcsx/read_manager/read_manager.go` | 36-120 |

### Key Operations
| Operation | File | Lines |
|-----------|------|-------|
| FileHandle creation | `/internal/fs/fs.go` | 2775-2813 |
| FileHandle destruction | `/internal/fs/fs.go` | 2993-3010 |
| CacheHandle creation | `/internal/cache/file/cache_handler.go` | 230-269 |
| FileInfo creation | `/internal/cache/file/cache_handler.go` | 131-217 |
| Read flow | `/internal/cache/file/cache_handle.go` | 151-269 |
| Job creation | `/internal/cache/file/downloader/downloader.go` | 106-124 |
| FileInode registration | `/internal/fs/inode/file.go` | 433-459 |

---

## 10. Summary & Key Takeaways

1. **FileHandle** acts as the main interface between FUSE operations and the underlying file system, managing reader lifecycle and coordinating access.

2. **CacheHandle** provides a stateful interface to local cached files, coordinating with download Jobs to manage when data is available for reading.

3. **FileInfo** stores cache metadata (generation, offset, size) in an LRU cache, enabling validation and tracking of cached file progress.

4. **Three-tier architecture:** FileHandle (FUSE) → ReadManager/FileCacheReader (read strategy) → CacheHandle/Job (cache management) → FileInfo (metadata)

5. **Generation awareness:** All components track GCS object generation to detect when objects are modified, requiring cache invalidation.

6. **Lazy initialization:** Readers and CacheHandles are created on first access, not at open time, for efficiency.

7. **Concurrent downloads:** Multiple CacheHandles can reference the same Job, enabling efficient sharing of download progress across readers.

8. **Cleanup hierarchy:** FileHandle → CacheHandle → FileInfo → Job, with each level responsible for its own cleanup.

