# Chapter 5. Metrics Monotoring and Alerting System

<br/>

# Step 1. Understand The Problem and Establish Design Scope

## High-Level requirements and assumption

- 100 million daily active users
- 1000 server pools, 100 machines per pool, 100 metrics per machine => ~10 million metric
- 1 year data retention
- Data rention policy : raw form for 7 days, 10minute resolution for 30 days, 1-hour resolutionn for 1 year
- Metrics can be monitored including Low-level (ex) EPU usuage etc), High-level (ex) Request count)

## Non-functional requirements

- Scalability, Low latency, Reliability, Flexibility

<br/>

# Step 2. Propose High-Level Design and Get Buy-In

## Fundamentals

<image: 5-Fundamenetals>

## Data model
- It is recorded as a time series that contains a set of values with their associated timestamps.

example) CPU load

<image: CPU load>

## Data access pattern

<image: Data access pattern>

- y-axis : a time series (uniquely identified by the names and labels)
- x-axis : time

<br/>

- `write load` : heavy
+) Many time-series data points written at any moment

- `read load` : spiky
+) Visualization and alert services send queries to the DB and depending on the access patterns of the graphs and alerts

<br/>

## Data Storage System

### Relational DataBase
- General Purpose RDB does not perform well under constant heavy write load
- It would require expert-level tuning to make it work at scale

ex)
- Time-Series Data : computing the moving average in a rolling time window requires complicated SQL that is difficult to read
- Tagging/Labeling data : Need to add inde

### +) PostgreSQL write logic

### NoSQL

- Cassandra, Bigtable can both be use for time series
- However, it would require deep knowledge of the internal workings of each NoSQL

### Time Series Database

#### OpenTSDB
- Distributed time-series database
- It is based on Hadoop and Hbase, running a Hadoop/HBase cluster adds complexity

#### MetricsDB
- Twitter uses and Amazon offers Timestream as a time-serie sdatabase

#### InfluxDB, Prometheus
- They are designed to store large volumes of time-series data
- Quickly perform real-time analysis on that data
- Rely on an in-memory cache and on-disk storage
- It is efficient aggregation and analysis of a large amount of time-series data bby labels
- Handle durability and performance quite well
ex) 8 core, 32GB RAM > 250,000 writes per second

### +) Carnidality

## High-level design

<image: high-level design>

<br/>

# Step 3. Design Deep Dive

## Metrics collection

<image: high-level design>

- It's acceptable for occasional data loss

<br/>

## Pull Model

- Over HTTP
- The metrics collector needs to know the complete list of service endpoints to pull data from

ex) Use a file to hold DNS/IP information for ever service endpoint on the "metric collector" servers

- etcd, ZooKeeper etc can be notified by the Service Discovery component whenever the list of service endpoints changes

<image: Service Discovery>

### +) etcd

### Pull Model in detail

<image: Pull model in detail>

1. The metric collector fetches configuration metadata of service endpoint from Service Discovery
2. The metric collector pulls metrics data via a pre-defined HTTP endpoint
3. Optionally, the metrics collector registers a change event notification with Service Discovery to receive an update whenever the service endpoints change

### Scailiability
- A single metrics collector will not be able to handle thousands of servers
- Designate each collector to a range in a consistent hash ring, map every single server being monitored by its unique name in the hash ring

<image: Consistent Hashing>

<br/>

## Push Model

<image: Push Model>

- A collection agent is commonly installed on every server being monitored
- The collection agent may also aggregate metrics logically before sending them to metric collectors

### Aggregation
- An effective way to reduce the volume of data
- Problem : If the servers are in an auto-scaling group, then holding data locally might result in data loss
- Solution : The Metric collector should be in an auto-scling cluster with a load balancer in front of it

<image: Load Balancer>

<br/>

## Push or Push?

### Easy Debugging
- `Pull` : One endpoint

### Health check
- `Pull` : If an application server doesn't respond to the pull, you can quickly figure out if an application server is down
- Push : The problem might be cause by network issues

### Short-lived jobs 
- `Push` : Some of the batch jobs might be short-lived and don't last long enough to be pulled

### Firewall or complicated network setups
- Pull : It is potentially problematic in multiple data center setups
- `Push` : If the metrics collector is set up with a LB and Autoscaling group, it is possible to receive data from anywhere

### Performence
- Pull : Use TCP
- `Push` : Use UDP

### +) TCP
### +) UDP

### Data authenticity
- Pull : Application servers to collect metrics from are defined in config files in advance
- Push : Any kind of client cna push metrics to the metric collecters. Require whitelisting servers, or authentication

<br/>

## Scale the metrics transmission pipeline

<image: Metrics transmission pipeline>

- The metrics collector is a cluster of servers
- The cluster receives enormous amounts of data
- It is set up for autoscaling, to ensure that there are an adequate number of collector instances to handle the demand

### Queueing components
- To mitigate data loss

<image: Add Queue>

1. The metric collector sends metrics data to queueing systems like Kafka
2. The consumers or streaming processing services push data to the time-series database

**[Advantages]**
- Kafka : a highly reliable and scalable distributed messaging platform
- It decouples the data collection and data processing service
- Easily prevent data loss when the database is unavailable

### Scale through Kafka

<image: Kafka partition>

- Configure the number of partitions based on throughput requirements
- Partition metrics data by metric names, so consumers can aggregate data by metric names

<br/>

## Where aggregation can happen

### Collection agent
- simple aggregation logic

### Ingestion pipeline
- We need stream processing engines such as Flink
- Advantages : The write volume will be significantly reduced since only the calculated result is written to the DB
- Disadvantages : Handling lat-arriving events could be a challenge an another downside is existed

### Query Side
- Faw data can be aggregate over a given time period at query time
- Advantages : Not data loss with the approach
- Disadvantages : The query speed might be slower

<br/>

## Cache Layer

<image: Cache Layer>

<br/>

## Storage layer 

### Data Encoding and Compression

<image: Data Encoding>

- Just store the difference between the time, instead of the full timestamp of 32 bits

### +) 32 bits

### +) Time stamp data

### Downsampling

- The process of converting high-resolution data to low-resolution to reduce overall disk usage

<image: 10-second resolution data>

<image: 30-second resolution data>

### Cold storage
- The storage of inactive data that is rarely used
- The financial cost for cold sotrage is much lower

### +) AWS Archeiving

<br/>

## Alert System

<image: Alert System>

1. Load config files to cache servers. Rules are defined as configure files on the disk. (ex) YAML format)
2. The alert manager fetches alery configs from the cache
3. Based on config rules, the alert manager calls the query service at a predifined interval

<image: Merge alerts>

- Filter, merge and dedupe alerts

4. The alert store is a key-value database that keeps the state of all alerts. It ensures a notification is sent at least once.
5. Eligible alerts are inserted into Kafka
6. Alert consumers pull alert events from Kafka
7. Alert consumers process alert events from Kafka and send notifications over to different channels such as email, text message, PagerDuty or HTTP endpoints

### +) YAML format
