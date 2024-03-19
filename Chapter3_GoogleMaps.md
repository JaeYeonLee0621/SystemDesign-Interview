# Google Maps

# Step 1. Understand the Problem and Establish Design Scope

- 1 bilion DAU
- Feature : Direction, Navigation, ETA (Estimated Time of Arrival), map rendering
- How large : Obtaining the road data from difference source, about TBs (terabytes) of raw data
- Consider Traffic conditions for accurate time estimation

## Non-functional requirements and constraints
- Accuracy: Users should not be given the wrong directions
- Smooth navigation: users should experience very smooth map rendering
- Data and battery usage: The client should use as little data and energy as possible
- General availability and scalability requirements

## Map 101

### Position System

![KakaoTalk_Photo_2024-03-19-11-39-00 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/6421711a-5ca9-4c5c-a783-110c3f99af3a)


- Lat (Latitude): denotes how far north or south we are
- Long (Longitude): denotes how far east or west we are

### Going from 3D to 2D

![KakaoTalk_Photo_2024-03-19-11-39-00 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1db94da8-1aa2-469b-abaa-0560657992c9)


- There are different ways to do map projection

### GeoCoding
- The process of converting addresses to geographic coordinates

ex) 1600 Amphitheatre Parkway, Mountain View CA
> latitude, longitude = 37.423021, -122.083739

- `Reverse geocoding` : The conversion from the latitude/longitude pair to the actual human-readable address

+) [Interpolation](https://www.mapzen.com/blog/interpolation/)

ex) Pelias geocoder > Mapzen Search

- A method of creating new data points from a set of known data points

![shortland_st2](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/df013ce3-22ed-415a-b0cf-249b8067b60a)

- Red and blue dots are known address points from OpenStreetMap and OpenAddresses
- The holes in the data result in our search engine returning a less granular result for queries which request the location of one of these missing addresses
- By filling in these gaps in the data, we can intelligently estimate where the missing house numbers would lie on each road

## GeoHashing

- An encoding system that encodes a geographic area into a short string of letters and digits
- It depicts the earth as a flattened surface and recursively divides the grids into sub-grids, which can be square or rectangular

![KakaoTalk_Photo_2024-03-19-11-39-00 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/73ab0d7c-992e-4e68-a29f-048d74c11fbd)


## Map rendering

### Tiling

- Instead of rendering the entire map as one large custom image, the world is broken up into smaller tiles
- The client only downloads the relevant tiles for the area the user is in and stitches them together like a mosaic for display

## Road data processing for nativation algorithms

![KakaoTalk_Photo_2024-03-19-11-39-00 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f3b9656b-4223-44b6-ae19-b9e33b944c40)

- All these (Dijkstra's, A*) algorithms operate on a graph data stucture
- The pathfinding performance for most of these algorithms is exremely sensitive to `the size of the graph`

### Grids routing tiles

![KakaoTalk_Photo_2024-03-19-11-39-01 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/542a8fbe-c130-4a6d-883e-951083e76a9b)

- By employing a similar subdivision technique as geohashig, we divide the world into small grids
- For each grid, we convert the roads within the grid into a small graph data structure that consists of the node andedges inside the geographical area covered by the grid
- Each routing tile holds references to all the other tiles it connects to

## Hierarchical routing tiles

![map tiling](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/72a79399-cc0b-4d52-a2f7-b643b19cef30)

- There are typically three sets of routing tiles with different levels of detail
- At the most detailed level, the routing tiles are small and contain only local roads
- At the lowest level of detail, the tiles cover large areas and contain only major highways connecting cities and states together

## Back-of-the-envelope estimation

- 1 foot = 0.3048 m

## Storage demand

- Map of the world
- Metadata : The metadata for each map tile could be negligible in size
- Road info : We assume there are TBs of road data from external sources. We transform this dataset into routing tiles, which are also likely to be TBs in size

## Map of the world

- Let's assume that each tile is a 256x256 pixel compressed PNG image, with the image size of about 100KB
- The entire set at the highest zoom level would need about 4.4 trillion x 100KB = 440 PB

![zoom level table](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/de317a4c-dd6d-4aa0-8124-d8351bd360e2)

<br/>

- 90% of the world's surface is natural and mostly uninhabited areas like oceans, deserts, lakes and mountaines
- We could conservatively reduce the storage estimate by 80~90%
- That would reduce the storage size to a range of 44 to 88PB
- Let's pick a simple round number of 50PB

```
50 + 50/4 + 50/(4^2) + ... + 50/(4^n) ~= 67PB
```

- At each lower zoom level, the number of tiles for both north-sourth and east-west directions drops by half
- This results in a total reduction of the number of tiles by 4x, which drops the sotrage size for the zoom level also by 4x 
- We need roughly about 100PB to store all the map tiles at varying levels of detail

## Server throughput

### Navigation requests
- Client to initiate a navigation session
- 1 bilion DAU X 35 minutes per weeks ~= 7200 QPS

### Location update requests
- Client as the user moves around during a navigation session

<br/>

1. send GPS coordinates every second 
- 5 billion minutes x 60 requests per day = 3 million QPS

2. send batched GPS and then sent to the server ever 15 seconds
- If users are stuck in traffic
- 3 million QPS / 15 second ~= 200,000

> Assume peak QPS is 5 times the average = 200,000 x 5 = 1 million

# Step 2. Propose High-Level Design and Get Buy-In

![KakaoTalk_Photo_2024-03-19-11-39-01 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7b1ee020-516b-40a2-961e-efac03d1b328)

## Location Service

![KakaoTalk_Photo_2024-03-19-11-39-01 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/fc4fa550-be27-46d8-91b8-600d78e96889)

- The location service is reponsible for recording a user's location update
- Even when location updates are batched, the write volume is still very high
- We need a database that is optimized for high write volume and is higly scalable sucha as Cassandra

### Communication Protocol : HTTP keep-alive option

- Support over HTTP/1.0+

<img width="593" alt="Keep Alive" src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f3462846-1fdd-4f5a-8295-0168c1879669">


**Persistent connection**

- Site locality : Have higher possibilities that client keeps sending continuous requests

<br/>

**Pros**

- Reduce Network Congestion : TCP, SSL/TCP connection request
- Reduce Network Cost : Sending multiple data through one connection (+) Reduce CPU and memory : [getsockopt](https://man7.org/linux/man-pages/man2/setsockopt.2.html))
- Reduce Latency : Sending data with only one 3-way handshake

**Cons**

- If servers are busy, keep alive might use significant resources, expoenntial connection increase and quickly reaches over MaxClient value
- Spend intense memory and it become the reason of performance degradation

<br/>

**Request Format**

<img width="286" alt="Screenshot 2024-03-19 at 10 57 47 AM" src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1c67042f-c8c0-4034-98cc-6fc129f80352">

- max (MaxKeepAliveRequests) : Maximum Requests
- Timeout (KeepAlivetimeout) : time connection can remain as idle status

<br/>

**Problem**

<img width="553" alt="Proxy" src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b0c6de9d-220e-43d4-90c9-f79378ffb622">

- If conenction establish with proxy which doesn't support keep alive option, the proxy assume it as extension header
- That Proxy might wait until client make connection close, but Client assume it as keep alive request, so keep connection alive
- So Proxy stay in hang status until connection is closed

<br/>

**[HTTP 1.1 Persistent Connection](https://datatracker.ietf.org/doc/html/rfc2068#section-8.1)**

- So after `HTTP/1.1`, Persistent connection is not sent Persistent connectino header to Proxy
- If Proxy support keep-alive connection, it uses `Proxy-Connection` header alternatively
- But the problem is still existed when dumb proxy is located middle of the multiple proxy connections

+) HTTP/1.1 Persistent Connection Criteria
- For maintaining connection alive, every connections don't have any errors

## Map rendering
- The map tiles must be fetched `on-demand` from the server based on the client's location and the zoom level of the client viewport

### Option 1
- The server builds the map tiles on the fly, based on the client location and zoom level of the client viewport

**Cons**
- A hugh load on the server cluster to generate every map tile dynamically
- It's hard to take advantage of caching

### Option 2
- Pre-generated set of map tiles at each zoom level
- Each tile is therefore represented by its geohash
- When a client needs a map tile, it first determines the map tile collection to use based on its zoon level

<br/>

- Use CDN, the map tiles are served from the nearest point of presence (POP) to the client

1. Data Usage
- user moves : 30km/h
- zoom level : each image covers a block of 200mx200m (256pixelsX256pixels)
- 1km x 1km : 25 images = 2.5 MB
- the speed = 30km/h > (30 x 2.5MB)/hour = 1.25MB/min

2. Traffic through CDN
- 5 billion minutes of navigation per day
- 5 billion x 1.25 MB = 6.25 billion MB per day
- 6.25 billion MB / 10^5 second in a day = 62,5000 MB
- Assumes there are 200 POPs
- Each POP = (62,500 / 200)MB per second

<br/>

### How Client get the tiles from URL?

1. Hardcoded Client-side algorithm
- Convert a latitude/longitude pair and zoom level to a tile URL
- It's time-consuming and risky to change the algorithm in client application

2. Server a s an intermediary constructing the tile URLs

![KakaoTalk_Photo_2024-03-19-11-39-01 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f6338a1a-7cfb-4d64-bd28-2367817b76a7)

- Load balancer forwards the request to the map tile service
- The map tile service takes the client's location and zoom level as inputs and returns 9 URLs of the tiles to the cliient
- The mobile client downloads tiles from the CDN
