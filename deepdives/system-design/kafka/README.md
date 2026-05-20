# Apache Kafka: Comprehensive Guide

## Table of Contents

1. [Introduction](#introduction)
2. [Core Architecture](#core-architecture)
3. [Fundamental Concepts](#fundamental-concepts)
4. [Scalability](#scalability)
5. [Fault Tolerance](#fault-tolerance)
6. [Error Handling](#error-handling)
7. [Performance Optimization](#performance-optimization)
8. [Retention Policies](#retention-policies)
9. [Internal Workings](#internal-workings)
10. [Producer Internals](#producer-internals)
11. [Consumer Internals](#consumer-internals)
12. [Security](#security)
13. [Monitoring and Operations](#monitoring-and-operations)
14. [Advanced Features](#advanced-features)
15. [Best Practices](#best-practices)

---

## Introduction

Apache Kafka is a distributed event streaming platform capable of handling trillions of events per day. Initially conceived at LinkedIn in 2010 and later open-sourced to the Apache Software Foundation, Kafka is designed for high-throughput, low-latency, and fault-tolerant real-time data streaming.

### What is Kafka?

Kafka is fundamentally a distributed commit log that provides a publish-subscribe messaging system. Unlike traditional message queues that remove messages after consumption, Kafka retains messages for a configurable period, allowing multiple consumers to process the same data at different times. This design makes Kafka ideal for event-driven architectures and stream processing.

**Key Philosophy:**
- **Write Once, Read Many**: Messages are written once and can be consumed multiple times by different consumer groups
- **Durable Storage**: Messages are persisted to disk, not just kept in memory
- **Scalable by Design**: Horizontal scaling is built into the architecture
- **Real-time Processing**: Designed for low-latency, high-throughput scenarios

### Key Characteristics

#### Distributed Architecture
Kafka runs as a cluster of nodes called brokers. This distributed nature provides:
- **Horizontal Scalability**: Add more brokers to increase capacity
- **Fault Tolerance**: No single point of failure
- **Load Distribution**: Work is spread across multiple nodes
- **Geographic Distribution**: Can span multiple data centers

**Example:** A 10-broker cluster can handle 10x the throughput of a single broker, and if one broker fails, the others continue operating.

#### Partitioning for Parallelism
Data is divided into partitions for parallelism:
- **Parallel Processing**: Multiple consumers can read from different partitions simultaneously
- **Ordered Guarantees**: Messages within a partition maintain strict ordering
- **Load Balancing**: Partitions are distributed across brokers
- **Scalability**: Add more partitions to increase parallelism

**Real-world Example:** A topic with 100 partitions can be consumed by 100 consumers in parallel, each processing a different subset of data.

#### Replication for Fault Tolerance
Each partition has multiple replicas for fault tolerance:
- **Data Redundancy**: Multiple copies of each message
- **Automatic Failover**: Leader election on failure
- **No Data Loss**: Replicas ensure data survives broker failures
- **Read Scalability**: Followers can serve read requests

**Example:** With replication factor of 3, each partition exists on 3 different brokers. If 2 brokers fail, data is still available.

#### Commit Log Semantics
Messages are appended to an immutable, ordered log:
- **Append-Only**: New messages are always added to the end
- **Immutable**: Existing messages are never modified
- **Ordered**: Each message has a unique offset
- **Sequential Access**: Optimized for sequential reads/writes

**Why Commit Log?** This design provides excellent performance for streaming workloads and enables replay capabilities.

#### Decoupled Producers and Consumers
Producers and consumers are decoupled in time and rate:
- **Temporal Decoupling**: Producers can write when consumers are offline
- **Rate Decoupling**: Producers can write faster than consumers can read
- **Independence**: Producers don't know about consumers and vice versa
- **Flexibility**: Add/remove consumers without affecting producers

**Example:** A producer can write 1 million messages per second while a consumer processes at 10,000 per second. Kafka buffers the difference.

### Use Cases with Detailed Examples

#### Log Aggregation
**Scenario:** A microservices architecture with 50 services generating logs.

**How Kafka Helps:**
- Each service sends logs to a Kafka topic (e.g., `application-logs`)
- Multiple consumers can process logs: one for real-time monitoring, one for archival, one for analytics
- Logs are retained for 7 days, enabling replay for debugging
- High throughput handles millions of log messages per minute

**Benefits:**
- Centralized logging without impacting application performance
- Multiple consumers for different purposes
- Replay capability for debugging incidents
- Decouples log producers from log consumers

#### Stream Processing
**Scenario:** Real-time fraud detection for financial transactions.

**How Kafka Helps:**
- Transaction events published to `transactions` topic
- Stream processing application (Kafka Streams, Flink, Spark) consumes transactions
- Real-time analysis detects suspicious patterns
- Alerts published to `fraud-alerts` topic
- All events retained for audit and replay

**Benefits:**
- Sub-second fraud detection
- Exactly-once processing guarantees
- Ability to replay events for model training
- Scalable to millions of transactions per second

#### Event Sourcing
**Scenario:** E-commerce system tracking order state changes.

**How Kafka Helps:**
- Each state change is an event: `OrderCreated`, `PaymentReceived`, `OrderShipped`
- Events stored in order topics in sequence
- Current state can be rebuilt by replaying events
- Multiple projections: order status view, inventory updates, customer notifications

**Benefits:**
- Complete audit trail of all changes
- Ability to rebuild state at any point in time
- Event-driven architecture enables loose coupling
- Natural fit with CQRS (Command Query Responsibility Segregation)

#### Publish-Subscribe Messaging
**Scenario:** E-commerce platform with multiple services needing order updates.

**How Kafka Helps:**
- Order service publishes events to `order-events` topic
- Inventory service consumes to update stock
- Notification service consumes to send emails
- Analytics service consumes for reporting
- Shipping service consumes to trigger fulfillment

**Benefits:**
- Single producer, multiple independent consumers
- Add new consumers without modifying order service
- Each consumer processes at its own pace
- Consumers can be added/removed dynamically

#### Commit Log for Distributed Systems
**Scenario:** Database change data capture (CDC).

**How Kafka Helps:**
- Database connector captures row-level changes
- Changes published to `database-changes` topic
- Downstream systems consume changes: search index, cache, analytics
- Changes retained for replay and recovery

**Benefits:**
- Reliable change propagation
- Consumers can be added without impacting database
- Ability to replay changes for recovery
- Decouples database from downstream systems

#### Activity Tracking
**Scenario:** User activity tracking for recommendation engine.

**How Kafka Handles:**
- User actions (clicks, views, purchases) published to `user-activity` topic
- Real-time consumption for personalization
- Batch consumption for model training
- Long retention for historical analysis

**Benefits:**
- High throughput for millions of user actions
- Real-time personalization capabilities
- Historical data for model training
- Flexible consumption patterns

### Kafka vs Traditional Messaging Systems

| Feature | Kafka | Traditional MQ (RabbitMQ, ActiveMQ) |
|---------|-------|-------------------------------------|
| Message Retention | Configurable retention (days to years) | Messages removed after consumption |
| Consumer Model | Multiple consumer groups | Typically single consumer per queue |
| Ordering | Per-partition ordering | Queue-level ordering |
| Throughput | Millions of messages per second | Thousands to hundreds of thousands |
| Scalability | Horizontal scaling via partitions | Vertical scaling or clustering |
| Storage | Disk-based persistence | Often memory-based |
| Replay Capability | Yes (messages retained) | No (messages deleted) |

### When to Use Kafka

**Ideal for:**
- Event-driven architectures
- Real-time stream processing
- Log aggregation and monitoring
- Change data capture (CDC)
- Activity tracking and analytics
- Microservices communication
- Event sourcing implementations

**Not ideal for:**
- Simple request-response patterns (use REST/RPC)
- Guaranteed once-only delivery without duplicates (use idempotence)
- Complex routing rules (use message brokers like RabbitMQ)
- Low-latency (<1ms) requirements (consider in-memory solutions)

---

## Core Architecture

### Kafka Cluster Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Broker 1   │  │   Broker 2   │  │   Broker 3   │      │
│  │              │  │              │  │              │      │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │      │
│  │  │ Topic A│  │  │  │ Topic A│  │  │  │ Topic A│  │      │
│  │  │ P0,P1  │  │  │  │ P2,P3  │  │  │  │ P4,P5  │  │      │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │      │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │      │
│  │  │ Topic B│  │  │  │ Topic B│  │  │  │ Topic B│  │      │
│  │  │ P0,P1  │  │  │  │ P2     │  │  │  │ P3     │  │      │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Components

#### 1. Broker

A Kafka broker is a single Kafka server that forms part of a Kafka cluster. Each broker hosts a subset of topic partitions and is responsible for storing and serving data to producers and consumers.

**Broker Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                        Kafka Broker                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Network     │  │   Request    │  │   Log        │      │
│  │  Threads     │  │   Handler    │  │  Manager     │      │
│  │              │  │              │  │              │      │
│  │  Handles TCP  │  │  Processes   │  │  Manages     │      │
│  │  connections │  │  requests    │  │  disk I/O    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Replica    │  │   Log        │  │   Cleaner    │      │
│  │  Manager     │  │  Cleaner     │  │   Thread     │      │
│  │              │  │              │  │              │      │
│  │  Handles     │  │  Compacts    │  │  Background  │      │
│  │  replication │  │  log data    │  │  cleanup     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Broker Responsibilities:**

1. **Store and Serve Topic Partitions**
   - Each broker stores partition data on local disk
   - Serves read requests from consumers
   - Serves write requests from producers
   - Manages log segment files

2. **Handle Partition Leadership**
   - Some partitions on the broker are leaders
   - Leader handles all read/write requests for its partitions
   - Followers replicate from leaders
   - Broker participates in leader election

3. **Participate in Controller Election**
   - All brokers can become controller
   - Controller manages cluster-wide operations
   - Brokers compete for controller role via Zookeeper

4. **Serve Producer and Consumer Requests**
   - Accepts produce requests and writes to log
   - Accepts fetch requests and reads from log
   - Handles metadata requests
   - Manages connection pools

5. **Manage Log Segments and Compaction**
   - Creates new log segments when size/time limits reached
   - Deletes old segments based on retention policies
   - Compacts log data if cleanup.policy=compact
   - Maintains index files for efficient lookups

6. **Handle Replica Synchronization**
   - Followers fetch data from leaders
   - Tracks in-sync replicas (ISR)
   - Updates high watermark
   - Handles out-of-sync replicas

**Broker Configuration:**

```properties
# Unique identifier for each broker (required)
broker.id=0

# Directories for storing log segments (comma-separated)
log.dirs=/mnt/kafka-logs-1,/mnt/kafka-logs-2

# Network processing threads (handles incoming requests)
# Rule: num.network.threads = 3 for production clusters
num.network.threads=3

# Disk I/O threads (handles disk reads/writes)
# Rule: num.io.threads = number of disks
num.io.threads=8

# Background threads for log compaction
log.cleaner.threads=4

# Socket send buffer size (network buffer for sending data)
socket.send.buffer.bytes=102400

# Socket receive buffer size (network buffer for receiving data)
socket.receive.buffer.bytes=102400

# Maximum size of a request (larger messages will be rejected)
socket.request.max.bytes=104857600

# Number of threads for handling requests
num.replica.fetchers=1

# Maximum number of connections
max.connections=10000

# Maximum connections per IP
max.connections.per.ip=10000
```

**Broker Startup Process:**

1. **Configuration Loading**: Reads server.properties
2. **Broker ID Assignment**: Uses configured broker.id or auto-generates
3. **Log Directory Initialization**: Creates log directories if needed
4. **Zookeeper Connection**: Connects to Zookeeper ensemble
5. **Broker Registration**: Registers broker metadata in Zookeeper
6. **Topic/Partition Loading**: Loads existing topic partitions
7. **Controller Election**: Participates in controller election
8. **Request Handler Startup**: Starts network and I/O threads
9. **Ready for Requests**: Begins accepting client connections

**Broker Shutdown Process:**

1. **Stop Accepting New Requests**: Network threads stop accepting connections
2. **Complete In-Flight Requests**: Waits for ongoing requests to complete
3. **Sync Logs**: Flushes any buffered data to disk
4. **Unregister from Zookeeper**: Removes broker metadata
5. **Close Log Files**: Closes all file handles
6. **Shutdown Threads**: Stops all background threads
7. **Exit**: Broker process terminates

**Broker Health Checks:**

- **Zookeeper Session**: Broker maintains ephemeral Zookeeper session
- **Controller Heartbeat**: Controller monitors broker liveness
- **Network Connectivity**: Broker must be reachable by other brokers
- **Disk Health**: Broker monitors disk health and available space
- **Memory Usage**: Broker monitors memory utilization

#### 2. Zookeeper

Zookeeper provides distributed coordination and configuration management for Kafka clusters. It acts as a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

**What is Zookeeper?**

Zookeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. It was designed to be simple and robust, making it ideal for Kafka's coordination needs.

**Zookeeper Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Zookeeper Ensemble                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   ZK Node 1  │  │   ZK Node 2  │  │   ZK Node 3  │      │
│  │              │  │              │  │              │      │
│  │  Leader      │  │  Follower    │  │  Follower    │      │
│  │              │  │              │  │              │      │
│  │  Handles     │  │  Replicates  │  │  Replicates  │      │
│  │  writes      │  │  data        │  │  data        │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                │                │                │
│         └────────────────┴────────────────┘                │
│                      │                                       │
│              Quorum-based consensus                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Zookeeper Responsibilities:**

1. **Broker Registration and Discovery**
   - Brokers register themselves on startup
   - Each broker creates an ephemeral node under `/brokers/ids/[broker.id]`
   - Ephemeral nodes are automatically deleted when broker session expires
   - Clients can discover all available brokers by reading `/brokers/ids`
   
   **Why Ephemeral Nodes?** If a broker crashes, its Zookeeper session expires, and the ephemeral node is automatically deleted. This allows other brokers to detect the failure and take action.

2. **Controller Election**
   - All brokers compete to create the `/controller` node
   - The broker that successfully creates the node becomes the controller
   - The `/controller` node contains the controller's broker ID
   - If controller fails, another broker creates the node and takes over

3. **Topic and Partition Configuration Storage**
   - Topic configurations stored under `/brokers/topics/[topic-name]`
   - Partition assignments stored under `/brokers/topics/[topic-name]/partitions/[partition-id]/state`
   - Configuration changes are persisted to Zookeeper
   - All brokers read configuration from Zookeeper

4. **Access Control Lists (ACLs)**
   - ACLs stored in Zookeeper
   - Controls who can perform operations on topics, groups, etc.
   - Enforced by brokers when processing requests

5. **Quota Management**
   - Producer and consumer quotas stored in Zookeeper
   - Limits on request rates or byte rates
   - Prevents clients from overwhelming the cluster

6. **Consumer Group Coordination (Legacy)**
   - Consumer group offsets stored in Zookeeper (pre-0.9)
   - Modern Kafka uses `__consumer_offsets` topic instead
   - Still used for some consumer group coordination

**Zookeeper Data Structure:**

```
/brokers
  /ids
    /0 -> {"host": "broker1.example.com", "port": 9092, ...}
    /1 -> {"host": "broker2.example.com", "port": 9092, ...}
    /2 -> {"host": "broker3.example.com", "port": 9092, ...}
  /topics
    /topic1
      /partitions
        /0 -> {"state": "leader", "leader": 0, "replicas": [0,1,2], "isr": [0,1,2]}
        /1 -> {"state": "leader", "leader": 1, "replicas": [1,2,0], "isr": [1,2,0]}
        /2 -> {"state": "leader", "leader": 2, "replicas": [2,0,1], "isr": [2,0,1]}
/controller -> {"brokerid": 0, "timestamp": "..."}
/controller_epoch -> 1
/consumers -> consumer group metadata (legacy)
/config
  /topics
    /topic1 -> topic-specific configuration
  /clients
    /client-id -> client-specific configuration
/admin
  /delete_topics -> topics marked for deletion
```

**Zookeeper Session Management:**

- Each broker maintains a session with Zookeeper
- Session timeout is configurable (default: 6 seconds)
- Heartbeats are sent periodically to keep session alive
- If session expires, ephemeral nodes are deleted
- This mechanism detects broker failures

**Zookeeper Configuration for Kafka:**

```properties
# Zookeeper connection string
zookeeper.connect=zk1.example.com:2181,zk2.example.com:2181,zk3.example.com:2181

# Session timeout in milliseconds
zookeeper.session.timeout.ms=6000

# Connection timeout in milliseconds
zookeeper.connection.timeout.ms=6000

# Sync time with Zookeeper
zookeeper.sync.time.ms=2000
```

**Zookeeper Best Practices:**

1. **Separate Zookeeper Cluster**: Don't run Zookeeper on the same nodes as Kafka brokers
2. **Odd Number of Nodes**: Use 3 or 5 Zookeeper nodes for quorum
3. **Dedicated Disk**: Zookeeper write-ahead log should be on a dedicated disk
4. **Monitor Zookeeper**: Monitor Zookeeper health and latency
5. **Backup Zookeeper Data**: Regularly backup Zookeeper data directory

**KRaft Mode (Kafka Raft)**

**Note:** Kafka 2.8+ introduces KRaft mode (Kafka Raft) which eliminates the need for Zookeeper.

**What is KRaft?**
- Kafka's own consensus protocol based on Raft
- Replaces Zookeeper for metadata management
- Simplifies Kafka deployment and operations
- Improves scalability and reduces latency

**KRaft Benefits:**
- **Simplified Architecture**: No separate Zookeeper cluster needed
- **Faster Metadata Operations**: No round-trip to Zookeeper
- **Better Scalability**: Can support more partitions
- **Easier Operations**: Fewer components to manage
- **Faster Recovery**: No Zookeeper dependency

**KRaft vs Zookeeper:**

| Feature | Zookeeper Mode | KRaft Mode |
|---------|---------------|------------|
| External Dependency | Requires Zookeeper | Self-contained |
| Metadata Operations | Slower (Zookeeper round-trip) | Faster (internal) |
| Scalability | Limited by Zookeeper | Higher scalability |
| Complexity | More complex | Simpler |
| Maturity | Very mature | Newer (still evolving) |
| Partition Limit | ~10,000 partitions per cluster | ~1,000,000 partitions per cluster |

#### 3. Controller

The controller is a special broker responsible for cluster management tasks. It is one of the brokers in the cluster that gets elected to perform administrative operations. There is always exactly one active controller in a Kafka cluster.

**What is the Controller?**

The controller is the brain of the Kafka cluster. While all brokers handle data storage and serving, the controller handles cluster-wide administrative tasks. Think of it as the manager that coordinates activities across all brokers.

**Controller Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Cluster                             │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Broker 0   │  │   Broker 1   │  │   Broker 2   │      │
│  │              │  │              │  │              │      │
│  │  CONTROLLER  │  │  Follower    │  │  Follower    │      │
│  │              │  │              │  │              │      │
│  │  Manages     │  │  Stores     │  │  Stores     │      │
│  │  cluster     │  │  data       │  │  data       │      │
│  └──────┬───────┘  └──────────────┘  └──────────────┘      │
│         │                                                      │
│         │ Coordinates cluster operations                      │
│         │                                                      │
│         └────────────────────────────────────────────────────  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Controller Responsibilities:**

1. **Partition Leadership Assignment**
   - When a topic is created, controller assigns partition leaders
   - Ensures leaders are distributed evenly across brokers
   - Uses rack-awareness if configured
   - Updates Zookeeper with leader information

2. **Partition Reassignment**
   - Moves partition replicas between brokers
   - Used during cluster expansion or broker removal
   - Ensures data is evenly distributed
   - Monitors reassignment progress

3. **Topic Creation and Deletion**
   - Creates topic metadata in Zookeeper
   - Assigns partitions to brokers
   - Handles topic deletion (marks for deletion, then removes)
   - Updates all brokers about topic changes

4. **Broker Failure Handling**
   - Monitors broker health via Zookeeper sessions
   - Detects broker failures when sessions expire
   - Elects new leaders from ISR for affected partitions
   - Triggers partition reassignment if needed

5. **Preferred Replica Election**
   - Sometimes replicas are not evenly distributed
   - Controller can trigger preferred replica election
   - Moves leadership back to preferred replica
   - Helps balance load across brokers

6. **Managing Inter-Broker Partitions**
   - Manages internal topics like `__consumer_offsets`
   - Ensures internal topics are properly replicated
   - Handles partition assignment for internal topics

**Controller Election Process:**

```
Step 1: All brokers start
        │
        ▼
Step 2: Each broker tries to create /controller node in Zookeeper
        │
        ▼
Step 3: First broker to create node becomes controller
        │
        ▼
Step 4: Controller writes its broker ID to /controller node
        │
        ▼
Step 5: Other brokers watch /controller node
        │
        ▼
Step 6: If controller fails, another broker creates /controller node
```

**Detailed Election Flow:**

1. **Broker Startup**: When a broker starts, it attempts to create the `/controller` ephemeral node in Zookeeper
2. **Election Success**: The broker that successfully creates the node becomes the controller
3. **Election Failure**: Other brokers receive "node already exists" error and become followers
4. **Controller Registration**: The controller writes its broker ID and epoch to Zookeeper
5. **Watch Setup**: All brokers set a watch on the `/controller` node
6. **Failover Detection**: If the controller's Zookeeper session expires, the node is deleted
7. **Re-election**: All brokers compete to create the node again
8. **New Controller**: The first broker to create the node becomes the new controller

**Controller Epoch:**

The controller epoch is a monotonically increasing counter that prevents stale controllers from operating.

```
Controller Epoch Flow:

Initial State: epoch = 0
                │
                ▼
Controller Elected: epoch = 1
                │
                ▼
Controller Fails
                │
                ▼
New Controller Elected: epoch = 2
                │
                ▼
Old Controller Recovers (rejected because epoch mismatch)
```

**Why Controller Epoch?**
- Prevents split-brain scenarios
- Old controllers cannot operate after failover
- All controller requests include epoch
- Brokers reject requests with stale epochs

**Controller State Machine:**

```
┌─────────────┐
│   INACTIVE  │  ← Initial state for all brokers
└──────┬──────┘
       │
       │ Becomes controller
       ▼
┌─────────────┐
│   ACTIVE    │  ← Currently controlling cluster
└──────┬──────┘
       │
       │ Loses controller role
       ▼
┌─────────────┐
│   FENCED    │  ← Was controller, now fenced
└─────────────┘
```

**Controller Failover Process:**

1. **Detection**: Controller's Zookeeper session expires (broker crash, network partition)
2. **Node Deletion**: `/controller` ephemeral node is automatically deleted
3. **Watch Trigger**: All brokers' watches on `/controller` node fire
4. **Election**: Brokers compete to create new `/controller` node
5. **New Controller**: One broker succeeds and becomes the new controller
6. **Epoch Increment**: New controller increments controller epoch
7. **State Rebuild**: New controller reads cluster state from Zookeeper
8. **Operation Resumes**: New controller begins handling cluster operations

**Controller Configuration:**

```properties
# Controller configuration (no direct config, but related settings)

# Zookeeper session timeout affects failover detection
zookeeper.session.timeout.ms=6000

# Time between controller operations
controller.socket.timeout.ms=30000

# Controller queue size
controller.quota.window.size.seconds=1
```

**Controller Best Practices:**

1. **Monitor Controller Health**: Watch for controller changes
2. **Plan for Failover**: Ensure any broker can become controller
3. **Avoid Controller Overload**: Don't create/delete topics too frequently
4. **Monitor Controller Metrics**: Track controller operation latency
5. **Test Failover**: Regularly test controller failover scenarios

---

## Fundamental Concepts

### Topic

A topic is a logical channel or feed name to which messages are published. Think of a topic as a category or feed name to which records are published. Topics are partitioned for scalability and parallelism.

**What is a Topic?**

A topic is like a table in a database or a folder in a file system. It's a logical grouping of messages that are related in some way. For example, all user click events might go to a topic called `user-clicks`, while all order events might go to a topic called `orders`.

**Topic Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                      Topic: "orders"                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Partition 0 │  │  Partition 1 │  │  Partition 2 │      │
│  │              │  │              │  │              │      │
│  │  Offset 0    │  │  Offset 0    │  │  Offset 0    │      │
│  │  Offset 1    │  │  Offset 1    │  │  Offset 1    │      │
│  │  Offset 2    │  │  Offset 2    │  │  Offset 2    │      │
│  │  ...         │  │  ...         │  │  ...         │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Topic Characteristics:**

1. **Named Category or Feed**
   - Topics are identified by name (case-sensitive)
   - Names should be descriptive of the data they contain
   - Examples: `user-events`, `order-placed`, `system-metrics`
   - Best practice: use lowercase with hyphens or underscores

2. **Multiple Partitions**
   - Each topic can have one or more partitions
   - Partitions enable parallel processing
   - More partitions = more parallelism
   - Trade-off: more partitions = more overhead

3. **Configurable Retention Policies**
   - How long messages are kept
   - Can be time-based (e.g., 7 days)
   - Can be size-based (e.g., 1GB per partition)
   - Can be infinite (never delete)

4. **Immutable, Ordered Sequence**
   - Messages are never modified once written
   - Each partition maintains strict ordering
   - New messages are always appended to the end
   - Cannot delete individual messages (only entire segments)

5. **Case-Sensitive Identification**
   - `orders` and `Orders` are different topics
   - Be careful with naming conventions
   - Consistent naming across environments

**Topic Configuration:**

```properties
# Number of partitions (can be increased but not decreased)
num.partitions=3

# Replication factor (how many copies of each partition)
default.replication.factor=3

# Retention settings
retention.ms=604800000  # 7 days in milliseconds
retention.bytes=1073741824  # 1GB per partition

# Cleanup policy (delete or compact)
cleanup.policy=delete  # or compact, or delete,compact

# Segment size (when to create new segment file)
segment.bytes=1073741824  # 1GB

# Segment roll time (max time before creating new segment)
segment.ms=604800000  # 7 days

# Maximum message size
max.message.bytes=1048576  # 1MB

# Minimum in-sync replicas
min.insync.replicas=2

# Compression type for topic
compression.type=producer  # or none, gzip, snappy, lz4, zstd
```

**Topic Naming Best Practices:**

- **Use lowercase**: `user-events` not `UserEvents`
- **Use hyphens or underscores**: `order-placed` not `orderplaced`
- **Be descriptive**: `payment-processed` not `events`
- **Use consistent naming**: All topics follow same pattern
- **Include environment**: `prod-user-events`, `dev-user-events`

**Topic Examples:**

1. **User Activity Topic**
   ```
   Topic: user-activity
   Partitions: 12
   Retention: 30 days
   Purpose: Track all user interactions
   ```

2. **Order Events Topic**
   ```
   Topic: order-events
   Partitions: 6
   Retention: 7 days
   Purpose: Track order lifecycle
   ```

3. **System Metrics Topic**
   ```
   Topic: system-metrics
   Partitions: 3
   Retention: 24 hours
   Purpose: Real-time system monitoring
   ```

### Partition

A partition is an ordered, immutable sequence of messages that is continually appended to. Each partition has a unique identifier within its topic. Partitions are the unit of parallelism in Kafka.

**What is a Partition?**

Think of a partition as a single log file. When you write to a topic, you're actually writing to one of its partitions. Each partition is an ordered, immutable sequence of messages that is continually appended to.

**Partition Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                  Partition 0 (Leader on Broker 0)           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Message 1 (Offset 0)                                │  │
│  │  Key: "order-123", Value: "{"type":"created"}"       │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │  Message 2 (Offset 1)                                │  │
│  │  Key: "order-456", Value: "{"type":"created"}"       │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │  Message 3 (Offset 2)                                │  │
│  │  Key: "order-789", Value: "{"type":"created"}"       │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │  ...                                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  Replicas:                                                   │
│  ┌────────────────┐  ┌────────────────┐                    │
│  │ Broker 1       │  │ Broker 2       │                    │
│  │ (Follower)     │  │ (Follower)     │                    │
│  └────────────────┘  └────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

**Partition Characteristics:**

1. **Ordered Sequence of Messages**
   - Messages within a partition maintain strict ordering
   - If message A comes before message B, it will always be read before B
   - Ordering is guaranteed only within a single partition
   - No ordering guarantees across different partitions

2. **Unique Offset per Message**
   - Each message has a unique offset (like a sequence number)
   - Offsets are monotonically increasing (0, 1, 2, 3, ...)
   - Offsets are unique within a partition
   - Offsets start at 0 for each partition

3. **Commit Log on Disk**
   - Partitions are stored as files on disk
   - Data is written sequentially for performance
   - Files are organized into segments
   - Sequential reads are very fast

4. **Replicated Across Brokers**
   - Each partition has multiple replicas
   - One replica is the leader, others are followers
   - Leader handles all read/write requests
   - Followers replicate from leader

5. **Leader-Follower Model**
   - One partition is the leader (handles all requests)
   - Other partitions are followers (replicate from leader)
   - If leader fails, a follower becomes leader
   - Followers can serve read requests (if configured)

**Partition Assignment:**

Partitions are distributed across brokers for load balancing and fault tolerance.

```
Example: Topic with 6 partitions across 3 brokers

Broker 0: Partition 0 (Leader), Partition 3 (Follower)
Broker 1: Partition 1 (Leader), Partition 4 (Follower)
Broker 2: Partition 2 (Leader), Partition 5 (Follower)

Replication Factor 2:
Partition 0: Leader on Broker 0, Follower on Broker 1
Partition 1: Leader on Broker 1, Follower on Broker 2
Partition 2: Leader on Broker 2, Follower on Broker 0
...and so on
```

**Assignment Strategies:**

1. **Round-Robin Assignment**
   - Partitions assigned to brokers in round-robin fashion
   - Ensures even distribution
   - Simple and predictable

2. **Rack-Aware Assignment**
   - Considers rack/host information
   - Replicas placed on different racks
   - Improves fault tolerance
   - Requires rack configuration

3. **Custom Assignment**
   - Use custom partition assigner
   - For specific requirements
   - Advanced use case

**Partition Offset:**

The offset is a unique identifier for each message within a partition.

```
Offset Example:

Partition 0:
Offset 0: Message A
Offset 1: Message B
Offset 2: Message C
Offset 3: Message D
...

Consumer Position:
Consumer last read offset 2
Next message to read: offset 3
```

**Offset Properties:**

- **Monotonically Increasing**: Offsets always increase (0, 1, 2, 3, ...)
- **64-bit Integer**: Can hold very large numbers (2^63 - 1)
- **Unique Within Partition**: Each offset identifies exactly one message
- **Consumer Tracking**: Consumers track their position using offsets
- **Committed Offsets**: Stored in `__consumer_offsets` topic
- **Can Replay**: Consumers can reset to any offset

**Offset Commit Process:**

```
1. Consumer reads messages
2. Consumer processes messages
3. Consumer commits offset
4. Offset stored in __consumer_offsets topic
5. If consumer fails, it resumes from last committed offset
```

**How Partitions Enable Parallelism:**

```
Single Partition (No Parallelism):
Producer → [Partition 0] → Consumer 1
Throughput: 1x

Multiple Partitions (Parallelism):
Producer → [Partition 0] → Consumer 1
Producer → [Partition 1] → Consumer 2
Producer → [Partition 2] → Consumer 3
Throughput: 3x (theoretical)
```

**Partition Count Considerations:**

**Too Few Partitions:**
- Limits parallelism
- Can't add more consumers than partitions
- Hot partitions can become bottlenecks

**Too Many Partitions:**
- Increased overhead (more files, more file handles)
- Longer rebalancing times
- More memory usage for metadata
- Slower recovery from failures

**Rule of Thumb:**
- Start with 3-6 partitions per topic
- Scale based on expected consumer count
- Consider future growth
- Monitor and adjust as needed

**Partition Replication:**

```
Partition Replication Example:

Topic: orders, Partitions: 3, Replication Factor: 3

Partition 0:
  Leader: Broker 0
  Replicas: Broker 0, Broker 1, Broker 2
  ISR: Broker 0, Broker 1, Broker 2

Partition 1:
  Leader: Broker 1
  Replicas: Broker 1, Broker 2, Broker 0
  ISR: Broker 1, Broker 2, Broker 0

Partition 2:
  Leader: Broker 2
  Replicas: Broker 2, Broker 0, Broker 1
  ISR: Broker 2, Broker 0, Broker 1
```

**Partition Configuration:**

```properties
# Partition count (set at topic creation, can only increase)
num.partitions=6

# Minimum in-sync replicas (must have this many replicas)
min.insync.replicas=2

# Replica lag time (how long a replica can lag)
replica.lag.time.max.ms=30000

# Segment size for partition
segment.bytes=1073741824

# Segment roll time
segment.ms=604800000
```

### Replication

Replication provides fault tolerance by storing multiple copies of each partition across different brokers. This ensures that if a broker fails, data is still available from other brokers.

**What is Replication?**

Replication is the process of copying data from one broker to another. In Kafka, each partition can have multiple replicas distributed across different brokers. If one broker fails, the data is still available on other brokers.

**Replication Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│              Partition Replication (RF=3)                    │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Broker 0   │  │   Broker 1   │  │   Broker 2   │      │
│  │              │  │              │  │              │      │
│  │  Partition 0  │  │  Partition 0  │  │  Partition 0  │      │
│  │  (LEADER)    │  │  (FOLLOWER)  │  │  (FOLLOWER)  │      │
│  │              │  │              │  │              │      │
│  │  Offset 0    │  │  Offset 0    │  │  Offset 0    │      │
│  │  Offset 1    │  │  Offset 1    │  │  Offset 1    │      │
│  │  Offset 2    │  │  Offset 2    │  │  Offset 2    │      │
│  │  ...         │  │  ...         │  │  ...         │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
│                      │                                       │
│              All replicas have same data                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Replica Types:**

1. **Leader Replica**
   - Handles all read and write requests for the partition
   - Only replica that accepts writes from producers
   - Maintains the authoritative copy of the data
   - Coordinates replication to followers
   - One leader per partition

2. **Follower Replica**
   - Passively replicates leader's data
   - Does not accept write requests (unless it becomes leader)
   - Can serve read requests (if configured)
   - Continuously fetches data from leader
   - Multiple followers per partition

3. **ISR (In-Sync Replica)**
   - Replicas that are fully caught up with leader
   - Have replicated all messages up to the leader's high watermark
   - Eligible to become leader if current leader fails
   - Leader maintains a list of ISR replicas
   - Dynamically shrinks/expands based on replica health

4. **OSR (Out-of-Sync Replica)**
   - Replicas that have fallen behind the leader
   - Not eligible to become leader
   - Will try to catch up and rejoin ISR
   - Can be removed from replica list if too far behind

**Replication Process:**

```
Step-by-Step Replication Flow:

1. Producer sends message to leader
   Producer → Leader Broker
   Message: {key: "order-123", value: {...}}

2. Leader appends message to its log
   Leader writes to local disk
   Offset assigned: 42

3. Leader waits for in-sync replicas to acknowledge
   Leader sends to Follower 1
   Leader sends to Follower 2
   Leader waits for acknowledgments

4. Followers fetch and append messages
   Follower 1 fetches from leader
   Follower 1 appends to its log
   Follower 1 acknowledges to leader
   Follower 2 fetches from leader
   Follower 2 appends to its log
   Follower 2 acknowledges to leader

5. Leader updates high watermark
   Once all ISR replicas acknowledge
   HW advances to offset 42

6. Leader responds to producer
   Leader acknowledges to producer
   Producer can send next message
```

**Replication Configuration:**

```properties
# Minimum in-sync replicas (must have this many replicas for writes)
# If ISR size drops below this, writes will fail
min.insync.replicas=2

# Replica lag time (how long a replica can be behind before being removed from ISR)
replica.lag.time.max.ms=30000  # 30 seconds

# Replica lag count (deprecated - use time-based instead)
replica.lag.max.messages=4000

# Default replication factor for new topics
default.replication.factor=3

# Replica fetch timeout
replica.fetch.timeout.ms=30000

# Replica fetch max bytes
replica.fetch.max.bytes=1048576
```

**Replication Factor Selection:**

| Replication Factor | Fault Tolerance | Use Case |
|-------------------|-----------------|----------|
| 1 | 0 broker failures | Development, non-critical data |
| 2 | 1 broker failure | Small production clusters |
| 3 | 2 broker failures | Standard production (recommended) |
| 5 | 4 broker failures | Critical systems, compliance |

**ISR (In-Sync Replicas) in Detail:**

```
ISR Dynamics:

Initial State:
Partition 0: [Leader: Broker 0, ISR: Broker 0, Broker 1, Broker 2]

Broker 2 falls behind:
- Broker 2 hasn't fetched for 30 seconds
- Controller removes Broker 2 from ISR
- Partition 0: [Leader: Broker 0, ISR: Broker 0, Broker 1]
- Writes still accepted (ISR size >= min.insync.replicas)

Broker 2 catches up:
- Broker 2 starts fetching again
- Broker 2 catches up to leader
- Controller adds Broker 2 back to ISR
- Partition 0: [Leader: Broker 0, ISR: Broker 0, Broker 1, Broker 2]
```

**Why ISR Matters:**

- **Data Durability**: Only ISR replicas are guaranteed to have the latest data
- **Leader Election**: Only ISR replicas can become leader (by default)
- **Write Guarantees**: Producer waits for ISR acknowledgments
- **Consistency**: Consumers can only read up to the high watermark

**High Watermark (HW):**

The high watermark is the last offset that has been successfully replicated to all ISR replicas.

```
High Watermark Example:

Leader Log:   [0, 1, 2, 3, 4, 5, 6]
Follower 1:    [0, 1, 2, 3, 4, 5]
Follower 2:    [0, 1, 2, 3, 4, 5]

High Watermark: 5 (all replicas have up to offset 5)
Consumers can read up to offset 5
Offset 6 is not yet visible to consumers
```

**Replication vs Availability Trade-off:**

```
Higher Replication Factor:
✓ Better fault tolerance
✓ Better data durability
✗ More disk space required
✗ More network bandwidth
✗ Higher latency for writes

Lower Replication Factor:
✓ Less disk space
✓ Lower write latency
✗ Less fault tolerance
✗ Risk of data loss
```

**Replication Best Practices:**

1. **Use RF=3 for Production**: Provides good balance of fault tolerance and performance
2. **Monitor ISR Size**: Alert if ISR size drops below min.insync.replicas
3. **Spread Replicas Across Racks**: Use rack-aware assignment for better fault tolerance
4. **Set min.insync.replicas=2**: Ensures at least 2 replicas have data before acknowledging
5. **Monitor Replication Lag**: Ensure followers are keeping up with leader

### Producer

A producer publishes messages to Kafka topics. Producers are client applications that send data to Kafka clusters.

**What is a Producer?**

A producer is a client application that writes (publishes) messages to Kafka topics. It's responsible for:
- Deciding which partition to write to
- Serializing the message data
- Handling compression
- Managing retries and error handling
- Batching messages for efficiency

**Producer Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                       Producer Client                         │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Serializer │  │ Partitioner  │  │   Record    │      │
│  │              │  │              │  │  Accumulator │      │
│  │  Converts    │  │  Determines  │  │  Batches     │      │
│  │  objects to  │  │  target      │  │  messages    │      │
│  │  bytes       │  │  partition   │  │  per partition│     │
│  └──────────────┘  └──────────────┘  └──────┬───────┘      │
│                                            │                │
│  ┌──────────────┐  ┌──────────────┐       │                │
│  │   Metadata   │  │   Network    │◄──────┘                │
│  │   Service    │  │   Client     │                        │
│  │              │  │              │                        │
│  │  Fetches     │  │  Sends       │                        │
│  │  cluster     │  │  requests    │                        │
│  │  metadata    │  │  to brokers  │                        │
│  └──────────────┘  └──────────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Producer Responsibilities:**

1. **Serialize Messages**
   - Convert objects to byte arrays
   - Uses configured serializer (String, JSON, Avro, etc.)
   - Separate serializers for keys and values
   - Must match consumer deserializer

2. **Partition Messages**
   - Determine which partition to write to
   - Uses message key (if present) with hash function
   - Round-robin if no key
   - Custom partitioner for advanced scenarios

3. **Handle Retries and Error Handling**
   - Automatic retry on transient failures
   - Configurable retry count and backoff
   - Handles leader not available errors
   - Network error recovery

4. **Manage Compression**
   - Compress batches before sending
   - Reduces network bandwidth
   - Reduces disk I/O on brokers
   - Configurable compression type

5. **Batch Messages for Efficiency**
   - Groups messages into batches
   - Reduces network overhead
   - Improves throughput
   - Configurable batch size and linger time

6. **Handle Acknowledgments**
   - Wait for broker acknowledgment
   - Configurable acknowledgment level
   - Ensures data durability
   - Balances latency and reliability

**Producer Configuration:**

```properties
# Bootstrap servers (comma-separated list of brokers)
bootstrap.servers=localhost:9092,localhost:9093,localhost:9094

# Key serializer (converts key object to bytes)
key.serializer=org.apache.kafka.common.serialization.StringSerializer

# Value serializer (converts value object to bytes)
value.serializer=org.apache.kafka.common.serialization.StringSerializer

# Acknowledgments (how many replicas must acknowledge)
acks=0  # No acknowledgment (fire and forget)
acks=1  # Leader acknowledgment only (default)
acks=all  # All in-sync replicas acknowledgment (most durable)

# Retries (how many times to retry on failure)
retries=2147483647  # Retry indefinitely (effectively)
retry.backoff.ms=100  # Backoff between retries

# Batch size (maximum size of a batch in bytes)
batch.size=16384  # 16KB (default)
# Larger batches = better throughput, higher latency

# Linger time (how long to wait for batch to fill)
linger.ms=0  # Send immediately (default)
linger.ms=10  # Wait up to 10ms for batch to fill
# Longer linger = better throughput, higher latency

# Buffer memory (total memory for buffering messages)
buffer.memory=33554432  # 32MB (default)
# Must accommodate max.in.flight.requests.per.connection

# Compression (compress batches before sending)
compression.type=none  # No compression (default)
compression.type=gzip  # High compression, CPU intensive
compression.type=snappy  # Balanced compression and speed
compression.type=lz4  # Fast compression, good ratio
compression.type=zstd  # Best compression, moderate CPU

# Max request size (maximum size of a request)
max.request.size=1048576  # 1MB (default)
# Must be <= broker's max.message.bytes

# Max in-flight requests (number of unacknowledged requests)
max.in.flight.requests.per.connection=5  # Default
# Higher = better throughput, potential ordering issues with retries

# Enable idempotence (prevents duplicate messages)
enable.idempotence=false  # Default
enable.idempotence=true  # Recommended for production

# Transactional ID (for exactly-once semantics)
transactional.id=my-unique-transactional-id
# Required for transactional producer

# Request timeout
request.timeout.ms=30000  # 30 seconds (default)

# Delivery timeout (total time for a message to be acknowledged)
delivery.timeout.ms=120000  # 2 minutes (default)
```

**Producer Send Process:**

```
Step-by-Step Message Send Flow:

1. Application calls producer.send()
   ProducerRecord<String, String> record = 
       new ProducerRecord<>("my-topic", "key", "value");
   producer.send(record);

2. Serializer converts key/value to bytes
   Key: "key" → bytes: [107, 101, 121]
   Value: "value" → bytes: [118, 97, 108, 117, 101]

3. Partitioner determines target partition
   If key exists: hash("key") % num_partitions
   If no key: round-robin (sticky)

4. Message added to record accumulator
   Batched with other messages for same partition
   Waits for batch to fill or linger time to expire

5. Sender thread picks up batch
   Creates produce request
   Sends to appropriate broker (partition leader)

6. Broker processes request
   Writes message to log
   Replicates to followers (if acks=all)
   Sends acknowledgment

7. Producer receives acknowledgment
   Calls callback (if async)
   Returns future (if sync)
   Handles errors/retries

8. Application notified of success/failure
   Callback invoked with RecordMetadata or exception
```

**Acknowledgment Levels Explained:**

| acks Value | Description | Latency | Durability | Use Case |
|------------|-------------|---------|------------|----------|
| 0 | No acknowledgment | Lowest | Lowest | Non-critical data, metrics |
| 1 | Leader acknowledgment | Medium | Medium | General purpose (default) |
| all | All ISR acknowledgment | Highest | Highest | Critical data, financial transactions |

**Example: acks=0 (Fire and Forget)**
```properties
acks=0
```
- Producer doesn't wait for acknowledgment
- Lowest latency
- Data can be lost if broker fails before writing to disk
- Use for: metrics, logs, non-critical data

**Example: acks=1 (Leader Acknowledgment)**
```properties
acks=1
```
- Producer waits for leader acknowledgment
- Leader writes to local disk
- Data can be lost if leader fails before replication
- Use for: general purpose applications

**Example: acks=all (All ISR Acknowledgment)**
```properties
acks=all
min.insync.replicas=2
```
- Producer waits for all in-sync replicas to acknowledge
- Highest durability
- Higher latency
- Use for: critical data, financial transactions, compliance

**Producer Example Code:**

```java
// Create producer configuration
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);
props.put("enable.idempotence", true);

// Create producer
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// Create message
ProducerRecord<String, String> record = 
    new ProducerRecord<>("my-topic", "my-key", "my-value");

// Send message (async with callback)
producer.send(record, new Callback() {
    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        if (exception == null) {
            System.out.println("Message sent to partition " + metadata.partition() +
                              " at offset " + metadata.offset());
        } else {
            exception.printStackTrace();
        }
    }
});

// Send message (sync)
try {
    RecordMetadata metadata = producer.send(record).get();
    System.out.println("Message sent to partition " + metadata.partition());
} catch (Exception e) {
    e.printStackTrace();
}

// Close producer (flushes any buffered messages)
producer.close();
```

**Producer Best Practices:**

1. **Use acks=all for Critical Data**: Ensures data is replicated before acknowledgment
2. **Enable Idempotence**: Prevents duplicate messages on retries
3. **Tune Batch Size and Linger**: Balance throughput and latency for your workload
4. **Use Compression**: Reduces network bandwidth and disk I/O
5. **Handle Callbacks**: Use callbacks for async error handling
6. **Close Producer Properly**: Ensures all messages are sent before shutdown
7. **Monitor Producer Metrics**: Track send rate, error rate, retry rate

### Consumer

A consumer subscribes to topics and processes messages. Consumers are client applications that read (consume) data from Kafka clusters.

**What is a Consumer?**

A consumer is a client application that reads messages from Kafka topics. It's responsible for:
- Subscribing to one or more topics
- Fetching messages from partitions
- Processing messages
- Tracking its position (offset) in the partition
- Committing offsets to track progress
- Handling rebalancing when consumers join/leave

**Consumer Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                       Consumer Client                         │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Deserializer │  │   Fetcher    │  │  Consumer    │      │
│  │              │  │              │  │  Coordinator │      │
│  │  Converts    │  │  Fetches     │  │  Manages     │      │
│  │  bytes to    │  │  messages    │  │  group       │      │
│  │  objects     │  │  from brokers│  │  membership  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                            │                │
│  ┌──────────────┐  ┌──────────────┐       │                │
│  │  Heartbeat   │  │   Offset     │       │                │
│  │    Thread    │  │   Manager    │◄──────┘                │
│  │              │  │              │                        │
│  │  Sends       │  │  Commits     │                        │
│  │  heartbeats  │  │  offsets     │                        │
│  └──────────────┘  └──────────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Consumer Responsibilities:**

1. **Subscribe to Topics**
   - Subscribe to one or more topics
   - Can subscribe to topic patterns (regex)
   - Can assign specific partitions
   - Dynamic subscription changes supported

2. **Fetch Messages from Partitions**
   - Fetch messages from assigned partitions
   - Configurable fetch size and wait time
   - Prefetching for better throughput
   - Handles broker failures

3. **Process Messages**
   - Application-specific message processing
   - Can be synchronous or asynchronous
   - Should handle processing errors
   - Implement dead letter queue for failed messages

4. **Commit Offsets**
   - Track position in each partition
   - Auto-commit or manual commit
   - Synchronous or asynchronous commit
   - Commit on successful processing

5. **Handle Rebalancing**
   - Respond to partition assignment changes
   - Commit offsets before losing partitions
   - Seek to appropriate offsets on new partitions
   - Implement rebalance listener

6. **Manage Consumer Group Coordination**
   - Join consumer group
   - Send heartbeats to coordinator
   - Participate in rebalancing
   - Handle group coordination

**Consumer Configuration:**

```properties
# Bootstrap servers (comma-separated list of brokers)
bootstrap.servers=localhost:9092,localhost:9093,localhost:9094

# Group ID (identifies the consumer group)
group.id=my-consumer-group
# Consumers with same group.id belong to same group

# Key deserializer (converts bytes to key object)
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer

# Value deserializer (converts bytes to value object)
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

# Auto offset reset (where to start if no committed offset)
auto.offset.reset=latest  # Start from newest (default)
auto.offset.reset=earliest  # Start from oldest
auto.offset.reset=none  # Throw exception if no committed offset

# Enable auto commit (automatically commit offsets periodically)
enable.auto.commit=true  # Default
enable.auto.commit=false  # Manual commit (recommended for reliability)

# Auto commit interval (how often to auto-commit)
auto.commit.interval.ms=5000  # 5 seconds (default)

# Session timeout (how long consumer can be inactive)
session.timeout.ms=10000  # 10 seconds (default)
# If no heartbeat within this time, consumer is considered dead

# Heartbeat interval (how often to send heartbeat)
heartbeat.interval.ms=3000  # 3 seconds (default)
# Should be less than session.timeout.ms / 3

# Max poll records (maximum records returned in a single poll)
max.poll.records=500  # Default
# Controls how many records to process per poll

# Max poll interval (maximum time between polls)
max.poll.interval.ms=300000  # 5 minutes (default)
# If processing takes longer, consumer is considered dead

# Fetch minimum bytes (minimum bytes to wait for in fetch)
fetch.min.bytes=1  # Default (return immediately if data available)

# Fetch maximum bytes (maximum bytes to return in fetch)
fetch.max.bytes=52428800  # 50MB (default)

# Fetch maximum wait time (how long to wait for fetch.min.bytes)
fetch.max.wait.ms=500  # 500ms (default)

# Max partition fetch bytes (maximum bytes per partition)
max.partition.fetch.bytes=1048576  # 1MB (default)

# Isolation level (read committed vs read uncommitted)
isolation.level=read_uncommitted  # Default (read all messages)
isolation.level=read_committed  # Only read committed messages (for transactions)

# Client ID (identifies the consumer)
client.id=my-consumer-client-1

# Enable auto commit on close
enable.auto.commit.on.close=true  # Default
```

**Consumer Subscribe vs Assign:**

**Subscribe (Dynamic Partition Assignment):**
```java
consumer.subscribe(Arrays.asList("topic1", "topic2"));
```
- Consumer joins a consumer group
- Partitions are automatically assigned
- Rebalancing happens when consumers join/leave
- Recommended for most use cases

**Assign (Manual Partition Assignment):**
```java
TopicPartition partition0 = new TopicPartition("topic1", 0);
TopicPartition partition1 = new TopicPartition("topic1", 1);
consumer.assign(Arrays.asList(partition0, partition1));
```
- Consumer manually specifies partitions
- No consumer group coordination
- No automatic rebalancing
- Use for specific scenarios

**Consumer Poll Process:**

```
Step-by-Step Consumer Poll Flow:

1. Consumer subscribes to topics
   consumer.subscribe(Arrays.asList("my-topic"));

2. Consumer joins consumer group
   Sends JoinGroup request to coordinator
   Coordinator assigns partitions to consumer

3. Consumer polls for messages
   ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

4. Fetcher fetches messages from brokers
   Sends FetchRequest to partition leaders
   Waits for messages (up to fetch.max.wait.ms)
   Receives messages from assigned partitions

5. Deserializer converts bytes to objects
   Bytes → String/JSON/Avro objects
   Uses configured deserializer

6. Application processes messages
   for (ConsumerRecord<String, String> record : records) {
       // Process record
       processRecord(record);
   }

7. Offsets tracked in memory
   Consumer tracks current offset for each partition

8. Offsets committed (auto or manual)
   Committed to __consumer_offsets topic
   Used for recovery on restart
```

**Offset Management Strategies:**

**At-Most-Once Semantics (Can Lose Messages):**
```java
// Commit before processing
consumer.commitSync();
processRecords(records);
```
- Offsets committed before processing
- If processing fails, messages are lost
- Lowest latency
- Use for: non-critical data where loss is acceptable

**At-Least-Once Semantics (Can Duplicate Messages):**
```java
// Commit after processing
processRecords(records);
consumer.commitSync();
```
- Offsets committed after processing
- If processing fails, messages are reprocessed (duplicates)
- Higher latency
- Use for: critical data where duplicates are acceptable

**Exactly-Once Semantics (No Loss, No Duplicates):**
```java
// Use transactions (requires transactional producer/consumer)
props.put("isolation.level", "read_committed");
```
- Requires transactional producer and consumer
- Highest complexity
- Use for: financial transactions, exactly-once requirements

**Consumer Example Code:**

```java
// Create consumer configuration
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-consumer-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("enable.auto.commit", false);  // Manual commit for reliability

// Create consumer
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

// Subscribe to topics
consumer.subscribe(Arrays.asList("my-topic"));

try {
    while (true) {
        // Poll for messages (timeout of 100ms)
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        
        // Process records
        for (ConsumerRecord<String, String> record : records) {
            System.out.println("Received message: " + record.value());
            System.out.println("Partition: " + record.partition() + ", Offset: " + record.offset());
            
            // Process the message
            processMessage(record);
        }
        
        // Commit offsets after successful processing
        consumer.commitSync();
    }
} catch (Exception e) {
    e.printStackTrace();
} finally {
    consumer.close();
}
```

**Consumer with Rebalance Listener:**

```java
consumer.subscribe(Arrays.asList("my-topic"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Called before partitions are revoked
        // Commit offsets before losing partitions
        consumer.commitSync();
        System.out.println("Partitions revoked: " + partitions);
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Called after partitions are assigned
        // Seek to appropriate offsets
        for (TopicPartition partition : partitions) {
            consumer.seek(partition, getOffsetForPartition(partition));
        }
        System.out.println("Partitions assigned: " + partitions);
    }
});
```

**Consumer Best Practices:**

1. **Use Manual Commit for Reliability**: Commit offsets after successful processing
2. **Handle Processing Errors**: Implement dead letter queue for failed messages
3. **Implement Rebalance Listener**: Commit offsets before losing partitions
4. **Tune Poll Timeout**: Balance between latency and throughput
5. **Monitor Consumer Lag**: Ensure consumers keep up with producers
6. **Handle Consumer Failures**: Design for graceful shutdown and restart
7. **Use Appropriate Group ID**: Different groups for different consumption patterns

### Consumer Group

A consumer group is a set of consumers that cooperate to consume messages from a topic. This is one of Kafka's most powerful features, enabling scalable and fault-tolerant message consumption.

**What is a Consumer Group?**

A consumer group is a logical grouping of consumers that work together to consume messages from topics. Each partition within a topic is consumed by exactly one consumer within the group. This enables:
- **Scalability**: Add more consumers to increase throughput
- **Fault Tolerance**: If a consumer fails, its partitions are reassigned
- **Load Balancing**: Work is distributed across consumers
- **Offset Tracking**: Group tracks consumption position for recovery

**Consumer Group Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│              Topic with 6 Partitions                         │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Partition 0 │  │  Partition 1 │  │  Partition 2 │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Partition 3 │  │  Partition 4 │  │  Partition 5 │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Assigned to
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Consumer Group: "order-processors"              │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Consumer 1  │  │  Consumer 2  │  │  Consumer 3  │      │
│  │              │  │              │  │              │      │
│  │  Partition 0 │  │  Partition 1 │  │  Partition 2 │      │
│  │  Partition 3 │  │  Partition 4 │  │  Partition 5 │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Consumer Group Characteristics:**

1. **Each Partition Consumed by Exactly One Consumer**
   - No two consumers in the same group consume the same partition
   - Ensures each message is processed exactly once by the group
   - If consumers > partitions, some consumers are idle
   - If partitions > consumers, some consumers consume multiple partitions

2. **Load Balancing Across Consumers**
   - Partitions are distributed evenly across consumers
   - Work is balanced based on partition count
   - Adding consumers increases parallelism
   - Removing consumers redistributes partitions

3. **Fault Tolerance**
   - If a consumer fails, its partitions are reassigned
   - Other consumers take over failed consumer's partitions
   - Consumption resumes from last committed offset
   - Automatic recovery without data loss

4. **Offset Tracking**
   - Group maintains committed offsets for each partition
   - Offsets stored in `__consumer_offsets` topic
   - Enables recovery from failures
   - Allows consumers to resume where they left off

**Partition Assignment Strategies:**

**Range Assignment (Default):**
```
Topic with 6 partitions, 2 consumers

Consumer 1: Partition 0, Partition 1, Partition 2
Consumer 2: Partition 3, Partition 4, Partition 5

Algorithm: partitions / consumers
```
- Assigns contiguous partition ranges
- Simple and predictable
- Can lead to imbalance if topics have different partition counts

**RoundRobin Assignment:**
```
Topic with 6 partitions, 2 consumers

Consumer 1: Partition 0, Partition 2, Partition 4
Consumer 2: Partition 1, Partition 3, Partition 5

Algorithm: partitions distributed round-robin
```
- Distributes partitions evenly
- Better balance across topics
- More complex rebalancing

**Sticky Assignment:**
```
Initial State:
Consumer 1: Partition 0, Partition 2
Consumer 2: Partition 1, Partition 3

After Consumer 3 joins:
Consumer 1: Partition 0
Consumer 2: Partition 1
Consumer 3: Partition 2, Partition 3 (stuck to Consumer 1)
```
- Tries to keep previous assignments
- Minimizes partition movement during rebalancing
- Reduces overhead of rebalancing

**CooperativeSticky Assignment:**
```
Gradual Rebalancing:
Consumer 1: Partition 0 → (keeps Partition 0)
Consumer 2: Partition 1 → (keeps Partition 1)
Consumer 3: (joins and gets Partition 2)
```
- Incremental rebalancing
- Consumers don't stop processing during rebalance
- Smoother rebalancing experience

**Rebalancing Process:**

```
Rebalance Trigger: Consumer joins/leaves group

Step 1: Consumer joins group
        │
        ▼
Step 2: Consumer sends JoinGroup request to coordinator
        │
        ▼
Step 3: Coordinator determines new partition assignment
        │
        ▼
Step 4: Coordinator sends SyncGroup request to all consumers
        │
        ▼
Step 5: Consumers stop processing
        │
        ▼
Step 6: Consumers revoke old partitions (onPartitionsRevoked)
        │
        ▼
Step 7: Consumers commit offsets
        │
        ▼
Step 8: Consumers receive new partition assignment (onPartitionsAssigned)
        │
        ▼
Step 9: Consumers start processing new partitions
```

**Rebalance Triggers:**

1. **Consumer Joins Group**
   - New consumer subscribes to topic
   - Triggers rebalance to redistribute partitions
   - Existing consumers may lose partitions

2. **Consumer Leaves Group**
   - Consumer crashes or shuts down
   - Session timeout expires
   - Triggers rebalance to reassign partitions

3. **Topic Partition Count Changes**
   - Partitions added to topic
   - Triggers rebalance to assign new partitions
   - Partitions cannot be decreased

4. **Session Timeout Expires**
   - Consumer doesn't send heartbeat within session.timeout.ms
   - Consumer considered dead
   - Triggers rebalance

5. **Max Poll Interval Exceeded**
   - Consumer doesn't call poll() within max.poll.interval.ms
   - Consumer considered dead
   - Triggers rebalance

**Group Coordinator:**

The group coordinator is a broker responsible for managing a consumer group.

**Coordinator Responsibilities:**
- Manage consumer group membership
- Assign partitions to consumers
- Handle heartbeat and offset commits
- Manage rebalancing process
- Store consumer group metadata

**Coordinator Selection:**
```
Coordinator = hash(group.id) % number_of_brokers

Example:
group.id = "my-consumer-group"
number_of_brokers = 3

hash("my-consumer-group") % 3 = 1
Coordinator = Broker 1
```

**Coordinator Operations:**

1. **JoinGroup Request**
   - Consumer sends JoinGroup to coordinator
   - Coordinator assigns consumer to group
   - Coordinator determines partition assignment

2. **SyncGroup Request**
   - Coordinator sends partition assignment to consumers
   - Consumers acknowledge assignment
   - Group becomes stable

3. **Heartbeat Requests**
   - Consumers send periodic heartbeats to coordinator
   - Coordinator tracks consumer liveness
   - Failure detection via missed heartbeats

4. **Offset Commit Requests**
   - Consumers commit offsets to coordinator
   - Coordinator writes to `__consumer_offsets` topic
   - Offsets persisted for recovery

**__consumer_offsets Topic:**

Internal topic that stores consumer group offsets.

```
__consumer_offsets Topic Structure:

Key: [group.id, topic, partition]
Value: [offset, metadata, commit_timestamp]

Example:
Key: ["my-group", "my-topic", 0]
Value: [42, "processed", 1234567890]
```

**Topic Configuration:**
```properties
# Compacted topic (keeps latest offset per key)
cleanup.policy=compact

# Replication factor
replication.factor=3

# Retention (compacted topics retain latest value)
retention.ms=-1
```

**Consumer Group Example:**

```
Scenario: E-commerce order processing

Topic: orders (6 partitions)
Consumer Group: order-processors

Initial State:
Consumer 1: Partition 0, Partition 1
Consumer 2: Partition 2, Partition 3
Consumer 3: Partition 4, Partition 5

Throughput: 1000 orders/second

Consumer 4 joins:
Rebalance triggered
New assignment:
Consumer 1: Partition 0
Consumer 2: Partition 1
Consumer 3: Partition 2
Consumer 4: Partition 3, Partition 4, Partition 5

Throughput: 1333 orders/second (theoretical)
```

**Consumer Group Configuration:**

```properties
# Group ID (required)
group.id=my-consumer-group

# Session timeout
session.timeout.ms=10000

# Heartbeat interval
heartbeat.interval.ms=3000

# Max poll interval
max.poll.interval.ms=300000

# Assignment strategy
partition.assignment.strategy=org.apache.kafka.clients.consumer.RangeAssignor
# Options: RangeAssignor, RoundRobinAssignor, StickyAssignor, CooperativeStickyAssignor
```

**Consumer Group Best Practices:**

1. **Match Consumers to Partitions**: Optimal when consumers = partitions
2. **Monitor Consumer Lag**: Ensure consumers keep up with producers
3. **Handle Rebalancing Gracefully**: Implement rebalance listener
4. **Use Appropriate Assignment Strategy**: Choose based on your needs
5. **Monitor Group Coordinator**: Ensure coordinator is healthy
6. **Tune Timeouts**: Balance between failure detection and stability
7. **Use Unique Group IDs**: Different groups for different consumption patterns

---

## Scalability

Kafka is designed for horizontal scalability, allowing you to add more resources to handle increased load. This section explains how Kafka scales and how you can scale your Kafka cluster.

### Horizontal Scalability

Kafka achieves horizontal scalability through partitioning, broker addition, and cluster expansion.

#### 1. Partitioning

**What is Partitioning for Scalability?**

Partitioning is the primary mechanism for horizontal scalability in Kafka. By dividing a topic into multiple partitions, you can distribute the load across multiple brokers and consumers.

**Partitioning Strategy:**

```
Single Partition (No Scalability):
┌─────────────────────────────────────┐
│         Topic: orders               │
│  ┌───────────────────────────────┐  │
│  │  Partition 0 (All Messages)   │  │
│  │  Message 1, 2, 3, 4, 5, ...  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
Throughput: Limited by single broker/consumer

Multiple Partitions (Scalable):
┌─────────────────────────────────────┐
│         Topic: orders               │
│  ┌──────────┐ ┌──────────┐ ┌──────┐│
│  │Partition0│ │Partition1│ │Part 2││
│  │Msg 1,4,7 │ │Msg 2,5,8 │ │3,6,9 ││
│  └──────────┘ └──────────┘ └──────┘│
└─────────────────────────────────────┘
Throughput: Scales with number of partitions
```

**How Partitioning Enables Scalability:**

1. **Topics Divided into Multiple Partitions**
   - Each partition can be hosted on different brokers
   - Load is distributed across the cluster
   - No single broker becomes a bottleneck

2. **Parallel Producer Writes**
   - Producers can write to multiple partitions in parallel
   - Multiple network connections to different brokers
   - Write throughput increases with partition count

3. **Parallel Consumer Reads**
   - Multiple consumers can read from different partitions simultaneously
   - Each consumer processes a subset of partitions
   - Consumer throughput scales with partition count

4. **Linear Scalability**
   - Throughput scales approximately linearly with partition count
   - Double partitions ≈ double throughput (theoretical)
   - Practical limits due to overhead

**Partitioning Benefits:**

- **Parallel Processing Across Multiple Consumers**
  - 10 partitions can be consumed by 10 consumers in parallel
  - Each consumer processes a different subset of data
  - Overall throughput increases

- **Load Distribution Across Brokers**
  - Partitions distributed evenly across brokers
  - No single broker overloaded
  - Better resource utilization

- **Isolation of Hot Partitions**
  - If one partition has high traffic, it doesn't affect others
  - Can scale hot partitions independently
  - Better resource isolation

- **Ordered Processing Within Each Partition**
  - Messages within a partition maintain order
  - Enables ordered processing guarantees
  - Critical for event sourcing

**Partitioning Considerations:**

**Too Few Partitions:**
- Limits parallelism (can't add more consumers than partitions)
- Hot partitions become bottlenecks
- Can't scale to handle increased load

**Too Many Partitions:**
- Increased overhead (more files, more file handles)
- Longer rebalancing times (more partitions to reassign)
- More memory usage for metadata
- Slower recovery from failures
- Potential for increased latency

**Rule of Thumb for Partition Count:**

```
Partitions = Number of Consumers × Expected Throughput per Consumer

Example:
- Expected throughput: 100,000 messages/second
- Consumer throughput: 10,000 messages/second
- Number of consumers: 10

Partitions = 10 × 10 = 100 partitions
```

**Maximum Partitions per Broker:**
- Limited by file handles (ulimit)
- Limited by memory (metadata for each partition)
- Typical limit: 10,000 partitions per broker
- Kafka 2.8+ with KRaft: up to 1,000,000 partitions per cluster

#### 2. Broker Addition

**Adding Brokers for Horizontal Scaling:**

Adding more brokers is the primary way to scale a Kafka cluster horizontally.

**Step-by-Step Broker Addition:**

```
Step 1: Start New Broker
- Configure new broker with unique broker.id
- Set log.dirs for data storage
- Start broker process

Step 2: Broker Registers with Zookeeper
- New broker creates ephemeral node in Zookeeper
- Broker metadata stored in /brokers/ids/[broker.id]
- Controller detects new broker

Step 3: Controller Detects New Broker
- Controller watches /brokers/ids
- New broker triggers controller action
- Controller can trigger partition reassignment

Step 4: Partition Reassignment (Optional)
- Move partition replicas to new broker
- Redistribute load across all brokers
- Maintain replication factor

Step 5: Data Replication
- New broker receives partition replicas
- Data copied from existing replicas
- New broker becomes part of ISR
```

**Partition Reassignment Process:**

Partition reassignment moves partition replicas to new brokers to balance load.

```
Before Reassignment:
Broker 0: Partition 0 (Leader), Partition 1 (Leader)
Broker 1: Partition 0 (Follower), Partition 1 (Follower)

After Adding Broker 2:
Broker 0: Partition 0 (Leader)
Broker 1: Partition 1 (Leader)
Broker 2: Partition 0 (Follower), Partition 1 (Follower)
```

**Using the Reassignment Tool:**

```bash
# Step 1: Create topics-to-move JSON file
cat > topics-to-move.json <<EOF
{
  "topics": [
    {"topic": "orders"},
    {"topic": "customers"}
  ],
  "version": 1
}
EOF

# Step 2: Generate reassignment plan
kafka-reassign-partitions.sh --zookeeper localhost:2181 \
  --generate \
  --topics-to-move-json-file topics-to-move.json \
  --broker-list "0,1,2"

# Output: reassignment-plan.json

# Step 3: Review the plan
cat reassignment-plan.json

# Step 4: Execute reassignment
kafka-reassign-partitions.sh --zookeeper localhost:2181 \
  --execute \
  --reassignment-json-file reassignment-plan.json

# Step 5: Verify reassignment (monitor progress)
kafka-reassign-partitions.sh --zookeeper localhost:2181 \
  --verify \
  --reassignment-json-file reassignment-plan.json
```

**Reassignment Characteristics:**

- **Online Operation**: Can be done without downtime
- **Maintains Replication Factor**: Ensures data durability during reassignment
- **Gradual Data Movement**: Data copied incrementally
- **Throttled**: Configurable throttle to avoid overwhelming network
- **Monitored**: Can track progress and verify completion

#### 3. Cluster Expansion

**Scaling Strategies:**

**Add Brokers for Storage Capacity:**
- When disk space is running low
- Add new brokers with additional disk space
- Reassign partitions to new brokers
- Old brokers can be decommissioned

**Add Partitions for Throughput:**
- When throughput is limited by partition count
- Add partitions to existing topics
- Consumers can be added to consume new partitions
- Throughput increases with partition count

**Rebalance Partitions for Even Distribution:**
- After broker addition
- Reassign partitions to balance load
- Ensure even distribution across brokers
- Monitor broker load after rebalancing

**Use Rack-Aware Assignment for Fault Tolerance:**
- Configure broker rack information
- Replicas placed on different racks
- Survives rack failures
- Better geographic distribution

### Vertical Scalability

While Kafka is designed for horizontal scaling, vertical scaling (adding more resources to existing brokers) can also improve performance.

**Resource Optimization:**

1. **Increase Broker CPU Cores**
   - More cores = more parallel processing
   - Better handling of network requests
   - Faster replication
   - Recommended: 8-16 cores for production

2. **Add More Memory for Page Cache**
   - Kafka relies heavily on OS page cache
   - More memory = better cache hit rate
   - Faster reads from cache
   - Recommended: 32-64GB RAM for brokers

3. **Use Faster Storage (SSD/NVMe)**
   - SSDs provide much faster I/O than HDDs
   - Better throughput for reads and writes
   - Lower latency
   - Recommended: NVMe SSDs for production

4. **Optimize Network Bandwidth**
   - 10GbE or higher for production
   - Multiple network interfaces
   - Bonded interfaces for redundancy
   - Better throughput for inter-broker traffic

5. **Tune JVM Heap Size**
   - Kafka doesn't need large heap (relies on page cache)
   - Recommended: 4-6GB heap
   - Larger heap doesn't improve performance
   - Can cause longer GC pauses

### Throughput Optimization

Optimizing throughput involves tuning producers, consumers, and brokers for maximum performance.

#### Producer Optimization

**Batching for Throughput:**

Batching groups messages together before sending, reducing network overhead.

```
No Batching:
Message 1 → Network → Broker
Message 2 → Network → Broker
Message 3 → Network → Broker
Overhead: 3 network round trips

With Batching:
[Message 1, Message 2, Message 3] → Network → Broker
Overhead: 1 network round trip
```

**Batch Size Tuning:**

```properties
# Default: 16KB
batch.size=16384

# High Throughput: 64KB-1MB
batch.size=524288

# Trade-off: Higher batch size = higher throughput, higher latency
```

**Linger Time:**

Wait for batch to fill before sending.

```properties
# Default: 0ms (send immediately)
linger.ms=0

# High Throughput: 10-100ms
linger.ms=50

# Trade-off: Longer linger = better throughput, higher latency
```

**Recommended Configuration for High Throughput:**

```properties
batch.size=32768  # 32KB
linger.ms=10  # Wait up to 10ms for batch to fill
```

**Compression for Throughput:**

Compression reduces network bandwidth and disk I/O.

```
Uncompressed:
Message size: 1KB
1000 messages = 1MB network transfer

Compressed (lz4, 2:1 ratio):
Message size: 500KB
1000 messages = 500KB network transfer
```

**Compression Comparison:**

| Compression Type | Ratio | Speed | CPU Usage | Use Case |
|-----------------|-------|-------|-----------|----------|
| none | 1:1 | Fastest | Lowest | Already compressed data |
| lz4 | 2:1 | Fast | Low-Medium | General purpose (recommended) |
| snappy | 2.5:1 | Medium | Medium | Balanced compression/speed |
| gzip | 3:1 | Slow | High | Maximum compression |
| zstd | 3:1 | Medium | Medium | Best compression/speed balance |

**Recommended:**
```properties
compression.type=lz4  # Best balance of speed and compression
```

**Pipelining:**

Multiple in-flight requests improve throughput.

```properties
# Default: 5
max.in.flight.requests.per.connection=5

# High Throughput: 10-20
max.in.flight.requests.per.connection=10

# Trade-off: Higher = better throughput, potential ordering issues with retries
```

#### Consumer Optimization

**Fetch Size for Throughput:**

Larger fetches reduce round trips and improve throughput.

```properties
# Default: 1 byte minimum
fetch.min.bytes=1

# High Throughput: 1KB minimum
fetch.min.bytes=1024

# Default: 50MB maximum
fetch.max.bytes=52428800

# High Throughput: 100MB maximum
fetch.max.bytes=104857600
```

**Fetch Wait Time:**

Wait for minimum bytes before returning.

```properties
# Default: 500ms
fetch.max.wait.ms=500

# High Throughput: 1000ms
fetch.max.wait.ms=1000

# Trade-off: Longer wait = larger batches, higher latency
```

**Prefetching:**

Consumers fetch ahead of processing to avoid waiting.

```properties
# Default: 1MB per partition
max.partition.fetch.bytes=1048576

# High Throughput: 5MB per partition
max.partition.fetch.bytes=5242880

# Trade-off: More prefetch = better throughput, higher memory usage
```

---

## Fault Tolerance

### Replication Mechanism

Replication is the foundation of Kafka's fault tolerance. By maintaining multiple copies of each partition across different brokers, Kafka can survive broker failures without data loss.

**How Replication Provides Fault Tolerance:**

```
Without Replication (Single Point of Failure):
┌─────────────────────────────────────┐
│         Topic: orders               │
│  ┌───────────────────────────────┐  │
│  │  Partition 0 (Broker 0)      │  │
│  │  All data on single broker  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
If Broker 0 fails → Data lost

With Replication (Fault Tolerant):
┌─────────────────────────────────────┐
│         Topic: orders               │
│  ┌──────────┐ ┌──────────┐ ┌──────┐│
│  │Broker 0  │ │Broker 1  │ │Brk 2 ││
│  │Part 0(L) │ │Part 0(F) │ │Part 0││
│  └──────────┘ └──────────┘ └──────┘│
└─────────────────────────────────────┘
If Broker 0 fails → Broker 1 becomes leader
```

#### Leader Election

**What is Leader Election?**

Leader election is the process of selecting a new leader replica when the current leader fails. The new leader is chosen from the in-sync replicas (ISR) to ensure data consistency.

**Leader Election Process:**

```
Step-by-Step Leader Election Flow:

Step 1: Leader Failure Detected
- Controller detects leader failure via Zookeeper
- Zookeeper session expires for failed broker
- Controller marks leader as offline

Step 2: Controller Selects New Leader
- Controller examines ISR for partition
- Selects new leader from available ISR replicas
- Preference: replica with highest offset (most up-to-date)

Step 3: Controller Updates Metadata
- Controller updates partition metadata in Zookeeper
- New leader information stored in /brokers/topics/[topic]/partitions/[partition]/state
- All brokers notified of new leader

Step 4: New Leader Starts Serving Requests
- New leader accepts write requests from producers
- New leader serves read requests from consumers
- Followers start replicating from new leader

Step 5: Followers Start Replicating
- Followers detect leader change
- Followers start fetching from new leader
- Followers catch up to new leader
- ISR updated if necessary
```

**Leader Election Example:**

```
Initial State:
Partition 0: Leader=Broker 0, ISR=[Broker 0, Broker 1, Broker 2]

Broker 0 fails:
- Controller detects failure
- Controller selects new leader from ISR
- Broker 1 selected (most up-to-date follower)

After Election:
Partition 0: Leader=Broker 1, ISR=[Broker 1, Broker 2]

Broker 0 recovers:
- Broker 0 rejoins as follower
- Broker 0 catches up to Broker 1
- Broker 0 rejoins ISR

Recovered State:
Partition 0: Leader=Broker 1, ISR=[Broker 0, Broker 1, Broker 2]
```

**Leader Election Strategies:**

- **Unclean Leader Election**: Can elect out-of-sync replica if ISR empty
- **Clean Leader Election**: Only elect in-sync replicas
- Configured via `unclean.leader.election.enable`

```properties
unclean.leader.election.enable=false  # Recommended for production
```

**Unclean Leader Election Scenario:**

```
Scenario:
Partition 0: Leader=Broker 0, ISR=[Broker 0, Broker 1]

Both Broker 0 and Broker 1 fail:
- No ISR replicas available
- Broker 2 is out-of-sync (OSR)

With unclean.leader.election.enable=true:
- Controller elects Broker 2 as leader
- Data may be lost (messages not replicated to Broker 2)
- Availability > consistency

With unclean.leader.election.enable=false (recommended):
- No leader elected
- Partition remains unavailable
- Consistency > availability
- Requires manual intervention
```

#### In-Sync Replicas (ISR)

**What is ISR?**

In-Sync Replicas (ISR) are replicas that are fully caught up with the leader. Only ISR replicas are eligible to become leader and are guaranteed to have the latest data.

**ISR Maintenance:**

```
ISR Dynamics:

Initial State:
Partition 0: [Leader: Broker 0, ISR: Broker 0, Broker 1, Broker 2]

Broker 2 falls behind:
- Broker 2 hasn't fetched for 30 seconds
- Controller removes Broker 2 from ISR
- Partition 0: [Leader: Broker 0, ISR: Broker 0, Broker 1]
- Writes still accepted (ISR size >= min.insync.replicas)

Broker 2 catches up:
- Broker 2 starts fetching again
- Broker 2 catches up to leader
- Controller adds Broker 2 back to ISR
- Partition 0: [Leader: Broker 0, ISR: Broker 0, Broker 1, Broker 2]
```

**ISR Configuration:**

```properties
# Replica lag time (how long a replica can be behind before being removed from ISR)
replica.lag.time.max.ms=30000  # 30 seconds

# Minimum in-sync replicas (must have this many replicas for writes)
min.insync.replicas=2  # Ensures at least 2 replicas have data before acknowledging
```

**Why ISR Matters:**

- **Data Durability**: Only ISR replicas are guaranteed to have the latest data
- **Leader Election**: Only ISR replicas can become leader (by default)
- **Write Guarantees**: Producer waits for ISR acknowledgments
- **Consistency**: Consumers can only read up to the high watermark

**ISR Shrinking:**
- When follower falls behind
- Removed from ISR
- Leader continues without it
- ISR size decreases
- May fail writes if ISR size < min.insync.replicas

**ISR Expansion:**
- When follower catches up
- Rejoins ISR
- ISR size increases
- Replication factor restored
- Writes can proceed normally

### Broker Failure Handling

Broker failures are a common failure scenario. Kafka handles broker failures automatically through replication and leader election.

**Failure Detection:**

```
Broker Failure Detection Flow:

1. Broker fails (crash, network partition, etc.)
        │
        ▼
2. Zookeeper session expires for broker
        │
        ▼
3. Controller detects broker failure
        │
        ▼
4. Controller initiates leader election for partitions on failed broker
        │
        ▼
5. New leaders elected from ISR
        │
        ▼
6. Cluster continues operating with reduced capacity
```

**Failure Detection Mechanisms:**

1. **Zookeeper Session Expiration**
   - Each broker maintains a session with Zookeeper
   - Session expires if broker doesn't send heartbeat
   - Default session timeout: 6 seconds
   - Configurable via `zookeeper.session.timeout.ms`

2. **Controller Monitoring**
   - Controller monitors broker liveness via Zookeeper
   - Controller watches `/brokers/ids` path
   - Immediate notification when broker session expires

**Failure Recovery:**

```
Broker Recovery Flow:

1. Failed broker restarts
        │
        ▼
2. Broker reconnects to Zookeeper
        │
        ▼
3. Broker registers with Zookeeper
        │
        ▼
4. Controller detects broker recovery
        │
        ▼
5. Broker rejoins ISR for partitions
        │
        ▼
6. Broker catches up with leader
        │
        ▼
7. Broker fully operational
```

**Broker Failure Example:**

```
Initial State:
Broker 0: Partition 0 (Leader), Partition 2 (Leader)
Broker 1: Partition 0 (Follower), Partition 1 (Leader)
Broker 2: Partition 1 (Follower), Partition 2 (Follower)

Broker 0 fails:
- Controller detects failure
- Leader election for Partition 0: Broker 1 becomes leader
- Leader election for Partition 2: Broker 2 becomes leader

After Failure:
Broker 0: Offline
Broker 1: Partition 0 (Leader), Partition 1 (Leader)
Broker 2: Partition 1 (Follower), Partition 2 (Leader)

Broker 0 recovers:
- Broker 0 rejoins cluster
- Broker 0 becomes follower for Partition 0 and Partition 2
- Broker 0 catches up with leaders
- Broker 0 rejoins ISR
```

**Configuration for Broker Failure Handling:**

```properties
# Zookeeper session timeout
zookeeper.session.timeout.ms=6000  # 6 seconds

# Zookeeper connection timeout
zookeeper.connection.timeout.ms=6000  # 6 seconds

# Controller socket timeout
controller.socket.timeout.ms=30000  # 30 seconds

# Replica fetch timeout
replica.fetch.timeout.ms=30000  # 30 seconds

# Replica lag time (before removing from ISR)
replica.lag.time.max.ms=30000  # 30 seconds
```

### Network Partitions

Network partitions occur when the network splits the cluster into two or more groups that cannot communicate with each other.

**What is a Network Partition?**

A network partition is a failure scenario where the network splits, preventing some brokers from communicating with others. This can lead to split-brain scenarios where different parts of the cluster believe they are the active cluster.

**Behavior During Network Partition:**

```
Network Partition Scenario:

Initial State:
Cluster: Broker 0, Broker 1, Broker 2 (all connected)
Controller: Broker 0

Network Partition:
Group A: Broker 0, Broker 1 (can communicate)
Group B: Broker 2 (isolated)

Group A Behavior:
- Broker 0 (controller) still sees Broker 1
- Broker 0 marks Broker 2 as failed
- Leader election for partitions on Broker 2
- Group A continues operating

Group B Behavior:
- Broker 2 cannot communicate with controller
- Broker 2 cannot become controller
- Broker 2 cannot serve clients
- Broker 2 is effectively down

Result: Split-brain scenario
- Group A believes it's the active cluster
- Group B is isolated
- Data divergence possible
```

**Split Brain Prevention:**

- **Zookeeper Quorum Required**: Zookeeper requires majority quorum for operations
- **Majority of Brokers Must Be Reachable**: Controller requires majority of brokers
- **Controller Requires Quorum**: Prevents multiple controllers
- **Prevents Multiple Leaders**: Only one active leader per partition

**Network Partition Handling:**

```
Network Partition Recovery Flow:

1. Network partition occurs
        │
        ▼
2. Minority partition isolated
        │
        ▼
3. Minority cannot elect controller
        │
        ▼
4. Partitions in minority lose leaders
        │
        ▼
5. No writes accepted until partition heals
        │
        ▼
6. Network partition heals
        │
        ▼
7. Isolated brokers reconnect
        │
        ▼
8. Controller detects reconnected brokers
        │
        ▼
9. Data reconciliation if necessary
        │
        ▼
10. Cluster fully operational
```

**Mitigation Strategies:**

1. **Configure Appropriate Timeouts**
   - Set appropriate session timeouts
   - Avoid false positives for temporary network issues
   - Balance between failure detection and stability

```properties
# Zookeeper session timeout
zookeeper.session.timeout.ms=6000  # 6 seconds

# Increase for unstable networks
zookeeper.session.timeout.ms=18000  # 18 seconds
```

2. **Monitor Network Connectivity**
   - Monitor network latency between brokers
   - Alert on network partitions
   - Use network monitoring tools

3. **Use Rack-Aware Assignment**
   - Place replicas across different racks/availability zones
   - Survives rack-level network partitions
   - Better geographic distribution

```properties
# Configure broker rack
broker.rack=rack1

# Rack-aware assignment ensures replicas on different racks
```

4. **Majority Quorum (KRaft Mode)**
   - KRaft mode uses majority quorum for decisions
   - Requires majority of brokers to be available
   - Prevents split-brain scenarios
   - Better handling of network partitions

**Best Practices for Network Partitions:**

1. **Use Appropriate Timeouts**: Balance failure detection with stability
2. **Monitor Network Health**: Detect partitions early
3. **Use Rack-Aware Assignment**: Survive rack-level failures
4. **Consider KRaft Mode**: Better handling of network partitions
5. **Have Recovery Procedures**: Manual intervention if needed

### Data Durability

Data durability ensures that messages are not lost even in the event of failures. Kafka provides multiple mechanisms to ensure data durability.

#### Acknowledgment Levels

The acknowledgment level determines how many replicas must acknowledge a write before the producer considers it successful.

**acks=0 (Fire and Forget):**

```
Producer → Broker (no acknowledgment)
Producer assumes success immediately
```

- **Producer doesn't wait for acknowledgment**
- **Lowest latency, lowest durability**
- **Data can be lost on broker failure**
- **Use case**: Non-critical data, metrics, logs where loss is acceptable

**acks=1 (Leader Acknowledgment):**

```
Producer → Broker Leader
Broker Leader → Producer (acknowledgment)
```

- **Producer waits for leader acknowledgment**
- **Balanced latency and durability**
- **Data can be lost if leader fails before replication**
- **Use case**: General purpose applications

**acks=all (All ISR Acknowledgment):**

```
Producer → Broker Leader
Broker Leader → Followers (replicate)
Followers → Broker Leader (acknowledgment)
Broker Leader → Producer (acknowledgment)
```

- **Producer waits for all in-sync replicas**
- **Highest durability, higher latency**
- **Requires `min.insync.replicas` configuration**
- **Use case**: Critical data, financial transactions, compliance

**Recommended Configuration for Durability:**

```properties
acks=all
min.insync.replicas=2  # Ensures at least 2 replicas have data
```

**Acknowledgment Level Comparison:**

| acks Value | Latency | Durability | Use Case |
|------------|---------|------------|----------|
| 0 | Lowest | Lowest | Non-critical data, metrics |
| 1 | Medium | Medium | General purpose (default) |
| all | Highest | Highest | Critical data, financial transactions |

#### Disk Flush

**Log Flush Behavior:**

Kafka relies on OS page cache for performance rather than syncing every write to disk. Durability is achieved through replication rather than disk flushes.

```
Write Flow:
1. Producer sends message to broker
2. Broker writes to OS page cache (in memory)
3. Broker acknowledges to producer
4. OS flushes page cache to disk (asynchronously)
5. Data persisted on disk
```

**Why Kafka Doesn't Sync Every Write:**

- **Performance**: Syncing every write to disk is slow
- **Replication**: Replication provides durability guarantees
- **OS Page Cache**: OS handles flushing efficiently
- **Trade-off**: Better throughput for slight durability risk

**Configuration:**

```properties
# Never flush based on message count (rely on OS)
log.flush.interval.messages=9223372036854775807

# Never flush based on time (rely on OS)
log.flush.interval.ms=9223372036854775807

# Note: Kafka recommends leaving these at default (effectively disabled)
# Durability is achieved through replication, not disk flush
```

**When to Force Disk Flush:**

- **Compliance Requirements**: Some regulations require immediate disk persistence
- **Extremely Critical Data**: When even replication is not sufficient
- **Trade-off**: Significantly reduced throughput

### Controller Failover

The controller is a special broker responsible for cluster management. Controller failures are handled through automatic re-election.

**Controller Failure Process:**

```
Controller Failure Flow:

1. Controller broker fails
        │
        ▼
2. Zookeeper detects controller failure
        │
        ▼
3. New controller elected from active brokers
        │
        ▼
4. Controller epoch incremented
        │
        ▼
5. New controller reads metadata from Zookeeper
        │
        ▼
6. Controller rebuilds cluster state
        │
        ▼
7. Controller resumes cluster management
```

**Controller Epoch:**

The controller epoch is a monotonically increasing counter that prevents stale controllers from making decisions.

```
Controller Epoch Mechanism:

Initial State:
Controller: Broker 0, Epoch: 0

Controller Failure:
- Broker 0 fails
- Broker 1 elected as new controller
- Epoch incremented to 1

Stale Controller Prevention:
- If Broker 0 recovers and tries to act as controller
- Other brokers reject requests with epoch 0
- Current epoch is 1
- Broker 0 must rejoin as regular broker
```

**Controller Epoch Properties:**

- **Monotonically Increasing Counter**: Never decreases
- **Prevents Stale Controllers**: Old controllers cannot make decisions
- **Used in All Controller Requests**: Every request includes epoch
- **Stored in Zookeeper**: Persisted across failures

**Controller Failure Example:**

```
Initial State:
Controller: Broker 0
Broker 0: Controller + Partition 0 (Leader)
Broker 1: Partition 0 (Follower)
Broker 2: Partition 0 (Follower)
Epoch: 0

Broker 0 (controller) fails:
- Zookeeper detects failure
- Broker 1 becomes controller
- Epoch incremented to 1
- Broker 1 elects new leader for Partition 0 (itself)

After Recovery:
Controller: Broker 1
Broker 0: Offline
Broker 1: Controller + Partition 0 (Leader)
Broker 2: Partition 0 (Follower)
Epoch: 1
```

---

## Error Handling

Kafka provides comprehensive error handling mechanisms for producers, consumers, and brokers. This section explains how to handle errors and ensure reliable message processing.

### Producer Error Handling

#### Retry Mechanism

Producers automatically retry on transient failures to ensure message delivery.

**Retry Configuration:**

```properties
# Number of retries before giving up
retries=2147483647  # Retry indefinitely (effectively)

# Backoff between retries (milliseconds)
retry.backoff.ms=100  # 100ms backoff

# Total time for a message to be acknowledged (milliseconds)
delivery.timeout.ms=120000  # 2 minutes
```

**Retry Behavior:**

```
Retry Flow:

Producer sends message
        │
        ▼
Error occurs (transient)
        │
        ▼
Wait retry.backoff.ms
        │
        ▼
Retry send
        │
        ▼
Success? → Yes → Acknowledge to producer
        │
        No
        ▼
Retry count exceeded?
        │
        No → Continue retrying
        │
        Yes → Fail with exception
```

**Retries On:**
- **Transient Errors**: Network timeouts, connection failures
- **Network Errors**: Temporary network issues
- **Leader Not Available**: Leader election in progress
- **Not Leader For Partition**: Request sent to wrong broker

**No Retries On:**
- **Permanent Errors**: Invalid message format, authentication failures
- **Message Too Large**: Message exceeds max size
- **Unknown Topic**: Topic doesn't exist
- **Authorization Failures**: No permission to write

**Idempotent Producer:**

The idempotent producer prevents duplicate messages when retries occur.

**What is Idempotence?**

Idempotence ensures that sending the same message multiple times has the same effect as sending it once. This prevents duplicates when the producer retries due to network issues.

**Configuration:**

```properties
enable.idempotence=true  # Enable idempotent producer
max.in.flight.requests.per.connection=5  # Must be <= 5 when idempotence enabled
acks=all  # Required for idempotence
retries=2147483647  # Retry indefinitely
```

**Idempotence Mechanism:**

```
Idempotent Producer Flow:

1. Producer assigns sequence number to each message
   Producer: {sequence: 1, message: "order-123"}
   Producer: {sequence: 2, message: "order-124"}

2. Producer sends message to broker
   Producer → Broker: {sequence: 1, message: "order-123"}

3. Broker tracks last sequence number per partition
   Broker Partition 0: last_sequence = 1

4. If retry occurs, broker detects duplicate
   Producer → Broker: {sequence: 1, message: "order-123"} (retry)
   Broker: sequence 1 <= last_sequence 1 → Duplicate detected
   Broker: Reject duplicate, acknowledge original

5. Ordering guarantees maintained
   Messages processed in sequence number order
```

**Benefits:**
- **Prevents Duplicate Messages**: Retries don't cause duplicates
- **Exactly-Once Semantics per Partition**: Each message processed exactly once
- **Automatic**: No application code changes needed
- **Ordering Guarantees**: Messages maintain order within partition

#### Transactional Producer

The transactional producer provides exactly-once semantics across multiple partitions by using transactions.

**What is Exactly-Once Semantics?**

Exactly-once semantics ensures that each message is processed exactly once, no more and no less. This is critical for financial transactions, inventory updates, and other scenarios where duplicates or data loss are unacceptable.

**Exactly-Once Semantics:**

- **Atomic Writes Across Multiple Partitions**: All messages in a transaction succeed or fail together
- **Transactional API**: begin/commit/abort transactions
- **Requires transactional.id**: Unique identifier for the producer
- **Coordinates with Consumer**: Consumers with `isolation.level=read_committed` see only committed messages

**Configuration:**

```properties
# Unique transactional ID (required for transactions)
transactional.id=my-unique-transactional-id

# Idempotence required for transactions
enable.idempotence=true

# Required for transactional writes
acks=all
```

**Transaction Flow:**

```
Transaction Flow:

1. Producer initializes transaction
   producer.initTransactions()

2. Producer begins transaction
   producer.beginTransaction()

3. Producer sends messages to multiple partitions
   producer.send(new ProducerRecord<>("orders", "order-123"))
   producer.send(new ProducerRecord<>("inventory", "item-456"))
   producer.send(new ProducerRecord<>("notifications", "notify-789"))

4. Producer commits transaction
   producer.commitTransaction()
   All messages become visible to consumers

OR

4. Producer aborts transaction (on error)
   producer.abortTransaction()
   All messages are discarded
```

**Transaction Coordinator:**

The transaction coordinator is a broker component that manages transaction state.

**Coordinator Responsibilities:**
- **Manages Transaction State**: Tracks active transactions
- **Stores Transaction Metadata**: In internal topic `__transaction_state`
- **Handles Transaction Recovery**: Recovers incomplete transactions on startup
- **Coordinates with Brokers**: Ensures all brokers commit or abort

**Transaction Coordinator Flow:**

```
Transaction Coordinator Flow:

1. Producer sends BeginTransaction request to coordinator
        │
        ▼
2. Coordinator assigns transaction ID
        │
        ▼
3. Producer sends messages to partition leaders
        │
        ▼
4. Partition leaders add messages to transaction log
        │
        ▼
5. Producer sends CommitTransaction request to coordinator
        │
        ▼
6. Coordinator sends commit markers to all partitions
        │
        ▼
7. Messages become visible to read-committed consumers
```

**Consumer Isolation Level:**

Consumers must use `read_committed` isolation level to see only committed messages.

```properties
# Consumer configuration
isolation.level=read_committed  # Only read committed messages
```

**Isolation Level Comparison:**

| Isolation Level | Behavior | Use Case |
|----------------|----------|----------|
| read_uncommitted | Read all messages, including aborted | High throughput, eventual consistency |
| read_committed | Only read committed messages | Exactly-once semantics, financial transactions |

#### Error Codes

Kafka uses error codes to communicate failures to producers and consumers.

**Common Producer Errors:**

| Error Code | Description | Retryable | Action |
|------------|-------------|-----------|--------|
| NETWORK_EXCEPTION | Network I/O error | Yes | Automatic retry |
| LEADER_NOT_AVAILABLE | Leader election in progress | Yes | Automatic retry |
| NOT_LEADER_FOR_PARTITION | Request sent to wrong broker | Yes | Automatic retry (metadata refresh) |
| UNKNOWN_TOPIC_OR_PARTITION | Topic/partition doesn't exist | No | Create topic or check configuration |
| MESSAGE_TOO_LARGE | Message exceeds max size | No | Reduce message size or increase max size |
| INVALID_RECORD | Invalid message format | No | Fix message format |
| OFFSET_OUT_OF_RANGE | Invalid offset (consumer error) | No | Seek to valid offset |

**Handling Producer Errors:**

```java
try {
    producer.send(record, new Callback() {
        @Override
        public void onCompletion(RecordMetadata metadata, Exception exception) {
            if (exception != null) {
                // Handle error
                if (exception instanceof RetriableException) {
                    // Will be retried automatically
                    log.warn("Retriable error", exception);
                } else {
                    // Non-retriable error
                    log.error("Non-retriable error", exception);
                    // Handle appropriately (e.g., send to DLQ)
                }
            }
        }
    });
} catch (Exception e) {
    // Handle serialization or other errors
    log.error("Error sending message", e);
}
```

### Consumer Error Handling

Consumers need to handle errors related to offset management, rebalancing, and processing failures.

#### Offset Management

Offset management tracks the consumer's position in each partition. Proper offset management is critical for reliable message processing.

**Auto Commit:**

Auto commit automatically commits offsets at regular intervals.

```properties
enable.auto.commit=true  # Enable auto commit (default)
auto.commit.interval.ms=5000  # Commit every 5 seconds
```

**Auto Commit Behavior:**

```
Auto Commit Flow:

Consumer processes messages
        │
        ▼
Auto commit interval expires (5 seconds)
        │
        ▼
Consumer commits current offsets
        │
        ▼
Offsets stored in __consumer_offsets topic
```

**Auto Commit Issues:**

- **Can Cause Duplicates**: If consumer fails after processing but before commit, messages are reprocessed
- **Can Cause Missed Messages**: If consumer fails before processing but after commit, messages are lost
- **No Control Over When to Commit**: Commits happen automatically based on time

**Manual Commit:**

Manual commit gives the application control over when to commit offsets.

```properties
enable.auto.commit=false  # Disable auto commit
```

**Manual Commit Types:**

```java
// Synchronous commit (blocks until acknowledged)
consumer.commitSync();

// Asynchronous commit (non-blocking)
consumer.commitAsync(new OffsetCommitCallback() {
    @Override
    public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
        if (exception != null) {
            // Handle commit error
            log.error("Commit failed", exception);
        }
    }
});
```

**Manual Commit Behavior:**

```
Manual Commit Flow (Synchronous):

Consumer processes messages
        │
        ▼
Application calls consumer.commitSync()
        │
        ▼
Consumer sends commit request to coordinator
        │
        ▼
Consumer waits for acknowledgment
        │
        ▼
Coordinator writes offsets to __consumer_offsets
        │
        ▼
Acknowledgment received
        │
        ▼
commitSync() returns
```

**Offset Management Strategies:**

**At-Most-Once Semantics (Can Lose Messages):**

```java
// Commit before processing
consumer.commitSync();
processRecords(records);
```

- Offsets committed before processing
- If processing fails, messages are lost
- Lowest latency
- Use for: non-critical data where loss is acceptable

**At-Least-Once Semantics (Can Duplicate Messages):**

```java
// Commit after processing
processRecords(records);
consumer.commitSync();
```

- Offsets committed after processing
- If processing fails, messages are reprocessed (duplicates)
- Higher latency
- Use for: critical data where duplicates are acceptable

**Exactly-Once Semantics (No Loss, No Duplicates):**

```java
// Use transactions (requires transactional producer/consumer)
props.put("isolation.level", "read_committed");
```

- Requires transactional producer and consumer
- Highest complexity
- Use for: financial transactions, exactly-once requirements

#### Rebalancing

Rebalancing occurs when the group membership changes, redistributing partitions among consumers.

**Rebalance Triggers:**

```
Rebalance Triggers:

1. Consumer Joins Group
   - New consumer subscribes to topic
   - Triggers rebalance to redistribute partitions
   - Existing consumers may lose partitions

2. Consumer Leaves Group
   - Consumer crashes or shuts down
   - Session timeout expires
   - Triggers rebalance to reassign partitions

3. Topic Partition Count Changes
   - Partitions added to topic
   - Triggers rebalance to assign new partitions
   - Partitions cannot be decreased

4. Session Timeout Expires
   - Consumer doesn't send heartbeat within session.timeout.ms
   - Consumer considered dead
   - Triggers rebalance

5. Max Poll Interval Exceeded
   - Consumer doesn't call poll() within max.poll.interval.ms
   - Consumer considered dead
   - Triggers rebalance
```

**Rebalance Protocol:**

```
Rebalance Protocol Flow:

1. Consumer joins group
        │
        ▼
2. Group coordinator assigns partitions
        │
        ▼
3. Consumers stop processing
        │
        ▼
4. Consumers revoke old partitions (onPartitionsRevoked)
        │
        ▼
5. Consumers commit offsets
        │
        ▼
6. Consumers fetch assigned partitions (onPartitionsAssigned)
        │
        ▼
7. Consumers resume processing
```

**Rebalance Listener:**

The rebalance listener allows the application to handle partition revocation and assignment.

```java
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Called before partitions are revoked
        // Commit offsets before losing partitions
        consumer.commitSync();
        log.info("Partitions revoked: {}", partitions);
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Called after partitions are assigned
        // Seek to appropriate offsets
        for (TopicPartition partition : partitions) {
            consumer.seek(partition, getOffsetForPartition(partition));
        }
        log.info("Partitions assigned: {}", partitions);
    }
});
```

**Rebalance Best Practices:**

1. **Commit Offsets Before Revocation**: Ensure no data loss during rebalance
2. **Handle Assignment Gracefully**: Seek to appropriate offsets on new partitions
3. **Minimize Rebalance Impact**: Use cooperative sticky assignor for smoother rebalancing
4. **Monitor Rebalance Frequency**: Too frequent rebalancing indicates instability

#### Consumer Failure

Consumers can fail due to crashes, network issues, or processing errors. Kafka detects consumer failures and triggers rebalancing.

**Failure Detection:**

```
Consumer Failure Detection Flow:

1. Consumer sends heartbeats to coordinator
        │
        ▼
2. Coordinator tracks heartbeat timing
        │
        ▼
3. Session timeout expires (no heartbeat)
        │
        ▼
4. Consumer considered dead
        │
        ▼
5. Coordinator triggers rebalance
        │
        ▼
6. Partitions reassigned to other consumers
```

**Failure Detection Mechanisms:**

1. **Heartbeat Mechanism**
   - Consumers send periodic heartbeats to coordinator
   - Default interval: 3 seconds
   - Configurable via `heartbeat.interval.ms`

2. **Session Timeout**
   - How long consumer can be inactive
   - Default: 10 seconds
   - Configurable via `session.timeout.ms`

3. **Max Poll Interval**
   - How long between poll() calls
   - Default: 5 minutes
   - Configurable via `max.poll.interval.ms`

**Configuration:**

```properties
# Session timeout (how long consumer can be inactive)
session.timeout.ms=10000  # 10 seconds

# Heartbeat interval (how often to send heartbeat)
heartbeat.interval.ms=3000  # 3 seconds
# Should be less than session.timeout.ms / 3

# Max poll interval (maximum time between polls)
max.poll.interval.ms=300000  # 5 minutes
```

**Handling Processing Errors:**

```java
try {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        try {
            processRecord(record);
        } catch (Exception e) {
            // Handle processing error
            log.error("Error processing record", e);
            // Send to dead letter queue
            sendToDeadLetterQueue(record);
        }
    }
    consumer.commitSync();
} catch (Exception e) {
    // Handle consumer error
    log.error("Consumer error", e);
}
```

### Broker Error Handling

Brokers handle errors related to log corruption, disk failures, and other system-level issues.

#### Log Corruption

Log corruption can occur due to disk failures, software bugs, or other issues. Kafka provides mechanisms to detect and handle corrupted logs.

**Corruption Detection:**

Kafka uses checksums to detect corrupted messages.

```
Corruption Detection Flow:

1. Kafka writes message with checksum
   Message: {data: "...", checksum: CRC32(data)}

2. On read, Kafka validates checksum
   Read: {data: "...", checksum: CRC32(data)}
   Validate: CRC32(data) == checksum

3. If checksum mismatch
   Corruption detected
   Error logged
   Configurable action taken
```

**Corruption Handling Options:**

```properties
# Enable checksum validation
message.checksum.crc32=true

# Log cleaner (compacts logs, removes old records)
log.cleaner.enable=true

# Action on corruption (Kafka 2.8+)
log.cleaner.delete.retention.ms=0  # Delete corrupted messages
```

**Corruption Recovery Strategies:**

1. **Skip Corrupted Messages**
   - Consumer skips corrupted messages
   - Continues processing
   - Some data loss

2. **Rebuild from Replicas**
   - Delete corrupted log
   - Rebuild from leader/follower
   - No data loss

3. **Restore from Backup**
   - Restore from backup
   - May have data loss
   - Depends on backup frequency

**Configuration:**

```properties
# Checksum type (CRC32, CRC32C, NONE)
checksum.type=crc32  # Default

# Log cleaner settings
log.cleaner.enable=true
log.cleaner.threads=4
log.cleaner.dedupe.buffer.size=134217728  # 128MB
```

#### Disk Failure

Disk failures can cause data loss and broker downtime. Proper configuration and monitoring are critical.

**Failure Handling:**

```
Disk Failure Handling Flow:

1. Broker detects disk failure
        │
        ▼
2. Broker logs error
        │
        ▼
3. Broker shuts down affected partitions
        │
        ▼
4. Controller triggers leader election
        │
        ▼
5. Failed broker removed from cluster
        │
        ▼
6. Data recovered from replicas
```

**Multi-Disk Configuration:**

Using multiple disks provides better performance and fault tolerance.

```properties
# Multiple log directories (comma-separated)
log.dirs=/mnt/kafka-1,/mnt/kafka-2,/mnt/kafka-3
```

**Benefits of Multi-Disk Configuration:**

- **Better Performance**: Parallel I/O across disks
- **Fault Tolerance**: Survives single disk failure
- **Isolation**: Different partitions on different disks
- **Scalability**: Add more disks for more capacity

**Disk Failure Recovery:**

```
Disk Failure Recovery:

1. Identify failed disk
        │
        ▼
2. Replace failed disk
        │
        ▼
3. Reconfigure log.dirs to include new disk
        │
        ▼
4. Restart broker
        │
        ▼
5. Reassign partitions to new disk
        │
        ▼
6. Data recovered from replicas
```

**Monitoring Disk Health:**

- Monitor disk I/O latency
- Monitor disk usage
- Monitor disk errors
- Alert on disk failures

#### Out of Memory (OOM)

Brokers can run out of memory due to high load or misconfiguration.

**OOM Prevention:**

```properties
# JVM heap size (Kafka doesn't need large heap)
-Xms4g -Xmx4g  # 4GB heap (recommended)

# Leave memory for OS page cache
# Kafka relies heavily on page cache for performance
```

**OOM Symptoms:**

- Broker crashes with OutOfMemoryError
- Slow response times
- High GC pauses
- Increased latency

**OOM Mitigation:**

1. **Reduce Heap Size**
   - Kafka doesn't need large heap
   - Rely on OS page cache
   - Recommended: 4-6GB heap

2. **Increase Physical Memory**
   - Add more RAM
   - More page cache
   - Better performance

3. **Reduce Partition Count**
   - Each partition uses memory
   - Reduce partition count per broker
   - Spread partitions across more brokers

4. **Tune GC Settings**
   - Use G1GC for large heaps
   - Tune GC parameters
   - Monitor GC logs

**GC Configuration:**

```bash
# Use G1GC for better performance
-XX:+UseG1GC

# GC logging
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/log/kafka/gc.log
```

---

## Performance Optimization

Performance optimization involves tuning producers, consumers, and brokers for maximum throughput and minimal latency. This section provides specific tuning values and guidelines.

### Producer Performance

#### Batching

Batching groups messages together before sending, reducing network overhead and improving throughput.

**Batch Size Tuning:**

```properties
# Default: 16KB
batch.size=16384

# High Throughput: 64KB-1MB
batch.size=524288  # 512KB

# Low Latency: 16KB-32KB
batch.size=32768  # 32KB
```

**Linger Time Tuning:**

```properties
# Default: 0ms (send immediately)
linger.ms=0

# High Throughput: 10-100ms
linger.ms=50  # Wait up to 50ms for batch to fill

# Low Latency: 0-5ms
linger.ms=5  # Wait up to 5ms
```

**Optimization Guidelines:**

```
Batch Size vs. Throughput:

Small Batches (16KB):
- Lower latency
- Higher network overhead
- Lower throughput
- Use for: low-latency requirements

Large Batches (512KB-1MB):
- Higher latency
- Lower network overhead
- Higher throughput
- Use for: high-throughput requirements

Recommended Tuning:
1. Start with batch.size=32768 and linger.ms=10
2. Monitor batch-size-avg metric
3. Increase batch.size if batches are filling quickly
4. Increase linger.ms if batches are not filling
5. Balance between latency and throughput
```

**Trade-off:**
- **Higher batch size = higher throughput, higher latency**
- **Lower batch size = lower latency, lower throughput**

#### Compression

Compression reduces network bandwidth and disk I/O at the cost of CPU usage.

**Compression Types Comparison:**

| Compression Type | Ratio | Speed | CPU Usage | Use Case |
|-----------------|-------|-------|-----------|----------|
| none | 1:1 | Fastest | Lowest | Already compressed data |
| lz4 | 2:1 | Fast | Low-Medium | General purpose (recommended) |
| snappy | 2.5:1 | Medium | Medium | Balanced compression/speed |
| gzip | 3:1 | Slow | High | Maximum compression |
| zstd | 3:1 | Medium | Medium | Best compression/speed balance |

**Configuration:**

```properties
# Recommended for general purpose
compression.type=lz4

# For maximum compression
compression.type=zstd

# For already compressed data (images, videos)
compression.type=none
```

**Compression Guidelines:**

```
When to Use Compression:

Use Compression:
- Text data (JSON, XML, logs)
- High network bandwidth usage
- Limited disk I/O
- CPU resources available

Don't Use Compression:
- Already compressed data (images, videos)
- Very small messages (< 1KB)
- CPU-constrained environments
- Low network bandwidth usage

Benchmarking:
1. Test each compression type with your data
2. Monitor CPU usage and throughput
3. Choose best balance for your workload
4. Consider lz4 or zstd for best overall performance
```

#### Buffering

Buffering holds messages in memory before sending to the network.

**Buffer Configuration:**

```properties
# Default: 32MB
buffer.memory=33554432

# High Throughput: 64MB-256MB
buffer.memory=134217728  # 128MB

# Low Memory: 16MB-32MB
buffer.memory=16777216  # 16MB
```

**Buffer Management:**

```
Buffer Behavior:

Buffer Not Full:
- Messages added to buffer
- Batches sent when full or linger expires
- Normal operation

Buffer Full:
- Producer blocks (send() call blocks)
- Waits for buffer space to free up
- Can cause latency spikes
- Consider increasing buffer.memory

Buffer Tuning:
1. Monitor buffer-available-bytes metric
2. If buffer frequently full, increase buffer.memory
3. If buffer never full, can decrease buffer.memory
4. Balance between memory usage and throughput
```

**Configuration Guidelines:**

```properties
# For high-throughput producers
buffer.memory=134217728  # 128MB
batch.size=524288  # 512KB
linger.ms=50

# For low-latency producers
buffer.memory=33554432  # 32MB
batch.size=16384  # 16KB
linger.ms=0

# For memory-constrained environments
buffer.memory=16777216  # 16MB
batch.size=8192  # 8KB
linger.ms=0
```

### Consumer Performance

#### Fetch Optimization

Fetch optimization involves tuning how much data consumers fetch from brokers in each request.

**Fetch Configuration:**

```properties
# Minimum bytes to wait for (default: 1 byte)
fetch.min.bytes=1

# High Throughput: Wait for more data
fetch.min.bytes=1024  # 1KB minimum

# Maximum bytes in a single fetch (default: 50MB)
fetch.max.bytes=52428800

# High Throughput: Larger fetches
fetch.max.bytes=104857600  # 100MB

# Maximum time to wait for fetch.min.bytes (default: 500ms)
fetch.max.wait.ms=500

# High Throughput: Wait longer for larger batches
fetch.max.wait.ms=1000  # 1 second

# Maximum bytes per partition (default: 1MB)
max.partition.fetch.bytes=1048576

# High Throughput: Larger per-partition fetches
max.partition.fetch.bytes=5242880  # 5MB
```

**Optimization Guidelines:**

```
Fetch Size vs. Throughput:

Small Fetches (1MB):
- Lower latency
- More frequent requests
- Higher network overhead
- Lower throughput
- Use for: low-latency requirements

Large Fetches (10MB-100MB):
- Higher latency
- Fewer requests
- Lower network overhead
- Higher throughput
- Use for: high-throughput requirements

Recommended Tuning:
1. Start with fetch.min.bytes=1, fetch.max.wait.ms=500
2. Monitor fetch-latency-avg metric
3. Increase fetch.min.bytes if latency is acceptable
4. Increase fetch.max.wait.ms to allow larger batches
5. Balance between latency and throughput
```

**Trade-off:**
- **Larger fetches = higher throughput, higher latency**
- **Smaller fetches = lower latency, lower throughput**

#### Prefetching

Prefetching allows consumers to fetch data ahead of processing, ensuring data is available when needed.

**Prefetch Strategy:**

```
Prefetching Flow:

Consumer Polls:
        │
        ▼
1. Consumer calls poll(timeout)
        │
        ▼
2. Fetcher sends fetch requests to brokers
        │
        ▼
3. Brokers return messages
        │
        ▼
4. Messages stored in consumer buffer
        │
        ▼
5. Application processes messages from buffer
        │
        ▼
6. Fetcher continues prefetching in background
```

**Configuration:**

```properties
# Max poll records (maximum records per poll)
max.poll.records=500  # Default

# High Throughput: More records per poll
max.poll.records=1000

# Low Memory: Fewer records per poll
max.poll.records=100

# Max poll interval (maximum time between polls)
max.poll.interval.ms=300000  # 5 minutes default
```

**Prefetching Guidelines:**

```
When to Increase Prefetching:
- Fast processing (consumer can keep up)
- High throughput requirements
- Sufficient memory for buffering
- Low latency not critical

When to Decrease Prefetching:
- Slow processing (consumer cannot keep up)
- Memory-constrained environments
- Low latency critical
- Small message sizes

Configuration Examples:

High Throughput Consumer:
max.poll.records=1000
max.partition.fetch.bytes=5242880  # 5MB
fetch.min.bytes=1024
fetch.max.wait.ms=1000

Low Latency Consumer:
max.poll.records=100
max.partition.fetch.bytes=1048576  # 1MB
fetch.min.bytes=1
fetch.max.wait.ms=100

Memory-Constrained Consumer:
max.poll.records=50
max.partition.fetch.bytes=524288  # 512KB
fetch.min.bytes=1
fetch.max.wait.ms=50
```

**Trade-off:**
- **More prefetching = better throughput, higher memory usage**
- **Less prefetching = lower memory usage, potential for consumer starvation**

### Broker Performance

#### File System

The file system choice and configuration significantly impact broker performance.

**File System Choice:**

```
File System Comparison:

XFS (Recommended):
- Best performance for large files
- Good for sequential I/O (Kafka's workload)
- Efficient allocation
- Recommended for production

EXT4 (Good Alternative):
- Good performance
- Widely available
- Stable and mature
- Acceptable for Kafka

ZFS/Btrfs (Not Recommended):
- Copy-on-Write (CoW) adds overhead
- Higher CPU usage
- Slower performance for Kafka
- Avoid for production
```

**Mount Options:**

```bash
# Recommended mount options for Kafka
noatime          # Don't update access time (reduces writes)
nodiratime       # Don't update directory access time
data=writeback  # Delayed writes (better performance)
barrier=0        # Disable barriers (better performance, slight risk)
nobh             # No block heap allocation
```

**Mount Command Example:**

```bash
# Mount XFS with optimal options
mount -t xfs -o noatime,nodiratime,data=writeback,barrier=0,nobh /dev/sdb1 /mnt/kafka
```

#### Disk I/O

Disk I/O is often the bottleneck for Kafka brokers. Proper disk configuration is critical.

**Disk Configuration:**

```
Disk Configuration Best Practices:

1. Separate Disks for Log Directories
   - Use dedicated disks for each log directory
   - Reduces I/O contention
   - Improves throughput
   - Example: log.dirs=/mnt/disk1,/mnt/disk2,/mnt/disk3

2. Use SSD for Better Performance
   - SSDs provide much better I/O than HDDs
   - Lower latency for reads and writes
   - Better throughput
   - Recommended for production

3. RAID Configuration for Durability
   - RAID 10: Best performance and redundancy
   - RAID 5: Good redundancy, slower writes
   - RAID 0: Best performance, no redundancy (not recommended)
   - Choose based on requirements

4. Monitor Disk I/O Metrics
   - iostat -x 1: Monitor I/O statistics
   - Monitor disk latency
   - Monitor disk utilization
   - Alert on high disk usage
```

**Configuration:**

```properties
# Multiple log directories (comma-separated)
log.dirs=/mnt/kafka-1,/mnt/kafka-2,/mnt/kafka-3

# Log segment size (affects disk I/O pattern)
segment.bytes=1073741824  # 1GB (default)
# Larger segments = fewer files, less metadata overhead

# Log flush settings (rely on OS for performance)
log.flush.interval.messages=9223372036854775807  # Never flush based on messages
log.flush.interval.ms=9223372036854775807  # Never flush based on time
```

**Disk I/O Monitoring:**

```bash
# Monitor disk I/O
iostat -x 1

# Monitor disk usage
df -h /mnt/kafka-*

# Monitor disk latency
iostat -x -d 1 | grep -E "Device|kafka"
```

#### Network

Network configuration affects broker-to-broker and client-to-broker communication.

**Network Tuning:**

```properties
# Socket send buffer size
socket.send.buffer.bytes=102400  # 100KB

# Socket receive buffer size
socket.receive.buffer.bytes=102400  # 100KB

# Maximum request size
socket.request.max.bytes=104857600  # 100MB

# Number of network threads
num.network.threads=3  # Default: 3
# Increase for high network load
```

**OS Tuning:**

```bash
# Increase TCP buffer sizes
net.core.rmem_max=16777216  # 16MB
net.core.wmem_max=16777216  # 16MB
net.ipv4.tcp_rmem=4096 87380 16777216  # Min, default, max
net.ipv4.tcp_wmem=4096 65536 16777216  # Min, default, max

# Enable TCP window scaling
net.ipv4.tcp_window_scaling=1

# Increase TCP backlog
net.core.somaxconn=1024
net.ipv4.tcp_max_syn_backlog=1024

# Enable TCP fast open
net.ipv4.tcp_fastopen=3

# Reduce TCP keepalive time
net.ipv4.tcp_keepalive_time=600
net.ipv4.tcp_keepalive_intvl=30
net.ipv4.tcp_keepalive_probes=3
```

**Apply OS Tuning:**

```bash
# Add to /etc/sysctl.conf
echo "net.core.rmem_max=16777216" >> /etc/sysctl.conf
echo "net.core.wmem_max=16777216" >> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem=4096 87380 16777216" >> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem=4096 65536 16777216" >> /etc/sysctl.conf
echo "net.ipv4.tcp_window_scaling=1" >> /etc/sysctl.conf

# Apply changes
sysctl -p
```

#### JVM Tuning

Kafka runs on the JVM, so proper JVM tuning is essential for broker performance.

**Heap Configuration:**

```bash
# Kafka broker heap (4-6GB is typically sufficient)
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
```

**Heap Guidelines:**

```
Heap Sizing:

Why Small Heap?
- Kafka relies on OS page cache for performance
- Larger heap doesn't improve performance
- Can cause longer GC pauses
- Recommended: 4-6GB heap

When to Increase Heap?
- Many partitions (each partition uses memory)
- Large message sizes
- High throughput with many connections
- Monitor GC logs to determine

When to Decrease Heap?
- Limited physical memory
- Need more page cache
- Few partitions and connections
- Small message sizes
```

**GC Configuration:**

```bash
# G1GC recommended for Kafka
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+ExplicitGCInvokesConcurrent \
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -Xloggc:/var/log/kafka/gc.log"
```

**GC Tuning Parameters:**

- **-XX:+UseG1GC**: Use G1 garbage collector (recommended for heaps > 4GB)
- **-XX:MaxGCPauseMillis=20**: Target max GC pause time (20ms)
- **-XX:InitiatingHeapOccupancyPercent=35**: Start concurrent GC at 35% heap
- **-XX:+ExplicitGCInvokesConcurrent**: Explicit GC runs concurrently
- **-XX:+PrintGCDetails**: Print detailed GC information
- **-XX:+PrintGCDateStamps**: Print GC timestamps
- **-Xloggc**: GC log file location

**GC Monitoring:**

```bash
# Monitor GC logs
tail -f /var/log/kafka/gc.log

# Check for long GC pauses
grep "GC pause" /var/log/kafka/gc.log | tail -20

# Check heap usage
jstat -gc <pid> 1000
```

**GC Performance Indicators:**

- **GC Pause Time**: Should be < 100ms for most pauses
- **GC Frequency**: Should not be too frequent
- **Heap Usage**: Should not consistently be near 100%
- **Full GCs**: Should be rare (G1GC avoids full GCs)

### Monitoring Metrics

Monitoring is essential for understanding Kafka performance and identifying issues. This section covers key metrics for producers, consumers, and brokers.

#### Producer Metrics

Producer metrics help understand producer performance and identify bottlenecks.

**Key Metrics:**

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| `record-send-rate` | Records sent per second | Monitor for throughput |
| `record-retry-rate` | Records retried per second | > 100/sec indicates issues |
| `record-error-rate` | Records with errors per second | > 0 indicates errors |
| `request-latency-avg` | Average request latency | > 100ms indicates slow network |
| `request-latency-max` | Maximum request latency | > 1000ms indicates spikes |
| `batch-size-avg` | Average batch size | Monitor for batching efficiency |
| `compression-rate` | Compression ratio | Monitor for compression efficiency |
| `buffer-available-bytes` | Available buffer memory | < 1MB indicates buffer full |
| `io-wait-time-ns-avg` | Average I/O wait time | High values indicate I/O bottleneck |

**Monitoring Example:**

```bash
# Using JMX to monitor producer metrics
jconsole <producer-pid>

# Key producer metrics to monitor
kafka.producer:type=producer-metrics,client.id=<client-id>
  - record-send-rate
  - record-retry-rate
  - record-error-rate
  - request-latency-avg
  - batch-size-avg
```

#### Consumer Metrics

Consumer metrics help understand consumer performance and identify lag.

**Key Metrics:**

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| `records-consumed-rate` | Records consumed per second | Monitor for throughput |
| `records-lag-max` | Maximum lag across partitions | > 10000 indicates consumer falling behind |
| `records-lag-avg` | Average lag across partitions | Monitor for consumption rate |
| `fetch-rate` | Fetch requests per second | Monitor for fetch efficiency |
| `fetch-latency-avg` | Average fetch latency | > 100ms indicates slow broker |
| `commit-rate` | Offset commits per second | Monitor for commit efficiency |
| `heartbeat-rate` | Heartbeats sent per second | Should be consistent |
| `last-heartbeat-seconds-ago` | Time since last heartbeat | > session.timeout.ms/2 indicates issues |

**Consumer Lag Monitoring:**

```
Consumer Lag Example:

Topic: orders (6 partitions)
Consumer Group: order-processors

Partition 0: Lag = 1000 (consumer is 1000 messages behind)
Partition 1: Lag = 0 (consumer is up to date)
Partition 2: Lag = 5000 (consumer is 5000 messages behind)
Partition 3: Lag = 0 (consumer is up to date)
Partition 4: Lag = 2000 (consumer is 2000 messages behind)
Partition 5: Lag = 0 (consumer is up to date)

Max Lag: 5000 (Partition 2)
Avg Lag: 1333

Action: Add more consumers or optimize consumer processing
```

**Monitoring Example:**

```bash
# Using Kafka Consumer Command Line Tool
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-consumer-group

# Key consumer metrics to monitor
kafka.consumer:type=consumer-fetch-manager-metrics,client.id=<client-id>
  - records-consumed-rate
  - records-lag-max
  - fetch-latency-avg
```

#### Broker Metrics

Broker metrics help understand broker performance and identify system-level issues.

**Key Metrics:**

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| `messages-in-per-sec` | Messages received per second | Monitor for load |
| `bytes-in-per-sec` | Bytes received per second | Monitor for network load |
| `bytes-out-per-sec` | Bytes sent per second | Monitor for network load |
| `request-latency-avg` | Average request latency | > 100ms indicates slow broker |
| `request-latency-max` | Maximum request latency | > 1000ms indicates spikes |
| `log-flush-time-ms` | Log flush time | High values indicate slow disk |
| `under-replicated-partitions` | Count of under-replicated partitions | > 0 indicates replication issues |
| `offline-partitions-count` | Count of offline partitions | > 0 indicates broker failures |
| `active-controller-count` | Count of active controllers | Should be 1 |
| `isr-shrinks-per-sec` | ISR shrinks per second | > 0 indicates replica issues |
| `isr-expands-per-sec` | ISR expands per second | Should be low |
| `cpu-percent` | CPU usage | > 80% indicates CPU bottleneck |
| `memory-percent` | Memory usage | > 90% indicates memory pressure |

**Critical Metrics to Monitor:**

```
Critical Broker Metrics:

1. Under-Replicated Partitions
   - Indicates replication issues
   - Can lead to data loss
   - Alert if > 0

2. Offline Partitions
   - Indicates broker failures
   - Partitions not available
   - Alert if > 0

3. Request Latency
   - Indicates broker performance
   - High latency = slow broker
   - Alert if > 100ms

4. Disk I/O
   - log-flush-time-ms
   - High values = slow disk
   - Alert if > 1000ms

5. CPU Usage
   - High CPU = overloaded broker
   - Alert if > 80%
```

**Monitoring Example:**

```bash
# Using JMX to monitor broker metrics
jconsole <broker-pid>

# Key broker metrics to monitor
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
kafka.server:type=KafkaServer,name=ControllerState
```

**Monitoring Tools:**

1. **JConsole**: Built-in Java monitoring tool
2. **JMX Exporter**: Expose JMX metrics to Prometheus
3. **Kafka Manager**: Web-based Kafka management tool
4. **Burrow**: Kafka consumer lag monitoring
5. **Prometheus + Grafana**: Metrics collection and visualization

---

## Retention Policies

Retention policies determine how long messages are stored in Kafka before being deleted. This section explains the different retention mechanisms and how to configure them.

### Retention Mechanisms

Kafka provides multiple retention mechanisms to control how long messages are stored.

#### Time-Based Retention

Time-based retention deletes messages older than a specified time period.

**Configuration:**

```properties
# Retention in milliseconds (default: 7 days)
retention.ms=604800000  # 7 days

# Common retention periods
retention.ms=86400000   # 1 day
retention.ms=604800000  # 7 days (default)
retention.ms=2592000000 # 30 days
retention.ms=-1         # Infinite retention (never delete)
```

**Behavior:**

```
Time-Based Retention Flow:

Topic: logs
Retention: 7 days

Day 1: Messages stored
Day 2: Messages stored
Day 3: Messages stored
...
Day 7: Messages stored
Day 8: Messages from Day 1 deleted
Day 9: Messages from Day 2 deleted
...
Day 14: Messages from Day 7 deleted
```

**How It Works:**

1. **Messages are stored in log segments**
   - Each segment contains messages for a time range
   - Segments are closed when they reach size/time limit

2. **Retention checked during cleanup**
   - Cleanup runs periodically (default: every 5 minutes)
   - Checks segment timestamps
   - Deletes segments older than retention period

3. **Active segment never deleted**
   - The currently open segment is never deleted
   - Ensures writes can continue
   - Oldest closed segments deleted first

**Use Cases:**

- **Log Aggregation**: Retain logs for 7-30 days for analysis
- **Event Streaming**: Retain events for replay and debugging
- **Audit Trails**: Retain audit logs indefinitely (retention.ms=-1)
- **Temporary Data**: Short retention for ephemeral data (1 day)

#### Size-Based Retention

Size-based retention deletes the oldest messages when the topic size exceeds a specified limit.

**Configuration:**

```properties
# Retention in bytes (per partition)
retention.bytes=1073741824  # 1GB
retention.bytes=10737418240 # 10GB
retention.bytes=-1          # Unlimited (default)
```

**Behavior:**

```
Size-Based Retention Flow:

Topic: events
Retention: 1GB per partition
Partition 0: 500MB
Partition 1: 500MB
Partition 2: 500MB
Total: 1.5GB

Partition 0 grows to 1.2GB:
- Oldest 200MB deleted
- Partition 0 size: 1GB
```

**How It Works:**

1. **Per-partition limit**
   - Each partition has its own size limit
   - Total topic size = partition count × retention.bytes
   - Oldest segments deleted when limit exceeded

2. **Combined with time-based retention**
   - Both policies can be active
   - Whichever limit reached first triggers deletion
   - Provides flexible retention control

**Use Cases:**

- **Storage-Constrained Environments**: Limit disk usage
- **Cost Optimization**: Control storage costs
- **Data Lifecycle**: Automatic cleanup based on storage
- **Multi-Tenant**: Fair resource allocation

**Combined Retention Example:**

```properties
# Delete after 7 days OR when size exceeds 1GB
retention.ms=604800000      # 7 days
retention.bytes=1073741824  # 1GB
```

### Cleanup Policies

Cleanup policies determine how old data is removed from Kafka.

#### Delete Policy

Delete policy is the default cleanup mechanism, removing old segments based on retention settings.

**Configuration:**

```properties
# Default cleanup policy
cleanup.policy=delete
```

**Behavior:**

```
Delete Policy Flow:

1. Log segment becomes eligible for deletion
   (based on retention.ms or retention.bytes)
        │
        ▼
2. Cleanup thread identifies eligible segments
        │
        ▼
3. Segment files are deleted from disk
        │
        ▼
4. Metadata updated
        │
        ▼
5. Space reclaimed
```

**Characteristics:**

- **Default Policy**: Used by most topics
- **Efficient for Log Data**: Optimized for append-only workloads
- **No Message Modification**: Messages never changed, only deleted
- **Simple to Understand**: Predictable behavior
- **Good for Streaming**: Event streaming, log aggregation

**When to Use:**

- Event streaming
- Log aggregation
- Metrics collection
- Any append-only workload

#### Compact Policy

Compact policy keeps only the latest message for each key, enabling change log semantics.

**Configuration:**

```properties
# Enable compaction
cleanup.policy=compact
```

**Behavior:**

```
Compaction Example:

Before Compaction:
Key: user-123, Value: {"name": "John", "status": "active"} (offset 0)
Key: user-456, Value: {"name": "Jane", "status": "active"} (offset 1)
Key: user-123, Value: {"name": "John", "status": "inactive"} (offset 2)
Key: user-456, Value: {"name": "Jane", "status": "inactive"} (offset 3)
Key: user-789, Value: {"name": "Bob", "status": "active"} (offset 4)

After Compaction:
Key: user-123, Value: {"name": "John", "status": "inactive"} (offset 2) - latest
Key: user-456, Value: {"name": "Jane", "status": "inactive"} (offset 3) - latest
Key: user-789, Value: {"name": "Bob", "status": "active"} (offset 4) - latest

Old messages with same key deleted
```

**Compaction Process:**

```
Compaction Flow:

1. Cleaner thread identifies compactable segments
        │
        ▼
2. Segments are copied with only latest values per key
        │
        ▼
3. Old segments are deleted
        │
        ▼
4. Compaction index updated
        │
        ▼
5. Runs in background (non-blocking)
```

**Configuration:**

```properties
# Enable log cleaner
log.cleaner.enable=true

# Number of cleaner threads
log.cleaner.threads=1  # Default: 1

# Deduplication buffer size
log.cleaner.dedupe.buffer.size=134217728  # 128MB

# I/O buffer size for cleaning
log.cleaner.io.buffer.size=524288  # 512KB

# Maximum I/O bytes per second (unlimited by default)
log.cleaner.io.max.bytes.per.second=9223372036854775807
```

**Compaction Configuration:**

```properties
# Minimum compaction lag (messages not compacted until this time)
min.compaction.lag.ms=0  # Default: 0 (immediate)

# Maximum compaction lag (force compaction if not done by this time)
max.compaction.lag.ms=9223372036854775807  # Default: infinite

# Delete retention for compacted topics (tombstone retention)
delete.retention.ms=86400000  # 24 hours
```

**Tombstones:**

Compaction uses tombstones (null values) to mark keys for deletion.

```
Tombstone Example:

1. Key: user-123, Value: {"name": "John"} (offset 0)
2. Key: user-123, Value: null (offset 1) - TOMBSTONE
3. Key: user-123, Value: {"name": "John"} (offset 2)

After Compaction:
Key: user-123, Value: {"name": "John"} (offset 2) - latest
Tombstone (offset 1) deleted after delete.retention.ms
```

**When to Use:**

- Database change data capture (CDC)
- State updates
- Configuration changes
- Any table-like data where latest value matters

#### Compact + Delete Policy

Combines compaction with time-based deletion for compacted topics.

**Configuration:**

```properties
# Enable both compaction and deletion
cleanup.policy=compact,delete
```

**Behavior:**

- Compaction keeps latest value per key
- Delete removes old compacted segments based on time
- Useful for compacted topics with time-based cleanup

**Use Case:**

- Compact topic with periodic cleanup
- Remove old tombstones
- Clean up deleted keys after time period

### Log Segments

Log segments are the fundamental storage unit in Kafka. Understanding segment management is important for retention.

#### Segment Management

**Segment Configuration:**

```properties
# Segment size in bytes (default: 1GB)
segment.bytes=1073741824  # 1GB

# Segment time in milliseconds (default: 7 days)
segment.ms=604800000  # 7 days

# Segment index size
segment.index.bytes=10485760  # 10MB
```

**Segment Rolling:**

```
Segment Rolling Flow:

Active Segment grows:
        │
        ▼
Reaches segment.bytes OR segment.ms
        │
        ▼
Active segment closed
        │
        ▼
New active segment created
        │
        ▼
Old segment becomes eligible for deletion
```

**Segment Rolling Triggers:**

1. **Size-based rolling**
   - When segment reaches segment.bytes
   - Default: 1GB
   - Good for most workloads

2. **Time-based rolling**
   - When segment reaches segment.ms
   - Default: 7 days
   - Ensures segments roll even with low volume

**Why Segment Rolling Matters:**

- **Retention operates on segments**: Old segments deleted, not individual messages
- **Compaction operates on segments**: Cleaner works on segment files
- **Index size**: Each segment has an index, smaller segments = more index overhead
- **Recovery time**: Larger segments = longer recovery on broker restart

**Segment Configuration Guidelines:**

```
Segment Sizing:

Large Segments (1GB):
- Fewer segments
- Less index overhead
- Longer recovery time
- Better for high-throughput topics

Small Segments (100MB):
- More segments
- More index overhead
- Faster recovery time
- Better for low-latency recovery

Recommended:
- Default 1GB for most topics
- Reduce to 100-500MB for low-volume topics
- Increase to 2-5GB for high-volume topics
```
- Segment naming: base offset

#### Segment Index

**Index Structure:**
- Memory-mapped index file
- Maps offset to physical position
- Sparse index (not every message indexed)
- Enables fast message lookup

**Index Configuration:**
```properties
index.interval.bytes=4096
log.index.size.max.bytes=10485760  # 10MB
```

### Log Deletion

#### Deletion Process

**Deletion Triggers:**
- Retention time exceeded
- Retention size exceeded
- Log size exceeds limit

**Deletion Steps:**
1. Log cleaner identifies deletable segments
2. Segments marked for deletion
3. File descriptors closed
4. Files deleted from disk
5. Metadata updated

**Configuration:**
```properties
log.retention.check.interval.ms=300000  # 5 minutes
```

---

## Internal Workings

This section explains the internal workings of Kafka, including message storage, offset management, replication protocol, and controller internals.

### Message Storage

#### Log Segment Structure

Kafka stores messages in log segments, which are the fundamental storage unit.

```
┌─────────────────────────────────────────────────────────────┐
│                      Log Segment                             │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  .log file   │  │  .index file  │  │ .timeindex   │      │
│  │              │  │              │  │              │      │
│  │  [Message 1] │  │  [Offset 0]   │  │  [Timestamp] │      │
│  │  [Message 2] │  │  [Offset 1]   │  │  [Timestamp] │      │
│  │  [Message 3] │  │  [Offset 2]   │  │  [Timestamp] │      │
│  │  ...         │  │  ...         │  │  ...         │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**File Types:**

1. **.log file**: Contains the actual message data
2. **.index file**: Maps offsets to physical positions in the .log file
3. **.timeindex file**: Maps timestamps to offsets

**Segment Naming:**

Segments are named based on their base offset:

```
Example:
00000000000000000000.log  # Base offset: 0
00000000000000000000.index
00000000000000000000.timeindex

00000000000000001000.log  # Base offset: 1000
00000000000000001000.index
00000000000000001000.timeindex
```

#### Message Format

**Message Structure:**

```
┌─────────────────────────────────────────────────────────────┐
│                      Message                                │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  CRC32   │  │  Magic   │  │ Attributes│ │  Key     │    │
│  │ Checksum │  │  Byte    │  │          │  │  (bytes) │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
│  ┌──────────┐  ┌──────────┐                              │
│  │  Value   │  │  Headers │                              │
│  │ (bytes)  │  │          │                              │
│  └──────────┘  └──────────┘                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Message Fields:**

- **CRC32 Checksum**: Message integrity validation
- **Magic Byte**: Message format version
- **Attributes**: Compression type, timestamp type
- **Key**: Message key (optional)
- **Value**: Message payload
- **Headers**: Key-value pairs for metadata

**Message Attributes:**

- **Compression Type**: none, gzip, snappy, lz4, zstd
- **Timestamp Type**: CreateTime or LogAppendTime
- **Timestamp Value**: Message timestamp
- **Transactional ID**: If message is part of a transaction
- **Isolation Level**: Read committed or read uncommitted

### Offset Management

#### Offset Storage

Consumer offsets are stored in an internal Kafka topic called `__consumer_offsets`.

**Consumer Offsets Topic:**

```
__consumer_offsets Topic Structure:

Topic: __consumer_offsets
Partitions: 50 (default)
Replication Factor: 3 (default)
Cleanup Policy: Compact (keeps latest offset per group)
```

**Offset Commit Format:**

```
Key: [group.id, topic, partition]
Value: [offset, metadata, commit_timestamp]
```

**Offset Storage Example:**

```
Consumer Group: order-processors
Topic: orders
Partition: 0

Key: "order-processors-orders-0"
Value: {offset: 12345, metadata: "", commit_timestamp: 1640995200000}
```

#### Offset Types

**Committed Offset:**

- Last offset successfully processed by consumer
- Stored in `__consumer_offsets` topic
- Used for recovery after consumer failure
- Updated when consumer commits

**Fetched Offset:**

- Current position in partition (next offset to fetch)
- Maintained in consumer memory
- Incremented as messages are consumed
- Not persisted until commit

**Log End Offset (LEO):**

- High watermark for partition
- Last offset in log (next offset to be written)
- Used to determine consumer lag
- Incremented when new messages are written

**High Watermark (HW):**

- Last offset fully replicated to all ISR
- Consumers can only read up to HW
- Ensures consistency across replicas
- Updated when all ISR replicas acknowledge

**Offset Relationship Diagram:**

```
Partition: orders-0

Messages: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
LEO: 10 (next offset to write)
HW: 8 (last offset replicated to all ISR)
Committed Offset: 5 (last offset processed by consumer)
Fetched Offset: 6 (next offset to fetch)

Consumer Lag: LEO - Committed Offset = 10 - 5 = 5 messages
```

### Replication Protocol

Replication ensures data durability by copying partitions across multiple brokers.

#### Fetcher Thread

Each follower replica has a fetcher thread that continuously fetches messages from the leader.

**Follower Fetch Process:**

```
Follower Fetch Flow:

1. Follower sends fetch request to leader
        │
        ▼
2. Leader responds with messages
        │
        ▼
3. Follower appends messages to log
        │
        ▼
4. Follower updates LEO
        │
        ▼
5. Follower sends fetch request with updated fetch offset
        │
        ▼
6. Process repeats continuously
```

**Fetch Configuration:**

```properties
# Maximum bytes to fetch in a single request
replica.fetch.max.bytes=1048576  # 1MB

# Maximum time to wait for fetch.min.bytes
replica.fetch.wait.max.ms=500  # 500ms

# Minimum bytes to wait for
replica.fetch.min.bytes=1  # 1 byte
```

#### High Watermark Propagation

The high watermark (HW) is propagated from leader to followers to ensure consistency.

**HW Update Process:**

```
HW Update Flow:

1. Leader appends message to log
        │
        ▼
2. Leader updates LEO
        │
        ▼
3. Followers fetch and append message
        │
        ▼
4. Followers send acknowledgment to leader
        │
        ▼
5. Leader checks if all ISR have acknowledged
        │
        ▼
6. Leader updates HW
        │
        ▼
7. HW propagated to followers in next fetch response
        │
        ▼
8. Followers update HW
```

**HW Example:**

```
Initial State:
Leader: LEO=10, HW=8
Follower 1: LEO=10, HW=8
Follower 2: LEO=9, HW=8
ISR: [Leader, Follower 1, Follower 2]

Leader writes message at offset 10:
Leader: LEO=11, HW=8 (HW not updated yet)

Follower 1 fetches and appends:
Follower 1: LEO=11, HW=8
Follower 1 acknowledges to leader

Follower 2 fetches and appends:
Follower 2: LEO=10, HW=8
Follower 2 acknowledges to leader

Leader updates HW:
Leader: LEO=11, HW=10 (all ISR have offset 10)

HW propagated to followers:
Follower 1: LEO=11, HW=10
Follower 2: LEO=10, HW=10
```

### Controller Internals

The controller is a special broker responsible for cluster management operations.

#### Controller State Machine

**Controller States:**

```
Controller State Machine:

┌─────────┐     Broker Election     ┌─────────┐
│Inactive│ ────────────────────────► │ Active  │
└─────────┘                         └─────────┘
     ▲                                     │
     │         New Controller Elected      │
     └─────────────────────────────────────┘
              (Failover)

Active: Controlling the cluster
Inactive: Not the controller
Fenced: Old controller after failover (cannot make changes)
```

**Controller Operations:**

- **Partition Leadership Changes**: Elect new leaders on broker failures
- **Topic Creation/Deletion**: Create or delete topics
- **Reassignment Operations**: Reassign partitions between brokers
- **Preferred Replica Election**: Elect preferred replica as leader
- **ISR Management**: Add/remove replicas from ISR
- **Broker Management**: Handle broker startup/shutdown

#### Controller Events

**Event Types:**

```
Controller Event Flow:

Broker Startup:
        │
        ▼
1. Broker registers with Zookeeper
        │
        ▼
2. Controller detects new broker
        │
        ▼
3. Controller updates cluster metadata
        │
        ▼
4. Partitions assigned to new broker if needed

Broker Shutdown:
        │
        ▼
1. Broker unregisters from Zookeeper
        │
        ▼
2. Controller detects broker failure
        │
        ▼
3. Controller initiates leader election
        │
        ▼
4. Partitions reassigned to other brokers

Controller Change:
        │
        ▼
1. Old controller fails
        │
        ▼
2. New controller elected
        │
        ▼
3. Controller epoch incremented
        │
        ▼
4. New controller rebuilds state from Zookeeper
```

---

## Producer Internals

This section explains the internal architecture and components of the Kafka producer.

### Producer Architecture

The Kafka producer is designed for high throughput and low latency through asynchronous, batched message sending.

```
┌─────────────────────────────────────────────────────────────┐
│                       Producer                              │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Serializer │  │ Partitioner  │  │   Record    │      │
│  │              │  │              │  │  Accumulator │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Sender     │  │  Network     │  │   Metadata  │      │
│  │   Thread     │  │   Client     │  │   Service   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Component Responsibilities:**

1. **Serializer**: Converts objects to byte arrays
2. **Partitioner**: Determines which partition to send each message
3. **Record Accumulator**: Batches messages by partition in memory
4. **Sender Thread**: Background thread that sends batches to brokers
5. **Network Client**: Manages TCP connections and I/O
6. **Metadata Service**: Maintains cluster metadata (brokers, topics, partitions)

### Record Accumulator

The record accumulator batches messages by partition to improve throughput.

**Accumulator Function:**

```
Record Accumulator Flow:

Application sends message
        │
        ▼
Serializer converts to bytes
        │
        ▼
Partitioner selects partition
        │
        ▼
Message added to partition's batch
        │
        ▼
Batch full OR linger time expired?
        │
        Yes → Sender thread sends batch
        │
        No → Wait for more messages
```

**Accumulator Configuration:**

```properties
# Total buffer memory for all batches
buffer.memory=33554432  # 32MB

# Maximum batch size per partition
batch.size=16384  # 16KB

# Time to wait for batch to fill
linger.ms=0  # 0ms (send immediately)
linger.ms=50  # 50ms (wait up to 50ms for batch to fill)
```

**Accumulator Behavior:**

- **Batches by Partition**: Each partition has its own batch
- **Memory Management**: Blocks when buffer is full
- **Backpressure**: Slows down producer when brokers are slow
- **Batch Size Control**: Ensures batches don't exceed max size

### Partitioning Strategies

Partitioning determines which partition each message is sent to.

#### Default Partitioner

**Partitioning Logic:**

```
Default Partitioner Algorithm:

1. If key is null:
   - Round-robin (sticky) across partitions
   - Batches messages to same partition before switching
   - Improves throughput

2. If key is present:
   - hash(key) % num_partitions
   - Uses murmur2 hash function
   - Ensures same key goes to same partition
```

**Example:**

```
Topic: orders (3 partitions)
Messages with keys:

Key: "user-123" → hash("user-123") % 3 = Partition 1
Key: "user-456" → hash("user-456") % 3 = Partition 2
Key: "user-123" → hash("user-123") % 3 = Partition 1 (same partition)
Key: "user-789" → hash("user-789") % 3 = Partition 0
```

**Custom Partitioner:**

```java
public class CustomPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        // Custom partitioning logic
        // Example: Partition by region
        String region = extractRegion(value);
        switch (region) {
            case "us-east": return 0;
            case "us-west": return 1;
            case "eu-west": return 2;
            default: return 0;
        }
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

**Configuration:**

```properties
# Use custom partitioner
partitioner.class=com.example.CustomPartitioner
```

### Metadata Management

The metadata service maintains cluster metadata for partition selection and request routing.

**Metadata Refresh:**

```
Metadata Refresh Flow:

1. Producer requests metadata
        │
        ▼
2. Metadata service checks if metadata is stale
        │
        ▼
3. If stale, requests metadata from broker
        │
        ▼
4. Broker returns cluster metadata
        │
        ▼
5. Metadata service updates cache
        │
        ▼
6. Producer uses metadata for partitioning
```

**Metadata Contents:**

- Broker list (broker IDs, hostnames, ports)
- Topic list
- Partition information (partition count, replica assignments)
- Leader information (which broker is leader for each partition)

**Configuration:**

```properties
# Maximum age of metadata before refresh
metadata.max.age.ms=300000  # 5 minutes

# Backoff between metadata refresh attempts
metadata.refresh.backoff.ms=500  # 500ms
```

### Network Client

The network client manages TCP connections and I/O operations.

**Request Sending:**

```
Network Client Flow:

1. Sender thread has batch to send
        │
        ▼
2. Network client checks connection to broker
        │
        ▼
3. If no connection, establishes TCP connection
        │
        ▼
4. Sends request over connection
        │
        ▼
5. Waits for response
        │
        ▼
6. Processes response
        │
        ▼
7. Invokes callback (success or error)
```

**Connection Configuration:**

```properties
# Maximum time to keep idle connections
connections.max.idle.ms=540000  # 9 minutes

# Request timeout
request.timeout.ms=30000  # 30 seconds

# Maximum in-flight requests per connection
max.in.flight.requests.per.connection=5
```

---

## Consumer Internals

This section explains the internal architecture and components of the Kafka consumer.

### Consumer Architecture

The Kafka consumer is designed for scalable, fault-tolerant message consumption.

```
┌─────────────────────────────────────────────────────────────┐
│                       Consumer                              │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Deserializer │  │  Fetcher     │  │   Consumer   │      │
│  │              │  │              │  │  Coordinator │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Heartbeat   │  │   Offset     │  │   Network    │      │
│  │   Thread     │  │  Manager     │  │   Client     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Component Responsibilities:**

1. **Deserializer**: Converts byte arrays to objects
2. **Fetcher**: Fetches messages from brokers
3. **Consumer Coordinator**: Manages consumer group membership and rebalancing
4. **Heartbeat Thread**: Sends heartbeats to coordinator
5. **Offset Manager**: Tracks and commits consumer offsets
6. **Network Client**: Manages TCP connections and I/O

### Fetcher Thread

The fetcher thread is responsible for fetching messages from brokers.

**Fetch Process:**

```
Fetch Process Flow:

1. Consumer calls poll()
        │
        ▼
2. Fetcher checks assigned partitions
        │
        ▼
3. Fetcher sends fetch requests to brokers
        │
        ▼
4. Brokers return messages
        │
        ▼
5. Fetcher deserializes messages
        │
        ▼
6. Messages returned to application
```

**Fetch Configuration:**

```properties
# Maximum bytes per partition in a single fetch
max.partition.fetch.bytes=1048576  # 1MB

# Maximum bytes in a single fetch
fetch.max.bytes=52428800  # 50MB

# Minimum bytes to wait for
fetch.min.bytes=1

# Maximum time to wait for fetch.min.bytes
fetch.max.wait.ms=500  # 500ms
```

### Consumer Coordinator

The consumer coordinator manages consumer group membership and rebalancing.

**Coordinator Responsibilities:**

- Join consumer group
- Handle rebalancing
- Commit offsets
- Track consumer heartbeats
- Detect consumer failures

**Coordinator Flow:**

```
Coordinator Flow:

1. Consumer joins group
        │
        ▼
2. Coordinator assigns partitions
        │
        ▼
3. Consumer starts consuming
        │
        ▼
4. Consumer sends heartbeats
        │
        ▼
5. Coordinator tracks consumer liveness
        │
        ▼
6. If consumer fails, coordinator triggers rebalance
```

**Configuration:**

```properties
# Session timeout (how long consumer can be inactive)
session.timeout.ms=10000  # 10 seconds

# Heartbeat interval
heartbeat.interval.ms=3000  # 3 seconds

# Max poll interval (maximum time between polls)
max.poll.interval.ms=300000  # 5 minutes
```

### Offset Management

The offset manager tracks and commits consumer offsets.

**Offset Commit Process:**

```
Offset Commit Flow:

1. Application commits offset
        │
        ▼
2. Offset manager sends commit request to coordinator
        │
        ▼
3. Coordinator writes offset to __consumer_offsets topic
        │
        ▼
4. Coordinator acknowledges commit
        │
        ▼
5. Offset manager notifies application
```

**Offset Storage:**

Offsets are stored in the internal `__consumer_offsets` topic:

```
__consumer_offsets Topic:
- Compacted topic (keeps latest offset per group)
- Replicated for durability
- Partitioned by consumer group ID
```

**Configuration:**

```properties
# Enable auto commit
enable.auto.commit=true

# Auto commit interval
auto.commit.interval.ms=5000  # 5 seconds

# Offset commit callback
enable.auto.commit=false  # Manual commit
```

---

## Security

Security in Kafka includes authentication, authorization, and encryption to protect your cluster and data.

### Authentication

Authentication verifies the identity of clients connecting to Kafka.

#### SASL Authentication

SASL (Simple Authentication and Security Layer) provides multiple authentication mechanisms.

**SASL Mechanisms:**

| Mechanism | Description | Use Case |
|-----------|-------------|----------|
| SASL/PLAIN | Username/password (requires SSL) | Simple authentication, SSL required |
| SASL/GSSAPI | Kerberos | Enterprise environments with Kerberos |
| SASL/SCRAM | Username/password with salted hashes | More secure than PLAIN |
| SASL/OAUTHBEARER | OAuth 2.0 | Token-based authentication |

**SASL/PLAIN Configuration:**

```properties
# Broker configuration
security.inter.broker.protocol=SASL_SSL
sasl.enabled.mechanisms=PLAIN
sasl.mechanism.inter.broker.protocol=PLAIN

# Producer/Consumer configuration
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="user" password="password";
```

**SASL/SCRAM Configuration:**

```properties
# Broker configuration
security.inter.broker.protocol=SASL_SSL
sasl.enabled.mechanisms=SCRAM-SHA-256,SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

# Producer/Consumer configuration
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="user" password="password";
```

**Creating SCRAM Users:**

```bash
# Create SCRAM user
kafka-configs.sh --zookeeper localhost:2181 \
  --alter --add-config 'SCRAM-SHA-256=[password=secret]' \
  --entity-type users --entity-name alice

# List SCRAM users
kafka-configs.sh --zookeeper localhost:2181 \
  --describe --entity-type users --entity-name alice
```

#### SSL/TLS Authentication

SSL/TLS provides encryption and authentication using certificates.

**SSL Configuration:**

```properties
# Broker configuration (two-way SSL)
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=password
ssl.key.password=password
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=password
ssl.client.auth=required  # Require client authentication

# Producer/Consumer configuration (two-way SSL)
security.protocol=SSL
ssl.keystore.location=/path/to/client.keystore.jks
ssl.keystore.password=password
ssl.key.password=password
ssl.truststore.location=/path/to/client.truststore.jks
ssl.truststore.password=password
```

**Generating SSL Certificates:**

```bash
# Generate CA key
openssl genrsa -out ca-key.pem 2048

# Generate CA certificate
openssl req -new -key ca-key.pem -x509 -days 365 -out ca-cert.pem

# Generate broker keystore
keytool -genkeypair -alias broker \
  -keyalg RSA -keysize 2048 \
  -keystore broker.keystore.jks \
  -validity 365 \
  -storepass password \
  -keypass password \
  -dname "CN=broker"

# Generate broker certificate signing request
keytool -certreq -alias broker \
  -keystore broker.keystore.jks \
  -file broker-cert-sign-request.csr \
  -storepass password

# Sign broker certificate
openssl x509 -req -CA ca-cert.pem -CAkey ca-key.pem \
  -in broker-cert-sign-request.csr \
  -out broker-cert-signed.pem \
  -days 365 -CAcreateserial

# Import CA certificate into broker keystore
keytool -import -alias ca-cert \
  -file ca-cert.pem \
  -keystore broker.keystore.jks \
  -storepass password

# Import signed certificate into broker keystore
keytool -import -alias broker \
  -file broker-cert-signed.pem \
  -keystore broker.keystore.jks \
  -storepass password

# Generate client keystore (similar process)
keytool -genkeypair -alias client \
  -keyalg RSA -keysize 2048 \
  -keystore client.keystore.jks \
  -validity 365 \
  -storepass password \
  -keypass password \
  -dname "CN=client"
```

### Authorization

Authorization controls what operations authenticated users can perform.

#### ACL Management

Access Control Lists (ACLs) define permissions for principals.

**ACL Operations:**

```bash
# Add ACL for user to read from topic
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:alice \
  --operation Read --topic test-topic

# Add ACL for user to write to topic
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:alice \
  --operation Write --topic test-topic

# Add ACL for user to describe topic
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:alice \
  --operation Describe --topic test-topic

# List ACLs for topic
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --list --topic test-topic

# List all ACLs
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --list

# Delete ACL
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --remove --allow-principal User:alice \
  --operation Read --topic test-topic
```

**ACL Configuration:**

```properties
# Enable ACL authorization
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
```

**ACL Operations:**

- **Read**: Consume from topic
- **Write**: Produce to topic
- **Describe**: Get metadata about topic
- **Delete**: Delete topic
- **Alter**: Modify topic configuration
- **Create**: Create topic

**Principal Types:**

- **User**: Specific user (e.g., User:alice)
- **Group**: User group (e.g., Group:developers)
- **Wildcard**: All principals (e.g., User:*)

**Resource Types:**

- **Topic**: Topic-level permissions
- **Group**: Consumer group permissions
- **Cluster**: Cluster-level permissions
- **TransactionalId**: Transactional ID permissions

**ACL Examples:**

```bash
# Allow user to read from all topics
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:alice \
  --operation Read --topic "*"

# Allow user to write to specific topic
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --allow-principal User:alice \
  --operation Write --topic "orders-*"

# Deny user from deleting topics
kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 \
  --add --deny-principal User:alice \
  --operation Delete --topic "*"
```

### Encryption

Encryption protects data in transit between clients and brokers, and between brokers.

#### SSL/TLS Encryption

SSL/TLS encrypts data in transit.

**Encryption Configuration:**

```properties
# Broker configuration
listeners=SSL://:9093
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=password
ssl.key.password=password
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=password

# Inter-broker encryption
security.inter.broker.protocol=SSL
```

**Producer/Consumer Configuration:**

```properties
# Enable SSL
security.protocol=SSL
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=password

# Disable hostname verification (not recommended for production)
ssl.endpoint.identification.algorithm=
```

---

## Monitoring

Monitoring is essential for understanding Kafka performance and identifying issues. This section covers key metrics and monitoring tools.

### Key Metrics

#### Producer Metrics

**Throughput Metrics:**

- `record-send-rate`: Records sent per second
- `byte-rate`: Bytes sent per second
- `record-queue-time-avg`: Average time records spend in the send buffer

**Latency Metrics:**

- `request-latency-avg`: Average request latency
- `request-latency-max`: Maximum request latency
- `record-send-rate`: Records sent per second

**Error Metrics:**

- `record-error-rate`: Records with errors per second
- `record-retry-rate`: Records retried per second

#### Consumer Metrics

**Throughput Metrics:**

- `records-consumed-rate`: Records consumed per second
- `bytes-consumed-rate`: Bytes consumed per second
- `fetch-rate`: Fetch requests per second

**Lag Metrics:**

- `records-lag-max`: Maximum lag across partitions
- `records-lag-avg`: Average lag across partitions

**Commit Metrics:**

- `commit-rate`: Offset commits per second
- `commit-latency-avg`: Average commit latency

#### Broker Metrics

**Throughput Metrics:**

- `messages-in-per-sec`: Messages received per second
- `bytes-in-per-sec`: Bytes received per second
- `bytes-out-per-sec`: Bytes sent per second

**Performance Metrics:**

- `request-latency-avg`: Average request latency
- `io-wait-time-ns-avg`: Average I/O wait time

**Health Metrics:**

- `under-replicated-partitions`: Count of under-replicated partitions
- `offline-partitions-count`: Count of offline partitions
- `active-controller-count`: Count of active controllers (should be 1)

### Monitoring Tools

#### JMX

Kafka exposes metrics via JMX (Java Management Extensions).

**Enable JMX:**

```bash
# Enable JMX for broker
export JMX_PORT=9999
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false"

# Start broker
bin/kafka-server-start.sh config/server.properties
```

**Connect with JConsole:**

```bash
jconsole localhost:9999
```

#### Prometheus + Grafana

Prometheus can scrape JMX metrics from Kafka.

**JMX Exporter Configuration:**

```yaml
# jmx_exporter_config.yml
rules:
  - pattern: 'kafka.producer<type=producer-metrics, client-id=(.+)><>(record-send-rate):'
    name: kafka_producer_record_send_rate
    labels:
      client_id: "$1"
```

**Prometheus Configuration:**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['localhost:9999']
```

**Grafana Dashboard:**

- Import Kafka dashboard from Grafana community
- Visualize producer, consumer, and broker metrics
- Set up alerts for critical metrics

#### Kafka Manager

Kafka Manager is a web-based tool for managing and monitoring Kafka clusters.

**Features:**

- Cluster management
- Topic management
- Consumer group monitoring
- Broker metrics
- ACL management

**Setup:**

```bash
# Download Kafka Manager
wget https://github.com/yahoo/kafka-manager/releases/download/v2.0.0.2/kafka-manager-2.0.0.2.zip

# Configure
cp conf/application.conf conf/application.conf.bak
# Edit application.conf with your cluster configuration

# Start
bin/kafka-manager
```

### Alerting

Set up alerts for critical metrics to proactively identify issues.

**Critical Alerts:**

1. **Under-Replicated Partitions > 0**
   - Indicates replication issues
   - Can lead to data loss

2. **Offline Partitions > 0**
   - Indicates broker failures
   - Partitions not available

3. **Consumer Lag > Threshold**
   - Consumer falling behind
   - May need more consumers

4. **Request Latency > Threshold**
   - Broker performance degradation
   - May need scaling

**Alert Configuration Example (Prometheus Alertmanager):**

```yaml
groups:
  - name: kafka_alerts
    rules:
      - alert: UnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 5m
        annotations:
          summary: "Kafka has under-replicated partitions"
          
      - alert: ConsumerLagHigh
        expr: kafka_consumer_lag > 10000
        for: 10m
        annotations:
          summary: "Consumer lag is high"
```

---

## Best Practices

This section provides best practices for designing, deploying, and operating Kafka clusters.

### Cluster Design

#### Hardware Sizing

**Broker Hardware Requirements:**

```
Production Broker Recommendations:

CPU:
- 4-8 cores for moderate load
- 8-16 cores for high load
- Consider hyperthreading

Memory:
- 4-6GB JVM heap (Kafka doesn't need large heap)
- Leave remaining memory for OS page cache
- 16-32GB total RAM for production

Disk:
- Use SSD for better performance
- Separate disks for log directories
- RAID 10 for durability (if using HDD)
- Monitor disk I/O latency

Network:
- 10GbE for high throughput
- Multiple network interfaces for redundancy
- Low latency network for broker-to-broker communication
```

**Cluster Sizing:**

```
Cluster Size Considerations:

Small Cluster (3 brokers):
- Minimum for replication
- Can survive 1 broker failure
- Good for development/testing

Medium Cluster (5-7 brokers):
- Better fault tolerance
- Can survive 2 broker failures
- Good for production

Large Cluster (10+ brokers):
- High fault tolerance
- Can survive multiple broker failures
- Good for high-throughput workloads
```

#### Topic Design

**Partition Count:**

```
Partition Count Guidelines:

Factors to Consider:
1. Throughput requirements
2. Number of consumers in group
3. Future growth
4. Broker count

Rule of Thumb:
- Start with 3-10 partitions per topic
- Each partition can handle ~10MB/s
- More partitions = more parallelism, more overhead

Example:
- Required throughput: 100MB/s
- Per-partition throughput: 10MB/s
- Partitions needed: 100/10 = 10 partitions
```

**Replication Factor:**

```
Replication Factor Guidelines:

Replication Factor 1:
- No fault tolerance
- Data loss on broker failure
- Use for: non-critical data, development

Replication Factor 2:
- Survives 1 broker failure
- Good for: general purpose

Replication Factor 3 (Recommended):
- Survives 2 broker failures
- Best for: production, critical data

Replication Factor > 3:
- Survives more broker failures
- Higher storage overhead
- Use for: extremely critical data
```

### Producer Best Practices

#### Configuration

**Recommended Producer Settings:**

```properties
# Enable idempotence for exactly-once semantics
enable.idempotence=true

# Use acks=all for durability
acks=all

# Ensure at least 2 replicas have data
min.insync.replicas=2

# Retry indefinitely for transient errors
retries=2147483647

# Use compression for text data
compression.type=lz4

# Batch messages for higher throughput
batch.size=32768
linger.ms=10

# Buffer configuration
buffer.memory=67108864  # 64MB
```

#### Error Handling

**Best Practices:**

1. **Handle Callbacks**: Always provide callbacks to handle success/failure
2. **Use Idempotence**: Enable idempotence for at-least-once semantics
3. **Monitor Errors**: Track error rates and retry rates
4. **Use Transactions**: For exactly-once across multiple partitions

### Consumer Best Practices

#### Configuration

**Recommended Consumer Settings:**

```properties
# Manual commit for better control
enable.auto.commit=false

# Set appropriate session timeout
session.timeout.ms=10000

# Set appropriate heartbeat interval
heartbeat.interval.ms=3000

# Set appropriate max poll interval
max.poll.interval.ms=300000

# Use read_committed for transactional topics
isolation.level=read_committed
```

#### Offset Management

**Best Practices:**

1. **Manual Commit**: Use manual commit for better control
2. **Commit After Processing**: Commit after successful processing
3. **Handle Rebalance**: Use rebalance listener to commit offsets
4. **Monitor Lag**: Monitor consumer lag to detect issues

### Operational Best Practices

#### Monitoring

**Key Metrics to Monitor:**

```
Critical Metrics:
- Under-replicated partitions
- Offline partitions
- Consumer lag
- Request latency
- Disk I/O latency
- CPU usage
- Memory usage
```

#### Backup and Recovery

**Backup Strategy:**

```
Backup Best Practices:

1. Replication is Primary Backup
   - Use replication factor >= 3
   - Data replicated across brokers

2. MirrorMaker for Cross-Cluster Replication
   - Replicate to disaster recovery cluster
   - Active-passive or active-active

3. Topic Configuration Backup
   - Backup topic configurations
   - Use kafka-configs.sh to export configs

4. Consumer Offsets Backup
   - Offsets stored in __consumer_offsets
   - Compacted topic, automatically replicated
```

#### Security

**Security Best Practices:**

```
Security Checklist:

1. Enable Authentication
   - Use SASL/SCRAM or SSL
   - Disable PLAINTEXT in production

2. Enable Authorization
   - Use ACLs to control access
   - Principle of least privilege

3. Enable Encryption
   - Use SSL/TLS for data in transit
   - Use disk encryption for data at rest

4. Regular Security Audits
   - Review ACLs regularly
   - Rotate certificates periodically
   - Update credentials regularly
```

### Performance Best Practices

#### Throughput Optimization

```
Throughput Optimization Checklist:

Producer:
- Increase batch.size
- Add linger.ms
- Use compression
- Increase buffer.memory

Consumer:
- Increase fetch.max.bytes
- Increase max.partition.fetch.bytes
- Increase fetch.min.bytes
- Increase max.poll.records

Broker:
- Use SSD storage
- Optimize disk I/O
- Tune JVM settings
- Use multiple log directories
```

#### Latency Optimization

```
Latency Optimization Checklist:

Producer:
- Reduce batch.size
- Set linger.ms=0
- Disable compression
- Reduce buffer.memory

Consumer:
- Reduce fetch.max.wait.ms
- Reduce fetch.min.bytes
- Reduce max.poll.records

Broker:
- Use faster storage
- Reduce network latency
- Optimize GC settings
```

### Disaster Recovery

**Disaster Recovery Plan:**

```
Disaster Recovery Checklist:

1. Cluster Failure
   - Have backup cluster ready
   - Use MirrorMaker for replication
   - Test failover procedures

2. Data Center Failure
   - Replicate across data centers
   - Use multi-region deployment
   - Test cross-region failover

3. Broker Failure
   - Ensure replication factor >= 3
   - Monitor under-replicated partitions
   - Have spare brokers ready

4. Network Partition
   - Use appropriate timeouts
   - Monitor network health
   - Have manual intervention procedures
```

## Conclusion

Apache Kafka is a powerful distributed streaming platform that provides:

- **Scalability**: Through partitioning and horizontal scaling
- **Fault Tolerance**: Through replication and leader election
- **Performance**: Through batching, compression, and zero-copy
- **Reliability**: Through acknowledgments and idempotence
- **Flexibility**: Through retention policies and compaction

Understanding Kafka's internal workings is essential for:
- Designing efficient streaming architectures
- Troubleshooting production issues
- Optimizing performance
- Ensuring reliability

This guide provides a comprehensive foundation for working with Kafka in production environments.
