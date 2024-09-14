[CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem)
====

- Consistency: every read receives the most recent write or an error.
- Availability: every request receives a response that is not an error.
- Partition tolerance: the system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes
- CAP theorem implies that in the presence of a network partition, one has to choose between consistency and availability
- CAP is frequently misunderstood as if one has to choose to abandon one of the three guarantees at all times. In fact, the choice is really between consistency and availability only when a network partition or failure happens; at all other times, no trade-off has to be made.
- [ACID](https://en.wikipedia.org/wiki/ACID) databases choose consistency over availability. Atomicity, consistency, isolation (concurrent transactions), and durability for database transactions. Implementation: locking vs multi versioning, distributed transactions.
- [BASE](https://en.wikipedia.org/wiki/Eventual_consistency) systems choose availability over consistency. basically-available, soft-state, eventual consistency.

The most appropriate approach to reconciliation depends on the application. A widespread approach is ["last writer wins"](https://en.wikipedia.org/wiki/Eventual_consistency#cite_note-Vogels-1). Another is to invoke a user-specified conflict [handler](https://en.wikipedia.org/wiki/Eventual_consistency#cite_note-Bayou-Conflicts-4). [Timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps) and [vector clocks](https://en.wikipedia.org/wiki/Vector_clock) are often used to detect concurrency between updates. [Some people](https://en.wikipedia.org/wiki/Eventual_consistency#cite_note-10) use "first writer wins" in situations where "last writer wins" is unacceptable.

Reconciliation of concurrent writes must occur sometime before the next read, and [can be scheduled](https://en.wikipedia.org/wiki/Eventual_consistency#cite_note-Confront-11) at [different instants](https://en.wikipedia.org/wiki/Eventual_consistency#cite_note-Bayou-3):

- Read repair: The correction is done when a read finds an inconsistency. This slows down the read operation.
- Write repair: The correction takes place during a write operation slowing down the write operation.
- Asynchronous repair: The correction is not part of a read or write operation.
