# Chapter 12 The Future of Data Systems

## Data Integration

### Combining Specialized Tools by Deriving Data
- Summary
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

### Composing Data Storage Technologies

### Designing Applications Around Dataflow

### Observing Derived State
