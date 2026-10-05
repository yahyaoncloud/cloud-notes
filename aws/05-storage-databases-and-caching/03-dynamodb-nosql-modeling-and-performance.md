# Amazon DynamoDB: NoSQL Architecture, Data Modeling & Performance

Amazon DynamoDB is a fully managed, serverless NoSQL database designed for single-digit millisecond response times at any scale. Unlike relational databases that scale vertically and rely on SQL joins, DynamoDB scales horizontally through partition hashing. Mastering primary key design, capacity planning, and secondary indexes is a frequent interview differentiator.

---

#### Explanation & Mentorship Breakdown

##### 1. The Real-World Analogy: The Infinite Mail Sorting Facility
* **Relational Database (SQL):** Like a master librarian with a leather-bound book containing cross-referenced index cards. If you ask: *"Show me all customers who bought blue shirts in March"*, the librarian flips through 4 different books, matches receipts, joins tables, and hands you the result. It works great for 1,000 books, but if you give the librarian 100 million receipts, they collapse under the indexing overhead.
* **Amazon DynamoDB:** An automated sorting conveyor belt with 10,000 robotic bins.
  * You drop an envelope into the machine. The envelope has a barcode (**Partition Key**).
  * The machine feeds the barcode into a high-speed mathematical shredder (**Hash Function**), which immediately directs the envelope down Chute #42.
  * Inside Chute #42, envelopes are sorted in exact chronological order by date stamp (**Sort Key**).
  * It doesn't matter whether the facility has 1,000 envelopes or 10 billion envelopes; calculating the hash and dropping down Chute #42 takes exactly **4 milliseconds**.

---

##### 2. Plain-English Breakdown: What is Actually Happening?

### Primary Key Topologies

1. **Simple Primary Key (Partition Key / HASH only):**
   * Example: `UserId` (e.g. `usr-101`).
   * No two items can share the same Partition Key.
   * DynamoDB hashes the key to determine which internal physical storage partition holds the item.
2. **Composite Primary Key (Partition Key HASH + Sort Key RANGE):**
   * Example: Partition Key = `CustomerId` (`cust-500`), Sort Key = `OrderDate` (`2026-10-03#order-99`).
   * Multiple items can share the same Partition Key as long as their Sort Key is unique!
   * Items with the same Partition Key are stored physically adjacent on the same storage node, sorted by the Sort Key.
   * Enables powerful range queries: `begins_with()`, `between()`, `>`, `<`.

---

### Internal Partition Limits (The Rule of 10GB / 1000 WCU / 3000 RCU)

DynamoDB automatically allocates physical partitions behind the scenes. Each internal partition has hard physical caps:
* **Max Storage per Partition:** **10 GB**
* **Max Write Throughput per Partition:** **1,000 Write Capacity Units (WCU)**
* **Max Read Throughput per Partition:** **3,000 Read Capacity Units (RCU)**

*The Hot Partition Anti-Pattern:* If you choose a low-cardinality Partition Key like `Gender` (`Male` vs `Female`) or `Status` (`Active`), all 100,000 requests hit the same physical partition node, exhausting the 1,000 WCU limit and throwing **`ProvisionedThroughputExceededException`**! Always use high-cardinality keys (like UUIDs, timestamps, customer IDs).

---

### Capacity Modes & Mathematics

```
+------------------------------------+------------------------------------+
| ON-DEMAND CAPACITY MODE            | PROVISIONED CAPACITY MODE          |
+------------------------------------+------------------------------------+
| Pay-per-request (No capacity plans)| You specify fixed WCU and RCU      |
| Instantly accommodates traffic     | Auto-scales based on target metrics|
| spikes without throttling          | (e.g. 70% utilization)             |
| Best for: Spiky, unpredictable, or | Best for: Steady, predictable,     |
| low-volume batch traffic           | continuous high-volume workloads   |
| Cost: ~5x more expensive per req   | Cost: Up to 70% cheaper if steady  |
+------------------------------------+------------------------------------+
```

#### Capacity Unit Calculations (Interview Essential!):
* **Write Capacity Unit (WCU):**
  * 1 WCU = **1 Write per second** for an item up to **1 KB** in size.
  * If item is 2.5 KB $\rightarrow$ Rounds up to 3 KB $\rightarrow$ Consumes **3 WCUs**.
* **Read Capacity Unit (RCU):**
  * 1 RCU = **1 Strongly Consistent Read per second** for an item up to **4 KB**.
  * 1 RCU = **2 Eventually Consistent Reads per second** for an item up to 4 KB (Eventually consistent is 50% cheaper!).
  * Transactional operations (`TransactWriteItems`, `TransactGetItems`) consume **2x** base capacity.

---

### Secondary Indexes: GSI vs. LSI

```
+------------------------------------+------------------------------------+
| GLOBAL SECONDARY INDEX (GSI)       | LOCAL SECONDARY INDEX (LSI)        |
+------------------------------------+------------------------------------+
| Partition Key: CAN BE DIFFERENT    | Partition Key: MUST BE SAME AS BASE|
| Sort Key: CAN BE DIFFERENT         | Sort Key: MUST BE DIFFERENT        |
| Creation: Can be added ANYTIME     | Creation: MUST BE DEFINED AT TABLE |
| (even on live production tables)   | CREATION (Cannot be added later!)  |
| Consistency: EVENTUALLY CONSISTENT | Consistency: Strongly OR Eventually|
| Capacity: Has its OWN WCU/RCU      | Capacity: Shares base table WCU/RCU|
| Storage Limit: UNLIMITED           | Storage Limit: 10 GB per partition |
+------------------------------------+------------------------------------+
```

---

### DynamoDB Streams & Time-To-Live (TTL)

* **DynamoDB Streams:** An ordered Change Data Capture (CDC) stream of item-level modifications (Insert, Modify, Delete) stored for 24 hours. Ideal for triggering AWS Lambda functions (e.g., sending a welcome email when a user item is created).
* **Time-To-Live (TTL):** Define a timestamp attribute (Epoch seconds). When the timestamp passes, DynamoDB automatically deletes the item in the background **at zero cost and with zero consumption of read or write capacity units!** Ideal for session stores and temporary cache entries.

---

##### 3. Visual Flow Diagram: Event-Driven Processing with DynamoDB Streams

```
[ Web Application ] === PutItem ===> [ DynamoDB Table ]
                                             |
                                             v (CDC Event)
                                  [ DynamoDB Stream ]
                                             |
                                             v (Event-Driven Trigger)
                                    [ AWS Lambda ]
                                             |
                               +-------------+-------------+
                               |                           |
                               v                           v
                     [ Amazon SQS Queue ]        [ Amazon OpenSearch ]
                     (Async Worker Job)          (Full-text Search Index)
```

---

##### 4. The "Aha!" Moment / Why People Get Confused

* **Query vs. Scan (The Cardinal Rule of DynamoDB):**
  * **`Query`:** Ultra-fast, highly efficient. Searches based on Partition Key equality (`PK = :val`) and optional Sort Key conditions. Reads only the matching items from the specific partition.
  * **`Scan`:** The ultimate anti-pattern! Scans every single row across the entire multi-terabyte table sequentially, consuming massive RCUs and bankrupting your AWS account. Production code should virtually **never** use `Scan`.

---

##### 5. Quick Hands-On Check

Execute a fast, targeted Query against a DynamoDB table using the CLI:

```bash
# Query items for a specific partition key using strongly consistent reads
aws dynamodb query \
    --table-name UsersTable \
    --key-condition-expression "UserId = :uid" \
    --expression-attribute-values '{":uid":{"S":"usr-101"}}' \
    --consistent-read \
    --region us-east-1
```
