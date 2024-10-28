# Design Pinterest Newsfeed

## Requirements

A following graph, Owner-to-Board mappings, and Board-to-Pin mappings need to be stored and updated in real-time.

Pinterest had this information in our MySQL and HBase clusters, it takes too long to query them from those data stores for online applications (more than one round trip per request, big fanout and large data scanning).

These stateful services must provide not only KV data model, but also more complicated data models like counts, lists, etc. These stateful services must answer queries with single digit P99 latency given the latency requirement of the ML ranking system depending on them.

## Design in 2018, Improve from 2015

Board-to-Pin mappings and some lightweight Pin data are stored and real-time updated in Polaris, which not only supports board based Pin retrieval but also filtering with passed-in bloom filter as well as lightweight scoring to select high quality pins.

When a request is sent to Feed Generator to fetch a user’s following feed, it concurrently fetches real-time signals for the user, boards followed by the user, and Pins seen by the user recently from RealPin, Apiary and Aperture, respectively.

Note that Scorpion aggressively caches static feature data in local memory, which is the larger part of all feature data required by ML models. Scorpion is sharded to achieve a cache hit rate over 90%.

Pixie is a new service built for real-time board recommendation. It periodically loads into memory an offline-generated graph consisting of boards and Pins. When recommended boards are requested for a user, a random walk is simulated in the Pixie graph by using the Pins engaged by the user as starting points.

The new systems must achieve low long-tail latency with big fanout (sometimes all shard fanout), and some systems are CPU intensive (e.g., Scorpion, Pixie and RealPin). We chose to adopt C++11, FBThrift, Folly and RocksDB to build these systems.

RocksDB is an embedded storage engine. We started with the write-to-all-replica approach and later moved to master-slave replication. One thing to note is we needed to leverage the TCP_USER_TIMEOUT socket options to fail fast when the OS on remote peer crashed. More details about data replication and traffic routing can be found in the Rocksplicator Github repo. To reduce the operational overhead and service downtime, we integrated Apache Helix (a cluster management framework open sourced by Linkedin) with Rocksplicator. For instance, we needed to tune the RocksDB compaction thread number and set L0 and L1 to be of the same size to reduce write amplification and improve write throughput.

Though RealPin was designed as an object retrieval system, we customized it to run as an online scoring system for several months. However, we noticed it was a challenge to operate the system given it co-locates computation and storage. This issue became more serious as we started to use more types of feature data. Ultimately, we developed the new Scorpion system, which separates computation from storage and caches feature data on the computation nodes.

As Scorpion is CPU intensive and running on big clusters, we needed to invest heavily in optimizing it. We carefully tuned the Scorpion threading model to achieve a good tradeoff between high concurrent processing and low context switch or synchronization overhead. The sweet spot is not fixed, as it depends on many factors such as if data is fetched from memory, local disk or RPC. We optimized the in-memory LRU caching module to achieve zero-copy; i.e., cached data is fed into ML models without any data copying. Batch scoring was implemented to allow GCC to better utilize SIMD instructions. Decision trees in GBDT models are compacted and carefully laid out in memory to achieve better CPU cache hit rates. Thundering herds on cache miss are avoided by object level synchronization.

Data stored in Aperture is bucketed along the time dimension. We use a frozen data format for old data which is immutable and suitable for fast read access. More recent data is stored in a mutable format which is efficient for updates. The max_successive_merges RocksDB option was leveraged to limit the number of merge operands in mem-table from the same key. This setting is critical for Aperture to achieve low read latency since it may need to read a RocksDB key with a large number of merge operands, which are expensive to process at read time. To save storage space, RocksDB was configured to use different compression policies and multiplier factors for different levels.

## References

1. https://medium.com/pinterest-engineering/building-a-dynamic-and-responsive-pinterest-7d410e99f0a9
2. https://medium.com/pinterest-engineering/redesigning-pinterests-ad-serving-systems-with-zero-downtime-3253d2432a0c
2. https://medium.com/pinterest-engineering/structured-datastore-sds-multi-model-data-management-with-a-unified-serving-stack-e628f8081971
3. https://medium.com/pinterest-engineering/tidb-adoption-at-pinterest-1130ab787a10