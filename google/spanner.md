
## Story

Google started with MySQL, aggressive growth. So partition data. Transactions became difficult with multiple servers.
They created Spanner - cloud (distributed) SQL database.

1. They use two-phase commit for atomicity (transaction is all or nothing).
2. Consistency. Spanner provides strong consistency at a global level. Use Paxos algorithm to find a partition leader. Different partition leaders might be deployed in different zones. Leaders handle writes, followers handle reads. They use TrueTime, a combination of GPS receivers and atomic clocks. It finds the current time in each data center with high accuracy. 

## References

1. https://newsletter.systemdesign.one/p/cloud-spanner-database