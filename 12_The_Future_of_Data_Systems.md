# Chapter 12 The Future of Data Systems

## Data Integration
- The most appropriate choice of software tool depends on the circumstances, a particular usage pattern
- In complex applications, data is often used in several different ways
- These factors cause the inevitability of data integration 

### Combining Specialized Tools by Deriving Data
- Heterogeneous data systems and formats become inevitable and hard to maintain
- Total ordering is the principle, especially if causality between events is critical
- Log-based derived data system and async updates are the major means
- Total ordering has cerntain limits which are still open for research. Also the scope of total ordering needs to be balanced between semantic correctness and system performance. 
#### Reasoning about dataflows
- The principle of deciding on a total order is the key to keep data system consistent
- CDC, event sourcing, and keep single source of truth by only writing raw data into one source system are means to achieve the principle
#### Derived data versus distributed transactions
- Different means to keep differnt data systems consistent
	- Distributed transaction uses 2PC and 2PL (decide on an ordering of writes by using locks for mutual exclusion)
	- Log-based derived data systems are often based on deterministic retry and idempotence and are often updated asynchronously
- log-based derived data is the most promising approach for integrating different data systems, in the absence of widespread support for a good distributed transaction protocol.
#### The limits of total ordering
- The throughput of events is greatrer than a single machine can handle (the leader)
- Need to have a seprate leader in each data center, if the servers are spread across multiple geographically distributed data centers. -> Multiple leaders -> undefined ordering of events
- When two events originate in different micro-services, there is no defined order for those events.
- Some applications maintain client-side state that is updated immediately on user input. Clients and servers are very likely to see events in different orders.
#### Ordering events to capture causality
- In cases where there is no causal link between events, the lack of a total order is not a big problem, since concurrent events can be ordered arbiturarily.
- However, for an example of an "unfriend" event and a "message-send" event, causality does matter
- If all types of events need to go through a total order broadcast, the derived data system would probably have a bottleneck here. It might be proper to handle it on the application level case by case.

### Batch and Stream Processing
#### Maintaining derived state
- async (e.g. CDC, event sourcing, log-based) is easier than distributed transaction
#### Reprocessing data for application evolution
- gradual evolution is better, i.e. maintain two independent derived view for old and new schema in transition phase
#### The lambda architecture
- core idea: stream processing incoming immutable events to provide approximate result, then batch reprocessing to provide accurate result
- practical problems: maintainability, compatibility, and cost for reprocessing
#### Unifying batch and stream processing
- log-based message broker is able to replay messages and stream processor is able to read from HDFS
- exactly-once sematics for steram processors
- tools for windowing by event time, not by processing time


## Unbundling Databases
- attempt to reconcile the two philosophies between Unix and relational database, in the hope that we can combine the best of both worlds

### Composing Data Storage Technologies
- There are parallels between the featues that are built into databases and the drived data systems that people are building with batch and stream processors.
#### Creating an index
- Index creation is very similar with setting up a new follower replica and also very similar to bootstrapping change data capture in a streaming system
- Initial consistent snapshot + write-ahead log
#### The meta-database of everything
- The dataflow across an entire organization starts looking like one huge database
- Whenever a batch, stream, or ETL process transports data from one place and form to another place and for , it is acting like the database sub-system that keeps indexes or materialized view up to date
- Two ways to achieve the goal:
	- Federated database: unifying reads
	- Unbundled database: unifying writes
#### Making unbundling work
- Unifying read is ultimately quite a manageable problem
- Unifying write is the harder engineering problem -> async event log with idempotent writes is more robust and practical than distributed transaction (Mike: similar philophy with Unix pipeline)
- Advantages of loose coupling:
	- avoid escalating local faults into large-scale failures
	- independent teams
#### Unbundled versus integrated systems
- The goal of unbundling is to allow you to combine several different databases in order to achieve good performance for a much wider range of workloads that is possible with a single piece of software. It's about breadth
- The advantages of unbundling and composition only come into the picture when there is no single piece of software that satisfies all your requirements.
#### What's missing?
- a high level declarative language similar with Unix shell, for example you can do `mysql | elasticsearch`

### Designing Applications Around Dataflow
- The database inside-out pattern
- Even spreadsheet is better than most mainstream programming languages in terms of dataflow programming capabilities -> change a value in a cell, then the other cells with formular which refers to this cell are automatically recalculated
- This is similar with the CQRS pattern, which stands for Command and Query Responsibility Segregation
- This section explains the mechanism of the pattern. The next section describes more details about the write path and the read path 
#### Application code as a derivation function
- In data-centric view, application code is like a derivation function, which is stateless, and takes the responsibility of transformation, for example, update search index, extract feature from raw input, or update cache
- However, creating derivation function requires custom development, which is not as simple as creating secondary index 
#### Separation of application code and state
- The trend is to separate application code and its state, which is normally stored in the database
- Database becomes a kind of mutable shared variable, and it provides persistence, concurrency control, and fault tolerance
- In database, normally you could not subscribe changes to data like in spreadsheet (Mike: what about trigger?)
#### Dataflow: Interplay between state changes and application code
- In derived data, we can subscribe to the data change
- However, derivde data is not the same as async job execution, for which messaging systems are traditionally designed
	- the order of state changes is often important
	- fault tolerance is key for derived data
#### Stream processors and services
- Stream processors are similar with REST API services
- The difference is stream processors use async/subscribe way, while REST API uses sync way
- In the dataflow approach, the code that processes purchases would subsribe to a stream of exchange rate updated ahead of time, and record the current rate in a local database whenever it changes. That's why it would never require to run a sync REST API call to get the current rate. (Loose coupling, async, better performance, fewer dependency)

### Observing Derived State
- The boundary between the write path and the read path of derived data is the storage
- Take an analogy with functional programming, the write path is similar to eager evaluation, and the read path is similar to lazy evalation
#### Materialized views and caching
- The role of caches, indexes, and materialized views is simple: they shift the boundary between the read path and the write path
#### Stateful, offline-capable clients
- We can think of the on-device state as a cache of state on the server
#### Pushing state changes to clients
- In terms of our model of write path and read path, actively pushing state changes all the way to client devices means extending the write path all the way to the end user
- To solve the "device offline" problem, a consumer of a log-based message broker can reconnect after failing or becoming disconnected, and ensure that it doesn't miss any messages that arrived while it was disconnected
#### End-to-end event streams
- Keep in mind thte option of subscribing to changes, not just querying the current state
#### Reads are events too
- It is also possible to represent read requests as sterams of events, and send both the read events and the write events through a stream processor; the processor responds to read events by emitting the result of the read to an output stream
- When both the writes and the reads are represented as events, and routed to the same stream operator in order to be handled, we are in fact performing a stream-table join betweeen the stream of read queris and database
- A one-off read request just passes the request through the join operator and then immediately forgets it; a subscribe request is a persistent join with past and future events on the other side of the join
- Writing read events to durable storage thus enables better tracking of causal dependencies
#### Multi-partition data processing
- This idea of "sending queries through a stream and collecting a stream of responses" opens the possibility of distributed execution of complex queries that need to combine data from several partitions, taking advantage of the infrastructure for message routing, partitioning, and joining that is already provided by stream processors.

## Aiming for Correctness
### The End-to-End Argument for Databases
- The low-level reliability features are not by themselves sufficient to ensure end-to-end correctness
- Transactions are expensive, especially when they involve heterogeneous storage technologies
#### Exactly-once execution of an operation
- Idempotence
#### Duplicate suppression
#### Operation identifiers
- Add request id to suppress duplicate, which is similar with event sourcing 
#### The end-to-end argument
#### Applying end-to-end thinking in data systems

### Enforcing Constraints
- Apart from adding request id, another way to suppress duplication is to add "uniqueness constraint"
- There are also other kinds of constraints
#### Uniqueness constraints require consensus
- The most common way of achieving the consensus is to make a single node the leader, and put it in charge of making all th e decisions
- If you need usernames to be unique, you can partition by hash of username
#### Uniqueness in log-based messaging
- It scales easily to a large request throughtput by increasing the number of partitions, as each partition can be processed independently
- The approach works not only for uniqueness constraints, but also for many other kinds of constraints
- Its fundamental principle is that any writes that may conflict are routed to the same partition and processsed sequentially 
#### Multi-partition request processing
- It turns out that equivalent correctness (namely without an atomic commit across all three partitions) can be achieved with partitioned logs, and without an atomic commit:
	- The request to transfer money from account A to account B is given a unique request ID by the client, and appended to a log partition based on the request ID
	- A stream processor reads the log of requests. For each request message it emits two messages to output streams: a debit instruction to the payer account A (partitioned by A), and a credit instruction to the payer account B (partitioned by B). The original request ID is included in those emitted messages
	- Further processors consume the streams of credit and debit instructions, deduplicate by request ID, and apply the changes to the account balances
- By breaking down the multi-partition transaction into two differently partitioned stages and using the end-to-end request ID, we have achieved the same correctness property, even in the presence of faults, and without using an atomic commit protocol. 

### Timeliness and Integrity
- Violations of timeliness are "eventual consistency", whereas violations of integrity are "perpetual inconsistency"
#### Correctness of dataflow systems
- We achieved this integrity through a combination of mechanisms:
	- Representing the content of the write operation as a single message, which can easily be written atomically - an approach that fits very well with event sourcing
	- Deriving all other state update from that single message using deterministic derivation fuctions, similarly to stored procedures
	- Passing a client-generated request ID through all these levels of processing, enabling end-to-end duplicate suppression and idempotence
	- Making messages immutable and allowing derived data to be reprocessed from time to time, which makes it easier to recover from bugs 
#### Loosely interpreted constraints
- Many real applications can actually get away with much weaker notinons of uniqueness
- In many business conetxts, it is actually acceptable to temporarily violate a constraint and fix it up later by apologizing
- These application do requrie integrity, but they do not require timeliness on the enforcement of the constraint
#### Coordination-avoiding data systems
- Such coordination-avoiding data systems can achieve better performance and fault tolerance than systems that need to perform synchronous coordination
- For example, such a system could operate distributed across multiple datacenters in a multi-leader configuration, asynchronously replicating between regions. Any one datacenter can continue operating independently from the others, because no synchronous cross-region coordination is required. Such a system wourld have weak timelines guarantees, but it can still have strong integrity guarantees.
- In this context, serializable transactiosn are still useful as part of maintaining derived state, but they can be run at a small scope where they work well. Heterogeneous distributed transactiosn such as XA transactions are not required. Synchronous coordination can still be introduced in places where it is needed (for example, to enforce strict constraints before an operation from which recovery is not possible), but there is no need for everything to pay the cost of coordination if only a small part of an application needs it.

### Trust, but Verify
#### Maintaining integrity in the face of software bugs
#### Don't just blindly trust what they promise
#### A culture of verification
#### Desinging for auditability
- If a transaction mutates several objects in a database, it is difficult to tell after the fact what that transaction means. Even if you capture the transaction logs, the insertions, updates, and deletions in various tables do not necessarily give a clear picture of why those mutations were performed. The invocation of the application logic that decided on those mutations is transient and cannot be reproduced.
- By contrast, Event-based systems can provide better auditability. In the event sourcing approach, user input to the system is represented as a single immutable event, and any resulting state updates are derived from that event. The derivation can be made deterministic and repeatable, so that running the same log of events through the same version of the derivation code will result in the same state updates.
- Being explicit about dataflow makes the provenance of data much clearer., which makes integrity checking much more feasible. For the event log, we can use hashes to check that the event storage has not be corrupted. For any derived state, we can rerun the batch and stream processors that derived it from the event log in order to check whether we get the same result, or even run a redundant derivation in parallel.
- A deterministic and well-defined dataflow also makes it easier to debug and trace the execution of a system in order to determine why it did something. If something unexpected occurred, it is valuable to have the diagnostic capability to reproduce the exact circumstances that led to the unexpected event - a kind of time-travel debugging capability.
#### The end-to-end argument again
#### Tools for auditable data systems
- Merkle Tree
- Certificate transparency
- Distributed ledgers

