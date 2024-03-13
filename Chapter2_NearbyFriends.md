# Chapter 2. Nearby Friends

- Nearby friends since it looks similar to proximity services
- You will find major differences : It's more dynamic than static location, due to change user locations frequently

# Step 1. Understand the Problem and Establish Design Scope

C: How geographically close is considered to be nearby?

I: 5 miles

<br/>

C: Can i assume the distance is calculated as the straight-line distance between two users?

I: Yes, that's a reasonable assumption

<br/>

C: How many users does the app have? Can I assumes 1 billion users and 10% of them use the nearby friends feature?

I: Yes, that's a reasonble assumption

<br/>

C: Do we need to store location history?

I: Location history can be valuable for different purposes such as machine learning.

<br/>

C: Could we assume if a friend is inactive for more than 10 minutes, that friend will disappear from the nearby friend list?

I: We can assume inactive friends will no longer be shown.

<br/>

C: Do we need to worry about privacy and data laws such as GDPR or CCPA?

I: For simplicity, don't worry about it for now.

## Functional Requirements

- User should be able to see nearby friends on their mobile apps. 
- Nearby friend lists should be updated every few seconds.

## Non-functional requirements

- Low latency : Receiving location updates from friends without too much delay
- Reliability : Occasional data point loss is acceptable
- Eventual consistency : No need strong consistency, A few seconds delay in receiving location data in different replicas is acceptable

# Back-of-the-envelope estimation

- Nearby friend are defiend as friends whose locations are within `a 5-mile radius`
- The location refersh interval is `30 seconds`
- On average, `100 million users` use the nearby firends feature everyday
- Assume the number of concurrent users is `10% of DAU (10 million)`
- On average, a user has `400 friends`
- The app displays `20 nearby friends` per page and may load more nearby friends upon request

## Calculate QPS
- 100 million DAU
- Concurrent users: 10% x 100 million = 10 million
- Users report their locations every 30 seconds
- Location update QPS = 10 million / 30 ~= 334,000

# Step 2. Propose high-Level Design and Get Buy-in

## High-level design
- It could in theory be done `purely peer-to-peer`, that is, a user could maintain `a persistent connetion` to every other active friend in the vicinity.

## Shared backend

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2210e8da-cb6d-4e01-a862-dc4cd7da6b6c" alt="IMG_2715" width="500"/>

- Receive location updates from all active users
- For each location update, find all the active friends who should receive it and forward it to those users' devices

### Problems
- To do this at scale is not easy

<br/>

- Active users : 10 million 
- Each user updating the location information : every 30 seconds

> 333K updates per second

<br/>

- Average number of user's friends : 400 friends
- Online and nearby friends : 10%

> Every second : 333K x 400 x 10% = 13 million

## Proposed design

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9d368d5f-334c-48d4-8e40-a0f3c011e17f" alt="IMG_2716" width="500"/>

## Load Balancer

- In front of the RESTful API servers and the stateful, bi-directional WebSocket servers
- It distributes traffic across those servers to spread out load evenly

## Websocket servers

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b90177f0-4bed-4167-9a25-b0ce334666a1" alt="IMG_2716" width="500"/>

- This is a cluster of stateful servers

1. Each client maintains one persistent WebSocket connection to one of these servers
2. Handling client initialization for the "nearby friends" features

### +) What is [WebSocket](https://en.wikipedia.org/wiki/WebSocket)

+) [RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)

- WebSocket protocol enables `full-duplex interaction`
- A computer communication protocol providing simultaneous two-way communication channels over a single TCP (Transmission Control Protocol)
- Although HTTP and WebSocket are different, WebSocket is designed to `work over HTTP ports 443 and 80` as well to support HTTP proxies and intermediaries
- It is beneficial for environments that block non-web internet connections using a firewall
- To achieve compatibility, the WebSocket handshake uses the HTTP Upgrade Header
- Once handshake is successfully completed, the data exchanged between the client and server is no longer encapsulated within HTTP packet. `Switching to TCP communication`

+) [What is a Network Load Balancer?](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)

### +) Polling
- half-duplex 

## Redis location cache

- Time To Live (TTL) : the most recent data for each active user
- When the TTL expires, the user is no longer active and the location data is expunged from the cache

## Redis pub/sub server

- A very lightweight message bus
- Channels in redis pub/sub are very cheap to create

+) A modern Redis server with GBs of memory could hold millions of channels

# Periodic location update

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/79221b32-c8af-4079-82b6-c2442e777ce2" alt="IMG_2718" width="500"/>

1. The mobile client sends a location update to the load balancer
2. The load balancer forwards the location update to the persistent conneciton on the Websocket server for that client
3. The WebSocket server saves the location data to the location history database
4. The WebSocket server updates the new location in the location cache. The update refershes the TTL.
5. The WebSocket server publishes the new location to the user's channel in the Redis pub/sub server
6. When Redis pub/sub receives a location update on a channel, it broadcasts the update to all the subscribers. In this case the subscribers are all the online friends of the user sending the update.
7. In receiving the mssage, the WebSocket server, on which the connection handler lives, computes the distance between the user sending the new location and the subscriber.

# Step 3. Design Deep Dive

## WebSocket servers

- WebSocket servers are stateful, so care must be taken when removing existing nodes
- We can mark a node as "draining" at the load balancer so that no new WebSocket connections will be routed to the draining server
- Once all the existing connections are closed, the server is then removed
- Most cloud load balancers handle this job very well

### +) stateful? stateless?

**[stateless]**
- A communication protocol in which the receivers must not retain session state from previous requests

**[stateful]**
- the receiver may retain session state from previous requests

### +) TCP
- session management : sender and receiver maintain state information about the session, which includes sequence numbers, acknowledgements and window size among other things

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5ffca82e-8acf-4254-9fc1-6bc12dbf28ac" alt="IMG_2718" width="500"/>

### +) UDP
- There is no need for the server to store session information in memory

+) [Chapter 10. Networks](https://tldp.org/LDP/tlk/net/net.html)

### +) Theologically, how many connection can be establish?

**Session Information**
- Protocol
- Client IP Address, Clinet Port Number
- Server IP Address, Server Port Number

> Port range = 2^16 = 65,353

## Location Cache

**[Memory]**
- With 10 million active users at peak, and with each loation taking no more than 100 bytes
- A single modern Redis server with many GBs of memory should be able to easily hold the location information for all users

<br/>

**[Load]**
- With 10 million active users roughly updating every 30 seconds
- The Redis server will have to handle 334K updates per second
- This is likely a little too high

> This cache data is easy to shard, and location data for each user is independent, 
> We can evenly spread the load among several Redis servers

## Redis pub/sub server

- Routing layer to direct message from one user to all the online firends
- `Very lightweight` to create new `channels`
- A new channel is created when someone subscribes to it
- If a message is published to a channel that has no subscribers, the message is dropped
- When a channel is created Redis uses a small amount of memory to maintain `a hash table` and  `a linked list` to track the subscribers

## How many Redis pub/sub servers do we need?

### Memory Usage
- Active users : 100million
- Average the number of friends per a user : 100 active friends
- Pointer (Linked list or Hash table) : 20bytes
- Total memory size : 200GB
- Modern Redis memory : 100GB

### CPU usage
- the pubsub server pushes about 13million updates per second to subscribers
- It is safe to assume that a single Redis server will not be able to handle that load. 
- So let's assume one redis server could handle 100,000 subscribers pushes per second.

> Total : 13million / 100,000 = 130 Redis

## Distributed Redis pub/sub server cluster

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/60d70f54-cbaa-4173-b8b8-659fb8b902f4" alt="IMG_2719" width="500"/>

- The channels are independent of each other
- Multiple pub/sub servers by sharding, based on the publisher's user ID

### Service discovery component
1. The ability to keep a list of servers in the service discovery component and a simple UI or API to update it
2. The ability for clients to subscribe to any updates to the "Value"

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f8a71912-47d5-44ca-8bb4-87eeec6c511c" alt="IMG_2720" width="500"/>

1. The WebSocket server consults the hash ring to determine the Redis pub/sub server to write to
2. WebSocket server publishes the location update to the user's channel on that Redis pub/sub server

### +) Server Side Discovery

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b0778d68-b2fa-414a-ad8a-7d6281817af3" alt="IMG_2720" width="500"/>

## Scaling considerations for Redis pub/sub servers

- When we resize a cluster, many channels will be moved to different servers on the hash ring
- When the service discovery component notifies all the Websocket servers of the hash ring update, 
- There will be a ton of `resubscription requeusts`
- During these mass resubscription events, some location updates might be missed by the clients
- Because of the potential interruptions, resizing should be done when usage is at its `the lowest in the day

## Nearby random person

<img src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/87d79161-d33a-41c0-a33d-5bbb6e80307c" alt="IMG_2722" width="500"/>

1. User 2 updates their location, the WebSocket connection handler computes the user's geohash ID and sends the location to the channel for that geohash
2. Anyone nearby who subscribes to the channel will receive a location update message
