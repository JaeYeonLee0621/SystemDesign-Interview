# Chapter 9. S3-like Object Storage

# Storage System 101

![KakaoTalk_Photo_2024-05-13-10-54-35 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a57826d1-f1e5-4462-8c98-2f955f742196)

## Block Storage
- It came first, in the 1960s
- Common storage devices like Hard Disk Drives (HDD) and Solid State Drives (SSD) that are physically attached to servers are `all considered as a block`
- It presents `the raw blocks` to the server as a volume, which is the most flexible and versatile form
- It is `not limited` to physically attached storage
- It could be connected to a server over a `high-speed network` or `over industry-standard connectivity protocols` like Fibre Channel (FC) and iSCSI
- It is fully owned by a single server, `not a shared resource`

## File Storage
- It is built on `top of block storage`
- It provides `a higher-level abstraction` to make it easier to handle files and directories
- Data is stored as files under a `hierarchical directory structure`
- The common file-level network protocols are `SMB/CIFS` and `NFS`
- The simplicity of file storage makes it a great solution for sharding a large number of files and folders within an organization

## Object Storage
- It targets relatively `"cold" data` (such as, archival and backup)
- It stores all data as `objects in a flat structure`
- There is `no heirarchical directory structure`
- Data access is normally provided via a `RESTful API`

ex) Amazone S3, Google block stoage, Azure blob storage

<br/>

# Terminology

## Bucket
- A logical container for objects
- The bucket name is globally unique
- To upload data to S3, we must first create a bucket

## Object
- It is an individual piece of data we store in a bucket
- It contains objects data and metadata

### +) Object data
- It can be any sequence of bytes we want to store
### +) Metadata
- It is a set of name-value pairs that describe the object

## Versioning
- A feature that keeps multiple variants of an object in same bucket
- It is enabled at bucket-level
- The users to recover objects that are deleted or overwritten by accident

## Uniform Resource Identifier (URI)
- Each resource is uniquely identified by its URI

## Service-level agreement (SLA)
- It is a contract between a service provider and a client

ex) Amazon S3
- Designed for durability of 99.999999999% of objects across multiple Availability Zones
- Data is resilient in the event of one entire Availability Zone destruction
- Designed for 99.9% availability

<br/>

# Step 1. Understand the Problem and Establish Design Scope

# Non-functional requirements

- 100PB of data
- Data durability is 6 nines
- Service availability is 4 nines
- Storage efficiency : Reduce storage costs while maintaining a high degree of reliability and preformance

# Back-of-the-envelope estimation

## Disk capacity
- 20% of all objects are small objects (less then 1MB)
- 60% of objects are medium-sized objects (1MB~64MB)
- 20% are large objects (size larger than 64MB)

## IOPS
- Assume that hard disk (SATA interface, 7200 rpm) is capable of doing 100~150 random seeks per second (100~150IOPS)

## Simplify the calculation
- 100PB = 100 x 1000 x 1000 x 1000MB = 10^11MB
- Small, Medium, Large object = 0.5MB, 32MB, 200MB
- (10^11 x 0.4)/(0.2 x 0.5MB + 0.6 x 32MB + 0.2 x 200MB) = 0.68 billion objects
- If we assume the metadata of an object is about 1KB in size, we need 0.68TB space to store all metadata information

# Step 2. Propose High-LEvel Design and Get Buy-In

# Characteristics of object storage

## Object immutability
- The main difference between objects storage and the other two types of storaage systems is that objects stored inside of `object storage are immutable`
- Delete them or replace them entirely with a new version

## Key-value store
- The object URI is the key and object data is the value

## Write once, read many times
- According to the research done by LinkedIn, 95% of requests are read operation

## Support both small and large objects

# Design philosophy of object storage

![KakaoTalk_Photo_2024-05-13-10-54-36 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9f238d04-b343-4c0f-88af-663898c7dd68)

- It is very similar to that of `the UNIX file system`

## Unix
- the filename is stored in a data structure called inode
- the file data is stored in different disk locations

### Inode
- A list of file block pointers that point to the disk locations of the file data
- We first fetch the metadata in the inode, read the file data by following the file block pointers to the actual disk location

## Object Storage

![KakaoTalk_Photo_2024-05-13-10-54-36 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/bfc1085c-6478-4a1b-b550-6338b4027a38)

# High-Level Design

![KakaoTalk_Photo_2024-05-13-10-54-36 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/4f13ccef-6131-475d-bf45-408cfed6d40c)

## Data store

![KakaoTalk_Photo_2024-05-13-10-54-36 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f09a2a24-5869-43bf-baea-bfc3adeb3d0d)

- Stores and retrieves the actual data
- All data-related operations are based on `object ID (UUID)`

# Uploading an Object

![KakaoTalk_Photo_2024-05-13-10-54-36 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0cd7dc7d-ed68-4637-87bd-957b37db7605)

1. The client sends an HTTP PUT request to create a bucket
2. The API service calls the IAM to ensure the user is authorized and has WRITE permission
3. The API service calls the metadata store to create an entry with the bucket info in the metadata database

<br/>

1. The client sents an HTTP PUT request to create an objects named "file"
2. The API service calls the IAM to ensure the user is authorized and has WRITE permission
3. The API service sends the objects data in the HTTP PUT payload to the data store and returns the UUID of the objects.
4. The API service calls the metadata store to create a new entry in the metadata database.

# Downloading an object

![KakaoTalk_Photo_2024-05-13-10-54-36 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7c83cd85-ef8b-4d81-b37b-912e9f620b0d)

1. The client sends an HTTP GET request to the load balancer
2. The API service queries the IAM to verify that the user has READ access to the bucket
3. The API service fetches the corresponding object's UUID from the metadata store
4. The API service fetches the object data from the data store by its UUID
5. The API service returns the object data to the client in HTTP GET response

<br/>

# Step 3. Design Deep Dive

# Data store

![KakaoTalk_Photo_2024-05-13-10-54-36 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1301f33c-7044-4193-bcdd-6c455987d4b7)

## Data Routing Service
- The data routing service provides RESTful or gRPC APIs to the data node cluster

## Placement Service
- It determines which data nodes should be chosen to store an object
- It maintains a virtual cluster map

![KakaoTalk_Photo_2024-05-13-10-54-36 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b0359f76-c2d6-4535-918b-c2e4ad8456ba)

- This is a critical service, we suggest building a cluster of 5 or 7 placement service nodes with Paxos or Raft consensus protocol

### Consensus Protocol
- It ensures that as long as more than half of the node are healthy, the service as a whole continues to work

## Data Node
- It stores the actual object data
- Each data node has a data service daemon running on it, which continuously sends heartbeats to the placement service
- When the placment service receives the heartbeat for the first time, it assings an ID for this data node, adds it to the virtual cluster map, and returns the following information

### +) Heartbeat Message
- How many disk drives does the data node manage?
- How much data is stored in each drive?

### +) Return information
- A unique ID of the data node
- The virtual cluster map
- Where to replicate data

# Data Persistence flow

![KakaoTalk_Photo_2024-05-13-10-54-36 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/92a0e6e2-7dc7-44e5-84c3-fcd3d06f0f3a)

1. The API service forwards the objects data to the data store
2. The data routing service generates a UUID for this object and queries the placement service to find out the data node to store this object
3. The data routing service sends data directly to the primary data node, together with its UUID
4. The primary data node saves the data locally and replicateds it to 2 secondary data nodes
5. The UUID of the object is returned to the API service

## Trade-off between consistency and latency

![KakaoTalk_Photo_2024-05-13-10-54-36 011](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ebd1441f-5162-4f0b-9331-3c1c033102d2)

# How data is organized

- Store each object in a stand-alone file
- This works, but the performance sufferes `when there are many small files`

# Too many small files on a file system

## Problems

### 1. It wastes many data blocks
- Disk blocks have the same size and the size is fixed when the volume is initialized
- The typical block size is around 4KB
- For a file smaller then 4KB, it would still consume the entire disk block

### 2. It could exceed the system's inode capacity
- For most file systems, the number of inode is fixed when the disk is initialized
- The OS system does not handle a large number of inodes very well

## Solutions

### 1. We can merge many small objects into a large file
- It works conceptually like a `WAL (Write Ahead Log)`

### +) WAL (Write Ahead Log)

![KakaoTalk_Photo_2024-05-13-10-54-36 012](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1cd01ec5-619d-45f6-b63f-e4c82faf6869)

- When we save an object, it is appended to an existing read-write file
- When the read-write file reaches its capacity threshold - usually set to a few GBs - the read-write file is marked as read-only, and a new read-write file is created to receive new objects

## Serialized Data

- The read-write file must be `serialized`

### Problem
- For a modern server with a large number of cores processing many incoming requests in parallel, this `seriously restricts write throughput`

### Solution
- To fix this, we could provide `dedicated read-write files`, one for each core processing incoming requests

# Object lookup

## NoSQL
- Fast for writes but slower for reads

## Relational DataBase
- Fast for reads but slower for writes :check:

### Deployment
- Mapping data is isolated within each data node
- A simple relational database on each data node

> SQLite is a good choice here = file-based relational database with a solid reputation

# Updated data persistence flow

![KakaoTalk_Photo_2024-05-13-10-54-37 013](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1cbc6792-5708-4d14-a21d-4ae2279bb941)

# Durability

## 1. Hardware failure

- Replicate data to multiple hard drives, so single disk failure does not impact the data availability

### Calculation
- Assume, spinning hard drive has an annual failure rate of 0.81%
- Making 3 copies of data gives us 1-(0.0081)^3 ~= 0.999999 reliability

## 2. Failure domain
- In a modern data center, a server is usually put into a rack, and the racks are grouped into rows/floors/rooms
- Since each rack shares network switches and power, all the servers in a rack are in a `rack-level failure domain`

ex) Data centers divide infrastructure that shares nothing into different Availability Zones

![KakaoTalk_Photo_2024-05-13-10-54-37 014](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/36bad591-64e9-400e-9f8b-55821939ea69)

- The choice of failure domain level doesn't directly increase the durability of data, but it will result in better reliability in extreme cases, such as large-scale power outages, cooling system failures, nature disasters, etc

## 3. Erase Coding

![KakaoTalk_Photo_2024-05-13-10-54-37 015](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/4f515b42-31b2-45d7-ad1e-fad41af9152c)

- It deals with data durability differently
- It chunks data into smaller pieces and creates parities for redundancy
- In the event failures, we can use chunk data and parities to reconstruct the data

# Data corruption

- Verifying checksums between process boundaries

## Checksum

![KakaoTalk_Photo_2024-05-13-10-54-37 016](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/263601c0-e4c8-4912-bc3e-8a3f2edeb2b2)

- It is a small-sized block of data that is used to detect data errors
- If they are different, data is corrupted
- Checksum algorithm : MD5, SHA1, HMAC etc
- A good checksum : outputs a sinificantly different value even for a small change made to the input

> Simple checksum algorithm : MD5

1. Fetch the object data and the checksum
2. Compute the checksum against the data received

# Object Versioning

- Keeping multiple versions of an object in a bucket
- With versioning, we can restore objects that are accidentally deleted or overwritten

![KakaoTalk_Photo_2024-05-13-10-54-37 017](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/64a32ab3-db4e-4f93-b584-06c68405a2d9)

- The object version is a TIMEUUID generated when the new row is inserted
- It should be efficient to look up the current version of an object

![KakaoTalk_Photo_2024-05-13-10-54-37 019](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/94c92cdb-7efa-4e3b-8e7e-b1eaba1b7374)

# Optimizing uploads of large files
- It is possible to upload such a large object file directly, but it could take a long time

## Multipart Upload

![KakaoTalk_Photo_2024-05-13-10-54-37 020](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/e796f8c5-c178-4225-86f5-bd34d7abb39b)

- Slicing a large object into smaller parts and upload them independently
- After all the parts are uploaded, the object store re-assembles the object from the parts
- ETag : MD5 checksum of that part, verifying multipart uploads

# Garbage collection
- It is the process of automatically reclaimoing storage space that is no longer used

## Candidates
1. Lazy object deletion.
2. Orphan data (ex) half uploaded data or abandoned multipart uploads)
3. Corrupted data (ex) fail the checksum verification)

## Compaction

![KakaoTalk_Photo_2024-05-13-10-54-37 021](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c02cf5ab-fda3-4a7b-bd23-9cf74dc29e2a)

- Periodically cleaned up with a compaction mechanism
