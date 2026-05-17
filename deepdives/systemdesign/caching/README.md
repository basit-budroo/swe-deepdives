# Caching in System Design: A Deep Dive

## Table of Contents
1. [Caching Fundamentals](#caching-fundamentals)
2. [Distributed Caching](#distributed-caching)
3. [Caching Strategies](#caching-strategies)
4. [Cache Eviction Policies](#cache-eviction-policies)
5. [Cache Invalidation](#cache-invalidation)
6. [Redis: Architecture and Internals](#redis-architecture-and-internals)
7. [Memcached: Architecture and Internals](#memcached-architecture-and-internals)
8. [Facebook's Memcached Scaling Story](#facebooks-memcached-scaling-story)

---

## Caching Fundamentals

### Core Concepts

#### Cache Hit Ratio

The cache hit ratio measures the effectiveness of your cache:

```
Hit Ratio = (Cache Hits / Total Requests) × 100%
```

A 95% hit ratio means only 5% of requests reach your database, effectively reducing database load by 20x. This metric directly correlates with system performance. For example, if your database can handle 1,000 queries per second (QPS) and you have 10,000 incoming QPS, a 95% cache hit ratio means only 500 QPS hit the database, which is well within capacity.

**Why it matters**: Even small improvements in hit ratio can dramatically reduce latency percentiles and infrastructure costs. Moving from 90% to 95% hit ratio halves the database load.

#### Temporal and Spatial Locality

Caching relies on two fundamental principles that make it effective:

- **Temporal locality**: Programs access the same data repeatedly over short time periods. For example, a user viewing their profile will likely refresh it multiple times in a session.
- **Spatial locality**: Programs access data located near recently accessed data. For example, when reading a user's feed, you're likely to access posts from the same friends.

Most real-world workloads exhibit these patterns. Consider a social media timeline: users repeatedly view their own feed (temporal locality) and access posts from the same subset of friends (spatial locality).

#### Time-to-Live (TTL)

TTL assigns an expiration time to each cache entry, after which it becomes invalid regardless of access patterns. TTL prevents indefinitely stale data and provides a safety net when explicit invalidation fails.

**The TTL Trade-off**: Setting appropriate TTL values requires balancing freshness against hit ratio:
- **Shorter TTLs** (e.g., 30 seconds): Ensure fresher data but increase database load as entries expire more frequently. Good for rapidly changing data like stock prices or live scores.
- **Longer TTLs** (e.g., 24 hours): Improve performance but risk serving outdated information. Good for relatively static data like product catalogs or user profile metadata.

**Practical TTL Strategy**: Match TTL to data volatility and business requirements:
- User sessions: 30 minutes - 2 hours (security + user experience)
- User profiles: 1 - 24 hours (infrequent changes)
- Product catalog: 12 - 24 hours (daily updates)
- Configuration: 5 - 60 minutes (quick propagation)
- Real-time analytics: 1 - 60 seconds (freshness critical)

**TTL Smearing**: For large cache invalidations (e.g., configuration changes), add random jitter to TTLs to prevent thundering herd problems where all entries expire simultaneously.

---

## Distributed Caching

### Why Distributed Caching?

Single-node caches have fundamental limitations that distributed caching addresses:

1. **Capacity Limitations**: A single server has finite memory. If your working set exceeds available RAM, you either need a larger machine or must distribute across multiple servers.
2. **Single Point of Failure (SPOF)**: If the single cache node fails, all cache misses hit the database simultaneously, potentially causing cascading failures.
3. **Geographic Latency**: A single cache location cannot serve users globally with low latency. Distributed caching allows placing cache nodes closer to users.

**Real-world example**: Netflix caches over 95% of catalog metadata across multiple regions, enabling sub-10ms response times globally. Without distributed caching, users in Asia would experience hundreds of milliseconds of latency accessing a US-based cache.

---

### Core Architecture Components

#### Cache Servers (Data Plane)
Cache servers hold cached data in memory and serve client requests. Each server manages a portion of the total keyspace, determined by the partitioning scheme. Popular implementations include Redis and Memcached, each optimized for different use cases.

#### Client Libraries
Client libraries implement the routing logic that connects applications to the correct cache node. A well-designed client library handles:
- **Consistent hashing** to determine which node holds a given key
- **Connection pooling** for efficiency (reusing TCP connections)
- **Retry logic** for transient failures with exponential backoff
- **Local topology caching** to avoid metadata lookups on every request

The choice between "smart clients" (that understand cluster topology) and "thin clients" (that rely on proxies) affects both performance and operational complexity. Smart clients reduce network hops but increase client complexity.

#### Metadata Store
Maintains cluster state including node membership, partition assignments, and configuration. Distributed coordination systems like etcd or ZooKeeper provide strong consistency guarantees for this critical metadata. Some systems embed this functionality directly (like Redis Cluster's gossip protocol), trading consistency guarantees for reduced operational dependencies.

#### Replication Manager
Coordinates data redundancy across nodes. This component handles leader election when primary nodes fail, monitors replication lag, and orchestrates failover procedures. The replication topology (leader-follower, multi-leader, or leaderless) fundamentally shapes consistency and availability characteristics.

---

### Data Partitioning with Consistent Hashing

#### The Problem with Modular Hashing

The simplest approach uses modular hashing:
```
node = hash(key) % N
```
where N is the number of nodes. This works until you add or remove a node. Suddenly most keys map to different nodes, causing a cascade of cache misses. For a 100-node cluster, adding one node remaps approximately 99% of keys.

**Impact**: This "cache remapping storm" can overwhelm your database with simultaneous cache misses, potentially causing cascading failures. All clients suddenly request the same data from the database at once.

#### Consistent Hashing Solution

Consistent hashing solves this problem elegantly by minimizing the number of keys that need to be remapped:

1. Imagine arranging all possible hash values on a circular ring from 0 to 2^32-1
2. Each cache node is assigned a position on this ring based on hashing its identifier (e.g., IP address)
3. To find which node stores a given key, hash the key, locate that position on the ring, and walk clockwise until you encounter a node
4. When nodes join or leave, only keys in the affected range need remapping (typically just 1/N of the total keyspace)

**Example**: With 10 nodes, adding one node only remaps ~10% of keys instead of ~90%. The affected keys are those whose hash falls between the new node and its previous node (counter-clockwise neighbor).

#### Virtual Nodes

Virtual nodes (vnodes) improve on basic consistent hashing by assigning each physical node multiple positions on the ring. Instead of one hash position per server, a node might occupy 150-200 virtual positions (each vnode gets a different hash by appending a suffix like "node1#1", "node1#2", etc.).

**Benefits**:
- **More uniform key distribution**: Prevents hot spots when physical nodes happen to cluster on the ring due to hash collisions
- **Weighted distribution**: More powerful servers can handle proportionally more data by having more vnodes
- **Better load balancing**: When nodes are added or removed, the impact is spread more evenly across the cluster
- **Failure isolation**: If a physical node fails, its vnodes are distributed across many other nodes rather than overwhelming a single replacement

---

### Replication for Fault Tolerance

Distributing data across nodes improves scalability but introduces a new problem: if any single node fails, the data it holds becomes unavailable. Replication addresses this by maintaining copies of each cache entry on multiple nodes.

#### Primary-Replica Replication

Each partition has one primary node that handles all writes and one or more replica nodes that maintain copies of the data.

**Synchronous Replication**:
- Writes flow to the primary, which replicates changes to replicas before acknowledging
- Guarantees no data loss on primary failure
- Increases write latency (must wait for all replicas to acknowledge)
- Example: Redis with WAIT command

**Asynchronous Replication**:
- Primary acknowledges immediately and replicates in the background
- Offers better performance but risks losing recent writes if the primary fails before replication completes
- Default in most systems (Redis, Memcached via client-side replication)
- Example: Redis default replication

**Trade-off decision**: Choose synchronous when data durability is critical (financial transactions). Choose asynchronous when performance is more important than perfect durability (social media likes).

#### Failover Process

When a primary node fails, the system must promote a replica to become the new primary. This process requires:
- **Consensus**: Remaining nodes must agree on which replica becomes the new primary (typically using Raft or Paxos)
- **Edge case handling**: What about writes that reached the old primary but weren't replicated? These are lost in asynchronous replication
- **Client discovery**: How do clients discover the new primary? Through DNS updates, configuration changes, or client library redirection
- **Split-brain prevention**: Ensure two nodes don't both think they're primary, which would cause data divergence

#### Replication Factor

The replication factor determines how many copies of each entry exist. A replication factor of 3 means each key exists on three nodes, tolerating up to two simultaneous failures.

**Trade-offs**:
- Higher replication factors improve durability and read availability (reads can go to any replica)
- Increase storage costs (3x storage for replication factor of 3)
- Increase write amplification (each write must be sent to N nodes)
- Most production systems use replication factor between 2 and 3

---

### Consistency in Distributed Caches

Distributed caching forces you to confront the CAP theorem. In the presence of network partitions, you must choose between consistency (all nodes see the same data) and availability (all requests receive a response). Most caching systems prioritize availability over consistency, accepting that stale data is better than no data.

#### Read-After-Write Consistency

Guarantees that after updating a key, subsequent reads return the new value. This becomes challenging in distributed systems because if a client writes to the primary and immediately reads from a replica that hasn't received the update yet, they'll see stale data.

**Example scenario**: User updates their profile picture. Write goes to primary. User immediately refreshes their profile. Read goes to a replica that hasn't received the update yet. User sees their old picture.

**Solutions**:
- **Route reads to the primary**: Guarantees consistency but reduces read scalability (primary becomes bottleneck)
- **Wait for replication**: Acknowledge writes only after replicas confirm (increases write latency)
- **Version vectors**: Attach version numbers to data; detect stale reads and retry from primary

#### Eventual Consistency

Accepts that replicas will converge over time but might serve different values during the convergence window. This model works well for many caching use cases where slight staleness is acceptable (e.g., seeing a post that's 1 second old vs. seeing it immediately).

**Use cases**: Social media feeds, recommendation systems, analytics dashboards - where seeing slightly stale data is acceptable.

#### Consistency Through Invalidation

Instead of trying to keep replicas synchronized, immediately invalidate cache entries when the underlying data changes. Combined with Change Data Capture (CDC), where database changes automatically trigger cache invalidations through message queues (Kafka, RabbitMQ), this pattern provides strong consistency guarantees without complex distributed consensus.

**Example**: When a user updates their profile in the database, CDC captures this change and publishes a message to Kafka. All cache nodes subscribe to this topic and invalidate the user's profile cache entry. Next read fetches fresh data.

---

## Caching Strategies

Caching strategies determine how data flows between your application, cache, and database. Choosing the right strategy depends on your application's specific requirements and access patterns.

### 1. Cache-Aside (Lazy Loading)

This is the most commonly used caching approach. The cache sits on the side and the application directly talks to both the cache and the database.

**Flow**:
```
Application → Check Cache → (Hit) → Return to Client
                ↓ (Miss)
            Query Database
                ↓
            Return to Client + Store in Cache
```

**Code Example**:
```java
public class UserService {
    private Cache cache;
    private Database db;
    
    public User getUser(String userId) {
        // Try to get from cache
        User user = cache.get("user:" + userId);
        if (user != null) {
            return user;  // Cache hit
        }
        
        // Cache miss - fetch from database
        user = db.query("SELECT * FROM users WHERE id = ?", userId);
        
        // Store in cache for future requests
        cache.set("user:" + userId, user, Duration.ofHours(1));
        
        return user;
    }
}
```

**Pros**:
- **Resilient to cache failures**: If cache goes down, system still operates by going directly to database
- **Flexible data model**: Cache can store different data structures than database (e.g., pre-aggregated results, denormalized data)
- **Simple to implement**: No complex cache infrastructure needed
- **Efficient**: Only caches data that's actually requested (lazy loading)

**Cons**:
- **Cache miss penalty**: First request for any data incurs extra latency (database query + cache write)
- **Stale data risk**: If database is updated directly (bypassing cache), data becomes stale until TTL expires
- **Application complexity**: Application must manage cache population logic
- **Thundering herd**: When popular key expires, many requests simultaneously hit database

**Use Cases**:
- Read-heavy workloads where data is requested many times after being loaded
- When cache can tolerate occasional failures
- When you need flexibility in cache data structure
- Most web applications (this is the default strategy for Memcached and Redis)

**Real-world example**: Twitter uses cache-aside for user profiles. When a profile is requested, it's fetched from database and cached. Subsequent requests hit the cache. When user updates profile, cache is invalidated.

---

### 2. Read-Through Cache

Read-through cache sits in-line with the database. When there is a cache miss, the cache itself loads missing data from the database, populates the cache, and returns it to the application. The application doesn't know about the database - it only talks to the cache.

**Flow**:
```
Application → Request from Cache → (Hit) → Return to Client
                    ↓ (Miss)
                Cache fetches from Database
                    ↓
                Cache updates itself
                    ↓
                Return to Client
```

**Code Example**:
```java
public class UserService {
    private ReadThroughCache cache;
    
    // With read-through, application code is simpler
    public User getUser(String userId) {
        // Just request from cache - cache handles misses automatically
        User user = cache.get("user:" + userId);  // Cache fetches from DB if miss
        return user;
    }
}
```

**Pros**:
- **Simpler application code**: Application doesn't need to handle cache misses or database logic
- **Consistent data model**: Cache stores data exactly as it exists in database
- **Reduced application complexity**: Cache library handles all the complexity
- **Easier to maintain**: Database changes only affect cache configuration, not application code

**Cons**:
- **Less flexible**: Data model in cache must match database exactly (can't store pre-aggregated results)
- **First request penalty**: First request for any data always results in cache miss
- **More complex cache**: Cache needs database integration (connection, query logic)
- **Similar stale data issues**: Still faces stale data problems like cache-aside

**Use Cases**:
- Read-heavy workloads when same data is requested many times
- When you want to simplify application code and delegate caching logic to cache layer
- When using managed cache services (e.g., DynamoDB Accelerator, Azure Cache for Redis)
- When cache and database data models should be identical

**Real-world example**: DynamoDB Accelerator (DAX) is a read-through cache. Your application talks to DAX, which automatically fetches from DynamoDB on cache misses. This simplifies your application code significantly.

---

### 3. Write-Through Cache

Data is first written to the cache and then to the database synchronously. The cache sits in-line with the database and writes always go through the cache to the main database. This ensures cache and database are always consistent.

**Flow**:
```
Application → Write to Cache → Write to Database → Acknowledge
                    ↓                    ↓
                Cache Updated      Database Updated
```

**Code Example**:
```java
public class UserService {
    private Cache cache;
    private Database db;
    
    public String updateUser(String userId, UserData data) {
        // Write to cache first
        cache.set("user:" + userId, data);
        
        // Then write to database
        db.execute("UPDATE users SET ? WHERE id = ?", data, userId);
        
        // Only acknowledge after both succeed
        return "success";
    }
}
```

**Pros**:
- **Strong consistency**: Cache always remains consistent with database
- **No stale reads**: Since cache is updated with database, reads always get fresh data
- **Eliminates invalidation**: When paired with read-through, eliminates need for cache invalidation logic
- **No data loss in cache**: No risk of losing data in cache since it's also in database

**Cons**:
- **Higher write latency**: Two write operations (cache + database) before acknowledging
- **Slower write performance**: Not suitable for write-heavy workloads
- **All writes must go through cache**: If you bypass cache, consistency is broken
- **Cache dependency**: If cache is down, writes fail (unless you have fallback logic)

**Use Cases**:
- Applications where consistency matters more than write latency
- Financial transactions, inventory management, banking systems
- When paired with read-through for complete consistency (read-through + write-through)
- When you need guaranteed cache consistency

**Real-world example**: Banking systems use write-through for account balances. When a transaction occurs, both cache and database are updated synchronously. This ensures that any read (whether from cache or database) returns the correct balance.

---

### 4. Write-Around

Data is written directly to the database, bypassing the cache. The cache only populates through read operations (lazy loading). This prevents the cache from filling with data that might never be read again.

**Flow**:
```
Application → Write to Database → Acknowledge
                ↓
            (Cache not updated)
                
Application → Read from Cache → (Miss) → Read from Database → Populate Cache
```

**Code Example**:
```java
public class LogService {
    private Cache cache;
    private Database db;
    
    public String writeLog(LogEntry logEntry) {
        // Write directly to database, bypassing cache
        db.execute("INSERT INTO logs VALUES ?", logEntry);
        return "success";
    }
    
    public List<LogEntry> readLogs(String userId) {
        // Cache is populated only on read
        List<LogEntry> logs = cache.get("logs:" + userId);
        if (logs == null) {
            logs = db.query("SELECT * FROM logs WHERE user_id = ?", userId);
            cache.set("logs:" + userId, logs, Duration.ofMinutes(5));
        }
        return logs;
    }
}
```

**Pros**:
- **Prevents cache pollution**: Cache doesn't fill with data that's never read
- **Good for write-heavy workloads**: No cache write overhead on every write
- **Reduces write latency**: No cache write operation
- **Efficient for write-once, read-maybe data**: Only caches data that's actually read

**Cons**:
- **Guaranteed cache miss**: First read after write is always a cache miss
- **Not suitable for write-read pattern**: If data is written and immediately read, you'll miss the cache
- **Can be combined with read-through**: Often paired with read-through for better read performance

**Use Cases**:
- Write-heavy workloads where most written data is never subsequently accessed
- Real-time logs, chatroom messages, analytics events
- Data that's written once and read infrequently (if at all)
- Audit trails, clickstream data

**Real-world example**: Chat applications use write-around for messages. When a user sends a message, it's written directly to the database. The message cache is only populated when someone reads the conversation. This prevents the cache from filling with messages from inactive conversations.

---

### 5. Write-Back (Write-Behind)

Application writes data to the cache, which stores the data and acknowledges immediately. The cache then asynchronously writes the data back to the database. This is similar to write-through but with asynchronous database writes instead of synchronous.

**Flow**:
```
Application → Write to Cache → Acknowledge (immediate)
                ↓
            Cache queues write
                ↓
            Cache asynchronously writes to Database
```

**Code Example**:
```java
public class CounterService {
    private WriteBackCache cache;
    private Database db;
    
    public String incrementCounter(String counterId) {
        // Write to cache only, acknowledge immediately
        cache.incr("counter:" + counterId);
        return "success";  // Returns immediately
    }
    
    // Background process periodically flushes to database
    public void flushToDatabase() {
        Map<String, Integer> counters = cache.getAllCounters();
        for (Map.Entry<String, Integer> entry : counters.entrySet()) {
            String counterId = entry.getKey();
            Integer value = entry.getValue();
            db.execute("UPDATE counters SET value = ? WHERE id = ?", value, counterId);
        }
    }
}
```

**Pros**:
- **Dramatically improves write latency**: Only cache write before acknowledging (microseconds vs milliseconds)
- **Write coalescing**: Multiple updates to same key result in single database write (e.g., 100 increments = 1 DB update)
- **Resilient to database failures**: Cache continues accepting writes even if database is temporarily down
- **Reduces database load**: Batching reduces overall database writes (important for pay-per-write databases like DynamoDB)
- **Excellent for write-heavy workloads**: Can handle millions of writes per second

**Cons**:
- **Data loss risk**: If cache node fails before persistence completes, data is permanently lost
- **Consistency window**: Cache and database differ until async write completes
- **Complex to implement**: Need to handle write queues, retries, failure scenarios
- **Requires monitoring**: Need to track unflushed writes and monitor queue sizes

**Use Cases**:
- High write throughput requirements (analytics event collectors, clickstream data)
- Applications that can tolerate some data loss (non-critical metrics, counters)
- When write latency is critical (real-time systems)
- Systems requiring high write performance (social media likes, views)

**Real-world example**: Instagram uses write-back caching for photo likes. When a user likes a photo, the like is written to cache and acknowledged immediately. A background process asynchronously flushes likes to the database. This provides excellent performance even during viral events with millions of likes per second.

---

### Strategy Comparison

| Strategy | Read Latency | Write Latency | Consistency | Complexity | Best For |
|----------|-------------|---------------|-------------|------------|-----------|
| Cache-Aside | Low (after warm) | Low | Eventual | Low | Read-heavy, simple apps |
| Read-Through | Low (after warm) | Low | Eventual | Medium | Read-heavy, managed caches |
| Write-Through | Low | High | Strong | Medium | Consistency-critical |
| Write-Around | High (first read) | Low | Eventual | Low | Write-heavy, infrequent reads |
| Write-Back | Low | Very Low | Weak | High | Write-heavy, high throughput |

---

## Cache Eviction Policies

Memory is finite, and your cache will inevitably fill up. Eviction policies determine which entries to remove when space is needed for new data. The goal is maximizing cache hit ratio by keeping the data most likely to be requested while discarding data unlikely to be accessed again.

### 1. Least Recently Used (LRU)

**Principle**: Evicts entries that haven't been accessed for the longest time, based on the assumption that recent access predicts future access (temporal locality). If you accessed something recently, you're likely to access it again soon.

**Implementation Details**:

Classic LRU uses a doubly-linked list combined with a hash map:
- **Hash map**: Provides O(1) key lookups
- **Doubly-linked list**: Maintains access order (head = most recent, tail = least recent)
- **On access**: Move entry to head of list
- **On eviction**: Remove from tail of list

**Code Example**:
```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> cache;
    private final Node<K, V> head;
    private final Node<K, V> tail;
    
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;
        
        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.head = new Node<>(null, null);  // dummy head
        this.tail = new Node<>(null, null);  // dummy tail
        this.head.next = this.tail;
        this.tail.prev = this.head;
    }
    
    public V get(K key) {
        if (cache.containsKey(key)) {
            Node<K, V> node = cache.get(key);
            remove(node);
            add(node);  // Move to head (most recent)
            return node.value;
        }
        return null;
    }
    
    public void put(K key, V value) {
        if (cache.containsKey(key)) {
            remove(cache.get(key));
        }
        Node<K, V> node = new Node<>(key, value);
        add(node);
        cache.put(key, node);
        if (cache.size() > capacity) {
            Node<K, V> toRemove = tail.prev;
            remove(toRemove);
            cache.remove(toRemove.key);
        }
    }
    
    private void add(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }
    
    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
}
```

**Pros**:
- Works well for most workloads with temporal locality (typical web applications)
- Simple to understand and implement
- Good performance for typical access patterns
- Predictable behavior

**Cons**:
- Can be expensive to maintain exact LRU order (O(1) but with constant factor overhead)
- Vulnerable to "scan" workloads (sequential access through large dataset pollutes cache)
- Vulnerable to "one-hit wonder" workloads (items accessed once then never again)
- Requires tracking access order for all items (memory overhead)

**Variants**:

- **Segmented LRU (SLRU)**: Divides cache into probationary (new items) and protected (items accessed ≥2 times) segments. Items accessed at least twice move to protected segment, protecting popular items from scan pollution. Used in many production systems.

- **Pseudo-LRU (PLRU)**: Approximation using binary tree of one-bit pointers. Only needs one bit per cache item instead of full timestamp. Used in CPU caches where hardware implementation cost is critical.

- **Clock-Pro**: Approximation of LRU with three "clock hands" that can measure reuse distance. Used in Linux kernel buffer cache. Balances accuracy with implementation cost.

---

### 2. Least Frequently Used (LFU)

**Principle**: Tracks access counts and evicts entries accessed the fewest times. Assumes that frequently accessed items are more valuable and should be kept in cache. If an item is accessed 1000 times and another is accessed once, the LFU keeps the popular one even if it hasn't been accessed recently.

**Implementation Details**:

Classic LFU maintains a counter for each key:
- **Counter**: Incremented on each access
- **Eviction**: Remove item with lowest counter when space needed
- **Challenge**: Items that were popular in the past but are no longer popular remain cached (cache pollution)

**Code Example**:
```java
public class LFUCache<K, V> {
    private final int capacity;
    private final Map<K, CacheItem> cache;
    private int minCount;
    
    private class CacheItem {
        V value;
        int count;
        
        CacheItem(V value, int count) {
            this.value = value;
            this.count = count;
        }
    }
    
    public LFUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.minCount = 0;
    }
    
    public V get(K key) {
        if (cache.containsKey(key)) {
            CacheItem item = cache.get(key);
            item.count++;
            minCount = Math.min(minCount, item.count);
            return item.value;
        }
        return null;
    }
    
    public void put(K key, V value) {
        if (cache.containsKey(key)) {
            CacheItem item = cache.get(key);
            item.value = value;
            item.count++;
            return;
        }
        
        if (cache.size() >= capacity) {
            // Evict item with lowest count
            for (Map.Entry<K, CacheItem> entry : cache.entrySet()) {
                if (entry.getValue().count == minCount) {
                    cache.remove(entry.getKey());
                    break;
                }
            }
        }
        
        cache.put(key, new CacheItem(value, 1));
        minCount = 1;
    }
}
```

**Pros**:
- Better than LRU for workloads with stable popularity distributions (e.g., popular videos on YouTube)
- Keeps consistently popular items even if not recently accessed
- Good for "hot set" workloads where certain items are always popular
- Resistant to scan pollution (one-time access doesn't increase count much)

**Cons**:
- Struggles with shifting popularity (item popular yesterday but not today remains cached due to accumulated count)
- Requires tracking access counts (memory overhead - typically 8-16 bytes per key)
- Can be affected by "cache pollution" from bursty access patterns (sudden spike makes item permanently popular)
- Cold start problem (new items have low counts and get evicted quickly)

**Variants**:

- **LFU with Dynamic Aging (LFUDA)**: Adds cache-age factor to reference count to accommodate shifts in popular objects. When new item is added, all existing counts are increased by a factor, preventing old popular items from dominating forever.

- **Least Frequent Recently Used (LFRU)**: Combines LFU and LRU with privileged (frequently accessed) and unprivileged (infrequently accessed) partitions. Uses LFU for unprivileged and LRU for privileged partition.

- **Redis LFU**: Uses a probabilistic counter requiring only 8 bits per key while approximating access frequency reasonably well. Uses a logarithmic counter that increments with probability inversely proportional to current count (easier to increment small counts, harder for large counts). This prevents any single item from dominating the cache.

**When to choose LFU over LRU**:
- When you have a stable set of popular items (e.g., top 100 videos on YouTube)
- When popularity doesn't shift frequently
- When you want to protect popular items from being evicted during scan operations
- When access frequency is a better predictor than recency for your workload

---

### 3. First In First Out (FIFO)

**Principle**: Evicts blocks in the order they were added, regardless of how often or how many times they were accessed. The oldest item is always evicted first, like a queue.

**Implementation**:
- Simple queue structure
- On add: enqueue item
- On eviction: dequeue oldest item
- No tracking of access patterns

**Code Example**:
```java
public class FIFOCache<K, V> {
    private final int capacity;
    private final Queue<Map.Entry<K, V>> queue;
    private final Map<K, V> cache;
    
    public FIFOCache(int capacity) {
        this.capacity = capacity;
        this.queue = new LinkedList<>();
        this.cache = new HashMap<>();
    }
    
    public V get(K key) {
        return cache.get(key);  // No access tracking
    }
    
    public void put(K key, V value) {
        if (cache.containsKey(key)) {
            return;  // Already exists, no change
        }
        
        queue.offer(new AbstractMap.SimpleEntry<>(key, value));
        cache.put(key, value);
        
        if (queue.size() > capacity) {
            Map.Entry<K, V> oldest = queue.poll();
            cache.remove(oldest.getKey());
        }
    }
}
```

**Pros**:
- Simplest to implement (just a queue)
- Predictable behavior (oldest always goes first)
- Lowest computational overhead (no access tracking)
- No need to track access patterns or maintain complex data structures

**Cons**:
- Ignores access patterns entirely (can evict frequently accessed items)
- Poor hit ratio for most workloads
- Susceptible to cache pollution (one-time access treated same as frequent access)
- Not suitable for typical web application workloads

**Use Cases**:
- Simple implementations where complexity must be minimized
- Workloads with no clear access pattern
- When eviction policy overhead must be absolutely minimal
- Hardware implementations where simplicity is critical (some CPU cache implementations)

---

### 4. Random Replacement (RR)

**Principle**: Selects a random item and discards it when space is needed. No consideration of access patterns or recency.

**Implementation**:
- No access history tracking required
- On eviction: generate random index and remove that item
- Zero overhead for maintaining eviction state

**Pros**:
- No access history tracking required
- Zero overhead for maintaining eviction state
- Used in ARM processors due to simplicity
- Allows efficient stochastic simulation
- Surprisingly effective for some workloads (when access pattern is truly random)

**Cons**:
- No consideration of access patterns
- Unpredictable hit ratios
- Can evict hot items randomly
- Not suitable for workloads with clear access patterns

**Use Cases**:
- Hardware implementations where simplicity is critical
- When overhead must be absolutely minimal
- Some CPU cache implementations (ARM)
- When access patterns are truly random (rare in practice)

---

### 5. Most Recently Used (MRU)

**Principle**: Discards the most recently used items first. Counter-intuitive but effective for specific access patterns.

**Why it works**: For cyclic/sequential access patterns, the most recently accessed item is least likely to be accessed again soon. If you're scanning through a large dataset sequentially, you just accessed item N and you're about to access item N+1, so item N is unlikely to be needed again.

**Pros**:
- Excellent for cyclic/sequential access patterns
- Retains older data which may be needed again
- Better than LRU for repeated scans over large datasets
- Prevents scan pollution (scan data is evicted immediately)

**Cons**:
- Poor for typical temporal locality workloads
- Evicts items that were just accessed (counter-intuitive)
- Not suitable for most web application workloads
- Can have terrible hit ratios for random access patterns

**Use Cases**:
- When files are being repeatedly scanned in a looping sequential pattern (database full table scans)
- Random access patterns with repeated scans over large datasets
- Cyclic access patterns where older data is more likely to be accessed
- Database query result caching for repeated scans

---

### 6. Advanced Algorithms

#### ARC (Adaptive Replacement Cache)

Balances LRU (recency) and LFU (frequency) dynamically by maintaining two lists:
- **T1**: Recently used items (LRU list)
- **T2**: Frequently used items (LFU list)
- **B1**: Ghost list tracking items evicted from T1
- **B2**: Ghost list tracking items evicted from T2

ARC adapts the size of T1 and T2 based on workload characteristics. If an item evicted from T1 is requested again (hit in B1), ARC increases T1 size (workload favors recency). If an item evicted from T2 is requested again (hit in B2), ARC increases T2 size (workload favors frequency).

**Why it's effective**: Automatically adapts to changing workload patterns without manual tuning. Works well for mixed workloads that sometimes favor recency and sometimes favor frequency.

#### SIEVE

Designed specifically for web caches and CDNs where many items are accessed only once (one-hit wonders). Uses:
- **Lazy promotion**: Doesn't update access order on cache hits (only on misses)
- **Quick demotion**: Quickly evicts newly inserted objects if they're not requested again
- **Single FIFO queue**: Simpler than LRU
- **Moving hand**: Pointer that moves through queue selecting eviction candidates

**How it works**: Each object has a "visited" bit. On cache hit, if bit is 0, set it to 1 (lazy promotion). On eviction, if bit is 0, evict immediately (quick demotion). If bit is 1, reset to 0 and keep (give it another chance).

**Why it's effective**: Web caches have high one-hit-wonder ratios (many URLs accessed once then never again). SIEVE filters these out quickly while retaining popular items. Outperforms LRU in CDN workloads.

#### S3-FIFO (2023)

Uses three FIFO queues to balance simplicity with effectiveness:
- **Small queue (10% of cache)**: Filters out one-hit-wonders
- **Main queue (90% of cache)**: Stores popular objects with reinsertion on access
- **Ghost queue**: Tracks metadata of evicted items from small queue

**How it works**: New items go to small queue. If accessed while in small queue, promote to main queue. Items in main queue are reinserted on access (moved to tail). If evicted from small queue, track in ghost queue. If item is requested and found in ghost queue, insert directly into main queue (bypass small queue).

**Why it's effective**: Simpler than LRU (just FIFO queues) but achieves better hit ratios for web cache workloads by filtering one-hit-wonders. The ghost queue catches items that were evicted too early from small queue.

#### 2Q Algorithm

Uses two queues to prevent scan pollution in LRU caches:
- **Q1**: FIFO queue for items accessed once (probationary)
- **Q2**: LRU queue for items accessed multiple times (protected)

**How it works**: New items go to Q1. If accessed again while in Q1, move to Q2. Q2 uses LRU. When eviction needed, prefer evicting from Q1 over Q2. This protects frequently accessed items from being evicted during scan operations.

**Why it's influential**: Developed in 1994 but remains influential. The concept of probationary and protected segments is used in many modern cache implementations (including Redis's segmented LRU).

---

### Eviction Policy Selection Guide

| Workload Characteristic | Recommended Policy | Reasoning |
|------------------------|-------------------|------------|
| Typical temporal locality (web apps) | LRU or Segmented LRU | Recent access predicts future access for most web workloads |
| Stable hot set (popular videos, products) | LFU | Frequency matters more than recency for consistently popular items |
| Shifting popularity (trending content) | Time-decayed LFU or ARC | Need to adapt to changing popularity patterns |
| Sequential/cyclic access (database scans) | MRU | Recently accessed items in scan are least likely to be needed again |
| Simple implementation needed | FIFO or RR | Minimize complexity at cost of hit ratio |
| Web cache/CDN (high one-hit-wonder ratio) | SIEVE or S3-FIFO | Specifically designed to filter out one-time access items |
| Unknown/variable patterns | ARC or adaptive algorithms | Automatically adapts to workload characteristics |

**Default recommendation**: Start with LRU (or Segmented LRU). It works well for most workloads and is the default in Redis and Memcached. Only switch to LFU or adaptive algorithms if you have specific workload characteristics that benefit from them.

---

## Cache Invalidation

Phil Karlton famously said: "There are only two hard things in computer science: cache invalidation and naming things." The difficulty stems from distributed systems' fundamental challenge: coordinating state changes across multiple independent processes. When data changes in your database, how do you ensure all cache nodes stop serving the old value?

### 1. Time-Based Invalidation (TTL)

**Principle**: Every cache entry expires after a configured duration regardless of whether the underlying data changed. This is the simplest form of invalidation.

**How it works**:
```
SET key value EX 3600  # Expires in 1 hour
# After 1 hour, key is automatically deleted
```

**Pros**:
- Simplest approach - no coordination required
- Provides consistency bound (data never more than TTL seconds stale)
- Safety net for other invalidation mechanisms
- Automatic cleanup (no manual deletion needed)
- No cross-service dependencies

**Cons**:
- Trade-off between freshness and hit ratio
- Shorter TTLs increase database load (more frequent cache misses)
- Longer TTLs risk serving outdated information
- Doesn't account for actual data changes (might expire data that's still valid)
- Stale data served until TTL expires even if data changed

**Best Practices**:
- Match TTL to data volatility and business requirements
- Use shorter TTLs for frequently changing data (real-time scores: 30-60 seconds)
- Use longer TTLs for relatively static data (product catalog: 12-24 hours)
- Consider hierarchical TTLs for different data types
- Use TTL smearing for large invalidations (add random jitter to prevent thundering herd)

---

### 2. Event-Based Invalidation

**Principle**: Actively removes cache entries when related data changes in the database. This provides immediate consistency when implemented correctly.

**Implementation Approaches**:

**Application-level invalidation**:
```java
public class UserService {
    private Cache cache;
    private Database db;
    
    public void updateUser(String userId, UserData data) {
        // Update database
        db.execute("UPDATE users SET ? WHERE id = ?", data, userId);
        
        // Invalidate cache
        cache.delete("user:" + userId);
    }
}
```

**Message queue-driven invalidation (CDC)**:
```java
public class CacheInvalidationListener {
    private Cache cache;
    
    @KafkaListener(topics = "user-updates")
    public void onUserUpdate(UserUpdateMessage message) {
        String userId = message.getUserId();
        cache.delete("user:" + userId);
    }
}
```

**Database triggers**:
```sql
CREATE TRIGGER invalidate_user_cache
AFTER UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION notify_cache_invalidation(OLD.id);
```

**Pros**:
- Immediate consistency when implemented correctly
- No stale data served (cache invalidated as soon as data changes)
- Efficient use of cache space (only valid data cached)
- Can be automated through CDC (no application code changes needed)
- Works across multiple cache nodes (if using pub/sub)

**Cons**:
- Requires discipline - every write path must include invalidation
- Bugs cause subtle, hard-to-diagnose staleness issues
- Adds complexity to write paths
- Requires coordination across systems (cache, database, message queue)
- Network partitions can cause invalidation messages to be lost

**Best Practices**:
- Use Change Data Capture (CDC) for automatic invalidation (Debezium + Kafka)
- Implement retry logic for failed invalidations (idempotent operations)
- Monitor invalidation success rates (alerting on high failure rates)
- Use message queues for reliability (Kafka, RabbitMQ with acknowledgments)
- Consider invalidation at write time vs. read time (write-time is simpler)

---

### 3. Version-Based Invalidation

**Principle**: Attaches version numbers to cache keys, allowing old and new versions to coexist during transitions. Instead of invalidating, you create a new version.

**Implementation**:
```
Old key: user_profile_123
New key: user_profile_123_v42
```

When data changes:
1. Increment version (42 → 43)
2. Write new data under new key (user_profile_123_v43)
3. Application starts using new key
4. Old version (v42) expires through TTL naturally

**Code Example**:
```java
public class UserService {
    private Cache cache;
    private MetadataStore metadata;
    private Database db;
    
    public User getUser(String userId) {
        // Get current version from metadata store
        int version = metadata.get("user_version:" + userId, 1);
        String key = "user:" + userId + ":v" + version;
        
        User user = cache.get(key);
        if (user == null) {
            user = db.query("SELECT * FROM users WHERE id = ?", userId);
            cache.set(key, user, Duration.ofHours(1));
        }
        return user;
    }
    
    public void updateUser(String userId, UserData data) {
        // Increment version
        int version = metadata.incr("user_version:" + userId);
        
        // Write to database
        db.execute("UPDATE users SET ? WHERE id = ?", data, userId);
        
        // Write to cache with new version
        String key = "user:" + userId + ":v" + version;
        cache.set(key, data, Duration.ofHours(1));
    }
}
```

**Pros**:
- Eliminates race conditions where invalidation and population occur simultaneously
- No coordination required during updates (no distributed invalidation needed)
- Graceful transition between versions (both old and new coexist briefly)
- Can serve both old and new versions during migration (A/B testing)
- No thundering herd on invalidation (new version populated lazily)

**Cons**:
- Increases storage requirements (multiple versions coexist temporarily)
- Complicates cache warming (need to know which version to warm)
- More complex key management (need version metadata store)
- Potential for old versions to linger if TTL is too long
- Increased memory usage during transitions

**Use Cases**:
- Schema migrations (transition between data representations)
- A/B testing different data representations
- Gradual rollouts of data structure changes
- When you need zero downtime during data migrations

---

### 4. Stampeding Herd Protection

**Problem**: When a popular cache entry expires, hundreds of concurrent requests simultaneously query the database and try to repopulate the cache, overwhelming the database. This is called a "cache stampede" or "thundering herd."

**Example**: A celebrity posts on Twitter. Their profile cache expires. Millions of followers simultaneously request the profile. All requests hit the database simultaneously, potentially causing cascading failures.

**Solutions**:

#### Request Coalescing (Single Flight)
Only one request populates the cache while others wait for the result.

```java
public class UserService {
    private Cache cache;
    private Database db;
    private final ConcurrentHashMap<String, Object> locks = new ConcurrentHashMap<>();
    
    public User getUser(String userId) {
        User user = cache.get(userId);
        if (user != null) {
            return user;
        }
        
        // Ensure only one request fetches from database
        Object lock = locks.computeIfAbsent(userId, k -> new Object());
        synchronized (lock) {
            // Double-check after acquiring lock
            user = cache.get(userId);
            if (user != null) {
                return user;
            }
            
            user = db.query("SELECT * FROM users WHERE id = ?", userId);
            cache.set(userId, user);
            return user;
        }
    }
}
```

#### Probabilistic Early Expiration
Randomly expire entries before actual TTL to spread out refresh requests.

```java
public class UserService {
    private Cache cache;
    private Database db;
    private Random random = new Random();
    
    public User getUser(String userId) {
        User user = cache.get(userId);
        if (user != null) {
            // 5% chance to refresh early
            if (random.nextDouble() < 0.05) {
                refreshUserAsync(userId);
            }
            return user;
        }
        
        user = db.query("SELECT * FROM users WHERE id = ?", userId);
        cache.set(userId, user);
        return user;
    }
    
    private void refreshUserAsync(String userId) {
        // Async refresh implementation
        CompletableFuture.runAsync(() -> {
            User user = db.query("SELECT * FROM users WHERE id = ?", userId);
            cache.set(userId, user);
        });
    }
}
```

#### Lock-Based Population
Use distributed locks (Redis SETNX, Memcached ADD) to ensure only one request populates each cache entry.

```java
public class UserService {
    private Cache cache;
    private Database db;
    
    public User getUser(String userId) {
        User user = cache.get(userId);
        if (user != null) {
            return user;
        }
        
        // Try to acquire lock
        String lockKey = "lock:user:" + userId;
        if (cache.add(lockKey, "1", Duration.ofSeconds(10))) {  // SETNX
            // We got the lock - fetch from database
            user = db.query("SELECT * FROM users WHERE id = ?", userId);
            cache.set(userId, user);
            cache.delete(lockKey);
            return user;
        } else {
            // Someone else is fetching - wait and retry
            try {
                Thread.sleep(100);
                return getUser(userId);  // Retry
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            }
        }
    }
}
```

#### Refresh-Ahead
Proactively refresh popular entries before they expire based on access patterns.

```java
public class UserService {
    private Cache cache;
    private Database db;
    private AccessPatternTracker tracker;
    
    public User getUser(String userId) {
        User user = cache.get(userId);
        if (user != null) {
            // If popular (high access rate), refresh in background
            if (tracker.isPopular(userId)) {
                refreshUserAsync(userId);
            }
            return user;
        }
        
        user = db.query("SELECT * FROM users WHERE id = ?", userId);
        cache.set(userId, user);
        return user;
    }
    
    private void refreshUserAsync(String userId) {
        CompletableFuture.runAsync(() -> {
            User user = db.query("SELECT * FROM users WHERE id = ?", userId);
            cache.set(userId, user);
        });
    }
}
```

---

## Redis: Architecture and Internals

### Overview

Redis (Remote Dictionary Server) is an open-source, in-memory data structure store created by Salvatore Sanfilippo in 2009. It's used as a database, cache, message broker, and streaming engine. Redis has become one of the most popular data platforms, with over 1 billion Docker pulls and used by companies like Twitter, GitHub, Stack Overflow, and Pinterest.

**Key Characteristics**:
- In-memory storage for microsecond-level latency (typically <1ms for simple operations)
- Rich data structures (String, List, Set, Hash, Sorted Set, Bitmap, HyperLogLog, Geospatial)
- Single-threaded command processing (simplifies concurrency, eliminates lock contention)
- Optional persistence to disk (RDB snapshots, AOF logs, or both)
- Built-in replication and high availability (Sentinel, Cluster)
- Support for transactions (MULTI/EXEC), pub/sub, and Lua scripting
- Atomic operations on data structures

### Why Redis is So Fast

#### 1. In-Memory Storage

Redis holds all data in memory (RAM). Memory reads and writes are generally 1,000X - 10,000X faster than disk reads/writes. This is the primary reason for Redis's performance. A typical Redis operation takes ~100 microseconds on a local network, compared to ~10-100 milliseconds for a database query.

**Trade-off**: Memory is expensive and finite. Redis is limited by available RAM. A 64GB server can only store ~64GB of data (minus overhead). This is why Redis is typically used for caching hot data rather than storing entire datasets.

#### 2. Optimized Data Structures

Redis stores data in specialized data structures optimized for fast reads and writes, avoiding the serialization/deserialization overhead typical of on-disk databases:

**Strings**:
- Binary-safe (can store any binary data)
- Maximum size: 512MB
- Operations: SET, GET, INCR, DECR, APPEND, STRLEN
- Use cases: Session tokens, counters, simple key-value storage

**Lists**:
- Implemented as linked lists (or quicklists - linked lists of ziplists)
- Fast push/pop from head or tail: O(1)
- Range operations: O(S+N) where S is offset and N is length
- Maximum size: 2^32-1 elements
- Use cases: Message queues, timelines, activity feeds

**Sets**:
- Unordered collections of unique strings
- Fast membership testing: O(1)
- Set operations (union, intersection, difference): O(N)
- Maximum size: 2^32-1 elements
- Use cases: Unique visitors, tags, relationships

**Hashes**:
- Maps between string fields and string values (like a hashmap)
- Perfect for representing objects
- Field access: O(1)
- Maximum size: 2^32-1 field-value pairs
- Use cases: User profiles, product information, session data

**Sorted Sets (ZSets)**:
- Sets with a score (floating-point number) associated with each element
- Ordered by score (elements with same score ordered lexicographically)
- Range queries: O(log(N) + M) where M is range size
- Ranking operations: O(log(N))
- Maximum size: 2^32-1 elements
- Use cases: Leaderboards, ranking systems, priority queues

**Bitmaps**:
- String type treated as array of bits
- Bit operations: SETBIT, GETBIT, BITCOUNT, BITOP
- Memory efficient for boolean data (1 bit per value)
- Use cases: User flags, attendance tracking, feature toggles

**HyperLogLog**:
- Probabilistic data structure for counting unique items
- Uses minimal memory (12KB for counting billions of items)
- ~1% error rate
- Use cases: Unique visitors, page view counting

#### 3. Single-Threaded Event Loop

Redis processes commands using a single thread (per Redis instance), which:
- Eliminates lock contention (no locks needed for data structures)
- Simplifies concurrency control (no race conditions to worry about)
- Reduces context switching overhead
- Makes command execution predictable and deterministic

**How it works**:
```
Client 1 → [Event Loop] → Command Queue → Single Thread → Response
Client 2 → [Event Loop] → Command Queue → Single Thread → Response
Client 3 → [Event Loop] → Command Queue → Single Thread → Response
```

The event loop uses I/O multiplexing (epoll on Linux, kqueue on BSD/ macOS, IOCP on Windows) to handle multiple connections efficiently without blocking.

**Note**: While command processing is single-threaded, Redis uses multiple threads for:
- Background persistence (RDB snapshotting, AOF rewriting)
- Unlinking large keys (asynchronous deletion)
- Some I/O operations in Redis 6.0+ (threaded I/O for reads/writes)

**Why single-threaded is fast**:
- No lock overhead (locks are expensive)
- No context switching between threads
- CPU cache locality (data stays in CPU cache)
- Predictable performance (no thread contention)

**When single-threaded is a problem**:
- CPU-bound operations (complex Lua scripts, large set operations)
- Network bandwidth saturation
- For these cases, use Redis Cluster to distribute load across multiple Redis instances

#### 4. Efficient Network Handling

Redis uses an event-driven architecture with:
- Non-blocking I/O (never blocks on network operations)
- Event loop processing (single thread handles all connections)
- Multiplexing (epoll/kqueue/IOCP) for efficient connection handling
- Minimal system call overhead (batching network operations)

**Connection handling**:
- Each connection is a file descriptor
- Event loop monitors all file descriptors for read/write readiness
- When data arrives on a connection, event loop reads and processes command
- When response ready, event loop writes response

This architecture allows a single Redis instance to handle tens of thousands of concurrent connections efficiently.

---

### Redis Architectures

#### 1. Single Redis Instance

The simplest deployment. A single Redis instance handles all requests.

**Pros**:
- Simplest to set up and operate (no coordination needed)
- No coordination overhead
- Maximum performance for single machine (no network latency between nodes)
- Easy to debug and troubleshoot

**Cons**:
- Single point of failure (if Redis goes down, all cached data is lost)
- Limited by single machine capacity (memory, CPU, network)
- Data loss on failure (unless persistence is enabled)
- Not suitable for production workloads requiring high availability

**Use Cases**:
- Development environments
- Small applications where downtime is acceptable
- Caching where data loss is acceptable (can be rebuilt from database)
- Prototyping and testing

---

#### 2. Redis High Availability (HA) with Replication

Main deployment with one primary (master) and one or more secondary (replica) instances kept in sync through replication.

**Replication Process**:
- Every primary instance has a replication ID and an offset
- Offset is incremented for every write operation on the primary
- Secondary instances connect to primary and request replication
- Primary sends write commands to replicas as they execute
- Replicas replay commands to maintain identical dataset

**Partial Synchronization**:
- If replica is just a few offsets behind, it receives remaining commands
- Only sends the commands that the replica missed
- Fast and efficient for temporary network partitions

**Full Synchronization**:
1. Primary creates a new RDB snapshot (point-in-time snapshot of dataset)
2. Primary sends snapshot to replica
3. Primary buffers intermediate updates between snapshot cut-off and current offset
4. Replica loads snapshot into memory
5. Primary sends buffered updates to replica
6. Normal replication resumes

**Pros**:
- Read scaling (reads can go to replicas, reducing primary load)
- Failover capability (replica can be promoted to primary if primary fails)
- Data redundancy (data exists on multiple nodes)
- Better availability (replica can serve reads if primary is down)

**Cons**:
- Increased operational complexity (need to manage replication topology)
- Replication lag (replicas might be slightly behind primary)
- More infrastructure (multiple Redis instances)
- Eventual consistency (reads from replica might be stale)

**Use Cases**:
- Production workloads requiring high availability
- Read-heavy workloads (scale reads across replicas)
- Applications where slight read staleness is acceptable

---

#### 3. Redis Sentinel

Provides high availability including monitoring, notification, automatic failover, and configuration provider.

**Architecture**:
- Multiple Sentinel processes (typically 3 or 5 for quorum) monitor Redis instances
- Sentinels agree on master failure via Raft-like consensus algorithm
- Promote replica to master when master fails
- Notify clients of new master (via pub/sub channel)
- Reconfigure other replicas to replicate from new master

**How failover works**:
1. Sentinel detects master is down (after configurable number of checks)
2. Sentinel sends sentinel-is-master-down-by-addr to other Sentinels
3. Sentinels vote on whether to failover (need majority/quorum)
4. If vote passes, Sentinel selects best replica (based on replication offset)
5. Sentinel promotes selected replica to master (SLAVEOF NO ONE)
6. Sentinel reconfigures other replicas to replicate from new master
7. Sentinel notifies clients of new master

**Pros**:
- Automatic failover (no manual intervention needed)
- High availability (can tolerate node failures)
- Monitoring and alerting (Sentinels monitor instance health)
- Configuration provider (clients can ask Sentinel for current master)

**Cons**:
- Additional infrastructure to manage (need multiple Sentinel instances)
- Failover takes time (typically seconds, depending on configuration)
- Split-brain risk if not configured correctly (need odd number of Sentinels)
- More complex deployment and operations

**Use Cases**:
- Production workloads requiring automatic failover
- Applications that can tolerate brief downtime during failover
- When you don't want to manually handle failover

---

#### 4. Redis Cluster

Built-in sharding and high availability solution. Automatically partitions data across multiple Redis nodes.

**Architecture**:
- Data automatically partitioned across multiple Redis nodes using hash slots
- 16384 hash slots (slot = CRC16(key) % 16384)
- Each node handles a subset of slots (and thus a subset of keys)
- Automatic failover within shard (replicas of each master)
- Clients can connect to any node (redirected to correct node for key)
- Nodes gossip to maintain cluster state (no central coordinator)

**How sharding works**:
```
CRC16("user:123") % 16384 = 5432
Slot 5432 is assigned to Node B
Request for user:123 goes to Node B
```

**Cross-slot operations**:
- Most operations work on single key (no problem)
- Multi-key operations (MGET, MSET, Lua scripts) require keys in same slot
- Use hash tags to force keys to same slot: `{user123}:profile` and `{user123}:settings` go to same slot

**Pros**:
- Automatic sharding (no manual partitioning needed)
- High availability built-in (each master has replicas)
- Linear scalability (add more nodes to handle more data/traffic)
- No external coordination service needed (gossip protocol)

**Cons**:
- Limited cross-slot operations (can't easily operate on keys in different slots)
- More complex client libraries required (must handle redirections)
- Some operations not supported (multi-key transactions across slots)
- Network partitions can cause availability issues (CAP theorem trade-offs)
- Data redistribution when adding/removing nodes (can be slow for large datasets)

**Use Cases**:
- Large datasets that don't fit on single machine
- High write throughput that exceeds single Redis capacity
- Production workloads requiring both sharding and high availability

---

### Redis Persistence Models

Redis offers multiple persistence options to balance performance and durability. Unlike Memcached (which has no persistence), Redis can persist data to disk, allowing it to be used as a database, not just a cache.

#### 1. No Persistence

Fastest option with no durability guarantees. Data is lost on restart.

**Configuration**:
```redis
save ""  # Disable all persistence
```

**Pros**:
- Fastest performance (no disk I/O overhead)
- Simplest configuration
- Lowest resource usage (no disk space for persistence)

**Cons**:
- Data loss on restart or crash
- Not suitable for critical data
- Pure caching use case only

**Use Cases**:
- Pure caching where data loss is acceptable
- Temporary data (sessions, short-lived cache)
- Development/testing environments
- When data can be rebuilt from database

---

#### 2. RDB (Redis Database)

Performs point-in-time snapshots of your dataset at specified intervals. Creates a binary file containing the entire dataset at a specific moment in time.

**How It Works**:
1. Redis forks the main process (using fork() system call)
2. Child process writes entire dataset to temporary RDB file
3. When complete, replaces old RDB file atomically (rename operation)
4. Parent process continues serving requests during snapshot

**The Magic of Forking and Copy-on-Write**:
- Parent and child processes share memory initially (copy-on-write)
- If no changes occur during snapshot, no new memory allocations needed
- If changes occur, kernel writes to new pages (copy-on-write)
- Child process has consistent memory snapshot
- Only fraction of memory is used for snapshot (only changed pages)

**Configuration**:
```redis
save 900 1      # Save after 900 seconds if at least 1 key changed
save 300 10     # Save after 300 seconds if at least 10 keys changed
save 60 10000   # Save after 60 seconds if at least 10000 keys changed
```

**Pros**:
- Compact file representation (binary format, efficient compression)
- Fast recovery (file loads quickly compared to replaying logs)
- Good for backups (point-in-time snapshots for disaster recovery)
- Minimal performance impact during normal operation (forking is fast)
- Child process doesn't block parent (parent continues serving requests)

**Cons**:
- Data between snapshots is lost (if Redis crashes, you lose data since last snapshot)
- Forking can cause momentary delay for large datasets (copy-on-write overhead)
- Not ideal for data durability guarantees (might lose minutes of data)
- Forking requires sufficient memory (copy-on-write can double memory usage temporarily)

**Use Cases**:
- When you can tolerate losing some data (minutes worth)
- When you need fast recovery from backups
- When disk space is a concern (RDB is compact)
- When performance is more important than perfect durability

---

#### 3. AOF (Append Only File)

Logs every write operation the server receives. Operations are replayed at server startup to reconstruct the dataset. Provides better durability than RDB at the cost of larger file size and slower recovery.

**How It Works**:
1. Every write command is appended to AOF buffer in memory
2. Buffer is flushed to disk based on fsync policy
3. fsync() ensures data is physically written to disk (not just in OS cache)
4. On restart, Redis replays all commands from AOF to reconstruct dataset

**fsync Policies**:
```redis
appendonly yes
appendfsync always    # fsync on every write (safest, slowest)
appendfsync everysec  # fsync once per second (good balance, default)
appendfsync no        # Let OS handle flushing (fastest, least safe)
```

**AOF Rewriting**:
- AOF file grows over time (every write is logged)
- Redis can rewrite AOF file in background to minimize file size
- Rewriting creates optimized version of current dataset
- Uses fork() similar to RDB snapshotting
- New commands are buffered during rewrite and appended after

**Configuration**:
```redis
auto-aof-rewrite-percentage 100  # Rewrite when AOF is 100% larger than previous rewrite
auto-aof-rewrite-min-size 64mb    # Only rewrite if AOF is at least 64MB
```

**Pros**:
- Much more durable than RDB (can lose at most 1 second of data with everysec)
- No data loss if fsync is always (every write synced to disk)
- Append-only format is crash-resistant (log format is easy to recover)
- Can be edited if needed (human-readable text format)
- Better for data durability guarantees

**Cons**:
- Larger file size than RDB (every write logged, not just final state)
- Slower recovery (must replay all commands, can be slow for large datasets)
- Higher write overhead (fsync is expensive)
- Uses more disk space (especially without frequent rewriting)
- Slower than RDB for normal operation

**Use Cases**:
- When data durability is critical (can't lose data)
- When you can tolerate slower recovery
- When you need maximum durability guarantees
- Financial systems, inventory management, critical business data

---

#### 4. RDB + AOF (Hybrid)

Combines both persistence mechanisms for maximum durability with reasonable performance.

**Behavior**:
- Redis uses AOF for recovery (most complete data)
- RDB can be used for backups (point-in-time snapshots)
- AOF can include RDB preamble for faster loading (Redis 4.0+)
- On restart, Redis loads RDB preamble from AOF, then replays remaining AOF commands

**Configuration**:
```redis
save 900 1
save 300 10
save 60 10000
appendonly yes
aof-use-rdb-preamble yes  # Include RDB in AOF for faster loading
```

**Pros**:
- Maximum durability (AOF ensures minimal data loss)
- Fast recovery (RDB preamble loads quickly)
- Complete data (AOF has all write operations)
- Best of both worlds (durability + recovery speed)
- Good backups (RDB snapshots for point-in-time recovery)

**Cons**:
- Highest performance overhead (both RDB and AOF I/O)
- Most complex configuration (need to tune both)
- Most disk usage (both RDB and AOF files)
- Highest resource usage (CPU for RDB, disk for AOF)

**Use Cases**:
- When both durability and fast recovery are critical
- Production workloads requiring strong guarantees
- When you can afford the resource overhead
- Mission-critical systems

---

### Redis Data Structure Internals

#### Strings (SDS - Simple Dynamic Strings)

Redis implements its own string type called SDS instead of using C strings:

```c
struct sdshdr {
    int len;        // Length of string (not including null terminator)
    int free;       // Available space (unused bytes)
    char buf[];     // Actual string data (null-terminated)
};
```

**Advantages over C strings**:
- O(1) length calculation (C strings require O(n) strlen)
- Binary-safe (can store null bytes in middle of string)
- Prevents buffer overflow (tracks length, doesn't rely on null terminator)
- Immutable string sharing (interning - same string value shared across keys)
- Efficient concatenation (pre-allocates extra space to reduce reallocations)

**Use cases**: Session tokens, counters, simple key-value storage, binary data storage.

---

#### Lists

Originally implemented as linked lists, now uses quicklist (linked list of ziplists) for better performance.

**Ziplist** (for small lists):
- Contiguous memory allocation
- Stores list elements sequentially
- No pointer overhead (more memory efficient)
- Used when list has few elements and elements are small
- O(N) operations for large lists (must scan entire list)

**Quicklist** (for larger lists):
- Linked list of ziplists
- Each ziplist contains a subset of list elements
- Balances memory efficiency (ziplist) with performance (linked list)
- O(1) push/pop from head or tail
- O(N) for random access (must traverse)

**Configuration**:
```redis
list-max-ziplist-size -2  # Use ziplist for lists with elements < 64 bytes
list-compress-depth 0      # Don't compress quicklist nodes
```

**Use cases**: Message queues, timelines, activity feeds, recent items lists.

---

#### Hashes

Uses one of two implementations based on size:

**Ziplist** (for small hashes):
- Contiguous memory allocation
- Stores field-value pairs sequentially
- Used when hash has few fields and values are small
- More memory efficient (no hash table overhead)

**Hash Table** (for larger hashes):
- Standard hash table with chaining for collision resolution
- Incremental rehashing (gradually resizes hash table to avoid blocking)
- O(1) field access
- Used when hash has many fields or values are large

**Incremental Rehashing**:
- When hash table grows/shrinks, needs rehashing (reallocate and redistribute)
- Redis does this incrementally (not all at once)
- Each operation moves a few entries to new hash table
- Prevents blocking during rehashing
- Maintains two hash tables during transition (old and new)

**Configuration**:
```redis
hash-max-ziplist-entries 512    # Use ziplist for hashes with < 512 fields
hash-max-ziplist-value 64      # Use ziplist when values < 64 bytes
```

**Use cases**: User profiles, product information, session data, object representation.

---

#### Sorted Sets (ZSets)

Uses two data structures together:

**1. Hash Table**:
- Maps element → score
- O(1) score lookup
- O(1) element existence check

**2. Skip List**:
- Maintains elements sorted by score
- O(log N) insertion, deletion, range queries
- Probabilistic data structure (alternative to balanced trees)
- Good cache locality (contiguous memory regions)

**Why Skip List instead of Balanced Tree**:
- Simpler implementation (fewer edge cases)
- Better cache locality (contiguous memory)
- Good performance for range queries
- Easier to implement concurrent operations (though Redis is single-threaded)

**Use cases**: Leaderboards, ranking systems, priority queues, range queries (top N, bottom N).

---

## Memcached: Architecture and Internals

### Overview

Memcached is a free, open-source, high-performance distributed memory object caching system. Originally developed by Brad Fitzpatrick in 2003 to speed up LiveJournal, it defined the concept of the distributed, in-memory key-value store. While newer tools like Redis have introduced complex data structures and persistence, Memcached remains a foundational piece of infrastructure for giants like Facebook, Netflix, and Twitter.

**Key Characteristics**:
- Simple key-value storage only (strings)
- Multi-threaded architecture (unlike Redis)
- Slab allocator for memory management (eliminates fragmentation)
- LRU eviction policy only
- No built-in persistence (pure cache)
- "Dumb server, smart client" philosophy

### The Philosophy: Dumb Servers, Smart Clients

Memcached operates on a "shared-nothing" architecture that fundamentally differs from Redis:

- **No clustering protocol**: Memcached nodes don't know about each other
- **No replication**: No built-in replication (must be handled client-side)
- **No internal communication**: Servers are entirely dumb
- **All distributed logic in clients**: Client libraries handle routing, failover, partitioning

**How it works**:
```
Application → [Client Library] → Hash key → Determine node → Send to specific Memcached server
```

**Benefits of this approach**:
- Extreme decoupling enables infinite scaling (adding 100 nodes adds zero network overhead to existing cluster)
- No cluster state to synchronize (no gossip protocol, no consensus)
- Simple server implementation (easier to maintain, fewer bugs)
- Flexible client implementations (different clients can implement different strategies)

**Trade-offs**:
- More complex client libraries (must handle all distributed logic)
- No built-in high availability (clients must handle failover)
- No built-in replication (clients must replicate if needed)
- Operational complexity moves to clients (harder to debug distributed issues)

### The Slab Allocator

When applications constantly allocate and free random sizes of memory (like a 12-byte string, then a 500-byte JSON object, then a 2MB image), RAM quickly becomes fragmented. This is called memory fragmentation, and it eventually causes systems to crash or aggressively swap to disk.

Memcached solves this using a strict memory management system called the Slab Allocator. Instead of using the operating system's standard malloc and free for every item, Memcached pre-allocates your RAM into highly structured blocks.

#### Memory Hierarchy

**1. Pages**:
- Memcached grabs memory from OS in massive, default 1-Megabyte chunks called Pages
- All memory allocation happens within these pages
- Pages are the unit of allocation from the OS

**2. Slab Classes**:
- Pages are divided into different "Classes" based on size
- Each slab class has a fixed chunk size
- Typical slab classes: 96 bytes, 120 bytes, 152 bytes, 200 bytes, 252 bytes, etc.
- Each slab class manages its own set of pages

**3. Chunks**:
- Inside each Page, memory is sliced into perfectly equal, fixed-size Chunks
- All chunks in a slab class are identical size
- Items are stored in chunks

#### Example

- **Slab Class 1**: 1MB page divided into 96-byte chunks (~10,922 chunks per page)
- **Slab Class 2**: 1MB page divided into 120-byte chunks (~8,738 chunks per page)
- **Slab Class 3**: 1MB page divided into 152-byte chunks (~6,892 chunks per page)

When storing an 80-byte JSON string, Memcached finds the smallest available chunk that fits (96-byte chunk in Slab Class 1).

#### Trade-offs

**Pros**:
- Eliminates memory fragmentation completely (predictable performance forever)
- Fast allocation and deallocation (O(1) operations)
- No compaction needed (fixed-size chunks)
- Predictable memory usage

**Cons**:
- Internal wasted space (80 bytes in 96-byte chunk wastes 16 bytes)
- Memory locked to size classes (if slab class not fully used, memory unavailable for other sizes)
- Pre-allocation makes memory usage appear high immediately (even if little actual data stored)
- Less efficient for variable-sized data

**Configuration**:
```bash
# Memcached startup with 4GB memory
memcached -m 4096

# Slab sizes are automatically calculated based on growth factor
# Default growth factor: 1.25
# Can be customized: memcached -f 1.5
```

### Multi-Threaded Architecture

This is where Memcached fundamentally diverges from Redis. While Redis processes commands using a single thread, Memcached is natively multi-threaded.

#### Architecture

**libevent Integration**:
Memcached utilizes libevent, a highly optimized asynchronous event notification library that handles network I/O efficiently.

**Thread Structure**:
1. **Main Thread**: Single dispatcher thread listens for incoming network connections on the main port
2. **Worker Threads**: When connection is established, dispatcher hands it off to pool of worker threads
3. **Thread Pool**: Typically configured based on CPU cores (e.g., 8 threads for 8-core machine)

**Locking Strategy**:
- Global connection lock (very coarse, only for connection acceptance)
- Per-slab locks (finer granularity for slab operations)
- Item-level locks for specific operations (when needed)
- Lock-free data structures where possible

**Benefits**:
- Multiple threads can read and write simultaneously across multiple CPU cores
- Scales to utilize every single core on multi-core systems
- Achieves throughputs of millions of operations per second on single machine
- No single-threaded bottleneck

**Trade-offs**:
- More complex implementation (concurrency bugs, race conditions)
- Lock contention possible under high concurrency
- Debugging concurrency issues is harder
- More memory overhead (per-thread structures)

**Performance**:
- Single Memcached process on 64-core machine can handle millions of ops/sec
- Linear scaling up to ~16-32 cores (diminishing returns after due to lock contention)
- Outperforms single-threaded Redis for multi-key operations on multi-core machines

### Memcached Eviction

Memcached uses LRU (Least Recently Used) eviction when memory is full. However, because of the Slab Allocator, it doesn't evaluate the entire cache at once.

#### Slab-Specific LRU

**Key insight**: LRU eviction in Memcached is Slab-Specific, not global.

- If Slab Class 1 (tiny chunks) is completely full
- But Slab Class 5 (large chunks) is mostly empty
- Memcached will NOT evict a large chunk to make room for a tiny one
- It will strictly evict the oldest item within Slab Class 1

**Why this design**:
- Prevents cross-slab memory fragmentation
- Maintains slab allocator guarantees
- Simplifies implementation (no cross-slab coordination)

**Trade-off**:
- Can lead to inefficient memory usage (some slabs full, others empty)
- Might evict frequently accessed small items while large items sit idle
- Requires careful capacity planning per slab class

#### LRU Implementation

- Doubly-linked list per slab class
- Head = most recently used, Tail = least recently used
- On access: move item to head
- On eviction: remove from tail
- Per-slab LRU lists (not global LRU)

**Configuration**:
```bash
# LRU can be tuned with these parameters
memcached -o lru_maintainer_thread=1  # Background thread for LRU maintenance
memcached -o maxconns=1024           # Max concurrent connections
```

---

### Memcached vs Redis Comparison

| Feature | Memcached | Redis |
|---------|-----------|-------|
| **Data Types** | Strings only | Strings, Lists, Sets, Hashes, Sorted Sets, Bitmaps, HyperLogLog, Geospatial |
| **Persistence** | None (pure cache) | RDB, AOF, or both (can be used as database) |
| **Threading** | Multi-threaded (scales across cores) | Single-threaded (simpler, no lock contention) |
| **Memory Management** | Slab allocator (no fragmentation) | Custom allocator (can fragment) |
| **Eviction** | LRU only (slab-specific) | LRU, LFU, TTL, volatile-* policies |
| **Replication** | Client-side (no built-in) | Built-in (primary-replica) |
| **Clustering** | Client-side (no built-in) | Built-in (Redis Cluster) |
| **Transactions** | No | Yes (MULTI/EXEC) |
| **Pub/Sub** | No | Yes |
| **Lua Scripting** | No | Yes |
| **Complexity** | Simple server, smart client | More complex server, simpler client |
| **Use Case** | Pure caching, simple key-value | Caching + data structures + messaging + database |

**When to choose Memcached**:
- Simple caching use case (only need key-value storage)
- Multi-core machines (want to utilize all cores)
- Very large cache (need slab allocator's predictable performance)
- Existing Memcached deployment (migration cost too high)
- Want maximum simplicity in server implementation

**When to choose Redis**:
- Need rich data structures (lists, sets, sorted sets, etc.)
- Need persistence (can't afford to lose data)
- Need built-in replication and clustering
- Need transactions or pub/sub
- Want simpler client libraries
- Need more than just caching (messaging, leaderboards, etc.)

---

## Facebook's Memcached Scaling Story

### The Challenge

As Facebook grew from a small social network to a global platform serving billions of users, their caching infrastructure faced unprecedented challenges:

- **Massive Scale**: Billions of cache requests per second at peak
- **Global Distribution**: Data centers across multiple continents
- **High Availability**: 99.99% uptime requirements
- **Low Latency**: Sub-10ms response times globally
- **Complex Workloads**: Social graphs, timelines, user profiles, news feeds

### The Evolution

#### Phase 1: Single Memcached Instance

Initially, Facebook used a single Memcached instance per web server:
- Simple deployment
- Limited capacity
- Single point of failure
- No coordination between instances

**Problem**: As traffic grew, single instances couldn't handle the load. Adding more instances didn't help because each instance had its own copy of data (no sharing).

#### Phase 2: Distributed Memcached with Client Libraries

Facebook developed sophisticated client libraries to:
- Implement consistent hashing to distribute keys across Memcached instances
- Handle failover (retry on different node if one fails)
- Manage connection pooling
- Implement retry logic with exponential backoff

**Problem**: Client libraries became increasingly complex. Every language needed its own implementation. Bugs in client logic caused subtle issues. Changing routing logic required updating all clients.

#### Phase 3: Introduction of Mcrouter

Facebook developed **mcrouter** (pronounced "mick-router"), a memcached protocol router that sits between clients and Memcached servers. This was a fundamental shift from "smart clients" to "smart proxy."

**What is Mcrouter?**
- A memcached protocol router for scaling memcached deployments
- Looks like a memcached server to clients
- Looks like a memcached client to servers
- Handles all traffic to, from, and between cache servers
- Proven at massive scale: handles close to 5 billion requests per second at peak
- Open-sourced in 2014 under BSD license

**Why Mcrouter?**
- Centralized routing logic (no need to update all clients)
- Consistent behavior across all clients
- Rich features not possible in client libraries
- Easier to debug and monitor (single point of observability)
- Can add new features without changing clients

### Mcrouter Architecture

#### Core Design Principles

**Protocol Compatibility**:
- Uses standard open source memcached ASCII protocol
- Any client that can talk memcached can talk to mcrouter
- Can be simply dropped in between clients and memcached boxes
- No client changes needed

**Configuration-Driven**:
- Configuration specified as JSON
- Graph of small routing modules called "route handles"
- Route handles share common interface (route request and return reply)
- Can be composed freely for arbitrarily complex logic
- Easy to adapt to any routing task

**Multi-Threaded**:
- Written mostly in C++ with C++11 features
- Starts one thread per core
- Each thread runs event loop using libevent
- Processes network events asynchronously
- Uses custom fiber library built on boost::context

#### Key Features

**1. Connection Pooling**:
Multiple clients can connect to a single mcrouter instance and share outgoing connections, reducing the number of open connections to memcached instances. This is crucial because each TCP connection consumes resources (file descriptors, memory).

**2. Multiple Hashing Schemes**:
- **furc_hash**: Proven consistent hashing algorithm for distribution across many instances
- **Hostname hashing**: Selects unique replica per client (useful for A/B testing)
- **Other specialized hashes**: For specific applications

**3. Prefix Routing**:
Mcrouter can route keys according to common key prefixes:
- All keys starting with "user:" go to one pool
- "post:" prefix goes to another pool
- Everything else goes to "wildcard" pool

This is a simple way to separate different workloads and data types.

**4. Replicated Pools**:
A replicated pool has the same data on multiple hosts:
- Writes are replicated to all hosts in the pool
- Reads are routed to a single replica chosen separately for each client
- Use cases: Per-host packet limitations where sharded pool can't handle read rate, or for increased availability

**5. Production Traffic Shadowing**:
When testing new cache hardware, route a complete copy of production traffic:
- Shadow test different pool size (re-hashing the key space)
- Shadow only a fraction of the key space
- Vary shadowing settings dynamically at runtime
- No impact on production

**6. Online Reconfiguration**:
Mcrouter monitors configuration files and automatically reloads on changes:
- Loading and parsing done on background thread
- New requests routed according to new configuration as soon as ready
- No extra latency from client's point of view
- Zero-downtime configuration changes

**7. Destination Health Monitoring**:
Mcrouter keeps track of health status of each destination:
- If destination marked unresponsive, fail over to alternate destination automatically
- Distinguish between "soft errors" (data timeouts) and "hard errors" (connection refused)
- Health check requests sent in background
- As soon as health check successful, revert to using original destination
- Completely transparent to client

**8. Cold Cache Warm Up**:
Smooth performance impact of starting brand new empty cache hosts:
- Automatically refill from designated "warm" cache
- Can warm up entire cluster from warm cluster
- Reduces cache miss storms during deployments
- Enables graceful scaling

**9. Reliable Delete Stream**:
In demand-filled look-aside cache, ensure all deletes are eventually delivered for consistency:
- Log delete commands to disk when destination not accessible
- Separate process replays deletes asynchronously
- Original delete command always reported as successful to client
- Transparent to client

**10. Multi-Cluster Support**:
Configuration management for large multi-cluster setups:
- Single config distributed to all clusters
- Based on command line options, mcrouter interprets config based on location
- Simplifies management of global deployments

**11. Quality of Service**:
Mcrouter allows throttling request rates:
- Throttle any type of request (get/set/delete) at any level
- Per-host, per-pool, or per-cluster throttling
- Reject requests over specified limit
- Rate limit requests to slow delivery
- Protects backend systems from overload

**12. Large Values**:
Mcrouter can automatically split/re-stitch large values:
- Handles values that wouldn't normally fit in memcached slab
- Transparent to client
- Automatic reassembly on retrieval

**13. Multi-Level Caches**:
Supports local/remote cache setup:
- Values looked up locally first
- Automatically set in local cache from remote after fetching
- Reduces network round trips for hottest data
- Improves latency for frequently accessed data

### Facebook's Cache Architecture

#### Multi-Tier Setup

Facebook uses a sophisticated multi-tier cache architecture:

**1. Local Cache (L1)**:
- In-process cache on each web server
- Holds hottest data (most frequently accessed)
- Sub-millisecond access (no network round-trip)
- Limited capacity (typically a few MB per server)
- Implemented as simple hash map in application process

**2. Regional Cache (L2)**:
- Mcrouter + Memcached clusters in each data center
- Holds warm data (frequently accessed but not hottest)
- Single-digit millisecond access
- Larger capacity (terabytes per data center)
- Mcrouter handles routing, failover, and replication

**3. Global Cache (L3)**:
- Cross-region replication for critical data
- Eventual consistency across regions
- Higher latency but global availability
- Used for data that must be available globally (user profiles, configuration)

#### Consistency Model

Facebook uses a sophisticated consistency model tailored to different data types:

**Read-after-write**: For user-generated content (posts, comments). Ensures that after updating, subsequent reads return the new value. Implemented through careful routing and invalidation.

**Eventual consistency**: For feed data and recommendations. Accepts that replicas will converge over time but might serve different values during convergence window. Good for social media feeds where slight staleness is acceptable.

**Invalidation-based**: For profile data. Immediately invalidate cache entries when underlying data changes. Combined with CDC (Change Data Capture) for automatic invalidation.

**TTL-based**: For analytics and aggregated data. Uses TTLs with appropriate values based on data volatility.

### Key Learnings from Facebook's Scaling

**1. Smart Routing is Critical**:
The ability to intelligently route cache requests based on:
- Key prefixes (separate workloads)
- Data types (different pools for different data)
- Access patterns (popular vs. unpopular)
- Geographic location (regional caching)
- Capacity constraints (avoid overloading specific nodes)

**2. Observability is Essential**:
Rich metrics and debug capabilities are crucial for operating at scale:
- Per-host, per-pool, per-key statistics
- Request routing visualization (which node would a request go to?)
- Performance profiling (latency, hit ratios, error rates)
- Cache hit/miss ratios by key pattern

**3. Graceful Degradation**:
Systems must handle failures gracefully:
- Automatic failover (no manual intervention)
- Retry logic with exponential backoff
- Circuit breakers to prevent cascading failures
- Degraded mode operation (serve stale data if needed)

**4. Testing with Production Traffic**:
Shadow testing is invaluable for scaling:
- Test new infrastructure with real traffic patterns
- A/B test different configurations
- Capacity planning with realistic data
- No risk to production (shadowed traffic doesn't affect production)

**5. Centralized Routing Logic**:
Moving from "smart clients" to "smart proxy" (mcrouter) was a game-changer:
- Easier to add new features (no client changes needed)
- Consistent behavior across all clients
- Easier to debug (single point of observability)
- Faster iteration (can update proxy without updating all clients)

### Impact and Legacy

Facebook's work on Mcrouter and Memcached scaling has had significant impact:
- Open-sourced in 2014, now used by many companies
- Handles billions of requests per second at Facebook
- Enabled Facebook to scale from millions to billions of users
- Influenced the design of other caching systems
- Demonstrated that "dumb server, smart client" can evolve to "smart proxy" for better operations

The key insight: at massive scale, operational simplicity and observability become as important as raw performance. Mcrouter provided both by centralizing routing logic and providing rich observability features.

---

## Conclusion

Caching is a fundamental technique in system design that can dramatically improve performance, scalability, and user experience. Understanding the fundamentals of caching, distributed caching architectures, various caching strategies, eviction policies, and the internals of popular caching systems like Redis and Memcached is essential for any system designer or backend engineer.

### Key Takeaways

1. **Caching is everywhere**: From CPU caches to CDNs, caching operates at every layer of the stack. Choose the right layer for your use case.

2. **Choose the right strategy**: Cache-Aside, Read-Through, Write-Through, Write-Around, and Write-Back each have their use cases. Match the strategy to your workload characteristics.

3. **Eviction policies matter**: LRU, LFU, and their variants significantly impact cache effectiveness. Start with LRU (or Segmented LRU) and only switch if you have specific workload characteristics.

4. **Distributed caching adds complexity**: Consistent hashing, replication, and consistency must be carefully designed. Understand the trade-offs between consistency and availability (CAP theorem).

5. **Redis and Memcached have different strengths**: Redis offers rich data structures and persistence; Memcached offers simplicity and raw multi-core performance. Choose based on your requirements.

6. **Learn from the giants**: Facebook's scaling story with mcrouter provides valuable lessons about the importance of smart routing, observability, and graceful degradation at scale.

7. **Operational excellence is critical**: Monitoring, alerting, and disaster recovery are as important as the cache itself. Build observability into your caching infrastructure from the start.

### When to Use Caching

**Good candidates for caching**:
- Read-heavy workloads (user profiles, product catalogs)
- Data that changes infrequently (configuration, static content)
- Expensive computations (aggregated analytics, complex queries)
- Data accessed repeatedly (session data, frequently viewed items)

**Poor candidates for caching**:
- Write-heavy workloads with infrequent reads (logs, audit trails)
- Data that changes constantly (real-time stock prices, live scores)
- Real-time data where freshness is critical (financial transactions)
- Very large individual objects (videos, large files)
- Data with strict consistency requirements (inventory management)

### The Future of Caching

The future of caching includes:
- **AI-driven cache management**: Machine learning to predict access patterns and optimize cache placement
- **Edge computing**: Caching closer to users at the network edge
- **Persistent memory**: Non-volatile RAM blurring the line between memory and disk
- **Serverless caching**: Managed caching services with automatic scaling
- **Multi-model caching**: Caching that understands data semantics beyond simple key-value

Caching will continue to be a critical component of system design as applications grow in scale and complexity. Mastering caching concepts and practices is an investment that pays dividends across all aspects of system architecture and development.
