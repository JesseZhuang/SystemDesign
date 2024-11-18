
## Story

Google started with MySQL, aggressive growth. So partition data. Transactions became difficult with multiple servers.
They created Spanner - cloud (distributed) SQL database.

1. They use two-phase commit for atomicity (transaction is all or nothing).
2. Consistency. Spanner provides strong consistency at a global level. Use Paxos algorithm to find a partition leader. Different partition leaders might be deployed in different zones. Leaders handle writes, followers handle reads. They use TrueTime, a combination of GPS receivers and atomic clocks. It finds the current time in each data center with high accuracy. And each server synchronizes its quartz clock with TrueTime every 30 seconds.
3. Isolation. They use two-phase locking (2PL) to avoid conflicts due to concurrent transactions. They also use snapshot isolation.
4. Durability. Result of a completed transaction never gets lost. They perform synchronous writes using Paxos. It gives durability as data is written to a majority of followers. Besides they separate the compute and storage layers in the database server. The compute layer handles reads and writes. While Google Colossus is used as the storage layer. Think of Colossus as a distributed file system for high performance and durability. Spanner offers 99.999% availability.

Here’s how they perform reads:

1. The request gets routed to the nearest zone even if it has a follower
2. The follower asks the leader for the latest timestamp of the requested data
3. The follower compares the leader’s response with its timestamp
4. The follower responds to the client
5. Also the follower will wait for the leader to synchronize in case its data is outdated.

two phase locking

1. Growing phase: transaction performs reads or writes once it acquires locks on data
2. Shrinking phase: transaction releases locks one at a time after it’s complete

Snapshot Isolation: The previous data isn’t overwritten.
Instead, a new value gets written along with a TrueTime timestamp.
Also older versions get automatically removed after a specific period to save storage.

## References

1. https://newsletter.systemdesign.one/p/cloud-spanner-database
