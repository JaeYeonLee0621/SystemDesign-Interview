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

```yaml
From: John Doe <johndoe@example.com>
To: Jane Smith <janesmith@example.com>
Subject: Example Email
Date: Tue, 6 May 2024 10:00:00 -0700
Message-ID: <1234567890@example.com>
MIME-Version: 1.0
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
X-Mailer: ExampleMailer v1.0
```

### Assume Storage for attchments in 1 year
- 20% of emails contian an attchment
- The average attachment size is 500KB

> 1 billion uesrs x 40 emails / day x 365 days x 20% x 500KB = 1,460 PB

# Step 2. Propose High-Level Design and Get Buy-In

# Email Protocols

![1](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f933fb84-2cc2-43c9-b042-a60f21df002d)

## Sending Emails

### [SMTP (Simple Mail Transfer Protocol)](https://www.afternerd.com/blog/smtp/)

+) [What is SMTP Server? An In-Depth Guide to Email Delivery in 2023](https://www.ranktracker.com/blog/what-is-smtp-server-an-in-depth-guide-to-email-delivery-in-2023/)

- Standard protocol for sending emails from one mail server to another

<br/>

- SMTP servers are made up of 2 key components

1. Mail Transfer Agents (MTAs)
- The trasfer of emails between servers, MTAs ensure that your messages reach their intended recipients

2. Mail Delivery Agents (MDAs)
- MDAs play a complementary role to MTAs in the email delivery process
- MDAs deliver emails to the recipient's mailbox

<br/>

- SMTP authentication : security measure that helps prevent spoofing and spamming
- SASL (Simple Authentication and Security Layer)

## Retrieving Emails

### POP (Post Office Protocol)

- [RFC 1939](https://www.ietf.org/rfc/rfc1939.txt)
- Once emails are `downloaded to your computer or phone`, they are deleted from the email server, which means you can `only access emails on one computer or phone`

### IMAP (Internet Mail Access Protocol)

- [RFC 3501](https://datatracker.ietf.org/doc/html/rfc3501)
- It only downloads a message when you click it and emails are not deleted from mail servers
- You can `access emails from multiple devices`

### HTTPS

- It is not technically a mail protocol, but i can be used to access your mailbox, particularly for web-based email

ex) Microsoft Outlook

## Domain name service (DNS)

![2](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/fc58adc5-cb44-42f5-acf6-5b91b7264fff)

- It is used to look up the mail exchanger record (MX record) for receipient's domain

```cmd
 nslookup
> set q=MX
> gmail.com
Server:		210.220.163.82
Address:	210.220.163.82#53

Non-authoritative answer:
gmail.com	mail exchanger = 40 alt4.gmail-smtp-in.l.google.com.
gmail.com	mail exchanger = 5 gmail-smtp-in.l.google.com.
gmail.com	mail exchanger = 20 alt2.gmail-smtp-in.l.google.com.
gmail.com	mail exchanger = 10 alt1.gmail-smtp-in.l.google.com.
gmail.com	mail exchanger = 30 alt3.gmail-smtp-in.l.google.com.

Authoritative answers can be found from:
```

- `# 53 = port 53` is indeed the standard port for DNS traffic
- `100.100.100.100` is indeed the IP address of the DNS server, but dummy value for demonstration or educational purpose

### DNS IP

<img width="655" alt="3" src="https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a0520773-ce75-499f-88df-6b3c9aca7ef8">

- DNS servers can be configured manually or obtained automatically from a DHCP server

### The priority numbers

- Preferences where the mail server with a lower priority numbers is more preferred
- A sending mail server will attempt to connect and send messages to this mail server first
- If the connection fails, the sending mail server will attempt to connect to the mail server with the next lowest priority

## Attachment

- Attachments are encoded using Base 64 algorithm

![5](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f23c8fe4-76f9-4661-8c07-3eacd0627d0e)

- A size limit for an email attachment is highly configurable and varies from individual to corporate accounts
- Multipurpose Internet Mail Extension (MIME) is a specification that allows the attachment to be sent over the internet
- [ISO-8859-1](https://en.wikipedia.org/wiki/ISO/IEC_8859-1)

<br/>

### Side note: Tranditional mail servers architecture

![KakaoTalk_Photo_2024-05-06-12-52-40 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/19cd1d36-3c4f-497e-914f-99c680809aeb)

## Storage

- Each email was stored in a seperate file with a unique name
- Each user maintained a user directory to store configuration data and mailboxes

### Maildir

![KakaoTalk_Photo_2024-05-06-12-52-40 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/213b3834-410c-4d2e-bb5c-752ab91d71c9)

- As the email volume grew and the file structure became more complex, disk I/O became a bottleneck
- The local directories also don't satisfy our high avilability and reliabiltiy requirements

<br/>

## Distributed mail server architecture

![KakaoTalk_Photo_2024-05-06-12-52-40 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1c10f05c-dd79-4328-b631-f38cc3e05045)

### Webmail
- Users use web browsers to receive and send emails

### Web servers
- Web servers are public-facing request/response services/API, used to manage features

### Real-time servers
- It is responsible for pushing new email updates to clients in real-time
- These are stateful servers because they need to maintain persistent connections

`Websocket` :star:
- advantage : elegant solution / drawback : browser compatibility
- To relieve the drawback, establishing a Websocket connection whenever possible and to use long-polling as a fallback

### Metadata database
- The database stores mail metadata including mail subject, body, from users etc

### Attachment store

`Amazon Simple Storage Service (S3)`
- It is a scalable storage infrastructure that's suitable for storing large files

`NoSQL`

![6](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/69c3a3d9-f8f1-42a5-9ca2-dfb2e2658a95)

- Column-family databases
- Cassandra supports blob data type and its maximum theoretical size for a blob is 2 GB, `The practical limit is less than 1MB`
- We can't use a row cache as attachments take `too much memory space`

### Distributed cache
- Since the most recent emails are repeatedly loaded by a client, caching recent emails in memory significantly improves the load time
- using `Redis` because it offers rich features such as lists and it is easy to scale

### Search store
- The search store is a distributed document store
- Inverted index that supports very fast full text searches

![7](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/6cb868cb-b181-4257-8f3b-4a03862ecfc3)

## Email sending flow

![KakaoTalk_Photo_2024-05-06-12-52-41 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/43aae2b3-e390-424a-b842-ca4511f3c56b)

### Basic email validation

- Each incoming email is checked against `pre-defined rules` such as email size limit
- Checking if the domain of the `recipient's email address is the same as the sender`

### Metadata

- Each message in the outgoing queue contains all the metadata required to create an email

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

- If the attachement of an email is too large to put into the queue, we can put it into the `attachment store (S3)`

### Online/Offline

If the receiver is currently `online`
- the email is pushed to real-time servers
- Real-time servers are WebSocket servers that allow clients to receive new emails in real-time

For `offline` users
- emails are stored in the storage layer. 
- When a user comes back online, the webmail client connects to web servers via RESTful API
- Web servers pull new emails from the storage layer and return them to the client

# Step 3. Design Deep Dive

# Metadata database

## Characteristics of email metadata

- Email headers are usually small and frequently accessed
- Email body sizes can range from small to big but are infrequently accessed. `You normally only read an email once.`
- Mails owned by a user are only accessible by that users and all the mail operations are performed by the same user
- `Data recency impacts data usage` : 82% of read queries are for data younger than 16 days

## Choosing the right database

- The database system is usually custom-made to reduce input/output operations per second (IOPS) as this can easily become a major constraint in the system

### Relational databse

`PROS`
- We can build indexes for email header and body
- Simple search queries are fast

`CONS`
- Relational databases are typically optimized for small chunks of data entries and are not ideal for large one (ex. html)
- Search qureis over unstructured Binary Large Object (BLOB) data type are not efficient

+) BLOB
- the native binary foramt = not directly human-readable
- it is suitable for "large objects"

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

![10](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0ca6f46a-801b-4bbf-acb1-84b10733fb7d)

- One user is stored in `a single shard`
- One potential limitation with this data model is that mesages are not shared among multiple users

![9](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ff2d834f-ddcc-413a-ac3e-d7f301d0fd82)

### The partition key
- It is responsible for distributing data cross nodes
- We want to spread the data envely

ex) user_id

### The clustering key
- It is responsible for sorting data within a partition

ex) user_id, folder_id / email_id (TIMEUUID)

**+) [TIMEUUID](https://pythonhosted.org/time-uuid/)**

- version 1 uuid
- `60-bit timestamp` : this is based on the number of 100-nanosecond intervals since October 15, 1582
- `14-bit clock sequence` : used to maintain uniqueness across changes in the system's network address
- `48-bit Node ID` : usually the MAC address of the machine generating the UUID = hardware-specific identifier

<br/>

- Privacy concerns : contaning the MAC address of the machine where it was generated
- Sorting : ordering according to the time component

> NoSQL database normally only supports queries on partition and cluster keys

### Procedure of querying data

1. Getting all folders for a user
2. Displaying all emails for a specific folder
3. CRD a special emails
4. Fetching all read or unread emails

+) denormalization emails seperated by read_emails table and unread_emils table
- it's a bit more complicated and harder to maintain but it improves the read performance of these queries at scale

### Bonus point: conversation threads

- Threads are a feautre supported by many email clients
- It groups email replies with their original message
- This allows uesrs to retrieve all emails associated with one conversation
- Traditionally : a thread is implemented using algorithms such as `JWZ threading algorithm`

![KakaoTalk_Photo_2024-05-06-12-52-41 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/13629d07-4ab9-49b8-bae1-628bdbd2ec76)

**+) JWZ threading algorithm**

![11](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/3332c00a-e0ee-4cab-ba50-9ee71e9b344a)

1. Gather all messages
2. Create a hash table of `message IDs`
3. Link Messages
- user the `References` header to find the previous message in the thread and link them accordingly
4. Parent-Child Relationships

**+) Modern methods**

- Machine Laerning
- Database-Driven Approaches : Leveraging databse relationships and query optimization
- client-Side Threading
- Hybrid Approaches

## Email deliverability

- The hard part its to get emails actually delivered to a user's inbox
- If an email ends up in the spam folder, it means there is a very high chance a recipient won't read it
- Email spam is a huge issue

### Dedicated IPs

- Dedicated IP addresses for sending emails
- Email providers are less liekly to accept emails from new IP addresses that have no history

### Classify emails

- Send different categories of emails from different IP addresses

ex) avoiding sending marketing and important emails from the same servers because it might make ISPs mark all emails as promotional

### Warm-up email sending slowly

- Warm-up new IP addresses to build a reputation with providers
ex) According to Amazon Simple Email Service, it takes about 2 to 6 weeks to warm up a new IP address
- ISPs closly monitor the sending behavior from the new IP address

**+) Internet Service Provider (ISPs)**

- Managing the flow of email traffic across the internet
- They act as gatekeepers, responsible for filtering and delivering emails to their users' inboxes
- ISPs typically operate their own mail server infrastructure consists of servers

**+) ISP Validation**

- ISP validation checks typically occur when an email is received by the ISP's email servers

1. Sender Reputation : Assessing the reputation of the sender's ISP and IP address to determine if they have a history of sending spam or engaging in abusive behavior
2. Authentication : Verifying that the sender's email server is authenticated
3. Content Filtering : Analyzing the content of the email for spam-like characteristics etc
4. Delivery Policies : Enforcing delivery policies and filtering rules set by the recipient's ISP

### Ban spammers quickly

- Spammers should be banned quickly before they have a significant impact on the server's reputation

### Retries

- If the system fails to process an event, it retries
- If the maximum retry threshold is reached, the event is stored in a queue

### Feedback processing

- It is very important to set up feedback loops with ISPs so we can keep the complaint rate low and ban spam account quickly

1. `Hard bounce` : Email is rejected by an ISP because the recipient's email address is invalid
2. `Soft bounce` : It indicates an email failed to deliver due to temporary conditions, such as ISPs being too busy
3. `Complaint` : A recipient clicks the "report spam" button

![KakaoTalk_Photo_2024-05-06-12-52-41 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/54a812f1-18de-4c55-8511-848e67acaa92)

### Email authentication

![KakaoTalk_Photo_2024-05-06-12-52-41 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/51de3949-f358-4e2b-9e18-8e54d4c76abc)

- Some of the common techniques to combat phishing are 

`Sender Policy Framework (SPF)`

![12](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7db66336-5cb6-47f8-bd78-face5136add6)

- Allowing domain owners to specify which mail servers are authorized to send emails on behalf of their domain
- When the email is received, the recipient's mail server checks the SPF record of the sender's domain to verify that the sending email server is authorized to send emails on behalf of that domain

`DomainKeys Identified Kail (DKIM)`

![13](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/91e128c2-3e10-4a5e-a2b1-ea8d901c6451)

- Digitally signing email messages to verify their authenticity and integrity
- The domain owner generates a public-private key pair and publishes the public key in a DNS TXT recrod
- When sending an email, the sender's mail server signs the email using the private key and adds a DKIM signature header to the message
- Upon receiving the emails, the recipient's mail server retrieves the DKIM signature from the mssage header and uses the public key published in the DNS record to verify the signature

`Domain-based Message Authentication, Reporting and Conformance (DMARC)`

![14](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/9fa08f87-69c3-4733-9e05-779b832cd7c4)

- It builds upon SPF and DKIM to provide additional controls and reporting capabilities for email authnetication
- Domain owners can specify policies for how their emails should be handled if they fail SPF or DKIM checks
- DMARC also allows domain owners to specify where to send aggregate and forensic reports detailing email authentication results 

### Search

- Searching for emails that contains any of the entered keywords in the subject or body
- The search features in email systems has a lot more writes than reads

`Option 1 : EalsticSearch`

![8](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d457a44a-8dce-47a3-913f-785c84ebd16b)

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
