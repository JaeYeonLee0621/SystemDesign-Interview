# Chapter 4. Distributed Message Queue

## Message Queue
- Decoupling
- Improved scalability
- Increased availability : if one part of the system goes offline, the other components can continue to interact with the queue
- Better performance : Asynchronous communication easy. Producers can add messages to a queue without waiting for the response and consumers consume message whenever they are available.

## Message queue vs event streaming platforms
- `Message Queue` : RocketMQ, ActiveMQ, RabbitMQ, ZeroMQ etc
- `Event streaming` : Kafka, Pulsar

+) Strictly speaking, Apache Kafka and Pulsar are not message queues

## [Difference between the message queue and event streaming](https://thenewstack.io/choosing-between-message-queues-and-event-streams/)

<img width="326" alt="Screenshot 2024-03-26 at 11 37 41 AM" src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ab09522f-8bc9-4ea2-92dd-d93fb0c7abd3">

### Message Queue

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b9130820-eb9e-4c9b-ab16-1b7b9a2934c6)

ex)
1. Communication between components written in different languages and "speaking" different protocols
2. Require complex message routing
3. Handling consumers that frequently disconnect

### Event Streaming

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9968b1ec-86d2-450e-a780-31ed7301f697)

- Continuous flow of event messages
- Messages can be stored in order for longer periods of time

ex)
1. Collecting, persisting and transmitting large volumes of event streams
2. Continuously processing and analyzing data
3. Log aggregation use cases

<br/>

# Step 1. Understand the Problem and Establish Design scope
- Producer send messages to a queue
- Consumers consume messages from it

<br/>

1. Only `text message` is allowed in the message queue (Message size is in the KB range)
2. Messages can be repeatedly consumed by different consumers
3. Messages should be consumed `in the same order they were produced`
4. `Data retention` is 2 weeks
5. The producers and consumers are going to suppport, the more the better
6. At `least-once data` delivery semantic need to support
7. Support high throughtput for use cases (ex) log aggregation) & low latency delivery for more traditional message queue use cases

## Non-functional requirements

- High throughput or low latency
- Scalable : the system should be distributed in nature
- Persistent and durable : data should be persisted on disk and replicated across multiple nodes

<br/>

## Adjustments for Traditional

### Message Queues

1. Do not have strong retention
2. Retain message in memory just long enough for them to be consumed
3. Do not typically maintain message ordering
4. The messages can be consumed in a different order than they were produced

# Step 2. Propose High-Level Design and Get Buy-In

![KakaoTalk_Photo_2024-03-26-09-20-25 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b29ab5a2-657f-45b2-a044-239abe6590e4)

1. Producer : sends messages to a message queue
2. Consumer : subscribes to a queue and consumes the subscribed messages
3. Message queue : a service in the middle the decouples the producers from the consumers allowing each of them to operate and scale independently

# Messaging Models

## Point to Point
- Each message can only be consumed by a `single consumer`

![KakaoTalk_Photo_2024-03-26-09-20-26 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/8978d817-8731-4834-ae81-02a95530e610)

- Once the consumer acknowledges that a message is consumed, it is removed from the queue
- There is `no data retention` in the point to point model

## Publish-Subscribe

![KakaoTalk_Photo_2024-03-26-09-20-26 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2426b1d2-ac9b-417a-86a1-46f67ee0eb4d)

- To ensure that all subscribers can receive and process each message, The message is not removed immediatley right after being consumed by consumer

1. Acknowledgement-based Removal
2. Retention Policy
3. Durable Subscription : MQTT or AMQP, durable subscriptions

ex) Google Cloud Pub/Sub : Ensure messages are retained until acknowledged by all subscribers

+) MQTT (Message Queuing Telemetry Transport)
- It is widely used iin the Internet of Things applications for machine to machine communication
- Devices can publish messages to a topic or subscribe to a topic to receive messages

+) AMQP (Advanced Message Queuing Protocol)
- Application Layer of the OSI model
- It is built around the concept of message queueing, providing a standarized way for applications to send and receive messages, decoupling the message producer from consumers

ex) Rabbit MQ

### Topic

- The categories used to organize messages
- Have a name that is unique across the entire message queue service
- Messages are sent to and read from a specific topic

# What if the data volume in a topic is too large for `a single server to handle`

- The place where messages are stored is indeed located on servers

## Partitions (Sharding)

![KakaoTalk_Photo_2024-03-26-09-20-26 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0f50300f-5354-44f0-ace2-f3818d235180)

- Partitions within a distributed messaging system as playing a role similar to a load balancer
- Partition as a small subset of the messages for a topic
- Each subset of the messages is independent
- Divide a topic into partitions and deliver messages `evenly across partitions`

## Broker

- `The server` that hold partitions
- Partitions are evenly distributed across the servers in the message queue cluster

## Offset

- The `position` of a message
- Each topic partition operated in the form of a queue with the `FIFO mechanism`
- We can `keep the order of messages` inside a partition

<br/>

1. Ensuring the order of messages
2. Enabling Message Replayability
3. Supporting Fault Tolerance
4. Facilitating Scalability and Load Balancing

## Procedure

![KakaoTalk_Photo_2024-03-26-09-20-26 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/73338fd8-5a97-4b6f-ba9b-13a4a29ec4e5)

- When a message is sent by a producer
- It is actually sent to one of the paritions for the topic
- Each message has an optional `message key`
- All messages for the same message key are sent to the same partition

+) If the message key is absent, the message is randomly sent to one of the partitions

### Difference between topic and message key

**Topic**
- Choose one category
- There are multiple partitions to allow for scalable and parallel processing of messages

**Message Key**
- The messaging system uses the key to determine which partition of the topic to send the mssage to

### Message Key is defined by Producer

**Producer**
- determining whether to include a message key and for ensuring that messages are sent to the appropriate partition
- ensuring that messages that need to be processed together or in a specific order are sent to the same partition

**Broker**
- Storing messages in their repective partitions, managing the partitions themselves and facilitating message consumption by clients

## Consumer Group
- A set of consumers, working together to consume messages from topics
- Each consumer group can subscribe to multiple topics and maintain its own consuming offsets

![KakaoTalk_Photo_2024-03-26-09-20-26 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/36515fba-0e5e-48d4-9aee-37df0657a668)

### One Problem
- Reading data in parallel improves the throughput, but `the consumption order of messages in the same partition cannot be guaranteed`

ex) Consumer 1, 2 both read from partition 1
-> Not be able to guarantee the message consumption order

### Solution

- We can fix this by adding a constraint : `A single partition` can only be consumed `by one consumer` in the same group

## High-Level Architecture

![KakaoTalk_Photo_2024-03-26-09-20-26 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7682f8a4-e4e9-42aa-bc40-335823ccb5fa)

### Core service and storage

`Broker`
- holds multiple partitions
- A partition holds a subset of messages for a topic

`Storage`
- Data storage : Messages are persisted in data storage in partitions
- State storage : consumer states are managed by state sotrage
- Metadata storage : configuration and properties of topics

`Coordination service`
- Service discovery : brokers are alive
- Leader election : one of the brokers is selected as the active controller
- active controller : responsible for assigning partitions

# Step 3. Design Deep dive

# Data storage

- Write-heavy, Read-heavy
- No update or delete operations
- Predominantly sequential read/write access

## Option 1. Database

- `Relational database` : create a topic table and write messages to the table as rows
- `NoSQL database` : create a collection as a topic and write messages as documents

<br/>

- It is hard to design a database that supports both write-heavy and read-heavy access patterns at a large scale

## Option 2. Write-ahead Log (WAL)

- A plain file where new entries are appended to an `append-only log`
- Recommend `persisting messages` as WAL log files on disk
- WAL has a pure `sequential read/write` access pattern
- Rotational disks have large capacity and they are pretty affordable
- The easiest option is to use the line number of the log file as the offset

### +) WAL (Write-Ahead Log)

- Providing atomicity and durability in database system
- Used for crash and transactino recovery
- All modifications are written to a log before they are applied
- Usually both redo and undo information is sotred in the log
- Allow the page cache to buffer updates to disk-resident pages while ensuring durability semantics in the larger context of a database system
- Persist all operations on disk until the cached copies of pages affected by these operations are synchronized on disk

### +) Kafka's Commit Log
- Each broker in a Kafka cluster maintains its own set of logs
- `The leader broker` : maintaining the active log to which new messages are written
- `The follower broker` : Replicating this log almost in real-time, maintaining copies of the leader
- s log to ensure data redundancy and fault tolerance

<br/>

### Segment

![KakaoTalk_Photo_2024-03-26-09-20-26 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7e3c782d-b7fe-48c2-86db-1cecfedbbc4a)

- A file cannot grow infinitely, so it is a good idea to `divide it into segments`
- New messages are appended only to the active segment file
- When the active segment reaches a certain size, a new active segment is created to receive new messages and the currently active segment becomes in active
- Old non-active segment files can be truncated if they exceed the retention or capacity limit

![KakaoTalk_Photo_2024-03-26-09-20-26 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1d97f5fe-0792-4d00-beeb-6b43e1c5b981)

ex) Partition-{:partition_id}

### A note on disk performance

- Only the case for random, the rotational disks are slow
- `The mordern disk drives` in a RAID configuration could comfortably achieve several hundred MB/sec of read and write speed
- `A modern OS caches disk` data in main memory very aggressively so much, so that it would happily use all available free memory to cache disk data

<br/>

## Message Data Structure

- Eliminating unnecessary data copying while the messages are in transit form the producers to the queue and finally to the consumers

### Message Key

- It is used to `determine the partition of the message`
- If the key is not defined, the partition is randomly chosen
- The key is string or number
- Otherwise the partition is chosen by `hash(key) % numberPartition` 
- If we need more flexibility, the producer can define its own mapping algorithm to choose partitions

### Message Value
- The payload of a message
- It can be plain text or a compressed binary block

### +) CRC (Cycle Redundancy Check)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/127a3ed4-4235-401f-a16b-f696e4a4f2c6)

- Encoding messages by adding a fixed-length check value, for the purpose of error detection in communication network
- Generate a short, fixed-length binary sequence, known as a CRC code or checksum from a block of data

<br/>

## Batching

- It is critical to the performance of the system
- It allows the OS to group messages together in a single network request and amortizes the cost of expensive network round trips
- The broker writes messages to the append logs in large chunks, which leads to larger blocks of sequential writes and larger contiguous blocks of disk cache, maintained by the OS

### +) Why larger blocks of sequential writes and larger continuous blocks in disk cache is efficient

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0eb74dd0-7921-4072-b290-3980f42877fa)

1. Reduced Seek time for HDDs
2. Optimized I/O Operations : Reducing the overhead of multiple write operations, prefetching larger blocks of data that are likely to be accessed soon can significantly reduce I/O wait times
3. Efficient Use of disk Cache
- larger chunk of data can be read from or sritten to the disk in a single operation
4. Reduced File System Fragmentation
5. Enhanced Throughput

### File System Fragmentation

- File system to lay out the contents of files non-continuously to allow in-place modification of their contents
- File system fragmentation negatively impacts seek time in spinning storage media

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/bac6aad6-b28b-4efb-bec8-4c0b3a4bb7af)

- Each file is using 10 blocks (4KB) of space
- If the file B is deleted, a second region of 10 blocks of free space is created
- a new file called F, which requires 7 blocks of space, also another new file called G, which needs only 3 blocks
- F needs to be expanded

1. Adding a new block somewhere else and indicating that F has a second extent ✔️
2. Moving files in the way of the expansion elsewhere, to allow F to remain contiguous : impratical for performance
3. Moving file F so it can be one contiguous file of the new, larger size : no single contiguous free space large enough to hold the new file

### How Disk Cache works

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a1cb6053-a97a-41dc-954d-9e79694f3dd3)

<br/>

- A tradeoff of large batch size is `latency`
- If the system latency might be more important, the system could be tuned to use a smaller batch size

![KakaoTalk_Photo_2024-03-26-09-20-26 012](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1a62402f-b761-4aee-9899-ea3e8123ecd9)

<br/>

# Producer Flow

- If a Producer wants to send messages to a partition, `which broker should it connect to?`

## A routing layer

### Routing Layer

![KakaoTalk_Photo_2024-03-26-09-20-26 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/03b8dd6d-a3bc-4183-83d6-7599208b9751)

- All messages sent to the routing layer are routed to the correct broker
- If the brokers are replicated, the correct broker is `the leader replica`
- Follower replicas pull data from the leader

<br/>

1. A new routing layer means additional network latency caused by overhead and addtional network hops
2. Request batching is one of the big drivers of efficiency

+) Why we need both leader and follower replicas : `Fault tolerance`

### Routing Layer in the Producer

![KakaoTalk_Photo_2024-03-26-09-20-26 011](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/4ade763a-9ed3-486c-a0d9-d1ec29dcda87)

- `Fewer network hops` mean lower latency
- Producers can have their own logic to determine which partition the message should be sent to 
- `Batching buffers messages` in memory and sends out larger batches in a single request. This increases throughput

<br/>

# Consumer flow

![KakaoTalk_Photo_2024-03-26-09-20-26 013](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9d141bd1-328c-4dc6-a334-c76af3b311b9)

## Push model

### Pros
- Low latency

### Cons
- If the rate of consumption falls below the rate of production, consumers could be overwhelmed
- It is difficult to deal with consumers with diverse processing power because the broker control the rate at which data is transfered

## Pull model
- Consumers control the consumption rate. 

### Pros
- If the rate of consumption falls below the rate of production, we can sale out the consumers, or simply catch up when it can
- The pull mode is more suitable for `batch processing

+) A pull model pulls all available messages after the consumer's current position in the log. It is suitable for aggressive batching of data.

### Cons
- When there is no message in the broker, a consumer might still keep pulling data, wasting resources.

+) To overcome this issue, many message queues support `long polling mode`

## How to pull the data

![KakaoTalk_Photo_2024-03-26-09-20-26 014](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/12f46809-63b1-4a1d-9d49-9cc2d00f382b)

1. A new consumer wants to join group 1 and subscribes to topic A
2. It finds the corresponding broker node by hashing the group name
3. All the consumers in the same group connect to the same broker 

+) Broker = the coordinator of this consumer group = the consumer group coordinator

4. The coordinator confirms that the consumer has joined the group and assigns partition 2 to the consumer

+) Partition assignment strategies : round-robin, range etc

5. Consumer fetches messages from the last consumed offset, which is managed by the state storage
6. Consumer processes messages and commits the offset to the broker

<br/>

## Consumer Rebalancing

- The coordinator plays an important role

### Coordinator

- One of the brokers responsible for communicating with consumers to achieve consumer rebalancing
- The coordinator receives heartbeat from consumers and manages their offset on the partition

![KakaoTalk_Photo_2024-03-26-09-20-26 015](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a9164fe6-128d-463b-967c-475dc0d74ad6)

- Each consumer belongs to a group, It finds the dedicated coordinator by hashing the group name
- `All consumers from the same group` are connected to `the same coordinator`
- The coordinator maintains a joined consumer list
- When the list changes the coordinator elects `a new leader of the group`
- As the new leader of the consumer group, it generates a new partition dispatch plan and reports it back to the coordinator
- The coordinator will `broadcast the plan` to the other consumers in the group

<br/>

![KakaoTalk_Photo_2024-03-26-09-20-26 016](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/635faee3-5f87-4bd6-89fa-786782dca743)

- When coordinator will no longer have heartbeats from consumers, the coordinator will trigger a rebalance process to redispatch the partitions

### New Consumer Joins

![KakaoTalk_Photo_2024-03-26-09-20-26 017](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/8d219ca6-9ce0-4e4a-b0c2-c662df007316)

### Existing consumer leaves

![KakaoTalk_Photo_2024-03-26-09-20-26 018](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d8315f5a-5e46-43a6-b67f-dcc4cc65391d)

### Existing consumer crashes

![KakaoTalk_Photo_2024-03-26-09-20-26 019](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/967b4263-a901-496a-ac55-402a68fc6784)

<br/>

## State storage

The state storage stores

- The mapping between partitions and consumers
- The last consumed offsets of consumer groups for each partition

![KakaoTalk_Photo_2024-03-26-09-20-26 020](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/56d4ffa8-c07d-41cc-a846-a17bcacea46b)

- Lots of storage solutions can be used for storing the consumer state data
- Considering the data consistency and fast read/write requirements a KV store like Zookeeper is a great choice

## Metadata storage

The metadata storage stores

- The configuration and properties of topics, including a number of partitions, retention period, and distribution of replicas
- Does not change frequently and the data volume is small, but it has a high consistency requirement

### +) Zookeeper

- It is an essential service for distributed systems offering a hierarchical key-value store

![KakaoTalk_Photo_2024-03-26-09-20-27 021](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7a5492ce-bfb4-4cf8-af79-b2217430de5e)

- Metadata and state storage are moved to Zookeeper
- The broker now only needs to maintain the data storage for messages
- Zookeeper helps with the leader election of the broker cluster

<br/>

## Replication

- Each partition has multiple replicas, distributed across different broker nodes
- The highlighted replicas are the leaders and the others are followers
- Producers only send messages to `the leader replica`
- The follower replicas keep pulling new synchronized to enough replicas, the leader returns an aknowledgment to the producer

![KakaoTalk_Photo_2024-03-26-09-20-27 022](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0e64c246-5ae2-40a2-a30c-c0d6974666b7)

- Replica distribution plan
- the leader generates the replica distribution plan and persists the plan in metadata storage

## In-Sync replicas

- Messages are persisted in multiple partitions to avoid single node failure, and each partition has multiple replicas
- Messages are only written to the leader, and followers synchronize data from the leader

### How to keep them in sync

- `ISR (In-Sync Replicas)` : Replicas that are "In-sync" with the leader
- In-sync depends on the topic configuration

### How ISR works?

![KakaoTalk_Photo_2024-03-26-09-20-27 023](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/3a0419bc-3688-4efa-95d3-26e97789c689)

+) `Committed offset` : All messages before and at this offset are already synchronized to all the replicas in ISR

- Replica-2 and 3 have fully caught up with the leader, so they are in ISR and can fetch new messages
- replica-4 did not fully catch up with the leader, so when it catches up again, it can be added to ISR

### Why do we need ISR?

- ISR reflects the trade-off between performance and durability
- The safest way to do that is to ensure all replicas are already in sync before sending an acknowledgment
- A slow replica will cause the whole partition to become slow or unavailable

<br/>

### ACK=all

![KakaoTalk_Photo_2024-03-26-09-20-27 024](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d23ddab3-141d-4a68-b808-aeb8fea3e447)

- The producer gets an ACK when all ISRs have received the message
- This means it takes a longer time to send a message because we need to wait for the slowest ISR, but it gives the strongest message durability

### ACK=1

![KakaoTalk_Photo_2024-03-26-09-20-27 025](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ef149d99-f156-49e5-aaa4-1ad8f9ffbe10)

- The producer receives an ACK once the leader persists the message
- The latench is improved by not waiting for data synchronization
- If the leader fails immediately after a message is acknowledged, then the message is lost

### ACK=0

![KakaoTalk_Photo_2024-03-26-09-20-27 026](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b5baeb4c-994d-4cfb-8977-870779376996)

- The producer keeps sending messages to the leader without waiting for any acknowledgment, and it never retries

<br/>

## Consumer Side

- The easiest setup is to let consumers connect to a leader replica to consume messages

1. Design and operational simplicity
2. Since message in one partition are dispatched to only one consumer within a consumer group, this limits the number of connections to the lead replica
3. The number of connections to the leader replicas is usually not large as long as a topic is not super hot
4. If a topic is hot, we can scale by expanding the number of partitions and consumers

+) But if consumer is located in a different data center from the leader replica, the read performance suffers

### How does ISR determine if a replica is ISR or not
- The leader for every partition tracks `the ISR list by computing the lag of every replica from itself

<br/>

# Scalability

## Producer
- The scalability of producer can easily be achieved by adding and removing producer instances

## Consumer

- Consumer groups are isolated from each other
- Inside a consumer group, the rebalancing mechanism helps to handle the cases where a consumer gets added or removed, or when it crashes

# Broker

## Broker node crashes

![KakaoTalk_Photo_2024-03-26-09-20-27 027](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d3b07bd5-2108-4195-8293-7432c179db19)

- To make the broker fault-tolerant, here are addtional consideration

1. The minimum number of ISRs specifies how many replicas the producer must receive before a message is considered to be successfully committed
2. If all replicas of a partition are in the same broker node, then we cannot tolerate the failure of this node
3. If all the replicas of a partition crash, the data for that partition is lost forever

- It is safer to distribute replicas across data centers, but this will occur much more latency and cost to synchronize data between replicas

<br/>

## Add new broker node

![KakaoTalk_Photo_2024-03-26-09-20-27 028](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c5188537-3fb1-4283-bf2d-1740d65dbca5)


- The broker controller can temporarily allow more replicas in the system than the number of replicas in the config file
- When the newly added broker catches up, we can remove the ones that are no longer needed

<br/>

# Partition

## Increase the number of partitions

- When the number of partitions changes, the producer will be notified after it communicates with any broker, and the consumer will trigger consumer rebalancing

![KakaoTalk_Photo_2024-03-26-09-20-27 029](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a53ac73f-18b7-45a4-a2c0-653f66c1edfb)


- Persisted messages are still in the old partitions, so there's no data migration
- After the new partition is added, new messages will be persisted in all new partitions

## Decrease the number of partitions

![KakaoTalk_Photo_2024-03-26-09-20-27 030](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7d4cde05-792b-4814-9a8b-b7d2a2db93a4)

- The decommissioned partition cannot be removed immediately because data might be currently consumed by consumers for a certain amount of time
- Only after the configured retention period passes, data can be truncated and storage space is freed up
- During this transitional period, producers only send messages to the remaining partitions, but consumers can still consume from all partitons
- After the retention period of the decommissioned partition expires, consumer groups need rebalancing

# Data delivery semantics

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/02a812b7-43df-4fa1-a5d2-417135216b54)

## At-most once
- A message will be delivered not more than once
- Messages may belost but are not redelivered

ex) ACK=0

## At-least once
- It's acceptable to deliver a message more than once, but no message should be lost
- Consumer fetches the message and commits the offest only after the data is successfully processed
- A message might be delivered more than once to the brokers and consumers

## [Exactly once](https://exactly-once.github.io/posts/exactly-once-delivery/)
- The most difficult delivery sementic to implement
- It is a popular concept and a desirable
+) [Kafka that claim to support exactly-once semantics.](https://kafka.apache.org/documentation/#semantics)


<br/>

# Advanced features

## Message filtering
- Topic : A logical abstraction that contains message of the same type
- Some consumer groups may only want to consume messages of certain subtypes

### Solution 1
- The consumer fetches the full set of messages and filters out unnecessary messages during processing time

### Solution 2

![KakaoTalk_Photo_2024-03-26-09-20-34 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/883d137a-cd4e-4817-98a0-d474497d3e6c)

- On the broker side so that consumers will only get messages they care about
- If data filtering requires data decryption or deserialization, it will degrade the performance of the brokers
- So it's better to attach tag and the messages can be filtered in multiple dimensions

ex) ACK=all, ACK=1

<br/>

# Delayed messages & Scheduled messages

![KakaoTalk_Photo_2024-03-26-09-20-35 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/207e4898-6bf9-475c-990a-98f840158447)

- We can send delayed messages to temporary storage on the broker side instead of the topics immediately
- Deliver them to the topics when time's up
