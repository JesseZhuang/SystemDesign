# Stream Processing

Assumption for batch processing: the input is bounded, i.e., of a known and finite size, so the batch process knows when finished reading input. For example, sorting that is central to MapReduce. It could happen that the last input record is the lowest key and needs to be the very first output record. Starting the output early is not an option.

In reality, a lot of data is unbounded because it arrives gradually over time. Thus, batch processors must artificially divide the data into chunks of fixed duration: for example, processing a day’s worth of data at the end of every day.

To reduce the delay for daily batch processing, we can run processing more frequently--say, every second--or even continuously. This is the idea befind stream processing.

In general, a “stream” refers to data that is incrementally made available over time. The concept appears in many places: in the stdin and stdout of Unix, programming languages (lazy lists: infinite streaming, lazy evaluation, e.g. Haskell `is.tl` and `is.take`, Python `generators`(`yield`)), filesystem APIs (such as Java’s FileInputStream), TCP con‐ nections, delivering audio and video over the internet, and so on.

## 11.1 Transmitting Event Streams

In both streaming and batch processing worlds, input data essentially is a small, self contained, immutable object containing the details of something that happened at some point in time. The input data could be a file for batch processing and an event for streaming.

In a filesystem, a filename identifies a set of related records; in a streaming system, related events are usually grouped together into a topic or stream.

Moving toward continual processing with low delays, polling becomes expensive if datastore is not designed for this kind of usage. The more often you poll, the lower the percentage of  new events, and thus the higher overheads. Instead, it is better for consumers to be notified when new events appear.

Databases have traditionally not supported notification mechanism well: relational databases commonly have triggers, which can react to a change (e.g., a row being inserted into a table), but they are very limited in what they can do and have been an afterthought in database design. Instead, specialized tools have been developed for event notifications.

## 11.1.1 Messaging Systems

A simple way to implement could be a Unix pipe or TCP connection between producer and consumer. That connects one sender with one recipient. Most messaging systems expand on this basic model. To differentiate, following two questions can be asked:

1. What happens if the producers send messages faster than the consumer can process?
    1. drop
    1. buffer in a queue. What happens as the queue grows?
        1. system crash if out of memory
        1. write messages to disk. How does disk access affect the performance?
    1. apply backpressure(flow control, blocking producer from sending). Unix pipes and TCS use backpressure.
1. What happens if nodes crash or temporarily go offline-are any messages lost? Durability may require some combination of disk writing and/or replication. If can afford to sometimes lose messages, can get higher throughput and lower latency on the same hardware.

Whether message loss is acceptable depends on application. Periodic metrics might be ok to miss afew data points, but counter will be off.

Batch processing system provide a strong reliability with retry and discarding partial failures. Output is same as if no failures occured, which simplify the programming model.

**Direct messaging from producers to consumers**

- UDP multicast is widely used in the financial industry for streaming stock feeds, where low latency is important. Application level protocols can recover lost packets (producer remember packets so that it can retransmit).
- Brokerless libraries such as ZeroMQ and nanomsg pub/sub over TCP/IP multicast.
- StatsD and Brubeck use UDP messaging for collecting metrics. (In StatsD, counter metrics are only correct if all messages are received. Using UDP makes the metrics at best approximate.)
- If the consumer exposes a service on the network, producers can make a direct HTTP or RPC request, which is the idea behidn webhooks, a pattern in which a callback URL of one service is registered with another service.

Require application code to be aware of message loss. Can tolerate limited faults. Generally assume producers and consumers are constantly online.

**Message Broker (Queue)**

Essentially a kind of database optimized for handling message streams. More easily tolerate clients that come and go (connect, disconnct, and crash). Some only keep messages in memory, others write to disk. They generally allow unbounded queueing as opposed to dropping message or backpressure.

Consumers are generally asynchronous considering that the producer does not wait for messaging processing.

**Message Brokers compared to databases**

Some message brokers can even participate in two-phase commit protocols using XA or JTA.

Important practical differences:

- Message broker automatically delete a message, thus not suitable for long-term storage. Databases keep until explicitly deleted.
- Message brokers assume the working set is small. If broker buffer a lot of messages, perhaps spilling messages to disk if they no longer fit in memory, each individual message takes longer to process and the overall throughput may degrade.
- Databases often support secondary indexes and various of searching, while message brokers often support subscribing to a subset of topics matching some pattern. Both are essentially ways for a client to select portion of the data.
- Database query result is based on a point-in-time snapshot of the data. Message brokers do not support arbitrary quries, but notify clients when data changes.

The above traditional view of message brokers are in standards like JMS (JSR-343 Java Message Service 2.0 specification) and AMQP (Advanced Message Queuing Protocol Specification) and implemented in software like RabbitMQ, ActiveMQ, HornetQ, Qpid, TIBCO enterprise message service, IBM MQ, Azure service bugs, and google cloud pub/sub.

**Multiple consumers**

When multiple consumers read messages in the same topic, two main patterns are used.

1. Load balancing. Each message is delivered to one of the consumers. The broker may assign messages to consumers arbitrarily. Useful when processing is expensive and can add consumers to parallelize. In AMQP, you can implement load balancing by having multi‐ ple clients consuming from the same queue, and in JMS it is called a shared subscription.
1. Fan out. Each message is delivered to all of the consumers. This feature is provided by topic subscriptions in JMS, and exchange bindings in AMQP.

The two patterns can be combined. Two separate groups, each group collectively receives all messages but within each group only one node receives each message.

**Acknowledgements and redelivery**

Broker can use ack to avoid processing failures or partial processing.

It is possible that the message was fully processed but the ack was lost in the network. Handling this requires an atomic commit protocol.

The combination of load balancing and redelivery leads to messages being reordered. To avoid, you can use a separate queue per consumer.

## 11.1.2 Partitioned Logs

Why can we not have a hybrid, combining the durable storage approach of databases with the low-latency notification facilities of messaging? This is the idea behind log- based message brokers.

**Using logs for message storage**

The Unix tool `tail -f`, which watches a file for data being appended, essentially works like this.

![](./fig.11-3.png)

The log can be partitioned. Different partitions can then be hosted on different machines, making each partition a separate log that can be read and written independently from other partitions. A topic can then be defined as a group of partitions that all carry messages of the same type.

Within each partition, the broker assigns a monotonically increasing sequence number, or offset, to every message. Such a sequence number makes sense because a partition is append-only, so the messages within a partition are totally ordered. There is no ordering guarantee across different partitions.

Apache Kafka, Amazon Kinesis Streams, and Twitter’s DistributedLog are log-based message brokers. Google Cloud Pub/Sub is architecturally similar but exposes a JMS-style API rather than a log abstraction. Even though these message brokers write all messages to disk, they are able to achieve throughput of millions of messages per second by partitioning across multiple machines, and fault tolerance by replicating messages.

**Logs compared to traditional messaging**

The log-based approach trivially supports fan-out messaging, because several consumers can independently read the log without affecting each other—reading a message does not delete it from the log. To achieve load balancing across a group of consumers, instead of assigning individual messages to consumer clients, the broker can assign entire partitions to nodes in the consumer group.

Each client then consumes all the messages in the partitions it has been assigned. Typically, when a consumer has been assigned a log partition, it reads the messages in the partition sequentially, in a straightforward single-threaded manner. This coarse grained load balancing approach has some downsides:

- The number of nodes sharing the work of consuming a topic can be at most the number of log partitions in that topic, because messages within the same partition are delivered to the same node.
- If a single message is slow to process, it holds up the processing of subsequent messages in that partition (a form of head-of-line blocking).

Thus, in situations where messages may be expensive to process and you want to parallelize processing on a message-by-message basis, and where message ordering is not so important, the JMS/AMQP style of message broker is preferable. On the other hand, in situations with high message throughput, where each message is fast to process and where message ordering is important, the log-based approach works very well.
