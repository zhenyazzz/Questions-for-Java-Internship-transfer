# Module 10 — NoSQL (Mongo + CAP) — Questions 143–161

> Questions 143–161 from *Questions for Java Internship transfer.pdf* (NoSQL vs relational, data models, when to choose NoSQL, MongoDB collections/documents/relationships/indexes, sharding vs replication, eventual consistency, binary storage, TTL indexes, MongoDB transactions, fault tolerance, CAP + PACELC, CP/AP examples, CAP in MongoDB). **Related:** **Module 9** ends at **question 142**; **Module 11** (Kubernetes) starts at **question 162** in the PDF.

> Each numbered section ends with a **Middle+ answer** you can adapt on interview; bullets and tables above are for follow-up drilling.

## Module summary (middle+ — how I’d frame it on interview)

<a id="summary"></a>

**NoSQL** is an umbrella for **non-relational** (or not-only-relational) stores optimized for **different shapes of data and scale patterns**: **document** (MongoDB), **key-value** (Redis), **wide-column** (Cassandra), **graph** (Neo4j). They relax or reshape parts of the **relational contract** (strict schema, joins, single-machine SQL) in exchange for **horizontal scale**, **flexible schema**, or **specialized query models**. The wrong interview answer is “NoSQL is faster”—truth is **fit for access pattern + operational model**.

**MongoDB** stores **BSON documents** in **collections** (schema-flexible, embedded vs referenced relationships, **indexes** like B-tree style for query planning, **replica sets** for HA, **sharded clusters** for horizontal data partition). **JOINs** exist in aggregation (`$lookup`) but are not the default OLTP mental model—**denormalization** and **application-side joins** are common.

**CAP** (**Consistency, Availability, Partition tolerance**) and **PACELC** frame **distributed trade-offs** under **network partitions**. **Eventual consistency** describes systems that **converge** without requiring every read to see the latest write immediately. **MongoDB** is often discussed as **CP-ish within a replica set** (primary + majority concern) vs **AP-leaning** client read preferences—real deployments blur labels; the middle+ skill is explaining **what breaks** when a partition happens, not memorizing a vendor badge.

### Quick map (question → one line)

| # | Topic | One-line takeaway |
|---|--------|-------------------|
| **143** | NoSQL vs SQL | Different models/consistency/scale—not “always faster.” |
| **144** | NoSQL types | Document, key-value, wide-column, graph—pick by access pattern. |
| **145** | When NoSQL | Flexible schema, scale-out, specialized shapes—not “avoid SQL.” |
| **146** | Popular stores | Mongo, Redis, Cassandra, Dynamo, Neo4j, Elasticsearch (search store). |
| **147** | Mongo collection vs table | Collection = bag of documents; flexible schema vs rigid rows. |
| **148** | Mongo document | BSON document; nested structures; `_id`. |
| **149** | Relationships / JOIN | Embed vs reference; `$lookup` exists but different from SQL JOIN culture. |
| **150** | Mongo indexes | B-tree-like structures; compound, multikey, TTL; same “wrong index hurts writes.” |
| **151** | Sharding + replication (NoSQL) | Shard = horizontal split; replica = copies for HA/read scaling. |
| **152** | Replication vs sharding | Copy vs partition—orthogonal concerns. |
| **153** | Eventual consistency | Reads may lag; system converges without strong every-read-latest. |
| **154** | Binary in MongoDB | GridFS vs object storage; BSON size limits. |
| **155** | TTL indexes | Auto-expire documents after time—great for sessions/ephemeral data. |
| **156** | Mongo transactions | Multi-doc ACID since 4.0 (replica set); limitations vs single-doc atomicity. |
| **157** | Fault tolerance | Replication, failover, racks, backups, monitoring lag. |
| **158** | CAP + extension | CAP under partition; **PACELC** when no partition (latency vs consistency). |
| **159** | Why not all three CAP | Formal argument: partition forces C vs A choice (binary simplification). |
| **160** | CP vs AP examples | Mongo/Cassandra modes vs Dynamo-style tunable—examples with nuance. |
| **161** | CAP in MongoDB | Primary + majority writes; read concerns; split-brain avoided by majority. |

## Table of contents

- [Module summary (middle+)](#summary)

143. [What are NoSQL databases and how do they differ from relational databases?](#q143)
144. [What are the main types of NoSQL databases?](#q144)
145. [In what cases is it better to use NoSQL instead of SQL?](#q145)
146. [What popular NoSQL databases do you know?](#q146)
147. [What is a collection in MongoDB and how does it differ from a table in SQL?](#q147)
148. [What is a document in MongoDB?](#q148)
149. [How are relationships between documents defined in MongoDB? Is there JOIN?](#q149)
150. [How is an index implemented in MongoDB and why is it needed?](#q150)
151. [What are sharding and replication in NoSQL?](#q151)
152. [How does replication differ from sharding?](#q152)
153. [What is eventual consistency?](#q153)
154. [How to store binary data (e.g., images) in MongoDB?](#q154)
155. [What are TTL indexes in MongoDB?](#q155)
156. [Can transactions be used in MongoDB? If so, how?](#q156)
157. [How is fault tolerance ensured in NoSQL databases?](#q157)
158. [What is the CAP theorem? What is the extension of the CAP theorem?](#q158)
159. [Why is it impossible to ensure all three properties simultaneously?](#q159)
160. [Give examples of CP and AP databases.](#q160)
161. [How does the CAP theorem manifest in MongoDB?](#q161)

---

<a id="q143"></a>

## 143. What are NoSQL databases and how do they differ from relational databases?

### Question Restatement
What is **NoSQL**, and how does it differ from **relational** databases?

### 1. Relational baseline
**RDBMS**: tables, rows, **ACID** transactions across normalized tables, **SQL**, **strong schema**, **joins**, referential integrity.

### 2. NoSQL umbrella
**Schema flexibility**, **horizontal partitioning** first-class, **denormalized** models, **weaker or different consistency** defaults, **APIs** beyond SQL (though some add SQL layers).

### 3. Middle+ answer (as on interview)

Short intro (Начни с этого):

"NoSQL, which stands for 'Not Only SQL', represents a variety of database technologies designed to handle large volumes of unstructured or semi-structured data. They were built to address the limitations of traditional relational databases, particularly when it comes to scalability and flexibility."

Detailed breakdown (Затем перечисли основные отличия):

"If we compare them to relational databases, the differences usually boil down to four key areas:

Data Model & Schema: SQL databases store data in tables with rows and columns, requiring a rigid, predefined schema (schema-on-write). NoSQL databases have a dynamic schema (schema-on-read). This means you can store data in formats like JSON documents, key-value pairs, or graphs, and different records in the same collection can have entirely different structures.

Scalability: SQL databases typically scale vertically (scaling up), meaning you have to upgrade the hardware of a single server (CPU, RAM). NoSQL databases are designed to scale horizontally (scaling out), meaning you can easily handle more traffic simply by adding more servers to your database cluster.

"Historically, SQL databases scaled vertically while NoSQL scaled horizontally. However, it’s important to note that today the lines are blurred. Modern relational databases can also scale horizontally using tools like Citus for Postgres or Vitess for MySQL, and there's a whole new category of NewSQL databases like CockroachDB built specifically for distributed SQL."

Relationships (JOINs): SQL is built for complex queries and relies heavily on JOIN operations to connect normalized tables. NoSQL generally avoids complex JOINs; instead, it relies on denormalization — grouping related data together in a single record or document to speed up read operations.

Transactions (ACID vs. BASE): Relational databases strictly follow ACID properties to guarantee data validity. Traditional NoSQL systems follow the BASE model, prioritizing high availability and performance over strict immediate consistency (they use eventual consistency). However, it's worth mentioning that modern NoSQL databases, like MongoDB, now fully support ACID transactions."

---

<a id="q144"></a>

## 144. What are the main types of NoSQL databases?

### Question Restatement
Main **types** of NoSQL databases?

### 1. Four canonical families

| Type | Data unit | Typical use |
|------|------------|-------------|
| **Document** | JSON/BSON documents | Product catalogs, content, flexible aggregates |
| **Key-value** | key → blob/value | Caching, sessions, feature flags |
| **Wide-column / column-family** | rows with dynamic columns per row | Time series, feeds at huge scale |
| **Graph** | nodes + edges | Recommendations, fraud graphs |

### 2. Middle+ answer (as on interview)

Interview Answer: Main Types of NoSQL Databases
Short intro (Вступление):

"There are four primary categories of NoSQL databases. Each type is optimized for specific data structures and use cases."

Detailed breakdown (Разбор каждого типа):

1. Document Stores (Document-oriented databases)

How it works: They store data in document formats like JSON, BSON, or XML. Each document is self-describing, and documents within the same collection can have completely different structures.

Best Use Cases: Content management systems, e-commerce catalogs, user profiles. It is generally the most versatile type for modern web applications.

Popular Examples: MongoDB, Couchbase.

2. Key-Value Stores

How it works: This is the simplest and fastest type. Data is stored as a massive hash table of key-value pairs. You provide a unique key, and it returns the associated value (which can be a string, JSON, or BLOB).

Best Use Cases: Caching, session management, user preferences, and real-time recommendations.

Popular Examples: Redis, Amazon DynamoDB, Memcached.

3. Wide-Column Stores (Column-Family databases)

How it works: Instead of storing data in rows like traditional SQL, they group and store data in flexible columns. This architecture makes them incredibly good at handling massive amounts of data distributed across multiple servers.

Best Use Cases: Time-series data (like IoT sensor readings), logging, big data analytics, and applications that require massive write performance.

Popular Examples: Apache Cassandra, HBase.

4. Graph Databases

How it works: Designed specifically to store and navigate relationships. Data is stored as nodes (entities, like users or products) and edges (the relationships between them).

Best Use Cases: Social networks (finding "friends of friends"), recommendation engines, fraud detection, and network topologies.

Popular Examples: Neo4j, Amazon Neptune.
Pick type by **query shape**, not hype.

---

<a id="q145"></a>

## 145. In what cases is it better to use NoSQL instead of SQL?

### Question Restatement
**When** prefer NoSQL over SQL?

### 1. Good NoSQL signals
- **Rapidly evolving schema** or heterogeneous records per entity type.
- **Massive horizontal scale** with **partition-friendly** access paths.
- **Simple key-based** or **document-shaped** reads dominating.
- **Geo-distributed** low-latency local reads with **eventual** acceptance.

### 2. Good SQL signals
- **Complex ad-hoc joins**, reporting, **strong transactional invariants** across many entities by default.

### 3. Middle+ answer (as on interview)

High-level reasons (Общие причины):

"Generally, I would choose a NoSQL database over a relational one in a few key scenarios:

Flexible Schema is needed: When the data structure is inconsistent, unpredictable, or evolving rapidly. NoSQL allows us to iterate quickly without running complex database migrations every time a new field is added.

Massive Scale: When we anticipate huge volumes of data or extreme read/write loads that require easy horizontal scaling across many servers.

Unstructured or Semi-structured Data: When dealing with data like JSON documents, logs, or sensor data that doesn't fit neatly into tables."

Specific Use Cases (Твоя мысль про разные типы):

"However, the exact use case heavily depends on the type of NoSQL database we are talking about. For example:

If the application requires blazing-fast data retrieval for caching or session management, I'd use a Key-Value store like Redis.

If we are building a content management system, a product catalog, or a user profile service where each item might have different attributes, a Document store like MongoDB is perfect.

If we need to ingest massive amounts of time-series data, like IoT sensor logs or real-time analytics with heavy write operations, a Wide-Column store like Cassandra is the best fit.

And if the core value of the application lies in the complex relationships between data points — like a recommendation engine or a social network graph — I would definitely choose a Graph database like Neo4j."

Conclusion (Резюме-противовес):

"On the flip side, if the domain model is highly structured and requires strict ACID compliance and complex multi-table JOINs, I would stick with a traditional SQL database."

---

<a id="q146"></a>

## 146. What popular NoSQL databases do you know?

### Question Restatement
Name **popular NoSQL** databases.

### 1. Speaking list (non-exhaustive)
- **MongoDB** — document.
- **Redis** — in-memory key-value / data structures.
- **Apache Cassandra** — wide-column, partition-tuned.
- **Amazon DynamoDB** — managed key-value/document.
- **Elasticsearch / OpenSearch** — search & analytics (inverted index).
- **Neo4j** — graph.
- **Couchbase**, **Amazon DocumentDB** (Mongo-compatible), **Azure Cosmos DB** (multi-model API).

### 2. Middle+ answer (as on interview)

1. Document databases
Example: MongoDB

Store data as JSON-like documents
Good for flexible schemas and rapid development
Common use cases: user profiles, catalogs, content systems

My experience:
I’ve worked with MongoDB in backend services. I used it with focus on access patterns, proper indexing, and pagination.

Production note:
Schema flexibility is often abused → leads to inconsistent data and slow queries if indexes are not designed upfront.

2. Key-value stores
Example: Redis

Data stored as key → value
Extremely fast (in-memory)
Used for caching, sessions, rate limiting

My experience:
I used Redis for things like caching and TTL-based data (for example, shopping cart with expiration).

Production note:
You must handle cache invalidation and memory limits, otherwise you get stale data or crashes.

3. Wide-column stores
Examples: Apache Cassandra, HBase

Data stored in column families
Designed for high write throughput and horizontal scaling

Use cases:
Logging, analytics, time-series data

Production note:
You must design tables based on queries first. No ad-hoc queries like SQL.

4. Graph databases
Example: Neo4j

Store data as nodes and relationships
Efficient for connected data

Use cases:
Social networks, recommendation systems

Production note:
Bad fit if you don’t actually need graph traversal → overkill and slower than simpler models.

---

<a id="q147"></a>

## 147. What is a collection in MongoDB and how does it differ from a table in SQL?

### Question Restatement
**MongoDB collection** vs **SQL table**.

### 1. Collection
A **collection** is a **namespace** grouping **documents**—analogous to a table, but **no fixed schema** enforced by the engine per row (schema validation is **optional**).

### 2. Table
SQL table: **columns defined upfront**, each row conforms, **types** per column.

### 3. Middle+ answer (as on interview)

Short definition (Определение):

"In MongoDB, a collection is a grouping of MongoDB documents. You can think of it as the direct equivalent of a table in a relational database. Just like tables, collections exist within a database."

Key differences (Главные отличия):

"However, there are a few fundamental differences between a collection and an SQL table:

Schema Flexibility (The biggest difference): An SQL table has a strict, predefined schema. Every row must have the exact same columns, and you have to define data types in advance. A MongoDB collection, by default, does not enforce a strict schema. Documents within the very same collection can have completely different fields and data structures.

Data Format: While an SQL table stores data in flat rows and columns, a collection stores data as BSON (Binary JSON) documents. This allows collections to easily store complex, nested data structures like arrays or sub-documents directly within a single record.

Relationships: In SQL tables, relationships are built across multiple tables using Foreign Keys. In MongoDB collections, related data is often embedded directly within the same document to avoid complex JOINs, although referencing between collections is also possible."

---

<a id="q148"></a>

## 148. What is a document in MongoDB?

### Question Restatement
What is a **document** in MongoDB?

### 1. Definition
A **document** is a **BSON** record (JSON-like types + binary/date types) stored in a collection. Each document has a unique **`_id`** (ObjectId by default).

### 2. Properties
**Nested objects and arrays** are first-class; **atomic updates** at document level are natural.

### 3. Middle+ answer (as on interview)

Core Definition:

"In MongoDB, a document is the fundamental unit of data. You can think of it as the direct equivalent of a single row or record in a traditional relational database."

Format and Structure:

"However, the structure is very different from a flat SQL row. A MongoDB document stores data as a set of key-value pairs in a format called BSON, which stands for Binary JSON.

There are several key characteristics that define a MongoDB document:

Rich and Nested Data: Unlike a flat table row, a document can contain complex data structures. The value of a key can be a simple primitive (like a string or integer), but it can also be an array or even another embedded document. This allows you to store related data together in a single read operation.

Flexible Schema: Documents do not have a rigid, predefined structure. Two documents living in the exact same collection can have a completely different set of fields and data types.

The _id Field: Every document must have a unique _id field, which acts as the primary key. If a developer doesn't explicitly provide this field during insertion, MongoDB will automatically generate a unique ObjectId for it.

BSON Advantages: Because documents use BSON rather than plain JSON, they are optimized for fast machine parsing and support advanced data types that standard JSON lacks, such as native Dates, 64-bit integers, and raw binary data."

---

<a id="q149"></a>

## 149. How are relationships between documents defined in MongoDB? Is there JOIN?

### Question Restatement
**Relationships** in MongoDB—**is there JOIN**?

### 1. Modeling patterns
- **Embedding** — child objects inside parent document (1:1, 1:few).
- **Referencing** — store `_id` of another document (1:many, many:many with link collections).

### 2. JOIN equivalent
**`$lookup`** in aggregation pipeline performs a **left outer join**-like operation between collections **on the server**.

### 3. Middle+ answer (as on interview)

Direct Answer regarding JOINs (Сразу закрываем вопрос про JOIN):

"Let me address the JOIN part first. Historically, people believed MongoDB doesn't support JOINs. However, since version 3.2, MongoDB does have a JOIN operation. It is implemented in the aggregation pipeline using the $lookup operator, which essentially performs a left outer join between two collections in the same database."

How relationships are defined (Как строятся связи):

"Even though $lookup exists, heavily relying on JOINs is considered an anti-pattern in NoSQL. Instead, relationships in MongoDB are defined using two primary modeling techniques: Embedding and Referencing.

1. Embedding (Denormalization):

How it works: You store related data directly inside the main document as nested objects or arrays of sub-documents.

When to use it: It is perfect for 'one-to-few' relationships or when data is almost always accessed together. For example, storing a user's multiple addresses directly inside the User document.

Advantage: Incredible read performance, as you retrieve all related data in a single database query.

2. Referencing (Normalization):

How it works: This is similar to SQL Foreign Keys. You store the _id of a document from one collection as a field in another document.

When to use it: It is used for 'one-to-many' or 'many-to-many' relationships, especially when the related data is large, frequently updated, or accessed independently. For example, storing thousands of review_ids in a Product document instead of embedding the actual reviews, which could exceed MongoDB's 16MB document size limit."

Summary (Резюме):

"In short, while we can use $lookup for joins, the best practice in MongoDB is to design the schema based on application access patterns—using Embedding for data read together, and Referencing when data is large or needs to be accessed separately."

---

<a id="q150"></a>

## 150. How is an index implemented in MongoDB and why is it needed?

### Question Restatement
**MongoDB indexes**—implementation and **why**.

### 1. Purpose
Speed **queries** (`find`, aggregation match), enforce **uniqueness**, support **sort** without in-memory sorts, power **covered queries**.

### 2. Implementation (high level)
Default **B-tree** style indexes (WiredTiger storage engine); **compound**, **multikey** (arrays), **text**, **geospatial**, **TTL** indexes.

### 3. Middle+ answer (as on interview)

"The primary purpose of an index in MongoDB is to drastically improve the speed of read operations.Without an index, MongoDB has to perform a Collection Scan — meaning it must read every single document in a collection to find the ones that match the query. If you have millions of documents, this is extremely slow and consumes a lot of CPU and memory.With an index, MongoDB can perform an Index Scan. It quickly searches the index structure to find the specific documents, completely avoiding the need to scan the entire collection.
"2. How is it implemented? (Как он реализован):"Under the hood, MongoDB (specifically using the default WiredTiger storage engine) implements indexes using a B-Tree (Balanced Tree) data structure.Here is how it works:An index takes the value of a specific field (or fields) you choose to index and stores it in a highly organized, sorted B-Tree structure.Each node in this tree contains the sorted value of the field and a direct pointer to the physical location of the actual document on the disk.Because it's a B-Tree, searching for a value takes $O(\log N)$ time complexity instead of $O(N)$ for a full scan.It's also worth noting that MongoDB automatically creates a unique index on the _id field for every collection by default."

---

<a id="q151"></a>

## 151. What are sharding and replication in NoSQL?

### Question Restatement
**Sharding** and **replication** in NoSQL?

### 1. Replication
**Copies** of the **same data** (or partition) on multiple nodes for **HA** and **read scaling** (secondaries).

### 2. Sharding (**horizontal partitioning**)
Split **dataset across nodes** by **shard key**—each shard holds a **subset** of data.

### 3. Middle+ answer (as on interview)

"Both sharding and replication are essential techniques for managing data in distributed NoSQL databases, but they serve entirely different purposes: replication is about reliability, while sharding is about capacity and scaling."

1. Replication (Репликация):

"Replication is the process of creating and maintaining exact copies of your data across multiple servers (or nodes).

How it works: When data is written to the primary node, it is automatically synchronized to one or more secondary (replica) nodes.

Why it is needed: The main goal is High Availability (HA) and Fault Tolerance. If the primary server crashes, a replica can immediately take over, ensuring no data loss and zero downtime. It can also be used to scale read operations, as clients can read from the replicas instead of overloading the primary node."

2. Sharding (Шардирование):

"Sharding, also known as horizontal partitioning, is the process of breaking up a massive dataset into smaller, more manageable chunks called shards, and distributing them across multiple separate servers.

How it works: You define a 'shard key' (like user ID or geographical region), and the database uses this key to route specific data to a specific server. Each shard holds a unique subset of the total data.

Why it is needed: The main goal is Horizontal Scaling (Scale-out). When your dataset becomes too large to fit on a single machine's hard drive, or when the read/write operations exceed the CPU/RAM limits of one server, sharding allows you to distribute that load across a cluster of machines."

---

<a id="q152"></a>

## 152. How does replication differ from sharding?

### Question Restatement
**Replication vs sharding**.

### 1. One-liner
**Replication** = **copy** for availability/reads. **Sharding** = **split** for scale.

### 2. Middle+ answer (as on interview)

Core Difference (Главное отличие одной фразой):

"To put it simply: Replication is about copying the same data across multiple servers, while sharding is about splitting different data across multiple servers."

Detailed Comparison (Детальное сравнение):

"If I were to contrast them directly:

Primary Purpose: Replication is used for High Availability (HA), fault tolerance, and data redundancy. Sharding is used for horizontal scaling, increasing storage capacity, and distributing heavy read/write loads.

Data Content: In replication, every node holds the exact same copy of the dataset (100% of the data). In sharding, each node (shard) holds only a unique fraction of the total dataset (e.g., 33% of the data if there are 3 shards).

Problem Solved: Replication solves the problem of a server crashing (no single point of failure). Sharding solves the problem of a single server running out of disk space, RAM, or CPU power."

The "Senior" Conclusion (Как добить этот ответ и показать уровень):

"However, in a real production environment, we never choose one or the other. We use them together. For example, in a large-scale MongoDB deployment, we build a Sharded Cluster, but each individual shard is actually a Replica Set (a primary node with its own secondary copies) to ensure that if a piece of the distributed data goes down, it's still protected."
---

<a id="q153"></a>

## 153. What is eventual consistency?

### Question Restatement
What is **eventual consistency**?

### 1. Definition
If **updates stop**, all replicas **eventually** converge to the **same value**; reads **may temporarily** return stale data.

### 2. Related ideas
**Read-your-writes**, **monotonic reads**, **causal consistency**—stronger session guarantees than “bare eventual.”

### 3. Middle+ answer (as on interview)

Core Definition (Определение):

"Eventual consistency is a consistency model used in distributed systems, typically associated with NoSQL databases. It guarantees that if no new updates are made to a specific piece of data, eventually all nodes (or replicas) in the database cluster will synchronize and return the exact same, most recent value."

How it works & Why we use it (Как это работает и зачем нужно):

"In a distributed system with multiple replicas, when a write operation occurs on the primary node, it takes some time (milliseconds to seconds) for that update to propagate over the network to all the secondary nodes.

During this synchronization window, if a user reads from a secondary node that hasn't been updated yet, they might get slightly outdated or 'stale' data. We accept this temporary inconsistency because it allows the database to provide extremely high availability and low latency. The system doesn't block read operations while waiting for all nodes to sync."

Real-world Example (Жизненный пример):

"A classic example is a social media platform like Instagram or YouTube. When a popular video gets published, different users around the world might temporarily see a different number of likes or views. The system doesn't freeze the video for everyone just to strictly synchronize the counter across all servers globally. It prioritizes keeping the video available, knowing that eventually, the view count will be consistent everywhere."

---

<a id="q154"></a>

## 154. How to store binary data (e.g., images) in MongoDB?

### Question Restatement
**Storing binaries** (images) in MongoDB?

### 1. Options
- **Small binaries** — BSON **BinData** inside document if comfortably under **16 MB** total document limit.
- **GridFS** — splits large files into **`fs.files` + `fs.chunks`** default collections (chunked storage).
- **Often better** — **object storage** (S3, GCS, Azure Blob) + store **URL + metadata** in Mongo.

### 2. Middle+ answer (as on interview)

Direct Answer (Технические способы):

"In MongoDB, there are two native ways to store binary data, such as images, depending on the file size:

BSON BinData (For small files): Because a MongoDB document is limited to 16 Megabytes, you can store small binary files (like user avatars or thumbnails) directly inside a document using the BSON BinData data type.

GridFS (For large files): If the file exceeds the 16MB limit, you must use GridFS. GridFS is a specification in MongoDB that breaks a large file into smaller pieces, called 'chunks' (typically 255 KB each). It uses two separate collections to manage this: fs.chunks to store the actual binary pieces, and fs.files to store the file's metadata."

The "Senior" Architect Answer (Лучшая практика):

"However, if you ask for my architectural recommendation, I would avoid storing images directly in MongoDB altogether. >
Storing large binary files in a database is generally considered an anti-pattern because databases are optimized for querying and indexing, and database storage/RAM is expensive. The industry best practice is to upload the images to an Object Storage service like Amazon S3 (or Google Cloud Storage), and then only store the image's URL string and metadata in the MongoDB document. This is much cheaper, performs better, and integrates easily with a CDN."

---

<a id="q155"></a>

## 155. What are TTL indexes in MongoDB?

### Question Restatement
What are **TTL indexes**?

### 1. Definition
A **TTL index** is a special **single-field date index** with `expireAfterSeconds`—MongoDB background thread **deletes** expired documents periodically.

### 2. Use cases
**Sessions**, **OTP codes**, **ephemeral audit**, **cache-like** rows with natural expiry.

### 3. Middle+ answer (as on interview)

What is it? (Что это такое):

"In MongoDB, a TTL (Time-To-Live) index is a special type of single-field index that automatically removes documents from a collection after a certain amount of time or at a specific clock time."

How does it work? (Как это работает):

"You create a TTL index on a field whose value is a Date. When you create the index, you specify an expireAfterSeconds parameter.
MongoDB has a background thread that runs every 60 seconds (by default). This thread checks the TTL index, finds any documents where the current time is greater than the document's date plus the expireAfterSeconds value, and automatically deletes them."

Common Use Cases (Где это применяется):

"This feature is incredibly useful for data that naturally expires and doesn't need to be kept forever. Common use cases include:

Session Management: Automatically logging users out by deleting their session tokens after 24 hours.

Logs and Analytics: Keeping system logs or event metrics only for the last 30 days to save disk space.

Temporary Data: Storing OTPs (One-Time Passwords) or password reset tokens that should expire in 10 minutes."

---

<a id="q156"></a>

## 156. Can transactions be used in MongoDB? If so, how?

### Question Restatement
**MongoDB transactions**—possible **how**?

### 1. Facts (modern Mongo)
**Multi-document ACID transactions** on **replica sets** since **MongoDB 4.0**; **sharded cluster** transactions in **4.2+** with restrictions/latency cost.

### 2. API
`session.startTransaction()` → CRUD across collections → `commitTransaction` / `abortTransaction`. Default **single-document** updates were always **atomic**.

### 3. Middle+ answer (as on interview)

Direct Answer (Прямой ответ):

"Yes, MongoDB fully supports multi-document ACID transactions.
Historically, MongoDB only provided atomicity at the single-document level. However, starting from version 4.0 for Replica Sets, and version 4.2 for Sharded Clusters, MongoDB introduced full support for distributed transactions."

How it is used (Как это используется на практике):

"Under the hood, transactions in MongoDB are bound to a Client Session. You start a session, begin a transaction, perform multiple read and write operations across different documents or collections, and finally either commit or abort the transaction.

If you are using Java with Spring Boot, the process is very similar to relational databases. Once you configure a MongoTransactionManager, you can simply use the standard @Transactional annotation on your service methods, and Spring handles the session management for you."

The "Senior" Architect Caveat (Важное замечание про архитектуру):

"However, as a best practice, transactions should be used sparingly in MongoDB. Because they require locks and network coordination, they carry a performance cost. The primary way to ensure data consistency in MongoDB should still be through schema design — specifically, embedding related data into a single document so that updates remain naturally atomic without needing a multi-document transaction."

---

<a id="q157"></a>

## 157. How is fault tolerance ensured in NoSQL databases?

### Question Restatement
**Fault tolerance** in NoSQL?

### 1. Mechanisms
**Replication** + **automatic failover**, **rack/zone awareness**, **backups** (point-in-time), **health checks**, **read repair** (Cassandra), **majority write concerns** (Mongo), **operational monitoring** (lag, disk).

### 2. Middle+ answer (as on interview)

"Fault tolerance in NoSQL databases is primarily achieved through distributed architecture and data replication. The core philosophy is to assume that hardware will eventually fail, so the system must be designed to heal itself automatically."

Key Mechanisms (Ключевые механизмы):

"There are three main mechanisms that ensure this:

Replication (Redundancy): As we discussed earlier, data is automatically copied across multiple independent nodes. If one node crashes, the data is not lost because exact copies exist on other nodes.

Automatic Failover and Leader Election: In systems with a Primary-Secondary architecture (like MongoDB), the nodes constantly monitor each other using a mechanism called 'Heartbeats' (pinging each other every few seconds). If the secondary nodes detect that the Primary is down, they automatically hold an election to promote a new Primary. This happens in milliseconds or seconds, usually without application downtime.

Masterless Architecture (No Single Point of Failure): Some NoSQL databases, like Cassandra, use a peer-to-peer (masterless) architecture. Every node is equal and can accept reads and writes. If any node goes down, the client request is simply routed to another available node that holds a replica of that data."

---

<a id="q158"></a>

## 158. What is the CAP theorem? What is the extension of the CAP theorem?

### Question Restatement
**CAP theorem** and its **extension**.

### 1. CAP (simplified)
In the presence of a **network partition**, a distributed data store cannot be both **fully available** (every request succeeds) and **fully consistent** (all nodes see latest writes) **simultaneously**—must choose trade-offs. **Partition tolerance** is given for distributed systems.

### 2. Extension — **PACELC**
If **no partition**: trade **Latency** vs **Consistency** (`else` case). **PACELC** = **PA** partition → **C** vs **A**; **else** → **L** vs **C**.

### 3. Middle+ answer (as on interview)

1. The CAP Theorem (Теорема CAP):

"The CAP theorem, formulated by Eric Brewer, states that it is impossible for a distributed data store to simultaneously provide more than two out of the following three guarantees:

Consistency (C): Every read receives the most recent write or an error. In other words, all nodes see the exact same data at the same time.

Availability (A): Every request receives a non-error response, but without the guarantee that it contains the most recent write.

Partition Tolerance (P): The system continues to operate despite an arbitrary number of network failures (partitions) that drop or delay messages between nodes."

The Reality of CAP (Суть выбора):

"In reality, because network failures in distributed systems are inevitable, Partition Tolerance (P) is not an option, but a requirement. Therefore, when a network partition occurs, an architect must choose to prioritize either Consistency (CP) or Availability (AP)."

2. The Extension: PACELC Theorem (Расширение теоремы):

"The limitation of the CAP theorem is that it only describes how a system behaves during a network partition. But what happens when the network is functioning normally?

This is addressed by the PACELC theorem, which is an extension of CAP:

PAC: If there is a Partition, how does the system trade off Availability and Consistency?

ELC: Else (during normal operation without partitions), how does the system trade off Latency and Consistency?

For example, a database might be highly available during a partition, but during normal operations, it sacrifices immediate consistency to achieve ultra-low latency."

---

<a id="q159"></a>

## 159. Why is it impossible to ensure all three properties simultaneously?

### Question Restatement
Why **not all three** CAP properties at once?

### 1. Argument sketch
During **partition**, if two sides accept **writes** independently, they **cannot remain consistent** without **rejecting** some requests (sacrifice **availability**) or **blocking** until partition heals (also availability loss). If you refuse writes on minority side, you pick **consistency** over **availability** for those writers.

### 2. Middle+ answer (as on interview)

The Reality of Networks (Суровая реальность):

"The impossibility of having all three guarantees simultaneously comes down to the simple physical reality of distributed networks. In the real world, network failures—like dropped packets, router crashes, or severed cables—are inevitable. Therefore, a distributed system must be Partition Tolerant (P). You cannot opt-out of network failures."

The Forced Choice (Логический тупик):

"So, assuming a network partition has happened (Node A and Node B can no longer communicate), let's look at a scenario:

A client writes a new piece of data to Node A. Because the network is down, Node A cannot replicate this new data to Node B.
Right after that, another client sends a read request to Node B. At this exact millisecond, the database must make a forced choice between the remaining two properties:

Prioritize Availability (AP): Node B can process the request and return its own, older version of the data. The system is 'Available' (it didn't crash or block the user), but it is no longer 'Consistent' because the client got stale data.

Prioritize Consistency (CP): Node B can refuse to process the request, returning an error or timing out until the network is fixed. The system guarantees 'Consistency' (no one gets the wrong data), but it has sacrificed 'Availability'.

Conclusion (Вывод):

"It is a logical paradox. It is physically impossible for Node B to give the user the newest data (Consistency) AND respond successfully to the user (Availability) when the bridge of communication to Node A is broken."

---

<a id="q160"></a>

## 160. Give examples of CP and AP databases.

Short Intro:

"Since Partition Tolerance (P) is mandatory in distributed systems, NoSQL databases generally fall into two categories based on their default behavior during a network partition: CP (prioritizing Consistency) and AP (prioritizing Availability)."

1. CP Databases (Consistency + Partition Tolerance):

"CP databases guarantee that all nodes will have the exact same data, even if it means the database must become temporarily unavailable to some requests during a network failure.

MongoDB: It is a classic CP database. It uses a single Primary node for writes. If a network partition isolates the Primary, the system stops accepting writes (sacrificing Availability) until a new Primary is elected, ensuring data consistency.

HBase: Built on top of Hadoop, it is designed for strict consistency. If a region server goes down, the data it served becomes unavailable until it is recovered.

Redis: In its standard clustered mode, it typically favors consistency."

2. AP Databases (Availability + Partition Tolerance):

"AP databases guarantee that the system will always respond to user requests, even if it means returning slightly outdated (stale) data during a network failure. They rely on Eventual Consistency.

Apache Cassandra: The most famous AP database. It uses a masterless (peer-to-peer) architecture. If a network partition occurs, any available node will continue to accept read and write requests independently. Once the network is restored, the nodes will synchronize and resolve conflicts.

CouchDB: It is designed to be highly available and supports multi-master replication, meaning you can write to any node at any time.

Amazon DynamoDB: While it can be configured for strong consistency, its default and most common operating mode is AP."

The "Senior" Caveat (Важное уточнение для сеньоров):

"It's important to mention that many modern databases are highly tunable. For instance, in Cassandra, you can configure the Consistency Level per query to act more like a CP system, and in DynamoDB, you can explicitly request a 'strongly consistent read'. The CAP classification usually describes their default or architectural foundation."

---

<a id="q161"></a>

## 161. How does the CAP theorem manifest in MongoDB?

### Question Restatement
**CAP** in **MongoDB** specifically?

Short Intro:

"Since Partition Tolerance (P) is mandatory in distributed systems, NoSQL databases generally fall into two categories based on their default behavior during a network partition: CP (prioritizing Consistency) and AP (prioritizing Availability)."

1. CP Databases (Consistency + Partition Tolerance):

"CP databases guarantee that all nodes will have the exact same data, even if it means the database must become temporarily unavailable to some requests during a network failure.

MongoDB: It is a classic CP database. It uses a single Primary node for writes. If a network partition isolates the Primary, the system stops accepting writes (sacrificing Availability) until a new Primary is elected, ensuring data consistency.

HBase: Built on top of Hadoop, it is designed for strict consistency. If a region server goes down, the data it served becomes unavailable until it is recovered.

Redis: In its standard clustered mode, it typically favors consistency."

2. AP Databases (Availability + Partition Tolerance):

"AP databases guarantee that the system will always respond to user requests, even if it means returning slightly outdated (stale) data during a network failure. They rely on Eventual Consistency.

Apache Cassandra: The most famous AP database. It uses a masterless (peer-to-peer) architecture. If a network partition occurs, any available node will continue to accept read and write requests independently. Once the network is restored, the nodes will synchronize and resolve conflicts.

CouchDB: It is designed to be highly available and supports multi-master replication, meaning you can write to any node at any time.

Amazon DynamoDB: While it can be configured for strong consistency, its default and most common operating mode is AP."

The "Senior" Caveat (Важное уточнение для сеньоров):

"It's important to mention that many modern databases are highly tunable. For instance, in Cassandra, you can configure the Consistency Level per query to act more like a CP system, and in DynamoDB, you can explicitly request a 'strongly consistent read'. The CAP classification usually describes their default or architectural foundation."

---
