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
- The boundary between the write path and the read path of derived data
- Take an analogy with functional programming, the write path is similar to eager evaluation, and the read path is similar to lazy evalation
#### Materialized views and caching

#### Stateful, offline-capable clients
#### Pusing state changes to clients
#### End-to-end event streams
#### Reads are events too
#### Multi-partition data processing
