# GCS Fuse Eviction Flow - Documentation Index

This index provides a guide to the comprehensive eviction flow documentation for the gcsfuse caching system.

---

## Document Overview

### 1. EVICTION_FLOW_DOCUMENTATION.md (Primary Document - 53 KB)
**Purpose**: Complete, detailed reference documentation on eviction

**Contains**:
- What gets evicted (FileInfo, cache files, download jobs)
- What triggers eviction (LRU, generation change, failed jobs, manual, prefix cleanup)
- Step-by-step eviction process (8 detailed steps with code)
- Component descriptions (LRU Cache, CacheHandler, JobManager, Job, CacheHandle, FileInfo)
- Interaction model (FileInfo, CacheHandle, FileHandle lifecycles)
- Cleanup process with error handling
- Eviction policies and configurations
- 10 edge cases with detailed handling
- 6 ASCII diagrams and sequence flows
- File references table

**Start here if**: You need complete understanding of the eviction system

---

### 2. EVICTION_QUICK_REFERENCE.md (Quick Guide)
**Purpose**: Fast reference for developers and debuggers

**Contains**:
- Quick summary (one paragraph)
- Table of what gets evicted
- Table of triggers
- Quick eviction sequence (4-step flow)
- Component overview table
- Configuration parameters table
- Interaction model diagrams
- 8-step cleanup process
- Edge cases table
- Test cases
- Critical code sections table
- Performance notes
- Quick debug tips

**Start here if**: You need quick answers about specific aspects

---

### 3. EVICTION_CODE_WALKTHROUGH.md (Line-by-Line Analysis)
**Purpose**: Detailed code walkthrough with actual repository code

**Contains**:
- 11 major code sections with full source
- Cache initialization explained
- Getting cache handle (trigger point)
- Adding entry (eviction decision)
- LRU cache insert algorithm
- LRU eviction algorithm
- Cleanup process (job invalidation)
- Job invalidation explained
- File truncation and deletion
- Complete eviction example with execution trace
- Concurrent access edge case
- Summary table of key sections

**Start here if**: You want to understand the actual code implementation

---

## File Locations in Repository

All documentation files are in the root directory:

```
/home/user/gcsfuse/
├── EVICTION_FLOW_DOCUMENTATION.md      (Primary - 53 KB)
├── EVICTION_QUICK_REFERENCE.md         (Reference - ~8 KB)
├── EVICTION_CODE_WALKTHROUGH.md        (Walkthrough - ~16 KB)
└── EVICTION_DOCUMENTATION_INDEX.md     (This file)
```

---

## Quick Navigation

### By Question

**"What gets evicted?"**
- Quick Reference → "What Gets Evicted" section
- Main Doc → Section 1: "What Gets Evicted"
- Code Walkthrough → Overview table

**"What triggers eviction?"**
- Quick Reference → "What Triggers Eviction" table
- Main Doc → Section 2: "What Triggers Eviction"
- Code Walkthrough → Section 3-4 (generation and job failure checks)

**"How does LRU work?"**
- Quick Reference → "Key Components" table
- Main Doc → "Components Involved" → "1. LRU Cache"
- Code Walkthrough → Section 5: "LRU Eviction Algorithm"

**"What happens during eviction?"**
- Quick Reference → "Quick Eviction Sequence"
- Main Doc → Section 3: "Eviction Process - Step by Step"
- Code Walkthrough → Section 10: "Complete Eviction Example"

**"How do FileHandle, CacheHandle, and FileInfo interact?"**
- Main Doc → Section 5: "Interaction with FileHandle, CacheHandle, and FileInfo"
- Code Walkthrough → Section 11: "Concurrent Access During Eviction"

**"What cleanup happens?"**
- Quick Reference → "Cleanup Process (8 Steps)"
- Main Doc → Section 6: "Cleanup Process"
- Code Walkthrough → Sections 6-9

**"What are edge cases?"**
- Quick Reference → "Edge Cases & Special Handling"
- Main Doc → Section 8: "Edge Cases and Special Handling"

**"Where's the code?"**
- Code Walkthrough → All sections have full code excerpts
- Quick Reference → "Critical Code Sections" table has file and line numbers

**"How do I debug eviction?"**
- Quick Reference → "Quick Debug Tips"
- Main Doc → "Edge Cases and Special Handling" has debugging info

---

## Key File References

### Core Eviction Files

| File | Purpose | Key Sections |
|------|---------|--------------|
| `/home/user/gcsfuse/internal/cache/lru/lru.go` | LRU cache algorithm | Insert (148-184), evictOne (128-139), EraseEntriesWithGivenPrefix (272-278) |
| `/home/user/gcsfuse/internal/cache/file/cache_handler.go` | Coordination and cleanup | GetCacheHandle (230-269), addFileInfoEntryAndCreateDownloadJob (140-217), cleanUpEvictedFile (109-129) |
| `/home/user/gcsfuse/internal/cache/file/cache_handle.go` | Read interface | Read (151-269), getFileInfoData (95-124), validateEntryInFileInfoCache (131-144) |
| `/home/user/gcsfuse/internal/cache/file/downloader/downloader.go` | Job management | InvalidateAndRemoveJob (143-153), Destroy (158-171) |
| `/home/user/gcsfuse/internal/cache/file/downloader/job.go` | Download job | Invalidate (195-212), updateStatusOffset (263-291) |
| `/home/user/gcsfuse/internal/cache/data/file_info.go` | Metadata | FileInfo struct (46-55), Size method (53-55) |
| `/home/user/gcsfuse/internal/cache/util/util.go` | Utilities | TruncateAndRemoveFile (171-183) |
| `/home/user/gcsfuse/internal/fs/fs.go` | Initialization | createFileCacheHandler (251-279) |
| `/home/user/gcsfuse/cfg/config.go` | Configuration | FileCacheConfig (413-437), MaxSizeMb (432) |

### Configuration Files

| File | Contains |
|------|----------|
| `cfg/config.go` | FileCacheConfig struct with all parameters |
| `cfg/rationalize.go` | Configuration validation |

### Test Files

| File | Purpose |
|------|---------|
| `internal/cache/lru/lru_test.go` | LRU eviction tests (ExpiresLeastRecentlyUsed, TestMultipleEviction, etc.) |
| `internal/cache/file/cache_handler_test.go` | CacheHandler tests |
| `internal/cache/file/cache_handle_test.go` | CacheHandle tests |

---

## Diagram References

### Main Documentation Contains

1. **Cache Data Structures** (Linked list + hash map visualization)
2. **FileInfo Cache Structure** (Data layout)
3. **Eviction Flow Sequence** (Complete flow diagram)
4. **Component Interaction Map** (System overview)
5. **State Transitions** (Job and FileInfo states)
6. **Time-Based Eviction Trigger** (Cache size evolution)

---

## Configuration Parameters

### File Cache Configuration

Parameter | Default | Meaning
----------|---------|--------
`max-size-mb` | User-configured | Cache size limit in MiB (-1 = unlimited)
`exclude-regex` | Empty | Pattern to exclude files from cache
`include-regex` | Empty | Pattern to include files in cache
`cache-file-for-range-read` | true | Whether to cache range reads
`download-chunk-size-mb` | Configured | Size of download chunks
`max-parallel-downloads` | math.MaxInt64 | Concurrent download limit
`enable-crc` | false | Whether to verify CRC

**Location**: `/home/user/gcsfuse/cfg/config.go` lines 413-437

---

## Key Concepts

### Eviction Triggers (Multiple)

1. **LRU**: When cache size exceeds maxSize
2. **Generation**: When object has new version in GCS
3. **Job Failure**: When download job fails or is invalidated
4. **Manual**: When InvalidateCache() is called
5. **Prefix**: When EraseEntriesWithGivenPrefix() is called

### Data Structures

- **LRU Cache**: Doubly-linked list + hash map (O(1) operations)
- **FileInfo**: Metadata (generation, offset, size)
- **CacheHandle**: Read interface to cached file
- **Job**: Download state machine (5 states)

### Synchronization

- LRU Cache: RWLocker (read-write synchronization)
- CacheHandler: Locker (mutual exclusion)
- JobManager: Locker (mutual exclusion)
- Job: Locker with invariant checking

### Special Patterns

- **Deadlock Prevention**: Careful lock release ordering in JobManager.InvalidateAndRemoveJob()
- **Two-Phase Cleanup**: Truncate (frees space immediately) then remove
- **Idempotent Cleanup**: "File not found" treated as success
- **Error Recovery**: Partial failures handled gracefully

---

## Testing

### Test Examples

The `lru_test.go` file contains important test cases:

- `ExpiresLeastRecentlyUsed()` - Basic LRU eviction
- `TestMultipleEviction()` - Multiple entries evicted at once
- `Overwrite()` - Update with size change triggers eviction
- `TestEraseCacheWithGivenPrefix()` - Batch deletion
- `TestWhenEntrySizeMoreThanCacheMaxSize()` - Oversized entry rejection

### Running Tests

```bash
cd /home/user/gcsfuse
go test ./internal/cache/lru -v
go test ./internal/cache/file -v
```

---

## Performance Considerations

### Time Complexity

- LRU operations: **O(1)** (doubly-linked list + hash map)
- Eviction cost: **O(n)** where n = number of entries evicted
- File cleanup: **O(1)** per file

### Space Complexity

- Overhead per entry: Fixed (pointers + metadata)
- Total cache: **O(maxSize)** - bounded by configuration

### Lock Contention

- Multiple locks in system (LRU, Handler, Manager, Job)
- Deadlock prevention via careful ordering
- No busy-waiting

---

## Debugging Guide

### Common Issues

**Cache not evicting**
- Check `max-size-mb` configuration
- Verify cache size calculation
- Monitor currentSize vs maxSize

**Files not deleted**
- Check file permissions
- Verify disk space availability
- Look for open file handles

**Generation mismatches**
- Compare ObjectGeneration vs current object
- Check GCS object updates
- Verify bucket

**Job failures**
- Check job status (Failed vs Invalid)
- Monitor download logs
- Verify network connectivity

### Debug Information to Collect

1. Cache metrics (currentSize, maxSize, entry count)
2. Job statuses (Downloading, Failed, Invalid)
3. File existence checks
4. Lock contention (if applicable)
5. Eviction logs

---

## Related Documentation

See also in the repository:

- `COMPONENT_LIFECYCLE_ANALYSIS.md` - Detailed lifecycle analysis
- `COMPONENT_ARCHITECTURE_SUMMARY.md` - System architecture
- `COMPONENT_QUICK_REFERENCE.md` - Component overview
- `COMPONENT_DOCUMENTATION_README.md` - Component docs index

---

## Summary

The eviction flow in gcsfuse is a well-designed system that:

1. **Uses LRU eviction** for optimal cache replacement
2. **Supports multiple triggers** (size, generation, job failure, manual, prefix)
3. **Maintains consistency** through careful synchronization
4. **Prevents deadlocks** through strategic lock ordering
5. **Handles edge cases** gracefully (open handles, partial failures)
6. **Provides configuration** flexibility (size limits, regex filters)

The system cleanly separates concerns:
- **LRU Cache** handles eviction algorithm
- **CacheHandler** coordinates all operations
- **JobManager** manages download jobs
- **Job** handles individual downloads
- **CacheHandle** provides read interface
- **FileInfo** stores metadata

---

## Quick Links

- **Complete Documentation**: EVICTION_FLOW_DOCUMENTATION.md
- **Quick Reference**: EVICTION_QUICK_REFERENCE.md
- **Code Walkthrough**: EVICTION_CODE_WALKTHROUGH.md
- **Source Code**: `/home/user/gcsfuse/internal/cache/`

---

Last Updated: 2025-11-09
Repository: gcsfuse
Branch: claude/document-handle-lifecycle-011CUwcPGBAGpwWZtivnsFxX

