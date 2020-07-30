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

## References

[RocksDB: A High Performance Embedded Key-Value Store for Flash Storage - Data@Scale](https://www.youtube.com/watch?v=V_C-T5S-w8g)

[Benchmarking the leveldb family](http://smalldatum.blogspot.com/2014/07/benchmarking-leveldb-family.html)

[Comparing LevelDB and RocksDB, take 2](http://smalldatum.blogspot.com/2015/04/comparing-leveldb-and-rocksdb-take-2.html)
