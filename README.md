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

## Step 3: Detailed Design

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




