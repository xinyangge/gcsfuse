# GCSFuse Component Lifecycle Documentation

This directory contains comprehensive research and documentation of three critical components in the gcsfuse codebase:
- **FileHandle** - FUSE file handle abstraction
- **CacheHandle** - Local cache file coordination
- **FileInfo** - Cache metadata storage

## Documents Overview

### 1. COMPONENT_LIFECYCLE_ANALYSIS.md (27 KB, 783 lines)
**Comprehensive technical deep-dive covering all aspects of the three components.**

Best for: Understanding the complete lifecycle and interactions

Contents:
- Detailed structure definitions with field explanations
- Complete creation flow with code locations
- Lifecycle phases (creation → usage → destruction)
- Full method documentation
- Interaction flows with detailed steps
- State machines for all components
- Concurrency considerations
- Error handling patterns
- Configuration tuning guide
- Complete reference table to source code locations

Key Sections:
- Section 1: FileHandle (lines 1-150)
  - Definition, creation, triggering operations
  - Component interactions
  - Lifecycle phases
  - Key methods
- Section 2: CacheHandle (lines 151-300)
  - Definition, creation flow
  - Triggering operations
  - Component interactions with FileInfo and Jobs
  - Read operations coordination
- Section 3: FileInfo (lines 301-450)
  - Definition and key generation
  - Creation flow
  - LRU cache integration
  - State tracking
- Section 4: Complete interaction flows (lines 451-550)
  - Read flow diagrams
  - Cache eviction flows
  - Generation change scenarios
- Section 5-10: State machines, concurrency, errors, config, code locations, summary

### 2. COMPONENT_ARCHITECTURE_SUMMARY.md (31 KB, 622 lines)
**Visual diagrams and flowcharts showing architecture and state transitions.**

Best for: Visual learners, understanding at a glance

Contents:
- ASCII diagrams of component relationships
- Timeline of component lifecycles
- Data flow diagrams (sequential read example)
- State machine diagrams (4 different components)
- Memory layout diagrams
- Cleanup sequence diagrams
- Thread safety model table

Key Sections:
- Component Relationships Diagram (ASCII tree showing all connections)
- Component Lifecycle Timeline (time-based sequence)
- Data Flow - Sequential Read (detailed read operation flow)
- State Transitions for:
  - FileHandle state machine
  - CacheHandle state machine
  - FileInfo state machine
  - Download Job state machine
- Memory Layout showing object relationships
- Cleanup Sequence for LRU eviction
- Thread Safety Model (concurrency table)

### 3. COMPONENT_QUICK_REFERENCE.md (12 KB, 390 lines)
**Quick lookup guide for developers and debuggers.**

Best for: Quick reference while coding, debugging, or exploring

Contents:
- Summary comparison table
- File locations reference
- Key code patterns (code snippets)
- Important constants
- Common error scenarios with solutions
- Configuration impact guide
- Concurrency model overview
- Performance characteristics
- Testing entry points
- Debugging tips

Key Sections:
- Component Summary Table (quick comparison)
- File Locations Reference (where to find each component)
- Key Code Patterns (copy-paste ready examples)
- Constants Reference
- Common Error Scenarios (5 scenarios with solutions)
- Configuration Impact (how settings affect behavior)
- Concurrency Model
- Performance Characteristics (best/worst case)
- Testing Entry Points
- Debugging Tips

## Quick Navigation

### By Use Case

**I want to understand how FileHandle works:**
1. Start: COMPONENT_LIFECYCLE_ANALYSIS.md Section 1 (FileHandle)
2. Visual: COMPONENT_ARCHITECTURE_SUMMARY.md (FileHandle State Machine)
3. Reference: COMPONENT_QUICK_REFERENCE.md (File locations & patterns)

**I need to understand the read flow:**
1. Start: COMPONENT_LIFECYCLE_ANALYSIS.md Section 4 (Complete Read Flow)
2. Visual: COMPONENT_ARCHITECTURE_SUMMARY.md (Data Flow - Sequential Read)
3. Code: COMPONENT_QUICK_REFERENCE.md (Key code patterns)

**I'm debugging cache issues:**
1. Reference: COMPONENT_QUICK_REFERENCE.md (Error scenarios & debugging)
2. Details: COMPONENT_LIFECYCLE_ANALYSIS.md (Error handling section)
3. Visual: COMPONENT_ARCHITECTURE_SUMMARY.md (State machines)

**I need to understand component interactions:**
1. Visual: COMPONENT_ARCHITECTURE_SUMMARY.md (Component Relationships)
2. Details: COMPONENT_LIFECYCLE_ANALYSIS.md (Section 4: Interaction flows)
3. Reference: COMPONENT_QUICK_REFERENCE.md (Concurrency model)

**I'm working on memory/performance optimization:**
1. Reference: COMPONENT_QUICK_REFERENCE.md (Performance characteristics)
2. Details: COMPONENT_LIFECYCLE_ANALYSIS.md (Configuration tuning)
3. Code: COMPONENT_QUICK_REFERENCE.md (Memory usage breakdown)

### By Time Available

**5 minutes:**
- Read: COMPONENT_QUICK_REFERENCE.md (Component Summary Table)
- Read: COMPONENT_QUICK_REFERENCE.md (File Locations Reference)

**15 minutes:**
- Read: COMPONENT_ARCHITECTURE_SUMMARY.md (Component Relationships)
- Skim: COMPONENT_LIFECYCLE_ANALYSIS.md (Executive Summary + Sections 1-3)

**30 minutes:**
- Read: COMPONENT_LIFECYCLE_ANALYSIS.md (Executive Summary + Sections 1-3)
- View: COMPONENT_ARCHITECTURE_SUMMARY.md (State machines)
- Reference: COMPONENT_QUICK_REFERENCE.md (as needed)

**1 hour:**
- Full read: COMPONENT_LIFECYCLE_ANALYSIS.md (Sections 1-6)
- Full review: COMPONENT_ARCHITECTURE_SUMMARY.md (all diagrams)
- Reference: COMPONENT_QUICK_REFERENCE.md (detailed read)

**2+ hours:**
- Complete study of all three documents
- Cross-reference with actual source code

## Source Code File Locations

All key locations are documented in COMPONENT_QUICK_REFERENCE.md under "File Locations Reference"

Quick access:
- FileHandle: `/home/user/gcsfuse/internal/fs/handle/file.go` (lines 37-385)
- CacheHandle: `/home/user/gcsfuse/internal/cache/file/cache_handle.go` (lines 30-301)
- FileInfo: `/home/user/gcsfuse/internal/cache/data/file_info.go` (lines 26-61)

## Key Insights

### The Three-Tier Architecture
```
FUSE Layer (OpenFile/ReadFile/CloseFile)
    ↓
FileHandle (orchestration)
    ↓
ReadManager/FileCacheReader (read strategy)
    ↓
CacheHandle + Job (cache management)
    ↓
FileInfo (metadata) + Local cache file (storage)
```

### Critical Relationships
1. **FileHandle owns FileInode** - Registers/deregisters with it
2. **FileHandle creates Reader/ReadManager** - On first read, destroyed on invalidation
3. **Reader/ReadManager uses CacheHandler** - To get CacheHandle for reads
4. **CacheHandler creates FileInfo + Job** - On first cache access
5. **CacheHandle coordinates Job** - Waits for download, reads local file
6. **FileInfo tracks generation** - Detects when objects change in GCS
7. **Job manages download** - Downloads data to local file, updates progress

### Generation Awareness
All components track GCS object generation to detect external modifications:
- FileHandle: `reader.Object().Generation` vs `inode.SourceGeneration()`
- CacheHandle: `validateEntryInFileInfoCache()` checks generation match
- FileInfo: `ObjectGeneration` stored for each cached object

### Lifecycle Rule
- **Creation**: Most components created lazily on first access
- **Reuse**: Reused if still valid (same generation)
- **Invalidation**: Destroyed when generation changes or validation fails
- **Cleanup**: Cleanup happens in reverse order of creation

## Common Patterns

### Reading a File
1. Application calls `open()` → FileHandle created
2. Application calls `read()` → Reader/ReadManager created (first read only)
3. Reader tries FileCacheReader first
4. FileCacheReader creates CacheHandle (first cache read only)
5. CacheHandle.Read() coordinates Job download + local file read
6. Application calls `close()` → FileHandle destroyed

### Cache Eviction
1. LRU cache full, new FileInfo needs space
2. LRU evicts oldest entry
3. CacheHandler.cleanUpEvictedFile() called
4. Job.Invalidate() stops download, removes from JobManager
5. Local cache file deleted
6. FileInfo removed from LRU

### Generation Change
1. Object re-uploaded to GCS (new generation)
2. Reader detects generation mismatch: `isValidReader()` returns false
3. Old reader destroyed
4. New reader created with new generation
5. Similar flow for CacheHandle and FileInfo

## Testing Locations

- Unit tests: `/internal/{fs/handle,cache/file,cache/data}/` (with `_test.go` suffix)
- Integration tests: `/tools/integration_tests/`
- Mocks available: `fake.NewFakeBucket()`, `fake.FakeStorage`

## Concurrency Model

**Lock Hierarchy** (acquire in this order to prevent deadlock):
1. FileSystem.mu
2. FileInode.mu
3. FileHandle.mu
4. CacheHandler.mu
5. JobManager.mu
6. Job.mu

**Shared Resources:**
- One FileInode per file (shared across FileHandles)
- One CacheHandler per filesystem (shared across all reads)
- One LRU Cache per filesystem (shared across all cached objects)
- Jobs shared across CacheHandles (shared download progress)

## Performance Characteristics

**Best Case** (Sequential read, cache hit):
- Data downloaded asynchronously, read from local file
- O(download_time) latency

**Worst Case** (Random read, cache miss, no prefetch):
- Direct GCS read, no cache benefit
- O(gcs_latency) latency

**Memory Usage:**
- Per FileHandle: ~1-100 KB
- Per Job: ~10-1000 KB (download buffers)
- Per FileInfo: ~200 bytes + key size
- Aggregate: LRU cache limited by configuration

## Related Documentation

- GCSFuse architecture: `/docs/semantics.md`
- Configuration: `/cfg/` directory
- Read implementation: `/internal/gcsx/`
- Cache implementation: `/internal/cache/`

## Questions & Exploration

The documentation provides sufficient detail to answer:

**Component Structure:**
- What fields does each component have?
- What are invariants and preconditions?

**Creation:**
- When is each component created?
- What triggers creation?
- What parameters are passed?

**Lifecycle:**
- What states does each component go through?
- What operations trigger state transitions?
- What cleanup happens?

**Interactions:**
- How do the three components relate?
- What operations create dependencies between them?
- How do they communicate?

**Concurrency:**
- What is thread-safe?
- What locks are used?
- What is the lock ordering?

**Error Handling:**
- What can go wrong?
- How is it detected?
- How is it handled?

## Document Maintenance Notes

These documents were created from a thorough analysis of the gcsfuse codebase as of:
- Branch: `claude/document-handle-lifecycle-011CUwcPGBAGpwWZtivnsFxX`
- Recent commits analyzed for consistency

They document the **current implementation** as of analysis time.

For updates to these documents:
1. Check if the component structures have changed
2. Verify lifecycle operations haven't changed
3. Update state machines if new states added
4. Refresh code location line numbers
5. Add new error scenarios as they're discovered

---

**Total Documentation:** ~1,800 lines, 70 KB across 3 comprehensive documents

Enjoy exploring the gcsfuse architecture!
