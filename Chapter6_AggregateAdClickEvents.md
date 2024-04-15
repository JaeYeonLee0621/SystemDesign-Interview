# Chapter 6. Ad Click Event Aggregation

## Digital Advertising 
- = Real-Time Bidding (RTB)
- Digital advertising inventory is bought and sold

## RTB Process

![KakaoTalk_Photo_2024-04-15-23-48-24 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/3d4bd204-5698-4a20-a44e-3e52e4cd275d)

### Important Feature
- The speed : less than a second
- Data accuracy : how much money advertisers pay

+) Click-Through Rate (CTR), Conversion Rate (CVR)

<br/>

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

<br/>

# Step 2 - Propose High-Level Design and Get Buy-In

# Data Model

![KakaoTalk_Photo_2024-04-15-23-48-24 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f17e6ddf-76e5-4938-bfdb-99fc87d5a26e)

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

![KakaoTalk_Photo_2024-04-15-23-48-24 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0fba86a5-b6c9-4fb2-9e97-5d7ad192b600)

- Time-series in nature
- The workflow is both read and write heavy
- Use the same type of database to store both raw data and aggregated data

<br/>

# Asynchronous processing

- If one component in the synchronous link is down, the whole system stops working
- Message Queue (Kafka) to decouple producers and consumers

### First Message Queue 

![KakaoTalk_Photo_2024-04-15-23-48-24 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/836230f0-8fb9-4d33-8f3b-5a284eb85922)

### Second Message Queue

1. Ad click counts aggregated at per-minute granularity

![KakaoTalk_Photo_2024-04-15-23-48-24 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/169e5074-6d7b-484d-81ba-f9c72b57a717)

2. Top N most clicked ads aggregated at per-minute granularity

![KakaoTalk_Photo_2024-04-15-23-48-24 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c76dfd76-22b0-4bcb-ae93-54f390e082da)

> Why we don't write the aggregated results to the database directly?
- We need the second message queue like Kafka to achieve end-to-end exactly once semantics (atomic commit)

<br/>

# Aggregation Service

![KakaoTalk_Photo_2024-04-15-23-48-24 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/849538e3-0b9f-4912-8227-c3c321b69a2c)

- Map Reduce framework is a good option to aggregate ad click events
- The directed acyclic graph (DAG) : The key to the DAG is to break down the system into small computing units

## Map node

![KakaoTalk_Photo_2024-04-15-23-48-25 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a492e8bd-d63d-403f-bffa-01c664f4b0b3)

- Read data from a data source and then filters and transforms the data

> Why we need the Map node?
1. An alternative option is to set up Kafka partitions or tags
2. We may not have control over how data is produced and therefore events with the same ad_id might land in different Kafka partitions

## Aggregate node
- Count ad click events by ad_id in memory every minute

## Reduce node

![KakaoTalk_Photo_2024-04-15-23-48-25 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/e5bbecdf-63a5-481d-8f13-90186f449403)

- Reduce aggregated results from all "Aggregation" nodes to the final results

### DAG model
- It represents the well-known MapReduce paradigm
- It is designed to take big data and use distributed computing to turn big data into little or regular sized data
- Intermediate data can be stored in memory and different nodes communicate with each other through either TCP (node running in different processes) or shared memory (nodes running in different threads)

<br/>

# Main Use cases

## Use Case 1: Aggregate the number of clicks

![KakaoTalk_Photo_2024-04-15-23-48-25 011](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/52a55593-0dc8-4188-98b2-6ab103eefa5d)

## Use Case 2: Return Top N most clicked ads

![KakaoTalk_Photo_2024-04-15-23-48-25 012](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/939dc3db-94ed-4a27-a297-72de302d8fe5)

## Use Case 3: Data Filtering

![KakaoTalk_Photo_2024-04-15-23-48-25 013](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/60e5e6b6-337f-4726-acbd-e164adab0c66)

- Simple to understand and build
- Can be reused to create more dimensions in the star schema
- Accessing data based on filtering criteria is fast because the result is pre-calculated

<br/>

# Step 3 - Design Deep Dive

# Streaming vs Batching

![KakaoTalk_Photo_2024-04-15-23-48-25 014](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0aa02200-5005-4da3-ac09-ec650a988db5)

- Both stream processing and batch processing are used

## Streaming Processing
- Process data as it arrives and generates aggregated results in a nar real-time fashion

## Batch Processing
- Historical data backup

### Services

![KakaoTalk_Photo_2024-04-15-23-48-25 015](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c9264ce5-3eaa-411e-a1fb-aa7876265b35)

1. Lambda
- System contains two processing path simultaneously
- Disadvantage : There are 2 codebases to maintain

2. Kappa Architecture
- Handle both real-time data processing and continous data reprocessing using a single stream processing engine

# Data Recalculation

![KakaoTalk_Photo_2024-04-15-23-48-25 016](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/396c1190-6b3f-45bd-af56-8c1efcc18b4e)

- Historical data replay
ex) If we discover a major bug in the aggregation service

1. Batched job : Retrieve data from raw data storage 
2. Sent to a dedicated aggregation service so that the real-time processing is not impacted by historical data reply
3. Aggregated results are sent to the second message queue, then updated in the aggregation database

<br/>

# Time
- Event time: When an ad click happens
- Processing time : refrrs to the system time of the aggregation machine that process the click event

## Late Event

![KakaoTalk_Photo_2024-04-15-23-48-25 017](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5bace2e1-38e1-40b0-be1e-708070a7ac1d)

- The gap between event time and processing time can be large due to network delays and asynchronous environments
- If processing time is used for aggregation, the aggregation result may not be accurate

![KakaoTalk_Photo_2024-04-15-23-48-25 018](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d02c9c57-9bd8-4d81-8c8e-f7488a7fad66)
![KakaoTalk_Photo_2024-04-15-23-48-25 019](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/02e15f1d-e8fb-4908-917a-efd5c3bd8264)

- Since data accuracy is very important, Recommend using event time for aggregation 

### Water Mark

![KakaoTalk_Photo_2024-04-15-23-48-25 020](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b29ee8a4-efc6-4f54-8c4c-880c998b2110)

- It is commonly utilized to handle slightly delayed events

![KakaoTalk_Photo_2024-04-15-23-48-25 021](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/466c0cfd-36b5-4fcc-aa75-9c7c6fcc389c)

- The extended rectangular which is regarded as an extension of an aggregation window
- Using watermark improves data accuracy but increases overall latency due to extneded wait time

<br/>

- We can argue that it is not worth the return on investment (ROI) to have a complicated design for low probability events
- We can always correct the tiny bit of inaccuracy with end-of-day reconciliation

<br/>

# Aggregation Window
- Tumbling window, Hopping window, Sliding window, Session window

## Tumbling Window

![KakaoTalk_Photo_2024-04-15-23-48-25 022](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5f14c66b-f8b1-4a10-92cd-2f5f53b9a35a)

- Time is partitioned into same-length, non-overlapping chunks
- A good fit for aggregating ad click events every minute

## Sliding Window

![KakaoTalk_Photo_2024-04-15-23-48-25 023](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/3f63abd4-c816-4248-9877-eaa300385167)

- Can be an overlapping one
- Satisfy our second use case; to get the top N most clicked ads during the last M minutes

<br/>

# Delivery guarantees

## Which delivery method shall we choose?
- In most circumstances, at-least once processing is good enough if a small percentage of duplicates are acceptable
- We recommend exactly-once delivery for the system

## Data Duplication

### Client-Side
- A client might resend the same event multiple times

### Server outage

![KakaoTalk_Photo_2024-04-15-23-48-25 024](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/77d2d22b-7c52-48c8-87e0-1a3a5dc9ef62)

- If an aggregation service node goes down in the middle of aggregation and the upstream service hasn't yet received an acknowledgment, the same event might be sent and aggregated again

### Solution

![KakaoTalk_Photo_2024-04-15-23-48-25 025](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/02fd9d48-9f35-4fd4-aaa2-4b0e0db30025)

- User the external file storage (ex) HDFS, S3)

### Problem

- If step 4 fails due to Aggregator outage, events from 100 to 110 will never be processed by a newly brought up aggregator node

### Solution

![KakaoTalk_Photo_2024-04-15-23-48-26 026](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9c74967c-9b38-41fc-9c04-20a6722b9a2a)

- Save the offset once we get an acknowledgment back from downstream

### Exact Only Processing

![KakaoTalk_Photo_2024-04-15-23-48-26 027](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/575d33e2-60b0-4284-b001-60c3b14b955a)

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

![KakaoTalk_Photo_2024-04-15-23-48-26 028](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1b3319b0-10be-4a14-97c4-fdf8605fde76)

- Inside a consumer group, the rebalancing mechanism helps to scale the consuming by addming or removing nodes
- It more consumers need to be added, try to do it during off-peak hours to minimize the impact

### Brokers

- Hashing Key : Using ad_id as hashing key
- The number of Partitions : Pre-allocate enough partitons in advance to avoid dynamically increasing the number of partitions in production
- Topic physical sharding : Split the data by geography or by business type

## Scale the aggregation Service

![KakaoTalk_Photo_2024-04-15-23-48-26 029](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/15425367-a5cb-4ef3-aa60-1da51aee2174)

- Horizontally scalable by adding or removing nodes

### How do we increase the throughput of the aggregation service?

- Option 1: Allocate events with different ad_ids to different threads

![KakaoTalk_Photo_2024-04-15-23-48-26 030](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d50815e9-bd51-48d2-b52a-d10fe2e18b18)

- Option 2: Deploy aggregation service nodes on resource providers like Apache Hadoop YARN (utilizing multi-processing)

## Scale the database
- Cassandra natively support horizontal scaling in a way similar to consistent hashing

![KakaoTalk_Photo_2024-04-15-23-48-36 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/49210e16-b775-4197-8e1e-39dab1bdeb77)

- Data is evenly distributed to every node with a proper replication factory
- Each node saves its own part of the ring based on hashed value and also saves copies from other virtual nodes

<br/>

# Hotspot Issue

- A shard or service that receives much more data than the others
- Mitigating by allocating more aggregation nodes to process popular ads

![KakaoTalk_Photo_2024-04-15-23-48-36 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2a792f03-04df-4a48-b7ca-980ff036eb7c)

1. The resource manager allocates more resources so the original aggregation node isn't overloaded
2. The original aggregation node split events into 3 groups and each aggregation node handle 100 events

# Fault Tolerance
- Since aggregation happends in memory, when an aggregation node goes down, the aggregated result is lost as well

## Solution

### 1. Replaying data
- from the beginning of Kayka is slow

### 2. Save the "system status"

![KakaoTalk_Photo_2024-04-15-23-48-36 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5e5e9b6e-6e43-4483-bb9e-070dbe60350b)

- If one aggregation service node fails, we bring up a new node and recover data from the latest snapshot

![KakaoTalk_Photo_2024-04-15-23-48-36 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b0d97aa2-520e-4e0f-9f7a-cc0751a8aafb)

<br/>

# Data monitoring and correctioness

## Continuous monitoring

### Latency
- It's invaluable to track timestamps as events flow through differnt parts of the system
### Message queue size
- Kafka is a message queue implemented as a distributed tommit log, so we need to monitor the records-lag metrics instead
- System resources on aggregation nodes : CPU, Disck, JVM etc

## Reconciliation

![KakaoTalk_Photo_2024-04-15-23-48-36 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/618aad82-6f4a-4007-939e-02970fe958b2)

- Comparing different sets of data in order to ensure data integrity
- Sort the ad click events by event time in every partition at the end of the day, by using a batch job and reconciling with the real-time aggregation result
- If we have higher accuracy requirements, we can use a smaller aggregation window
- No matter which aggregation window is used, the result from the batch job might not match exactly with the real-time aggretation result

## Alternative Design

![KakaoTalk_Photo_2024-04-15-23-48-37 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9f494c76-d674-48c4-94ce-4943b4be5a0e)

