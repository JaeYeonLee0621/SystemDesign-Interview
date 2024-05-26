# Chapter 10. Real-time Gaming Leaderboard

## What is Leaderboards
- Showing who is leading a particular tournament or competition

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/6e8f84fb-fd09-4cdd-8cd3-8a5dcae11076)

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

![1](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f212e04b-b45d-49f8-8d9e-12e0153277e1)


1. Client -> Game Server : A player wins a game,
2. Game Server : Checking the validation of the win
3. Game Server -> Leaderboard Service : Updating the score
4. Leaderboard service -> Leaderboard Store : Updating the user's score
5. Client -> Leaderboard Service : Fetching leaderboard data
Sa. top 10 leaderboard
b. the rank of the player on the leaderboard

# Can client calculate score and request to leaderboard service directly?

![2](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/151d9861-8fde-4230-a629-93dd29afdba3)


## Alternative Design
- The score is set by the client 👉 `Man in the middle attack`
- Players can put in a proxy and change scores at will

> We need the score to be set on the server-side

<br/><hr/><br/>

## +) [MiTM (Man in the middle)](https://www.invicti.com/blog/web-security/man-in-the-middle-attack-how-avoid/)

![3](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/09737885-e1ac-4911-99de-4c755934908b)

### Where is the MiTM?
1. Public networks (ex) Public Wifi connections, WiFi hotspots, free WiFi at cafes)
2. Own computer (ex) MiTM browser)
3. Home router 
- Home router supplied by ISPs use default admin credential
- Their firmware is often outdated

### How do MiTM attacks work?

### 1. Unencrypted Connection

### 1-1. [ARP (Address Resolution Protocol) Spoofing](https://www.invicti.com/learn/mitm-arp-spoofing/)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/767f3395-8119-4900-a163-9080877ec795)

- the attacker sends forged ARP (Address Resolution Protocol) messages to associate their own MAC address with the IP address of the legitimate LAN gateway (usually the router) on the local network.
- ARP process with Routers : Device A wants to communicate with a device outside its local network

### +) ARP Procedures

1. When a Device Joins the LAN (let's call it Device A)
- Being assigned an IP address : 1) Manually 2) automatically via DHCP (Dynamic Host Configuration Protocol)
- When Device A wants to send a packet to another IP address, it needs to know the MAC address corresponding to that IP
2. ARP Request : Device A will broadcast an ARP request packet to all devices in the LAN

### 1-2. [IP spoofing](https://www.invicti.com/learn/mitm-ip-spoofing-ip-address-spoofing/)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ed6dc4fd-ce7e-4f1f-b9be-345adecbdc17)

- The attacker can intercept an ongoing TCP/IP connection with the gateway by injecting TCP packets

### 1-3. [DNS spoofing](https://www.invicti.com/learn/mitm-dns-spoofing-dns-cache-poisoning/)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1d608820-ccfb-490e-9ea9-6c504fa03317)

- A malicious actor may use address resolution protocol (ARP) to access router traffic and alter the domain name resolution records.
- The attacker can modify an authoritative DNS server’s records, redirecting traffic to the fraudulent website.

1. A user’s device (client) sends a `DNS query to a DNS server to resolve a domain name into an IP address.`
2. An attacker `intercepts this DNS query` and responds to it before the legitimate DNS server can.
3. The attacker crafts a `fake DNS response`, which includes a falsified IP address pointing to a malicious server controlled by the attacker instead of the legitimate IP address.
4. If the forged response reaches the DNS server before the legitimate one, `the DNS server caches the malicious IP address.`

### 1-4. [Web browser bar spoofing](https://www.invicti.com/blog/web-security/web-browser-address-bar-spoofing/)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ea0c5734-d372-40fc-8245-136de6e78569)

- The attacker registers a domain name that looks very much like the domain that you want to connect to.
- Then they deliver the false URL to your using other techniques such as phishing

### 2. Encrypted Connection

### 2-1. [SSL hijacking](https://www.invicti.com/learn/mitm-ssl-hijacking/)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/34b0f569-09d4-4eea-84b5-67fe30e88e68)

- This requires control of your computer
- The attacker adds a trusted CA to your computer
- When you attempt to connec to a secure site, the MiTM agent serves you a false website signed using this CA
- CA is added to your computer, there is no alarm
- hijacks a user’s legitimate session and pretends to be that user

### 2-2. [SSL stripping](https://www.invicti.com/learn/mitm-ssl-stripping/) (=HTTPS hijacking)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/72cf2e71-9b2e-4dc2-9277-9710dd5a8385)

- The MiTM agent makes your computer believe that an HTTPS connection is not available and that HTTP must used

1. Initial Client Request
- The client atempts to connect to a secure website using HTTPS

2. Interception of Request
- The attacker intercepts the client's rquest before it reaches the intended server

3. Presenting a Fake Cretificate
- The attacker `intercepts the SSL handshake process`
- The server is supposed to send its SSL certificate to the client
- The attacker generates a fake certificate

4. Client accepts this certificate
- Proceeds to establish an encrypted connection with the attacker

4-1. User-Induce Acceptance
- Modern browser will typically display a security warning
- However users can be tricked into accpeting these certificates

4-2. Installing a Malicious Root Certificate
- Installing a malicious root certificate on the victim's device (ex) Physical access, malware, software installation)

5. Decryption of Data by Attacker
- The attacker encrypts and decrypts the data using fake cretificate they presented

### 2-3. Attacks on old SSL ciphers
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
- It is an in-memory data store supporting key-value pairs and ⭐`sorted sets`⭐, allowing for fast reads and writes
- It is not going back to Skip List to Zip List to prevent `flipping effects`

<br/><hr/><br/>

## +) What are Sorted Sets?

![8](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a325f8b4-e421-4089-96e9-bf5f4be5c490)

- Members are UNIQUE and ORDERED

## 1. [Zip List](http://redisgate.kr/redis/configuration/ds_ziplist_hashes.php)

- For small sorted sets
- A compact, memory-efficient data structure
- Time complexity = normal array regarding insertion and deletion operation (=O(n))

![7](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c2b90a00-190e-457c-b002-46ef099da48a)

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
zlbytes | zltail | zllen | Prevlen(0) | Encoding(string) | "banana" | Prevlen(7) | Encoding(string) | "apple" | zlend
```
- Prevlen(6byte) = length('apple\0')

3. RPUSH cherry
```
zlbytes | zltail | zllen | Prevlen(0) | Encoding(string) | "banana" | Prevlen(7) | Encoding(string) | "cherry" | Prevlen(6) | Encoding(string) | "apple" | zlend
```

- When the size of a dataset in Redis `exceeds the thresholds` for ziplist encoding, Redis will `convert the entire dataset` to a more suitable data structure

## 2. [Skip List (ZADD)](http://redisgate.kr/redis/configuration/internal_skiplist.php)

![9](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a9ec632d-7a7c-4c29-8c15-487c255910e1)

- The main data structure for `Zset`
- a list structure that allows for fast search (O(logN))

### 1. Leveled Linked List

![4](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/455beb6e-fc7d-4c58-8b1a-cb1d19bad0e1)

- Each node have defined levels to decrease the number of comparing
- `Problem` : Whenever nodes are inserted and deleted, need to convert all nodes' levels

### 2. Applying probability of flipping coin

![5](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2f341695-e4ed-45f5-867a-8ccb10d0f15d)

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

![6](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9cf6b05a-d4a9-4cd1-82d7-62a921585bbb)

- 1/e is the fatest probability but have minor difference between 1/4

```
#define ZSKIPLIST_P   0.25       /* Skiplist P = 1/4 */
```

### Example) Searching 45

![10](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/27038a0d-e363-4c4a-9754-cd139ba276de)

<br/><hr/><br/>

# Step 3. Design Deep Dive

# 1. Manage our own services

![11](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/39bc49be-5843-442e-869c-d480994f63a0)

# 2. Build on the cloud
- Assuming our existing infrastructure is built on aWS

## AWS API gateway

- Managed Service for HTTP & REST and WebSocket Endpoints

![12](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9f5016a0-fb35-411f-a014-f8161005a967)

## AWS Lambda
- Allowing you to run code without having to provision or manage the servers yourself
- It runs only when needed and will scale automatically based on traffic

![13](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/28becd89-be4a-4d43-85a0-448a6e4fd9af)

# Scaling Redis

- Assuming we have 500 million DAU, which is 100 times our original scale
- size of the leaderboard = 650MB x 100 = 65GB
- QPS = 2,500 x 100 = 250,000

## Data Sharding

### 1. Fixed Partition

![14](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b71cdf81-fa64-485f-849d-f9d8df600f23)

- We need to adjust the score range in each shard to make sure of a `relatively even distribution`
- Update : When a user increases their score and moves between shards

### 2. Hashed Partition

![15](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d239a2ac-aa85-48ab-b56d-d27a0a213c39)

ex) [CRC16(key) % 16384](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)

### +) CRC 16 (Cyclic Redundancy Check)
- Error-detecting code (ex) digital network, storage device)
- generates a 16-bit (2^16 = 65536) checksum based on the input data

### +) Redis Slot (=Shard) : Why 16384 (2^14)

- Normal heartbeat packets carry the full configuration of a node, that can be replaced in an idempotent way with the old in order to update an old config
- This means they contain `the slots (shards) configuration for a node (redis server)`

- 16384 (2^14) messages only occupy 2k

+) 16k = `2` * 8 (8 bit/byte) * 1024(1k) = 2K bitmap size

- while 65535 (CRC 16 = 2^16 - 1) would require 8k

+) 65k = `8` * 8 (8 bit/byte) * 1024(1k) = 8K bitmap size

- So `2K bitmap size (=Save 2000 node information)` was in the right range to ensure enough slots per master with a max of `1000 mater nodes`

### +) Gossip Protocol
- Redis Cluster nodes communicate with each other using a gossip protocol
- Nodes exchange information about their state, status, and cluster topology regularly

### Fetch : Scatter-Gather

![16](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/8a645469-01f1-419d-b735-d86ae37cbe54)

- We need to gather the top 10 players from each shard, have the application sort the data

## Sizing a Redis node
- Write-heavy applications require much more avilable memory, since we need to be able to accomodate all of the writes to create the snpshot in case of a failure

## Alternative solution : NoSQL
- Optimized for writes
- Efficiently sort items within the same partition by score

1. Year-Month

![18](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/e171a260-af0d-41be-8d74-f7d90a7d7c95)

- Partition key = `game_name#{year-month}`
- Sort key = score
- Problem : Hot Partiton

2. User ID

![17](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7583b5c7-6786-4df3-977e-5ed5046485e6)

- Partition key = `user_id % number_of_partition` 
- Write sharding

3. Ideal Prtition Key

![19](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/19a97e36-993c-427e-b5a5-b772232985a7)

- Partition Key = `game_name#{year-month}#p{partition_number}`
