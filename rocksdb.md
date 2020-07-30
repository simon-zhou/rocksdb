Based on the [talk](https://www.youtube.com/watch?v=V_C-T5S-w8g) from early RocksDB engineer, there are various issues with Leveldb:

1. Write throughput was not high enough, most likely because it cannot leverage multiple CPU cores. Similar issue for compaction because it's a single thread thing.
2. For certain applications, p99 latencies were tens of seconds, because of single-threaded flush, which sometimes conflicts with compaction. It seems that Leveldb is not designed for server workload. Maybe it's good for embedded library for applications other than databases. As a result of Leveldb's stalls, RocksDB implemented thread-aware compaction which reduced p99 latency to be less than a second.
  a) Dedicated thread(s) to flush memtable
  b) Pipelined memtables
3. High write amplification. For certain application, it can be as high as 70. As a result, RocksDB implemented Universal Style Compaction, which starts from newest file and include next file in candidate set if candidate set size >= size of next file. This reduces write amplification to < 10. RocksDB has monitoring to report the value and sources of write-amplification ([source](http://smalldatum.blogspot.com/2014/07/benchmarking-leveldb-family.html));
4. High read amplification for secondary index service, because Leveldb didn't use bloom filter for scans. RocksDB implemented prefix scans: a) range scans within same key prefix. b) blooms created for prefix. This reduced read amplification. (Note: Bloom filter was introduced into Leveldb in early 2012 in commit 85584d497e7b354853b72f450683d59fcf6b9c5c, version 1.4. Not sure why the talk says Leveldb doesn't use Blooms in late 2013. Maybe it means there was no Blooms for key prefix.)
5. Read modify write was not optimized in Leveldb, or pretty much any databases. Such operations need 2x IO which are expensive. RocksDB introduced MergeRecord, where compaction merges all MergeRecords.

Fundamentally, Leveldb cannot tune system and uses fixed file sizes. On the other hand, RocksDB focuses on pluggable compaction filter, eg, TimeToLive, pluggable memtable/sstable for RAM/flash and pluggable compaction algorithm.

Things that RocksDB inherits from Leveldb:
1. Log structured merge DB
2. Gets/Puts/Scans of keys
3. Forward and Reverse iteration

## [RocksDB Features that are not in LevelDB](https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB):
# Performance
Multithread compaction
Multithread memtable inserts
Reduced DB mutex holding
Optimized level-based compaction style and universal compaction style
Prefix bloom filter
Memtable bloom filter
Single bloom filter covering the whole SST file
Write lock optimization
Improved Iter::Prev() performance
Fewer comparator calls during SkipList searches
Allocate memtable memory using huge page.

# Features
Column Families
Transactions and WriteBatchWithIndex
Backup and Checkpoints
Merge Operators
Compaction Filters
RocksDB Java
Manual Compactions Run in Parallel with Automatic Compactions
Persistent Cache
Bulk loading
Forward Iterators/ Tailing iterator
Single delete
Delete files in range
Pin iterator key/value

# Alternative Data Structures And Formats
Plain Table format for memory-only use cases
Vector-based and hash-based memtable format
Clock-based cache (coming soon)
Pluggable information log
Annotate transaction log write with blob (for replication)

# Tunability
Rate limiting
Tunable Slowdown and Stop threshold
Option to keep all files open
Option to keep all index and bloom filter blocks in block cache
Multiple WAL recovery modes
Fadvise hints for readahead and to avoid caching in OS page cache
Option to pin indexes and bloom filters of L0 files in memory
More Compression Types: zlib, lz4, zstd
Compression Dictionary
Checksum Type: xxhash
Different level size multiplier and compression type for each level.

# Manageability
Statistics
Thread-local profiling
More commands in command-line tools
User-defined table properties
Event listeners
More DB Properties
Dynamic option changes
Get options from a string or map
Persistent options to option files

## References

[RocksDB: A High Performance Embedded Key-Value Store for Flash Storage - Data@Scale](https://www.youtube.com/watch?v=V_C-T5S-w8g)

[Benchmarking the leveldb family](http://smalldatum.blogspot.com/2014/07/benchmarking-leveldb-family.html)

[Comparing LevelDB and RocksDB, take 2](http://smalldatum.blogspot.com/2015/04/comparing-leveldb-and-rocksdb-take-2.html)
