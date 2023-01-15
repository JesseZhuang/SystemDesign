# Reliable, Scalable, and Maintainable Applications

Many applications are data-intensive as opposed to compute-intensive.

## 1.1 Thinking About Data Systems

![](./1-1.data.system.png)

Many new tools no longer neatly fit into traditional categories [1].

1. Redis, datastore also used as message queues
1. Kafka, message queues with database-like durability

Influence factors

1. skills and experience of people involved
1. legacy system dependencies
1. time-scale for delivery
1. organization tolerance on differnt kinds of task
1. regulatory constraints

## 1.2 Reliability

1. functional requirements
1. tolerate user mistakes or using the software in unexpected ways
1. performance under load and data volume
1. prevents unauthorized access and abuse

Continue to work correctly even when things go wrong. Fault-tolerant (certain types) or resilient.

Fault is not the same as a failure [2]. Fault is usually one component only and failure is when the whole system stops providing the required service to the user. It's impossible to reduce fault probability to zero and usually best to prevent faults from causign failures.

Many critical bugs are due to poor error handling [3]. Deliverately inducing faults can increase confidence and ensure fault-tolerance machinery is continually exercised and tested. The netflix chaos monkey [4] is an example.

We generally prefer tolerating faults over preventing. Although for security matters prevention is better.

### 1.2.1 Hardware Faults



## 1.1 Scalability
## 1.1 Maintainability

<!-- references -->

[2]: Walter L. Heimerdinger and Charles B. Weinstock: “A Conceptual Framework for System Fault Tolerance,” Technical Report CMU/SEI-92-TR-033, Software Engineering Institute, Carnegie Mellon University, October 1992.

[4]: Yury Izrailevsky and Ariel Tseitlin: “The Netflix Simian Army,” techblog.net‐ flix.com, July 19, 2011.
