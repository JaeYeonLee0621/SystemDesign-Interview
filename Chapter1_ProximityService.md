# A Proximity Service : Discover nearby places

![IMG_2696](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/0e7fc48d-30ef-4d3f-8220-618809ca0a0f)

<br/>

# Step 1. Understand the Problem and Establish Design Scope

Q) Can a user specify the search radius?

A) We only care about business within a specified radius

<br/>

Q) What is the maximal radius allowed?

A) 20km

<br/>

Q) Can a user change the search radius on the UI?

A) 0.5km, 1km, 2km, 5km, 20km

<br/>

Q) Business information need to reflect in real-time?

A) Business owners can add, delete or update a business. Assume newly added/updated businesses will be effective the next day.

<br/>

Q) A user might be moving while using the app/web-site. Do we need to refresh the page to keep the results up to date?

A) Let's assume a user's moving speed is slow and we don't need to constantly refresh the page

<br/>

# Back-of-the-envelope estimation

- Assume we have 100 million daily active users and 200 million businesses
- Seconds in a day = 24 x 60 x 60 = 86,400 (round it up to 10^5)
- Assume a user makes 5 search queries per day
- Search QPS = 100million x 5 / 10^5 = 5,000

<br/>

# Step 2. Propose High-Level Design and Get Buy-In

## Data model

1. LBS (Location-Based Service)
2. Business-Related Service

## LBS (Location-Based Service)
- `Read-heavy` service with no write requests
- `QPS is high`, especially during peak hours in dense areas
- Stateless, easy to scale horizontally

## Business Service
- Business Owner create, update, or delete businesses
- QPS is high during peak hours

<br/>

## Read/Write Ratio

**Read volume is high**
- Search for nearby businesses
- View the detailed information of a business

**Write volume is low**
- Adding, removing and editing business info are infrequent operations

<br/>


## Database cluster
- `Primary` database : Write operations
- `Secondary` database : Read operations
- Some discrepandy between data read and data written to the primary database
- This inconsistency is usually `not an issue` because business information doesn't need to be updated in real-time

<br/>

# Algorithms to fetch nearby businesses

+) Companies might use existing geospatial databases such as Geohas in Redis or Postgres wit hPostGIS extension

<br/>

## Option 1: Two-dimensional search

![IMG_2698](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/4379a554-2faf-4789-a2bf-cd681947cbb0)


```sql
SELECT business_id, latitude, longtitude
FROM business
WHERE 
(latitude BETWEEN {:my_lat} - radius AND {:my_lat} + radius) 
AND
(latitude BETWEEN {:my_long} - radius AND {:my_long} + radius) 
```

- We need to perform an intersect operation on those two datasets
- Not efficient because we need to `scan the whole table`

<br/>

### 👉 Can we map two-dimensional data to one dimension?

- Divide the map into smaller areas and build indexes for fast search

ex) geohash, quadtree, Googld S2 etc

<br/>

## Option 2: Evenly divided grid

- Issue : The distribution of business is not even
ex) There could be lots of business in downtown in NewYork

<br/>

## Option 3: Geohash

- Geohash is better than the evenly divded grid option
- It algorithms work by recursively dividing the world into smaller and smaller grids with each additional bit.

<br/>

**1. Divide the planet into four quadrants along with the prime meridian and equator**

![IMG_2700](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/3caab1e3-1d95-4388-ae63-5fef26f00bb3)

<br/>

**2. Second, divide each grid into four smaller grides**

![IMG_2701](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/74410e82-14a2-4232-9435-a1f016f75de4)

<br/>

**3. Repeat this subdivision until the grid size is within the precision desired**

ex) length=6 : `01001 10110 01001 10000 11011 11010` (base32 in binary) -> 9q9hvu

+) `Base32` : encoding the input data five bytes at a time.

<br/>

- We are only interested in geohashes with lengths between `4 and 6`

![IMG_2703](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/07b8e3c1-bbd6-4b78-9b48-613ea4baa45a)

<br/>

### Issues

#### 1. Boundary Issues

ex) prefix : 9q8zn

![IMG_2704](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/374e62cc-9dd0-4da0-82bf-c3f22ff272c2)

<br/>

**1-1) Two locations can be very close but have no shared prefix at all**

![IMG_2705](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/beefdd6c-3476-42b8-a850-86c26f8c9f87)

- Two close locations on either side of the equator or prime meridian belong to different halves of the world
- A simple prefix SQL query would fail to fetch all nearby businesses

```sql
SELECT * FROM geohash_index WHERE geohash LIKE '9q9zn%'
```

<br/>

**1-2) Two positions can have a long shared prefix, but they belong to different geohashs as shown**

![IMG_2706](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/9d396a12-d46c-495c-b87f-808e0f6b3c27)

- A common solution is to fetch all business not only within the current grid but also from its neighbors

<br/>

#### 2. Not enough Business

What should we do if there are not enough businesses returned from the current grid and all the neighbors combined?

- Option 1 : Only return businesses within the radius
- Option 2 : Increase the search radius

<br/>

## Option 4: Quadtree

- Another popular solution
- A quadtree is a data structure that is commonly used to partition a two-dimensional space by recursively subdividing it into `four quadrants until the contents of the grids meet certain criteria`

![IMG_2708](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/ffdba74d-56d9-4998-93f6-af1764e4e7fe)

- Quadtree is an `in-memory data structure` and it is not a database solution
- It runs on each LBS server and the data structure is `built at server start-up time`
- The root node is recursively broken down into 4 quadrants `until no nodes are left with more than 100 businesses`

<br/>

### How much memory does it need to store the whole quadtree?

- Each grid can store a maximal of 100 businesses
- Number of leaf nodes = ~200 million / 100 = ~2 million
- Number of internal nodes = 2 million x 1/3 = ~0.67
- Total memory requirement = 2 million x 832 bytes + 0.67 million x 64 bytes = ~1.71 GB

**+) Why 1/3 of leaf nodes are internal nodes?**



**+) 832 byte**

![IMG_2709](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/39e07b4d-916b-4f09-80e5-645184560ee5)

**+) 64 byte**

![IMG_2710](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/f39bbe33-216a-4cbd-95c7-9c8d74803f7f)

- The quadtree index doesn't take too much memory and can easily fit in one server

<br/>

### How long does it take to build the whole quadtree?
- The time complexity to build the tree = `(n/100)log(n/100) (n: number of businesses)`
- It might take `a few minutes` to build the whole quatree with 200 million businesses

<br/>

### The operational implications of such a long server start-up time
- We should toll out a new release of the server incrementally to a small subset of servers at a time
- `Blue/Green deployment` can also be used, but an entire cluster of new servers fetching 200 million businesses at the same time from the database service can put a lot of strain on the system

<br/>

### How to update the quadtree as businesses are added and removed over time
- Some servers would `return stale data` for a short period of time
- This is generally an `acceptable compromise` based on the requirements

<br/>

## Option 5: Google S2

ex) Google, Tinder etc

- `In-memory solution`
- It maps a sphere to a 1D index based on the `Hilbert curve`

<br/>

**The Hilbert curve**
- Two points that are close to each other on the Hilbert curve are close in 1D space
- Search on 1D space is much more efficient than on 2D

![IMG_2711](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/a94fab5e-8c59-463b-ac01-e83f14326037)

<br/>

### Advantages

**1. It can cover arbitrary areas with varing levels**
- Geofencing allows us to define perimeters that surround the areas of interest and to send notifications to users who are out of the areas

![IMG_2712](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/142f06d3-6233-40dc-9546-a173c4b8564f)

**2. We can specify min level, max level and max cells**

<hr/>

![IMG_2713](https://github.com/JaeYeonLee0621/SystemDesign-Interview/assets/32635539/b85290f4-bc22-4f8d-b15e-032d7e94a2cd)

