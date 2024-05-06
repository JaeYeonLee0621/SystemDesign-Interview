# Chapter 8. Distributed Email Service

# Step 1. Understand the Problem and Establish Design Scope

## Basic of the envelope estimation

### Assume QPS for sending emails
- 1 billion users
- The average number of emails a person sends per day is 10

> 10^9 x 10 / (10^5) = 100,000

### Assume Metadata
- The average number of emails a person receives in a day is 40
- The average size of email metadata is 50KB

+) Metadata refers to everything related to an email, excluding attchment files

- Metadata is stored in a database. 
- Storage requirement for maintaining metadata in 1 year

> 1 billion users x 40 emails / day x 365 days x 50KB = 730 PB

### Assume Storage for attchments in 1 year
- 20% of emails contian an attchment
- The average attachment size is 500KB

> 1 billion uesrs x 40 emails / day x 365 days x 20% x 500KB = 1,460 PB

# Step 2. Propose High-Level Design and Get Buy-In

# Email Protocols

## Sending Emails

### SMTP (Simple Mail Transfer Protocol)
- Standard protocol for sending emails from one mail server to another

## Retrieving Emails

### POP (Post Office Protocol)
- RFC 1939
- Once emails are downloaded to your computer or phone, they are deleted from the email server, which means you can only access emails on one computer or phone

### IMAP (Internet Mail Access Protocol)
- RFC 3501
- It only downloads a message when you click it and emails are not deleted from mail servers
- You can access emails from multiple devices

### HTTPS
- It is not technically a mail protocol, but i can be used to access your mailbox, particularly for web-based email

ex) Microsoft Outlook

## Domain name service (DNS)
- It is used to look up the mail exchanger record (MX record) for receipient's domain

![KakaoTalk_Photo_2024-05-06-12-52-40 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/862e2eae-d4c8-4d35-99c5-dab443373b6f)


### The priority numbers
- Preferences where the mail server with a lower priority numbers is more preferred
- A sending mail server will attempt to connect and send messages to this mail server first
- If the connection fails, the sending mail server will attempt to connect to the mail server with the next lowest priority

## Attachment
- Base 64 encoding
- A size limit for an email attachment is highly configurable and varies from individual to corporate accounts
- Multipurpose Internet Mail Extension (MIME) is a specification that allows the attachment to be sent over the internet

### Side note: Tranditional mail servers architecture

![KakaoTalk_Photo_2024-05-06-12-52-40 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/19cd1d36-3c4f-497e-914f-99c680809aeb)

## Storage
- Each email was stored in a seperate file with a unique name
- Each user maintained a user directory to store configuration data and mailboxes

### Maildir

![KakaoTalk_Photo_2024-05-06-12-52-40 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/213b3834-410c-4d2e-bb5c-752ab91d71c9)

- As the email volume grew and the file structure became more complex, disk I/O became a bottleneck
- The local directories also don't satisfy our high avilability and reliabiltiy requirements

## Distributed mail server architecture

![KakaoTalk_Photo_2024-05-06-12-52-40 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1c10f05c-dd79-4328-b631-f38cc3e05045)

### Webmail
- Users use web browsers to receive and send emails

### Web servers
- Web servers are public-facing request/response services/API, used to manage features

### Real-time servers
- It is responsible for pushing new email updates to clients in real-time
- These are stateful servers because they need to maintain persistent connections

<br/>

`Websocket`
- advantage : elegant solution / drawback : browser compatibility
- To relieve the drawback, establishing a Websocket connection whenever possible and to use long-polling as a fallback

### Metadata database
- The database stores mail metadata including mail subject, body, from users etc

### Attachment store

`Amazon Simple Storage Service (S3)`
- It is a scalable storage infrastructure that's suitable for storing large files

`NoSQL`
- Column-family databases
- Cassandra supports blob data type and its maximum theoretical size for a blob is 2 GB, **The practical limit is less than 1MB**
- We can't use a row cache as attachments take **too much memory space**

### Distributed cache
- Since the most recent emails are repeatedly loaded by a client, caching recent emails in memory significantly improves the load time
- using redis because it offers rich features such as lists and it is easy to scale

### Search store
- The search store is a distributed document store
- Inverted index that supports very fast full text searches

## Email sending flow

![KakaoTalk_Photo_2024-05-06-12-52-41 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/43aae2b3-e390-424a-b842-ca4511f3c56b)

### Basic email validation
- Each incoming email is checked against pre-defined rules such as email size limit
- Checking if the domain of the recipient's email address is the same as the sender

### Metadata
- Each message in the outgoing queue contains all te metadata required to create an email

### Asynchronous mail processing
- A distributed message queue is a critical component that allows asynchronous mail processing
- By decoupling SMTP outgoing workers from the web servers, we can scale SMTP outgoing workers independently

### Monitoring the size of the outgoing queue
- The recipent's mail server is unavailable : In this case, we need to retry sending the email at a later time
- Not enough consumers to send emails : We may need more consumers to reduce the processing time

## Email receiving Flow

![KakaoTalk_Photo_2024-05-06-12-52-41 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0b5874e0-e2be-4a6d-8e3e-5fe93dcab23f)

### Email Acceptance policy
- It can be configured and applied at the SMTP-connection level

ex) Invalid emails are bounced to avoid unnecessary email processing

### Attachement
- If the attachement of an email is too large to put into the queue, we can put it into the attachment store (s3)

### Online/Offline
- If the receiver is currently online, the email is pushed to real-time servers
- Real-time servers are WebSocket servers that allow clients to receive new emails in real-time
- For offline users, emails are stored in the storage layer. When a user comes back online, the webmail client connects to web servers via RESTful API
- Web servers pull new emails from the storage layer and return them to the client

# Step 3. Design Deep Dive

# Metadata database

## Characteristics of email metadata
- Email headers are usually small and frequently accessed
- Email body sizes can range from small to big but are infrequently accessed. You normally only read an email once.
- Mails owned by a user are only accessible by that users and all the mail operations are performed by the same user
- Data recency impacts data usage : 82% of read queries are for data younger than 16 days

## Choosing the right database
- The database system is usually custom-made to reduce input/output operations per second (IOPS) as this can easily become a major constraint in the system

### Relational databse

`PROS`
- We can build indexes for email header and body
- Simple search qureis are fast

`CONS`
- Relational databases are typically optimized for small chunks of data entreis and are not ideal for large one
- Search qureis over unstructured BLOB data type are not efficient

### Distributed object storage
- Store raw emails in cloud storage such as Amazon S3

`PROS`
- A good option for backup storage

`CONS`
- Hard to efficiently support features such as marking emails as read, searching emails based on keywords etc

### NoSQL databases
- Google Bigtable is used by Gmail

`PROS`
- It's a viable solution

`CONS`
- Bigtable is not opensourced an how email search is implemented remains a mystery

## Data model

- One user is stored in a single shard
- One potential limitation with this data model is that mesages are not shared among multiple users

### The partition key
- It is responsible for distributing data cross nodes
- We want to spread the data envely

ex) user_id

### The clustering key
- It is responsible for sorting data within a partition

ex) user_id, folder_id / email_id (TIMEUUID)

> NoSQL database normally only supports queries on partition and cluster keys

### Procedure of querying data

1. Getting all folders for a user
2. Displaying all emails for a specific folder
3. CRD a special emails
4. Fetching all read or unread emails

- Denormalization tables : read_emails, unread_emails (user_id, folder_id)
- Improvement the read performance of these queries at scale

### Bonus point: conversation threads
- Threads are a feautre supported by many email clients
- It groups email replies with their original message
- This allows uesrs to retrieve all emails associated with one conversation
- Traditionally : a thread is implemented using algorithms such as JWZ algorithm

![KakaoTalk_Photo_2024-05-06-12-52-41 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/13629d07-4ab9-49b8-bae1-628bdbd2ec76)

## Email deliverability
- The hard part its to get emails actually delivered to a user's inbox
- If an email ends up in the spam folder, it means there is a very high chance a recipient won't read it
- Email spam is a hugh issue

### Dedicated IPs
- Dedicated IP addresses for sending emails
- Email providers are less liekly to accept emails from new IP addresses that have no history

### Classify emails
- Send different categories of emails from different IP addresses

ex) avoiding sending marketing and important emails from the same servers because it might make ISPs mark all emails as promotional

### Warm-up email sending slowly
- Warm-up new IP addresses to build a reputation with bih providers
ex) According to Amazon Simple Email Service, it takes about 2 to 6 weeks to warm up a new IP address

### Ban spammers quickly
- Spammers should be banned quickly before they have a significant impact on the server's reputation

### Retries
- If the system fails to process an event, it retires
- If the maximum retry threshold is reached, the event is stored in a queue

### Feedback processing
- It is very important to set up feedback loops with ISPs so we can keep the complaint rate low and ban spam account quickly

1. Hard bounce : Email is rejected by an ISP because the recipient's email address is invalid
2. Soft bounce : It indicates an email failed to deliver due to temporary condidions, such as ISPs being too busy
3. Complaint : A recipient clicks the "report spam" button

![KakaoTalk_Photo_2024-05-06-12-52-41 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/54a812f1-18de-4c55-8511-848e67acaa92)

### Email authentication
- Some of the common techniques to combat phishing are :

1. Sender Policy Framework (SPF)
2. DomainKeys Identified Kail (DKIM)
3. Domain-based Message Authentication, Reporting and Conformance (DMARC)

![KakaoTalk_Photo_2024-05-06-12-52-41 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/51de3949-f358-4e2b-9e18-8e54d4c76abc)

### Search
- Searching for emails that contains any of the entered keywords in the subject or body
- The search features in email systems has a lot more writes than reads

`Option 1 : EalsticSearch`
- We can group underlying documents to the same nodes using user_id as the partition key

![KakaoTalk_Photo_2024-05-06-12-52-41 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b2205d98-08ca-462c-a360-69f1c5ad86a1)

- A search request is synchronous
- Reindexing is needed and it can be done with offline jobs
- Kafka is used in the design to decouple services that trigger reindexing from services that actually perform reindexing
- One challenge of adding Elasticsearch is to keep out primary email store in synch with it

`Option 2: Custom Search Solution`
- The main bottleneck of the index server is usually disk I/O
- Since the process of building the index is write-heavy, a good strategy might be to use Log-Structure Merge-Tree (LSM)

![KakaoTalk_Photo_2024-05-06-12-54-56](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/41038ab5-45c7-43ad-91c8-209f7fc6a0ca)

- The write path is optimized by only performing sequential writes
- LSM trees are the core data structure behind databases such as BigTable, Cassamdra and RocksDB
- When a new email arrives, it is first added to level 0 in memory cache, and when data size in memory reaches the predefined threshold, data is merged to the next level
- LSM is to seperate data that change frequently from those that don;t

ex) Folder metadata : change more often due to different filter rules
Email data : don't change

> A general rule of thumb is that for a smaller scale email system, Elastic search is a good option

## Scalability and availability
- Replicated across multiple data centers
- User communicate with a mail server that is physically closer to them in the network topology

![KakaoTalk_Photo_2024-05-06-12-52-46](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/24548713-9146-4409-86d5-55fd0a39116b)
