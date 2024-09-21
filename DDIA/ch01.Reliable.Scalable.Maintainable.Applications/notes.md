# Reliable, Scalable, and Maintainable Applications

Many applications are data-intensive as opposed to compute-intensive.

## 1.1 Thinking About Data Systems

![](./1-1.data.system.png)

Many new tools no longer neatly fit into traditional categories [1].

1. Redis, datastore also used as message queues
2. Kafka, message queues with database-like durability

Influence factors

1. skills and experience of people involved
2. legacy system dependencies
3. time-scale for delivery
4. organization tolerance on different kinds of task
5. regulatory constraints

## 1.2 Reliability

1. functional requirements
2. tolerate user mistakes or using the software in unexpected ways
3. performance under load and data volume
4. prevents unauthorized access and abuse

Continue to work correctly even when things go wrong. Fault-tolerant (certain types) or resilient.

Fault is not the same as a failure [2]. Fault is usually one component only and failure is when the whole system stops providing the required service to the user. It's impossible to reduce fault probability to zero and usually best to prevent faults from causing failures.

Many critical bugs are due to poor error handling [3]. Deliberately inducing faults can increase confidence and ensure fault-tolerance machinery is continually exercised and tested. The netflix chaos monkey [4] is an example.

We generally prefer tolerating faults over preventing. Although for security matters prevention is better.

### 1.2.1 Hardware Faults

Hard disks have a mean time to failure (MTTF) of about 10 to 50 years [5], [6]. Thus, on a storage cluster with 10k disks, we should expect on average one disk to die per day.

Our first response is usually to add redundancy. Disks may be set up in a RAID configuration, servers may have dual power supplies and hot-swappable CPUs, and datacenters may have batteries and diesel generators for backup power.

Until recently, redundancy of hardware components was sufficient. As long as restore a backup onto a new machine fairly quickly, the downtime in case of failure is not catastrophic in most applications. Multi-machine redundancy was only required by a small number of applications.

However, more applications have begin using larger number of machines, which proportionally increases the rate of hardware faults. Moreover, in some cloud platforms such as AWS it is common for virtual machine to become unavailable without warning [7], as the platforms are designed to prioritize flexibility and elasticity over single-machine reliability.

There is a move towards systems tolerating loss of entire machines, by using software fault-tolerance techniques in addition to hardware redundancy. Such system can tolerate machine failure can be patched one node at a time, without downtime (a rolling upgrade, chapter 4).

### 1.2.2 Software Errors

Typically, hardware faults are random and independent except for weak correlations: temperature in the server rack.

Another category is systematic error [8]. Such faults are harder to anticipate and because they are correlated across nodes, they will cause many more system failures than hardware faults [9]. Examples:

- leap second on June 30 2012 causing many applications to hang simultaneously due to a bug in linux kernel [9]
- a runaway process using up resource: cpu, memory, disk, or network bandwidth
- a dependent service slow down, unresponsive, or returning corrupted responses
- cascading failures, when a small fault in one component triggers a fault in another, which in turn triggers further [10]

These bugs often lie dormant for a long time until triggered by an unusual set of circumstances. Assumption may no longer be true [11].

Lots of small things can help:

1. carefully thinking about assumptions and interactions
2. thorough testing
3. process isolation
4. allowing process to crash and restart
5. measuring, monitoring, and analyzing behavior in prod
6. constantly check guarantee (e.g., messaging system incoming messages == outgoing)


### 1.2.3 Human Errors

One study found configuration errors by operators were the leading cause of outages, whereas hardware faults is 10-25% [13].

- Design in a way to minimize opportunities for error. Well-designed APIs, abstractions, and admin interfaces make it easy to do "the right thing". Balance not to be too strict so people will work around.
- Provide fully-featured non-production sandbox environments using real data without affecting real users.
- Testing thoroughly. Unit tests, whole-system integration tests, and manual tests [3]. Automated tests is widely used, well understood, and specially valuable in covering corner cases rarely arise in normal operation.
- Allow quick and easy recovery from human errors, minimize impact. Fast to roll back configuration changes, roll out new code gradually (unexpected bugs only affect a small subset of users), and provide tools to recompute data (old computation incorrect).
- Detailed and clear monitoring, performance and error rates, referred to as telemetry (after rocket launch [14]).
- Implement good management practices and training


### 1.2.4 How Important Is Reliability?

Consider a parent who stores pictures of videos of all children in your photo application [15]. Would they know how to restore from DB corruption?

You may choose to sacrifice reliability to reduce development cost, e.g., when developing a prototype product for an unproven market. Or operational cost, e.g., for a service with a narrow profit margin. But should be conscious when cutting corners.

## 1.3 Scalability

One common reason for degradation is increasing load, e.g., 10k-100k concurrent users, or 1 to 10 million.

Scalability is not 1D label. "How can we add computing resources to handle the additional load?"

### 1.3.1 Describing Load

The best choice of load parameters depends on the system architecture:

- QPS web server
- DB read/write ratio
- number of simultaneous users in a chat room
- cache hit rate
- perhaps average case
- perhaps a number of extreme cases is the bottleneck

Using Twitter 2012 data as example [16].

Two main operations:

1. post tweet 4.6k avg, 12k peak QPS
2. home timeline 300k QPS

Handling 12k QPS tweet write is fairly easy. However, twitter's scaling challenge is due to fan-out (a term borrowed from electrical engineering, describing number of logic gate inputs connected to another gate's output. The output need to supply enough current to drive all the attached inputs. In transaction processing systems, describing number of requests to other services in order to serve one incoming request).

Broadly two ways to implement:

1, Posting simply inserts. Timeline look up all and merge.

```sql
SELECT tweets.*, users.* FROM tweets
  JOIN users ON tweets.sender_id = users.id 
  JOIN follows ON follows.followee_id = users.id
  WHERE follows.follower_id = current_user
```

2, Maintain a cache for timeline, insert new tweet to follower's caches.

The first version of twitter used approach 1, but the systems struggled to keep up with the timeline queries. The average rate of published tweets is almost two orders of magnitude lower than the rate of timeline reads. It's preferable to do more work at write time and less at read time.

On average, a tweet is delivered to about 75 followers, so 4.6k QPS becomes 345k to the timeline caches. This average hides that number of followers varies wildly. Some users have over 30 million followers.

The distribution of followers per user (weighted by how often those users tweet) is a key load parameter for discussing scalability, since it determines the fan-out load.

Twitter is moving to a hybrid of the two approaches. Celebrities are excepted from this fan-out.

### 1.3.2 Describing Performance

- when increase a load parameter and keep the system resources (cpu, memory, network bandwidth) unchanged, how is performance affected
- when increase a load parameter, how much to increase the resources if you want to keep performance unchanged?

In a batch processing system such as Hadoop, we usually care about **throughput**-number of records we can process per second, or the total time to run a job on a dataset of a certain size (in an ideal world, the **running time** of a batch job is the size of the dataset divided by the throughput. In practice, running time is often longer due to skew, where data is not spread evently across workers). In online systems, what's usually more important is the service's **response time**.

Latency (the duration that a request is waiting to be handled) and response time (what the client sees, including network delays and queuing delays) **are not the same**. 

Random additional latency can be introduced by

- a context switch to a background process
- loss of a network package and tcp transmission
- a garbage collection pause
- a page fault forcing a read from a disk
- mechanical vibrations in the server rack [18]

Median, 50th percentile, p50. Note if the user makes several requests over the course of a session or because several resources are included in a single page, the probability of at least one request is slower than median is much greater than 50%.

High percentiles of response times, also known as tail latencies are important because they directly affect users' experience. Customer with the slowest requests are often those who have the most data on their accounts because they have made many purchases, the most valuable customer [19]. Amazon has also observed that a 100 ms increase in response time reduces sales by 1% [20]. A 1-second slowdown reduces customer satisfaction metric by 16% [21],[22]. Optimizing the p99.99 percentile was deemed too expensive and to not yield enough benefit for Amazon's purposes.

Percentiles are often used in service level objectives (SLOs) and service level agreements (SLAs). For example, the service is considered up if p50 < 200 ms and p99 is < 1 s. Availability is 99.9%.

Queueing delays often account for a large part of the response time at high percentiles. It only takes a small number of slow requests to hold up the processing of subsequent requests, known as head-of-line blocking. 

Even if only a small percentage of backend calls are slow, the chance of getting a slow call increases if an end-user request requires multiple back-end calls, tail latency amplification.

You need to efficiently calculate response time percentiles. You may want to keep a rolling window of the last 10 minutes. The naive implementation is to keep a list and sort that list every minute. There are algorithms that can calculate a good approximation at minimal cpu and memory cost, **such as forward decay** [25], t-digest [26], or HdrHistogram [27]. Beware that averaging percentiles, e.g., to reduce the time resolution or to combine data from several machines, is mathematically meaningless. The right way of aggregating response data is to add the histograms [28].



### 1.3.3 Approaches for Coping with Load

An architecture appropriate for one level of load is unlikely to cope with 10 times that load.

People often talk of a dichotomy between scaling up (vertical scaling, more powerful machine) and scaling out (horizontal scaling, multiple smaller machines). Distributing load across multiple machines is known as a shared-nothing architecture.

Some systems are elastic, whereas other systems are scaled manually. An elastic system can be useful if load is highly unpredictable, but manually scaled systems are simpler and may have fewer operational surprises.

There is no magic scaling sauce, a generic one-size-fits-all scalable architecture. The problem may be

- volume of reads
- volume of writes
- volume of data to store
- complexity of the data
- response time requirements
- the access patterns

For example, a system to handle 100k QPS, each 1kB in size looks very different from one for 3 request per minute, each 2 GB in size. Although the two have the same data throughput 6 GB per minute.

An architecture is built around assumptions of the load parameters. If those assumptions turn out to be wrong, the effort for scaling is at best wasted, at worst counterproductive. In an early-stage startup it is usually more important to be able to iterate quickly on product features than to scale to some hypothetical future load.

## 1.4 Maintainability

### 1.4.1 Operability: Making Life Easy for Operations

Good operations can often work around the limitations of bad or incomplete software but good software cannot run reliably with bad operations [12].

Operations teams typically are responsible for following and more [29]:

- monitoring system health and quickly restore if it goes into a bad state
- tracking down the cause of problems such as system failures and degraded performance
- keeping software and platforms up to date, including security patches
- keeping tabs on how different systems affect each other, so that a problematic change can be avoided before it causes damage
- anticipating future problems and solving them before they occur (e.g., capacity planning)
- establishing good practices and tools for deployment, configuration management, and more
- performing complex maintenance tasks, such as moving an application from one platform to another
- maintaining the security of the system as configuration changes are made
- defining processes that make operations predictable and help keep the production environment stable
- preserving the organization's knowledge about the system, even as individual people come and go

### 1.4.2 Simplicity: Managing Complexity

A software project mired in complexity is sometimes described as a big ball of mud [30].

- explosion of the state space
- tight coupling of modules
- tangled dependencies
- inconsistent naming and terminology
- hacks for performance
- special casting workaround

Mosley and Marks [32] define complexity as accidental if it is not inherent in the problem that the software solves but arises only from the implementation.

One of the best tools we have for removing accidental complexity is abstraction. For example, high level programming languages are abstractions that hide machine code, CPU registers, and system calls. SQL is an abstraction that hides complex on-disk and in-memory data structures, concurrent requests, and inconsistencies after crashes.

### 1.4.3 Evolvability: Making Change Easy

- learn new facts
- previously unanticipated use cases emerge
- business priorities change
- users request new features
- regulatory requirements change
- platform upgrade or replacement
- growth of the system forces architectural changes

Agile working patterns provide a framework for adapting to change. For example, test-driven development (TDD) and refactoring.

## 1.5 Summary

Functional requirements: what it should do, such as allowing data to be stored, retrieved, searched, and processed in various ways. 

Non-functional requirements: general properties like security, reliability, compliance, scalability, compatibility, and maintainability.

<!-- references -->

[2]: Walter L. Heimerdinger and Charles B. Weinstock: “A Conceptual Framework for System Fault Tolerance,” Technical Report CMU/SEI-92-TR-033, Software Engineering Institute, Carnegie Mellon University, October 1992.

[4]: Yury Izrailevsky and Ariel Tseitlin: “The Netflix Simian Army,” techblog.net‐ flix.com, July 19, 2011.

[7]: Laurie Voss: “AWS: The Good, the Bad and the Ugly,” blog.awe.sm, December 18, 2012.

[12]: Jay Kreps: “Getting Real About Distributed System Reliability,” blog.empathy‐ box.com, March 19, 2012.

[16]: Raffi Krikorian: “Timelines at Scale,” at QCon San Francisco, November 2012.

[19]: Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, et al.: “Dynamo: Amazon’s Highly Available Key-Value Store,” at 21st ACM Symposium on Operating Systems Principles (SOSP), October 2007.

[25]: Graham Cormode, Vladislav Shkapenyuk, Divesh Srivastava, and Bojian Xu: “Forward Decay: A Practical Time Decay Model for Streaming Systems,” at 25th IEEE International Conference on Data Engineering (ICDE), March 2009.

[26]: Ted Dunning and Otmar Ertl: “Computing Extremely Accurate Quantiles Using t-Digests,” github.com, March 2014.

[27]: Gil Tene: “HdrHistogram,” hdrhistogram.org.

[28]: Baron Schwartz: “Why Percentiles Don’t Work the Way You Think,” vividcor‐tex.com, December 7, 2015.

[29]: James Hamilton: “On Designing and Deploying Internet-Scale Services,” at 21st Large Installation System Administration Conference (LISA), November 2007.

[30]: Brian Foote and Joseph Yoder: “Big Ball of Mud,” at 4th Conference on Pattern Languages of Programs (PLoP), September 1997.

[32]: Ben Moseley and Peter Marks: “Out of the Tar Pit,” at BCS Software Practice Advancement (SPA), 2006.
