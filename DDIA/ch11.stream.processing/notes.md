# Chapter 11 Stream Processing

**Assumption for batch processing: the input is bounded**, i.e., of a known and finite size, so the batch process knows when finished reading input. For example, sorting that is central to MapReduce. It could happen that the last input record is the lowest key and needs to be the very first output record. Starting the output early is not an option.

In reality, a lot of data is unbounded because it arrives gradually over time. Thus, batch processors must artificially divide the data into chunks of fixed duration: for example, processing a day’s worth of data at the end of every day.

To reduce the delay for daily batch processing, we can run processing more frequently--say, every second--or even continuously. This is the idea befind stream processing.

In general, a “stream” refers to data that is incrementally made available over time. The concept appears in many places: in the stdin and stdout of Unix, programming languages (lazy lists: infinite streaming, lazy evaluation, e.g. Haskell `is.tl` and `is.take`, Python `generators`(`yield`)), filesystem APIs (such as Java’s FileInputStream), TCP connections, delivering audio and video over the internet, and so on.

## 11.1 Transmitting Event Streams

In both streaming and batch processing worlds, input data essentially is a small, self contained, immutable object containing the details of something that happened at some point in time. The input data could be a file for batch processing and an event for streaming.

In a filesystem, a filename identifies a set of related records; in a streaming system, related events are usually grouped together into a **topic or stream**.

Moving toward continual processing with low delays, polling becomes expensive if datastore is not designed for this kind of usage. The more often you poll, the lower the percentage of new events, and thus the higher overheads. Instead, it is better for consumers to be notified when new events appear.

Databases have traditionally **not supported notification mechanism well**: relational databases commonly have triggers, which can react to a change (e.g., a row being inserted into a table), but they are very limited in what they can do and have been an **afterthought** in database design. Instead, specialized tools have been developed for event notifications.

### 11.1.1 Messaging Systems

A simple way to implement could be a Unix pipe or TCP connection between producer and consumer. That connects one sender with one recipient. Most messaging systems expand on this basic model. To differentiate, following two questions can be asked:

1. What happens if the producers send messages faster than the consumer can process?
    1. drop
    1. buffer in a queue. What happens as the queue grows?
        1. system crash if out of memory
        1. write messages to disk. How does disk access affect the performance?
    1. apply backpressure(flow control, blocking producer from sending). **Unix pipes and TCP use backpressure**.
1. What happens if nodes crash or temporarily go offline-are any messages lost? Durability may require some combination of disk writing and/or replication. If can afford to sometimes lose messages, can get higher throughput and lower latency on the same hardware.

Whether message loss is acceptable depends on application. Periodic metrics might be ok to miss a few data points, but counter will be off.

Batch processing system provide a strong reliability with retry and discarding partial failures. Output is same as if no failures occured, which simplify the programming model.

#### 11.1.1.1 Direct messaging from producers to consumers

1. **UDP multicast** is widely used in the financial industry for streaming stock feeds, where low latency is important. Application level protocols can recover lost packets (producer remember packets so that it can retransmit).
1. Brokerless libraries such as ZeroMQ and nanomsg pub/sub over TCP/IP multicast.
1. StatsD and Brubeck use UDP messaging for collecting metrics. (In StatsD, counter metrics are only correct if all messages are received. Using UDP makes the metrics at best approximate.)
1. If the consumer exposes a service on the network, producers can make a direct HTTP or RPC request, which is the idea behind **webhooks**, a pattern in which a callback URL of one service is registered with another service.

Require application code to be aware of message loss. Can tolerate limited faults. Generally assume producers and consumers are constantly online.

#### 11.1.1.2 Message Broker (Queue)

Essentially a kind of database optimized for handling message streams. More easily tolerate clients that come and go (connect, disconnct, and crash). Some only keep messages in memory, others write to disk. They generally allow **unbounded queueing** as opposed to dropping message or backpressure.

Consumers are generally asynchronous considering that the producer does not wait for messaging processing.

#### 11.1.1.3 Message Brokers compared to databases

Some message brokers can even participate in two-phase commit protocols using XA (eXtended Architecture) or JTA (Java Transaction API).

Important practical differences:

1. Message broker automatically delete a message, thus not suitable for long-term storage. Databases keep until explicitly deleted.
1. Message brokers assume the working set is small. If broker buffer a lot of messages, perhaps spilling messages to disk if they no longer fit in memory, each individual message takes longer to process and the overall throughput may degrade.
1. Databases often support secondary indexes and various of searching, while message brokers often support subscribing to a subset of topics matching some pattern. Both are essentially ways for a client to select portion of the data.
1. Database query result is based on a point-in-time snapshot of the data. Message brokers do not support arbitrary quries, but notify clients when data changes.

The above traditional view of message brokers are in standards like JMS (JSR-343 Java Message Service 2.0 specification) and AMQP (Advanced Message Queuing Protocol Specification) and implemented in software like RabbitMQ, ActiveMQ, HornetQ, Qpid, TIBCO enterprise message service, IBM MQ, Azure service bugs, and google cloud pub/sub.

#### 11.1.1.4 Multiple consumers

When multiple consumers read messages in the same topic, two main patterns are used.

1. Load balancing. Each message is delivered to one of the consumers. The broker may assign messages to consumers arbitrarily. Useful when processing is expensive and can add consumers to parallelize. In AMQP, you can implement load balancing by having multiple clients consuming from the same queue, and in JMS it is called a shared subscription.
1. Fan out. Each message is delivered to all of the consumers. This feature is provided by topic subscriptions in JMS, and exchange bindings in AMQP.

![](./11-1.load.bal.fan.out.png)

The two patterns can be combined. Two separate groups, each group collectively receives all messages but within each group only one node receives each message.

#### 11.1.1.5 Acknowledgements and redelivery

Broker can use ack to avoid processing failures or partial processing.

It is possible that the message was fully processed but the ack was lost in the network. Handling this requires an atomic commit protocol.

The combination of load balancing and redelivery leads to messages being reordered. To avoid, you can use a separate queue per consumer(i.e., not use the load balancing feature). Message reordering is not a problem if messages are completely independent of each other, but it can be important if there are causal dependencies between messages, as we shall see later in the chapter.

### 11.1.2 Partitioned Logs

Why can we not have a hybrid, combining the durable storage approach of databases with the low-latency notification facilities of messaging? This is the idea behind log-based message brokers.

#### 11.1.2.1 Using logs for message storage

The Unix tool `tail -f`, which watches a file for data being appended, essentially works like this.

![](./11-3.log.partition.png)

The log can be partitioned. Different partitions can then be hosted on different machines, making each partition a separate log that can be read and written independently from other partitions. A topic can then be defined as a group of partitions that all carry messages of the same type.

Within each partition, the broker assigns a monotonically increasing sequence number, or offset, to every message. Such a sequence number makes sense because a partition is append-only, so the messages within a partition are totally ordered. There is no ordering guarantee across different partitions.

**Apache Kafka, Amazon Kinesis Streams, and Twitter’s DistributedLog** are log-based message brokers. Google Cloud Pub/Sub is architecturally similar but exposes a JMS-style API rather than a log abstraction. Even though these message brokers write all messages to disk, they are able to achieve throughput of millions of messages per second by partitioning across multiple machines, and fault tolerance by replicating messages.

#### 11.1.2.2 Logs compared to traditional messaging

The log-based approach trivially supports fan-out messaging, because several consumers can independently read the log without affecting each other—reading a message does not delete it from the log. To achieve load balancing across a group of consumers, instead of assigning individual messages to consumer clients, the broker can assign entire partitions to nodes in the consumer group.

Each client then consumes all the messages in the partitions it has been assigned. Typically, when a consumer has been assigned a log partition, it reads the messages in the partition sequentially, in a straightforward single-threaded manner. This coarse grained load balancing approach has some downsides:

1. The number of nodes sharing the work of consuming a topic can be at most the number of log partitions in that topic, because messages within the same partition are delivered to the same node.(In general, single-threaded processing of a partition is preferable, and parallelism can be increased by using more partitions.)
1. If a single message is slow to process, it holds up the processing of subsequent messages in that partition (a form of head-of-line blocking).

Thus, in situations where messages may be expensive to process and you want to parallelize processing on a **message-by-message basis**, and where message ordering is not so important, the JMS/AMQP style of message broker is preferable (SQS no order). On the other hand, in situations with high message throughput, where **each message is fast to process** and where message ordering is important (SQS FIFO?), the log-based approach works very well.

#### 11.1.2.3 Consumer offsets

With periodically marking an offset while consuming a partition sequentially, the broker does not need to track acknowledgments for every single message. **The reduced bookkeeping, opportunities for batching and pipelining** help increase the throughput for log-based systems.

This offset is similar to the log sequence number that is commonly found in single-leader database replication. In database replication, the log sequence number allows a follower to reconnect and resume replication without skipping any writes. Exactly the same principle is used here: the message broker behaves like a leader database, and the consumer like a follower.

If a consumer node fails, another node in the consumer group is assigned the failed consumer’s partitions, and it starts consuming messages at the last recorded offset. If the consumer had processed subsequent messages but not yet recorded their offset, those messages will be processed a second time upon restart.

#### 11.1.2.4 Disk space usage

Log is divided into segments and deleted or moved to archive from time to time to reclaim disk space.

A slow consumer may fall far behind that the offset **points to a deleted segment** and may miss some messages. Effectively, the log implement a bounded-size buffer when it gets full, a circular buffer or ring buffer. The buffer is on disk and can be very large.

A typical large hard drive has a capacity of 6 TB and a sequential **write throughput of 150 Mb/s**. It takes about 11 hours to fill. Thus, the disk can buffer 11 hours' worth of messages. This ratio remains the same even if you use many drives and machines. In practice, deployments rarely use the full write bandwidth, so the log can typically keep a buffer of several days or weeks.

The throughput of a log remains more or less constant, since every message is written to disk anyway. In contrast to keeping messages in memory by default: such systems are **fast when queues are short and much slower when they start writing to disk**.

#### 11.1.2.5 When consumers cannot keep up with producers

We discussed three choices (dropping messages, buffering, or applying backpressure) previously. The log-based approach is a form of buffering with a large fixed-size buffer (available disk space).

If a consumer falls so far behind, the broker effectively drops old messages that go back further than buffer size can accomodate. You can mointor how far behind and raise an alert. As the buffer is large, there is enough time for a human operator to fix the slow consumer.

Even if a consumer does fall too far behind, only that consumer is affected; it does not disrupt other consumers. This fact is a big operational **advantage**: you can experimentally consume a production log for development, testing, or debugging purposes, without having to worry much about disrupting production services. When a consumer is shut down or crashes, it stops consuming resources—the only thing that remains is its consumer offset.

This behavior also **contrasts** with traditional message brokers, where you need to be careful to delete any queues whose consumers have been shut down—otherwise they continue unnecessarily accumulating messages and taking away memory from consumers that are still active.

#### 11.1.2.6 Replaying old messages

Processing and acknowledging messages

1. is a destructive operation for AMQP-and JMS-style message brokers, since it causes the messages to be deleted on the broker.
1. in a log-based message broker, consuming messages is more like reading from a file: it is a read-only operation that does not change the log.

The offset is under the consumer’s control, so it can easily be manipulated if necessary: for example, you can start a copy of a consumer with yesterday’s offsets and write the output to a different location, in order to reprocess the last day’s worth of messages. You can repeat this any number of times, varying the processing code.

This aspect makes log-based messaging more like the batch processes. It allows **more experimentation and easier recovery** from errors and bugs, making it a good tool for integrating dataflows within an organization.

## 11.2 Databases and Streams

The connection between databases and streams run deeper than just the physical storage of logs on disk--it is quite fundamental.

**A replication log is a stream of databse writes**, produced by the leader as it processes transactions. The followers apply the stream to their own copy and end up with an accurate copy.

### 11.2.1 Keeping Systems in Sync

There is no single system satisfying all data storage, querying, and processing needs. For example, using an OLTP (online transaction processing) DB to serve user requests, a cache to speed up common requests, a full-text index to handle search queries, and a data warehouse for analytics. Each has its own copy of data stored in its own representation that is optimized for its own purposes.

Data need to be kept in sync. With data warehouses this is performed by ETL processes, a batch process.

If periodic full dump is too slow, an alternative is **dual writes**, in which application code explicitly writes to each, e.g., first writing to DB, then updating the search index, then invalidating the cache (or write concurrently).

Dual writes have some problems, one of which is race condition as illustrated in Figure 11-4. Client 1 wants to set the value to A, and client 2 wants to set it to B.

![](./11-4.race.condition.png)

Another possibility is that one write may fail and the other succeeds. This is a fault-tolerance problem but two systems are inconsistent, which is a case of the atomic commit pronlem and is expensive to solve (see “Atomic Commit and Two-Phase Commit (2PC)”). In Figure 11-4, the database has a leader and the search index has a leader as well. Neither follows the other so conflicts can occur.

### 11.2.2 Change Data Capture

Databases' replication logs are considered internal implementation and not a public API, thus not intended to be parsed by external clients.

#### 11.2.2.1 Implementing change data capture

Change data capture is a mechanism for ensuring that all changes made to the system of record are also reflected in the derived data systems so that the derived systems have an accurate copy of the data. A log-based message broker is well suited since it preserves the ordering of messages.

Database **triggers** can be used to implement change data capture by registering triggers that observe all changes to data tables and add corresponding entries to a changelog table. However, they tend to be **fragile and have significant performance overheads**. **Parsing the replication log** can be a more robust approach, although it also comes with challenges, such as handling schema changes.

LinkedIn’s [Databus](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying), Facebook’s [Wormhole](https://www.usenix.org/system/files/conference/nsdi15/nsdi15-paper-sharma.pdf), and Yahoo!’s [Sherpa](http://web.archive.org/web/20160801221400/https://developer.yahoo.com/blogs/ydn/sherpa-7992.html) use this idea at large scale. [Bottled Water](https://github.com/confluentinc/bottledwater-pg) implements CDC for PostgreSQL using an [API](https://martin.kleppmann.com/2015/04/23/bottled-water-real-time-postgresql-kafka.html) that decodes the write-ahead log, [Maxwell](https://maxwells-daemon.io/) and [Debezium](https://debezium.io/) do something similar for MySQL by parsing the binlog, [Mongoriver](https://github.com/stripe-archive/mongoriver) reads the MongoDB [oplog](https://www.slideshare.net/danharvey/change-data-capture-with-mongodb-and-kafka), and GoldenGate provides similar facilities for Oracle.

Like message brokers, change data capture is usually **asynchronous**: the system of record database does not wait for the change to be applied to consumers before committing it. This design has the operational advantage that adding a slow consumer does not affect the system of record too much, but it has the downside that all the issues of replication lag apply.

#### 11.2.2.2 Initial snapshot

The snapshot of the database must correspond to a known position or offset in the change log, so that you know at which point to start applying changes after the snapshot has been processed. Some CDC tools integrate this snapshot facility, while others leave it as a manual operation.

#### 11.2.2.3 Log compaction

If you can only keep a limited amount of log history, you need to go through the snapshot process every time you want to add a new derived data system. However, log compaction provides a good alternative.

The **principle** is simple: the storage engine periodically looks for log records with the same key, throws away any duplicates, and keeps only the most recent update for each key. This compaction and merging process runs in the background.

In a log-structured storage engine, an update with a special null value (a **tombstone**) indicates that a key was deleted, and causes it to be removed during log compaction.

The same idea works in the context of log-based message brokers and change data capture. If the CDC system is set up such that every change has a primary key, and every update for a key replaces the previous value for that key, then it’s sufficient to keep just the most recent write for a particular key.

Now, whenever you want to rebuild a derived data system such as a search index, you can start a new consumer from offset 0 of the log-compacted topic, and sequentially scan over all messages in the log. The log is guaranteed to contain the most recent value for every key in the database (and maybe some older values)—in other words, you can use it to obtain a full copy of the database contents without having to take another snapshot of the CDC source database.

This **log compaction feature is supported by Apache Kafka**. As we shall see later in this chapter, it allows the message broker to be used for **durable storage**, not just for transient messaging.

#### 11.2.2.4 API support for change streams

Databases are beginning to support change streams as **first-class interface**, e.g., **RethinkDB** allows notification subscription when query result change, Firebase and CouchDB provide data sync based on a change feed also available to applications, and Meteor uses the **MongoDB** oplog to subscribe to data changes and update UI.

**VoltDB** represents an output stream in the relational data model as a table into which transactions can insert tuples, but cannot be queried. The stream consists of the log of tuples of the committed transactions, in the order they were committed. External consumers can asynchronously consume this log and use it to update derived data systems.

**Kafka Connect** is an effort to integrate change data capture tools for a wide range of database systems with Kafka.

### 11.2.3 Event Sourcing

There are some parallels between CDC and event sourcing, which was developed in the domain-driven design (DDD) community. Simlarly **event sourcing also stores changes** as a log of change events.

The biggest difference is that event sourcing applies the idea at a different level of abstraction:

1. In CDC, application uses DB in a mutable way. The log of changes is extracted from DB at a low level (e.g., parsing the replication log) to ensure the order of writes and avoid race condition. The application does not need to be aware of CDC.
1. In event sourcing, application logic is built on **immutable events** written to an event log. The event store is append-only. Updates or deletes are discouraged or prohibited. Events are designed to reflect things happened at the application level, rather than state changes.

Event sourcing makes it easier to evolve applications over time, to understand why something happended, and guards against application bugs.

Event sourcing is similar to the chronicle data model, and there are also similarities between an event log and the fact table that you find in a star schema (see “Stars and Snowflakes: Schemas for Analytics”).

**Specialized DB such as Event Store** have been developed to support applications using event sourcing. A conventional DB or a log-based message broker can be used to build applications in this style.

#### 11.2.3.1 Deriving current state from the event log

Applications that use event sourcing can take the log of events and transform into application state. The transformation logic can be arbitrary but deterministic.

Like CDC, replaying event log reconstruct the current state of the system. However, log compaction needs to be handled differently:

1. A CDC event for update contains the entire new version of the record and log compaction can discard previous events for the same key.
1. On the other hand, with event sourcing, events are modeled at a higher level: an event typically expresses the intent of a user action, not the mechanics of the state update that occurred as a result of the action. In this case, later events typically do not override prior events, and so you need the full history of events to reconstruct the final state. **Log compaction is not possible in the same way**.

#### 11.2.3.2 Commands and events

With event sourcing, a user issues a command which may fail. The application must validate **synchronously**. If the validation succeeds, the command becomes an event, which is durable and immutable. For example, if a user tries to reserve a seat.

A consumer of the event stream is not allowed to reject an event: it is immutable part of the log and may have been seen by other consumers.

Alternatively the reservation could be split into two events: first a tentative reservation, and then a seprate confirmation event once the reservation has been validated. The split allows the validation process **asynchronous**.

### 11.2.4 State, Streams, and Immutability

From Pat Helland,

>Transaction logs record all the changes made to the database. High-speed appends are the only way to change the log. From this perspective, the contents of the database hold a caching of the latest record values in the logs. The truth is the log. The database is a cache of a subset of the log. That cached subset happens to be the latest value of each record and index value from the log.

Log compaction is one way of bridging the distinction between log and DB state: it retains only the latest version of each record.

#### 11.2.4.1 Advantages of immutable events

Immutability in databases is an old idea. For example, **accountants** have been using immutability for centuries in financial bookkeeping. When a transaction occurs, it is recorded in an append-only ledger, which is essentially a log of events describing money, goods, or services that have changed hands. The accounts, such as profit and loss or the balance sheet, are derived from the transactions in the ledger by adding them up.

The incorrect transaction still remains in the ledger forever, because it might be important for auditing reasons.

Immutable events also capture more information than just the current state. Customer adds to cart and remove later. The information might be useful for analytics. Perhaps the customer found a substitute.

#### 11.2.4.2 Deriving several views from the same event log

One can derive **several different read-oriented representations** from the same log of events, just like having multiple consumers of a stream. For example, Druid ingests directly from Kafka, Pistachio is a distributed key-value store using kafka as a commit log, Kafka Connect sinks exports to various databases and indexes. It would make sense for many other storage and indexing systems, such as search servers, to similarly take their input from a distributed log (see “Keeping Systems in Sync”).

Having an explicit translation step from an event log helps application evolution over time. If you want to introduce a new feature, you can build a separate read-optimized view of the new feature and run alongside existing systems. It is **much easier than schema migration**.

Storing data is normally quite straightforward if you don’t have to worry about how it is going to be queried and accessed; many of the complexities of schema design, indexing, and storage engines are the **result of** wanting to support certain query and access patterns (see Chapter 3). For this reason, you gain a lot of flexibility by separating the form in which data is written from the form it is read, and by allowing several different read views. This idea is sometimes known as command query responsibility segregation (CQRS).

On page 11, for twitter's home timelines, a cache of recently written tweets by people that a user follows. This is another example of read-optimized state: home timelines are highly denormalized, since your tweets are duplicated in all the timelines of the people following you. However, the fan-out service keeps the duplicated state in sync with new tweets and new following relationships, which keeps the duplication manageable.

#### 11.2.4.3 Concurrency control

The biggest **downside** of event sourcing and change data capture is that the consumers of the event log are usually async, we discussed previously "Reading Your Own Writes".

One solution is to perform the updates synchronously with appending the event to the log. This requries a transaction to combine the writes into an atomic unit. So either you keep the event log and the read view in the same storage system, or use a **distributed transaction** across different systems.

On the other hand, much of the need for multi-object transactions stem from a single user action requiring data changes in different places. With event sourcing, the event can be self-contained requiring a single write--appending log, which is easy to make atomic.

If processing an event in partition 3 only requires updating partition 3 of the application state, a straightforward single-threaded consumer needs no concurrency control for writes. The log removes the nondeterminism of concurrency by defining a serial order of events in a partition.

#### 11.2.4.4 Limitations of immutability

Various databases internally use immutables or multi-version data to support point-in-time snapshots. Version control systems such as Git, Mercurial, and Fossile also rely on immutable data.

To what extent is it feasible to keep an immutable history forever? The answer depends on the amout of churn in the dataset.Some workloads mostly add data and rarely update or delete. They are easy to make immutable.Other workloads have a high rate of updates and deletes on a small dataset. The **immutable history may grow prohibitiely large**, **fragmentation** may become an issue, and compaction and garbage collection becomes crucial.

Data may need to be deleted for **admin reasons**, e.g., privacy regulations may require deleting user's personal info after account closing, data protection legislation may require erroneous info to be removed, or an accidental leak of sensitive info may need to be contained. In these cases, you acctually want to rewrite history and pretend the data was never written. Datomic calls this feature excision and Fossil has a similar concept called shunning.

Truly deleting is **surprisingly hard**, e.g., storage engines, file systems, and SSDs often write to a new location rather than overriting in place, and backups are often deliberately immutable to prevent accidental deletion or corruption.

## 11.3 Processing Streams

Streams come from (user activity events, sensors, and writes to DBs) and are transported through direct messaging, via message brokers, and in event logs.

What you can do with the stream

1. Take the data in the events and write to a DB, cache, search index, or similar storage system. This is a good way to keep a DB in sync with other parts of the system.
1. Push the events to users in some way, e.g., sending email or push notifications, or by streaming to a real-time dashboard. In this case, a human is the ultimate consumer.
1. Process one or more streams to produce one or more output streams. Streams may go through a pipeline consisting several such stages before they eventually end up at an output (option 1 or 2).

A piece of code processing streams is known as **an operator or a job**, closely related to Unix processes and MapReduce jobs. Pattern is similar: consume in a read-only fashion and writes to a different location in an append-only fashion.

The patterns for partitioning, parallelization, transforming, and filtering are also similar to those in MapReduce and dataflow engines.

The one crucial difference is that a stream never ends. **Implications**: sorting does not make sense, so sort-merge joins (p403) cannot be used. Fault-tolerance mechanisms must also change: for a stream job running for several years, restarting from beginning after a crash may not be a viable option.

### 11.3.1 Uses of Stream Processing

Stream processing has long been used for monitoring. For **example**:

1. Fraud detection systems block a credit card if it is likely to be stolen.
1. Trading systems to examine price changes and execute trades.
1. Manufacturing systems to monitor machine statuses and quickly identify malfunctions.
1. Military and intelligence systems to track aggressor activities.

These applications require sophisticated pattern matching and correlations. However, other uses of stream processing have also emerged over time. In this section we will briefly compare and contrast some of these applications.

#### 11.3.1.1 Complex event processing

Complex event processing (CEP) was developed in 1990s especialy geared toward searching for certain event patterns. Similar to a regular expression, CEP allows to specify rules to search for patterns of events in a stream.

CEP often use a high-level declarative query language like SQL, or a gaphical UI, to describe the patterns. The queries are submitted to a processing engine that consumes the input streams and maintains a state machine. If a match is found, a complex event with details of the pattern detected will be emitted.

CEP engines **reverse the roles** compared to databases. The queries are stored long-term and events flow past. Usually a database stores data persistently and treat queries as transient.

Implementations of CEP include Esper, IBM InfoSphere Streams, Apama, TIBCO streambase, and SQLstream. Distributed stream processors like Samza are also gaining SQL support for declarative queries on streams, **AWS EventBridge rules, AWS Kinesis**.

#### 11.3.1.2 Stream analytics

The boundary between CEP and stream analytics is blurry, but as a general rule, analytics is more oriented toward aggregations and statistical metrics.

1. Measuring the rate of some type of event
1. calculating the rolling average of a value over time
1. comparing current statistics to previous time intervals

Such statistics are usually computed over fixed time intervals, e.g., QPS over the last 5 minutes, and p99 of response time. The aggregation time interval is known as a window.

Stream analytics systems sometimes use **probabilistic algorithms**, such as Bloom filters (which we encountered in “Performance optimizations”) for set membership, HyperLogLog for cardinality estimation, and various percentile estimation algorithms (see “Percentiles in Practice”). Probabilistic algorithms produce approximate results, but have the advantage of requiring significantly less memory in the stream processor than exact algorithms. This use of approximation algorithms sometimes leads people to believe that stream processing systems are always lossy and inexact, but that is wrong: there is **nothing inherently approximate about stream processing**, and probabilistic algorithms are merely an optimization.

Many distributed stream processing frameworks are designed with analytics in mind: Apache Storm, Spark Streaming, Flink, Concord, Samza, and Kafka Streams. Hosted services include Google Cloud Dataflow and Azure Stream Analytics.

#### 11.3.1.3 Maintaining materialized views

We saw in “Databases and Streams” on page 451 that a stream of changes to a database can be used to keep derived data systems, such as caches, search indexes, and data warehouses, up to date with a source database. We can regard these examples as **specific cases** of maintaining materialized views (see “Aggregation: Data Cubes and Materialized Views”): deriving an alternative view onto some dataset so that you can query it efficiently, and updating that view whenever the underlying data changes.

Similarly, in event sourcing, application state is maintained by applying a log of events; here the application state is also a kind of materialized view. Unlike stream analytics scenarios, it is usually not sufficient to consider only events within some time window: building the materialized view **potentially requires all** events over an arbitrary time period, apart from any obsolete events that may be discarded by log compaction (see “Log compaction”). In effect, you need a window that stretches all the way back to the beginning of time.

In principle, any stream processor could be used for materialized view maintenance, although the need to maintain events forever runs counter to the assumptions of some analytics-oriented frameworks that mostly operate on windows of a limited duration. Samza and Kafka Streams support this kind of usage, building upon Kafka’s support for log compaction.

#### 11.3.1.4 Search on streams

Besides CEP, sometimes there is a need to search for events based on complex criteria, such as full-text search queries.

For example, media monitoring services subscribe to feeds of news articles and broadcasts from media outlets, and search for any news mentioning companies, products, or topics of interest. This is done by **formulating a search query in advance**, and then continually matching the stream of news items against this query. Similar features exist on some websites: for example, users of real estate websites can ask to be notified when a new property matching their search criteria appears on the market. The percolator feature of Elasticsearch is one option for implementing this kind of stream search.

Conventional search engines first index the documents and then run quries over the index. Searching a stream stores the queries and documents run past the quries, like in CEP. In the simpliest case, you can test every document against every query. To optimize, it is **possible to index the queries as well as the documents**.

#### 11.3.1.5 Message passing and RPC

In “Message-Passing Dataflow” we discussed message-passing systems as an alternative to RPC—i.e., as a mechanism for services to communicate, as used for example in the actor model. Although these systems are also based on messages and events, we normally don’t think of them as stream processors:

1. Actor frameworks are primarily a mechanism for managing concurrency and distributed execution of communicating modules, whereas stream processing is primarily a data management technique.
1. Communication between actors is often ephemeral and one-to-one, whereas event logs are durable and multi-subscriber.
1. Actors can communicate in arbitrary ways (including cyclic request/response patterns), but stream processors are usually set up in acyclic pipelines.

Crossover areas: Apache Storm has a feature called distributed RPC, which allow queries to be farmed out to a set of nodes that also process event streams; the queries are interleaved with events from the input streams, and results can be aggregated and sent back to the user.

### 11.3.2 Reasoning About Time

Using the timestamps in the events allows the processing to be deterministic. On the other hand, many frameworks use the local system clock on the processing machine (the processing time) to determine windowing.

#### 11.3.2.1 Event time versus processing time

There are many reasons why processing may be delayed. Moreover, message delays can also lead to unpredictable ordering of messages.

Analogy with Star Wars movies: releasing time and viewing time are similar to event timestamp and processing time.

![](./11-7.processing.artifact.png)

#### 11.3.2.2 Knowing when you’re ready

A tricky problem is that you can never be sure when you have received all of the events for a particular window. You can time out but some events can be delayed due to network interruption. You need to be able to handle such **straggler** events that arrive after the window declared complete. Broadly, you have two options

1. ignore the straggler events. probably a small percentage. You can track the dropped events as a metric and alert if it becomes significant.
1. publish a correction, may also need to retract the previous output.

In some cases it is possible to use a special message to indicate timeout which can be used by consumers to trigger windows. However, if several producers each has their own minimum timestamp thresholds, the consumers need to keep track of each producer individually. Adding and removing producers is trickier.

#### 11.3.2.3 Whose clock are you using, anyway?

Assigning timestamps is even more difficult if events are **buffered** at several points in the system. For example, mobile app may buffer events for hours or days when offline and send them to server when back online.

In this context, the timestamp should really be when the user interaction occured, according to the mobile device's clock. However, the clock on a user-controlled device often cannot be trusted, as it might be accidentally or deliberately set to the wrong time (p289 clock synchronization). The time the event was received by the server is under your control but less meaningful in terms of user interaction.

To adjust for incorrect device locks, one approach is to **log three timestamps**:

1. device clock: event occur time
1. device clock: event sent to server
1. server clock: event received

Subtracting second from the third, you can estimate the offset between device clock and server clock (assuming network delay is negiligible). You can then apply the offset to the event timestamp to estimate the true time when the event occurred (assuming the device clock offset did not change during that whole period).

Batch processing suffers from the same issues of reasoning about time. It is just more noticeable in a streaming context.

#### 11.3.2.4 Types of windows

The window can be used for aggregations, e.g., to count events or calculate average. Several types of windows are in common use:

1. tumbling window, fixed length, rounding down to nearest minute
2. hopping window, fixed length, allow overlap, e.g., 5-min window with a hop size of 1 min would cover 10:03:00-10:07:59 and the next 10:04:00-10:08:59. You can implement hopping window by calculating 1-min tumbling windows then aggregating over several adjacent windows.
3. sliding window, 5-min sliding window cover 10:03:39 and 10:08:12, because they are less than 5 min apart (note tumbling and hopping 5-min windows would not have put the two in the same window as they use fixed boundaries). can be implemented by keeping a buffer of events sorted by time and removing old events when they expire.
4. session window, no fixed duration. grouping all events for the same user occur closely together in time. window ends when the user has been inactive for some time, e.g., no events for 30 min. Sessionization is common requirement for website analytics (see group by on p406).

### 11.3.3 Stream Joins

In chapter 10, we discussed batch jobs **join datasets by key**. Stream processing generalizes data pipelines to incremental processing of unbounded datasets, there is exactly the same need for joins on streams.

Since new events can appear anytime on a stream, joins on streams is more challenging.

1. stream-stream joins
1. stream-table joins
1. table-table joins

#### 11.3.3.1 Stream-stream join(window join)

Advertising systems need to measure click through rate for searches. The click and search events are joined by session ID.

1. The time difference between the search event and click event can vary a lot: a few seconds, days, or weeks. Due to network delays, the click event may arrive before the search event.
1. Note embedding the details of the search in the click event only tells about the cases where the result was clicked and not about the searches where the user did not click any.

**You may choose to join if they occur at most one hour apart**. To implement this kind of join, a stream processor needs to maintain state, e.g., all events occurred in the last hour indexed by session ID. Whenever a new event occurs, it is added to the appropriate index, and the stream processor also checks the other index for same session ID. If the search event expires without a matching click event, you emit an event saying which search results were not clicked.

#### 11.3.3.2 Stream-table join(stream enrichment)

Page 404 has an example where a process to enrich the activity events with user information from the user data base. The database lookup could be implemented by querying a remote database, which can be slow and risk overloading the database.

Another approach is to load a copy of the database to the stream processor similar to batch job. The local copy can be an in-memory hash table or an index on the local disk. A stream processor is long running compared to the batch job and the local copy needs to be kept up to date. This can be solved by change data capture. We obtain a join between two streams: activity events and the profile udpates.

For the user profile changelog stream, the join uses a window that reaches back to the beginning of time (conceptually infinite window), with newer versions overwriting older ones. For the activity stream input, the join may not maintain a window at all.

#### 11.3.3.3 Table-table join (materialized view maintenance)

For twitter timeline example on page 11, we use a timeline cache:

1. when user `u` sends a new tweet, add to time of of all followers (push).
1. when a tweet is deleted, remove from all users timelines (push)
1. when user `u` starts following `v`, recent tweets by `v` are added to `u`'s timeline (push)
1. when u unfollows `v`, tweets by `v` are removed from `u`'s timeline

To implement the cache maintenance in a stream processor, you need tweet events stream and follow relationships. The stream process maintains a database containing follow relationships to know which timelines to update when a new tweet arrives.

Another way of looking at this stream process is materialized view for a query joining two tables (tweets and follows):

```sql
SELECT follows.follower_id AS timeline_id, array_agg(tweets.* ORDER BY tweets.timestamp DESC)
FROM tweets
JOIN follows ON follows.followee_id = tweets.sender_id GROUP BY follows.follower_id
```

The timelines are effectively a cache of the query result, updated every time the underlying tables change. The stream of changes to the materialized join follows the product rule (u·v)′ = u′v + uv′. In words: any change of tweets is joined with the current followers, and any change of followers is joined with the current tweets.

#### 11.3.3.4 Time-dependence of joins

The three types of join above all require the stream processor to maintain some state. The order of the events that maintain the state is important (follow then unfollow, or the other way around). If state changes over time, what point in time do you use for the join?

Another example is joining sales to tax rates. You probably want to join with the tax rate at the time of the sale.

If the ordering across streams is undetermined, the join becomes **nondeterministic**. You cannot rerun the same job and get the same result. The events on the streams may be interleaved in a different way when you run the job again.

In data warehouses, this issue is known as a **slowly changing dimension** (SCD) and is often addressed by using a unique ID for versioning. For example, a new id when the tax rate changes. The join is deterministic but the consequence is **log compaction is not possible** since all versions need to be retained.

### 11.3.4 Fault Tolerance

Batch job can tolerate faults pretty easily: if a task in a MapReduce job fails, it can start again in another machine and the output of the failed task is discarded. The transparent retry is possible because input files are immutable, each task writes the output to a separate file in HDFS, and output is only made visible when a task completes successfully. This principle is known as **exactly-once semantics**, although **effectively-once** is more descriptive.

The same issue is harder to handle in stream processing: waiting task finish before output visible is not an option, stream is never ending.

#### 11.3.4.1 Microbatching and checkpointing

One approach is to break the stream into small blocks and treat each as miniature batch process. **This microbatching approach is used in Spark Streaming**. Batch size is typically around one second, which is a performance compromise: smaller batches incur greater scheduling and coordination overhead, while larger batches mean a longer delay before results are visible.

Microbatching implicitly provides a **tumbling window equal to the batch size** (windowed by processing time, not event time).

A variant approach in Apaphe Flink is to periodicially generate **rolling check-points of state** and write to durable storage. If a stream operator crashes, it can restart from its most recent checkpoint. The checkpoints are triggered by barriers in the message stream, similar to boundaries between microbatches, but without forcing a specific window size.

Within the confines of stream processing framework, the above approaches provide the same exactly-once semantics as batch processing. However, after the output is sent (writing to DB, email, message to an external broker), the framework is no longer able to discard the output of a failed batch. In this case, restarting a failed task causes the external side effect to happen twice.

#### 11.3.4.2 Atomic commit revisited

Previsouly discussed on page 360 in "Exactly-once message processing" in the context of distributed transactions and two-phase commit. In Chapter 9, the discussion was traditional implementations of distributed transactions, such as XA. In more restricted environments, it is possible to implement such an atomic facility efficiently. This approach is used in Google Cloud Dataflow and VoltDB, plans to add to Kafka. Unlike XA, it does not attempt to provide transactions across heterogeneous technologies, but **keep them internal by managing both state changes and messaging within the stream processing framework**. The overhead of the transaction protocol can be **amortized** by processing several input messages within a single transaction.

#### 11.3.4.3 Idempotence

Our goal is to discard the partial output of failed tasks so they can be safely retried without taking effect twice. Distributed transactions are one way, another is idempotence.

Setting a key in key-value store to some fixed value is idempotent, whereas incrementing a counter is not. If an operation is not natually idempotent, it can often be made idempotent with **a bit of extra metadata**. When consuming from Kafka, every message has a persistent monotonically increasing offset. When writing to external DB, you can include the offset to tell whether an update has already been applied and avoid performing the same update again.

State handling in Storm's Trident is based on a similar idea. Relying on idempotence implies:

1. restarting a failed task must replay same messages in the same order (a log based message broker does this)
1. the processing must be deterministic
1. no other node may concurrently update the same value

When failing over from one processing node to another, fencing (p301) may prevent interference from a node thought to be dead but actually active. Idempotent operations may achieve exactly-once semantics with only a small overhead.

#### 11.3.4.4 Rebuilding state after a failure

Stream processing may require state and must ensure recover after failure.

1. any windowed aggregations such as counters, averages, and histograms
1. any tables and indexes used for joins

One option is to keep the state in remote DB and replicate it. Although querying a remote DB for each message can be slow (p473, stream-table join). **An alternative is to keep state local to stream processor and replicate periodically**. When recovering, the new task can read the replicated state and resume processing without data loss.

For example, Flink periodically snapshots operator state and writes them to a durable storage such as HDFS; Samza and Kafka streams replicate by sending them to a dedicated Kafka topic with log compaction, similar to change data capture. VoltDB replicates state by redundantly processing each message on several nodes (p252, actual serial execution).

In some cases, state can be rebuilt from input streams and may not need to be replicated. If the state consists of aggregations over a fairly short window, it may be fast to simply replay. If the state is a local replica of a DB maintainted by change data capture, the DB can be rebuilt from the log-compacted change stream.

The trade-offs depend on the performance characteristics and may shift as storage and networking technologies evolve. In some systems, network delay may be lower than disk access latency, network bandwidth may be comparable to disk bandwidth.

## Summary

In some ways, stream processing is very much like the batch processing in Chapter 10, but done continuously on unbounded (never-ending) streams rather than on a fixed-size input. From this perspective, message brokers and event logs serve as the streaming equivalent of a filesystem.

Two types of message brokers:

1. AMQP/JMS-style. Broker assigns messages to consumers and consumers ackowledge after successful processing. Message is deleted after ack. The approach is appropriate as an async form of RPC (see “message-passing dataflow” on p136), for example in a task queue, the order is not important and no need to go back and read old messages after processed.
1. Log-based. Assigns all messages in a partition to the same consumer node, and always deliver messages in the same order. Parallelism is achieved through partitioning. Consumers track progress by checkpointing the offset of the last message processed. The broker retains messages on disk, so it is possible to reread old messages.


<!--References -->

