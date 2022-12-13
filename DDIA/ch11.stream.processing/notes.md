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

**Message Brokers**

