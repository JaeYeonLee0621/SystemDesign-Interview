A Proximity Service : Discover nearby places

# Step 1. Understand the Problem and Establish Design Scope

Q) Can a user specify the serach radius?
A) We only care about business within a specified radius

Q) What is the maximal radius allowed?
A) 20km

Q) Can a user change the search radius on the UI?
A) 0.5km, 1km, 2km, 5km, 20km

Q) Business information need to reflect in real-time?
A) Business owners can add, delete or update a business. Assume newly added/updated businesses will be effective the next day.

Q) A user might be moving while using the app/web-site. Do we need to refresh the page to keep the results up to date?
A) Let's assume a user's moving speed is slow and we don't need to constantly refresh the page

## Back-of-the-envelope estimation

- Assume we have 100 million daily active users and 200 million businesses
- Seconds in a day = 24 x 60 x 60 = 86,400 (round it up to 10^5)
- Assume a user makes 5 search queries per day
- Search QPS = 100million x 5 / 10^5 = 5,

# Step 2 - Propose High-Level Design and Get Buy-In

## Data model

**[Read/Write Ratio]**

Read volume is high
- Search for nearby businesses
- View the detailed information of a business

Write volume is low
- Adding, removing and editing business info are infrequent operations

## High-Level Design
- The system comprises 2 parts

1. LBS (Location-Based Service)
2. Business-Related Service

## LBS (Location-Based Service)
- Read-heavy service with no write requests
- QPS is high, especially during peak hours in dense areas
- Stateless, easy to scale horizontally

## Business Service
- Business Owner create, update, or delete businesses
- QPS is high during peak hours

## Database cluster
- Primary database : Write operations
- Secondary database : Read operations
- Some discrepandy between data read and data written to the primary database
- This inconsistency is usually not an issue because business informatino doesn't need to be updated in real-time

# Algorithms to fetch nearby businesses

- Companies might use existing geospatial databases such as Geohas in Redis or Postgres wit hPostGIS extension

## Option 1: Two-dimensional search

```sql
SELECT business_id, latitude, longtitude
FROM business
WHERE 
	(latitude BETWEEN {:my_lat} - radius AND {:my_lat} + radis) 
AND (latitude BETWEEN {:my_long} - radius AND {:my_long} + radis) 
```

- Not efficient because we need to scan the whole table
- We need to perform an intersect operation on those two datasets
- This is not efficient because each dataset contains lots of data

> Can we map two-dimensional data to one dimension?

- Divide the map into smaller areas and build indexes for fast search

ex) geohash, quadtree, Googld S2 etc

### Option 2: Evenly divided grid

- Issue : The distribution of business is not even
ex) There could be lots of business in downtown in NewYork

### Option 3: Geohash

- Geohash is better than the evenly divded grid option
- It algorithms work by recursively dividing the world into smaller and smaller grids with each additional bit.

1. Divide the planet into four quadrants along with the prime meridian and equator
2. Second, divide each grid into four smaller grides
3. Repeat this subdivision until the grid size is within the precision desired

ex) length=6 : 1001 10110 01001 10000 11011 11010 (base32 in binary) -> 9q9hvu

- We are only interested in geohashes with lengths between 4 and 6

#### Boundary Issues

ex) prefix : 9q8zn

1. Two locations can be very close but have no shared prefix at all
- Two close locations on either side of the equator or prime meridian belong to different halves of the world
- A simple prefix SQL query would fail to fetch all nearby businesses

```sql
SELECT * FROM geohash_index WHERE geohash LIKE '9q9zn%'
```

2. Two positions can have a long shared prefix, but they belong to different geohashs as shown

- A common solution is to fetch all business not only within the current grid but also from its neighbors

#### Not enough Business

What should we do if there are not enough businesses returned from the current grid and all the neighbors combined?

Option 1 : Only return businesses within the radius
Option 2 : Increase the search radius

### Option 4: Quadtree

- Another popular solution
- A quadtree is a data structure that is commonly used to partition a two-dimensional space b yrecursively subdividing it into four quadrants until the contents of the grids meet certain criteria
- With a qudtree, we build an in-memory tree structure to answer quries
- Quadtree is an in-memory data structure and it is not a database solution
- It runs on each LBS server and the data structure is built at server start-up time
- The root node is recursively broken down into 4 quadrants until no nodes are left with more than 100 businesses

#### How much memory does it need to store the whold quadtree?

- Each grid can store a maximal of 100 businesses
- Number of leaf nodes = ~200 million / 100 = ~2 million
- Number of internal nodes = 2 million x 1/3 = ~0.67
- Total memory requirement = 2 million x 832 bytes + 0.67 million x 64 bytes = ~1.71 GB

- The quadtree index doesn't take too much memory and can easily fit in one server

#### How long does it take to build the whole quadtree?
- The time complexity to build the tree = (n/100)log(n/100) (n: number of businesses)
- It might take a few minues to build the whole quatree with 200 million businesses

#### The operational implications of such a long server start-up time
- We should toll out a new release of the server incrementally to a small subset of servers at a time
- Blue/Gree deployment can also be used, but an entire cluster of new servers fetching 200 million businesses at the same time from the database service can put a lot of strain on the system

#### How to update the quadtree as businesses are added and removed over time

- Some servers would return stale data for a short period of time
- This is generally an acceptable compromise based on the requirements

### Option 5: Google S2

ex) Google, Tinder etc

- In-memory solution
- It maps a sphere to a 1D index based on the Hilbert curve

The Hilbert curve
- Two points that are close to each other on the Hilbert curve are close in 1D space
- Search on 1D space is much more efficient than on 2D

#### Advantages

1. It can cover arbitrary areas with varing levels
- Geofencing allows us to define perimeters that surround the areas of interest and to send notifications to users who are out of the areas

2. We can specify min leve, max level and max cells

## Step 3. Design Deep Dive

### Scale the database

#### Geospatial index table

Option 1. geohash, list_of_business_ids
Option 2. geohash, business_id

> Recommend: Option 2

Option 1
- Update a business, we need to fetch the array of business_ids and scan the whole array to find the business to update
- Inserting a new business, we have to scan the entire array to make sure there is no duplicate
- Lock the row to prevent concurrent updates

Option 2
- The addition and removal of a business are very simple

#### Scale the geospatial index
- Geohah table sharding is complicated
- A better approach, series of read replicas to help with the read load
