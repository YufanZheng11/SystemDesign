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
User -> Click UI -> Add View Count Service -> Database -> Return View Count Service -> UI -> User

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
| Costly for a large scale (Billions of events per day) | |
