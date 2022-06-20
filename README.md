# System Design

## Step 1: Collect Requirements

System Design usually comes with a very vague question, we need to clarify the question and then decide to how we will design the system.

For example, we have a problem to **Analyses data in real-time**, below are the questions we need to clarify:
- What does data analysis mean ?
- Who sends us data ?
- Who uses the result of this data ?
- What real-time really means ?

For any design question, there might be multiple technical solutions, reason to clarify the requirements is we can have a better decision of techniques.

For example, count YouTube views can be resolved by many technology stacks, but what technology we choose really depends on the requirements
- SQL Database: MySQL, PostgreSQL
- NoSQL Database: Cassandra, DynamoDB
- Cache: Redis
- Stream Processing: Kafka + Spark
- Cloud native stream processing: Kinesis
- Batch processing: Hadoop MapReduce

For an open-ended design task, questions we need to ask can be categorized into **4 categories**:
- Users / Customers
    - Who will use the system ?
    - How would the system be used ?
- Scale (Read and write)
    - How many read queries per second ?
    - How much data is queried per request ?
    - How many video views are processed per second ?
    - Can there be spikes in traffic ?
- Performance
    - What is expected write-to-read data delay ?
    - What is expected p99 latency for read queries ?
- Cost
    - Should the design minimize the cost of development ?
    - Should teh design minimize the cost of maintenance ?

### Functional Requirements
- System behavior
    - A set of operations the system will support
Example
- View video
- Return number of views of video
Questions
- Do we need to extend this ?
    - If yes, a generic API interface can be introduced

### Non-Functional Requirements
- Scalability
    - Tens of thousands of video views per second
- High Performance
    - Few tens of milliseconds to return total views count for a video
- High Availability
    - Survives hardware / network failures, no single point of failure
- Consistency
- Cost
    - Hardware, development, maintenance

## Step 2: High level design

Write down in the whiteboard:
- Sentences describing the **Functional** requirements
    - Increase view count when user click
    - Return latest view count when open video
- Write down the non functional requirements, but we can come to the details later
    - Scalability
    - Performance
    - Availability
- A simplest diagram showing the workflow
    - User -> Click UI -> Add View Count Service -> Database -> Return View Count Service -> UI -> User

## Step 3: Detailed Design - Data Storage

### Data Model - How do we store the data?
In the count view example, we have **2** ways of saving the view data

#### Option 1 - Save individual event
| Video | Timestamp | ... |
| ---- | ---- | ---- |
| A | 2020-01-01 14:12:11 | ... |
| B | 2020-01-01 14:12:15 | ... |

#### Option 2 - Save the aggregated date
| Video | Timestamp | Count |
| ---- | ---- | ---- |
| A | 2020-01-01 14:12:11 | 2 |
| B | 2020-01-01 14:12:15 | 3 |

#### Pro & Cons
| Option 1 | Option 2 |
| ---- | ---- |
| Fast Write | Fast Read |
| Can slice and dice data when need | Data is ready for decision making |
| Can recalculate numbers if needed | |
| - | - |
| Slow Read | Can query only when the way data was aggregated |
| Costly for a large scale (Billions of events per day) | Require data aggregation pipeline |
| | Hard or impossible to fix the bug |

Decision making
- Store the raw event ?
- Aggregate the data on the fly ?

Metrics to take into consideration
- Expected data delay
    - Time between the event happened & when it's proceed
        1. If no more than several minutes - we must aggregate the data on the fly
        2. If several hours is ok - we can store the raw events and process them in the background

- **Stream Data Processing** >>> Choice 1
- **Batch Data Processing** >>> Choice 2

#### Store raw events in real time
- Because the raw events are many
    - We will store events for several days or week only
    - Purge old data
- Calculate the numbers in real time 
    - so the statistics is available for users right away
- By saving both raw events & aggregated data
    - **Fast Read**
    - **Ability to aggregate data differently**
    - **Re-calculate statistics if needed**
- Drawback
    - The system will become expensive & complex

#### What database will we choose ?
- Both **SQL** and **NoSQL** database can scale and perform well
- We need to consider the non functional requirements
    - We need to evaluate the database against these requirements

| Questions | 
| ---- |
| How to scale **Read** |
| How to scale **Write** |
| How to make both **writes and reads fast** |
| How **not to lose data** |
| How to achieve **strong consistency**? Trade-offs ? |
| How to **recover data** in case of an outage |
| How to ensure **data security** |
| How to make it **extensible** |
| Where to run (**cloud vs on-premises** data center) ? |
| How much does it **cost** |

#### SQL Database
- We can first store data in a single machine
- If a single machine is not enough, we can split the data into multiple machines
    - The process is called **sharding** or **horizontal partitioning**
    - Each shard stores a subset of data
- As multiple machines exist, service needs to know
    - How many machine exist ?
    - Which one to pick to store & retrieve data ?

We can have each service connects to the database and query / process.
- Processing Service -> Store data -> Database
- Query Service <- Retrieve data <- Database

A better option is to introduce a light proxy server **Cluster Proxy**
- Processing Service -> **Cluster Proxy** -> Store data -> Database
- Query Service <- **Cluster Proxy** <- Retrieve data <- Database
- Services only talk to Cluster Proxy only
    - Service don't need to know each nodes & database machine any more
- Proxy server talks to the database machines
    - Proxy server knows all database machines
    - Proxy server route requests to target machines / shards
    - If a service dies, proxy machine knows
    - If a new node is added to the cluster, proxy server knows
- To make the proxy service know status of cluster
    - We introduce the **configuration service** (i.e. **Zookeeper**)
    - Instead of calling the database directly, we can apply **ShardProxy**
- Shard Proxy can help
    - cache query results
    - monitor database instance health
    - publish metrics
    - terminate queries that take too long to return
- How to ensure the data isn't lost?
    - Replicate the data
    - Master shard - follower shard
    - Write goes to the master shard
    - Read can go from master & follower shards
    - Put the master data / follower data in different data center

<img src="./SQL Database" />

#### NoSQL Database (Cassandra)

How Cassandra nodes works?
- Instead of having leaders & followers, each node are equal
- No longer needs configuration service to monitor health of each shard
- Allow shards talk to each other and exchange information about their state
- Every second, shard exchange information with a few other shards (no more than 3)
- Quick enough, state information about every node propagates throughout the cluster -- gossip protocol
- SQL needs to have a Cluster Proxy to know location of each shards
- Cassandra nodes know each other 
    - request no need to call a specific component for routing requests

How Cassandra nodes process the request
- Process service send a request to node-4
    - We can use a simple round robin algo to choose the initial node
    - or we can just choose the node which is **closet** to the client in terms of **network distance**
- The chosen node is named **coordinator node**, this node needs to decide
    - which node to store data for the request
    - we can use **consistent hashing** algo to pick the node
    - make a call to the node which should store the data and wait for the response
    - it will replicate the data at same time
    - but wait for all replicates done is slow, it will consider success if 2 replications done -- **quorum writes**
- Concept of **quorum read**
    - When the query service retrieves data
    - coordinator will **initialize several read requests in parallel**
    - in theory, the coordinator node may get **different responses** from replica nodes
        - because some nodes could have been unavailable when the write request happened
        - let's say one node has stale data and other 2 nodes have up-to-date data
        - read quorum defines min number of nodes that have to agree the response
        - Cassandra uses the version number to determine the staleness of data

<img src="./NoSQL Database" />

#### Availability VS Consistency
- Availability > Consistency
- Show stale data rather than no data at all
- Replication of data is slow
    - Synchronous is slow
    - We usually replicate the data asynchronously
- Some nodes might have stale data **temporally**
    - But after data propagate in whole nodes
    - the data will eventually consistent -- this is what call **eventual consistency**

### How to store data

#### SQL database
Data is stored in table format, we apply normalization in relational database

#### NoSQL database
- No more normalization, we store everything required by the report together
- Instead of adding rows, we keep adding columns every next hour

<img src="./SQL vs NoSQL Store" />

#### 4 types of NoSQL Database
- column
    - Cassandra
    - HBase
- document
    - NoSQL
- key-value
- graph

Cassandra is
- Fault-tolerant
- Scalable (both read-write throughput can scale upon added machine)
- Support multi data center replications
- Works well with time series data

MongoDB
- leader-based replication

HBase
- master-based replication

## Step 4: Detailed Design - Data Processing

### Processing Service
- How to **scale** ?
- How to achieve **high throughput** ?
- How to **not lose data** when processing node crashes ?
- What to do when **database is unavailable** or slow ?

How to make data processing **scalable, reliable and fast** ?
- Scalable = Partitioning
- Reliable = Replication and check-pointing
- Fast = In-memory

#### Data aggregation
**Should we pre-aggregate data in the processing service ?**
- Choice 1:
    - For every incoming event, we increase the count by 1 in the database
- Choice 2:
    - We store the temporally count in the processing service memory
    - And sync the count to database every period of time

Choice 2 is definitely better solution if we want to scale

We can better introduce a message queue if we can ensure no data is lost.
- So user click event will be sent to the message queue
- The processing service will pull data from the message queue
- so even the processing service crash, the message queue data persist and can re-process them
- We use the check point to tell where the data is consumed

Processing service
- Read count events from partition / shard
- Count event in memory
- Flush the counted values to the database periodically

**Dead letter queue**
- When flush data into database, in case of failure, save the data into this queue
- A separate thread read data from this queue and store the data into the database
- This can store data in the disk of processing service

**Recovery memory data from failure**
- Load events again from checkpoint
- Or periodically store the states in a durable storage

## Step 5: Detailed Design - Ingestion path components

User -> API Gateway -> Load Balancer -> Partition Services -> Partitions -> Processing Service -> Databases

| Partition Service Client | Load Balancer | Partition Service and Partitions |
| ---- | ---- | ---- |
| Blocking vs non-blocking I/O | Hardware vs software load balancing | Partition strategy |
| Buffering and batching | Networking protocols | Service discovery |
| Timeout | Load balancing algorithm | Replication |
| Retries | DNS | Message format |
| Exponential backoff and jitter | Health Check | |
| Circuit breaker | High Availability | |


### Blocking vs Non-Blocking
When client make a request to the server, it builds a connection via socket
- Blocking service will create a thread per connection
    - Modern multi-core machines can handle hundreds of concurrent connections each
    - But when more requests come, machines may go a death spiral, and whole cluster may die
    - **Rate limiter** is important at this stage
    - Blocking system is easier to debug issue, when a request comes, we can easily trace the process into each thread stack
- Non-blocking service will pill up request from queue
    - No more thread created
    - More efficient as threads are more expensive than the picking up events
    - High throughput
    - Problem with non-blocking system is --- hard to debug

### Buffering and batching 
- We should somehow combine the events (buffering) and send them together (batching)
- Sending them one by one is not efficient
- Drawbacks: increase complexity

### Timeouts
- Connection timeouts
    - usually tens of milliseconds
- Request timeouts
    - we can use 1% percentile timeout as the request timeout
- Retry when timeouts
    - to avoid retry storm --- exponential backoff and jitter

### Exponential backoff and jitter
- Retry with a different time intervals so avoid retry storm

### Circuit breaker 
- If request failure rate exceed threshold - stop retrying
- If success rate becomes higher - re-start retrying
- Make system hard to test
- And hard to set the proper error threshold

### Hardware vs software load balancing 
- Hardware load balancers are network devices we buy from known organization
    - Powerful machines with many CPUs, memory and optimized to handle very high throughput: millions of requests / second
- Software load balancer is only software we install in the hardware we choose
    - No need big machine, many load balancer are open source

Example of load balancer:
- ELB from AWS

### Networking protocols
- TCP load balancer
    - Simply forward network package without inspecting the content
    - Super fast, can handle 1m+ requests / second
- Http load balancer
    - LB gets HTTP request from a client
    - Establish a connection to the server
    - Send request to the server
    - It can look inside the content of msg and make a choice upon content
        - on cookie or header etc.

### Load balancing algorithm
- Round robin algo distribute the requests **in order** across the list of servers
- Least connection algorithms distribute requests to the server with **lowest number of active connections**
- Least response time algorithms distribute requests to the server with **fastest response time**
- Hash-based algorithms distribute requests based on a key we define, such as client IP or url

### Load balancer questions
- How does client knows load balancer ?
- How does load balancer know about services machines ?
- How does load balancer guarantees high availability ?

### DNS - Domain Name System
- DNS is like a phone book in the internet
    - It maintains domain names & translate them into IP address
- We register the services in DNS, specify the domain name
    - i.e. xxxservice.domain.com
    - associate the domain with the IP address of the load balancer device
- When client hits the domain name,
    - a request is made to the load balancer
- **how does load balance knows the services machine**
    - we need to explicitly tell the load balancer the IP address of each machine
    - Hardware / software LB provides API to register/unregister the servers

### LB health check
- LB needs to know which server is available & un-available
- pings the server periodically 

### LB High Availability
- Primary & secondary nodes
    - Primary : accepts connections and server requests
    - Secondary : monitor the primary
        - If the primary one fails to build connections
        - Secondary takes over
    - They lives in different data center

### Partition services
- Get messages and store them on **disk** in the form of append-only log file

### Partition Strategy
- Calculate the hash based on some content key - choose machine based on hash
    - hot partition

### Service discovery 
- Server side discovery (Load balancer)
- Client side discovery
    - Every instance registers itself in some common place, named service registry
    - service registry provides health check and high availability
    - i.e. Zookeeper

### Replication, When & how to choose replication strategy
- Single leader replication
    - SQL database
    - Each partition will have a leader and several followers
    - Only write and read from leader only
    - When a leader is alive, all followers copy events from the leader
    - When a leader dies, we choose a new leader from followers
    - Leader keeps track of followers, check if they are alive
        - If a follower dies, remove from the list
- Leaderless replication
    - Cassandra
- Multi leader replication
    - Mostly used when replicate data across different data centers

### Message formats
- Text format: XML, JSON
    - human readable
- Binary format: Thrift, Protocol, Buffers and Avro
    - more compact, and fast

## Step 6: Detailed Design - Data Retrieval Path

### Rollup solution for large time series data
- Granularity
    - Recent time records, we save per second/minutes
    - Mid recent time records, we save per hour
    - Long time ago records, save per day/week/month
- Old data do not need to be stored in database
    - They can be stored in object storage
- Hot storage vs cold storage
    - Hot storage - frequent used data, access fast
    - Cold storage - less frequent used data

### distributed cache
- Store the query results in distributed cache to improve performance

## Step 7: Technology stack
Client side
- Netty: high performance non-blocking IO framework for developing network applications
- Netflix Hystrix
- Polly
Load balancing
- NetScaler - hardware load balancer
- NGINX - Software load balancer
- AWS ELB - Cloud load balancer
Messaging systems
- Apache Kafka
- Amazon Kinesis (Kafka public cloud counterpart)
Data Processing
- Apache Spark
- Apache Flink
- AWS Kinesis Data Analytics
Storage
- Apache Cassandra
- Apache HBase
- InfluxDB (Optimized for time series data)
- Large data can be stored in Hadoop, or AWS Redshift
- Cold data can be used in AWS S3
Other technologies
- Vitess
    - Manage large clusters of MySQL instances
- Redis
    - In memory cache
- RabbitMQ / AWS SQS
    - Dead-letter queue (temporally undelivered messages)
- RocksDB
    - A high performance embeded database (i.e. for enrichment data)
- Zookeeper or Netflix Eureka
    - Service Registry
- Monitoring the services
    - AWS CloudWatch
    - ElasticSearch, Logstash, Kibana
