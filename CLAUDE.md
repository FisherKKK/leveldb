# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About LevelDB

LevelDB is a fast key-value storage library written at Google that provides an ordered mapping from string keys to string values. This is a mature, production-quality codebase that receives very limited maintenance - only critical bug fixes and changes needed by internally supported clients are accepted.

**Key characteristics:**
- Written in C++11, follows Google C++ Style Guide
- No exceptions, no RTTI (both disabled in build)
- Public API is in `include/leveldb/*.h` - all other headers are internal
- Keys and values are arbitrary byte arrays stored sorted by key
- Data is compressed using Snappy (default) or Zstd

## Build Commands

### Initial Setup
```bash
# Clone with submodules (for GoogleTest, Google Benchmark)
git clone --recurse-submodules https://github.com/google/leveldb.git

# Standard build
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
```

### Testing
```bash
# From build directory
ctest --verbose

# Run specific test binary
./leveldb_tests

# Run individual test files (these are separate executables)
./c_test
./env_posix_test    # POSIX only
./env_windows_test  # Windows only
```

### Benchmarking
```bash
# From build directory
./db_bench           # Main LevelDB benchmarks
./db_bench_sqlite3   # SQLite comparison (if available)
./db_bench_tree_db   # Kyoto Cabinet comparison (if available)
```

### Code Formatting
```bash
# Format a single file
clang-format -i --style=file <file>

# Format all C++ files
find . -iname '*.cc' -o -iname '*.h' -o -iname '*.h.in' | xargs clang-format -i --style=file
```

## Architecture Overview

### Core Components

**DB Layer** (`db/`)
- `db_impl.cc/h`: Main database implementation (`DBImpl` class)
- `memtable.cc/h`: In-memory write buffer using skip list
- `skiplist.h`: Lock-free skip list (header-only template)
- `write_batch.cc`: Atomic batch of updates
- `version_set.cc/h`: Manages different versions of the database state
- `version_edit.cc/h`: Describes changes between versions
- `log_reader.cc/log_writer.cc`: Write-ahead log (WAL) handling
- `db_iter.cc/h`: Database iterator implementation
- `table_cache.cc/h`: Cache of open SSTable files

**Table Layer** (`table/`)
- `table.cc/table_builder.cc`: SSTable (Sorted String Table) reading/writing
- `block.cc/block_builder.cc`: Block-level storage within SSTables
- `format.cc/h`: SSTable file format structures
- `filter_block.cc/h`: Bloom filter blocks for fast lookups
- `merger.cc`: Merging iterator for multiple sorted sources
- `two_level_iterator.cc/h`: Iterator for two-level index structures

**Utilities** (`util/`)
- `env.cc`: OS abstraction layer interface
- `env_posix.cc`: POSIX implementation of Env
- `env_windows.cc`: Windows implementation of Env
- `cache.cc`: LRU cache implementation
- `bloom.cc`: Bloom filter implementation
- `arena.cc/h`: Simple arena allocator
- `coding.cc/h`: Encoding/decoding of integers (varint, fixed)
- `crc32c.cc/h`: CRC32C checksums
- `comparator.cc`: Key comparison functions

**Port Layer** (`port/`)
- `port.h`: Platform-specific types and functions
- `port_stdcxx.h`: Standard C++ implementation
- `thread_annotations.h`: Thread safety annotations

**Public API** (`include/leveldb/`)
- `db.h`: Main database interface - start here
- `options.h`: Configuration options for database and operations
- `write_batch.h`: Atomic write batching
- `iterator.h`: Iterator interface
- `comparator.h`: Custom key comparison
- `slice.h`: Non-owning byte array reference (like string_view)
- `status.h`: Error reporting
- `env.h`: OS environment abstraction
- `cache.h`: Cache interface
- `filter_policy.h`: Bloom filter policy
- `table.h/table_builder.h`: Low-level table interface

### Data Flow

**Write Path:**
1. Write goes to write-ahead log (WAL) file (*.log)
2. Write applied to in-memory MemTable (skip list)
3. When MemTable reaches ~4MB, it becomes immutable
4. Background thread writes MemTable to Level-0 SSTable (*.ldb)
5. Background compaction merges SSTables across levels

**Read Path:**
1. Check MemTable
2. Check immutable MemTable (if exists)
3. Check Level-0 SSTables (may overlap, must check all)
4. Check Level-1+ SSTables (binary search, non-overlapping within level)
5. Use Bloom filters to skip SSTables that don't contain the key
6. Use block cache to avoid re-reading/decompressing blocks

**LSM-Tree Structure:**
- Level-0: Up to 4 SSTables, may have overlapping key ranges
- Level-1: ~10MB total, non-overlapping SSTables
- Level-N: 10^N MB total, non-overlapping SSTables
- Compaction merges files from Level-L to Level-(L+1) when size threshold exceeded

### File Types

- `*.log`: Write-ahead log for durability
- `*.ldb`: SSTable file (sorted key-value pairs)
- `MANIFEST-*`: Lists all SSTables and their levels/key ranges
- `CURRENT`: Points to the current MANIFEST file
- `LOG`: Current info/error log file
- `LOG.old`: Previous log file
- `LOCK`: Prevents multiple processes from opening same DB

## Testing

Test files follow the pattern `*_test.cc`. Most tests use GoogleTest framework.

**Main test executable:** `leveldb_tests` in build directory runs most tests together when building shared libraries. When building static libraries, tests are more comprehensive and include:
- All db/ tests (db_test.cc, corruption_test.cc, etc.)
- All table/ tests
- All util/ tests

**Individual test executables:**
- `c_test`: Tests C API (`db/c_test.c`)
- `env_posix_test` or `env_windows_test`: Platform-specific environment tests

**Test utilities:**
- `util/testutil.cc/h`: Common test helpers
- Tests use temporary directories that are automatically cleaned up

## Code Style Notes

- Follows Google C++ Style Guide strictly
- Use `clang-format` with the provided `.clang-format` file before committing
- No exceptions: functions return `Status` objects for error handling
- No RTTI: avoid `dynamic_cast` and `typeid`
- Prefer `Slice` over `std::string` for string parameters to avoid copies
- Use `Status` not exceptions for error reporting
- Thread safety: Iterators and WriteBatch require external synchronization; DB object itself is thread-safe

## Important Development Notes

- **Never modify public API** (`include/leveldb/*.h`) without strong justification - API stability is critical
- **Tests are mandatory** for all changes
- **Platform support:** POSIX (Linux/macOS) and Windows only
- **External dependencies:** Optional support for Snappy, Zstd, CRC32C, TCMalloc
- C++11 required (configurable to C++14/17/20 for downstream users)
- Build system is CMake - changes to `CMakeLists.txt` are generally not accepted
- Comparator names are persisted in database files - changing comparator name requires migration
- The codebase disables C++ exceptions (`-fno-exceptions`) and RTTI (`-fno-rtti`)

## Common Patterns

**Opening a database:**
```cpp
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/path/to/db", &db);
if (!status.ok()) {
  // handle error
}
// ... use db ...
delete db;
```

**Status checking:**
```cpp
leveldb::Status s = db->Put(...);
if (!s.ok()) {
  std::cerr << s.ToString() << std::endl;
}
// Check specific errors:
if (s.IsNotFound()) { ... }
if (s.IsCorruption()) { ... }
if (s.IsIOError()) { ... }
```

**Atomic writes:**
```cpp
leveldb::WriteBatch batch;
batch.Delete(key1);
batch.Put(key2, value);
leveldb::Status s = db->Write(leveldb::WriteOptions(), &batch);
```

**Iteration:**
```cpp
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // it->key() and it->value() return Slice objects
}
assert(it->status().ok());
delete it;
```

## Documentation

- Main documentation: `doc/index.md`
- Implementation notes: `doc/impl.md`
- Table format: `doc/table_format.md`
- Log format: `doc/log_format.md`
