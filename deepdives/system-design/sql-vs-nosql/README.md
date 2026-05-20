# SQL vs NoSQL: A System Design Deep Dive (Internals, Tradeoffs, and How to Choose)

## Table of Contents
1. [The Big Picture](#the-big-picture)
2. [What “SQL” and “NoSQL” Really Mean](#what-sql-and-nosql-really-mean)
3. [Relational (SQL) Databases](#relational-sql-databases)
   1. [Core Ideas: Relations, Schema, Constraints](#core-ideas-relations-schema-constraints)
   2. [Transactions and ACID (With Real Internals)](#transactions-and-acid-with-real-internals)
   3. [Indexes and Storage Engines](#indexes-and-storage-engines)
   4. [Query Execution: Optimizer, Plans, and Joins](#query-execution-optimizer-plans-and-joins)
   5. [Concurrency Control: Locks vs MVCC](#concurrency-control-locks-vs-mvcc)
   6. [Replication, High Availability, and Failover](#replication-high-availability-and-failover)
   7. [Scaling SQL: Vertical, Sharding, and Distributed SQL](#scaling-sql-vertical-sharding-and-distributed-sql)
4. [NoSQL Databases](#nosql-databases)
   1. [Why NoSQL Exists](#why-nosql-exists)
   2. [NoSQL Taxonomy](#nosql-taxonomy)
   3. [Key-Value Stores (Dynamo-style)](#key-value-stores-dynamo-style)
   4. [Document Stores (MongoDB-style)](#document-stores-mongodb-style)
   5. [Wide-Column Stores (Cassandra/HBase-style)](#wide-column-stores-cassandrahbase-style)
   6. [Graph Databases](#graph-databases)
   7. [Search/Index Engines](#searchindex-engines)
   8. [Time-Series Databases](#time-series-databases)
   9. [Common NoSQL Storage Internals: LSM Trees](#common-nosql-storage-internals-lsm-trees)
5. [Consistency, CAP, and the Real World (PACELC)](#consistency-cap-and-the-real-world-pacelc)
6. [Data Modeling Tradeoffs](#data-modeling-tradeoffs)
7. [Operational Tradeoffs](#operational-tradeoffs)
8. [Decision Framework: How to Choose](#decision-framework-how-to-choose)
9. [Practical Examples](#practical-examples)
10. [Cheat Sheet Summary](#cheat-sheet-summary)

---

## The Big Picture

“SQL vs NoSQL” is less about syntax and more about **data model + consistency guarantees + scaling strategy + operational constraints**. The distinction represents a fundamental philosophical difference in how we think about organizing and retrieving data at scale. This is not merely a technical choice—it reflects deep tradeoffs in system design that have cascading effects on application architecture, operational complexity, and business capabilities.

At system design level, you usually choose a database based on:

- **Workload shape**: read-heavy vs write-heavy, OLTP vs analytics, high fanout queries, time-series ingestion.

  The shape of your workload determines the optimal storage layout. For example:
  - **OLTP workloads** require low latency point operations and strong transactional guarantees
  - **Analytical workloads** benefit from columnar storage and batch-oriented processing
  - **High fanout queries** (like fetching a user's social feed with posts from hundreds of friends) push systems toward denormalized, read-optimized structures because joining across hundreds of tables at runtime would be prohibitively expensive
  - **Time-series ingestion** workloads benefit from append-optimized storage that can handle millions of writes per second while providing efficient time-window queries
- **Correctness requirements**: do you need strict invariants (money never disappears), or are you okay with eventual convergence?

  This is perhaps the most critical decision point:
  - **Financial systems, inventory management**, and any domain where invariants must hold at all times typically require ACID semantics
  - **Social media feeds, recommendations**, and other best-effort systems can tolerate eventual consistency where replicas may temporarily diverge but eventually converge

  The key insight: "correctness" is application-dependent—some applications require that every read returns the most recent write, while others can tolerate stale reads as long as consistency is eventually achieved.
- **Latency and availability goals**: single-region vs multi-region, partition tolerance expectations.

  If you need to serve users globally with low latency, you face the CAP/PACELC tradeoff directly:
  - Strong consistency across regions requires coordination that adds latency
  - Availability during partitions means accepting weaker consistency models

  Your geographic deployment strategy fundamentally constrains your database choices. For example:
  - A **global e-commerce platform** might choose a multi-master setup with conflict resolution to ensure availability during network partitions
  - A **banking system** might choose to accept higher latency for strong consistency
- **Data access patterns**: many-to-many joins and ad-hoc queries vs known fixed queries.

  Relational databases excel when you don't know all queries upfront—analysts can write ad-hoc SQL without schema changes. NoSQL systems typically require you to design your data model around known access patterns, trading flexibility for performance at scale.

  The difference is fundamental:
  - **Relational databases** are general-purpose tools that can handle arbitrary queries
  - **NoSQL databases** are specialized tools optimized for specific access patterns

  This is why NoSQL systems often require "query-driven data modeling"—you must design your schema based on how you'll query it.
- **Operational reality**: schema migrations, backups, debugging incidents, cost, team expertise.

  The best database in theory becomes the worst in practice if your team cannot operate it effectively. Consider your operational maturity:
  - Do you have expertise in distributed systems troubleshooting?
  - Can you handle complex resharding operations?
  - What's your tolerance for operational complexity?

  A distributed NoSQL system might solve your scaling problems but introduce new operational challenges like hotspot detection, compaction tuning, and consistent hashing rebalancing that your team may not be prepared to handle.

A useful mental model:

- **SQL (relational)** systems optimize for:
  - strong constraints
  - expressive ad-hoc querying
  - transactional correctness
  - consistency that’s easier to reason about

- **NoSQL** systems optimize for:
  - horizontally scalable writes and storage
  - flexible schema or denormalized models
  - predictable performance at huge scale (if queries match the model)
  - availability under failure/partition (often with weaker guarantees)

---

## What “SQL” and “NoSQL” Really Mean

### SQL
“SQL database” typically means a **relational database management system (RDBMS)**, built on the theoretical foundation of relational algebra and tuple calculus. The relational model, introduced by E.F. Codd in 1970, represents data as relations (mathematical sets of tuples) and provides a formal basis for query optimization and data integrity.

- **Relational data model**: tables (relations), rows (tuples), columns (attributes).

  This mathematical foundation enables powerful query optimization because the optimizer can transform queries using algebraic rules. For example:
  - The optimizer knows that join operations are commutative and associative, allowing it to reorder joins for efficiency
  - The relational model provides a clear separation between the logical schema (how users see data) and physical storage (how data is actually stored on disk)

- **Schema**: columns and types are defined upfront.

  This schema-on-write approach provides several benefits:
  - Type safety prevents many classes of bugs
  - The optimizer can use type information for optimization
  - The schema serves as documentation

  However, it also introduces rigidity—schema changes require migrations that can be operationally complex for large, busy systems.

- **Constraints**: primary keys, foreign keys, uniqueness, check constraints.

  These constraints move business logic into the database, ensuring invariants are maintained regardless of which application accesses the data. For example:
  - Foreign key constraints prevent referential integrity violations that would leave orphaned records
  - The database can enforce these constraints efficiently because it has direct access to the data structures

- **Transactions**: multi-statement operations that are atomic and isolated.

  Transactions are the fundamental unit of consistency in relational databases. They provide a way to group multiple operations into a single logical unit that either completely succeeds or completely fails. This is essential for maintaining invariants across multiple tables—for example, transferring money between accounts requires debiting one account and crediting another as an atomic operation.

- **Declarative query language**: you specify *what* you want; the database figures out *how*.

  SQL is declarative rather than imperative:
  - Instead of writing algorithms to retrieve data, you describe the result set you want
  - The query optimizer then determines the most efficient execution plan
  - This abstraction is powerful because it allows the database to adapt its execution strategy based on data statistics, indexes, and system conditions without requiring application code changes

Examples: PostgreSQL, MySQL, SQL Server, Oracle.

### NoSQL
“NoSQL” is not one thing. It’s an umbrella term for databases that are **not primarily relational** and often relax some relational assumptions.

Common traits (not universal):

- Different data models (document, key-value, wide-column, graph):

  Each NoSQL category optimizes for specific access patterns:
  - Document stores model data as hierarchical objects
  - Key-value stores provide simple O(1) access
  - Wide-column stores optimize for range scans within partitions
  - Graph stores specialize in relationship traversals

  These models trade the generality of the relational model for performance in specific domains.
- Denormalization-first designs: NoSQL typically encourages denormalization—storing related data together rather than normalizing across tables.

  This approach has tradeoffs:
  - Reduces the need for joins (which are expensive in distributed systems)
  - Increases write complexity because updates must touch multiple denormalized copies

  The philosophy is to optimize for read patterns since reads typically outnumber writes in many web applications.
- Horizontal scaling built-in: NoSQL systems were designed from the ground up to run on clusters of commodity machines.

  They:
  - Partition data automatically across nodes
  - Provide mechanisms for adding capacity without manual sharding

  This contrasts with traditional RDBMS where scaling out often requires complex application-level sharding.
- "Tunable" consistency: Many NoSQL systems allow you to configure consistency levels on a per-operation basis.

  For example:
  - You might choose strong consistency for critical operations
  - Eventual consistency for less critical reads

  This tunability lets you make explicit tradeoffs between consistency, latency, and availability based on your application's needs.
- Fewer cross-record transactional guarantees (varies widely):

  Early NoSQL systems often abandoned multi-record transactions entirely to achieve scalability. However, modern NoSQL databases have added transactional capabilities:
  - MongoDB supports multi-document ACID transactions
  - DynamoDB supports transactions across multiple items

  The key difference is that these transactions are often scoped differently or have different performance characteristics than RDBMS transactions.

Examples: DynamoDB (key-value), MongoDB (document), Cassandra (wide-column), Neo4j (graph), Elasticsearch/OpenSearch (search).

Important nuance: many modern systems blur the line:

- PostgreSQL has JSONB and can behave like a document store:

  PostgreSQL's JSONB type provides:
  - Efficient storage and indexing of JSON documents
  - Query operators that can navigate nested structures

  This allows PostgreSQL to serve use cases that might traditionally require a document database while still providing relational features when needed.
- MongoDB added multi-document transactions:

  MongoDB 4.0 introduced ACID transactions across multiple documents and collections, addressing a major criticism of early NoSQL systems. However:
  - These transactions have performance implications
  - They are not intended to be used as heavily as RDBMS transactions
- "Distributed SQL" systems (CockroachDB, YugabyteDB, Spanner) look like SQL but are distributed/consensus-based:

  These systems provide:
  - A SQL interface and relational semantics
  - Distributed consensus protocols (Raft, Paxos) for replication and coordination

  They aim to combine the best of both worlds: the familiarity and query power of SQL with the horizontal scalability and fault tolerance of NoSQL systems. However:
  - They introduce latency overhead due to consensus coordination
  - They are more complex operationally than single-node RDBMS

---

## Relational (SQL) Databases

### Core Ideas: Relations, Schema, Constraints

#### Schema and types
A schema is a contract: it defines the structure, types, and constraints of your data at the database layer rather than relying solely on application validation. This schema-on-write approach provides several important benefits:

- It enforces **types and invariants** at the data layer, preventing invalid data from ever being stored. For example, attempting to insert a string into an integer column will fail immediately rather than corrupting your data. Type safety catches entire classes of bugs at the database boundary.

- It enables the optimizer to make better choices because the optimizer knows the data types, constraints, and statistical distributions of columns. This information allows the optimizer to choose efficient execution plans, estimate costs accurately, and apply type-specific optimizations.

- It reduces the chance of “garbage data” due to application bugs because the database acts as a final validation layer. Even if application code has bugs, the schema prevents certain classes of invalid states from ever reaching persistent storage.

Tradeoff: schema changes require migrations and operational care. Adding a column, changing a type, or adding a constraint requires a migration that must be carefully planned for large, busy production systems. These migrations can lock tables, require significant time for data backfills, and need rollback strategies. The rigidity that provides safety also introduces operational overhead.

#### Normalization and joins
Relational modeling encourages **normalization**: the process of organizing data to minimize redundancy and dependency. Normalization typically involves decomposing tables into smaller, well-structured relations and defining relationships between them through foreign keys. The goal is to ensure that each piece of data is stored in exactly one place.

- Store each fact once: In a normalized schema, a given fact appears in only one row of one table. For example, a customer's address is stored in the customers table, not duplicated in every order row. This eliminates redundancy and reduces the risk of inconsistencies where the same fact has different values in different places.

- Represent relationships through keys: Foreign keys establish relationships between tables without duplicating data. An order table references a customer_id rather than storing customer details directly. This preserves the relational model's mathematical foundation while enabling efficient joins when needed.

- Avoid update anomalies: Normalization prevents several classes of data integrity problems. Update anomalies occur when you must update the same fact in multiple places; if one update fails, you end up with inconsistent data. Insertion anomalies occur when you cannot add a fact without also adding unrelated facts. Deletion anomalies occur when deleting a fact inadvertently removes other data. Normalization eliminates these by ensuring each fact has a single, canonical location.

This pushes complexity into queries (joins), but helps correctness. Instead of reading data from a single denormalized table, queries must join multiple tables to reconstruct the complete picture. The database optimizer handles the join execution, but the query writer must understand the schema and write appropriate join conditions.

Tradeoff: joins can be expensive at massive scale, especially across partitions. In distributed databases, joins that require data from multiple partitions or nodes require network round trips and can become performance bottlenecks. This is why many large-scale systems denormalize specific access patterns even when using relational databases—the cost of joins at scale can outweigh the benefits of normalization for read-heavy workloads.

#### Constraints
Relational constraints move business rules into the database: instead of relying solely on application code to enforce invariants, the database itself ensures that data always satisfies certain rules. This approach has several advantages for correctness and reliability.

- Foreign keys prevent orphaned rows by ensuring that references between tables remain valid. If you attempt to insert an order with a customer_id that doesn't exist in the customers table, the database rejects the insert. Similarly, you cannot delete a customer if orders reference that customer unless you specify cascading behavior. This referential integrity prevents the database from ever containing inconsistent relationships.

- Unique constraints prevent duplicates by ensuring that no two rows have the same value for specified columns. For example, a unique constraint on email addresses ensures that each user can have only one account. The database enforces this at the storage level, typically using a unique index that automatically detects and rejects duplicate values.

- Check constraints enforce invariants by validating that column values satisfy specified conditions. For example, a check constraint can ensure that account_balance is never negative, that age is greater than zero, or that end_date is after start_date. These constraints are evaluated on every insert and update, preventing invalid states from ever being persisted.

Tradeoff: constraints add overhead on writes and complicate sharding. Every write operation must check all relevant constraints, which adds CPU overhead and can lock resources. Foreign key checks in particular can be expensive because they may require reading from other tables. In sharded environments, constraints become even more complex because foreign key relationships often span shards, requiring cross-shard coordination that undermines the benefits of horizontal scaling. This is why many large-scale systems either avoid sharding altogether or implement constraints at the application level rather than in the database.

---

### Transactions and ACID (With Real Internals)

ACID is the traditional contract for transactions: a set of properties that guarantee that database transactions are processed reliably. These properties were formalized in the 1980s and have become the foundation for transactional database systems.

- **Atomicity**: all-or-nothing.

  A transaction is an indivisible unit of work—either all of its operations complete successfully, or none of them take effect. If a transaction fails partway through (due to an error, crash, or explicit rollback), the database must roll back any changes made so far, restoring the database to its state as if the transaction never began. This prevents partial updates that could leave the database in an inconsistent state. For example, in a money transfer, atomicity ensures that you never debit one account without crediting the other.

- **Consistency**: invariants hold before/after (not "CAP consistency").

  This property ensures that a transaction transforms the database from one valid state to another valid state, obeying all defined rules including constraints, cascades, triggers, and any combination thereof. Importantly, this is different from the "consistency" in CAP theorem:
  - ACID consistency refers to application-level invariants (like account balances summing correctly)
  - CAP consistency refers to whether all replicas see the same data at the same time

  The database is responsible for maintaining consistency, but the transaction itself must be written correctly to achieve the desired business logic.

- **Isolation**: concurrent transactions behave as if executed in some serial order (depending on isolation level).

  When multiple transactions execute concurrently, the database must ensure that their effects don't interfere with each other in unexpected ways. Different isolation levels provide different guarantees about how transactions interact, trading off between correctness and performance:
  - The strongest level, serializable isolation, guarantees that the concurrent execution is equivalent to some serial ordering of the transactions
  - Weaker levels allow certain anomalies for better performance but require the application to handle potential conflicts

- **Durability**: committed changes survive crashes.

  Once a transaction commits, its changes are permanent and will survive system failures including power loss, crashes, and restarts. The database must ensure that committed data is written to stable storage before acknowledging the commit to the client. This typically involves:
  - Write-ahead logging
  - Careful management of disk buffers to ensure that data is not lost in the event of a crash

#### How durability is implemented: WAL (Write-Ahead Log)
Most RDBMS use a **write-ahead log**: this is the fundamental mechanism that enables atomicity and durability. The write-ahead logging protocol requires that all changes be written to a log before they are applied to the actual data pages. This simple rule has profound implications for database reliability and performance.

- Before modifying data pages on disk, the database appends a log record describing the change.

  The log record contains enough information to:
  - Redo the change if needed (the new values)
  - Undo it if the transaction aborts (the old values)

  The log is written to stable storage (typically using fsync to ensure the data reaches the disk) before any data pages are modified. This ensures that even if a crash occurs after the log write but before the data page write, the database can recover by replaying the log.

- On crash, recovery replays committed log entries and rolls back incomplete ones.

  During recovery, the database scans the log from the last checkpoint forward:
  - For transactions that committed before the crash, it reapplies their changes (redo phase) to ensure all committed changes are reflected in the data pages
  - For transactions that were in progress at the time of the crash, it reverses their changes (undo phase) to ensure atomicity—partial transactions are rolled back as if they never happened

Why it matters:

- Sequential append is fast:

  Appending to the end of a log file is much faster than random disk I/O because:
  - The disk head doesn't need to seek
  - Modern SSDs also handle sequential writes much more efficiently than random writes

  This performance characteristic makes WAL efficient for write-heavy workloads.

- Data pages can be flushed later:

  Because the log contains all the information needed to recover, the database can defer writing dirty data pages to disk. It can:
  - Batch page writes
  - Optimize I/O patterns
  - Write pages only when necessary (such as during checkpointing or when the buffer pool is full)

  This write coalescing significantly improves performance.

- Enables crash recovery and replication (log shipping):

  The same WAL that provides durability also enables other critical features:
  - Physical replication typically works by shipping WAL records from the primary to replicas, which then apply the same changes to maintain copies of the data
  - Point-in-time recovery is possible by replaying WAL from a backup to a specific moment
  - Change Data Capture (CDC) systems can read the WAL to stream changes to downstream systems

Key concepts:

- **LSN (Log Sequence Number)**: the position in the WAL.

  Each log record is assigned a monotonically increasing LSN that identifies its position in the log:
  - Data pages are stamped with the LSN of the most recent log record that modified them
  - During recovery, the database compares page LSNs with log LSNs to determine which pages need recovery
  - LSNs also serve as checkpoints for replication and backup

- **Checkpoint**: a point where dirty pages are flushed, limiting recovery time.

  As the WAL grows, recovery time increases because the database must scan more log records. Checkpoints truncate the log by:
  - Ensuring that all changes up to a certain LSN have been flushed to data pages
  - After a checkpoint, log records before that point can be discarded or archived

  Checkpoints are typically performed periodically or when the WAL reaches a certain size.

Tradeoff:

- Heavy write workloads stress WAL IO and checkpoints:

  - Every write must go to the WAL before being acknowledged to the client, making WAL throughput a bottleneck for write-intensive applications
  - Checkpoints can cause I/O spikes as dirty pages are flushed, potentially impacting performance during the checkpoint window

  Database administrators must tune checkpoint frequency, WAL size, and buffer pool configuration to balance recovery time against runtime performance.

#### Atomicity: undo/redo and commit protocol
Databases ensure atomicity via a combination of logging and commit protocols that guarantee transactions are all-or-nothing. The implementation details vary by database, but the fundamental principles are similar across systems.

- **Undo info** (to roll back):

  Each log record contains the before-image of the modified data—the values as they existed before the change. This undo information:
  - Allows the database to reverse the effects of a transaction if it aborts or if the transaction was in progress during a crash
  - During rollback, the database applies undo records in reverse order to restore the database to its state before the transaction began

- **Redo info** (to reapply after crash):

  Each log record also contains the after-image—the new values written by the transaction. This redo information:
  - Allows the database to reapply committed transactions that may not have been fully written to data pages before a crash
  - During recovery, the database applies redo records in forward order to ensure all committed changes are durable

The commit protocol ensures atomicity across the different stages of transaction processing:

- When a transaction commits, the database performs a commit record write to the WAL
- This commit record marks the transaction as committed
- Only after the commit record is safely on disk does the database acknowledge the commit to the client

This two-phase approach (write log, then acknowledge) ensures that committed transactions can always be recovered even if the data pages themselves weren't flushed.

Some engines use ARIES-like logging. ARIES (Algorithms for Recovery and Isolation Exploiting Semantics) is a recovery algorithm developed by IBM that provides a comprehensive framework for transaction recovery:

- ARIES uses three phases:
  - Analysis (determine which transactions were active)
  - Redo (reapply changes from committed transactions)
  - Undo (roll back changes from incomplete transactions)

- It introduces concepts like:
  - Write-ahead logging with steal and force policies
  - Dirty page tables
  - Transaction tables to optimize recovery

Many modern databases, including PostgreSQL and SQL Server, use recovery algorithms inspired by ARIES, though the exact mechanics vary by DB.

#### Isolation: isolation levels
Isolation levels define how transaction interleaving is controlled and what anomalies can occur. The SQL standard defines four isolation levels, though databases implement them differently in practice. Understanding these levels is critical because they represent fundamental tradeoffs between correctness and performance.

Common isolation levels:

- **Read Committed**: each statement sees only committed data; non-repeatable reads possible.

  This is the default isolation level in many databases:
  - It guarantees that a transaction never reads uncommitted data from other transactions
  - It does not guarantee that if the transaction re-reads the same row, it will see the same value

  If another transaction commits a change to that row between your two reads, you'll see the new value on the second read. This anomaly is called a "non-repeatable read" or "fuzzy read." Read committed is a good balance for many workloads because it provides basic consistency without excessive locking.

- **Repeatable Read**: stable snapshot for the transaction; prevents non-repeatable reads.

  In this level, a transaction sees a consistent snapshot of the database as of the time it begins (or as of its first read, depending on implementation):
  - If the transaction reads a row twice, it will see the same value both times, even if other transactions commit changes to that row in the meantime
  - However, repeatable read typically does not prevent "phantom reads"—new rows that match your query criteria might appear if another transaction inserts them

  This is because repeatable read locks existing rows but not the gaps between them where new rows could be inserted.

- **Serializable**: strongest; guarantees serial behavior (often implemented via predicate locking or SSI).

  Serializable isolation guarantees that the concurrent execution of transactions is equivalent to some serial ordering of those transactions. This eliminates all concurrency anomalies:
  - Dirty reads
  - Non-repeatable reads
  - Phantom reads
  - Write skew

  Implementing true serializability efficiently is challenging:
  - Traditional approaches use predicate locking (locking not just rows but the query predicates that select them) or range locks
  - Modern databases like PostgreSQL use Serializable Snapshot Isolation (SSI), which detects dangerous patterns of conflicts and aborts transactions rather than preventing them upfront

Tradeoff:

- Stronger isolation reduces anomalies but increases contention/aborts:

  Higher isolation levels require more locking or more sophisticated conflict detection, which can reduce concurrency and throughput:
  - Serializable isolation in particular can cause transaction aborts when conflicts are detected, requiring the application to retry transactions

  Database administrators must choose the appropriate isolation level based on the correctness requirements of the application—many systems use read committed by default and upgrade to repeatable read or serializable only for specific operations that need stronger guarantees.

---

### Indexes and Storage Engines

#### Pages, buffer pool, and disk layout
RDBMS store data in **pages** (e.g., 8KB/16KB blocks). Access pattern:

- Data lives on disk/SSD as pages: databases don't read or write individual rows—they read and write entire pages.

  When you update a single row:
  - The database reads the entire page containing that row into memory
  - Modifies the row
  - Marks the page as "dirty"
  - Eventually writes the entire page back to disk

  This page-oriented I/O is necessary because disk storage operates on blocks, not individual bytes. The page size is a critical tuning parameter—smaller pages reduce data read per operation but increase overhead, while larger pages improve sequential scan throughput but waste memory for point queries.
- A **buffer pool** caches hot pages in memory: the buffer pool is a cache of database pages in RAM, typically the largest memory consumer in a database server (often 70-80% of available memory).

  The buffer pool manager implements caching policies (like LRU or clock algorithms) to decide which pages to keep and which to evict:
  - When a query needs a page, the buffer pool is checked first
  - If cached (a "hit"), the operation is fast (memory access)
  - If not (a "miss"), the page must be read from disk, which is orders of magnitude slower

  The buffer pool hit ratio is a key performance metric—a 99% hit ratio means 99% of page accesses are served from memory.
- Reads/writes operate on pages: all database operations eventually translate to page reads and writes.

  Even simple operations require page access:
  - A point query on a primary key requires reading the page(s) containing that row
  - Updates require reading the page, modifying it in memory, and eventually writing it back
  - Deletes require reading the page, marking the row as deleted, and writing the page back

  This page-level granularity means performance is heavily influenced by whether required pages are in the buffer pool.

If your working set fits in memory, latency is dominated by CPU. If not, it’s dominated by random IO.

#### B-Tree indexes
Most OLTP RDBMS use **B-Tree / B+Tree** indexes.

- Balanced tree optimized for block storage: B-Trees and B+Trees are self-balancing tree structures designed specifically for disk-based storage.

  Unlike binary trees which have at most 2 children per node, B-Trees have many children per node (often hundreds), making each node roughly the size of a disk page:
  - This design minimizes the number of disk I/O operations needed to traverse the tree
  - A B-Tree with 100 children per node and 3 levels can index 1 million records with just 3 disk reads
  - The tree automatically rebalances on inserts and deletes to maintain balance, ensuring consistent lookup performance

##### B-Tree vs B+Tree

**B-Tree:**
- Data stored in all nodes (internal and leaf)
- Each node contains keys and actual data/pointers
- Lower fanout (fewer keys per node because data takes space)
- Deeper tree structure

**B+Tree (used in most databases):**
- Data stored ONLY in leaf nodes
- Internal nodes store only keys for navigation
- Higher fanout (more keys per node → shallower tree)
- Leaf nodes linked together for range scans
- Better for disk storage and range queries

**B+Tree Structure Example:**

```
                ┌─────────┐
                │  50,100 │  ← Internal node (keys only)
                └────┬────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
     ┌───▼───┐   ┌──▼──┐   ┌───▼───┐
     │ 10,30 │   │ 70  │   │ 120   │  ← Internal nodes
     └───┬───┘   └──┬──┘   └───┬───┘
         │          │          │
     ┌───▼───┐  ┌──▼────┐  ┌──▼────┐
     │10,20,30│→ │70,80,90│→ │120,130│  ← Leaf nodes (linked)
     └────────┘  └────────┘  └───────┘
      (data)     (data)      (data)
```

**Concrete Example:**

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100)
);

INSERT INTO users VALUES (5, 'Alice', 'alice@email.com');
INSERT INTO users VALUES (2, 'Bob', 'bob@email.com');
INSERT INTO users VALUES (8, 'Charlie', 'charlie@email.com');
```

**Clustered B+Tree on Primary Key (id):**

```
                ┌─────────┐
                │   5,8   │  ← Internal node
                └────┬────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
     ┌───▼───┐   ┌──▼──┐   ┌───▼───┐
     │  2,5  │   │  8  │   │ (empty)│  ← Leaf nodes with data
     └───┬───┘   └──┬──┘   └────────┘
         │          │
     ┌───▼────┐  ┌─▼──────┐
     │id=2    │  │id=5,8  │  ← Actual data
     │Bob     │  │Alice,  │
     └────────┘  │Charlie │
                 └────────┘
```

**Secondary Index on (name):**

```
                ┌──────────────┐
                │ 'Alice','Eve'│  ← Internal node
                └──────┬───────┘
                       │
           ┌───────────┼───────────┐
           │           │           │
       ┌───▼────┐  ┌──▼───┐  ┌───▼─────┐
       │ 'Bob'  │  │'Char'│  │ 'Diana' │  ← Leaf nodes
       └───┬────┘  └───┬───┘  └─────────┘
           │            │
       ┌───▼────┐  ┌───▼────┐
       │ id=2   │  │ id=8   │  ← Points to primary key
       └────────┘  └────────┘
```

**Index Lookup Example:**

Query: `SELECT * FROM users WHERE id = 8`

1. Start at root: 8 > 5? Yes → go right
2. Reach leaf node containing key=8
3. Return data: id=8, name='Charlie'

Without index: Must scan all rows (O(n))
With index: O(log n) - 2-3 disk reads

- Supports point lookups and range scans efficiently: B+Trees, the variant most commonly used in databases, store all data values in the leaf nodes and link leaf nodes together in a linked list.

  This structure enables efficient range queries:
  - Once you find the starting point in the tree, you can scan through sequential leaf nodes to retrieve all values in the range without traversing back up the tree
  - Point lookups are efficient because the tree structure allows binary search through the index to find the exact key

  The combination of efficient point lookups and range scans makes B+Trees ideal for a wide variety of query patterns in OLTP workloads.

##### B-Tree Tradeoffs

- Random inserts cause page splits: when a page becomes full and a new key needs to be inserted, the B-Tree must split the page into two pages and redistribute the keys.

  This operation:
  - Requires writing two pages instead of one, increasing I/O
  - Random insertion order (as opposed to sequential) maximizes page splits because inserts are distributed across the tree rather than concentrated at the end
  - Page splits also cause index fragmentation over time, as the physical layout of pages becomes suboptimal

  Most databases provide index maintenance operations (like REINDEX or OPTIMIZE) to rebuild indexes and eliminate fragmentation.

- Secondary indexes duplicate key info (write amplification): each secondary index stores a copy of the indexed columns along with a pointer to the row.

  For a table with N indexes, each insert/update/delete must update all N indexes, multiplying the I/O cost:
  - This write amplification is the primary reason why adding many indexes slows down write performance
  - Secondary indexes also consume significant storage space—in some cases, the indexes can be larger than the data itself

  Database administrators must carefully balance read performance (which benefits from more indexes) against write performance and storage costs (which suffer from more indexes).

#### Write-Ahead Log (WAL) and Durability

Databases use a Write-Ahead Log (also known as Redo Log) to ensure durability and enable crash recovery.

##### WAL Structure and Storage

- **Sequential log files**: The WAL is stored as sequential log files on disk, separate from the main data files.

  Typical storage:
  - Separate directory (e.g., PostgreSQL's `pg_wal`, MySQL's `ib_logfile0`, `ib_logfile1`)
  - Rotating files with fixed size segments (e.g., 16MB per segment in PostgreSQL)
  - Sequential naming: `000000010000000000000001`, `000000010000000000000002`
  - Append-only: New log entries are appended to the current WAL file

- **Write-ahead semantics**: Before modifying any data pages, the database writes the change to the WAL.

  This ensures:
  - If a crash occurs, the database can replay the WAL to recover committed transactions
  - The WAL contains all changes needed to reconstruct the database state

##### fsync and Durability

- **fsync() system call**: Forces data from OS page cache to stable storage (disk).

  How it works:
  1. Normal write: Data written to OS page cache (RAM) - very fast
  2. fsync(): Forces OS to flush cached data to actual disk - slower
  3. Call blocks until data is durably on disk

  Why fsync matters:
  - Without fsync, data in page cache is lost if system crashes or loses power
  - Databases use fsync to ensure committed transactions survive crashes
  - WAL must be fsync'd before acknowledging transactions

##### Transaction Commit Sequence

**Standard ACID mode (durability guaranteed):**

```
1. Transaction begins
2. Changes written to WAL buffer (RAM)
3. fsync WAL to disk ← Ack sent here
4. Changes applied to data pages in memory (buffer pool)
5. Data pages flushed to disk later (checkpoint/background)
```

**Key insight:**

- Ack sent after WAL fsync, NOT after data page fsync
- Data pages are written lazily in the background
- If crash occurs after WAL fsync but before data pages reach disk, database replays WAL during recovery
- Only one durable copy (WAL or data pages) is needed for recovery

##### When WAL Can Be Lost

- **Before fsync() completes**: If system crashes after data is in OS cache but before fsync, WAL entries are lost
- **Disk failure**: Physical disk failure can corrupt or destroy WAL files
- **Filesystem corruption**: Severe filesystem corruption can affect WAL files
- **Power loss during write**: Power loss during actual disk write can leave WAL in inconsistent state

##### Configuration Tradeoffs

- **Asynchronous commit** (`synchronous_commit = off` in PostgreSQL):
  - Client gets ack after WAL written to OS cache (not fsync'd)
  - Higher performance, but ~1 second of potential data loss on crash

- **fsync disabled** (`fsync = off`):
  - Ack sent immediately, no durability guarantee
  - Highest performance, total data loss on crash
  - Only for non-critical data

- **Default (sync commit + fsync)**:
  - Ack sent after WAL fsync to disk
  - Minimal data loss risk
  - Standard for production databases

##### Recovery Process

On database startup after crash:

1. Read WAL from last checkpoint
2. Replay committed transactions (redo)
3. Rollback uncommitted transactions (undo)
4. Bring database to consistent state

The WAL is the foundation of ACID durability in relational databases.

#### Clustered vs heap tables (important for performance)
The choice between heap and clustered storage organization has profound implications for query performance and is a key difference between database engines like PostgreSQL (heap) and InnoDB (clustered).

- **Heap table**: rows stored roughly in insertion order; indexes point to row locations.

  In a heap table, the data is not organized in any particular order—new rows are appended wherever there's free space:
  - Indexes (including the primary key index) store the indexed columns along with a physical pointer (TID in PostgreSQL) to the row's location on disk
  - This design makes inserts fast because the database can append rows without worrying about their position relative to other rows
  - However, range queries on any column require scanning index entries and then following pointers to scattered data pages, causing random I/O

- **Clustered table** (e.g., InnoDB): data is stored ordered by primary key; secondary indexes reference primary key.

  In a clustered table, the table itself is organized as a B-Tree indexed by the primary key:
  - The data rows are the leaf nodes of this B-Tree, stored in primary key order
  - This means that range queries on the primary key can read sequential data pages, which is much more efficient than random I/O
  - However, secondary indexes must store the primary key value rather than a physical pointer, making them larger
  - Inserts with random primary keys can cause page splits throughout the table as the database maintains the sorted order

Tradeoff:

- Clustered improves range scans on primary key: because data is stored in primary key order, queries that fetch ranges of primary key values (like "WHERE id BETWEEN 1000 AND 2000") can read sequential data pages, which is much faster than random I/O.

  This makes clustered tables excellent for:
  - Time-series data (when using timestamps as the primary key)
  - Workloads that frequently query ranges of the primary key

- But random primary keys cause page splits and fragmentation: if you use random UUIDs or random integers as primary keys in a clustered table, inserts will be distributed throughout the table's B-Tree rather than appended at the end.

  This causes:
  - Frequent page splits and fragmentation, degrading performance over time

  This is why many databases recommend using sequential or monotonically increasing primary keys (like auto-increment integers or Snowflake IDs) for clustered tables. The choice of primary key design becomes critical for clustered table performance in a way that it doesn't for heap tables.

#### Secondary indexes
Secondary indexes accelerate queries on non-primary key columns.

Tradeoffs:

- Every secondary index makes writes slower: each INSERT, UPDATE, or DELETE must update not just the table data but also every secondary index.

  If a table has 5 secondary indexes, a single write operation becomes 6 write operations (1 for the table + 5 for the indexes):
  - This write amplification is cumulative and can become a severe bottleneck for write-heavy workloads
  - The performance impact is often non-linear because the additional I/O can cause buffer pool pressure, leading to more cache evictions and more disk reads

- Indexes must be maintained during updates: when you update a column that's indexed, the database must remove the old entry from the index and insert a new entry with the updated value.

  This can cause:
  - Page splits in the index B-Tree, further increasing I/O
  - Even updating a non-indexed column can be slower in tables with many indexes because the database must still check whether the update affects any indexed columns (through foreign key relationships or functional indexes)

  This maintenance overhead is why database administrators carefully monitor index usage and remove unused indexes.

---

### Query Execution: Optimizer, Plans, and Joins

SQL is declarative, but execution is physical. This fundamental distinction is what makes relational databases so powerful:

- In a declarative language, you describe what you want ("SELECT * FROM users WHERE age > 25") without specifying how to get it
- The query optimizer then determines the most efficient execution plan based on statistics, indexes, and system state
- This abstraction allows the same query to execute efficiently even as data distributions change or new indexes are added—the optimizer automatically adapts the execution strategy

In contrast, imperative APIs (like many NoSQL query interfaces) require you to specify exactly how to retrieve data, shifting the burden of optimization to the application developer.

#### Query optimizer
The optimizer is a sophisticated piece of software that transforms your declarative SQL query into an executable physical plan. Modern databases use cost-based optimizers that enumerate many possible execution strategies and choose the one with the lowest estimated cost.

The optimizer chooses plans based on:

- Table statistics (row counts, distributions): the database maintains statistics about each table, including total row count, distinct value counts for columns, histograms of value distributions, and correlation between columns.

  These statistics allow the optimizer to estimate how many rows will match each predicate in your query:
  - For example, if a column has 1 million distinct values and you query for a specific value, the optimizer estimates that 1 row will match
  - If statistics are missing or outdated, these estimates can be wildly inaccurate, leading to poor plan choices

- Index selectivity: the optimizer considers whether indexes exist on columns used in WHERE clauses, JOIN conditions, and ORDER BY clauses.

  It estimates the selectivity of each index—how many rows will be returned for a given lookup:
  - A highly selective index (returning few rows) is valuable because it reduces the amount of data that must be scanned
  - The optimizer must decide between using an index or performing a full table scan, a decision that depends on selectivity and the cost of random I/O versus sequential I/O

- Cost model (IO + CPU): the optimizer assigns costs to different operations based on system characteristics.

  It estimates:
  - I/O costs based on whether data will be read sequentially (cheaper) or randomly (more expensive)
  - CPU costs based on the complexity of operations like sorting, hashing, and expression evaluation

  The cost model is calibrated for the specific hardware and storage configuration, which is why moving a database to different hardware can sometimes cause query plans to change.

This is why:

- Stale statistics can cause sudden slow queries: if the database's statistics don't reflect the actual data distribution, the optimizer may choose a suboptimal plan.

  For example:
  - If a table has grown significantly but statistics haven't been updated, the optimizer might choose a nested loop join assuming one table is small, when in reality it's now large

  This is why database administrators schedule regular statistics collection jobs (ANALYZE in PostgreSQL, UPDATE STATISTICS in SQL Server).

- The same query can have different performance as data grows: as data volumes change, different execution strategies become optimal.

  For example:
  - A nested loop join might be fastest when one table has 100 rows, but a hash join becomes faster when it has 1 million rows

  The optimizer automatically adapts to these changes, but the transition can sometimes be abrupt—performance might suddenly degrade when a table crosses a size threshold that causes the optimizer to switch plans.

#### Join algorithms
Common join strategies:

- **Nested loop join**: good when one side is small and index exists.

  Nested loop join is the simplest join algorithm: for each row in the outer table, it scans the inner table to find matching rows:
  - If the inner table has an index on the join key, the scan becomes an index lookup, making the join much more efficient
  - The complexity is O(N*M) in the worst case (where N and M are the table sizes), but with an index it becomes O(N*log(M))

  This algorithm is optimal:
  - When one table is very small (a few hundred rows)
  - When an index allows efficient lookups
  - It's also the only algorithm that can handle non-equi joins (joins with conditions other than equality)
- **Hash join**: builds hash table on smaller input; good for large equi-joins.

  Hash join works by:
  - Reading the smaller input table and building a hash table in memory keyed by the join columns
  - Scanning the larger table and probing the hash table to find matches

  This algorithm:
  - Has linear complexity O(N+M) and doesn't require indexes, making it excellent for large equi-joins where neither table is indexed
  - Can handle very large datasets, but if the hash table doesn't fit in memory, it must spill to disk, which significantly degrades performance

  The optimizer chooses which table to build the hash table from based on size estimates—building from the smaller table minimizes memory usage.
- **Sort-merge join**: sorts both sides then merges; good when inputs already sorted.

  Sort-merge join sorts both input tables on the join key, then merges them in a single pass similar to the merge step of merge sort:
  - This algorithm has complexity O(N log N + M log M) due to the sorting step, but if inputs are already sorted (perhaps from an ORDER BY clause or an index scan), it becomes O(N+M)

  Sort-merge join:
  - Is memory-efficient because it doesn't require building a hash table—it can process data in streams
  - Is also the best choice when the result needs to be sorted on the join key anyway, as it produces sorted output naturally
  - However, the sorting cost makes it slower than hash join for unsorted large datasets

Tradeoff:

- Hash joins need memory (or spill to disk): the hash table built during a hash join must fit in memory for optimal performance.

  If the hash table exceeds available memory:
  - The database must partition the data and spill to disk, dramatically increasing I/O

  This is why hash joins have a memory work_mem parameter (in PostgreSQL) that controls how much memory a single hash join can use:
  - Setting this too low causes excessive disk spilling
  - Setting it too high can cause the operating system to swap or cause out-of-memory errors

  Database administrators must tune this parameter based on workload characteristics.

- Sort-merge has sort cost: sorting is computationally expensive, especially for large datasets.

  The sort operation:
  - Requires O(N log N) comparisons
  - May also require disk I/O if the sort buffer is exceeded

  However, sort-merge join can be more efficient than hash join when inputs are already sorted or when the result needs to be sorted anyway. The optimizer estimates sort costs based on data size and available work_mem, choosing sort-merge only when it's likely to be faster than alternatives.

#### Why SQL is great for ad-hoc queries
Because SQL composes well: SQL's declarative nature and relational foundation enable powerful composition of operations.

You can:
- Combine joins, aggregations, window functions, and complex filters in arbitrary ways without worrying about the underlying implementation
- The query optimizer figures out how to execute your composition efficiently

- Joins: SQL allows you to express relationships between tables declaratively.

  You can:
  - Join any number of tables on any columns
  - The optimizer determines the best join order and algorithm

  This flexibility is crucial for ad-hoc analysis where you might need to explore relationships you didn't anticipate when designing the schema.

- Aggregations: SQL provides a rich set of aggregation functions (SUM, AVG, COUNT, etc.) combined with GROUP BY and HAVING clauses.

  You can:
  - Aggregate at multiple levels
  - Compute complex expressions
  - Filter on aggregated results—all in a single query

  The optimizer can push down predicates and use various aggregation strategies depending on data distribution.

- Window functions: SQL's window functions allow you to perform calculations across rows related to the current row without collapsing rows like GROUP BY does.

  This enables powerful analytical queries like:
  - Running totals
  - Ranking
  - Moving averages

  These operations would require complex application code in most NoSQL systems.

- Complex filters: SQL's WHERE clause supports arbitrary boolean expressions, subqueries, and set operations.

  You can:
  - Express complex business logic directly in the query
  - The optimizer can often optimize these expressions in ways that application code cannot

NoSQL systems often require queries to be known upfront because the physical layout is designed for specific access paths:

- In document databases, you must design your document structure around your query patterns
- In key-value stores, you need to know your access keys in advance

This query-driven data modeling trades flexibility for performance at scale, but it makes ad-hoc querying difficult or impossible. SQL's separation between logical schema and physical layout means you can add new queries without changing your data model.

---

### Concurrency Control: Locks vs MVCC

Concurrency control is where many “internals” decisions show up.

#### Lock-based concurrency
Classic approach: lock-based concurrency control uses explicit locks to coordinate access to shared data:

- When a transaction wants to read a row, it acquires a shared (read) lock
- When it wants to write, it acquires an exclusive (write) lock

Shared locks allow multiple readers to access the same data simultaneously, but exclusive locks block all other access (both reads and writes). The database tracks which transaction holds which locks and releases them when the transaction commits or aborts.

- Read locks, write locks: the two-phase locking protocol requires transactions to acquire locks before accessing data and release locks only after the transaction completes.

  This ensures serializability but can cause significant contention:
  - In strict two-phase locking, exclusive locks are held until commit, preventing other transactions from reading uncommitted data (which would violate isolation)
  - Some databases use intention locks (intention shared, intention exclusive) to lock at higher granularities (table, page) to improve performance

- Deadlocks are possible: when two transactions each hold locks that the other needs, a deadlock occurs.

  For example:
  - Transaction A locks row 1 and wants row 2, while transaction B locks row 2 and wants row 1
  - Neither can proceed

  The database must:
  - Detect deadlocks (typically using a wait-for graph and cycle detection)
  - Choose a victim transaction to abort

  Deadlocks are more likely with long-running transactions and complex access patterns. Application developers must be prepared to retry transactions that are aborted due to deadlocks.

Tradeoff:

- High contention can reduce throughput: under lock-based concurrency, readers block writers and writers block everyone.

  In read-heavy workloads:
  - This can severely limit throughput because each write must wait for all in-progress reads to complete

  In write-heavy workloads:
  - Lock contention becomes a bottleneck as transactions queue up waiting for locks

  The locking overhead itself (acquiring, releasing, and tracking locks) also consumes CPU resources. This is why many modern databases have moved toward MVCC for better concurrency.

#### MVCC (Multi-Version Concurrency Control)
Many RDBMS implement MVCC: instead of using locks to prevent concurrent access, MVCC allows multiple versions of the same row to exist simultaneously:

- When a transaction updates a row, it doesn't overwrite the existing row—it creates a new version
- The old version remains available for transactions that started before the update

This approach fundamentally changes the concurrency model by eliminating the reader-writer conflict.

- Each row can have multiple versions: the database maintains version information for each row, typically including the transaction ID that created the version and the transaction ID that made it obsolete (xmin and xmax in PostgreSQL).

  When a transaction reads a row:
  - It checks whether the version is visible based on its transaction ID and the visibility rules

  Old versions are kept around until they're no longer needed by any active transaction, at which point they can be cleaned up.

- Readers get a snapshot without blocking writers: when a transaction begins, it takes a snapshot of which transactions are active.

  It then:
  - Sees only row versions created by committed transactions before the snapshot
  - Ignores versions created by concurrent transactions

  This means:
  - Readers never block writers and writers never block readers
  - A writer can update a row while other transactions continue reading the old version

  This snapshot isolation provides the appearance of a consistent view of the database as of the transaction's start time.

- Writers create new versions: when a transaction updates a row, it creates a new version with its own transaction ID.

  The old version remains visible to transactions with earlier snapshots:
  - If multiple transactions update the same row concurrently, only the first to commit succeeds
  - Subsequent commits fail with a serialization error and must be retried

  This optimistic concurrency approach trades potential retry overhead for reduced blocking.

Benefits:

- High read concurrency: because readers never block writers and writers never block readers, MVCC enables excellent read throughput even under heavy write load.

  This is particularly valuable for:
  - Read-heavy workloads like web applications, reporting systems, and analytics dashboards

  Multiple readers can access the same data simultaneously without contention, and writes can proceed in parallel with reads. This non-blocking behavior is why databases using MVCC (PostgreSQL, MySQL InnoDB, Oracle) can handle much higher concurrency than lock-based systems.

- Fewer blocking reads: in lock-based systems, a read might block waiting for a write to complete, or a write might block waiting for reads to finish.

  With MVCC:
  - Reads always proceed immediately—they simply see the most recent committed version as of their snapshot
  - This eliminates read latency spikes caused by lock contention and makes query latency more predictable
  - Applications don't need to implement complex retry logic for read operations, simplifying application code

Costs:

- Vacuum/cleanup required (old versions must be reclaimed): because MVCC keeps old row versions around for concurrent transactions, the database needs a mechanism to reclaim space when versions are no longer needed.

  This cleanup process (called VACUUM in PostgreSQL, purge in MySQL):
  - Scans tables to identify dead versions and marks the space as reusable
  - If vacuum doesn't run frequently enough, tables can become bloated with dead versions, wasting storage and degrading performance (more data to scan, fewer rows per page)
  - Vacuum itself consumes I/O and CPU, so database administrators must tune vacuum frequency and aggressiveness based on workload characteristics

- Long-running transactions can bloat storage: if a transaction stays open for a long time (hours or days), it prevents cleanup of any row versions created after it started.

  Even if those rows have been updated and committed many times:
  - The old versions must be kept because the long-running transaction might still need to see them
  - This can cause significant table bloat and performance degradation

  This is why:
  - Database administrators monitor for long-running transactions and either set timeouts or aggressively kill transactions that exceed certain durations
  - Applications should be designed to keep transactions short and avoid holding transactions open across user interactions

---

### Replication, High Availability, and Failover

#### Primary-replica replication
Common in MySQL/Postgres setups: primary-replica replication is the most common high availability strategy for relational databases:

- The primary server handles all writes and streams change data to one or more replica servers
- The replicas apply the changes to maintain copies of the data

- Primary handles writes: all write operations (INSERT, UPDATE, DELETE) are sent to the primary server.

  The primary:
  - Executes these transactions
  - Writes to its WAL
  - Acknowledges the commit to the client

  The primary is the single source of truth for the dataset, which simplifies consistency but creates a write bottleneck:
  - Read operations can be directed to replicas to distribute load, but this introduces read-your-writes consistency issues
  - A client might write to the primary and immediately read from a replica that hasn't received the update yet

- Replicas apply WAL/binlog changes: replicas receive a stream of change data from the primary, typically either the WAL itself (physical replication) or a logical change stream (logical replication).

  They:
  - Apply these changes in the same order they occurred on the primary
  - Maintain an eventually consistent copy

  Replication can be:
  - Synchronous (the primary waits for replicas to acknowledge before committing)
  - Asynchronous (the primary commits immediately and replication happens in the background)

  Tradeoffs:
  - Synchronous replication provides stronger consistency but increases write latency
  - Asynchronous replication provides lower latency but risks data loss if the primary fails before replicas catch up
- Reads can be served from replicas: one of the primary benefits of replication is the ability to distribute read load across multiple servers.

  Analytical queries, reporting dashboards, and other read-heavy workloads can be directed to replicas, reducing load on the primary:
  - This read scaling can dramatically improve overall system throughput

  However, reading from replicas introduces consistency challenges:
  - Replicas may lag behind the primary by milliseconds or seconds, meaning clients might read stale data
  - Applications must decide whether to accept eventual consistency for reads or route all reads to the primary for strong consistency
  - Some databases offer read-after-write consistency by routing a client to the primary for a time window after a write

Tradeoffs:

- Replication lag means replicas may be stale: the time between a write committing on the primary and being applied on a replica is called replication lag.

  Lag can vary from milliseconds to seconds or even minutes depending on:
  - Network conditions
  - Replica load
  - The volume of writes

  During lag periods, replicas serve stale data:
  - This is acceptable for some use cases (analytics, recommendations) but unacceptable for others (financial transactions, inventory)
  - Applications must be designed to handle stale reads or avoid replicas for critical operations
  - Monitoring lag is essential—excessive lag can indicate performance problems on replicas or network issues

- Failover requires careful handling of split-brain: when the primary fails, the system must promote a replica to become the new primary.

  This failover process is complex and error-prone:
  - If the old primary hasn't actually failed but is merely disconnected from the replicas (a network partition), both the old primary and the newly promoted replica may accept writes independently, causing data divergence (split-brain)
  - Most systems use a consensus protocol or an external coordinator (like ZooKeeper, etcd, or a managed service's automated failover) to ensure only one primary exists at a time
  - Failover also requires handling unreplicated writes from the old primary, which may need to be reconciled or discarded

#### Synchronous vs asynchronous replication
The choice between synchronous and asynchronous replication represents a fundamental tradeoff between durability, latency, and availability—directly related to the PACELC theorem.

- **Async**: lower write latency, but a primary crash can lose acknowledged writes.

  In asynchronous replication:
  - The primary acknowledges writes to the client immediately after writing to its local WAL, without waiting for replicas
  - This provides the lowest possible write latency because the client doesn't wait for network round trips to replicas

  However:
  - If the primary crashes before replicating the write, those acknowledged writes are permanently lost
  - This data loss window is typically small (milliseconds to seconds) but is unacceptable for many applications

  Asynchronous replication is suitable for workloads that can tolerate some data loss in exchange for lower latency.

- **Sync**: stronger durability but higher latency and lower availability under network issues.

  In synchronous replication:
  - The primary waits for at least one replica to acknowledge receiving and persisting the write before acknowledging to the client
  - This ensures that any acknowledged write is safely stored on at least two nodes, providing strong durability

  The costs:
  - Higher write latency due to network round trips
  - More critically, synchronous replication reduces availability—if a replica is unreachable due to network issues, the primary cannot commit writes (or must degrade to asynchronous mode), potentially blocking all writes

  This is the availability penalty from the CAP theorem: strong consistency (sync replication) reduces availability during partitions.

---

### Scaling SQL: Vertical, Sharding, and Distributed SQL

#### Vertical scaling
Scale up CPU/RAM/IOPS: vertical scaling means increasing the capacity of a single database server by adding more CPU cores, more RAM, faster storage, or higher network bandwidth.

This is the simplest scaling strategy because:
- It requires no changes to the application or database configuration
- You just provision a larger server and migrate your database

Pros:

- Simplest operationally: vertical scaling requires minimal operational complexity.

  There's no need to:
  - Implement sharding logic
  - Configure replication topologies
  - Handle cross-node coordination

  The database continues to operate as a single node:
  - Preserving all ACID guarantees
  - Simplifying monitoring, backup, and maintenance

  This simplicity is why many organizations start with vertical scaling and only consider horizontal scaling when they hit hardware limits.

- Strongest guarantees: a single-node database provides the strongest consistency and isolation guarantees because all data is on one machine.

  There's:
  - No network latency for coordination
  - No replication lag
  - No possibility of split-brain scenarios

  Transactions can be fully ACID without the compromises required by distributed systems. This makes vertical scaling ideal for applications that require strong correctness guarantees.

Cons:

- Limited by hardware: there's an upper bound to how much you can scale a single machine.

  The largest available servers have limits on:
  - CPU cores
  - RAM capacity
  - Storage I/O

  Once you hit these limits, vertical scaling is no longer an option:
  - Different components may bottleneck at different times—CPU, memory, or I/O
  - Upgrading one component may not help if another is the bottleneck

- Expensive at top end: high-end servers with large amounts of RAM, fast SSDs, and many CPU cores are exponentially more expensive than commodity hardware.

  For example:
  - A server with 1TB of RAM costs significantly more than ten servers with 100GB each, even though the total capacity is the same

  This cost curve makes vertical scaling economically unsustainable at very large scales. Additionally:
  - Larger servers have higher single points of failure
  - When a massive server fails, the impact is greater than when a smaller server fails

#### Sharding (application-level partitioning)
Split data across many nodes by shard key: sharding is the practice of horizontally partitioning data across multiple database servers:

- Each shard contains a subset of the total data, determined by a shard key (like user_id, region, or hash of some identifier)
- The application layer is responsible for routing queries to the correct shard based on the shard key

This allows the system to scale writes and storage by adding more shards.

Pros:

- Scales writes and storage: because each shard handles only a subset of the data, write throughput scales linearly with the number of shards.

  For example:
  - If one shard can handle 10,000 writes per second, ten shards can handle 100,000 writes per second

  Similarly:
  - Storage capacity scales with the number of shards

  This horizontal scaling allows you to exceed the limits of any single machine. Sharding is the traditional way to scale relational databases beyond what vertical scaling can achieve.

Cons:

- Cross-shard joins are painful: queries that need to join data from different shards become extremely expensive.

  The application must:
  - Query multiple shards and join the results in memory, which is slow and complex

  This is why sharding requires careful data model design:
  - You should design your schema so that most queries access data from a single shard
  - Denormalization and data duplication are often necessary to avoid cross-shard joins

  This loss of relational flexibility is a significant tradeoff.

- Transactions across shards become complex: ACID transactions that span multiple shards are difficult to implement.

  Most sharded systems:
  - Either don't support cross-shard transactions
  - Or implement them using two-phase commit, which has performance and reliability issues

  Application developers must:
  - Design their data access patterns to avoid needing cross-shard transactions
  - Implement compensation logic for eventual consistency

  This complexity moves into the application layer.

- Resharding is operationally difficult: as data grows, you may need to add new shards or rebalance data across existing shards.

  Resharding:
  - Requires moving data between shards, which is a complex, error-prone operation
  - Can cause significant downtime or performance degradation

  This is why:
  - Choosing a good shard key initially is critical because changing it later is extremely difficult
  - Hotspots can develop if a shard key has uneven distribution, causing some shards to be overloaded while others are underutilized

#### Distributed SQL
Distributed SQL databases (CockroachDB, YugabyteDB, Google Spanner, TiDB) combine the familiar SQL interface and relational model of traditional databases with the horizontal scalability and fault tolerance of distributed systems.

They provide a single logical database that automatically partitions data across nodes, replicates for fault tolerance, and maintains strong consistency through distributed consensus protocols.

##### What Distributed SQL Provides

- **SQL interface and relational model**: Distributed SQL databases speak SQL and support schemas, constraints, and transactions.

  This allows developers to:
  - Use existing SQL knowledge and tools without learning a new query language
  - Write SQL queries as if it were a single-node database
  - The system handles the complexity of distribution, replication, and coordination transparently

- **Automatic partitioning and replication**: Unlike traditional databases where sharding must be implemented at the application layer, distributed SQL databases handle this automatically.

  The database:
  - Automatically splits data across nodes based on primary key ranges or user-defined partitioning schemes
  - Has built-in replication—each partition is replicated to multiple nodes for fault tolerance
  - Handles rebalancing when nodes are added or removed
  - Manages failover when nodes fail

- **Strong consistency via consensus**: Distributed SQL databases use consensus protocols like Raft or Paxos to agree on the order of operations across replicas.

  This ensures:
  - All replicas see the same sequence of writes, providing strong consistency
  - When a client writes data, the write is propagated through the consensus protocol and only acknowledged once a quorum of replicas have agreed

  This approach provides the consistency guarantees of traditional databases while operating across multiple nodes.

##### How It Works Internally

- **Data partitioning**: The database automatically partitions data based on primary key ranges.

  For example:
  - Rows with primary keys 1-1000 might be on one node, 1001-2000 on another, and so on

  This range-based partitioning:
  - Allows efficient range scans within a partition while distributing data across the cluster
  - Some systems also support hash-based partitioning or user-defined partitioning schemes

  The partitioning is transparent to the application:
  - The router layer knows which partition contains which keys and routes queries accordingly

- **Per-partition replication**: Each partition is replicated to multiple nodes (typically 3 or 5) for fault tolerance.

  If one node fails:
  - The other replicas can continue serving requests

  Replication:
  - Is handled through the consensus protocol, ensuring all replicas agree on the order of writes
  - The replication factor is configurable—higher replication factors provide better fault tolerance but increase cost and write latency

- **Consensus groups**: Each partition has its own consensus group (a set of replicas that participate in the Raft or Paxos protocol).

  This design:
  - Allows different partitions to make progress independently—if the leader for one partition fails, it doesn't affect other partitions
  - Ensures writes are durably committed only once a quorum of replicas have agreed

- **Distributed transactions**: When a transaction spans multiple partitions, distributed SQL databases use two-phase commit (2PC) coordinated with the consensus protocol.

  The transaction coordinator:
  - Prepares the transaction on all participating partitions
  - Waits for them to agree via their local consensus protocols
  - Then commits the transaction globally

  This:
  - Ensures ACID semantics across partitions but adds significant latency due to multiple rounds of consensus
  - Most distributed SQL systems optimize for single-partition transactions to avoid this overhead

Tradeoffs:

- Higher write latency (consensus quorum): because distributed SQL databases must achieve consensus across multiple replicas before acknowledging writes, write latency is significantly higher than single-node databases.

  Each write:
  - Requires multiple network round trips to achieve quorum
  - Adds latency proportional to network latency between nodes

  This:
  - Makes distributed SQL less suitable for latency-sensitive write workloads
  - Read latency can also be higher if strong consistency is required, as reads may need to consult the quorum

- Complexity is moved into the database (less into app): distributed SQL databases absorb much of the complexity that would otherwise be in the application layer with manual sharding.

  The database handles:
  - Partitioning
  - Replication
  - Failover
  - Distributed transactions automatically

  This:
  - Simplifies application development and reduces the surface area for bugs
  - However, this complexity doesn't disappear—it's moved into the database itself, which becomes more complex to operate and debug
  - Troubleshooting performance issues in a distributed SQL system requires understanding consensus protocols, network topology, and distributed systems concepts

- Excellent for correctness + scale, but not always cost-effective: distributed SQL databases provide strong correctness guarantees (ACID transactions, strong consistency) at scale, which is valuable for many applications.

  However:
  - They require more hardware resources than manually sharded setups because each partition is replicated and consensus adds overhead
  - The operational complexity is also higher than single-node databases
  - For workloads that don't require strong consistency or can tolerate eventual consistency, simpler and cheaper solutions may be more appropriate

  Distributed SQL shines when you need both scale and strong correctness, but you pay a premium for those guarantees.

---

## NoSQL Databases

### Why NoSQL Exists
NoSQL became popular because large-scale internet workloads needed: in the late 2000s and early 2010s, companies like Google, Amazon, Facebook, and Twitter encountered scaling challenges that traditional RDBMS couldn't handle efficiently.

These workloads had different characteristics than the transactional business applications that relational databases were designed for.

- Horizontal scaling on commodity hardware: internet-scale applications needed to handle millions of users and petabytes of data.

  Vertical scaling:
  - Was insufficient and too expensive

  NoSQL databases:
  - Were designed from the ground up to run on clusters of commodity servers
  - Automatically partitioning data and distributing load

  This horizontal scaling allowed companies to:
  - Add capacity incrementally by adding more servers rather than replacing existing ones with larger machines

- Predictable performance at high throughput: at scale, the variability of query performance becomes as important as average performance.

  NoSQL databases optimize for predictable latency by:
  - Avoiding expensive operations like joins, complex query optimization, and distributed transactions
  - Simplifying the query model and optimizing for known access patterns

  This allows:
  - NoSQL systems to provide consistent performance even under heavy load
  - This predictability is critical for SLAs and user experience at scale

- Flexible data models for rapidly evolving products: internet products evolve rapidly, with frequent schema changes.

  RDBMS schema migrations:
  - Are operationally complex and can cause downtime

  NoSQL's schemaless or schema-flexible models:
  - Allow developers to iterate quickly without planning migrations far in advance
  - Document databases, for example, allow adding new fields to documents without requiring a schema change

  This flexibility accelerates development velocity, which is crucial for fast-moving product teams.

- Geo-distribution and high availability: global applications need to serve users worldwide with low latency.

  NoSQL databases:
  - Were designed with multi-region deployment in mind
  - Use tunable consistency to balance latency and correctness
  - Many NoSQL systems can continue accepting writes even during network partitions by relaxing consistency requirements

  This always-available design is valuable for globally distributed services where availability is prioritized over strong consistency.

The key idea: **optimize for specific access patterns**, often by: unlike relational databases which are general-purpose tools, NoSQL databases are specialized for specific access patterns.

This specialization comes from recognizing that at scale, you can't optimize for everything—you must choose what to optimize for.

- Denormalizing: storing related data together to avoid joins.

  For example:
  - In a social media application, you might store a user's posts directly in the user document rather than in a separate posts table

  This denormalization:
  - Makes reads faster (single document fetch) but makes writes slower (you must update all denormalized copies)

  The philosophy is to optimize for the common case (reads) and accept complexity for the less common case (writes).

- Avoiding joins: joins are expensive in distributed systems because they require coordinating data from multiple nodes, potentially across the network.

  NoSQL databases avoid joins by:
  - Designing the data model around access patterns—if you need to fetch related data together, store it together

  This query-driven data modeling:
  - Requires upfront planning about access patterns
  - Enables predictable performance at scale

- Relaxing constraints/transactions: not all applications need ACID transactions.

  Many internet applications:
  - Can tolerate eventual consistency
  - Can implement application-level compensation for inconsistencies

  By relaxing these requirements:
  - NoSQL databases can achieve higher throughput and better availability

  This doesn't mean NoSQL can't provide consistency—many systems offer tunable consistency that allows you to choose strong consistency when you need it and weaker consistency when you don't.

---

### NoSQL Taxonomy

NoSQL commonly falls into: while "NoSQL" is an umbrella term, different NoSQL databases serve fundamentally different use cases. Understanding these categories is crucial because the right choice depends entirely on your access patterns and requirements.

- **Key-Value**: `key -> blob/value` - the simplest NoSQL model, providing O(1) access to values by key.

  Key-value stores:
  - Treat values as opaque blobs—the database doesn't understand the structure of the data, it just stores and retrieves it
  - This simplicity enables extreme performance and scalability

  Use cases include:
  - Caching (Redis, Memcached)
  - Session storage
  - Configuration data
  - Simple object storage

  The tradeoff is limited query capabilities—you can only get by key or scan all keys. Complex queries require application-side processing or additional indexes.

- **Document**: JSON-like documents with indexes - document stores store hierarchical data structures (JSON, BSON, XML) as documents.

  Each document:
  - Can have a different structure, providing schema flexibility

  Document databases:
  - Automatically index fields within documents, enabling rich query capabilities including nested field queries and array operations

  Use cases include:
  - Content management
  - Catalogs
  - User profiles
  - Any data that naturally maps to hierarchical structures

  Examples include MongoDB, Couchbase, and RavenDB. The tradeoff is that document size limits exist (typically 16MB in MongoDB), and complex queries across many documents can be slow.

- **Wide-Column**: partition key + clustering keys + columns (Bigtable/Cassandra) - wide-column stores use a multi-dimensional map model: row key (partition key) + column key + timestamp maps to a value.

  This design:
  - Enables efficient range scans within a partition
  - Automatic sorting by clustering keys

  Wide-column stores excel at:
  - Time-series data
  - Logging
  - Any workload with predictable access patterns based on partition keys

  They:
  - Provide tunable consistency and linear scalability

  Examples include Cassandra, ScyllaDB, and Amazon DynamoDB. The tradeoff is complex data modeling—you must design your partition keys carefully to avoid hotspots, and joins are not supported.

- **Graph**: nodes/edges for relationship traversals - graph databases specialize in storing and querying relationships.

  Data is modeled as:
  - Nodes (entities) and edges (relationships) with properties

  They support graph traversal queries like:
  - "Find all friends of friends who live in San Francisco" which would require expensive recursive joins in relational databases

  Use cases include:
  - Social networks
  - Recommendation engines
  - Fraud detection
  - Network analysis

  Examples include Neo4j, Amazon Neptune, and ArangoDB. The tradeoff is that graph databases are specialized—don't use one if your primary access patterns are simple lookups or aggregations.

- **Search/Index**: inverted indexes optimized for full-text - search databases build inverted indexes that map terms to document IDs.

  This enables:
  - Fast full-text search
  - Faceted search
  - Relevance ranking

  They support complex queries including:
  - Fuzzy matching
  - Phrase queries
  - Boolean combinations

  Use cases include:
  - E-commerce product search
  - Log analysis
  - Document search

  Examples include Elasticsearch, OpenSearch, and Solr. The tradeoff is eventual consistency—indexing is asynchronous, so newly written data may not be immediately searchable.

- **Time-Series**: optimized for timestamped metrics/events - time-series databases are specialized for data indexed by time.

  They:
  - Use specialized compression algorithms that take advantage of the temporal nature of the data
  - Provide efficient downsampling and aggregation over time windows

  Use cases include:
  - Monitoring metrics
  - IoT sensor data
  - Financial tick data
  - Application performance monitoring

  Examples include InfluxDB, TimescaleDB, and Prometheus. The tradeoff is that they're specialized—don't use them for general-purpose data storage.

Each category makes different tradeoffs in storage layout and query capabilities.

The key insight is that:
- NoSQL is not "one thing"—it's a collection of specialized tools, each optimized for different problems
- Choosing the right NoSQL database requires understanding your access patterns and matching them to the appropriate category

---

### Key-Value Stores (Dynamo-style)

Examples: DynamoDB, Riak (historical patterns), many internal KV stores.

#### Core design
Key-value stores are built on a simple but powerful design that enables extreme scalability and performance.

- Data partitioned by key hash: the cluster uses consistent hashing to map keys to nodes.

  Each key:
  - Is hashed (using algorithms like MD5, SHA-1, or Murmur3)
  - The hash determines which node stores that key

  This:
  - Ensures an even distribution of keys across the cluster
  - When nodes are added or removed, consistent hashing minimizes the number of keys that must be moved—only keys whose hash falls into the affected ranges are remapped
  - This design allows the cluster to scale linearly by adding more nodes

- Replicated across nodes: each key-value pair is replicated to multiple nodes (the replication factor N) for fault tolerance.

  If a node fails:
  - The data is still available from other replicas

  Replication:
  - Can be synchronous (writes wait for replicas to acknowledge) or asynchronous (writes return immediately and replication happens in the background)
  - The replication strategy affects both durability and write latency
  - Most key-value stores allow you to configure the replication factor per table or globally

- Operations are simple: get/put - the simplicity of the key-value model is its strength.

  With only get and put operations (and sometimes delete):
  - The database can optimize heavily for these specific operations
  - There's no query optimization, no complex indexing, no joins

  This:
  - Simplicity enables predictable performance at scale
  - Some key-value stores add secondary indexes or conditional writes (compare-and-set), but the core model remains simple
  - This simplicity also makes the database easier to understand and operate

#### Consistency model
Often "tunable": key-value stores typically allow you to configure consistency on a per-operation basis.

This tunability lets you:
- Make explicit tradeoffs between consistency, latency, and availability based on your application's needs

- Read quorum `R`: when reading a key, the system reads from R replicas and returns the most recent value based on version information.

  A higher R:
  - Provides stronger consistency (more likely to see the latest write)
  - But increases read latency (must wait for more replicas)
  - And reduces availability (read fails if fewer than R replicas are available)

- Write quorum `W`: when writing a key, the system writes to W replicas before acknowledging success.

  A higher W:
  - Provides stronger durability (more replicas have the data) and consistency (more replicas see the write)
  - But increases write latency (must wait for more replicas)
  - And reduces availability (write fails if fewer than W replicas are available)

- Replication factor `N`: the total number of replicas for each key.

  A higher N:
  - Provides better fault tolerance (can tolerate more node failures)
  - But increases cost (more storage)
  - And can increase latency (more replicas to coordinate)

Rule of thumb:

- If `R + W > N`, you can achieve strong reads *for a single key* (under certain assumptions): this quorum condition ensures that any read quorum overlaps with any write quorum.

  When this holds:
  - A read will always see the most recent committed write because at least one replica from the read set must have participated in the most recent write
  - This provides linearizability for single-key operations, which is often sufficient for many applications

- If not, you get eventual consistency: when R + W ≤ N, it's possible for a read to return stale data because the read quorum might not include any replica that has the latest write.

  The system:
  - Will eventually converge to a consistent state as writes propagate to all replicas
  - But reads in the interim may see old values

  This eventual consistency model:
  - Provides lower latency and higher availability at the cost of temporary inconsistency

Internals you should know: key-value stores implement several sophisticated distributed systems mechanisms to provide scalability and fault tolerance.

- **Consistent hashing** distributes keys: consistent hashing is a technique for distributing keys across a changing set of nodes.

  Unlike naive modulo hashing (key % number_of_nodes):
  - Which would require moving almost all keys when nodes are added or removed

  Consistent hashing:
  - Maps both nodes and keys to a ring (a circular address space)
  - Each key is assigned to the next node clockwise on the ring
  - When a node is added, it only takes responsibility for keys in the arc between itself and its predecessor
  - This minimizes data movement during cluster changes
  - Virtual nodes (multiple points on the ring representing the same physical node) are often used to ensure even distribution of keys and hotspots

- **Vector clocks / versioning** (or last-write-wins) resolves concurrent writes: when two clients write to the same key concurrently, the system needs a way to resolve conflicts.

  Vector clocks:
  - Track the version history across replicas—each replica maintains a counter that increments on each write
  - The vector clock is the set of all replica counters
  - When reads return multiple conflicting versions, the client or application can resolve the conflict using business logic

  Simpler systems:
  - Use last-write-wins (LWW) based on timestamps, which is easier to implement but can lose data if clocks are skewed

  The choice between vector clocks and LWW represents a tradeoff between correctness and complexity.

- **Hinted handoff** and **anti-entropy** repair divergence: in eventually consistent systems, replicas can diverge due to network partitions or node failures.

  Hinted handoff:
  - Is a mechanism where if a target replica is down during a write, the coordinator stores a "hint" and later delivers the write when the replica recovers

  Anti-entropy:
  - Is a background process where replicas compare their data and reconcile differences
  - Typically using Merkle trees to efficiently detect which keys differ

  These mechanisms ensure the system eventually converges to a consistent state even after temporary failures or network partitions.

Tradeoffs:

- Great scalability: key-value stores excel at horizontal scaling.

  The simple data model:
  - And lack of complex operations allow them to scale linearly with the number of nodes

  Consistent hashing:
  - Enables easy addition and removal of nodes with minimal data movement

  This scalability makes key-value stores ideal for workloads that need to handle massive throughput across large datasets, such as:
  - Session stores
  - Caching layers
  - Simple object storage

- Limited query patterns: the simplicity of key-value stores is also their limitation.

  You can:
  - Only retrieve data by key (or scan all keys, which is expensive)

  There are:
  - No secondary indexes in pure key-value stores (though some like DynamoDB add them)
  - No ad-hoc queries
  - No joins

  If you need to query by attributes other than the primary key:
  - You must either maintain your own indexes in application code or use a different data model

  This limitation requires careful data modeling upfront—you must know your access patterns before designing your keys.

- Multi-item transactions are hard: most key-value stores don't support multi-key transactions, especially across different partitions.

  DynamoDB:
  - Added transaction support for up to 25 items, but this is an exception

  Without transactions:
  - Applications must implement compensation logic for operations that involve multiple keys

  For example:
  - Transferring funds between two accounts requires updating both keys atomically, which key-value stores can't guarantee

  This limitation often means key-value stores are used for simpler use cases or combined with other systems for transactional requirements.

---

### Document Stores (MongoDB-style)

Examples: MongoDB, Couchbase.

#### Data model
Documents are nested objects (JSON/BSON): document stores model data as hierarchical structures rather than flat tables.

A document can contain:
- Nested objects
- Arrays
- Primitive types

All within a single record. This document model:
- Closely matches how applications represent data in memory
- Reducing the object-relational impedance mismatch that exists with relational databases

BSON (Binary JSON):
- Adds support for additional data types like dates, binary data, and ObjectId that aren't native to JSON

Great for:

- Aggregate-style entities (user profile, product with variants): the document model excels when an entity naturally contains all its related data.

  A user profile document might include:
  - The user's contact information
  - Preferences
  - Recent activity
  - Social connections—all in one document

  A product document might include:
  - The product details
  - Pricing
  - Inventory
  - Variants (size, color)
  - Reviews

  This aggregation:
  - Eliminates the need for joins when fetching an entity and its related data

  The key is to design documents around how you access data—if you always fetch a user and their profile together, store them in the same document.

- Evolving schemas: document databases are schemaless or have flexible schemas.

  Different documents in the same collection:
  - Can have different fields or structures

  You can:
  - Add new fields to documents without requiring a schema migration—old documents simply lack the new field, and new documents include it

  This flexibility:
  - Accelerates development velocity, especially in rapidly evolving products
  - However, this flexibility requires discipline: the application must handle missing or unexpected fields gracefully

  Some document databases (like MongoDB):
  - Added schema validation features to enforce structure when needed, providing a balance between flexibility and control

#### Indexing and querying
Document DBs often support: while document stores are schemaless, they still need efficient query capabilities.

Most document databases provide indexing and query features that bridge the gap between the flexibility of documents and the need for structured queries.

- Secondary indexes: you can create indexes on any field within documents, including nested fields.

  Unlike relational databases where indexes are defined on table columns:
  - Document store indexes are defined on document fields

  When you query on an indexed field:
  - The database can use the index to avoid scanning the entire collection

  This enables efficient queries like:
  - "Find all users where status is 'active'"
  - "Find all products where price < 100"

  Secondary indexes:
  - Are essential for performance but add write overhead—each write must update all relevant indexes

- Compound indexes: compound indexes index multiple fields together, enabling efficient queries that filter on multiple fields.

  For example:
  - A compound index on (status, created_at) can efficiently answer queries like "find all active users created in the last week"

  The order of fields in a compound index matters:
  - Queries must use the fields in prefix order to use the index

  Document stores also:
  - Support multikey indexes on array fields, creating an index entry for each element in the array

- Limited joins (or special operators): early document stores avoided joins entirely, requiring denormalization.

  Modern document stores like MongoDB:
  - Added $lookup for joining collections, but these joins are typically less efficient than relational joins because they're executed in the application layer or require memory-intensive operations

  Instead of joins:
  - Document stores encourage embedding related data within documents

  For relationships that can't be embedded (like many-to-many):
  - Applications often perform application-side joins or use special operators like $graphLookup for graph traversals

  The philosophy is to design documents around query patterns to avoid joins.

Internals (typical): document stores borrow many techniques from relational databases while adapting them for the document model.

- Storage engine (e.g., WiredTiger) often uses B-Trees or LSM-like structures: MongoDB's WiredTiger storage engine uses B-Trees for indexing and document storage, similar to relational databases.

  Some document stores:
  - Use LSM (Log-Structured Merge) trees, which optimize for write-heavy workloads by buffering writes in memory and periodically flushing to disk in sorted runs

  The choice between B-Trees and LSM trees:
  - Represents a tradeoff between read and write performance—B-Trees provide better read performance for random lookups, while LSM trees provide better write throughput

  Document size limits (typically 16MB in MongoDB):
  - Exist because very large documents can cause performance issues and complicate storage engine operations

- Journaling/WAL for durability: like relational databases, document stores use write-ahead logging for durability.

  Before modifying data on disk:
  - The database writes a log record describing the change

  On crash:
  - The database replays the log to recover committed writes

  MongoDB calls this journaling:
  - It can be configured for different durability levels (from acknowledging writes after journal write to acknowledging after data is flushed to disk)

  The WAL approach:
  - Provides the same benefits as in relational databases—crash recovery and the ability to flush data pages lazily

- Replication via oplog: document stores typically replicate using an operation log (oplog).

  The primary:
  - Records all write operations in a capped collection (the oplog)
  - Secondaries tail this log and apply the operations in order

  This:
  - Is similar to WAL-based replication in relational databases but operates at a higher level of abstraction (logical operations rather than physical page changes)
  - The oplog enables secondaries to catch up after being disconnected and provides a point-in-time recovery mechanism
  - Replication in document stores is typically primary-replica with automatic failover, similar to relational databases

Tradeoffs:

- Flexible schema helps iteration: the ability to evolve data structures without migrations accelerates development, especially in early-stage products where requirements change frequently.

  Developers can:
  - Add new fields, change field types, or restructure documents without coordinating with database administrators or scheduling downtime

  This flexibility:
  - Also enables polyglot persistence—different microservices can use different document structures for the same collection if needed
  - However, this flexibility requires discipline to avoid schema chaos over time

- But you must enforce many constraints in application code: without schema enforcement, the database won't prevent invalid data types, missing required fields, or referential integrity violations.

  The application:
  - Must validate all data before writing it, which adds complexity and the risk of bugs

  For example:
  - The database won't enforce that email addresses are unique or that foreign key references are valid

  Some document stores added validation features (MongoDB's schema validation, Couchbase's N1QL) to address this:
  - But the default is still flexibility over enforcement
  - This constraint enforcement burden shifts from the database to the application layer

- Joins and complex aggregations can be costlier than in SQL: while document stores support aggregation pipelines and some join operations, these are typically less efficient than relational database operations.

  Aggregations:
  - Often require materializing entire documents into memory
  - Joins are performed by the application or require expensive lookups

  For complex analytical queries that involve joining multiple collections and performing multi-stage aggregations:
  - Relational databases with their sophisticated query optimizers are often more efficient

  Document stores shine:
  - For operational workloads where you fetch individual entities or simple aggregates, not for complex analytical queries

---

### Wide-Column Stores (Cassandra/HBase-style)

Examples: Cassandra, HBase, Bigtable-like systems.

#### Mental model
Data is organized by: wide-column stores use a multi-dimensional map model where data is organized by partition keys and clustering keys.

This design:
- Is fundamentally different from relational tables
- Requires thinking about data modeling in terms of access patterns

- **Partition key** (decides which node stores the row): the partition key is hashed to determine which node in the cluster stores the data.

  All rows with the same partition key:
  - Are stored together on the same node

  This is the most important design decision in wide-column stores:
  - You must choose partition keys that distribute data evenly across the cluster (to avoid hotspots) while grouping data that's accessed together

  For example:
  - In a time-series application, you might partition by (device_id, date) so all readings from a device on a given date are stored together and can be fetched efficiently

- **Clustering key** (sort order inside partition): within a partition, rows are sorted by the clustering key.

  This sorting:
  - Is physical—data is stored on disk in clustering key order, enabling efficient range scans

  For example:
  - If your clustering key is timestamp, all rows in a partition are stored in timestamp order
  - Querying for a time range becomes a sequential disk read rather than random I/O

  You can:
  - Have multiple clustering key components, creating a hierarchical sort order (e.g., (date, timestamp) sorts by date first, then by timestamp within each date)

This is a powerful design because it makes common queries extremely fast: by designing your partition and clustering keys around your query patterns, you can achieve predictable performance that scales linearly.

The database:
- Doesn't need to sort or scan irrelevant data
- It goes directly to the partition and reads sequentially through the clustering key range

- Read by partition key: fetching all data for a specific partition key is a single coordinated read from the relevant node(s).

  This:
  - Is O(1) complexity regardless of how much data is in other partitions

  For example:
  - Fetching all readings for device_123 on 2024-01-15 is efficient regardless of whether you have 100 devices or 1 million devices

- Scan a contiguous range of clustering keys: because data is physically sorted by clustering key, range scans are efficient sequential reads.

  Fetching all readings between 10:00 and 11:00 for a device:
  - Reads a contiguous range on disk

  This is why:
  - Wide-column stores excel at time-series data, logging, and any workload with predictable range queries

Internals: wide-column stores are optimized for write-heavy workloads using storage structures that differ from traditional B-Tree databases.

- Typically LSM-tree storage: Log-Structured Merge (LSM) trees optimize for write throughput by buffering writes in memory and periodically flushing sorted runs to disk.

  Unlike B-Trees where updates modify existing pages in place:
  - LSM trees are append-only—new writes are always added to new files

  This:
  - Eliminates random disk I/O for writes, making them sequential and much faster
  - Reads must check multiple sorted runs (memory table + disk files) and merge results, which can be slower than B-Tree reads

  This read-write tradeoff:
  - Is intentional—LSM trees sacrifice some read performance for superior write performance and compression

- Append-heavy writes: the append-only nature of LSM trees makes them ideal for write-heavy workloads like logging, time-series data, and event sourcing.

  Writes:
  - Are always appended, never updated in place
  - Even updates are written as new versions, with old versions marked for deletion during compaction

  This append-only pattern:
  - Enables efficient compression because similar data is stored contiguously
  - It also simplifies replication—replicas can simply replicate the append-only log of operations

  The downside:
  - Storage can grow unbounded without compaction, and deleted data isn't reclaimed until compaction runs

- Compaction merges files: over time, the number of sorted files grows, which slows down reads (more files to check) and wastes space (multiple versions of the same key).

  Compaction:
  - Merges multiple sorted files into fewer, larger files, removing deleted data and combining versions
  - This is similar to the merge phase of merge sort

  Compaction:
  - Is I/O-intensive and can impact performance while running
  - So databases tune compaction strategies (size-tiered vs. leveled compaction) based on workload characteristics

  The compaction process:
  - Is critical for maintaining read performance and reclaiming space in LSM-based systems

Tradeoffs:

- You must design tables around queries: wide-column stores require query-driven data modeling.

  Unlike relational databases where you design a normalized schema and then write queries:
  - In wide-column stores you must know your query patterns upfront and design your partition and clustering keys accordingly

  If your access patterns change:
  - You may need to redesign your tables and migrate data

  This:
  - Upfront planning is a barrier to ad-hoc querying but enables predictable performance at scale
  - The mental model shift—from data modeling to access pattern modeling—is significant for teams transitioning from relational databases

- Secondary indexes are limited/expensive: secondary indexes in wide-column stores are fundamentally different from relational indexes.

  Because data is partitioned:
  - A secondary index must either be local (indexing only data within a partition) or global (indexing across all partitions)

  Local indexes:
  - Are efficient for queries that include the partition key but useless for queries that don't

  Global indexes:
  - Enable queries by non-partition keys but require cross-node coordination and are expensive to maintain

  Cassandra's secondary indexes, for example:
  - Are generally not recommended for high-cardinality data because they can cause hotspotting and performance issues

  The recommended approach:
  - Is to denormalize data into multiple tables with different partition keys optimized for different queries

- Cross-partition aggregations are hard: wide-column stores excel at single-partition queries but struggle with operations that require coordinating across partitions.

  Aggregations that span multiple partitions (like "count all users worldwide"):
  - Require querying multiple nodes and combining results, which is slow and complex

  There's:
  - No built-in distributed query optimizer like in relational databases

  Applications:
  - Must implement these aggregations themselves or use external systems like Spark for analytics

  This limitation:
  - Means wide-column stores are best for operational workloads where queries are scoped to a partition, not for analytical workloads that require aggregating across the entire dataset

---

### Graph Databases

Examples: Neo4j, JanusGraph.

Graph is best when your primary workload is: graph databases are specialized for workloads where relationships between entities are as important as the entities themselves.

They:
- Excel at queries that would require expensive recursive joins in relational databases

- Deep relationship traversals: queries like "find all friends of friends of friends who live in San Francisco" require traversing multiple levels of relationships.

  In a relational database:
  - This requires recursive queries or multiple self-joins, which are expensive and complex

  Graph databases:
  - Store relationships as first-class citizens (edges) with direct pointers between nodes, making traversals efficient
  - A traversal from node to adjacent node is an O(1) pointer dereference rather than a join operation

  This:
  - Makes graph databases orders of magnitude faster for deep traversals

- Variable-length path queries: many real-world queries have variable depth—find the shortest path between two nodes, find all nodes within N hops, find cycles in the graph.

  Relational databases:
  - Struggle with these queries because they require dynamic SQL generation or recursive CTEs with unpredictable performance

  Graph databases:
  - Have built-in traversal algorithms (BFS, DFS, shortest path) optimized for these patterns
  - Cypher (Neo4j's query language) and Gremlin provide natural syntax for expressing path queries

- Recommendations, social graphs, network topology: these domains are inherently graph-structured.

  Social networks:
  - Friends, followers

  Recommendation engines:
  - Users, products, purchases

  Fraud detection:
  - Accounts, transactions, relationships

  Network management:
  - Routers, links, paths

  All involve complex relationship patterns. Graph databases:
  - Model these domains naturally without the impedance mismatch of mapping graphs to tables
  - The ability to efficiently traverse relationships enables powerful queries like "recommend products purchased by people who purchased what you purchased" or "detect circular money laundering patterns"

Internals: graph databases use storage structures optimized for relationship traversals, which is fundamentally different from the row-based storage of relational databases.

- Storage is optimized to follow pointers from node -> edges -> adjacent nodes: in a graph database, each node stores direct references (pointers) to its adjacent edges, and each edge stores references to its source and target nodes.

  This:
  - Means traversing from a node to its neighbors is a simple pointer dereference operation, not a join
  - The storage layout physically co-locates related nodes and edges when possible, reducing disk seeks during traversals

  Some graph databases:
  - Use native graph storage (Neo4j), while others layer graph capabilities on top of other storage engines (JanusGraph on top of Cassandra/HBase)

  The key principle:
  - Relationships are stored as first-class objects with direct pointers, not as foreign keys that require lookups

- Index-free adjacency is a common idea (fast traversals): index-free adjacency means that the database can navigate from node to node without using indexes to find relationships.

  Instead of maintaining a global index on relationships:
  - Each node maintains a list of its adjacent relationships

  This:
  - Eliminates the index lookup overhead during traversals
  - When you traverse from node A to node B, the database follows the direct pointer stored in node A's adjacency list, not an index lookup
  - This makes traversals O(1) per hop regardless of graph size

  The tradeoff:
  - Index-free adjacency requires more memory (each node stores references to all its relationships)
  - And can make some operations (like finding all nodes with a specific property) slower, requiring full scans or property indexes

Tradeoffs:

- Great for traversals: graph databases excel at relationship-heavy workloads.

  Queries that involve navigating complex relationship patterns:
  - Social network analysis
  - Recommendation engines
  - Fraud detection
  - Dependency analysis

  These are dramatically faster in graph databases than in relational databases. The pointer-based traversal model:
  - Enables performance that scales with the depth of traversal rather than the size of the dataset

  For these use cases:
  - Graph databases are often the only practical solution at scale

- Less ideal for high-throughput OLTP aggregates and simple key-based lookups at massive scale: graph databases are specialized tools.

  If your primary workload is:
  - Simple CRUD operations
  - Aggregations on large datasets
  - Key-value lookups

  A relational database or key-value store:
  - Will likely perform better and be simpler to operate

  Graph databases:
  - Typically have higher overhead per operation than simpler databases because they maintain complex graph structures and index-free adjacency lists
  - They also often have weaker horizontal scaling characteristics than key-value or wide-column stores

  The rule of thumb:
  - Use a graph database when relationships are the primary complexity in your workload; otherwise, use a more general-purpose database

---

### Search/Index Engines

Examples: Elasticsearch/OpenSearch, Solr.

Internals: search engines are built on inverted indexes, a data structure optimized for full-text search that's fundamentally different from the B-Tree indexes used in relational databases.

- Inverted index maps terms -> documents: an inverted index is conceptually similar to the index at the back of a book—for each term (word), it stores a list of documents containing that term.

  When you search for "database":
  - The index immediately returns all documents containing that word without scanning the documents themselves

  The index also:
  - Stores position information (where in the document the term appears), enabling phrase queries ("database design") and proximity searches (terms within N words of each other)

  This structure:
  - Enables fast full-text search but requires significant storage overhead—the index can be larger than the documents themselves

- Tokenization, analyzers, scoring models (BM25): before indexing, documents are processed through analyzers that tokenize text into terms, apply normalization (lowercasing, stemming, removing stop words), and handle language-specific rules.

  This processing:
  - Ensures that variations of words match (e.g., "running", "runs", "ran" might all map to "run")

  Scoring models like BM25:
  - Rank results by relevance based on term frequency (how often the term appears in the document) and inverse document frequency (how rare the term is across all documents)

  This ranking:
  - Is crucial for user experience—finding the most relevant documents rather than all matching documents

  Different analyzers and scoring models:
  - Can be configured per field to optimize for specific use cases
- Segments, merges, refresh cycles: search engines like Elasticsearch use Lucene under the hood, which organizes indexes into immutable segments.

  When documents are indexed:
  - They're written to an in-memory buffer
  - Periodically (based on time or size), this buffer is flushed to disk as a new segment

  Over time:
  - The number of segments grows, which would slow down searches (more segments to check)
  - Compaction/merge processes periodically merge smaller segments into larger ones, improving search performance and reclaiming space
  - This merge process is similar to LSM tree compaction

  Refresh cycles:
  - Make newly indexed documents searchable—by default, Elasticsearch refreshes every second, making documents visible
  - This near-real-time behavior is a key characteristic of search engines, distinguishing them from traditional databases with immediate consistency

Tradeoffs:

- Phenomenal full-text and filtering: search engines provide unmatched capabilities for full-text search, fuzzy matching, faceted navigation, and relevance ranking.

  Complex queries:
  - That would be difficult or impossible in SQL (like "find documents containing 'database' within 5 words of 'performance' with a relevance score above 0.5") are natural in search engines

  They also:
  - Excel at filtering and aggregations across large datasets, making them popular for log analysis and observability

  The combination:
  - Of full-text search, structured filtering, and aggregations makes search engines versatile for many use cases

- Not a primary source of truth unless carefully managed: search engines are eventually consistent by design—indexing is asynchronous, and there's no guarantee that the search index matches the primary data source at any given moment.

  Search engines:
  - Can lose documents during failures, and they don't provide ACID transaction guarantees

  For these reasons:
  - They're typically used as secondary indexes alongside a primary database (like PostgreSQL or MySQL)
  - The application writes to the primary database and then asynchronously updates the search index

  This pattern:
  - Requires careful handling of failures and consistency—what happens if the write to the primary succeeds but the index update fails?
  - The application must implement reconciliation logic or accept temporary inconsistency

- Near-real-time semantics: search engines provide near-real-time visibility for newly indexed data (typically 1-2 second delay in Elasticsearch).

  This:
  - Is sufficient for most use cases but not for systems that require immediate consistency

  The refresh interval:
  - Is tunable—more frequent refreshes increase visibility but add performance overhead

  This near-real-time model:
  - Is a tradeoff that enables high write throughput while keeping search performance acceptable

  Applications:
  - Must be designed to tolerate this delay—for example, a user might create a document and immediately search for it, but it won't appear until the next refresh cycle

---

### Time-Series Databases

Examples: InfluxDB, TimescaleDB (Postgres extension), Prometheus TSDB.

Time-series is optimized for: time-series databases are specialized for data indexed by time, such as metrics, events, and sensor readings.

The temporal nature of this data:
- Enables optimizations that general-purpose databases cannot provide

- High ingest rates: time-series workloads are typically write-heavy—thousands or millions of data points per second from sensors, applications, or infrastructure.

  Time-series databases:
  - Optimize for high write throughput using techniques like batch writes, append-only storage, and write-optimized data structures

  Unlike relational databases:
  - Where each write might require updating multiple indexes, time-series databases often have a simple primary key (timestamp + tags) that enables efficient batching

  This optimization:
  - Allows them to handle ingestion rates that would overwhelm general-purpose databases

- Time-window queries: the most common query pattern in time-series data is retrieving data within a time range (e.g., "CPU usage for server X in the last hour").

  Time-series databases:
  - Optimize for this by physically organizing data by time and using time-based partitioning
  - A query for a time range becomes a sequential read of contiguous time-partitioned files, which is much faster than random I/O
  - Range scans by time are efficient because data is stored in chronological order

  Some databases:
  - Also use time-based indexes or specialized data structures (like Gorilla compression for floating-point time series) to accelerate time-window queries

- Downsampling and retention policies: time-series data often follows a hierarchy of detail—recent data is needed at high resolution, while older data can be downsampled to lower resolution.

  Time-series databases:
  - Provide built-in mechanisms for automatic downsampling (aggregating raw data into averages, minimums, maximums over time windows) and retention (deleting old data)

  For example:
  - You might keep raw data for 7 days, 5-minute averages for 90 days, and hourly averages for 1 year

  These policies:
  - Are configured once and applied automatically, reducing operational complexity compared to implementing this logic in application code

Internals vary: different time-series databases use different approaches, but they share common optimization strategies for temporal data.

- Columnar compression: time-series data often has repetitive patterns—metric values change slowly, and many timestamps share the same tags (like host, region, service).

  Columnar storage:
  - Stores each column separately, enabling compression algorithms that exploit these patterns

  For example:
  - Gorilla compression (developed by Facebook for Prometheus monitoring) achieves 10x compression for floating-point time series by encoding deltas between successive values rather than storing raw values
  - Dictionary encoding compresses tag values by mapping repeated strings to integers

  This columnar approach:
  - Also enables efficient queries that only read the columns they need, skipping irrelevant data

- Time-partitioned chunks: time-series databases typically partition data into time-based chunks (e.g., one chunk per hour or day).

  Each chunk:
  - Contains all data for that time period

  This design:
  - Enables efficient time-range queries—the database can quickly identify which chunks overlap with the query range and read only those chunks
  - Old chunks can be dropped entirely for retention without scanning individual records
  - Time partitioning also simplifies downsampling—when a chunk ages out, it can be aggregated into a lower-resolution chunk

  This chunking strategy:
  - Is similar to how wide-column stores partition by partition key, but optimized specifically for time ranges

- Write-optimized logs: many time-series databases use append-only write patterns similar to LSM trees.

  New data points:
  - Are always appended, never updated in place (except for corrections)

  This append-only pattern:
  - Enables sequential writes, which are much faster than random writes
  - Some databases use write-ahead logs or in-memory buffers that periodically flush to disk
  - The immutable nature of time-series data (once written, timestamps don't change) makes this append-only approach natural

  This design:
  - Trades some read efficiency (may need to merge multiple sources) for superior write performance

Tradeoffs:

- Excellent for metrics/events: time-series databases provide superior performance and storage efficiency for their target workloads.

  The combination:
  - Of high ingest rates, efficient time-range queries, built-in downsampling, and automatic retention makes them ideal for monitoring, IoT, observability, and any application dealing with timestamped data

  The specialized data model and optimizations:
  - Enable capabilities that would be difficult or expensive to implement on top of a general-purpose database

  For these use cases:
  - Time-series databases are often the best choice

- Not a general-purpose OLTP store: time-series databases are specialized tools with significant limitations outside their domain.

  They typically:
  - Don't support complex transactions, joins, or ad-hoc queries well
  - The data model is rigid—everything must be indexed by time, which doesn't fit all use cases
  - Updating or deleting historical data is often difficult or inefficient

  If your application needs:
  - General-purpose CRUD operations, complex relationships, or flexible querying, a relational database or document store will be more appropriate

  Time-series databases:
  - Are best used as a specialized component alongside a primary database, not as a general-purpose data store

---

### Common NoSQL Storage Internals: LSM Trees

Many NoSQL systems (and some SQL ones) use **Log-Structured Merge Trees (LSM)**.

#### Why LSM exists
B-Trees are great for reads and range scans, but random writes can be expensive.

When you update a B-Tree:
- You must locate the specific page containing the key, read it, modify it, and write it back
- If the page doesn't fit, it splits, causing additional writes

On HDDs:
- Random writes require disk seeks (5-10ms each), making them orders of magnitude slower than sequential writes

Even on SSDs:
- Random writes cause wear and have lower throughput than sequential writes

For write-heavy workloads like time-series data, logging, or event streaming:
- This random write overhead becomes a bottleneck

LSM turns random writes into sequential writes: Log-Structured Merge trees solve this by making all writes sequential.

Instead of modifying existing data in place:
- LSM trees always append new data

This:
- Eliminates random I/O for writes, dramatically improving write throughput

- Writes go to an in-memory structure (memtable): when a write arrives, it's added to an in-memory sorted structure (typically a skip list or red-black tree).

  This in-memory write:
  - Is fast—no disk I/O required
  - The memtable maintains data in sorted order, which enables efficient range queries even before data is flushed to disk
  - When the memtable reaches a size threshold, it's frozen and a new memtable is created for incoming writes

- Flushed to disk as immutable sorted files (SSTables): the frozen memtable is written to disk as an SSTable (Sorted String Table)—an immutable file containing sorted key-value pairs.

  Because the data is already sorted in memory:
  - Flushing is a sequential write operation

  SSTables:
  - Are immutable—once written, they're never modified
  - This immutability simplifies concurrency (no locks needed for reading) and crash recovery (no need to undo partial writes)
  - Old versions of data remain in older SSTables until cleaned up by compaction

- Background **compaction** merges files to keep read performance reasonable: as more SSTables accumulate, reads must check multiple files to find the latest value for a key.

  Compaction:
  - Merges multiple SSTables into fewer, larger files, removing overwritten data and combining versions
  - This is similar to the merge phase of merge sort
  - Compaction improves read performance (fewer files to check) and reclaims space (removing old versions)

  The compaction strategy (size-tiered vs. leveled):
  - Is a key tuning parameter that affects write amplification and read performance

Key internal components: LSM trees consist of several cooperating components that work together to provide the write-optimized storage model.

- **Memtable**: in-memory sorted map - the memtable is the write buffer where all incoming writes are first stored.

  It's typically:
  - Implemented as a skip list or red-black tree for O(log n) inserts and efficient range scans

  The memtable:
  - Keeps data sorted, which enables efficient range queries even before data is flushed to disk
  - When the memtable reaches a configurable size threshold (e.g., 64MB in RocksDB), it becomes immutable and a new memtable is created for incoming writes

  The size threshold:
  - Is a tuning parameter—larger memtables improve write efficiency (fewer flushes) but increase memory usage and recovery time

- **Write-ahead log**: protects memtable writes - since the memtable is in-memory, a crash before flushing would lose data.

  The write-ahead log (WAL):
  - Records all writes before they're applied to the memtable
  - On recovery, the database replays the WAL to reconstruct the memtable
  - The WAL is typically append-only for sequential writes

  Some systems:
  - Support disabling the WAL for pure write-through workloads where durability is less critical than latency

  The WAL:
  - Can be truncated once the corresponding memtable is flushed to disk

- **SSTables**: immutable sorted files - SSTables (Sorted String Tables) are the on-disk storage format.

  Each SSTable:
  - Contains sorted key-value pairs and is immutable once written
  - SSTables often include indexes for fast key lookup and block-based storage for efficient reads

  Because SSTables are immutable:
  - They can be read without locks, enabling high read concurrency
  - SSTables may also include bloom filters (see below) to avoid unnecessary disk reads

  The file format:
  - Is typically self-describing, containing metadata about compression, bloom filters, and block indexes

- **Bloom filters**: avoid unnecessary disk reads - a bloom filter is a probabilistic data structure that can quickly determine whether a key might exist in an SSTable.

  Before reading an SSTable:
  - The database checks the bloom filter
  - If the filter says the key doesn't exist, the disk read is skipped entirely

  Bloom filters:
  - Can have false positives (saying a key exists when it doesn't) but never false negatives

  This optimization:
  - Dramatically reduces disk I/O for point lookups, especially when many SSTables exist

  The tradeoff:
  - Is memory usage—bloom filters require additional memory per SSTable

- **Compaction**: merges SSTables; controls amplification - compaction is the process of merging multiple SSTables into fewer, larger files.

  There are two main strategies:
  - Size-tiered compaction (merge similar-sized SSTables when there are enough of them)
  - Leveled compaction (maintain a hierarchy of levels, each with a target size)

  Compaction:
  - Removes overwritten data, combines versions, and improves read performance
  - The compaction strategy is a critical tuning parameter—aggressive compaction improves read performance but increases write amplification (more data rewritten)
  - Lazy compaction reduces write overhead but degrades read performance

Tradeoffs (very important): LSM trees optimize for writes at the cost of amplification in other dimensions.

Understanding these tradeoffs:
- Is critical for tuning LSM-based systems

- **Write amplification**: compaction rewrites data multiple times - in a B-Tree, each write typically modifies one page (one disk write).

  In an LSM tree:
  - A single logical write may be rewritten multiple times: first when flushed from memtable to SSTable, then during compaction when SSTables are merged

  Write amplification:
  - Is the ratio of bytes written to disk vs. bytes written by the application
  - A write amplification of 10 means 10 bytes are written for every 1 byte of application data
  - This increases disk wear on SSDs and consumes I/O bandwidth

  Write amplification:
  - Is controlled by compaction strategy—leveled compaction typically has higher write amplification than size-tiered compaction but provides more predictable read performance

- **Read amplification**: reads may check multiple SSTables - to read a key, an LSM tree must check the memtable, then potentially multiple SSTables in reverse chronological order (newest first) until the key is found.

  Read amplification:
  - Is the number of disk reads per logical read
  - With many SSTables, a read might check dozens of files
  - Bloom filters help by avoiding unnecessary SSTable reads, but they don't eliminate the problem entirely
  - Read amplification increases as compaction falls behind

  This:
  - Is why LSM trees often have worse read latency than B-Trees for point lookups, especially for cold data that's been compacted less frequently

- **Space amplification**: duplicates exist until compaction finishes - because LSM trees never overwrite data in place, old versions persist until compaction removes them.

  Space amplification:
  - Is the ratio of total storage vs. useful data
  - A space amplification of 2 means you're using 2x the storage for your actual data

  This happens because:
  - Updates create new versions while old versions remain, and deleted data isn't immediately reclaimed
  - Space amplification is controlled by compaction frequency—more aggressive compaction reduces space amplification but increases write amplification

  This:
  - Is a fundamental tension in LSM trees: you can optimize for write performance, space efficiency, or read performance, but not all three simultaneously

LSM systems can be amazing for write-heavy workloads, but require tuning and careful operational management.

---

## Consistency, CAP, and the Real World (PACELC)

### CAP theorem (practical interpretation)
In the presence of a network partition, a distributed system must choose between: the CAP theorem states that a distributed system can provide at most two of three guarantees: Consistency, Availability, and Partition tolerance.

In practice:
- Partition tolerance is not optional—network partitions are inevitable in real distributed systems

So the real choice:
- Is between consistency and availability during a partition

- **Consistency (C)**: every read sees the latest write (strong) - in a consistent system, all nodes see the same data at the same time.

  When you write a value and immediately read it:
  - You get the value you just wrote

  Achieving strong consistency during a partition:
  - Requires rejecting writes or reads that cannot be guaranteed to be consistent

  For example:
  - If a partition isolates some nodes from the primary, a CP system would reject writes on the isolated nodes rather than allow inconsistent state
  - This ensures correctness but reduces availability

- **Availability (A)**: every request receives a response (not necessarily latest) - in an available system, every request receives a (non-error) response, without guarantee that it contains the most recent write.

  During a partition:
  - An AP system continues accepting writes on all nodes, even if those writes cannot be synchronized with other partitions
  - The system may return stale data or allow concurrent writes to diverge, but it remains operational

  This:
  - Maximizes uptime but risks inconsistency

Partition tolerance is not optional in real distributed systems: network partitions are inevitable—routers fail, network cables are cut, data centers lose connectivity.

Any system:
- That runs across multiple machines must handle partitions
- The question is not whether partitions occur, but how the system behaves when they do

This:
- Makes the CAP theorem's choice between C and A during partitions the fundamental design decision for distributed databases

Common real-world positions:

- "CP" systems: prefer correctness over availability under partition (often consensus-based).

  Systems like HBase, MongoDB (with strong consistency), and traditional relational databases with synchronous replication:
  - Prioritize consistency

  During a partition:
  - They may become unavailable for some operations to prevent inconsistency
  - These systems use consensus protocols (Paxos, Raft) to ensure that only one partition can make progress

  This:
  - Is appropriate for financial systems, inventory management, and other use cases where correctness is critical

- "AP" systems: prefer availability and eventual convergence (often Dynamo-style).

  Systems like Cassandra, DynamoDB (eventual consistency mode), and Couchbase:
  - Prioritize availability

  During a partition:
  - All partitions continue accepting writes, which may diverge
  - When the partition heals, the system resolves conflicts through mechanisms like last-write-wins, vector clocks, or application-level reconciliation

  This:
  - Is appropriate for social media feeds, shopping carts, and other use cases where availability is more important than immediate consistency

### PACELC
CAP talks about partitions, but most of the time there is no partition. PACELC says: the PACELC theorem extends CAP by recognizing that partitions are rare in normal operation.

When the system is operating normally (no partition):
- There's still a fundamental tradeoff between latency and consistency

- **If Partition (P)** occurs: choose **Availability (A)** or **Consistency (C)** - this is the same choice as CAP.

  During a partition:
  - You must choose between staying available (accepting potential inconsistency) or staying consistent (rejecting some requests)
  - This is the emergency behavior during failures

- **Else (E)**: choose **Latency (L)** or **Consistency (C)** - when there's no partition, the tradeoff shifts.

  You can have strong consistency:
  - But it typically requires coordination (quorum reads/writes, consensus protocols) which increases latency

  Or you can have low latency:
  - By relaxing consistency (reading from local replicas, asynchronous replication)

  This:
  - Is the normal operating behavior

Meaning:

- Even without partitions, stronger consistency often increases latency (quorum/coordination): to achieve strong consistency, a system must typically coordinate across multiple replicas before acknowledging a write.

  For example:
  - In a system with R+W>N quorum, a write must wait for W replicas to acknowledge, which involves network round trips
  - A read must wait for R replicas and compare results
  - This coordination adds latency

  In contrast:
  - A system that prioritizes latency might acknowledge writes immediately after writing locally (asynchronous replication) or serve reads from the nearest replica without checking other replicas
  - This provides lower latency but risks returning stale data

  The PACELC framework:
  - Makes explicit that the consistency-latency tradeoff exists even in normal operation, not just during partitions

### Consistency models you should know
Consistency models define the guarantees a system provides about when and how writes become visible to reads.

Understanding these models:
- Is crucial for choosing the right database and designing correct applications

- **Strong consistency**: read returns most recent committed write - also known as linearizable consistency, this is the strongest model.

  Every read:
  - Returns the most recent committed write, as if there were a single copy of the data
  - If you write value X and immediately read, you will see X

  This model:
  - Provides the illusion of a single, atomic data store, which simplifies application reasoning

  However:
  - Achieving strong consistency typically requires coordination across replicas (quorum reads/writes, consensus protocols), which increases latency and reduces availability during partitions
  - Most traditional relational databases provide strong consistency by default

- **Linearizability**: operations appear to occur atomically in real-time order - linearizability is a formal definition of strong consistency.

  Operations:
  - Appear to happen instantaneously at some point between their invocation and response
  - All operations appear to happen in a total order that respects real-time
  - If operation A completes before operation B starts, then A appears to happen before B in the total order

  Linearizability:
  - Is the gold standard for consistency but comes with significant performance costs
  - It requires synchronizing across all replicas for each operation

- **Sequential consistency**: same order for all, but not necessarily real-time - sequential consistency is slightly weaker than linearizability.

  All processes:
  - See the same order of operations, but that order doesn't have to match real-time
  - If operation A completes before operation B starts, B might appear to happen before A in the sequential order

  Sequential consistency:
  - Is easier to implement than linearizability because it doesn't require real-time synchronization
  - But it still requires a global order that all replicas agree on

- **Causal consistency**: respects cause/effect order - causal consistency ensures that operations that are causally related are seen by all processes in the same order.

  If operation A causally depends on operation B (e.g., A reads a value written by B):
  - Then all processes must see B before A

  However:
  - Operations that are not causally related can be seen in different orders by different processes

  Causal consistency:
  - Provides a useful middle ground—stronger than eventual consistency but weaker than sequential consistency
  - It allows for lower latency while preserving important application invariants

- **Eventual consistency**: if no new writes, replicas converge - eventual consistency is the weakest common model.

  It guarantees:
  - That if no new writes are made, eventually all accesses will return the last updated value
  - During the convergence period, replicas may serve stale or conflicting data

  Eventual consistency:
  - Enables high availability and low latency but requires applications to handle temporary inconsistencies
  - Many NoSQL databases (Cassandra, DynamoDB in eventual mode) use eventual consistency

  The challenge:
  - Is designing applications that work correctly despite temporary inconsistencies

Tradeoff lens: the choice of consistency model represents a fundamental tradeoff between application complexity and system performance.

- Strong consistency simplifies application invariants: when the database provides strong consistency, applications can reason about data as if there were a single copy.

  You don't need:
  - To worry about reading stale data, handling conflicts, or implementing reconciliation logic
  - Business invariants like "account balance never goes negative" can be enforced by the database through constraints and transactions

  This:
  - Simplicity reduces application complexity and bug surface area

  However:
  - This simplicity comes at the cost of performance—strong consistency requires coordination, which increases latency and reduces throughput
  - During partitions, strong consistency systems may become unavailable, impacting user experience

- Weaker consistency can increase availability/throughput but forces app-level compensation: eventual consistency and other weak models enable higher availability and lower latency because replicas can operate independently without coordination.

  This:
  - Is valuable for globally distributed systems where network latency would make strong consistency prohibitively slow

  However:
  - The complexity shifts from the database to the application
  - Applications must handle stale reads, implement conflict resolution (last-write-wins, application-specific merge logic), and design invariants that tolerate temporary inconsistency

  For example:
  - Instead of enforcing "unique usernames" at the database level, you might need to implement a reservation system that handles conflicts

  This application-level complexity:
  - Is significant and often underestimated

  The rule of thumb:
  - Use the strongest consistency that meets your performance requirements—only relax consistency when you have a clear need and are prepared to handle the complexity

---

## Data Modeling Tradeoffs

### Relationships: the core difference
The fundamental difference between SQL and NoSQL data modeling is how they handle relationships between entities.

Relational data excels at: the relational model was designed around relationships—tables represent entity types, and foreign keys represent relationships between entities.

This design:
- Provides powerful capabilities for modeling complex relationships

- Modeling many-to-many relationships: relational databases handle many-to-many relationships naturally through junction tables.

  For example:
  - Students and courses have a many-to-many relationship—you create a students table, a courses table, and an enrollments junction table that references both
  - This design is normalized and flexible—you can add attributes to the relationship (like enrollment date, grade) without changing the student or course tables

  Querying these relationships:
  - Is declarative—JOIN statements express the relationship, and the optimizer determines the most efficient execution strategy

- Enforcing referential integrity: relational databases can enforce foreign key constraints, ensuring that relationships remain valid.

  If you try to delete a student who has enrollments:
  - The database can reject the deletion or cascade the delete to related records

  This enforcement:
  - Prevents orphaned records and maintains data integrity
  - The database handles this enforcement automatically, reducing application complexity and preventing bugs

- Expressing queries across relations: SQL's JOIN syntax allows you to query across arbitrarily complex relationship graphs in a single query.

  You can:
  - Join multiple tables, filter on relationships, and aggregate across related data—all in a declarative query
  - The query optimizer determines the most efficient join order and algorithm

  This capability:
  - Is powerful for ad-hoc analysis and complex business logic that spans multiple entities

NoSQL often prefers: NoSQL databases typically take a different approach to relationships, optimized for specific access patterns rather than general-purpose modeling.

- Aggregate-oriented models: NoSQL databases (especially document and wide-column stores) encourage modeling data as aggregates—self-contained units that include all related data.

  For example:
  - A customer document might include their contact information, address, orders, and preferences—all in one document

  This aggregate model:
  - Aligns with how applications use data (you often fetch an entity and its related data together)
  - The aggregate boundary becomes the unit of consistency and distribution, simplifying transactions and scaling

- Embedding related data (denormalization): instead of normalizing data into separate tables/collections, NoSQL encourages embedding related data within the same document or partition.

  For example:
  - A blog post document might embed comments directly rather than storing them in a separate collection

  This embedding:
  - Eliminates the need for joins—fetching the post fetches its comments in one operation
  - The tradeoff is that data is duplicated (comments might also be embedded in a user document) and updates must update all copies

- Avoiding joins: joins are expensive in distributed systems because they require coordinating data from multiple nodes, potentially across the network.

  NoSQL databases:
  - Avoid joins by designing the data model around access patterns—if you need to fetch related data together, store it together
  - This query-driven data modeling requires upfront planning about access patterns but enables predictable performance at scale

  For relationships that can't be embedded (like many-to-many):
  - Applications often perform application-side joins or maintain duplicate data in multiple aggregates

### Normalization vs denormalization
The choice between normalization and denormalization is a fundamental data modeling decision with significant implications for performance, complexity, and correctness.

- **Normalization**:
  - Pros: fewer inconsistencies, easier updates, less duplication - normalization organizes data to minimize redundancy and dependency.

  Each piece of data:
  - Is stored in exactly one place, which eliminates inconsistencies (you can't have two different values for the same attribute)
  - Updates are simpler because you only need to modify data in one place

  For example:
  - If a user changes their email address, you update it in the users table and it's automatically correct everywhere

  This:
  - Simplification reduces bugs and makes the data model easier to understand and maintain
  - Normalization also reduces storage requirements by eliminating duplicate data

  - Cons: joins, cross-partition pain, more round trips - the cost of normalization is that queries often require joining multiple tables to reconstruct complete entities.

  Each join:
  - Adds computational overhead and, in distributed systems, may require network round trips across partitions

  For example:
  - Fetching a user with their orders requires joining users and orders tables, which may be on different nodes in a sharded system

  This:
  - Can significantly impact performance at scale
  - The read amplification from joins is particularly problematic for read-heavy workloads where the same joined data is fetched repeatedly

- **Denormalization**:
  - Pros: faster reads, query-driven layouts - denormalization stores related data together, eliminating the need for joins at query time.

  If you always fetch a user with their profile:
  - Store them in the same document or table

  Reads:
  - Become single-record lookups rather than multi-table joins, which is dramatically faster, especially in distributed systems

  Denormalization:
  - Enables query-driven data modeling—you design your data structure around how you access data rather than around abstract purity principles
  - This can lead to significant performance improvements for read-heavy workloads

  - Cons: write complexity, data duplication, consistency bugs - the cost of denormalization is that data is duplicated.

  When you update data:
  - You must update all copies

  For example:
  - If a user's email is stored in multiple documents, an email update must update all of them atomically, which may require distributed transactions or compensation logic
  - This write complexity increases application code and the risk of inconsistencies (what if some copies update but others fail?)

  Data duplication:
  - Also increases storage requirements and can lead to consistency bugs if not managed carefully

  Denormalization:
  - Shifts complexity from reads to writes, which is appropriate for read-heavy workloads but problematic for write-heavy ones

### Schema evolution
How databases handle schema changes is a key operational and developmental difference between SQL and NoSQL.

- SQL: schema evolution is explicit; migrations can be risky but controlled - relational databases require explicit schema changes through DDL statements (ALTER TABLE, ADD COLUMN, etc.).

  Schema migrations:
  - Are planned, versioned operations that must be carefully coordinated with application deployments
  - This explicitness provides control—you can review and test migrations before applying them to production

  However:
  - Migrations can be risky for large tables—adding a column with a default value may require rewriting the entire table, causing downtime or performance degradation
  - Some migrations (like changing a column type) may be incompatible with existing data and require data transformation
  - The explicit nature of SQL schema evolution forces discipline but adds operational overhead

  Tools like Flyway and Liquibase:
  - Help manage migrations as code

- NoSQL: schema evolution is implicit; easy to start, but can accumulate "schema debt" (multiple shapes of documents) - document and schemaless databases allow implicit schema evolution.

  You can:
  - Add new fields to documents without any migration—old documents simply lack the new field, and new documents include it

  This:
  - Flexibility accelerates development, especially in early-stage products where requirements change frequently

  However:
  - This flexibility can lead to schema debt—over time, documents in the same collection may have wildly different structures, making queries and application logic complex
  - Handling missing fields, type mismatches, and structural variations becomes increasingly difficult

  Some NoSQL databases:
  - Added schema validation features (MongoDB's schema validation, Couchbase's N1QL) to provide guardrails, but the default is flexibility

  The challenge:
  - Is balancing the initial velocity of implicit evolution with the long-term maintainability of a coherent schema

### Constraints and invariants
Business rules and data integrity constraints are fundamental to many applications. The choice of database affects how easily these can be enforced.

If you need strong invariants (e.g., balances never go negative): enforcing such invariants requires coordination and transactional guarantees.

- SQL is usually the easiest place to enforce them (constraints + transactions) - relational databases provide built-in constraint enforcement (CHECK constraints, UNIQUE constraints, NOT NULL, FOREIGN KEYS) that the database enforces automatically.

  For example:
  - A CHECK constraint can ensure that account_balance >= 0, and the database will reject any update that would violate this
  - Combined with ACID transactions, you can enforce complex invariants across multiple tables—for example, transferring funds between accounts requires debiting one account and crediting another atomically

  The database:
  - Handles this enforcement, ensuring that the invariant is never violated regardless of concurrent access or failures
  - This enforcement is declarative and centralized, reducing application complexity and bug surface area

- With NoSQL, you often implement them using conditional writes, transactions (if available), or application-level workflows - many NoSQL databases don't provide the same level of constraint enforcement.

  You must:
  - Implement invariants in application code, which is more complex and error-prone

  Some NoSQL databases:
  - Offer conditional writes (compare-and-set operations) that help—for example, you can update an account balance only if the current value is sufficient
  - However, this only works for single-key operations; multi-key invariants require transactions (which some NoSQL databases support but with limitations) or application-level compensation logic

  For example:
  - To transfer funds between accounts in DynamoDB without transactions, you'd need to debit account A, then credit account B, and handle the case where the credit fails (requiring compensating transactions to rollback the debit)

  This:
  - Complexity increases the risk of bugs and makes the application harder to reason about
  - The general guidance is: if strong invariants are critical, SQL is often the better choice

---

## Operational Tradeoffs

### Backups and restores
Backup and restore strategies differ significantly between SQL and NoSQL databases due to their different architectures and consistency models.

- SQL: consistent backups often rely on snapshots + WAL archiving - relational databases provide well-established backup mechanisms that leverage the WAL.

  A common approach:
  - Is to take a filesystem snapshot of the database files while the database is running, then archive WAL files generated after the snapshot
  - To restore, you restore the snapshot and replay the WAL files to bring the database to a consistent point-in-time

  This approach:
  - Provides consistent, crash-consistent backups with minimal downtime

  Many relational databases:
  - Also support logical backups (pg_dump, mysqldump) that export data as SQL statements, which are portable but slower
  - The backup tools are mature, well-documented, and support point-in-time recovery—a critical feature for many applications

- NoSQL: backups vary widely; some require cluster-level snapshots and careful restore procedures - NoSQL backup strategies vary significantly by database type and deployment model.

  Document databases like MongoDB:
  - Support point-in-time recovery using oplog replay, similar to SQL WAL-based backups

  Wide-column stores like Cassandra:
  - Often require snapshotting each node independently and restoring the entire cluster as a unit, which is more complex

  Managed NoSQL services (DynamoDB, Cosmos DB):
  - Provide automated backups with point-in-time recovery, but self-managed deployments require more manual configuration

  Some NoSQL databases:
  - Don't provide built-in backup tools at all, requiring application-level backup or third-party solutions

  The restore procedure:
  - Can be complex—you may need to restore to a fresh cluster, ensure consistent replication, and handle conflicts
  - This operational complexity is an important consideration when choosing a NoSQL database

Key risk: restores must preserve consistency (schema + data + indexes) and be tested.

### Observability and debugging
The ability to understand database behavior and diagnose performance issues differs significantly between SQL and NoSQL databases.

- SQL: slow query logs, `EXPLAIN`, locks, stats are mature - relational databases have decades of tooling development for observability.

  Slow query logs:
  - Identify problematic queries automatically

  The `EXPLAIN` command (and variants like `EXPLAIN ANALYZE`):
  - Shows the query execution plan, allowing you to understand how the database will execute your query and identify bottlenecks
  - System catalogs provide detailed statistics about table sizes, index usage, and query patterns
  - Lock monitoring tools help identify blocking and contention issues

  These tools:
  - Are well-documented and standardized across most relational databases
  - The centralized nature of SQL databases also simplifies debugging—you're typically dealing with a single node or a simple primary-replica setup, making it easier to trace issues

- NoSQL: depends heavily on system; often more distributed complexity (hot partitions, compaction issues) - NoSQL observability tooling varies widely by database type and is generally less mature than SQL tooling.

  Many NoSQL databases:
  - Provide metrics for request latency, throughput, and error rates, but deeper diagnostic tools are often lacking

  Distributed NoSQL systems:
  - Introduce new failure modes that are harder to debug: hot partitions (uneven data distribution causing load imbalance), compaction storms (background compaction consuming resources), replication lag between nodes, and consistency violations

  Diagnosing these issues:
  - Requires understanding the database's internal architecture and often involves correlating metrics across multiple nodes

  Some managed NoSQL services:
  - Provide integrated observability (AWS CloudWatch for DynamoDB, Azure Monitor for Cosmos DB), but self-managed deployments require more manual configuration and third-party tools

### Hot partitions
A frequent NoSQL scaling failure mode: in distributed NoSQL databases that partition data across nodes, choosing the wrong partition key can create hotspots that cripple performance.

- If partition key distribution is skewed, one node gets overloaded - when you choose a partition key, you're deciding how data is distributed across the cluster.

  If the partition key:
  - Has low cardinality (few unique values) or uneven distribution, some nodes will receive much more data and traffic than others

  For example:
  - If you partition by region and 80% of your users are in the US-West region, the node(s) handling US-West will be overloaded while other nodes sit idle
  - This hotspot reduces overall throughput because the overloaded node becomes the bottleneck

  Even with high cardinality keys:
  - Hotspots can develop if certain keys are accessed much more frequently than others (e.g., a "popular" user ID that's queried constantly)

  Designing partition keys:
  - Requires understanding your access patterns and data distribution, often requiring upfront analysis and sometimes adding random prefixes to distribute load

SQL can also suffer from hotspots (sequence/monotonic keys on clustered indexes), but NoSQL partitioning makes this a central design concern - relational databases can also experience hotspots, particularly with clustered tables using sequential primary keys.

  All inserts:
  - Go to the end of the table, creating a hotspot on the last page

  However:
  - SQL databases are typically single-node or have simple replication, so the impact is more limited

  In distributed NoSQL systems:
  - Partition key design is absolutely critical—get it wrong, and your system won't scale regardless of how many nodes you add
  - This makes capacity planning and load testing essential for NoSQL deployments
  - Monitoring for hotspots (uneven request distribution across nodes) is an important operational practice

### Multi-region
Deploying databases across multiple geographic regions introduces fundamental tradeoffs between latency, consistency, and cost.

- Strong consistency across regions often means high latency - to provide strong consistency across regions, writes must be coordinated across all regions before being acknowledged.

  If you have regions in US-East, EU-West, and Asia-Pacific:
  - A write might need to wait for acknowledgments from all three regions, which could take hundreds of milliseconds due to network latency
  - This high latency degrades user experience for write-heavy workloads

  The alternative:
  - Is to accept eventual consistency, where writes are acknowledged locally and asynchronously replicated to other regions
  - This provides low latency for writes but risks temporary inconsistency—users in different regions might see different versions of data for a period

  This tradeoff:
  - Is a practical application of the PACELC theorem: in the normal case (no partition), you choose between latency and consistency

- Many systems use different strategies to handle multi-region deployments, each with different tradeoffs:
  - Single-writer region + async replicas - designate one region as the primary for writes, with other regions serving as read-only replicas.

  Writes:
  - Go to the primary region and are asynchronously replicated to replicas
  - Reads can be served from any replica, potentially returning stale data

  This:
  - Provides low write latency for users in the primary region and low read latency globally
  - But writes from other regions incur high latency (must go to primary), and replicas may be stale
  - This is a common pattern for globally distributed applications where most users are in one region or where stale reads are acceptable

  - Per-region writes with conflict resolution - allow writes in each region independently, with conflict resolution when data converges.

  This:
  - Provides low latency for writes everywhere but requires conflict resolution logic (last-write-wins, application-specific merge, or CRDTs)
  - This pattern works well for data that can tolerate temporary inconsistency and where conflicts are rare or can be resolved automatically

  The complexity:
  - Is in designing the conflict resolution strategy and handling edge cases

  - Consensus-based global DBs (higher cost/latency) - use distributed consensus protocols (Raft, Paxos) to coordinate writes across regions, providing strong consistency with automatic failover.

  Systems like CockroachDB, YugabyteDB, and Google Spanner:
  - Use this approach

  This:
  - Provides the strongest consistency guarantees and automatic failover, but at the cost of higher latency (writes must wait for consensus across regions) and higher operational complexity
  - These systems are often more expensive due to the resource requirements of consensus and the need for more replicas

---

## Decision Framework: How to Choose

Use these questions in order.

### 1) Do you need strict multi-row invariants?
If yes, start with SQL. The ability to enforce invariants across multiple rows or tables through ACID transactions and declarative constraints is SQL's strongest advantage. These invariants are critical for data integrity in many domains.

Examples:

- Payments, ledgers, inventory correctness - financial systems require strict invariants like "account balance never goes negative" or "total debits equal total credits".

  These invariants:
  - Span multiple rows (transactions affecting multiple accounts) and must be enforced atomically
  - SQL databases provide this through ACID transactions and constraints

  Implementing these invariants in NoSQL:
  - Requires complex application-level logic, distributed transactions (which many NoSQL databases don't support), or compensation patterns, all of which increase complexity and bug risk

- Unique constraints that must never be violated - business rules like "email addresses must be unique" or "product SKUs must be unique" are naturally expressed as UNIQUE constraints in SQL.

  The database:
  - Enforces these automatically, preventing violations at the data level

  In NoSQL:
  - You must implement uniqueness checks in application code, which is prone to race conditions unless you use conditional writes or application-level locking
  - For example, checking if an email exists before inserting it is not sufficient—two concurrent requests might both see the email as available and both insert, violating uniqueness

  SQL's constraint enforcement:
  - Eliminates this class of bugs

### 2) Are queries ad-hoc and evolving?
If analysts and product requirements demand flexible queries, SQL is usually best.

  SQL's:
  - Declarative query language and query optimizer enable ad-hoc queries without requiring upfront data modeling decisions
  - You can write new queries as requirements change without restructuring your data

  This flexibility:
  - Is valuable for analytics, reporting, and applications where query patterns are unpredictable

  NoSQL databases:
  - Typically require query-driven data modeling—you must know your access patterns upfront and design your data structure accordingly
  - If requirements change, you may need to restructure data and migrate

  For applications:
  - Where query patterns are stable and well-understood, this upfront planning is acceptable
  - Where queries are exploratory or frequently changing, SQL's flexibility is a significant advantage

### 3) Is your workload extremely write-heavy and horizontally distributed?
If yes, consider NoSQL (LSM-based stores, Dynamo-style KV) *if your access patterns are simple and well-defined*.

  LSM-based NoSQL databases:
  - Optimize for write throughput by turning random writes into sequential writes
  - This makes them ideal for write-heavy workloads like logging, event streaming, and time-series data

  However:
  - This write optimization comes at the cost of read performance and complexity (compaction, amplification)
  - Additionally, NoSQL's horizontal scaling enables handling massive write throughput by adding more nodes

  But:
  - This scaling requires simple access patterns—if you need complex queries or joins, NoSQL may not be the right choice despite the write workload

  The key question:
  - Can you design your data model around simple key-based access patterns?
  - If yes, NoSQL may be appropriate
  - If no, SQL with appropriate scaling (read replicas, partitioning) might be better despite the write overhead

### 4) Do you need high availability during partitions?
If you must accept writes even when parts of the system can't communicate, AP-style systems may fit.

  According to the CAP theorem:
  - During a network partition, you must choose between consistency and availability

  If your application:
  - Must remain writable even during partitions (e.g., a globally distributed social network where users in each region should be able to post even if regions can't communicate), an AP system with eventual consistency is appropriate
  - However, this comes with the cost of handling temporary inconsistencies and conflict resolution

  If your application:
  - Can tolerate being unavailable for writes during partitions (e.g., a financial system where correctness is more critical than availability), a CP system with strong consistency is better

  The choice:
  - Depends on your business requirements—can you tolerate stale data or conflicts, or is correctness more important than availability?

### 5) Can you model everything as aggregates?
If yes (and joins are rare), document/KV stores can work very well.

  The aggregate model:
  - Storing related data together as a unit—eliminates the need for joins and enables efficient single-record reads
  - If your data naturally falls into aggregates (a user document with profile, preferences, and recent activity; a product document with details, variants, and reviews), document stores are an excellent fit

  The key test:
  - Do you ever need to query across aggregates?
  - If you frequently need to join different entity types or perform complex multi-table queries, SQL is likely better
  - If your access patterns are primarily fetching and updating individual aggregates, document/KV stores provide simplicity and performance

  Be careful:
  - About many-to-many relationships—these don't fit well into aggregates and require either embedding (which causes duplication) or application-side joins

### 6) What is your team's operational maturity?
Distributed systems failure modes are real: operating a distributed NoSQL database requires different skills than operating a single-node SQL database. NoSQL introduces new failure modes that your team must understand and handle.

- Compaction storms - LSM-based databases require background compaction to merge files and reclaim space.

  Poorly tuned compaction:
  - Can cause "storms" where compaction consumes excessive CPU and I/O, degrading performance

  Your team:
  - Needs to understand compaction strategies and tuning parameters to avoid this

- Quorum timeouts - distributed systems rely on quorum-based coordination.

  If nodes are slow or unresponsive:
  - Quorum operations can timeout, causing failures

  Your team:
  - Needs to understand quorum configurations, timeout settings, and how to diagnose and respond to quorum failures

- Split brain - during network partitions, different nodes may believe they're the leader, causing divergent state.

  Your team:
  - Needs to understand consensus protocols, leader election, and how to detect and recover from split-brain scenarios

- Schema drift - schemaless databases can accumulate schema debt over time as documents evolve in different directions.

  Your team:
  - Needs processes to manage schema evolution and prevent uncontrolled divergence

- Inconsistent duplicates - denormalized data requires updating multiple copies.

  Your team:
  - Must implement and test update logic to ensure consistency across copies, handling failures appropriately

Sometimes a single well-tuned Postgres cluster beats an overcomplicated distributed design.

  The operational complexity:
  - Of distributed systems is often underestimated

  If your team:
  - Lacks distributed systems expertise, a simpler SQL solution may be more reliable and cost-effective, even if it doesn't scale as far

---

## Practical Examples

### Example 1: Payments and balances
Prefer SQL (or distributed SQL) because:

- transactions and constraints are the product
- correctness > availability
- auditability and strong semantics matter

### Example 2: Product catalog + search
Typical architecture:

- SQL or document store as source of truth
- search engine for full-text + faceting
- async indexing pipeline (CDC/outbox)

### Example 3: High write IoT metrics
Time-series or wide-column/LSM store is often best:

- high ingest
- time-window queries
- retention policies

### Example 4: Session store / rate limiting
Key-value store:

- simple operations
- high throughput
- TTL support

### Example 5: Social graph traversals
Graph DB (or specialized graph service) if deep traversals are core.

---

## Cheat Sheet Summary

### SQL is a great default when you need
- strong invariants and transactions
- joins and relational querying
- mature tooling and predictable correctness

### NoSQL shines when you need
- horizontal scaling and predictable throughput
- flexible/aggregate-first data models
- low-latency simple queries at huge scale

### The most common real-world outcome
Use multiple specialized datastores:

- SQL for system of record
- NoSQL KV/document for specific high-scale paths
- search engine for text
- time-series for metrics

The key system design skill is not picking one “best” database. It’s:

- choosing the right tool for each workload
- making tradeoffs explicit
- designing data flows to keep correctness and operability manageable
