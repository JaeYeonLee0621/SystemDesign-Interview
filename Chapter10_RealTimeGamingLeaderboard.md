# Chapter 10. Real-time Gaming Leaderboard

## What is Leaderboards
- Showing who is leading a particular tournament or competition

![1](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f212e04b-b45d-49f8-8d9e-12e0153277e1)

# Step 1. Understand the Problem and Establish Design Scope

## Non-functional requirements
- Real-time update on scores
- Score update is reflected on the leaderboard in real-time
- General scalability, availability and reliability requirements

## Back-of-the-envelope estimation

### The average users per second
- `Distribution of players` : 5 million DAU / 10^5 seconds (24 hours) ~= 50
- `Uneven distribution` (due to different time zone) : peak load = 5 times the average ~= 250 users per second

### QPS (Query Per Second)
- `QPS (Query Per Second)` : 50 (the average number of users per second) x 10 (a user plays 10 games per day on average) ~= 500
- `Peak QPS` ~= 5 x 500

<br/><hr/><br/>

# Step 2. Propose High-Level Design and Get Buy-In

# High-level architecture

![2](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/151d9861-8fde-4230-a629-93dd29afdba3)

1. Client -> Game Server : A player wins a game,
2. Game Server : Checking the validation of the win
3. Game Server -> Leaderboard Service : Updating the score
4. Leaderboard service -> Leaderboard Store : Updating the user's score
5. Client -> Leaderboard Service : Fetching leaderboard data
a. top 10 leaderboard
b. the rank of the player on the leaderboard

# Should the client talk to the leaderboard service directly?

![3](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/09737885-e1ac-4911-99de-4c755934908b)

## Alternative Design
- The score is set by the client :finger: `Man in the middle attack`
- Players can put in a proxy and change scores at will

> We need the score to be set on the server-side

<br/><hr/><br/>

## +) [MiTM (Man in the middle)](https://www.invicti.com/blog/web-security/man-in-the-middle-attack-how-avoid/)

### Where is the MiTM?
1. Public networks (ex) Public Wifi connections, WiFi hotspots, free WiFi at cafes)
2. Own computer (ex) MiTM browser)
3. Home router 
- Home router supplied by ISPs use default admin credential
- Their firmware is often outdated

### How do MiTM attacks work?

### 1. Unencrypted Connection

#### [ARP Spoofing](https://www.invicti.com/learn/mitm-arp-spoofing/)
- An attacker sends false ARP packets pretending to be the gateway
- Your computer connects to the attacker's computer thinking that it is the gateway to the internet

#### [IP spoofing](https://www.invicti.com/learn/mitm-ip-spoofing-ip-address-spoofing/)
- The attacker can intercept an ongoing TCP/IP connection with the gateway by injecting TCP packets

#### [DNS spoofing](https://www.invicti.com/learn/mitm-dns-spoofing-dns-cache-poisoning/)
- As long as you connect to a vulnerable DNS cache (DNS resolver), the attacker injects false information into the DNS resolver and your computer connects to the server controlled by the attacker

#### [Web browser bar spoofing](https://www.invicti.com/blog/web-security/web-browser-address-bar-spoofing/)
- The attacker registers a domain name that looks very much like the domain that you want to connect to.
- Then they deliver the false URL to your using other techniques such as phishing

### 2. Encrypted Connection

#### [SSL hijacking](https://www.invicti.com/learn/mitm-ssl-hijacking/)
- This requires control of your computer
- The attacker adds a trusted CA to your computer
- When you attempt to connecto to a secure site, the MiTM agent serves you a false website signed using this CA
- CA is added to your computer, there is no alarm

#### [SSL stripping](https://www.invicti.com/learn/mitm-ssl-stripping/)
- The MiTM agent makes your computer believe that an HTTPS connection is not available and that HTTP must used

#### Attacks on old SSL ciphers
- Connect to is using a vulnerable cipher

<br/><hr/><br/>

# Do we need a message queue between the game service an the leaderboard service?

- `Yes` : If the data is used in other places or support multiple functionalities

# Data models

## 1. Relational database solution
- It works when the data set is small
- But the query becomes very slow when there are millions of rows
- SQL databases are not performant when we have to process large amounts of continuously changing information

## 2. Redis solution
- It is an in-memory data store supporting key-value pairs and sorted sets, allowing for fast reads and writes
- It is not going back to Skip List to Zip List to prevent `flipping effects

<br/><hr/><br/>

## +) What are Sorted Sets?

[Image : 8]

- Members are UNIQUE and ORDERED

## 1. [Zip List](http://redisgate.kr/redis/configuration/ds_ziplist_hashes.php)

- For small sorted sets
- A compact, memory-efficient data structure
- Time complexity = normal array regarding insertion and deletion operation (=O(n))

[Image : 7]

- `A` : Hash Table for organizing Redis DB Key
- `value of dictEntry` : pointing redisObject
- `redisObject` : pointing Zip list

### Structure

```
zlbytes | zltail | zllen | Entries | zlend
```
- `zlbytes` : Total memory size of the ziplist
- `zltail` : Holding the offset to the last entry in the ziplist.
- `zllen` : Number of entries in the ziplist
- `Entries` : Contain the actual data with backward traversal capability and efficient encoding
- `zlend (0xFF)`: Marks the end of the ziplist for simple traversal

### Example

- `Prevlen` : The length of previous entry
- `Encoding` : How the entry's content is encoded and its length

1. LPUSH apple
```
zlbytes | zltail | zllen | Prevlen(0) | Encoding(string) | "apple" | zlend
```

2. LPUSH banana
```
zlbytes | zltail | zllen | Prevlen(6) | Encoding(string) | "banana" | Prevlen(0) | Encoding(string) | "apple" | zlend
```
- Prevlen(6byte) = length('apple\0')

3. RPUSH cherry
```
zlbytes | zltail | zllen | Prevlen(6) | Encoding(string) | "banana" | Prevlen(7) | Encoding(string) | "cherry" | Prevlen(0) | Encoding(string) | "apple" | zlend
```

- When the size of a dataset in Redis `exceeds the thresholds` for ziplist encoding, Redis will `convert the entire dataset` to a more suitable data structure

## 2. [Skip List (ZADD)](http://redisgate.kr/redis/configuration/internal_skiplist.php)

[Image: 9]

- The main data structure for `Zset`
- a list structure that allows for fast search (O(logN))

### 1. Leveled Linked List

![4](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/455beb6e-fc7d-4c58-8b1a-cb1d19bad0e1)


- Each node have defined levels to decrease the number of comparing
- `Problem` : Whenever nodes are inserted and deleted, need to convert all nodes' levels

### 2. Applying probability of flipping coin

[Image : 5]
- Span : index of next node

#### Node 1
1. Flipping coin
- Level 1

#### Node 2
1. Flipping coin
- Different side : Level 1
- Same side : Go to Next Level
2. Flipping coin
- Different side : Level 2
- Same side : Go to Next Level

<br/>

- `Problem` : need to decrease the memory to save Pointer and Order
- `Solution` : Lowering levels

### 3. Applying probability of throwing the Dice 
- Decrease 1/2 to 1/6

[Image : 6]

- 1/e is the fatest probability but have minor difference between 1/4

```
#define ZSKIPLIST_P   0.25       /* Skiplist P = 1/4 */
```

### Example) Searching 45

[Image : 10]

<br/><hr/><br/>

# Step 3. Design Deep Dive

# 1. Manage our own services

[Image : 11]

# 2. Build on the cloud
- Assuming our existing infrastructure is built on aWS

## AWS API gateway

- Managed Service for HTTP & REST and WebSocket Endpoints

[Image : 12]

## AWS Lambda
- Allowing you to run code without having to provision or manage the servers yourself
- It runs only when needed and will scale automatically based on traffic

[Image : 13]

# Scaling Redis

- Assuming we have 500 million DAU, which is 100 times our original scale
- size of the leaderboard = 650MB x 100 = 65GB
- QPS = 2,500 x 100 = 250,000

## Data Sharding

### 1. Fixed Partition

[Image : 14]

- We need to adjust the score range in each shard to make sure of a `relatively even distribution`
- Update : When a user increases their score and moves between shards

### 2. Hashed Partition

[Image : 15]

ex) CRC16(key) % 16384

[Image : 16]

- Fetch : Scatter-Gather
- We need to gather the top 10 players from each shard, have the application sort the data

## Sizing a Redis node
- Write-heavy applications require much more avilable memory, since we need to be able to accomodate all of the writes to create the snpshot in case of a failure

## Alternative solution : NoSQL
- Optimized for writes
- Efficiently sort items within the same partition by score

1. Year-Month

[Image : 17]

- Partition key = `game_name#{year-month}`
- Sort key = score
- Problem : Hot Partiton

2. User ID

[Image : 18]

- Partition key = `user_id % number_of_partition` 
- Write sharding

3. Ideal Prtition Key

[Image : 19]

- Partition Key = `game_name#{year-month}#p{partition_number}`
