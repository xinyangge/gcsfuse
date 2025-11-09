# GCSFuse Component Architecture - Visual Summary

## Component Relationships Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          FUSE Kernel                                │
│                    (Open/Read/Write/Close)                          │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   FileSystem (fs.go)    │
                    │  - nextHandleID         │
                    │  - handles map          │
                    │  - fileCacheHandler     │
                    └────────────┬────────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
    ┌───────────▼────────┐      │      ┌──────────▼──────────┐
    │   OPEN: Create     │      │      │  CLOSE: Destroy     │
    │   FileHandle       │      │      │  FileHandle         │
    └───────────┬────────┘      │      └──────────▲──────────┘
                │               │                 │
                └───────────┬───▼─────────────────┘
                            │
                ┌───────────▼────────────────┐
                │   FileHandle               │
                │  - inode (FileInode)       │
                │  - reader/readManager      │
                │  - fileCacheHandler ref    │
                │  - mutex (mu)              │
                └───────────┬────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        │           ┌───────▼────────┐          │
        │           │  On First Read  │          │
        │           └───────┬────────┘          │
        │                   │                   │
    ┌───▼─────────┐    ┌───▼──────────┐    ┌──▼──────────┐
    │ RandomReader│    │ ReadManager   │    │Inode dirty? │
    │ (legacy)    │    │ (new path)    │    │  Invalidate │
    └───┬─────────┘    └───┬──────────┘    └─────────────┘
        │                  │
        │    ┌─────────────┼──────────────┐
        │    │             │              │
        │    │    ┌────────▼──────────┐   │
        │    │    │ FileCacheReader   │   │
        │    │    │ - fileCacheHandle │   │
        │    │    └────────┬──────────┘   │
        │    │             │              │
        │    │   ┌─────────▼────────┐     │
        │    │   │ CacheHandler     │     │
        │    │   │ GetCacheHandle() │     │
        │    │   └────────┬────────┘      │
        │    │            │               │
        │    │   ┌────────▼──────────────┐│
        │    │   │ CacheHandle          ││
        │    │   │ - fileHandle (OS)    ││
        │    │   │ - fileDownloadJob    ││
        │    │   │ - fileInfoCache ref  ││
        │    │   └────────┬─────────────┘│
        │    │            │              │
        │    │   ┌────────▼──────────────┐
        │    │   │ On CacheHandle.Read()│
        │    │   │ - Detect sequential  │
        │    │   │ - Coordinate Job     │
        │    │   │ - Read local file    │
        │    │   │ - Validate FileInfo  │
        │    │   └──────────────────────┘
        │    │
        └────┼──────────────────────────────┐
             │                              │
       ┌─────▼──────────────────────────┐  │
       │ Download Job (JobManager)      │  │
       │ - FileInfo cache ref           │  │
       │ - Download async/sync          │  │
       │ - Emit notifications           │  │
       └─────┬──────────────────────────┘  │
             │                              │
       ┌─────▼──────────────────────────┐  │
       │ FileInfo (in LRU Cache)        │  │
       │ - ObjectGeneration (GCS)       │  │
       │ - Offset (download progress)   │  │
       │ - FileSize                     │  │
       └────────────────────────────────┘  │
                                           │
                    ┌──────────────────────┘
                    │
              ┌─────▼──────────────┐
              │ Local Cache File   │
              │ (on disk)          │
              └────────────────────┘
```

## Component Lifecycle Timeline

```
Time ────────────────────────────────────────────────────────────────>

1. User opens file
   │
   ├─► fs.OpenFile()
   │   └─► FileHandle.NewFileHandle()
   │       └─► FileInode.RegisterFileHandle()  ◄── Write tracking
   │
   [FileHandle lives as long as file is open]

2. First read operation
   │
   ├─► fs.ReadFile()
   │   └─► FileHandle.Read() or ReadWithReadManager()
   │       └─► Check if reader valid? [reader == nil]
   │           ├─► YES: Destroy old, create new
   │           │   └─► gcsx.NewRandomReader() + fileCacheHandler
   │           │       or ReadManager with FileCacheReader
   │           │
   │           └─► Use existing reader
   │
   [Reader/ReadManager lifetime: first read to next invalidation]

3. File cache access (if enabled)
   │
   ├─► FileCacheReader.tryReadingFromFileCache()
   │   └─► Check if CacheHandle exists
   │       ├─► NO: CacheHandler.GetCacheHandle()
   │       │   ├─► addFileInfoEntryAndCreateDownloadJob()
   │       │   │   ├─► Create FileInfo entry in LRU cache
   │       │   │   │   └─► Generate key: {bucket}{time}{object}
   │       │   │   └─► JobManager.CreateJobIfNotExists()
   │       │   │       └─► Create download Job
   │       │   └─► Create OS file handle to cache file
   │       │       └─► NewCacheHandle returns
   │       │
   │       └─► YES: Use existing CacheHandle
   │
   │       CacheHandle.Read()
   │       ├─► Determine if sequential read
   │       ├─► Job.Download(requiredOffset)
   │       │   └─► Wait or don't wait (based on read type)
   │       ├─► fileHandle.ReadAt() from local file
   │       └─► validateEntryInFileInfoCache()
   │           └─► Update LRU order
   │
   [CacheHandle lifetime: first cache read to cache error]

4. Download job runs (parallel)
   │
   ├─► Job.downloadAsync()
   │   ├─► Download data from GCS
   │   ├─► Write to local cache file
   │   ├─► Update offset as data arrives
   │   └─► Notify subscribers of progress
   │
   [Job lifetime: created to Completed/Failed/Invalid]

5. FileInfo update cycle
   │
   ├─► As download progresses:
   │   └─► FileInfo.Offset tracked (indirectly via Job)
   │
   ├─► On each read:
   │   └─► LRU.LookUp() updates FileInfo recency
   │
   [FileInfo lifetime: created to LRU eviction]

6. LRU cache eviction (when capacity reached)
   │
   ├─► LRU.Insert() for new FileInfo
   │   └─► Erase oldest FileInfo
   │       └─► CacheHandler.cleanUpEvictedFile()
   │           ├─► Job.Invalidate()
   │           │   ├─► Set state = Invalid
   │           │   ├─► Cancel download if running
   │           │   └─► Job removes itself from JobManager
   │           ├─► Truncate local cache file
   │           └─► Delete local file
   │
   [FileInfo + Job cleanup triggered]

7. User closes file
   │
   ├─► fs.ReleaseFileHandle()
   │   └─► FileHandle.Destroy()
   │       ├─► FileInode.DeRegisterFileHandle()
   │       │   └─► Decrement write handle count
   │       │       └─► Cleanup buffered writes if needed
   │       ├─► reader.Destroy() if exists
   │       └─► readManager.Destroy() if exists
   │
   [FileHandle destroyed, but CacheHandle/FileInfo/Job live on]
```

## Data Flow - Sequential Read

```
Application Request: read(file_fd, buffer, size) at offset=0
        │
        ▼
FUSE ReadFile Operation
        │
        ▼
FileHandle.ReadWithReadManager(ctx, dst, offset=0)
        │
        ▼
ReadManager.ReadAt(ctx, p, offset=0)
        │
        ├─► Try FileCacheReader[0] (priority 1)
        │   │
        │   ▼
        │   FileCacheReader.tryReadingFromFileCache()
        │   │
        │   ├─► [First read: CacheHandle = nil]
        │   │   │
        │   │   ▼
        │   │   CacheHandler.GetCacheHandle()
        │   │   │
        │   │   ├─► addFileInfoEntryAndCreateDownloadJob()
        │   │   │   │
        │   │   │   ├─► Create FileInfoKey
        │   │   │   │   └─► key = "bucket" + "1609459200" + "file.txt"
        │   │   │   │
        │   │   │   ├─► LRU cache lookup by key
        │   │   │   │   └─► NOT FOUND
        │   │   │   │
        │   │   │   ├─► Create new FileInfo
        │   │   │   │   {
        │   │   │   │     Key: {bucket, time, object_name},
        │   │   │   │     ObjectGeneration: 1234567890,
        │   │   │   │     Offset: 0,
        │   │   │   │     FileSize: 100MB
        │   │   │   │   }
        │   │   │   │
        │   │   │   ├─► LRU.Insert(key, FileInfo)
        │   │   │   │   └─► [May evict oldest if full]
        │   │   │   │
        │   │   │   └─► JobManager.CreateJobIfNotExists()
        │   │   │       └─► Create new Job for download
        │   │   │           {
        │   │   │             object,
        │   │   │             bucket,
        │   │   │             fileInfoCache ref,
        │   │   │             fileSpec (path, perms)
        │   │   │           }
        │   │   │
        │   │   └─► Create OS file handle
        │   │       └─► local cache file
        │   │
        │   ├─► [Subsequent read: CacheHandle exists]
        │   │   └─► Use existing CacheHandle
        │   │
        │   ▼
        │   CacheHandle.Read(ctx, bucket, object, offset, dst)
        │   │
        │   ├─► isSequential = IsSequential(offset=0)
        │   │   └─► true (offset == prevOffset + small_gap)
        │   │
        │   ├─► requiredOffset = offset + len(dst) = 1MB
        │   │
        │   ├─► Job.GetStatus()
        │   │   └─► {Name: Downloading, Offset: 0, Error: nil}
        │   │
        │   ├─► waitForDownload = true (sequential)
        │   │
        │   ├─► Job.Download(ctx, requiredOffset=1MB, wait=true)
        │   │   │
        │   │   ├─► [Async download in Job goroutine]
        │   │   │   │
        │   │   │   ├─► Download 1MB from GCS
        │   │   │   ├─► Write to local cache file
        │   │   │   └─► Update status.Offset = 1MB
        │   │   │
        │   │   └─► [Block until Offset >= 1MB]
        │   │       └─► Subscriber notified
        │   │
        │   ├─► fileHandle.ReadAt(dst, offset=0)
        │   │   └─► Read 1MB from local cache file
        │   │
        │   ├─► validateEntryInFileInfoCache()
        │   │   │
        │   │   ├─► LRU.LookUp(key)  [changeCacheOrder=true]
        │   │   │   └─► Moves FileInfo to most-recent
        │   │   │
        │   │   ├─► Verify:
        │   │   │   ├─► FileInfo.ObjectGeneration == object.Generation ✓
        │   │   │   ├─► FileInfo.Offset >= 1MB ✓
        │   │   │   └─► FileInfo exists ✓
        │   │   │
        │   │   └─► return nil
        │   │
        │   └─► return (bytesRead=1MB, cacheHit=true, err=nil)
        │
        ├─► [Cache hit achieved]
        │   └─► Return cached data to application
        │
        └─► [If cache miss or error]
            └─► Fall back to next reader (GCS reader)
                └─► Read from GCS API directly
```

## State Transitions Diagram

### FileHandle State Machine
```
                        ┌──────────────┐
                        │ NOT_CREATED  │
                        └──────┬───────┘
                               │
                    [OpenFile]  │
                               ▼
                        ┌──────────────┐
                    ┌──▶│   CREATED    │◀─────┐
                    │   │ reader=nil   │      │
                    │   │ rdMgr=nil    │      │
                    │   └──────┬───────┘      │ [Next read]
                    │          │              │ [Generation ok]
              [Inode│ [First   │              │
               dirty]│  read]   │              │
                    │          ▼              │
                    │   ┌──────────────┐      │
                    │   │   READING    │──────┘
                    │   │ (reader OR   │
                    │   │  rdMgr set)  │
                    │   └──────┬───────┘
                    │          │
              [Gen │ [Invalid  │ [Close]
                  │  gen]      │
                  │          ▼
                  │   ┌──────────────────────┐
                  └──▶│ READING_INVALIDATED  │──┐
                      │ (reader/rdMgr torn   │  │ [ReleaseFileHandle]
                      │ down, recreate next) │  │
                      └──────────────────────┘  │
                                                │
                                        ┌───────▼──────┐
                                        │  DESTROYED   │
                                        │ (all cleaned)│
                                        └──────────────┘
```

### CacheHandle State Machine
```
                        ┌──────────────┐
                        │ NOT_CREATED  │
                        └──────┬───────┘
                               │
           [FileCacheReader]    │
           [first read]         │
                               ▼
                        ┌──────────────┐
                    ┌──▶│   CREATED    │◀────┐
                    │   │ fh open      │     │
                    │   │ job set      │     │
                    │   └──────┬───────┘     │
                    │          │ [Read]     │
                    │          ▼            │
                    │   ┌──────────────┐    │
                    │   │   READING    │────┘
                    │   │ coordinating │
                    │   │ downloads    │
                    │   └──────┬───────┘
                    │          │
              [Error]│ [Invalid]│
                    │          ▼
                    │   ┌──────────────┐
                    └──▶│   INVALID    │
                        │ fh=nil       │
                        │ [Recreate or │
                        │  skip cache] │
                        └──────────────┘

Closure sequence (independent of Job):
    CREATED/READING/INVALID
        │
        ▼
    CacheHandle.Close()
        │
        ├─► fileHandle.Close()  (OS file)
        ├─► fileHandle = nil
        └─► Job continues independently
```

### FileInfo State Machine
```
                        ┌──────────────┐
                        │ NOT_CREATED  │
                        └──────┬───────┘
                               │
   [CacheHandler.GetCacheHandle]
   [addFileInfoEntry...]       │
                               ▼
                        ┌──────────────┐
                        │   IN_CACHE   │
                        │ (LRU managed)│
                        └──────┬───────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
          [Each read]│              [Gen mismatch]
          [LRU.LookUp│ or job failed │
           updates]  │                │
                    ▼                ▼
                ┌─────────┐   ┌──────────────┐
                │  VALID  │   │ INVALIDATED  │
                │ Offset  │   │ (marked for  │
                │ matches │   │  eviction)   │
                └──┬──────┘   └──────┬───────┘
                   │                │
            [More reads]        [Next insert]
                   │                │
                   ▼                ▼
            ┌──────────────┐   ┌──────────────┐
            │ MODIFIED     │   │   EVICTED    │
            │ (Offset      │   │              │
            │  progresses) │   └──┬───────────┘
            └──────────────┘      │
                                  ▼
                        ┌──────────────────────┐
                        │ CLEANUP_EVICTED_FILE │
                        │ - Job.Invalidate()   │
                        │ - Delete file        │
                        │ - LRU remove         │
                        └──────────────────────┘
```

### Download Job State Machine
```
                        ┌──────────────┐
                        │ NOT_STARTED  │
                        └──────┬───────┘
                               │
                    [First CacheHandle.Read]
                               │
                               ▼
                        ┌──────────────┐
                        │ DOWNLOADING  │◄────┐
                        │ (async)      │     │
                        └──────┬───────┘     │
                               │            │
                    ┌──────────┴──────────┐  │
                    │                     │  │
                    ▼                     ▼  │
            ┌─────────────┐      ┌──────────┐ (Subscribers notified)
            │ COMPLETED   │      │ FAILED   │ when offset reached
            │(file fully  │      │(error    │
            │ downloaded) │      │ during   │
            └─────────────┘      │download) │
                                 └──────┬───┘
                                        │ [Can be retried
                                        │  or invalidated]
                                        │
                                        ▼
                                   ┌────────────┐
                                   │  INVALID   │
                                   │(cancelled) │
                                   │            │
                                   └────────┬───┘
                                            │
                                            ▼
                                   ┌────────────────┐
                                   │ REMOVED_FROM_  │
                                   │   JOBMANAGER   │
                                   │ (self-removed  │
                                   │  via callback) │
                                   └────────────────┘
```

## Memory Layout - CacheHandle Creation

```
Application calls: read(fd, buffer, size) at offset=0
        │
        ▼
Stack Frame (FileHandle.ReadWithReadManager):
    ┌─────────────────────────────────┐
    │ FileHandle (locked)              │
    │  .inode: ────────────┐           │
    │  .reader: ────────┐  │           │
    │  .readManager: ──┐│  │           │
    │  .fileCacheHandler: ──┬──┐       │
    └─────────────────────────┼───┼───┘
                              │   │
                              ▼   ▼
Heap (Persistent Objects):
    ┌──────────────────┐
    │ CacheHandler     │
    │  .fileInfoCache  │──────┐
    │  .jobManager     │──────┤
    │  .mu (lock)      │      │
    └──────────────────┘      │
                              │
                    ┌─────────┴──────────┐
                    │                    │
                    ▼                    ▼
    ┌──────────────────────┐  ┌──────────────────┐
    │ FileInfo (in LRU)    │  │ JobManager       │
    │ {                    │  │ {                │
    │   Key: {...},        │  │   jobs[]: {      │
    │   Generation: 1234,  │  │     Job{         │
    │   Offset: 0,         │  │       object     │
    │   FileSize: 100MB    │  │       bucket     │
    │ }                    │  │       fileHandle │
    └──────────────────────┘  │       status{    │
                              │         Offset:0 │
                    ┌─────────┴─────────┐        │
                    │                   ▼        │
        ┌──────────────────────┐  ┌─────────────┴──┐
        │ CacheHandle (temp)   │  │ Job goroutine  │
        │ {                    │  │ (downloading)  │
        │   fileHandle:────────┼──────┐            │
        │   fileDownloadJob:───┼──────┼───┐        │
        │   fileInfoCache:─────┼──┐   │   │        │
        │   isSequential: true  │  │   │   │        │
        │   prevOffset: 0       │  │   │   │        │
        │ }                     │  │   │   │        │
        └──────────────────────┘  │   │   │        │
                                  │   │   ▼
                    ┌─────────────┼───┼──────────────────┐
                    │             │   │                  │
                    ▼             ▼   ▼                  ▼
            ┌──────────────┐  ┌────────────┐  ┌──────────────────┐
            │ Local File   │  │ GCS API    │  │ Download Channel │
            │ /tmp/cache/  │  │ (bidi      │  │ (notifications)  │
            │ bucket/      │  │  reader)   │  │                  │
            │ file.txt     │  │            │  │ offset: 1MB ─────┼──┐
            │ (0 bytes)    │  │            │  │ complete: false  │  │
            └──────────────┘  └────────────┘  └──────────────────┘  │
                  ▲                                                   │
                  │                                                   │
                  └───────────────────────────────────────────────────┘
                          Data written as downloaded
```

## Cleanup Sequence Diagram

```
LRU Cache Full Scenario:
────────────────────────

1. Add new FileInfo to cache
   │
   └─► LRU.Insert(key2, newFileInfo)
       │
       ├─► Cache capacity exceeded
       │
       └─► LRU evicts entry
           │
           └─► Evicted = oldestFileInfo (key1)
               │
               ▼
2. CacheHandler.cleanUpEvictedFile(evicted)
   │
   ├─► Get bucket/object from evicted.Key
   │
   ├─► JobManager.InvalidateAndRemoveJob()
   │   │
   │   ├─► Get Job reference
   │   │
   │   └─► Job.Invalidate()
   │       │
   │       ├─► job.mu.Lock()
   │       │
   │       ├─► Set status = Invalid
   │       │
   │       ├─► If downloading:
   │       │   ├─► job.cancelFunc() call
   │       │   └─► Close(job.doneCh)  [signal]
   │       │
   │       ├─► job.mu.Unlock()
   │       │
   │       ├─► Wait <-job.doneCh (blocking)
   │       │
   │       └─► job.removeJobCallback()  [removes from JobManager.jobs]
   │
   ├─► util.TruncateAndRemoveFile()
   │   │
   │   ├─► os.Truncate()  [set size to 0]
   │   │
   │   └─► os.Remove()    [delete file]
   │
   └─► FileInfo removed from LRU cache
       │
       └─► Complete cleanup: File + Job + Metadata gone
```

## Thread Safety Model

```
Component        Synchronization     Guarded By        Access Pattern
─────────────────────────────────────────────────────────────────────
FileHandle       InvariantMutex      fh.mu             RWLock (read/write)
                                                       Reader held during reads
                                                       Writer held during 
                                                       reader/readManager updates

CacheHandle      None                (per-instance)    Single reader per
                 (not concurrent)                      FileCacheReader

FileInfo         LRU Cache internal  fileInfoCache     Lock acquired on lookup
(in LRU)         locks               (no external)     

FileInode        syncutil.Mutex      inode.mu          RWLock pattern
                                                       Write handle count
                                                       protected

CacheHandler     Locker              chr.mu            Atomic operations:
                                                       addFileInfo...Job
                                                       GetCacheHandle
                                                       InvalidateCache

JobManager       Locker              jm.mu             Job map operations
                 + Semaphore         jm.maxParallel    Concurrency limit

Job              Locker              job.mu            Subscriber list
                 + Channels          (+ doneCh)        State transitions
                                                       Progress notifications

ReadManager      None (no state)     (stateless)       Read-through to readers
(delegates)                                            Multiple readers strategy
```

