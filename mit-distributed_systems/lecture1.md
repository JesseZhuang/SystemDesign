# Introduction

Why do people build distributed systems?
- to increase capacity via parallel processing
- to tolerate faults via replication
- to match distribution of physical devices e.g. sensors
- to achieve security via isolation

But it's not easy:
- many concurrent parts, complex interactions
- must cope with partial failure
- tricky to realize performance potential

MAIN TOPICS

This is a course about infrastructure for applications.
* Storage.
* Communication.
* Computation.

A big goal: hide the complexity of distribution from applications.

Topic: fault tolerance
- 1000s of servers, big network -> always something broken
- We'd like to hide these failures from the application.
- "High availability": service continues despite failures
- Big idea: replicated servers.

Topic: consistency
- General-purpose infrastructure needs well-defined behavior.
    E.g. "Get(k) yields the value from the most recent Put(k,v)."
- Achieving good behavior is hard!
    "Replica" servers are hard to keep identical.

Topic: performance
- The goal: scalable throughput
    Nx servers -> Nx total throughput via parallel CPU, disk, net.
- Scaling gets harder as N grows:
    Load imbalance.
    Slowest-of-N latency.
    Some things don't speed up with N: initialization, interaction.

Topic: tradeoffs
- Fault-tolerance, consistency, and performance are enemies.
- Fault tolerance and consistency require communication
    e.g., send data to backup
    e.g., check if my data is up-to-date
    communication is often slow and non-scalable
- Many designs provide only weak consistency, to gain speed.
    e.g. Get() does *not* yield the latest Put()!
    Painful for application programmers but may be a good trade-off.
- We'll see many design points in the consistency/performance spectrum.
