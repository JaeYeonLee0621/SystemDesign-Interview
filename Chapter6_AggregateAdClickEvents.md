# Chapter 6. Ad Click Event Aggregation

## Digital Advertising 
- = Real-Time Bidding (RTB)
- Digital advertising inventory is bought and sold

## RTB Process

[Image: RTB Process]

### Important Feature
- The speed : less than a second
- Data accuracy : how much money advertisers pay

+) Click-Through Rate (CTR), Conversion Rate (CVR)

# Step 1 - Understand the Problem and Establish Design Scope

### Functional requirements
- Aggregate the number of clicks of ad_id in the last M minutes
- Return the top 100 most clicked ad_id every minute
- Support aggregation filtering by different attributes
- Dataset volume is at Facebook or Google scale

### Back-of-the-envelope estimation
- 1 billion DAU
- 1 billion ad click events per day (each user click at least 1 ad)
- Ad click QPS = 10^9 events / 10^5 seconds in a day = 10,000
- Peak ad click QPS = 50,000
- 0.1KB (Single ad click event) x 1 billion (DAU) = 100GB
- Monthly storage requirement = 100GB X 30 = around 3TB

# Step 2 - Propose High-Level Design and Get Buy-In

# Data Model

[Image: Comparision]

## Choose the right database

### Write heavy
- Peak Write QPS = 50,000

### Read heavy
- Raw data is used as backup and a source for recalculation
- Read volume is low

<br/>

### Relational Database
- Scaling the write can be challenging

### NoSQL Database
- More suitable due to optimization for write and time-range queries

### Amazon S3
- Colummar data format (ex) ORC, Parquet, AVRO)
- Put a cap on the size of each file (10GB)
- Raw data could handle the file rotation when the size cap is reached

<br/>

## Aggregation Data

[Image: Aggregation Workflow]

- Time-series in nature
- The workflow is both read and write heavy
- Use the same type of database to store both raw data and aggregated data

<br/>

# Asynchronous processing

- If one component in the synchronous link is down, the whole system stops working
- Message Queue (Kafka) to decouple producers and consumers

[Image: High Level Design]

### First Message Queue 

[Image: First Message Queue]

### Second Message Queue

1. Ad click counts aggregated at per-minute granularity

[Image: Second Message Queue]

2. Top N most clicked ads aggregated at per-minute granularity

[Image: Second Message Queue]

> Why we don't write the aggregated results to the database directly?
- We need the second message queue like Kafka to achieve end-to-end exactly once semantics (atomic commit)

# Aggregation Service

[Image: DAG]

- Map Reduce framework is a good option to aggregate ad click events
- The directed acyclic graph (DAG) : The key to the DAG is to break down the system into small computing units

## Map node

[Image: Map node]

- Read data from a data source and then filters and transforms the data

> Why we need the Map node?
1. An alternative option is to set up Kafka partitions or tags
2. We may not have control over how data is produced and therefore events with the same ad_id might land in different Kafka partitions

## Aggregate node
- Count ad click events by ad_id in memory every minute

## Reduce node

[Image: Reduce node]

- Reduce aggregated results from all "Aggregation" nodes to the final results

<br/>

### DAG model
- It represents the well-known MapReduce paradigm
- It is designed to take big data and use distributed computing to turn big data into little or regular sized data
- Intermediate data can be stored in memory and different nodes communicate with each other through either TCP (node running in different processes) or shared memory (nodes running in different threads)

<br/>

# Main Use cases

## Use Case 1: Aggregate the number of clicks

[Image: UseCase1]

## Use Case 2: Return Top N most clicked ads

[Image: UseCase2]

## Use Case 3: Data Filtering

[Image: UseCase3]

- Simple to understand and build
- Can be reused to create more dimensions in the star schema
- Accessing data based on filtering criteria is fast because the result is pre-calculated

# Step 3 - Design Deep Dive

# Streaming vs Batching

[Image: Streaming vs Batching]

- Both stream processing and batch processing are used

## Streaming Processing
- Process data as it arrives and generates aggregated results in a nar real-time fashion

## Batch Processing
- Historical data backup

### Services

[Image: Services]

1. Lambda
- System contains two processing path simultaneously
- Disadvantage : There are 2 codebases to maintain

2. Kappa Architecture
- Handle both real-time data processing and continous data reprocessing using a single stream processing engine

# Data Recalculation

[Image: Recalculation Service]

- Historical data replay
ex) If we discover a major bug in the aggregation service

1. Batched job : Retrieve data from raw data storage 
2. Sent to a dedicated aggregation service so that the real-time processing is not impacted by historical data reply
3. Aggregated results are sent to the second message queue, then updated in the aggregation database

# Time
- Event time: When an ad click happens
- Processing time : refrrs to the system time of the aggregation machine that process the click event

## Late Event

[Image: Late Event]

- The gap between event time and processing time can be large due to network delays and asynchronous environments
- If processing time is used for aggregation, the aggregation result may not be accurate

[Image: Event Time]
[Image: Processing Time]

- Since data accuracy is very important, Recommend using event time for aggregation 

### Water Mark

[Image: Water Mark]

- It is commonly utilized to handle slightly delayed events

[Image: Extended Water Mark]

- The extended rectangular which is regarded as an extension of an aggregation window
- Using watermark improves data accuracy but increases overall latency due to extneded wait time

<br/>

- We can argue that it is not worth the return on investment (ROI) to have a complicated design for low probability events
- We can always correct the tiny bit of inaccuracy with end-of-day reconciliation

# Aggregation Window
- Tumbling window, Hopping window, Sliding window, Session window

## Tumbling Window

[Image: Tumbling Window]

- Time is partitioned into same-length, non-overlapping chunks
- A good fit for aggregating ad click events every minute

## Sliding Window

[Image: Sliding Window]

- Can be an overlapping one
- Satisfy our second use case; to get the top N most clicked ads during the last M minutes

# Delivery guarantees

## Which delivery method shall we choose?
- In most circumstances, at-least once processing is good enough if a small percentage of duplicates are acceptable
- We recommend exactly-once delivery for the system

## Data Duplication

### Client-Side
- A client might resend the same event multiple times

### Server outage

[Image: Server Outage]

- If an aggregation service node goes down in the middle of aggregation and the upstream service hasn't yet received an acknowledgment, the same event might be sent and aggregated again

### Solution

[Image: Record the offset]

- User the external file storage (ex) HDFS, S3)

### Problem

- If step 4 fails due to Aggregator outage, events from 100 to 110 will never be processed by a newly brought up aggregator node

### Solution

[Image: Solution]

- Save the offset once we get an acknowledgment back from downstream

### Exact Only Processing

[Image: Exact Only Processing]

- We need to put operations between step 4 to step 6 in one distributed transaction
- If any of the operation fails, the whole transaction is rolled back

### Problem
- It's not easy to dedupe data in large-scale systems

# Scale the System
- Three independent components : message queue, aggregation service, database
- Can scale each one independently

## Scale the message queue

### Producer
- We don't limit the number of producer instances, so scalability of producers can be easily archieved

### Consumer

[Image: Consumer]

- Inside a consumer group, the rebalancing mechanism helps to scale the consuming by addming or removing nodes
- It more consumers need to be added, try to do it during off-peak hours to minimize the impact

### Brokers

- Hashing Key : Using ad_id as hashing key
- The number of Partitions : Pre-allocate enough partitons in advance to avoid dynamically increasing the number of partitions in production
- Topic physical sharding : Split the data by geography or by business type

## Scale the aggregation Service

[Image: Aggregation Service]

- Horizontally scalable by adding or removing nodes

### How do we increase the throughput of the aggregation service?

Option 1: Allocate events with different ad_ids to different threads

[Image: Option 1]

Option 2: Deploy aggregation service nodes on resource providers like Apache Hadoop YARN (utilizing multi-processing)

## Scale the database
- Cassandra natively support horizontal scaling in a way similar to consistent hashing

[Image: Virtual Node]

- Data is evenly distributed to every node with a proper replication factory
- Each node saves its own part of the ring based on hashed value and also saves copies from other virtual nodes

# Hotspot Issue

- A shard or service that receives much more data than the others
- Mitigating by allocating more aggregation nodes to process popular ads

[Image: Allocate more aggregation nodes]

1. The resource manager allocates more resources so the original aggregation node isn't overloaded
2. The original aggregation node split events into 3 groups and each aggregation node handle 100 events

# Fault Tolerance
- Since aggregation happends in memory, when an aggregation node goes down, the aggregated result is lost as well

## Solution
### 1. Replaying data
- from the beginning of Kayka is slow

### 2. Save the "system status"

[Image: Data in a snapshot]

- If one aggregation service node fails, we bring up a new node and recover data from the latest snapshot

[Image: Aggregation node failover]

# Data monitoring and correctioness

## Continuous monitoring

### Latency
- It's invaluable to track timestamps as events flow through differnt parts of the system
### Message queue size
- Kafka is a message queue implemented as a distributed tommit log, so we need to monitor the records-lag metrics instead
- System resources on aggregation nodes : CPU, Disck, JVM etc

## Reconciliation

[Image: Final Design]

- Comparing different sets of data in order to ensure data integrity
- Sort the ad click events by event time in every partition at the end of the day, by using a batch job and reconciling with the real-time aggregation result
- If we have higher accuracy requirements, we can use a smaller aggregation window
- No matter which aggregation window is used, the result from the batch job might not match exactly with the real-time aggretation result

## Alternative Design

[Image: Alternative Design]

