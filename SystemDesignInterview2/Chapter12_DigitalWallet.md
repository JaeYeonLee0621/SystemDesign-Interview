# Chapter 12. Digital Wallet

![KakaoTalk_Photo_2024-06-19-11-39-54 011](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/605bdd76-1c50-4254-9f48-80dd1c0eaa05)

- Digital wallet can store money and spend it later

ex) You can add money to your digital wallet from your bank card
when you buy products from an e-commerce website
you are given the option to pay using the money in your wallet

## Compared with the bank-to-bank transfer

- Direct transfer between digital wallets is faster
- Digital wallet does not charge an extra fee

<br/><hr/><br/>

# Step 1. Understand the Problem and Establish Design Scope

- Support balance trasfer operation between 2 digital wallets
- Support 1,000,000 TPS
- Reliability is at least 99.99%
- Support transactions
- Support reproducibility

# Back-of-the-envelope estimation

## TPS (Transaction Per Second)

- Each transfer command requires 2 operations

1. Deducting money 
2. Depositing money

ex) Database node : 1,000 TPS / (Reaching : 1 million TPS x 2 operations) = 1,000 database nodes

- Due to decreasing financial expense, increasing the number of transaction a single node can handle

# In-memory sharding solution

- `<user, balance> relationship` => a hash table (map) or key-value store

## Redis

- One Redis node is not enough to handle 1 million TPS

### Partitioning or Sharding

`Partition = hashing(accountID) % Partition Number`

### Transfer commands = Wallet Service

![KakaoTalk_Photo_2024-06-19-11-41-45](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/88eac6b7-cc5d-42a9-bc54-a2e5f92b8611)

1. Receiving the transfer command
2. Validating the transfer command
3. If the command is valid, it updates the account balances for the 2 users involved in the transfer, most likely in 2 different Redis nodes

<br/><hr/><br/>

# Step 2. Propose High-Level Design and Get Buy-In

# Distributed transaction - Database Sharding

## Q) How do we make the updates to 2 different storage node atomic?

- Replacing each redis node with a transactional relational database node

![KakaoTalk_Photo_2024-06-19-11-39-53 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2a1fc5d1-c3e9-4c3c-8ff9-7a5e21c5faa2)

- Using `transactional databases` only solves part of the problem
- There is no guarantee that 2 update operations will be handled at exactly the same time

# A-1) Distributed Transaction: 2 phase commit (2PC)

![KakaoTalk_Photo_2024-06-19-11-39-54 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/38888508-462c-4f4e-bc8e-a66395bbdbb6)

![KakaoTalk_Photo_2024-06-19-11-39-54 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/13d042fd-f7fb-41fd-a091-2340d3416614)

# A-2) Distributed transaction: Try Confirm/Cancel (TC/C)

- Type of compensating transaction

1. In the first phase, the coordinator asks all databases to reserve resources for the transaction
2. In the second phase, the coordinator collects replies from all databases:

### 2-1) `the Try-Confirm` process
- If all databases reply with "yes" : The coordinator asks all databases to confirm the operation 

### 2-2) `the Try-Cancel` process
- If any database replies with "no" : The coordinator asks all databases to cancel the operation

ex)

1. Try

![KakaoTalk_Photo_2024-06-19-11-39-54 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/edcec023-3fd0-4c50-a561-349ae4e5d292)

- For the database that contains account C, the coordinator gives it a NOP (no operation)
- Let's assume the coordinator sends to this database a NOP command
- The database does nothing for `NOP commands` and `always replies to the coordinator with a success message`

2. Confirm

![KakaoTalk_Photo_2024-06-19-11-39-54 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/01cd5ed2-24be-4cb1-828f-4e5b2a11ac76)

- If both databases reply "yes", the wallet service starts the next Confirm phase

3. Cancel

![KakaoTalk_Photo_2024-06-19-11-39-54 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/aa4833a5-6417-4e87-8ad4-43c2173a89af)

# Potential problems

## 1. Phase status table

- Q) If the wallet service restarted right after it updated the first account balance, how can we make sure the second account will be updated as well?
- We can store the progress of a TC/C as phase status in a transactional table

![KakaoTalk_Photo_2024-06-19-11-39-54 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2dfe91b9-f888-4f4f-8e27-fe21d6a38d0b)

## 2. Unbalanced balance

### 2-1) TC/C comprises several independent local transactions

- TC/C is driven by application, the application itself is able tosee the intermediate result between these local transactions

### 2-2) The database transaction or 2PC version of the distributed transaction

- Maintaining by databases that are invisible to high-level applications

## 3. Data discrepancy

![KakaoTalk_Photo_2024-06-19-11-39-54 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/eefb1ceb-478a-4d1e-a7f4-32d985696b63)

- This discrepancies might be transparent to us because lower-level systems such as databases already fixed the discrepancies
- If not we have to handle it ourselves

## 4. Out of order execution

### Problems

- One side effect of TC/C is the out-of-order execution

![KakaoTalk_Photo_2024-06-19-11-39-54 011](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/6beb4242-d1da-485b-93ca-44ea17700196)

### Solutions

- Each node is allowed to Cancel a TC/C without receiving a Try instruction by enhancing the existing logic

1. The out-of-order Cancel operation leaves a flag in the database indicating that it has seen a Cancel operation, but it has not seen a Try operation yet

2. The Try operation is enhanced so it always checks whether there is an out-of-order flag, and it returns a failure if there is

# A-3) Distributed transaction: Saga Linear order execution

- There is another popular distributed transaction solution
- Saga is the de-facto standard in a microsservice architecture

![KakaoTalk_Photo_2024-06-19-11-39-54 012](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/8a6a39f0-cd86-459e-8bfe-6b13ff2b3178)

1. All operations are ordered `in a sequence`. Each operation is `an independent transaction` on its own database.
2. Operations are executed `from the first to the last`. When one operation has fininshed, the nest operation is triggered
3. When an operation has failed, `the entire process starts to roll back` from the current operation to the first operation in reverse order, using compensating transactions. So if a distributed transaction has operations, we need to perpare opreations: fot the normal case and another for the compenstaing transaction during rollback

### 1. Choreography

- In a microservice architecture, all the services involved in the Saga distributed transaction do their jobs by subscribing to other services' events. So it is fuly decentralized coordination

### 2. Orchestration

- A single coordinator instructs all services to do their jobs in the correct order

> The orchestration solution handels complexity well, so it is usually the preferred solution in a digital wallet system

<br/><br/>

## Comparision 3 solutions

### TC/C
- If the system is latency-sensitive and contains many services/operations
- If a database supports transactions

### Saga
- If there is no latnecy requirement, or there are very few services

<br/><hr/><br/>

# Event Sourcing

- Event Sourcing is a technique developed in `Domain Driven Design (DDD)`

## 1. Command

- It is the intended action from the outside world
- Commands are usually put into a FIFO queue

## 2. Event

- The result of the fulfillment

ex) A command must be validated before we do anything about it => It is valid and must be fulfilled

### Differences

1. Events
- must be executed because they represent a validated fact
- must be dterministic
- represent historical facts

2. Commands
- may contain randomness or I/O

### Event generation process

1. One command may generate any number of events
2. Event generation may contain randomness meaning it is not guaranteed that a Command always generates the same event(s)

- Events are stored in a FIFO queue

## 3. State

- It is what will be changed when an Event is applied

### State Machine

**[2 major functions]**

1. Validate commands and generate Events
2. Apply Event to update State

- Event sourcing requires the behavior of the State Machine to be deterministic
- Therefore, the State Machine itself should never contain any randomness

![KakaoTalk_Photo_2024-06-19-11-39-54 013](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a35ad22d-3da3-42b8-8c82-3c7927955ab1)

- The state machine is responsible for converting the Command to an Event and for applying the Event

![KakaoTalk_Photo_2024-06-19-11-39-54 014](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c541e2ec-b882-4a7d-bd50-00a24e126823)

<br/>

ex) The state machine works

![KakaoTalk_Photo_2024-06-19-11-39-54 015](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/04c1d820-6e58-48ab-92ac-247848acbb8f)

1. Read command from the command queue (usually Kafka)
2. Read balance state from the database
3. Validate the command. If it is valid, generate 2 events for each of the accounts
4. Read the next event
4. Apply the event by updating the blance in the database

<br/>

### Advantages of Event sourcing : Reproducibility

![KakaoTalk_Photo_2024-06-19-11-39-54 016](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5aa91bf7-bb60-4916-ba1a-4fbfa4e6679d)

- All changes are saved first as immutable history
- The database is only used as an updated view of what balance looks like at any given point in time

# Command-Query responsibility segragation (CQRS)

- There is one State Machine responsible for the write part of the state, but there can be many read-only state machines
- Pushlishing all the events
- The external world could rebuild any customized state itself

![KakaoTalk_Photo_2024-06-19-11-39-54 017](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/bc7450d7-56a7-4829-a3aa-6c89f669e185)

- The read-only State Machines lag behind to some extent, but will always catch up
- The architecture design is `eventually consistent`

<br/><hr/><br/>

# Step 3. Design Deep Dive

# 1. High-performance event sourcing
- Kafka as the command and event store
- The database as a State store

## 1-1. File-based command and event list

- Saving command and event in a local disk, rather than to a remote store like Kafka
- This avoid transit time across the network
- The event list uses an append only data structure
- Appending is a sequential write operations
- It works well even for magnetic hard drives because the OS is heavily optimized for sequential reads and writes

## 1-2. Cache Recent commands and Events in memory

![KakaoTalk_Photo_2024-06-19-11-39-54 018](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/887723a9-13b0-40ea-bf3d-74f2f29933a6)

- We process Command and Event right after they are persisted
- We may cache them in memory to save the time of loading them back from the local disk

### mmap

- It can write to a local disk and cache recent content in memory at the same time
- The OS caches certain sections of the file in memory to accelerate the read and write operations
- It is almost guaranteed that all data are saved in memory which is very fast

## 1-3. File-based State

![KakaoTalk_Photo_2024-06-19-11-39-54 019](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/cca122b0-7de3-4a4c-8b08-38b34c3912a8)

- State information can be saved to the local disk
- We can use `SQLite` which is a file-based local relational database which is a local file-based key-value store

<br/>

- RocksDB is chosen because it uses a log-structured merge-tree (LSM), which is optimized for write opertaions
- To improve read performance, the most recent data is cached

## 1-4. Snapshot

![KakaoTalk_Photo_2024-06-19-11-39-54 020](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/634de101-94db-4594-a487-d503e86c6017)

- How to acceleate the reproductibility process
- A snapshot is an immutable view of a historical State
- Once a snapshot is saved, the State Machine does not have to restart from the very beginning anymore
- It can read data from a snapshot, verify where it left off, and resume processing from there
- A snapshot is a giant binary file and a common solution is to save it in an object storage solution such as HDFS

<br/><hr/><br/>

# 2. Reliable high-performance event sourcing

## 2-1. Consensus

- We need to replicate the Event list across multiple nodes

1. No data loss
2. The relative order of data within a log file remains the same across nodes

- The consensus algoithm make ssure that multiple nodes reach a consensus on what the Event list is

## ex) Raft consensus algorithm 

- The Raft algorithm guarantees that as long as more than half of the nodes are online, the append-only lists on them have the same data
- To synchronize their data, as long as at least 3 of the nodes are up, the system can still work properly as a whole

### Reliable solution

![KakaoTalk_Photo_2024-06-19-11-39-54 021](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a6beb0e9-09be-472f-b3cc-be1774edad18)

1. Process

- The leader : takes incoming command requests from external users, converts them into Events and appends Events into the local Event list
- The Raft algorithm replicates newly added Events to the followers
- All nodes process the Event list an update the State > Ensuring the leader and followers have the same Event lists

2. Handling crashes - leader crashes

- If the leader crashes : automatically selects a new leader from the remaining healthy nodes
- The client would notice the issue either by a timeout or by receiving an error response
- The client needs to resend the same Command to the newly elected leader

3. Handling crashes - followers crashes

- Requests sent to it will faile
- By retrying indefinitely until the crashed node is restarted or a new one replaces it

# 3. Distributed event sourcing

## 3-1. pull vs push

### pull

![KakaoTalk_Photo_2024-06-19-11-39-55 022](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/478a58e4-8ae1-414d-af42-e244b52a9cc6)

- An external user periodically pulls execution status from the read-only State Machine
- This model is not real-time and may overload the wallet service if the pulling frequency is set too high

![KakaoTalk_Photo_2024-06-19-11-39-55 023](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2cd64e66-fdf8-41e1-b4aa-161f646d9c5a)

- It can be improved by adding a `reverse proxy` between the external user and the Event Sourcing node
- The external user sends a Command to Event Sourcing nodes and periodically pulls the execution status

### push

- We could make the response faster by modifying the read-only State Machine
- State Machine pushes execution status back to the reverse proxy, as soon as it receives the Event
- This will give the user a feeling of real-time response

## 3-2. Distributed transaction

- We can reuse the distributed transaction solution (ex) TC/C or Saga)

![KakaoTalk_Photo_2024-06-19-11-39-55 024](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/564eacba-1a54-4392-9be6-4258f7e43e1d)

1. User A sends a distributed transaction to the Saga coordinator
2. Saga coordinator creates a record in the Phase Status Table to trace the status of a transaction
3. Saga coordinator examines the order of operations and determines that it neds o handle which events first
4. Partition 1's Raft leader receives the A command and stores it in the command list then validates the command. If it is valid, it is converted into an Event. The Raft consensus algorithm is used to synchronized data across different nodes.
5. After the Event is synchronized, the Event Sourcing framework of Partition 1 synchronizes the data to the read path using CQRS.
6. The read path of Partition 1 pushes the status back to the caller of the Event Sourcing framework, which is the Saga coordinator
7. Saga coordinator receives the success status from Partitoin 1
8. The saga coordinator creates a record, indicating the operation in Partition 1 is successful, in the Phase Status Table
9. Because the first operation succeeds, the Saga coordinator executes the second operation
10. Partition 2's Raft leader receives the command and saves it and do the same thing happening in stage 4
11. After the Event is synchronized, the Event Sourcing Framework of Partition 2 synchronizes the data to the read path using CQRS
12. The read path of Partition 2 pushes the status back tot he caller of the Event Sourcing framework, which is the Saga coordinator
13. The Saga coordinator receives success status from Partition 2
14. The Saga coordinator creates a record, in dicating the operation in Partiton 2 is successful in the Phase Status Table
15. At this time, all operations succeed and the distributed transaction is completed. The ssaga coordinator responds to its caller with the result.
