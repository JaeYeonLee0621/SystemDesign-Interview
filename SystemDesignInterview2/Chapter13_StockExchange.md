# Chapter 13. Stock Exchange

# Step 1. Understand the Problem and Establish Design Scope

## Non-functional requirements

- `Availability` : At least 99.99%
- `Fault tolerance` : Limiting the impact of a production incident
- `Latency` : Focus on the 99th percentile latency
- `Security` : Should have an account management system (ex) KYC (Know Your Client), DDoS (Distributed Denial-Of-Service)

## Back-of-the-envelope estimation

- QPS = 1 billion / 6.5 / 3600 ~= 43,000
- 1 billion orders per day
- NYSE Stock exchange is open Monday ~ Friday (09:30 ~ 16:00)

> Peak QPS = 5 x QPS = 215,000

# Step 2. Propose High-Level Design and Get Buy-In

# Business Knowledge 101

## Broker

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ef6fe0a0-08d0-43ad-8ebc-1f638d90bf0e)

- Most retail clients trade with an exchange via a broker

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/41578006-f230-4675-a25f-54fd0d4783c1)

- These brokers provide a friendly user interface for retail users to place trades and view market data

## Institutional Client
- Trading in large volumes using specialized trading software
- Diferrent institutional clients operate with different requirements

ex) pension funds (aiming for a stable income)
- Trade infrequently but the volume is large
- Therefore, they need features like order splitting to minimize the market impact of their sizable orders

ex) hudge fund (earning income via commission rebates)
- Low lateny trading abilities
- they can't simply view market data on a web page or a mobile app, as retail clients do

## Limit Order

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/affd096b-8893-448c-af1c-d205bc99671a)

- buy or sell order with a fixed price
- It might not find a match immediately, or it might just be partially matched

## Market Order

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/511e0aa4-c9e0-40dc-b79c-3289fc87b1cd)

- Doesn't specify a price
- Prevailing market price immediately

## Market data levels

### Bid price
- The highest price `a buyer is willing to pay` for a stock

### Ask price
- The lowest price `a seller is wiling to sell` the stock

### L1

![KakaoTalk_Photo_2024-06-01-14-02-12 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/416b9ae2-9443-4419-8f7f-b001e841d0db)

- Basic Information

1. Bid and Ask Prices
2. Last Trade : The most recent trade
3. Volume : Total number of shares traded during a given period
4. Summary Statistics

### L2

![KakaoTalk_Photo_2024-06-01-14-02-12 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c84240d7-1acd-42a1-a6b1-d72a9942c0d2)

- Order Book : Real-time bids and asks from market participants at different price levels
- Better Transparency : more informed decision-making

### L3

![KakaoTalk_Photo_2024-06-01-14-02-12 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/361dd9a9-3326-4121-a6ea-58f1e68c45cf)

- Full Market View, Order Book Details, Market Maker Activity, Best for Professional

## Candlestick chart

![KakaoTalk_Photo_2024-06-01-14-02-12 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c5b9bd32-eb39-47d2-bf1c-48e441042ed9)

## FIX (Financial Information eXchange) Protocol

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5db084b0-67be-4a6c-8c97-692aab564909)

- A vender-neutral communication protocol for exchanging securities transaction information

```
8=FIX.4.2|9=65|35=A|49=SERVER|56=CLIENT|34=177|52=20090107-18:15:16|98=0|108=30|10=062|
```

<br/>

# High Level Design

![KakaoTalk_Photo_2024-06-01-14-02-13 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7fe00633-eb86-44d9-bd06-a352305689cc)

Step 3: Client gateway
- Input validation, rate limiting, authentication, normalization, etc

Step 4-5: Order manager
- Risk checks based on rules set by the risk manager

Step 6: Order manager
- Verify there are sufficient funds in the wallet for the order

Step 7-9: Matching engine
- Emit 2 executions with one each for the buy and sell sides
- To guarantee that matching results are deterministic when replay, both orders and executions are sequenced in the sequencer

Step M1: Matching engine
- Generate a stream of executions

Step M2: Market data publisher
- Construct the candlestick charts and the order books from the stream of executions as market data

Step M3: Market data publisher
- Sending the market data to the data service
- The published market data is saved to specialized storage for real-time analytics
- The brokers connect to the data service to obtain timely market data
- Brokers relay market data to their clients

Step R1-R2 (Reporting flow)
- The reporter collects all the necessary reporting fields from orders and executions and writes the consolidated records to the database

# Trading flow

## Matching machine (called cross engine)

### Responsibilities
1. Maintaining the roder book for each symbol
2. Match buy and sell orders
3. Distribute the execution stream as market data

### Sequence of Orders
- Produce matches in a `deterministic order`
- The matching engine must produce the same sequence of executions as an output when the sequence is replayed

## Sequencer

![KakaoTalk_Photo_2024-06-01-14-02-13 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/e359cad2-83cd-4145-9654-7f88bb7c399b)

- Make the matching engine deterministic
- Stamp every incoming order with a sequence ID before it is processed by the matching engine
- Inbound and Outbound have each sequencer and each maintaining its own sequences
- Function as message queue

### Reasons for Stamping sequence IDs
1. Timeliness and fairness
2. Fast recovery / replay
3. Exactly-once guarantee

## Order Manager
- Receiving orders on one end and receives executions on the other
- Should be fast, efficient, accurate
- Maintain the current states for the orders

### Performing
- Send the order for risk checks
- User's wallet, check there are sufficient funds to cover the trade
- Send the order to the sequencer where the order is stamped with a sequence ID (Order manager only sends the necessary attributes)

## Client Gateway

![KakaoTalk_Photo_2024-06-01-14-02-13 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/989f7ab1-b1f7-44f7-ad9b-3b020e41aa2e)

- Gatekeeper for the exchange
- Crtical path, latency-sensitive, lightweight

### Colocation (colo) Engine
- Trading engine software running on some servers rented by broker in the exchange's data center
- The latency is literally the time it takes for light to travel from the colocated server to the exchange server

## Market Data Flow

![KakaoTalk_Photo_2024-06-01-14-02-13 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/6c246442-ec14-44a0-acee-6c58f8309be0)

### MDP (Market Data Publisher)
- Receive executions (files) from the matching engine
- Build the order books and candlestick charts from the stream of executions

## Reporting Flow

![KakaoTalk_Photo_2024-06-01-14-02-13 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/bd394ab6-0482-445a-8b9b-7a8e9e80e604)

- Provide trading history, tax reporting, compliance reporting, settlements etc
- Less sensitive to latency
- The reporter merges the attributes from both sources for the reports

## Data Models

### Prodct, Order, Execution
- Order : The inbound instruction for a buy or sell order
- Execution (= a fill) : The outbound matched result, not every order has an execution

![KakaoTalk_Photo_2024-06-01-14-06-39](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/e77a6a3b-e4a8-4750-9d66-3d6079db915a)

- Orders and executions are not stored in a database but stored in `the sequencer` for fast recovery, data is archived after the market closes
- The reporter writes orders and executions to the database for reporting use cases like reconciliation and tax reporting
- Executions are forwarded to `the market data processor` to reconstruct the order book and candlestick chart data

## Order book

- A list of buy and sell orders for a specific security or financial instrument, organized by price level

ex)

![KakaoTalk_Photo_2024-06-01-14-07-39](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/356efe5f-1582-4be1-9190-29ba322599d8)

### Requirements
- Constant lookup time
- Fast add/cancel/execute operations, preferably O(1) time complexity
- Fast update

### O(1)

![KakaoTalk_Photo_2024-06-01-14-02-13 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ec1d4671-5c09-42af-bbb7-167ca43e98c0)

- Doubly-linked List : deletion type of operation is also O(1)

## Candlestick chart
- Tracking price history in candlestick charts for many symbols at many time intervals consumes a lot of money

### Optimization
1. Use pre-allocated ring buffer to hold sticks to reduce the number of new object allocations
2. Limit the number of sticks in the memory and persist the rest of disk

- Market data is usually persisted in an in-memory columnar database for real-time analytics
- The market is closed, data is persisted in a historical database

# Step 3. Design Deep Dive

# Performance

- Latency is very important for an exchange
- The level of stability is the 99th percentile latency

## 2 ways to reduce latency

### 1. Decrease the number of tasks on the critical path

- Critical Path : Only containing the necessary components
```
gateway -> order manager -> sequencer -> matching engine
```
- Even logging is removed from the critical path to achieve low latency


### 2. Shorten the time spent on each task
- By reducing or eliminating network and disk usage
- By reducing execution time for each task

**[Total End-To-End Latency]**

+) [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832)

1. Network Latency
- The round trip network latency ~= 500ms
- Multiple components = 500ms x multiple components ~= single-digit milliseconds

2. Disk Latency
- The sequencer is an event store that persists events to disk
- The latency of disk access (by sequencer) ~= tens of milliseconds

> Total end-to-end latency ~= tens of milliseconds

- The time-tested design `eliminates the network hops` by putting everthing in `one server` > mmap as event store

## Single Server Architecture

![KakaoTalk_Photo_2024-06-01-14-02-13 011](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/dc562fd4-6ecd-4292-863b-50672d551016)

### Application Loop

- Keep polling for tasks to execute in a while loop and is the primary task execution mechanism
- Only the most mission critical tasks should be processed by the application loop
- Reducing the execution time for each component and to guarantee a highly predictable execution time

### Component

![KakaoTalk_Photo_2024-06-01-14-02-13 012](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/24a5d36a-b0c2-4b98-9510-ba86b54ec8a1)

- Process on the server
- To maximize CPU efficiency, each application loop is single-thread
- Thread is pinned to a fixed CPU core

**[Benefits of pinning the application loop to the CPU]**

1. No context switch
2. No locks and no lock contention

**[Loss]**

1. Making coding more complicated
- potentiall block subsequent tasks : Occupying the application loop thread for too-long

### mmap

- POSIX-compliant UNIX system call
- A file into the memory of a process
- Provide high-performance sharing of memory between prcesses
- With /dev/shm backing file, the performance advantage is compounded

+) /dev/shm : memory-backed file system

- the access to the shared memory does not result in any disk access at all

- Modern exchanges take advantage of this to eliminate as much disk acess from the critical path as possible
- Implement mmap as a message bus over which the components on the critical path communicate
- The communication pathway = Sending a message on this mmap message bus = sub-microsecond

### Event sourcing

![KakaoTalk_Photo_2024-06-01-14-02-13 013](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/285d532f-fd3e-41e5-a353-3ba8efb640ee)

**[Problems]**
- Traditional application : States are persisted in a database
- Database only keep the current states

**[Solutions]**
- Immutable log of all state-hanging events
- These events are the golden source of truth

![KakaoTalk_Photo_2024-06-01-14-02-13 014](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5e9c5313-2c1e-44dd-b86d-5534a815954b)

- FIX over Simple Binary Encoding (SBE) for fast and compact encoding and sends (Pre-defined format)

**[Difference between draft system design]**

1. Order Manager
- Become a reusable library that is embedded in different components
- Centralized order manager would heart latency
- Each component maintains the order states by itself, with event sourcing the states are guaranteed to be identical and replayable

2. The sequencer
- One single event store for all messages
- Sequencer field is injected by sequencer
- Do one simple thing and is super fast

+) Have multiple sequencers
- They will fight for the right to write to the event store
- A lot of time would be wasted on lock contention

![KakaoTalk_Photo_2024-06-01-14-02-13 015](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/e648409a-5ef1-464e-a4ac-64f64da357e4)

- The sequencer pulls events from the ring buffer that is local to each component
- It stamps a sequence ID on the event and sends it to the event store
- We can have backup sequencers for high aviliability

# High Availability
- Aim for 4 nines (99.9%) = 8.64 seconds of downtime per day

### Consider

- Identify single-point-of-failure in the exchange archietcture and set up redundant instances
- Detection of failure and decision to failoover to the backup instance should be fast

## Scaling

- Stateless service (ex) Client gateway) : horizontally scale by adding more servers
- Stateful components (ex) Order manager, Matching machine) : can copy state data across replica

![KakaoTalk_Photo_2024-06-01-14-02-13 016](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a2387bd8-5884-4e52-b87b-07acd6a56b0a)

- Hot matching engine : works as the primary instance
- Warm matching engine : receive and process the exact same events
- Event sourcing is a great fit for the exchange architecture

### Detect Potential Problems
- Sending heartbeats from the matching engine
- The problem with this hot-warm design is that it only works within the boundary of a single server
- Extend this concept across multiple machines or even across data centers

### Replicating entire event store across machines
- It takes time
- Reliable UDP to efficiently braodcast the event messages to all warm servers

# Fault Tolerance

- What if the warm instances go down as well?

## 1. If the primary instance goes down, how and when do we decide to failover to the backup instance?

### Meaning of down
1. The system might send out false alarms, which cause unnecessary failovers
2. Bugs in the code might cause the primary instance to go down. This same bug could bring down the backup instance fater the failover.

### Suggestion
1. Release a new system, we might need to perform failover manually

ex) Ensure system confidence using Chaos engineering

## 2. How do we choose the leader among backup instances?

### Battle-tested leader-election algorithms

![KakaoTalk_Photo_2024-06-01-14-02-13 017](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d7d4d96e-5b40-4240-a76a-fef6eff2ccfe)

1. Leader sends data to all the other instances (followers)
2. Minimum number of votes required to perform an operation is (N/2+1) (e.g., N = the number of members in the cluster)

ex) Raft

![KakaoTalk_Photo_2024-06-01-14-02-13 018](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/57984547-9d2e-4c16-93a7-264f2cdc1917)

**[Select next leader]**

1. The leader sends heartbeats message
2. Follower receives heartbeats

2-1. If a follower has not received heartbeat messages for a period of time, it triggers an election timeout that initiates a new election

2-2. The first follower that reaches election timeout becomes a candidate

3. Ask the rest of the followers to vote
4. Aggregating votes and results

4-1. If the first follower receives a majority of votes, it becomes a new leader

4-2. If the first follower has a lower term value than the new node, it cannot be the leader

**[Multiple candidates = split vote]**

1. The leader sends heartbeats message
2. Multiple follower receives heartbeats and become candidates at the same time
3. New election is initiated

## 3. What is the recovery time nedded (RTO - Recovery Time Objective)?
- Recovery Time Objective (RTO) : the amount of time an application can be down without causing significant damage to the business

## 4. What functionalities need to be recovered (RPO - Recovery Point Objectivies)? Can our system operate under degraded conditions?
- Backing up data frequently
- It the new leader crashes, the new leader should be able to function immediately

ex) Stock exchange
- RPO is near zero = Data loss is not acceptable

# Matching Algorithm

- FIFO Matching Algorithm

ex) a FIFO with LMM (Lead Market Marker) algorithm
- Allocate a certain quantity to the LLM based on a predefined ratio ahead of the FIFO queue
- The LLM firm negotites with the exchange for the privilege

ex) CME Website, Dark pool

# Determinism

## Functional Determinism
- The design choices we make, such as sequencer, event sourcing, guarantee that if the events are replayed in the same order, the results will be the same

![KakaoTalk_Photo_2024-06-01-14-02-14 019](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b2883610-6a32-4481-a8b3-6e30701f55c3)

- The actual time when the event happen does not matter most of the time
- What matter is the order of the events
- Unevent dots in the time dimension are converted to continuous dots
- The time spent on replay/recovery can be greatly reduced

## Latency Determinism

![KakaoTalk_Photo_2024-06-01-14-02-14 020](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/01f82726-d093-4ff1-9d36-f58faf2e7d26)

- Having almost the same latency through the system for each trade
- We can leaverage Hdrhistogram to calculate latency
- If the 99th percentile lateny is low, the exchange offers stable performance across almost all the trades

### Investigate large latency fluctation

ex) HotSpot JVM Stop-the-World garbage collection
- During safe points, all threads are suspended until the task is finished
- Including Garbage collection, Code deoptimization, Various debug operations

# Market data publisher optimizations

- It's expensive to get the more detailed L2/L3 order book data
- Many hedge funds record the data themselves via the exchange real-time API to build their own candlestick charts and other charts for technical analysis
- MDP (Market Data Publisher) receives matched results from the matching engine and rebuilds the order book and candlestick charts based on that

## The Order book Rebuild
- MDP is a service with many levels

ex) A retail client 
- It can only view 5 levels of L2 data by default and needs to pay extra to get 10 levels
- MDP's memory cannot expand forever, so we need to have an upper limit on the candlestick

![KakaoTalk_Photo_2024-06-01-14-02-14 019](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/40fab665-e459-44ad-b58c-d93f782b5710)

- Ring buffer = Circular buffer : fixed-size queue with the head connected to the tail
- A producer continously produces data
- One or more consumers pull data off it
- A ring buffer is pre-allocated
- There is no object creation or deallocation necessary
- The data structure is also lock-free
- Other techniques to make the data structure

ex) Padding ensures that the ring buffer's sequence nmber is never in a cache line with anything else

## Distribution fairness of market data
- It is important to guarantee that all the receivers of market data get that data at the same time

ex) First one alway receiving data first, Smart clients will fight to be the first on the list when the market open.

### Solutions
- Multicast using reliable UDP
- Good solution to braodcast updates to many participants as once
- Assign a random order when the subscriber connects to it

## Multicast

- A commonly-used protocol in exchange design
- By configuring several receivers in the same multicast group, they will in theory receive data at the same time
- UDP is an unreliable protocol and the datagram might not reach all the receivers
- There are solutions to handle retransmission

## Colocation
- The latency in placing an order to the matching engine is essentially proportional to the length of the cable
- Colocation does not break the notion of fairness
- It can be considered as a paid for VIP service

## Network security

### Combat DDoS
1. Isolate public services and data from private services, multiple read onl copies to isolate problems
2. Using a cache layer
3. Harden URLs

ex) 
- AS-IS : data?from=123&to=456
- TO-BE : data/recent/

4. An effective safelist/blocklist mechnism
5. Rate limiting
