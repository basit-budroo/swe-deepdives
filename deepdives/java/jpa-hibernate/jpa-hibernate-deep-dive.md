# JPA & Hibernate Deep Dive — Complete Guide

## Table of Contents
0. [ORM Fundamentals & Theoretical Background](#0-orm-fundamentals--theoretical-background)
1. [JPA Specification vs Hibernate Implementation](#1-jpa-specification-vs-hibernate-implementation)
2. [Core Architecture Components](#2-core-architecture-components)
3. [Entity Lifecycle States](#3-entity-lifecycle-states)
4. [The @Id Annotation](#4-the-id-annotation)
5. [EntityManager.persist() Internals](#5-entitymanagerpersist-internals)
6. [Lazy Loading Mechanisms](#6-lazy-loading-mechanisms)
7. [Bytecode Enhancement Explained](#7-bytecode-enhancement-explained)
8. [Limitations of Proxy-Based Lazy Loading](#8-limitations-of-proxy-based-lazy-loading)
9. [Transaction Management](#9-transaction-management)
10. [Inheritance Mapping Strategies](#10-inheritance-mapping-strategies)
11. [Locking Mechanisms](#11-locking-mechanisms)
12. [Cascade Operations](#12-cascade-operations)
13. [Query Optimization](#13-query-optimization)
14. [Second-Level Cache](#14-second-level-cache)
15. [Batch Processing](#15-batch-processing)
16. [Glossary of Key Terms](#16-glossary-of-key-terms)

---

## 0. ORM Fundamentals & Theoretical Background

### 0.1 What is Object-Relational Mapping (ORM)?

**Object-Relational Mapping (ORM)** is a programming technique that converts data between incompatible type systems in object-oriented programming languages and relational databases. It creates a "virtual object database" that can be used from within the programming language.

**The Core Problem: The Object-Relational Impedance Mismatch**

The fundamental challenge ORM addresses is the conceptual mismatch between:

| Object-Oriented Paradigm | Relational Paradigm |
|-------------------------|---------------------|
| **Granularity**: Objects are fine-grained with behavior | **Granularity**: Tables are coarse-grained, data-only |
| **Identity**: `a == b` (memory address) or `a.equals(b)` (logical equality) | **Identity**: Primary key (single or composite) |
| **Inheritance**: Class hierarchies with polymorphism | **Inheritance**: No native support (simulated via foreign keys) |
| **Associations**: Object references (bidirectional, navigable) | **Associations**: Foreign keys (unidirectional, require joins) |
| **Navigation**: Direct object graph traversal (`order.getCustomer().getName()`) | **Navigation**: SQL queries with JOINs |
| **State**: Objects have state and behavior | **State**: Tables hold data only |
| **Encapsulation**: Private fields with public methods | **Encapsulation**: No concept (all columns are accessible) |

**ORM as the Solution Layer**

```
┌─────────────────────────────────────────────────────────────────┐
│                     Application Layer (Java)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Domain Objects (Entities)                                │  │
│  │  - Order, Customer, Product                                 │  │
│  │  - Rich with behavior, relationships, inheritance          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ORM Layer (Hibernate/JPA)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Mapping Metadata (@Entity, @Table, @OneToMany, etc.)    │  │
│  │  - Defines how objects map to tables                      │  │
│  │  - Handles state synchronization                          │  │
│  │  - Manages identity, relationships, inheritance            │  │
│  │  - Provides caching, lazy loading, dirty checking         │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Database Layer (Relational)                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Tables, Columns, Foreign Keys, Constraints               │  │
│  │  - Normalized schema                                      │  │
│  │  - Set theory based (relations)                           │  │
│  │  - Declarative integrity constraints                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 0.2 Key ORM Concepts & Terminology

#### Entity

An **entity** is a persistent object that has its own identity and lifecycle. Unlike transient objects (POJOs), entities are associated with a database row and can be retrieved, updated, or deleted.

**Theoretical Basis**: Entities represent the **domain model** — the conceptual layer of your application that captures business rules and relationships independently of persistence concerns.

**Characteristics:**
- **Identity**: Distinguished by a unique identifier (primary key)
- **Persistence**: Can exist beyond the lifetime of the JVM (stored in database)
- **Lifecycle**: Transitions between states (Transient, Managed, Detached, Removed)
- **Equality**: Based on identity (primary key), not memory address

#### Value Object

A **value object** is an immutable object without identity that is defined by its attributes. Unlike entities, two value objects with the same attributes are considered equal.

**Example:**
```java
// Entity (has identity)
@Entity
public class Customer {
    @Id private Long id;
    private String name;
    private Address address;  // Value object
}

// Value object (no identity, defined by attributes)
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;
    
    // No @Id - equality based on all fields
    @Override
    public boolean equals(Object o) {
        // Compare all fields
    }
}
```

#### Aggregation

**Aggregation** represents a "whole-part" relationship where the child cannot exist independently of the parent. When the parent is deleted, the child is also deleted (cascade delete).

**Theoretical Significance**: Aggregation models **composition** in UML — a strong form of association where the lifecycle of the part is bound to the whole.

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items;
    // OrderItems cannot exist without an Order
}
```

#### Association

An **association** represents a relationship between entities where both can exist independently.

**Theoretical Significance**: Associations model **loose coupling** between entities. They can be unidirectional or bidirectional.

```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
    // Customer exists independently of Order
}
```

### 0.3 Theoretical Foundations of Hibernate Architecture

#### Unit of Work Pattern

The **Unit of Work** pattern maintains a list of objects affected by a business transaction and coordinates the writing out of changes.

**In Hibernate**: The `Session` (or `EntityManager`) implements the Unit of Work pattern through the **Persistence Context**.

**Theoretical Benefits:**
1. **Atomicity**: All changes are committed as a single unit
2. **Consistency**: In-memory state matches database state
3. **Identity Map**: Guarantees that each entity is loaded only once per session
4. **Lazy Loading**: Defers data fetching until actually needed
5. **Dirty Checking**: Automatically detects which objects changed

#### Identity Map Pattern

The **Identity Map** ensures that each object is loaded only once per session, maintaining a map of already-loaded objects keyed by their identity.

**Theoretical Significance**: This pattern prevents:
- **Duplicate objects**: Loading the same database row twice creates two Java objects
- **Inconsistent state**: Two objects representing the same row could diverge
- **Memory waste**: Loading the same data multiple times

**Implementation in Hibernate:**
```java
// The Persistence Context IS an Identity Map
Map<EntityKey, Entity> identityMap = new HashMap<>();

// When loading:
EntityKey key = new EntityKey(Customer.class, 42L);
if (identityMap.containsKey(key)) {
    return identityMap.get(key);  // Return existing instance
} else {
    Customer customer = loadFromDB(42L);
    identityMap.put(key, customer);
    return customer;
}
```

#### Lazy Loading Pattern

**Lazy Loading** defers the initialization of an object until the point at which it is needed.

**Theoretical Basis**: This is an application of the **Proxy pattern** and the **Virtual Proxy** pattern specifically. It implements the principle of **separation of concerns** — the concern of when to load data is separated from the concern of how to use it.

**Benefits:**
1. **Performance**: Only load data that's actually used
2. **Memory**: Reduce memory footprint by not loading unnecessary data
3. **Scalability**: Reduce database load

**Trade-offs:**
1. **Complexity**: Requires session management
2. **N+1 Problem**: Can trigger excessive queries if not managed
3. **LazyInitializationException**: Access outside session context fails

#### Dirty Checking Pattern

**Dirty Checking** automatically detects which objects have changed since they were loaded, so only modified objects need to be written to the database.

**Theoretical Implementation**: Hibernate maintains a **snapshot** of each entity's state when loaded. At flush time, it compares the current state against the snapshot.

**Theoretical Significance**: This implements the **Observer pattern** implicitly — the Persistence Context "observes" entity state changes without requiring explicit notification.

#### Write-Behind Pattern

**Write-Behind** defers database writes until the end of a transaction, allowing multiple changes to be batched together.

**Theoretical Benefits:**
1. **Performance**: Reduces database round-trips
2. **Batching**: Enables JDBC batch operations
3. **Ordering**: Ensures proper execution order (INSERTs before UPDATEs before DELETEs)

### 0.4 Theoretical Basis for Caching

#### First-Level Cache (Session Cache)

**Theoretical Foundation**: The first-level cache is an implementation of the **Identity Map** pattern scoped to a single Unit of Work (Session).

**Properties:**
- **Scope**: Single Session/EntityManager
- **Lifetime**: Transaction/request boundary
- **Mandatory**: Cannot be disabled
- **Content**: Full entity objects (hydrated)

#### Second-Level Cache (SessionFactory Cache)

**Theoretical Foundation**: The second-level cache implements the **Cache-Aside pattern** at the application level, shared across all sessions.

**Properties:**
- **Scope**: Application-wide (SessionFactory)
- **Lifetime**: Application lifetime
- **Optional**: Must be explicitly enabled
- **Content**: Dehydrated state (serialized form)

**Theoretical Trade-offs:**
- **Consistency**: Stale data vs performance
- **Concurrency**: Locking strategies (READ_WRITE vs NONSTRICT_READ_WRITE)
- **Memory**: Cache size vs hit rate

### 0.5 Theoretical Basis for Transaction Management

#### ACID Properties

Transactions must satisfy the **ACID** properties:

| Property | Theoretical Definition | Hibernate Implementation |
|----------|----------------------|-------------------------|
| **Atomicity** | All operations in a transaction succeed or fail as a unit | Transaction.commit() or rollback() |
| **Consistency** | Database transitions from one valid state to another | Constraint enforcement, cascading |
| **Isolation** | Concurrent transactions don't interfere with each other | Isolation levels (READ_COMMITTED, etc.) |
| **Durability** | Committed transactions survive system failures | Database write-ahead logs |

#### Isolation Levels

**Theoretical Foundation**: Isolation levels represent different trade-offs between **consistency** and **concurrency**. They implement different degrees of the **serializability** concept — the theoretical ideal where transactions execute sequentially.

**From Theory to Practice:**
- **SERIALIZABLE**: Full serializability (highest consistency, lowest concurrency)
- **REPEATABLE_READ**: Prevents non-repeatable reads (medium consistency)
- **READ_COMMITTED**: Prevents dirty reads (default in many DBs)
- **READ_UNCOMMITTED**: No protection (highest concurrency, lowest consistency)

---

## 1. JPA Specification vs Hibernate Implementation

**JPA (Jakarta Persistence API)** is a *specification* — a set of interfaces and annotations. **Hibernate** is the most widely-used *implementation*.

| JPA Specification (Interface) | Hibernate Implementation (Class) |
|-----------------------------|----------------------------------|
| `EntityManagerFactory` | `SessionFactoryImpl` |
| `EntityManager` | `SessionImpl` |
| `EntityTransaction` | `TransactionImpl` |
| `Query` | `QueryImpl` |
| `Persistence Context` (concept) | First-Level Cache inside `Session` |

```java
// You can always unwrap to get the Hibernate-native object
Session session = entityManager.unwrap(Session.class);
```

---

## 2. Core Architecture Components

### 2.1 High-Level Architecture

```
Application Code
      │
      ▼
╔═══════════════════════════════════════════════╗
║     JPA Specification (jakarta.persistence.*) ║
║                                               ║
║  EntityManagerFactory, EntityManager          ║
║  @Entity, @Table, @Id, etc.                  ║
╚═══════════════════════════════════════════════╝
      │ implements
      ▼
╔═══════════════════════════════════════════════╗
║   Hibernate ORM                               ║
║                                               ║
║  SessionFactory, Session (+ 1st-Level Cache)  ║
║  Transaction, Type System, Dirty Checking       ║
║  Query Engine (HQL/Criteria)                    ║
╚═══════════════════════════════════════════════╝
      │
      ▼
   JDBC / DataSource
      │
      ▼
   Relational Database
```

### 2.2 EntityManagerFactory / SessionFactory

- **One per application** (per persistence unit)
- **Thread-safe**, heavyweight, created at startup
- Reads configuration, builds metadata (entity mappings, SQL dialects, connection pool)
- Caches `EntityPersister` (one per entity class) and `CollectionPersister`

### 2.3 EntityManager / Session

- **One per unit of work** (typically per HTTP request or transaction)
- **NOT thread-safe** — must not be shared
- Wraps a JDBC `Connection` (lazily acquired)
- Contains the **Persistence Context**

### 2.4 Persistence Context (First-Level Cache)

The Persistence Context is a **Map** of managed entity instances, keyed by `(entity type, primary key)`:

```
┌─────────── Persistence Context ────────────────┐
│                                                  │
│  Identity Map (entityMap):                        │
│  ┌────────────────────┬────────────────────────┐  │
│  │ EntityKey           │ Entity Instance       │  │
│  │ (User, id=42)      │ → User@0x7f3a         │  │
│  │ (Order, id=100)    │ → Order@0x8b2c        │  │
│  └────────────────────┴────────────────────────┘  │
│                                                  │
│  Snapshot Map:                                   │
│  ┌────────────────────┬────────────────────────┐  │
│  │ EntityKey           │ Field values at load  │  │
│  │ (User, id=42)      │ → {"Alice", 30, ...}  │  │
│  └────────────────────┴────────────────────────┘  │
│                                                  │
│  Action Queue:                                   │
│  ┌──────────────────────────────────────────────┐│
│  │ INSERT User(id=3, ...)                       ││
│  │ UPDATE Order SET status='SHIPPED' WHERE id=  ││
│  │ DELETE Invoice WHERE id=99                   ││
│  └──────────────────────────────────────────────┘│
└──────────────────────────────────────────────────┘
```

**Three critical purposes:**
1. **Identity Map** — guarantees `a == b` for same ID within a Session
2. **Dirty Checking** — compares current values vs snapshot at flush
3. **Write-Behind** — SQL is queued, not executed immediately

### 2.5 First-Level Cache vs Second-Level Cache

| Property | First-Level Cache (L1) | Second-Level Cache (L2) |
|----------|------------------------|-------------------------|
| **Scope** | Single `Session` | Shared across all sessions |
| **Lifetime** | Transaction / request | Application lifetime |
| **Disableable?** | No — always on | Yes — opt-in per entity |
| **Stores** | Actual Java objects | Dehydrated state |
| **Concurrency** | Single-threaded | Thread-safe (EhCache, etc.) |

---

## 3. Entity Lifecycle States

### State Diagram

```
                    new Entity()
                      │
                      ▼
               ┌─────────────┐
               │  TRANSIENT   │
               │              │
               │ • No ID      │
               │ • No PC      │
               │ • No DB row  │
               └──────┬──────┘
                      │ em.persist()
                      ▼
               ┌─────────────┐◄──── em.find() / JPQL query
               │   MANAGED    │◄──── em.merge() (returns managed)
               │              │
               │ • Has ID     │──── em.detach() ──────────────┐
               │ • In PC      │                               │
               │ • Dirty-     │                               ▼
               │   checked    │                        ┌─────────────┐
               └──────┬──────┘                        │  DETACHED    │
                      │                                │              │
                      │ em.remove()                    │ • Has ID     │
                      │  ┌──────────────────────┐      │ • NOT in PC  │
                      │  │ 1. Mark DELETED in PC │      │ • No dirty   │
                      │  │ 2. Queue DELETE       │      │   checking   │
                      │  │ 3. Cascade if needed  │      │ • Lazy =     │
                      │  └──────────────────────┘      │   exception  │
                      ▼                                └──────┬──────┘
               ┌─────────────┐                               │
               │   REMOVED    │         em.merge() ───────────┘
               │              │         (creates managed copy
               │ • Still in PC│          in PC from detached)
               │ • DELETE     │
               │   scheduled  │
               └──────┬──────┘
                      │ flush + commit
                      ▼
               ┌─────────────┐
               │  TRANSIENT   │  (or GC'd)
               └─────────────┘
```

### 3.1 TRANSIENT State

An entity is **Transient** when created with `new` but has no association with any Persistence Context.

```java
User user = new User();
user.setName("Alice");
// user is TRANSIENT here
```

**Characteristics:**
- No row in the database
- Persistence Context doesn't know about it
- `@Id` field is `null` or not yet generated
- Garbage-collectible — if you drop the reference, it's gone
- No dirty checking

### 3.2 MANAGED State

An entity is **Managed** when associated with an open Persistence Context.

**How an entity becomes managed:**

| Operation | From State | What Happens |
|-----------|------------|--------------|
| `em.persist(entity)` | Transient → Managed | Entity added to PC |
| `em.find(Class, id)` | (loads from DB) → Managed | Entity loaded and placed in PC |
| `em.merge(entity)` | Detached → *(returns new)* Managed | Copy created in PC |
| JPQL/Criteria query | (loads from DB) → Managed | Result entities placed in PC |

**Key behaviors:**
- **Identity guarantee**: `em.find(X.class, 42) == em.find(X.class, 42)` — same Java object reference
- **Automatic dirty checking**: At flush time, Hibernate compares current field values to the snapshot
- **Write-behind**: No SQL is executed immediately on `setX()` calls; SQL is batched until flush

### 3.3 DETACHED State

An entity is **Detached** when it was previously managed but is no longer associated with any open Persistence Context.

**How an entity becomes detached:**

| Trigger | What Happens |
|---------|--------------|
| `em.detach(entity)` | Single entity removed from PC |
| `em.clear()` | All entities removed from PC |
| `em.close()` | PC destroyed, all entities detached |
| Serialization | Object leaves JVM context |

**Characteristics:**
- Still has a populated `@Id`
- Persistence Context no longer holds a reference
- No dirty checking — field changes are invisible to Hibernate
- Lazy-loaded collections throw `LazyInitializationException` if accessed

### 3.4 REMOVED State

An entity is **Removed** when it is managed AND scheduled for deletion at the next flush.

```java
User user = em.find(User.class, 42L);  // MANAGED
em.remove(user);  // REMOVED — still in PC, but flagged for deletion
```

**Characteristics:**
- Still in the Persistence Context (identity map) but marked with `DELETED` status
- An `EntityDeleteAction` is queued in the action queue
- At flush: `DELETE FROM table WHERE id = 42` is executed
- After flush + commit: the entity becomes **Transient** (or eligible for GC)

---

## 4. The @Id Annotation

### 4.1 Purpose

`@Id` marks a field as the **primary key** of the entity — the unique identifier that maps to the `PRIMARY KEY` column in the database table.

Hibernate uses it internally for almost everything:

```java
// Pseudocode of what Hibernate does internally
EntityKey key = new EntityKey(entityId, entityPersister);
// This key is used for:
//   - L1 cache (identity map) lookup
//   - Snapshot map lookup
//   - Guaranteeing a == b for same ID within a Session
```

### 4.2 ID Generation Strategies

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

| Strategy | How It Works | When ID Is Assigned | Trade-off |
|----------|--------------|---------------------|-----------|
| `IDENTITY` | DB auto-increment (`SERIAL`, `AUTO_INCREMENT`) | **Immediately at `persist()`** — forces instant INSERT | Breaks JDBC batching |
| `SEQUENCE` | DB sequence (`nextval`) | At `persist()` — only a `SELECT nextval()`, INSERT deferred to flush | Best for batching |
| `TABLE` | Simulates sequence via a table | At `persist()` | Portable but slow (row lock) |
| `AUTO` | Hibernate picks one of the above | Depends on chosen strategy | Least predictable |

### 4.3 The IDENTITY Problem

With `IDENTITY`, Hibernate **must execute the INSERT immediately** on `persist()` because the DB generates the ID and Hibernate needs it right away to create the `EntityKey`. This breaks write-behind and JDBC batch inserts.

With `SEQUENCE`, Hibernate calls `SELECT nextval('my_seq')` to get the ID *without inserting*, then defers the actual INSERT to flush.

### 4.4 Application-Assigned IDs

When there's **no `@GeneratedValue`** annotation:

```java
@Id
@Column(name = "PK", nullable = false, length = 19)
private Long pk;
```

Hibernate treats this as `Assigned` strategy — it expects your code (or a framework) to set the `pk` value **before** `persist()` is called. If `pk` is `null` at persist time, Hibernate throws:

```
org.hibernate.id.IdentifierGenerationException: 
  ids for this class must be manually assigned before calling save()
```

---

## 5. EntityManager.persist() Internals

### 5.1 High-Level Flow

```
em.persist(entity)
  │
  ├─ 1. State validation
  ├─ 2. Event listeners (@PrePersist)
  ├─ 3. ID generation
  ├─ 4. Entity enters Persistence Context
  ├─ 5. Snapshot captured
  ├─ 6. INSERT action queued (write-behind)
  └─ 7. SQL executed later at flush
```

### 5.2 Step 1: State Validation

Hibernate checks the entity's current state:

| Current State | Result |
|---------------|--------|
| **Transient** (new) | ✅ Proceeds normally |
| **Managed** | Ignored — already tracked, no-op |
| **Detached** | ❌ Throws `PersistentObjectException` |
| **Removed** | Transitions back to Managed, cancels pending DELETE |

### 5.3 Step 2: Event Listeners

Before anything is written, lifecycle callbacks fire:

```
1. @PrePersist methods on the entity
2. @EntityListeners classes (PrePersist handlers)
```

Typical use: setting audit fields (`createdBy`, `createdTimestamp`), generating business keys, or assigning application-managed IDs.

### 5.4 Step 3: ID Generation

**SEQUENCE (preferred):**
```
→ SELECT nextval('my_sequence')        -- just gets the ID
→ entity.setId(generatedValue)
→ INSERT deferred to flush              -- write-behind preserved ✅
```

**IDENTITY (auto-increment):**
```
→ INSERT INTO my_table (...) VALUES (...) -- executed IMMEDIATELY
→ entity.setId(JDBC getGeneratedKeys())
→ Write-behind BROKEN for this entity  -- can't batch inserts ❌
```

**ASSIGNED (no @GeneratedValue):**
```
→ Hibernate reads existing entity.getId()
→ If null → IdentifierGenerationException
→ INSERT deferred to flush
```

### 5.5 Step 4: Entity Enters the Persistence Context

With the ID now available, Hibernate builds the cache key and adds the entity:

```java
// Pseudocode inside Hibernate's SessionImpl
EntityKey key = new EntityKey(id, entityPersister);

// Identity Map — stores the actual Java object
persistenceContext.addEntity(key, entity);

// Entity status set to MANAGED
entityEntry = new EntityEntry(
    Status.MANAGED,
    loadedState,    // null for new entities (no DB state yet)
    id,
    version,
    LockMode.WRITE,
    existsInDatabase: false,  // not yet flushed
    entityPersister,
    disableVersionIncrement: false
);
persistenceContext.addEntry(entity, entityEntry);
```

### 5.6 Step 5: Snapshot Captured

Hibernate takes a **deep copy** of all persistent field values:

```java
Object[] currentState = entityPersister.getPropertyValues(entity);
// e.g., ["Alice", 30, "alice@mail.com", ...]
```

This snapshot is stored in the `EntityEntry` and used later during **dirty checking**.

### 5.7 Step 6: INSERT Action Queued (Write-Behind)

Hibernate does **not** execute SQL immediately (except for IDENTITY strategy). Instead, it queues an action:

```java
ActionQueue actionQueue = session.getActionQueue();

EntityInsertAction insertAction = new EntityInsertAction(
    id, 
    currentState,    // field values to insert
    entity, 
    version, 
    entityPersister, 
    isVersionIncrementDisabled, 
    session
);

actionQueue.addAction(insertAction);
```

The **ActionQueue** maintains separate ordered lists:

```
ActionQueue
├── insertions          ← EntityInsertAction
├── updates             ← EntityUpdateAction
├── collectionRemovals
├── collectionUpdates
├── collectionCreations
└── deletions           ← EntityDeleteAction
```

### 5.8 Step 7: SQL Executed at Flush

Flush happens at:
1. **`em.flush()`** — explicit
2. **Transaction commit** — automatic
3. **Before a query** — auto-flush if query touches a table with pending changes

**Flush process:**

```
session.flush()
  │
  ├─ 1. Dirty checking (for already-loaded entities)
  │     → Compare current field values vs snapshots
  │     → Generate EntityUpdateAction for changed entities
  │
  ├─ 2. Execute actions IN THIS ORDER:
  │     a. OrphanRemovalAction
  │     b. EntityInsertAction    ← your persist()'d entity INSERT
  │     c. EntityUpdateAction
  │     d. CollectionRemoveAction
  │     e. CollectionUpdateAction
  │     f. CollectionRecreateAction
  │     g. EntityDeleteAction
  │
  └─ 3. After execution, EntityEntry updated:
        existsInDatabase = true
```

**SQL Generation:**

```java
// Pre-compiled at startup:
String insertSQL = "INSERT INTO users (id, name, age, email, version) VALUES (?, ?, ?, ?, ?)";

// At flush, Hibernate binds parameters:
PreparedStatement ps = connection.prepareStatement(insertSQL);
ps.setLong(1, id);          // PK
ps.setString(2, "Alice");   // name
ps.setInt(3, 30);           // age
ps.setString(4, "a@b.com"); // email
ps.setLong(5, 0L);          // version (@Version starts at 0)
ps.addBatch();              // if JDBC batching enabled
```

### 5.9 Key Insight: Write-Behind Behavior

Because of write-behind, the INSERT uses the **latest field values** at flush time, not the values at `persist()` time:

```java
Order order = new Order();
order.setStatus("PENDING");

em.persist(order);           // INSERT queued

order.setStatus("CONFIRMED"); // Change after persist

em.flush();
// INSERT will contain "CONFIRMED", not "PENDING"
// Because write-behind captures final state
```

---

## 6. Lazy Loading Mechanisms

There are two mechanisms:

```
Lazy Loading
├── 1. Proxy-based     (for @ManyToOne, @OneToOne, em.getReference())
└── 2. Bytecode-based  (for basic fields, @OneToOne owned side)
```

### 6.1 Proxy-Based Lazy Loading

When Hibernate loads an entity that has a `@ManyToOne(fetch = LAZY)` association, it does **not** query the related table. Instead, it creates a **proxy object**.

```java
@Entity
public class Order {
    @Id
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "CUSTOMER_ID")
    private Customer customer;  // ← this will be a PROXY, not a real Customer
}
```

At startup, Hibernate generates a **subclass** of `Customer` at runtime:

```
Customer                     (your real class)
    └── Customer$HibernateProxy$xyz  (generated at runtime)
            │
            ├── Holds: session reference
            ├── Holds: entity ID (the FK value)
            ├── Holds: initialized = false
            ├── Holds: LazyInitializer (the interceptor)
            └── Overrides: ALL non-final methods
```

**What the proxy does:**

```java
// Hibernate generates something like this at runtime:
public class Customer$HibernateProxy$abc extends Customer implements HibernateProxy {
    private LazyInitializer handler;

    @Override
    public String getName() {
        if (!handler.isInitialized()) {
            handler.initialize();  // ← triggers SELECT
        }
        return handler.getTarget().getName();  // ← delegates to real entity
    }

    @Override
    public Long getId() {
        return handler.getIdentifier();  // no SQL needed
    }
}
```

### 6.2 The Initialization Flow

```
order.getCustomer().getName()
       │                │
       │                └── getName() on the PROXY
       │                    │
       │                    ├── handler.isInitialized()? → NO
       │                    │
       │                    ├── handler.initialize()
       │                    │     │
       │                    │     ├── Check: is session still open?
       │                    │     │   → NO → throw LazyInitializationException ❌
       │                    │     │   → YES → continue
       │                    │     │
       │                    │     ├── Check L1 cache (Persistence Context)
       │                    │     │   EntityKey(Customer, 42)
       │                    │     │   → HIT → use cached entity
       │                    │     │   → MISS → continue
       │                    │     │
       │                    │     ├── Check L2 cache (if enabled)
       │                    │     │   → HIT → hydrate, store in L1
       │                    │     │   → MISS → continue
       │                    │     │
       │                    │     ├── Execute SQL:
       │                    │     │   SELECT * FROM customer WHERE id = 42
       │                    │     │
       │                    │     ├── Hydrate Customer entity from ResultSet
       │                    │     ├── Store in Persistence Context (L1)
       │                    │     ├── handler.target = loadedCustomer
       │                    │     └── handler.initialized = true
       │                    │
       │                    └── return handler.target.getName()
       │
       └── Returns the proxy (which IS-A Customer)
```

### 6.3 LazyInitializationException

```
org.hibernate.LazyInitializationException: 
  could not initialize proxy [Customer#42] - no Session
```

**When it happens:**

```java
@Transactional
public Order getOrder(Long id) {
    return em.find(Order.class, id);  // Session closes at end of @Transactional
}

// Later, OUTSIDE the transaction:
order.getCustomer().getName();  // 💥 LazyInitializationException!
```

The proxy still holds a reference to the **closed session**. When initialization is triggered, it can't execute SQL.

**Solutions:**

| Approach | How | Trade-off |
|----------|-----|-----------|
| **Fetch join in query** | `JOIN FETCH o.customer` | Best — one SQL, explicit |
| **`@EntityGraph`** | `@EntityGraph(attributePaths = "customer")` | Clean, declarative |
| **Initialize before closing** | `Hibernate.initialize(order.getCustomer())` | Manual, controlled |
| **DTO projection** | `SELECT new OrderDTO(...)` | No entities, no proxies |
| **Open Session in View** | `spring.jpa.open-in-view=true` | Session stays open through HTTP response |
| **`FetchType.EAGER`** | `@ManyToOne(fetch = EAGER)` | ❌ Worst — always loads |

### 6.4 Collection Lazy Loading

`@OneToMany` and `@ManyToMany` use **persistent collections**:

```java
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
private List<OrderItem> items;
```

Hibernate replaces your `ArrayList` with its own **PersistentBag**:

```
order.items = PersistentBag {
    initialized: false
    session: SessionImpl@xyz
    owner: Order@abc
    role: "Order.items"
    list: null              ← no data loaded yet
}
```

Any method call on the collection triggers loading:

```java
order.getItems().size();     // SELECT * FROM order_item WHERE order_id = ?
order.getItems().isEmpty();  // Same — full collection loaded
order.getItems().get(0);     // Same — full collection loaded
```

**Extra-lazy:** With `@LazyCollection(LazyCollectionOption.EXTRA)`:

```java
order.getItems().size();     // SELECT COUNT(*) FROM order_item WHERE order_id = ?
                             // Does NOT load all items
```

---

## 7. Bytecode Enhancement Explained

### 7.1 Why It Exists

Proxy-based lazy loading has limitations:

1. **Can't lazily load basic fields** — `@Basic(fetch = LAZY)` on a `String` or `byte[]` is ignored
2. **Can't proxy the owned side of `@OneToOne`** — if FK is in the *other* table, Hibernate must query to know if value is `null` or not
3. **Can't proxy `final` classes or intercept `final` methods**
4. **Dirty checking is expensive** — without enhancement, Hibernate compares every field of every managed entity against its snapshot

Bytecode enhancement solves all of these by **rewriting the entity class itself**.

### 7.2 When Does Enhancement Happen?

**Build-Time Enhancement (Recommended):**

A Gradle or Maven plugin modifies `.class` files after compilation:

```groovy
// Gradle
plugins {
    id 'org.hibernate.orm' version '5.6.x'
}

hibernate {
    enhance {
        enableLazyInitialization = true
        enableDirtyTracking = true
        enableAssociationManagement = true
    }
}
```

```
javac                        Hibernate Plugin
  │                              │
  ▼                              ▼
Customer.java ──► Customer.class ──► Customer.class (enhanced)
                  (original)         (rewritten bytecode)
```

### 7.3 What Enhancement Actually Does

#### Original Entity

```java
@Entity
public class Customer {
    @Id
    private Long id;

    @Basic
    private String name;

    @Basic(fetch = FetchType.LAZY)
    private byte[] profilePhoto;    // large blob, want to load lazily

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public byte[] getProfilePhoto() {
        return profilePhoto;
    }
}
```

#### After Enhancement

```java
@Entity
public class Customer 
    implements PersistentAttributeInterceptable,   // ← ADDED
               SelfDirtinessTracker,               // ← ADDED
               ManagedEntity {                     // ← ADDED

    @Id
    private Long id;

    private String name;

    @Basic(fetch = FetchType.LAZY)
    private byte[] profilePhoto;

    // ══════════════════════════════════════════
    // ADDED FIELDS
    // ══════════════════════════════════════════

    @Transient
    private PersistentAttributeInterceptor $$_hibernate_attributeInterceptor;

    @Transient
    private Set<String> $$_hibernate_tracker_dirtyAttributes;
    //  tracks which fields have been modified

    // ══════════════════════════════════════════
    // REWRITTEN GETTER — read interception
    // ══════════════════════════════════════════

    public String getName() {
        return $$_hibernate_read_name();
    }

    private String $$_hibernate_read_name() {
        if ($$_hibernate_attributeInterceptor != null) {
            name = (String) $$_hibernate_attributeInterceptor
                .readObject(this, "name", name);
        }
        return name;
    }

    // ══════════════════════════════════════════
    // REWRITTEN SETTER — write interception
    // ══════════════════════════════════════════

    public void setName(String val) {
        $$_hibernate_write_name(val);
    }

    private void $$_hibernate_write_name(String val) {
        if ($$_hibernate_attributeInterceptor != null) {
            val = (String) $$_hibernate_attributeInterceptor
                .writeObject(this, "name", name, val);
        }

        // Dirty tracking: record that "name" changed
        if ($$_hibernate_tracker_dirtyAttributes != null) {
            if (!Objects.equals(name, val)) {
                $$_hibernate_tracker_dirtyAttributes.add("name");
            }
        }

        name = val;  // actual field assignment
    }

    // ══════════════════════════════════════════
    // LAZY FIELD — profilePhoto (blob)
    // ══════════════════════════════════════════

    public byte[] getProfilePhoto() {
        return $$_hibernate_read_profilePhoto();
    }

    private byte[] $$_hibernate_read_profilePhoto() {
        if ($$_hibernate_attributeInterceptor != null) {
            // THIS is where lazy loading happens for basic fields
            profilePhoto = (byte[]) $$_hibernate_attributeInterceptor
                .readObject(this, "profilePhoto", profilePhoto);
            // If profilePhoto is uninitialized (null marker),
            // the interceptor executes:
            //   SELECT profile_photo FROM customer WHERE id = ?
            // and sets the field value
        }
        return profilePhoto;
    }

    // ══════════════════════════════════════════
    // INTERFACE METHODS (added by enhancement)
    // ══════════════════════════════════════════

    public PersistentAttributeInterceptor $$_hibernate_getInterceptor() {
        return $$_hibernate_attributeInterceptor;
    }

    public void $$_hibernate_setInterceptor(PersistentAttributeInterceptor i) {
        this.$$_hibernate_attributeInterceptor = i;
    }

    // SelfDirtinessTracker methods
    public Set<String> $$_hibernate_getDirtyAttributes() {
        return $$_hibernate_tracker_dirtyAttributes;
    }

    public boolean $$_hibernate_hasDirtyAttributes() {
        return $$_hibernate_tracker_dirtyAttributes != null 
            && !$$_hibernate_tracker_dirtyAttributes.isEmpty();
    }

    public void $$_hibernate_clearDirtyAttributes() {
        if ($$_hibernate_tracker_dirtyAttributes != null) {
            $$_hibernate_tracker_dirtyAttributes.clear();
        }
    }
}
```

### 7.4 Three Enhancement Capabilities

#### 1. Lazy Field Initialization (`enableLazyInitialization`)

```java
em.find(Document.class, 42L)
  │
  ├── SELECT id, title FROM document WHERE id = 42
  │   (profilePhoto NOT selected — it's lazy)
  │
  ├── customer.name = "Alice"           ← populated
  ├── customer.profilePhoto = <uninit>  ← NOT loaded
  │
  └── customer.getProfilePhoto()
      │
      ├── interceptor.readObject(this, "profilePhoto", null)
      │   → detects uninitialized
      │   → SELECT profile_photo FROM document WHERE id = 42
      │   → sets customer.profilePhoto = <bytes>
      │
      └── returns the loaded bytes
```

**Fetch groups:** You can group lazy fields so they load together:

```java
@LazyGroup("photos")
@Basic(fetch = FetchType.LAZY)
private byte[] profilePhoto;

@LazyGroup("photos")
@Basic(fetch = FetchType.LAZY)
private byte[] thumbnailPhoto;

// Accessing either one loads BOTH in a single SELECT
```

#### 2. Dirty Tracking (`enableDirtyTracking`)

**Without enhancement:**
```
flush()
  └── for EVERY managed entity:
      currentState = reflectionGetAllFields(entity)
      if (currentState != snapshot)  ← deep comparison of ALL fields
          queue UPDATE
```

**With enhancement:**
```java
entity.setName("Bob")
  └── $$_hibernate_write_name()
      └── dirtyAttributes.add("name")  ← tracked immediately

flush()
  └── for EVERY managed entity:
      if (entity.$$_hibernate_hasDirtyAttributes())  ← O(1) check
          dirtyFields = entity.$$_hibernate_getDirtyAttributes()
          queue UPDATE for only those fields
```

#### 3. Association Management (`enableAssociationManagement`)

Keeps bidirectional relationships in sync automatically:

```java
// Without enhancement — you must manage both sides manually:
order.setCustomer(customer);
customer.getOrders().add(order);   // ← easy to forget

// With enhancement — Hibernate does it for you:
order.setCustomer(customer);
// $$_hibernate_write_customer() automatically adds 'order' 
// to customer.getOrders()
```

### 7.5 Proxy vs Enhancement — Comparison

| Feature | Proxy-Based | Bytecode Enhancement |
|---------|-------------|----------------------|
| **Mechanism** | Runtime subclass | Build-time class rewriting |
| **Works on** | `@ManyToOne`, `@OneToOne`, `getReference()` | Basic fields, `@OneToOne` owned side, dirty tracking |
| **Entity class** | Unchanged | Modified at compile time |
| **`getClass()`** | Returns proxy class ⚠️ | Returns real class ✅ |
| **`final` classes** | ❌ Cannot proxy final classes | ✅ Works on final classes |
| **`final` methods** | ❌ Cannot intercept | ✅ Intercepted via field access |
| **Field access** | Not intercepted (only methods) | Intercepted at field level |
| **Granularity** | Whole entity loaded at once | Individual fields/groups |
| **Setup** | Automatic, zero config | Requires build plugin |
| **Runtime overhead** | Proxy creation + reflection | Minimal — direct field access |

---

## 8. Limitations of Proxy-Based Lazy Loading

### 8.1 Limitation 1: Can't Lazily Load Basic Fields

**The Problem:**

```java
@Entity
public class Document {
    @Id
    private Long id;

    private String title;           // small, always want this

    @Basic(fetch = FetchType.LAZY)
    private byte[] fileContent;     // 50MB PDF, want this lazy
}
```

Without enhancement, `@Basic(fetch = LAZY)` is **silently ignored**. Hibernate loads everything:

```sql
-- What you WANT:
SELECT id, title FROM document WHERE id = 42

-- What you GET (without enhancement):
SELECT id, title, file_content FROM document WHERE id = 42
```

**Why:** Proxies work by **subclassing the entity and overriding methods**. But `title`, `fileContent` are all fields *inside the same object*. There's no separate object to proxy. The proxy operates at the **entity level** — it's all-or-nothing.

**How Enhancement Fixes It:**

Enhancement rewrites `getFileContent()` to call the interceptor *for that specific field*:

```java
// Enhanced
public byte[] getFileContent() {
    if ($$_hibernate_interceptor != null) {
        fileContent = (byte[]) $$_hibernate_interceptor
            .readObject(this, "fileContent", fileContent);
        //  Interceptor checks: is "fileContent" in uninitializedFields?
        //  YES → SELECT file_content FROM document WHERE id = 42
        //  NO  → return current value
    }
    return fileContent;
}
```

### 8.2 Limitation 2: Can't Proxy the Owned Side of `@OneToOne`

**The Setup:**

```java
@Entity
public class User {
    @Id
    private Long id;

    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
    private UserProfile profile;  // ← LAZY should mean: don't load profile
}

@Entity
public class UserProfile {
    @Id
    private Long id;

    @OneToOne
    @JoinColumn(name = "USER_ID")  // ← FK is HERE, in UserProfile's table
    private User user;
}
```

**Table layout:**
```
USERS table:                  USER_PROFILE table:
┌────┬───────┐               ┌────┬─────────┐
│ ID │ NAME  │               │ ID │ USER_ID  │  ← FK is here
├────┼───────┤               ├────┼─────────┤
│ 1  │ Alice │               │ 10 │ 1       │
│ 2  │ Bob   │               │    │         │  ← no profile for Bob
└────┴───────┘               └────┴─────────┘
```

**The Problem:**

When Hibernate loads `User` with id=2 (Bob), it needs to set `user.profile`. But:

- To create a **PROXY**, Hibernate needs to know: "Does a UserProfile row exist with USER_ID = 2?"
- The USERS table has **NO FK column** pointing to UserProfile
- The FK is in the **OTHER table**: USER_PROFILE.USER_ID
- Hibernate **CANNOT know** if profile is `null` or not WITHOUT querying:
  ```sql
  SELECT id FROM user_profile WHERE user_id = 2
  ```
- If it must query anyway, lazy loading is **DEFEATED**

**Why is this different from `@ManyToOne`?**

```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "CUSTOMER_ID")  // ← FK is HERE, in Order's table
    private Customer customer;
}
```

For `@ManyToOne`, the FK column is in the **same row** Hibernate already loaded. It can see `CUSTOMER_ID = 42` and create a proxy with that ID. **No extra query needed.**

For `@OneToOne(mappedBy=...)`, the FK is in the **other table**. Hibernate has to query to find out.

**Solutions:**

```java
// Solution 1: Move the FK to this side (best)
@Entity
public class User {
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "PROFILE_ID")  // FK now in User's table
    private UserProfile profile;       // Proxy works! ✅
}

// Solution 2: Use optional = false (if profile ALWAYS exists)
@Entity
public class User {
    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY, optional = false)
    private UserProfile profile;  
    // Hibernate trusts you: profile is NEVER null
    // → safe to always create a proxy ✅
}

// Solution 3: Bytecode enhancement
// Enhancement intercepts field access, not method calls
// It can track "uninitialized" state per field
// → Works correctly with null detection
```

### 8.3 Limitation 3: Can't Proxy `final` Classes or Intercept `final` Methods

**The Mechanism:**

Hibernate proxies work by **subclassing**:

```java
// Hibernate generates at runtime:
class Customer$HibernateProxy extends Customer {
    @Override
    public String getName() {
        // intercept and lazy-load
    }
}
```

**`final` class:**

```java
public final class Customer {  // ← can't subclass
    // ...
}

// Hibernate tries:
class Customer$HibernateProxy extends Customer { }
//                              ^^^^^^^^^^^^^^
// COMPILATION ERROR: Cannot extend final class
```

**`final` method:**

```java
public class Customer {
    public final String getName() {  // ← can't override
        return name;
    }
}

// Proxy:
class Customer$HibernateProxy extends Customer {
    @Override
    public String getName() {  // ❌ Cannot override final method
        // lazy loading code never runs
    }
}
```

If a `final` method is called on a proxy, it executes the **original method body** directly, bypassing the lazy-loading interceptor. The method reads the field directly from the proxy object (which is uninitialized), returning `null` or default values silently — **no exception, just wrong data.**

**How Enhancement Fixes It:**

Enhancement doesn't subclass — it **rewrites the class itself**. The interception code is injected *inside* the getter, even if it's `final`:

```java
// Original (final):
public final String getName() {
    return name;
}

// After enhancement (still final, but rewritten):
public final String getName() {
    if ($$_hibernate_interceptor != null) {
        name = (String) $$_hibernate_interceptor
            .readObject(this, "name", name);
    }
    return name;
}
```

The method stays `final` — no subclassing needed. The interception is **inside** the method.

### 8.4 Limitation 4: Dirty Checking Is Expensive Without Enhancement

**How Default Dirty Checking Works:**

At flush time, Hibernate must determine which managed entities have changed. Without enhancement:

```
flush()
  │
  └── for EACH entity in Persistence Context:
      │
      ├── Object[] currentState = persister.getPropertyValues(entity)
      │   // Uses reflection to read EVERY field
      │
      ├── Object[] snapshot = entityEntry.getLoadedState()
      │
      └── for (int i = 0; i < fields.length; i++):
              if (!type[i].isEqual(currentState[i], snapshot[i]))
                  dirtyFields.add(i)
```

**The Cost:**

```
Persistence Context contains:
  500 managed entities × 20 fields each = 10,000 field comparisons

Every flush:
  ├── 500 reflection calls (getPropertyValues)
  ├── 10,000 equality comparisons
  ├── Most entities haven't changed
  └── All that work for maybe 3 actual updates
```

**With Enhancement — Inline Dirty Tracking:**

```java
entity.setName("Bob")
  └── $$_hibernate_write_name()
      └── dirtyAttributes.add("name")  ← tracked immediately

flush()
  └── for EACH entity in Persistence Context:
      └── entity.$$_hibernate_hasDirtyAttributes()  ← O(1) check
          ├── false → SKIP (no reflection, no comparison) ✅
          └── true  → read only dirty field names
                    → generate UPDATE for those fields only
```

**Performance comparison:**

| Aspect | Without Enhancement | With Enhancement |
|--------|---------------------|------------------|
| Reflection calls | 500 | 0 |
| Field comparisons | 10,000 | 0 |
| Dirty detection | O(entities × fields) | O(entities) |
| Per-entity check | O(fields) | O(1) |
| Memory for tracking | Full snapshot array | Small Set<String> |


---

## 9. Transaction Management

### 9.1 Transaction Boundaries

A transaction in JPA/Hibernate defines a **boundary** within which all database operations are treated as a single atomic unit.

```
Transaction Boundary
┌─────────────────────────────────────────────────────┐
│                                                     │
│  em.getTransaction().begin()                       │
│  │                                                 │
│  │  em.persist(order)                              │
│  │  order.setStatus("CONFIRMED")                   │
│  │  em.persist(invoice)                            │
│  │                                                 │
│  │  em.getTransaction().commit()                   │
│  │                                                 │
│  └─→ All changes committed atomically              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**What Happens at Transaction Boundaries:**

**At begin():**
- JDBC connection is acquired from the connection pool (if not already acquired)
- Auto-commit is disabled on the JDBC connection
- Database-specific isolation level is applied
- A new transaction context is established

**Within the transaction:**
- All entity operations are tracked in the Persistence Context
- Dirty checking monitors entity state changes
- SQL statements are queued (write-behind pattern)
- The database connection remains open and locked to this transaction

**At commit():**
- Flush occurs (Persistence Context synchronized to database)
- All queued SQL is executed in the correct order
- Database commits the transaction (ACID durability)
- Connection is released back to the pool (or kept for reuse)
- Persistence Context can be cleared or closed

**At rollback():**
- All queued SQL is discarded
- Database rolls back all changes
- Connection is released
- Persistence Context is typically cleared

**Key Insight: Transaction vs Session Lifecycle**

The transaction boundary is NOT the same as the Session/EntityManager lifecycle:

```java
// One Session, Multiple Transactions (less common)
EntityManager em = emf.createEntityManager();  // Session created

em.getTransaction().begin();
em.persist(order1);
em.getTransaction().commit();  // Transaction 1 ends

em.getTransaction().begin();
em.persist(order2);
em.getTransaction().commit();  // Transaction 2 ends

em.close();  // Session ends
```

**Typical Pattern (One Session = One Transaction):**

```java
// In Spring with @Transactional
@Transactional
public void createOrder() {
    // Spring creates EntityManager at method start
    // Transaction begins
    Order order = new Order();
    em.persist(order);
    // Transaction commits at method end
    // EntityManager closed
}
```

**Transaction Boundary Best Practices:**

1. **Keep transactions short** - Long transactions hold database locks and connections, reducing concurrency
2. **Avoid I/O operations in transactions** - Don't make HTTP calls, file operations, or slow computations within a transaction
3. **Define clear boundaries** - Transaction should align with a single business operation
4. **Handle exceptions properly** - Unhandled exceptions trigger rollback

**Common Anti-Pattern:**

```java
// BAD: Long transaction with I/O
@Transactional
public void processOrderWithNotification(Long orderId) {
    Order order = em.find(Order.class, orderId);
    order.setStatus("PROCESSED");
    
    // BAD: HTTP call inside transaction holds DB lock
    emailService.sendNotification(order);  // Could take seconds
    
    // Transaction still open, DB locked
}
```

**Correct Pattern:**

```java
// GOOD: Split transaction and I/O
@Transactional
public void processOrder(Long orderId) {
    Order order = em.find(Order.class, orderId);
    order.setStatus("PROCESSED");
    // Transaction commits here
}

public void processOrderWithNotification(Long orderId) {
    processOrder(orderId);  // Transaction completes
    Order order = em.find(Order.class, orderId);  // New transaction
    emailService.sendNotification(order);  // I/O outside transaction
}
```

### 9.2 Transaction Isolation Levels

Isolation levels determine how transactions interact with each other. Set via JDBC connection:

| Isolation Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Use Case |
|-----------------|-------------|---------------------|---------------|----------|
| **READ_UNCOMMITTED** | ✅ Possible | ✅ Possible | ✅ Possible | Rare - only for logging |
| **READ_COMMITTED** | ❌ Prevented | ✅ Possible | ✅ Possible | Default in many DBs (PostgreSQL, SQL Server) |
| **REPEATABLE_READ** | ❌ Prevented | ❌ Prevented | ✅ Possible | Default in MySQL InnoDB |
| **SERIALIZABLE** | ❌ Prevented | ❌ Prevented | ❌ Prevented | Highest consistency, lowest performance |

**Understanding the Phenomena:**

**Dirty Read**: Transaction A reads uncommitted changes made by Transaction B. If B rolls back, A has read invalid data.

```java
// Transaction A
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public void checkBalance(Long accountId) {
    Account account = em.find(Account.class, accountId);
    // Reads balance = 1000 (uncommitted from Transaction B)
    System.out.println(account.getBalance());
}

// Transaction B (running concurrently)
@Transactional
public void deposit(Long accountId, BigDecimal amount) {
    Account account = em.find(Account.class, accountId);
    account.setBalance(account.getBalance().add(amount));  // balance = 1000
    // Not yet committed, but Transaction A can read it
}
```

**Non-Repeatable Read**: Transaction A reads the same row twice and gets different values because Transaction B committed a change in between.

```java
// Transaction A
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processAccount(Long accountId) {
    Account account1 = em.find(Account.class, accountId);  // balance = 500
    
    // ... Transaction B commits a change to balance = 600 ...
    
    Account account2 = em.find(Account.class, accountId);  // balance = 600
    // Non-repeatable read! Same query, different result
}
```

**Phantom Read**: Transaction A executes the same query twice and gets different numbers of rows because Transaction B inserted/deleted rows in between.

```java
// Transaction A
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void listActiveOrders() {
    List<Order> orders1 = em.createQuery(
        "SELECT o FROM Order o WHERE o.status = 'ACTIVE'", Order.class
    ).getResultList();  // Returns 5 orders
    
    // ... Transaction B inserts a new ACTIVE order and commits ...
    
    List<Order> orders2 = em.createQuery(
        "SELECT o FROM Order o WHERE o.status = 'ACTIVE'", Order.class
    ).getResultList();  // Returns 6 orders
    // Phantom read! Same query, different row count
}
```

**Configuration:**

```properties
# application.properties
spring.jpa.properties.hibernate.connection.isolation=2  # READ_COMMITTED
# Values: 1=READ_UNCOMMITTED, 2=READ_COMMITTED, 4=REPEATABLE_READ, 8=SERIALIZABLE
```

**Or programmatically in Spring:**

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    // This method uses REPEATABLE_READ isolation
}
```

**Practical Scenarios and Recommendations:**

**READ_COMMITTED (Recommended for most applications):**
- Good balance between consistency and performance
- Prevents dirty reads (most dangerous phenomenon)
- Allows non-repeatable reads and phantom reads (usually acceptable)
- Default in PostgreSQL, SQL Server, Oracle

**Use READ_COMMITTED when:**
- Most web applications
- Reporting systems where slight inconsistencies are acceptable
- High-concurrency environments

**REPEATABLE_READ (Recommended for financial/accounting):**
- Prevents dirty reads and non-repeatable reads
- Still allows phantom reads
- Default in MySQL InnoDB
- Slightly higher locking overhead

**Use REPEATABLE_READ when:**
- Financial transactions (banking, accounting)
- Inventory management
- Any scenario where reading inconsistent data for the same entity is unacceptable

**SERIALIZABLE (Highest consistency, lowest performance):**
- Prevents all three phenomena
- Uses extensive locking (range locks)
- Can cause deadlocks and significant performance degradation
- Only use when absolutely necessary

**Use SERIALIZABLE when:**
- Critical financial operations (e.g., stock trading)
- Audit trails where absolute consistency is required
- Low-concurrency environments where performance is not critical

**READ_UNCOMMITTED (Rarely used):**
- No isolation guarantees
- Only for logging/analytics where exact accuracy doesn't matter
- Can read rolled-back data (invalid state)

**Use READ_UNCOMMITTED when:**
- Debugging/profiling (to see uncommitted state)
- Logging systems where approximate data is acceptable
- Never for user-facing applications

**Database-Specific Defaults:**

| Database | Default Isolation | Notes |
|----------|-------------------|-------|
| PostgreSQL | READ_COMMITTED | SERIALIZABLE available |
| MySQL InnoDB | REPEATABLE_READ | READ_COMMITTED available |
| SQL Server | READ_COMMITTED | All levels available |
| Oracle | READ_COMMITTED | Only READ_COMMITTED and SERIALIZABLE |

**Performance Impact:**

Higher isolation levels = more locking = lower concurrency:

```
READ_UNCOMMITTED:  ████████████████████ (highest throughput)
READ_COMMITTED:    ████████████████     (slightly less)
REPEATABLE_READ:   ████████████         (moderate locking)
SERIALIZABLE:      ████                 (extensive locking, lowest throughput)
```

### 9.3 Transaction Propagation (Spring)

When using Spring's `@Transactional`, propagation behavior controls how transactions relate to each other:

| Propagation | Description | Example |
|-------------|-------------|---------|
| **REQUIRED** (default) | Join existing transaction or create new | Service A → Service B (both in same tx) |
| **REQUIRES_NEW** | Suspend current, create new transaction | Service A → Service B (B commits independently) |
| **MANDATORY** | Must join existing transaction, throw if none | Service methods must be called from another tx |
| **SUPPORTS** | Join if exists, execute non-transactionally if not | Read-only operations that can run without tx |
| **NOT_SUPPORTED** | Execute non-transactionally, suspend if exists | Reports, analytics |
| **NEVER** | Execute non-transactionally, throw if tx exists | Explicitly non-tx operations |
| **NESTED** | Create nested savepoint within existing tx | Partial rollback capability |

**Detailed Examples for Each Propagation:**

**REQUIRED (default):**

```java
@Service
public class OrderService {
    
    @Transactional  // Propagation.REQUIRED (default)
    public void createOrderWithItems(OrderDto dto) {
        Order order = new Order(dto);
        em.persist(order);  // Part of transaction T1
        
        // Joins the same transaction T1
        itemService.addItems(order.getId(), dto.getItems());
        
        // Both operations commit together
    }
}

@Service
public class ItemService {
    
    @Transactional  // Propagation.REQUIRED (default)
    public void addItems(Long orderId, List<ItemDto> items) {
        Order order = em.find(Order.class, orderId);
        for (ItemDto itemDto : items) {
            Item item = new Item(itemDto);
            item.setOrder(order);
            em.persist(item);  // Part of transaction T1
        }
    }
}
```

**Behavior:**
- If called from non-transactional context: Creates new transaction
- If called from existing transaction: Joins that transaction
- Most commonly used propagation

**REQUIRES_NEW:**

```java
@Service
public class OrderService {
    
    @Transactional
    public void processOrder(Long orderId) {
        Order order = em.find(Order.class, orderId);
        order.setStatus("PROCESSED");
        
        // This runs in a SEPARATE transaction T2
        // Transaction T1 is suspended
        auditService.logAction("ORDER_PROCESSED", orderId);
        
        // If auditService fails, order is still committed (in T1)
        // If orderService fails, audit log is still committed (in T2)
    }
}

@Service
public class AuditService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAction(String action, Long entityId) {
        AuditLog log = new AuditLog(action, entityId);
        em.persist(log);  // Part of transaction T2
    }
}
```

**Behavior:**
- Always creates a new transaction
- Suspends any existing transaction
- Independent commit/rollback from caller
- Use case: Audit logging, email sending, notifications

**MANDATORY:**

```java
@Service
public class PaymentService {
    
    @Transactional(propagation = Propagation.MANDATORY)
    public void processPayment(Long orderId, BigDecimal amount) {
        // This method MUST be called from within a transaction
        Payment payment = new Payment(orderId, amount);
        em.persist(payment);
    }
}

// Caller MUST be transactional
@Service
public class OrderService {
    
    @Transactional
    public void checkout(Long orderId) {
        // This works - has transaction
        paymentService.processPayment(orderId, new BigDecimal("100.00"));
    }
}

// This would throw IllegalTransactionStateException
public void badCall() {
    paymentService.processPayment(1L, new BigDecimal("100.00"));
}
```

**Behavior:**
- Must join existing transaction
- Throws exception if no transaction exists
- Use case: Methods that should never be called outside transaction context

**SUPPORTS:**

```java
@Service
public class ReportService {
    
    @Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
    public OrderReport generateReport(Long orderId) {
        Order order = em.find(Order.class, orderId);
        // If called with transaction: joins it
        // If called without transaction: runs non-transactionally
        return new OrderReport(order);
    }
}

// Called from transactional method
@Transactional
public void processWithReport(Long orderId) {
    reportService.generateReport(orderId);  // Joins existing transaction
}

// Called from non-transactional method
public void justReport(Long orderId) {
    reportService.generateReport(orderId);  // Runs non-transactionally
}
```

**Behavior:**
- Joins transaction if one exists
- Executes non-transactionally if none
- Use case: Read-only operations that don't require transaction

**NOT_SUPPORTED:**

```java
@Service
public class AnalyticsService {
    
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void updateAnalytics(Long orderId) {
        // Suspends any existing transaction
        // Runs non-transactionally
        Analytics analytics = analyticsRepository.findByOrderId(orderId);
        analytics.incrementViewCount();
        analyticsRepository.save(analytics);
    }
}

@Transactional
public void processOrder(Long orderId) {
    Order order = em.find(Order.class, orderId);
    order.setStatus("PROCESSED");
    
    // Transaction T1 suspended here
    analyticsService.updateAnalytics(orderId);
    // Transaction T1 resumes here
    
    // If analytics fails, order still commits
    // If order fails, analytics still commits
}
```

**Behavior:**
- Always executes non-transactionally
- Suspends existing transaction if present
- Use case: Long-running operations, analytics, reports

**NEVER:**

```java
@Service
public class CacheService {
    
    @Transactional(propagation = Propagation.NEVER)
    public void refreshCache(Long orderId) {
        // Must NOT run in a transaction
        // Throws exception if called from transactional context
        cacheManager.evict("order_" + orderId);
    }
}

// This works - no transaction
public void refreshCacheDirectly(Long orderId) {
    cacheService.refreshCache(orderId);
}

// This throws IllegalTransactionStateException
@Transactional
public void badRefresh(Long orderId) {
    cacheService.refreshCache(orderId);
}
```

**Behavior:**
- Must execute non-transactionally
- Throws exception if transaction exists
- Use case: Cache operations, external calls that shouldn't be transactional

**NESTED:**

```java
@Service
public class OrderService {
    
    @Transactional
    public void processOrderWithRetry(Long orderId) {
        Order order = em.find(Order.class, orderId);
        order.setStatus("PROCESSING");
        
        try {
            // Nested transaction T2 (savepoint)
            paymentService.processPayment(orderId);
            inventoryService.reserveItems(orderId);
        } catch (PaymentException e) {
            // Rollback only the nested transaction
            // Order status "PROCESSING" is preserved
            throw e;
        }
    }
}

@Service
public class PaymentService {
    
    @Transactional(propagation = Propagation.NESTED)
    public void processPayment(Long orderId) {
        Payment payment = new Payment(orderId);
        em.persist(payment);
        // If this fails, only this nested transaction rolls back
        // Outer transaction can continue
    }
}
```

**Behavior:**
- Creates nested transaction within existing transaction
- Uses database savepoints
- Can rollback nested transaction independently
- Outer transaction can still commit
- Use case: Partial rollback scenarios, retry logic
- Note: Not supported by all databases (requires savepoint support)

**Propagation Decision Tree:**

```
Should this method run in a transaction?
├─ Yes, always create new → REQUIRES_NEW
├─ Yes, join if exists, create if not → REQUIRED (default)
├─ Yes, but only if one exists → MANDATORY
├─ No, never → NEVER
├─ No, but can join if exists → SUPPORTS
├─ No, suspend if exists → NOT_SUPPORTED
└─ Yes, nested within existing → NESTED
```

### 9.4 Transaction Rollback

**Automatic rollback on:**
- `RuntimeException` or `Error`
- Checked exceptions (if configured)
- `setRollbackOnly()` call

```java
@Transactional
public void processOrder(Long orderId) {
    Order order = em.find(Order.class, orderId);
    order.setStatus("PROCESSED");
    
    if (order.getTotal() > 10000) {
        // Force rollback
        em.getTransaction().setRollbackOnly();
        // or throw new RuntimeException("Amount too high");
    }
}
```

**Spring configuration:**

```java
@Transactional(rollbackFor = {CustomBusinessException.class})
public void processWithCustomException() {
    // CustomBusinessException will trigger rollback
}
```

**Spring's Default Rollback Behavior:**

Spring's `@Transactional` defaults to rolling back only on `RuntimeException` and `Error`, NOT on checked exceptions. This design choice stems from the principle that checked exceptions are often "business exceptions" that the application might want to handle and continue execution.

```java
// Checked exception - does NOT trigger rollback by default
@Transactional
public void processOrder(Long orderId) throws InsufficientStockException {
    Order order = em.find(Order.class, orderId);
    if (!inventoryService.hasStock(order.getItems())) {
        throw new InsufficientStockException("Not enough stock");
        // Transaction will NOT rollback - changes will commit!
    }
}
```

**Configuring Rollback for Checked Exceptions:**

```java
@Transactional(rollbackFor = {InsufficientStockException.class})
public void processOrder(Long orderId) throws InsufficientStockException {
    Order order = em.find(Order.class, orderId);
    if (!inventoryService.hasStock(order.getItems())) {
        throw new InsufficientStockException("Not enough stock");
        // Transaction WILL rollback
    }
}
```

**Preventing Rollback on Runtime Exceptions:**

```java
@Transactional(noRollbackFor = {BusinessValidationException.class})
public void processOrder(Long orderId) {
    Order order = em.find(Order.class, orderId);
    
    if (order.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
        throw new BusinessValidationException("Invalid amount");
        // Transaction will NOT rollback despite being RuntimeException
    }
}
```

**setRollbackOnly() - Programmatic Rollback:**

```java
@Transactional
public void processOrderWithValidation(Long orderId) {
    Order order = em.find(Order.class, orderId);
    order.setStatus("PROCESSING");
    
    try {
        paymentService.charge(order);
        inventoryService.reserve(order);
    } catch (PaymentFailedException e) {
        // Mark transaction for rollback without throwing
        em.getTransaction().setRollbackOnly();
        
        // Still can execute cleanup logic
        notificationService.notifyFailure(order, e.getMessage());
        
        // Transaction will rollback at commit time
    }
}
```

**When to use setRollbackOnly():**
- When you want to execute cleanup logic after determining rollback is needed
- When you don't want to propagate an exception to the caller
- When handling exceptions in a catch block but still want to rollback

**Rollback in Nested Transactions:**

```java
@Transactional
public void outerMethod() {
    // Outer transaction T1
    innerMethod();  // NESTED transaction T2
}

@Transactional(propagation = Propagation.NESTED)
public void innerMethod() {
    // If this fails, only T2 rolls back to savepoint
    // T1 can continue and commit
    throw new RuntimeException("Nested failure");
    // T2 rolls back, T1 continues
}
```

**Rollback in REQUIRES_NEW Transactions:**

```java
@Transactional
public void outerMethod() {
    // Outer transaction T1
    try {
        innerMethod();  // REQUIRES_NEW transaction T2
    } catch (Exception e) {
        // T2 rolled back, but T1 continues
    }
    // T1 can still commit
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void innerMethod() {
    // Separate transaction T2
    throw new RuntimeException("Independent failure");
    // T2 rolls back, T1 unaffected
}
```

**Common Rollback Pitfalls:**

**Pitfall 1: Swallowing Exceptions**

```java
// BAD: Exception swallowed, no rollback
@Transactional
public void badExample() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Failed", e);
        // Exception caught, transaction commits!
    }
}
```

**Correct:**

```java
// GOOD: Re-throw to trigger rollback
@Transactional
public void goodExample() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Failed", e);
        throw e;  // Re-throw to trigger rollback
    }
}
```

**Pitfall 2: Checked Exception Not Rolling Back**

```java
// BAD: Checked exception doesn't rollback by default
@Transactional
public void badExample() throws BusinessException {
    if (invalid) {
        throw new BusinessException("Invalid");
        // Transaction commits despite exception!
    }
}
```

**Correct:**

```java
// GOOD: Configure rollback for checked exception
@Transactional(rollbackFor = BusinessException.class)
public void goodExample() throws BusinessException {
    if (invalid) {
        throw new BusinessException("Invalid");
        // Transaction will rollback
    }
}
```

**Pitfall 3: @Transactional on Private Methods**

```java
// BAD: @Transactional doesn't work on private methods
@Transactional
private void badExample() {
    // This won't be transactional!
    em.persist(new Order());
}
```

**Correct:**

```java
// GOOD: Use public or protected methods
@Transactional
public void goodExample() {
    em.persist(new Order());
}
```

**Rollback and Persistence Context:**

When a transaction rolls back:
- All queued SQL is discarded
- The Persistence Context is typically cleared
- Managed entities become detached
- Changes to entity fields are lost (not persisted to DB)

```java
@Transactional
public void demonstrateRollbackEffect() {
    Order order = new Order();
    order.setStatus("PENDING");
    em.persist(order);  // Queued for INSERT
    
    order.setStatus("CANCELLED");  // Change queued for UPDATE
    
    // ... exception thrown, transaction rolls back ...
    
    // After rollback:
    // - INSERT never executed
    // - UPDATE never executed
    // - order is detached
    // - order.getId() returns null (no ID assigned)
}
```

### 9.5 Flush Modes

Controls when Hibernate synchronizes the Persistence Context with the database:

| Flush Mode | When Flush Occurs | Use Case |
|------------|------------------|----------|
| **AUTO** (default) | Before query execution, at commit | Most common |
| **COMMIT** | Only at transaction commit | Batch processing, read-heavy |
| **ALWAYS** | Before every query (even if no changes) | Rare, testing |
| **MANUAL** | Only when `em.flush()` called | Full control |

```java
// Set flush mode
em.setFlushMode(FlushModeType.COMMIT);

// Or via Spring
@Transactional(flushMode = FlushModeType.COMMIT)
public void batchProcess() {
    // Flush only at commit
}
```

**Understanding Flush vs Commit:**

**Flush** is the process of synchronizing the in-memory Persistence Context state with the database (executing queued SQL). This happens within the transaction.

**Commit** is the database operation that makes the flushed changes permanent (ACID durability).

```
Transaction Timeline:
┌─────────────────────────────────────────────────────────┐
│  begin()                                                │
│    │                                                    │
│    ├─ em.persist(order)  → queued (not executed)        │
│    ├─ order.setStatus(...) → queued (not executed)     │
│    │                                                    │
│    ├─ em.flush()  ← FLUSH: SQL executed here            │
│    │   INSERT INTO orders ...                           │
│    │   UPDATE orders SET status = ...                  │
│    │                                                    │
│    └─ commit()  ← COMMIT: changes made permanent       │
└─────────────────────────────────────────────────────────┘
```

**AUTO (Default) - Most Common:**

Flush occurs:
- Before query execution (if the query might be affected by pending changes)
- At transaction commit
- When explicitly calling `em.flush()`

```java
@Transactional(flushMode = FlushModeType.AUTO)  // default
public void autoFlushExample() {
    Order order = new Order();
    em.persist(order);  // Queued, not executed yet
    
    // Hibernate detects that this query might be affected
    // by the pending INSERT, so it flushes first
    List<Order> orders = em.createQuery(
        "SELECT o FROM Order o WHERE o.status = 'PENDING'", 
        Order.class
    ).getResultList();
    // Flush occurred before this query
}
```

**When AUTO flushes before queries:**
- JPQL/Criteria queries that might be affected by pending changes
- Native SQL queries (if configured)
- NOT for simple `em.find()` by ID (identity map guarantees consistency)

**COMMIT - Batch Processing Optimized:**

Flush occurs ONLY at transaction commit. No automatic flushing before queries.

```java
@Transactional(flushMode = FlushModeType.COMMIT)
public void batchInsertOrders(List<OrderDto> dtos) {
    for (OrderDto dto : dtos) {
        Order order = new Order(dto);
        em.persist(order);
        // No flush - just queued
        
        // Even if we query, no auto-flush
        Long count = em.createQuery(
            "SELECT COUNT(o) FROM Order o", Long.class
        ).getSingleResult();
        // Query sees old state (doesn't include new orders)
    }
    // Flush occurs only here at commit
}
```

**Use COMMIT when:**
- Batch processing large datasets
- Read-heavy operations where you know you won't modify data
- You want to control exactly when SQL executes
- Performance optimization (avoid unnecessary dirty checking)

**ALWAYS - Maximum Consistency:**

Flush before EVERY query execution, even if there are no pending changes.

```java
@Transactional(flushMode = FlushModeType.ALWAYS)
public void alwaysFlushExample() {
    // Even though we're just reading
    Order order = em.find(Order.class, 1L);
    
    // This triggers a flush (even though nothing changed)
    List<Order> orders = em.createQuery(
        "SELECT o FROM Order o", Order.class
    ).getResultList();
}
```

**Use ALWAYS when:**
- Testing scenarios where you want maximum consistency
- Debugging to ensure state is always synchronized
- Rarely used in production (performance overhead)

**MANUAL - Full Control:**

Flush ONLY when explicitly called. No automatic flushing at all.

```java
@Transactional(flushMode = FlushModeType.MANUAL)
public void manualFlushExample() {
    Order order = new Order();
    em.persist(order);  // Queued
    
    // Query won't trigger flush
    List<Order> orders = em.createQuery(
        "SELECT o FROM Order o", Order.class
    ).getResultList();
    // Query sees old state
    
    // Explicit flush
    em.flush();  // SQL executed now
    
    // Now query sees new state
    orders = em.createQuery(
        "SELECT o FROM Order o", Order.class
    ).getResultList();
}
```

**Use MANUAL when:**
- You need complete control over when SQL executes
- Complex scenarios with conditional flushing
- Advanced optimization scenarios
- When you understand the implications and can manage consistency manually

**Practical Example: Batch Processing with COMMIT:**

```java
@Transactional(flushMode = FlushModeType.COMMIT)
public void importOrdersFromCSV(InputStream csv) {
    int batchSize = 50;
    int count = 0;
    
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(csv))) {
        String line;
        while ((line = reader.readLine()) != null) {
            Order order = parseOrder(line);
            em.persist(order);
            
            count++;
            if (count % batchSize == 0) {
                // Manual flush to avoid memory issues
                em.flush();
                em.clear();  // Clear Persistence Context
            }
        }
    }
    // Final flush at commit
}
```

**Practical Example: Read-Heavy with AUTO:**

```java
@Transactional(flushMode = FlushModeType.AUTO)  // default
public Order getOrderDetails(Long orderId) {
    Order order = em.find(Order.class, orderId);
    
    // No modifications, so no flush needed
    // AUTO mode is smart enough to know this
    
    List<Item> items = em.createQuery(
        "SELECT i FROM Item i WHERE i.order.id = :orderId",
        Item.class
    ).setParameter("orderId", orderId)
     .getResultList();
    
    order.setItems(items);
    return order;
}
```

**Flush Mode and Performance:**

```
Flush Mode Performance Impact:

AUTO:      ████████████████████ (balanced)
COMMIT:    ████████████████████ (better for batch, worse for consistency)
ALWAYS:    ████████             (worst performance, best consistency)
MANUAL:    ████████████████     (depends on usage)
```

**Key Considerations:**

1. **AUTO is usually the right choice** - It balances consistency and performance
2. **COMMIT for batch operations** - Reduces unnecessary flushes
3. **Be careful with MANUAL** - Can lead to stale data if not managed properly
4. **Flush mode doesn't affect commit** - Commit always happens at transaction end regardless of flush mode

**Common Pitfall: Stale Data with COMMIT:**

```java
// POTENTIAL BUG
@Transactional(flushMode = FlushModeType.COMMIT)
public void problematicMethod() {
    Order order = new Order();
    em.persist(order);  // Queued, not flushed
    
    // This query doesn't see the new order!
    Order found = em.createQuery(
        "SELECT o FROM Order o WHERE o.id = :id",
        Order.class
    ).setParameter("id", order.getId())
     .getSingleResult();
    // Returns null or throws exception
}
```

**Solution:**

```java
// FIX: Use AUTO (default) or explicit flush
@Transactional(flushMode = FlushModeType.AUTO)
public void fixedMethod() {
    Order order = new Order();
    em.persist(order);
    
    // AUTO flushes before query, so this works
    Order found = em.createQuery(
        "SELECT o FROM Order o WHERE o.id = :id",
        Order.class
    ).setParameter("id", order.getId())
     .getSingleResult();
    // Returns the order
}
```

---

## 10. Inheritance Mapping Strategies

JPA supports three strategies for mapping entity inheritance hierarchies to database tables.

### 10.1 SINGLE_TABLE (Default)

All classes in the hierarchy are mapped to a single table with a discriminator column.

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "account_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Account {
    @Id
    private Long id;
    private String owner;
    private BigDecimal balance;
}

@Entity
@DiscriminatorValue("SAVINGS")
public class SavingsAccount extends Account {
    private Double interestRate;
}

@Entity
@DiscriminatorValue("CHECKING")
public class CheckingAccount extends Account {
    private Double overdraftLimit;
}
```

**Database Schema:**
```
┌────────────────────────────────────────────────────┐
│ ACCOUNT                                             │
├──────────┬──────────┬──────────┬───────────────────┤
│ ID       │ OWNER    │ BALANCE  │ ACCOUNT_TYPE      │
├──────────┼──────────┼──────────┼───────────────────┤
│ 1        │ Alice    │ 1000.00  │ SAVINGS           │
│ 2        │ Bob      │ 5000.00  │ CHECKING          │
└──────────┴──────────┴──────────┴───────────────────┘
```

**Pros:**
- ✅ Best performance (single table, no joins)
- ✅ Simple schema
- ✅ Polymorphic queries are fast

**Cons:**
- ❌ Null columns for subclass-specific fields
- ❌ Cannot enforce NOT NULL on subclass fields
- ❌ Table can become wide with many subclasses

### 10.2 JOINED

Each class has its own table, with FK relationships to parent tables.

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Account {
    @Id
    private Long id;
    private String owner;
    private BigDecimal balance;
}

@Entity
public class SavingsAccount extends Account {
    private Double interestRate;
}
```

**Database Schema:**
```
┌──────────────────────┐    ┌──────────────────────┐
│ ACCOUNT              │    │ SAVINGS_ACCOUNT      │
├──────────┬───────────┤    ├──────────┬───────────┤
│ ID       │ OWNER     │    │ ID       │ INTEREST_ │
│ 1        │ Alice     │◄───│ 1        │ RATE      │
│ 2        │ Bob       │    │          │ 0.05      │
│          │ 1000.00   │    └──────────┴───────────┘
└──────────┴───────────┘
```

**Pros:**
- ✅ Normalized schema
- ✅ NOT NULL constraints possible on subclass fields
- ✅ Schema evolution is easier

**Cons:**
- ❌ Requires joins for queries
- ❌ Slower performance
- ❌ Complex queries for deep hierarchies

**Query Example:**
```sql
-- Hibernate generates:
SELECT a.id, a.owner, a.balance, s.interest_rate
FROM account a
LEFT JOIN savings_account s ON a.id = s.id
WHERE a.id = 1
```

### 10.3 TABLE_PER_CLASS

Each concrete class has its own table containing all fields (including inherited ones).

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Account {
    @Id
    private Long id;
    private String owner;
    private BigDecimal balance;
}

@Entity
public class SavingsAccount extends Account {
    private Double interestRate;
}
```

**Database Schema:**
```
┌───────────────────────────────┐    ┌───────────────────────────────────────┐
│ SAVINGS_ACCOUNT               │    │ CHECKING_ACCOUNT                      │
├──────────┬──────────┬──────────┤    ├──────────┬──────────┬──────────────────┤
│ ID       │ OWNER    │ BALANCE  │    │ ID       │ OWNER    │ BALANCE          │
│ 1        │ Alice    │ 1000.00  │    │ 2        │ Bob      │ 5000.00          │
├──────────┼──────────┼──────────┤    ├──────────┼──────────┼──────────────────┤
│ INTEREST_│          │          │    │ OVERDRAFT│          │                  │
│ RATE     │          │          │    │ LIMIT    │          │                  │
│ 0.05     │          │          │    │ 1000.00  │          │                  │
└──────────┴──────────┴──────────┘    └──────────┴──────────┴──────────────────┘
```

**Pros:**
- ✅ No discriminator column needed
- ✅ Each table is self-contained
- ✅ NOT NULL constraints on all fields

**Cons:**
- ❌ Polymorphic queries require UNION (slow)
- ❌ ID generation issues with sequences
- ❌ Not supported by all JPA providers

**Query Example:**
```sql
-- Polymorphic query generates UNION:
SELECT id, owner, balance, interest_rate, NULL as overdraft_limit
FROM savings_account
UNION ALL
SELECT id, owner, balance, NULL as interest_rate, overdraft_limit
FROM checking_account
```

### 10.4 Strategy Comparison

| Aspect | SINGLE_TABLE | JOINED | TABLE_PER_CLASS |
|--------|-------------|--------|----------------|
| **Performance** | Best (no joins) | Poor (joins required) | Poor for polymorphic queries |
| **Schema** | Denormalized | Normalized | Denormalized per class |
| **NOT NULL** | Not on subclass fields | Yes, on subclass fields | Yes, on all fields |
| **Polymorphic Queries** | Fast | Slower (joins) | Slowest (UNION) |
| **Table Width** | Can get wide | Normal tables | Normal tables |
| **Schema Evolution** | Harder (add columns) | Easier (add tables) | Easier (add tables) |
| **ID Generation** | Any strategy | Any strategy | Only TABLE or IDENTITY |

---

## 11. Locking Mechanisms

### 11.1 Optimistic Locking

Optimistic locking assumes conflicts are rare. It uses version numbers or timestamps to detect concurrent modifications.

**Theoretical Foundation:** Optimistic locking implements the **optimistic concurrency control** pattern from database theory. It allows concurrent reads and writes, detecting conflicts at commit time rather than preventing them upfront. This trades off the possibility of conflicts for higher concurrency and performance.

#### Version-Based Locking (Recommended)

```java
@Entity
public class Product {
    @Id
    private Long id;
    
    private String name;
    private Integer stock;
    
    @Version  // ← Adds a version column
    private Long version;
}
```

**Database Schema:**
```
┌──────────────────────────────────────────┐
│ PRODUCT                                  │
├──────────┬──────────┬──────────┬─────────┤
│ ID       │ NAME     │ STOCK    │ VERSION │
├──────────┼──────────┼──────────┼─────────┤
│ 1        │ Widget   │ 100      │ 0       │
└──────────┴──────────┴──────────┴─────────┘
```

**How It Works:**

```java
// Thread 1
Product p1 = em.find(Product.class, 1L);  // version = 0
p1.setStock(95);
em.flush();  // UPDATE ... SET version = 1 WHERE id = 1 AND version = 0

// Thread 2 (concurrent)
Product p2 = em.find(Product.class, 1L);  // version = 0
p2.setStock(90);
em.flush();  // UPDATE ... SET version = 1 WHERE id = 1 AND version = 0
// ❌ No rows affected! Thread 1 already updated to version 1
// ❌ Throws OptimisticLockException
```

**SQL Generated:**
```sql
UPDATE product 
SET name = ?, stock = ?, version = version + 1 
WHERE id = ? AND version = ?
```

**Internal Implementation:**

When you use `@Version`, Hibernate:

1. **At load time**: Reads the current version value from the database
2. **In Persistence Context**: Stores the version in the entity's snapshot
3. **At flush time**: Generates UPDATE with `WHERE version = ?` clause
4. **After UPDATE**: Checks affected rows count
   - If 1 row affected → Success, version incremented
   - If 0 rows affected → Conflict detected, throws `OptimisticLockException`

**Version Field Types:**

```java
@Version
private Long version;              // Most common - numeric sequence

@Version
private Integer version;           // Also works, but can overflow

@Version
private short version;             // Not recommended - easily overflows

@Version
private Timestamp version;         // Timestamp-based (less common)
```

**Best Practice:** Use `Long` for version to avoid overflow in high-write scenarios.

#### Timestamp-Based Locking

```java
@Entity
public class Product {
    @Id
    private Long id;
    
    @Version
    @Temporal(TemporalType.TIMESTAMP)
    private Date lastModified;
}
```

**Pros:**
- Provides audit information (when entity was last modified)
- Human-readable

**Cons:**
- Clock synchronization issues across servers
- Precision limitations (millisecond conflicts possible)
- Not recommended for high-concurrency systems

**When to use:** Only when you need the timestamp for audit purposes AND conflict detection is a secondary concern.

#### Manual Optimistic Locking

```java
Product product = em.find(Product.class, 1L, LockModeType.OPTIMISTIC);
// or
em.lock(product, LockModeType.OPTIMISTIC);
```

**When to use manual optimistic locking:**
- When you don't want a permanent `@Version` field
- When you need selective optimistic locking on specific operations
- When working with legacy schemas that can't be modified

**Lock Modes:**

| Lock Mode | Behavior | Use Case |
|-----------|----------|----------|
| **OPTIMISTIC** | Read lock, checks version on commit | Read-heavy, conflict detection |
| **OPTIMISTIC_FORCE_INCREMENT** | Increments version even on read | Force version bump |
| **READ** (deprecated) | Same as OPTIMISTIC | Legacy compatibility |
| **WRITE** (deprecated) | Same as OPTIMISTIC_FORCE_INCREMENT | Legacy compatibility |

**OPTIMISTIC_FORCE_INCREMENT Example:**

```java
@Transactional
public void markProductForReview(Long productId) {
    Product product = em.find(Product.class, productId, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
    // Even though we're just reading, version increments
    // Useful to signal "this entity was touched"
    reviewService.queueForReview(product);
}
```

**Why use OPTIMISTIC_FORCE_INCREMENT:**
- Signal that an entity was accessed even if not modified
- Prevent concurrent modifications during read-only operations
- Implement "soft locks" by version bumping

### 11.2 Pessimistic Locking

Pessimistic locking assumes conflicts are likely. It acquires database locks to prevent concurrent modifications.

**Theoretical Foundation:** Pessimistic locking implements the **pessimistic concurrency control** pattern. It prevents conflicts by acquiring locks before accessing data, ensuring exclusive access. This trades off concurrency for consistency guarantees.

#### Lock Modes

```java
// PESSIMISTIC_READ - Shared lock (SELECT FOR SHARE)
em.find(Product.class, 1L, LockModeType.PESSIMISTIC_READ);

// PESSIMISTIC_WRITE - Exclusive lock (SELECT FOR UPDATE)
em.find(Product.class, 1L, LockModeType.PESSIMISTIC_WRITE);

// PESSIMISTIC_FORCE_INCREMENT - Exclusive + version increment
em.find(Product.class, 1L, LockModeType.PESSIMISTIC_FORCE_INCREMENT);
```

**Detailed Lock Mode Behavior:**

**PESSIMISTIC_READ (Shared Lock):**
- Allows multiple transactions to read the same data
- Prevents other transactions from modifying the locked data
- Other transactions can also acquire PESSIMISTIC_READ locks
- Use case: Read-heavy scenarios where you need to ensure data doesn't change while reading

**PESSIMISTIC_WRITE (Exclusive Lock):**
- Prevents other transactions from reading or modifying the locked data
- Only one transaction can hold this lock at a time
- Use case: Update scenarios where you need exclusive access

**PESSIMISTIC_FORCE_INCREMENT:**
- Acquires exclusive lock AND increments version field
- Even if you don't modify other fields, version increments
- Use case: When you need to signal that an entity was accessed

**SQL Generated by Database:**

```sql
-- PostgreSQL
-- PESSIMISTIC_READ
SELECT * FROM product WHERE id = 1 FOR SHARE

-- PESSIMISTIC_WRITE
SELECT * FROM product WHERE id = 1 FOR UPDATE

-- PESSIMISTIC_WRITE with SKIP LOCKED (PostgreSQL 9.5+)
SELECT * FROM product WHERE id = 1 FOR UPDATE SKIP LOCKED


-- MySQL
-- PESSIMISTIC_READ (MySQL uses FOR SHARE in 8.0+, FOR UPDATE LOCK IN SHARE MODE in older)
SELECT * FROM product WHERE id = 1 FOR SHARE

-- PESSIMISTIC_WRITE
SELECT * FROM product WHERE id = 1 FOR UPDATE

-- PESSIMISTIC_WRITE with NOWAIT
SELECT * FROM product WHERE id = 1 FOR UPDATE NOWAIT


-- Oracle
-- PESSIMISTIC_READ
SELECT * FROM product WHERE id = 1 FOR UPDATE

-- Oracle doesn't have true shared locks, FOR UPDATE is exclusive
-- Use PESSIMISTIC_WRITE for both read and write scenarios in Oracle


-- SQL Server
-- PESSIMISTIC_READ
SELECT * FROM product WHERE id = 1 WITH (UPDLOCK, HOLDLOCK)

-- PESSIMISTIC_WRITE
SELECT * FROM product WHERE id = 1 WITH (XLOCK, HOLDLOCK)
```

**Database-Specific Considerations:**

| Database | PESSIMISTIC_READ Support | Notes |
|----------|-------------------------|-------|
| PostgreSQL | ✅ `FOR SHARE` | Full support, true shared locks |
| MySQL 8.0+ | ✅ `FOR SHARE` | Full support |
| MySQL < 8.0 | ⚠️ `FOR UPDATE LOCK IN SHARE MODE` | Deprecated syntax |
| Oracle | ❌ No true shared locks | Use PESSIMISTIC_WRITE for both |
| SQL Server | ✅ `WITH (UPDLOCK, HOLDLOCK)` | Complex locking syntax |

#### Lock Timeout

Configure how long to wait for a lock:

```java
// Set timeout in seconds
Map<String, Object> hints = new HashMap<>();
hints.put("javax.persistence.lock.timeout", 5); // 5 seconds
em.find(Product.class, 1L, LockModeType.PESSIMISTIC_WRITE, hints);

// Or via query hints
Query query = em.createQuery("SELECT p FROM Product p WHERE p.id = :id");
query.setParameter("id", 1L);
query.setLockMode(LockModeType.PESSIMISTIC_WRITE);
query.setHint("javax.persistence.lock.timeout", 5);
```

**Timeout Behavior:**

```java
// Scenario: Thread 1 holds lock on product 1
// Thread 2 tries to acquire lock with timeout

@Transactional
public void updateWithTimeout(Long productId) {
    try {
        Product product = em.find(
            Product.class, 
            productId, 
            LockModeType.PESSIMISTIC_WRITE,
            Collections.singletonMap("javax.persistence.lock.timeout", 3) // 3 seconds
        );
        product.setStock(100);
    } catch (PessimisticLockException | LockTimeoutException e) {
        // Lock not acquired within timeout
        throw new BusinessException("Product is locked, please try again");
    }
}
```

**Special Timeout Values:**

```java
// Wait indefinitely (default)
hints.put("javax.persistence.lock.timeout", 0);

// No wait - fail immediately if lock not available
hints.put("javax.persistence.lock.timeout", -1);
// Equivalent to FOR UPDATE NOWAIT in PostgreSQL/MySQL
```

#### Configuration

```properties
# application.properties
spring.jpa.properties.javax.persistence.lock.timeout=5000  # milliseconds
spring.jpa.properties.hibernate.jdbc.time_zone=UTC
```

**Hibernate-Specific Properties:**

```properties
# Configure default lock timeout globally
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true

# For PostgreSQL-specific hints
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

#### Pessimistic Locking in Queries

You can apply pessimistic locks to JPQL queries:

```java
@Transactional
public void updateProductPrices() {
    // Lock multiple rows
    Query query = em.createQuery(
        "SELECT p FROM Product p WHERE p.category = :category"
    );
    query.setParameter("category", "ELECTRONICS");
    query.setLockMode(LockModeType.PESSIMISTIC_WRITE);
    query.setHint("javax.persistence.lock.timeout", 10);
    
    List<Product> products = query.getResultList();
    products.forEach(p -> p.setPrice(p.getPrice() * 1.1));
}
```

**Lock Scope:**

Pessimistic locks are held until:
- Transaction commits
- Transaction rolls back
- Transaction timeout occurs
- Connection is closed

**Important:** Locks are NOT released when you call `em.clear()` or `em.detach()`. They're tied to the transaction, not the Persistence Context.

### 11.3 Optimistic vs Pessimistic Comparison

| Aspect | Optimistic | Pessimistic |
|--------|------------|-------------|
| **Conflict Probability** | Low (assumes rare) | High (assumes likely) |
| **Performance** | Better (no DB locks) | Worse (blocks other transactions) |
| **Database Load** | Lower | Higher |
| **User Experience** | Retry on conflict | Wait for lock |
| **Best For** | Read-heavy, web applications | Write-heavy, batch processing |
| **Error Handling** | OptimisticLockException (retry) | PessimisticLockException / timeout |
| **Implementation** | @Version field | LockModeType on find/lock |
| **Concurrency** | High (no blocking) | Low (blocking) |
| **Scalability** | Better (scales horizontally) | Worse (lock contention) |
| **Complexity** | Simpler (automatic) | More complex (manual lock management) |
| **Deadlock Risk** | None | Possible (requires ordering) |

**Performance Comparison:**

```
Throughput (transactions/second):

Optimistic:  ████████████████████████████████ (highest)
Pessimistic: ██████████████████ (lower due to blocking)

Latency:

Optimistic:  ████ (low, no waiting)
Pessimistic: ███████████████ (higher, may wait for locks)
```

**Decision Framework:**

```
Choose Optimistic Locking when:
├─ Conflicts are rare (< 5% of transactions)
├─ Read operations significantly outnumber writes
├─ User-facing web applications
├─ High concurrency requirements
├─ Horizontal scaling needed
├─ Short transaction duration
└─ Can handle retry logic

Choose Pessimistic Locking when:
├─ Conflicts are likely (> 20% of transactions)
├─ Write-heavy workloads
├─ Batch processing jobs
├─ Financial/accounting operations
├─ Inventory management
├─ Long-running transactions
├─ Cannot tolerate retries
└─ Need guaranteed consistency
```

**Real-World Scenario Examples:**

**Scenario 1: E-commerce Product Page (Use Optimistic)**

```java
@Entity
public class Product {
    @Id
    private Long id;
    
    @Version
    private Long version;
    
    private Integer stock;
    private BigDecimal price;
}

// Multiple users viewing product concurrently
// Conflicts rare (only when purchasing)
@Transactional
public void purchaseProduct(Long productId, int quantity) {
    Product product = em.find(Product.class, productId);
    if (product.getStock() >= quantity) {
        product.setStock(product.getStock() - quantity);
        // OptimisticLockException if concurrent purchase
    }
}
```

**Why Optimistic:**
- Thousands of reads per second
- Purchases are relatively rare compared to views
- Users can retry if conflict occurs

**Scenario 2: Inventory Batch Update (Use Pessimistic)**

```java
@Transactional
public void batchUpdateInventory(List<InventoryUpdate> updates) {
    for (InventoryUpdate update : updates) {
        // Lock each product to prevent concurrent updates
        Product product = em.find(
            Product.class, 
            update.getProductId(), 
            LockModeType.PESSIMISTIC_WRITE
        );
        product.setStock(update.getNewStock());
    }
}
```

**Why Pessimistic:**
- High conflict probability (batch job vs concurrent operations)
- Cannot afford retries (must complete once)
- Write-heavy operation

**Scenario 3: Financial Transfer (Use Pessimistic)**

```java
@Transactional
public void transferFunds(Long fromAccountId, Long toAccountId, BigDecimal amount) {
    Account from = em.find(
        Account.class, 
        fromAccountId, 
        LockModeType.PESSIMISTIC_WRITE
    );
    Account to = em.find(
        Account.class, 
        toAccountId, 
        LockModeType.PESSIMISTIC_WRITE
    );
    
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
}
```

**Why Pessimistic:**
- Financial operations require absolute consistency
- Cannot tolerate conflicts or retries
- High value of each transaction

**Scenario 4: User Profile Update (Use Optimistic)**

```java
@Entity
public class UserProfile {
    @Id
    private Long id;
    
    @Version
    private Long version;
    
    private String name;
    private String email;
}

@Transactional
public void updateProfile(Long userId, ProfileUpdateDto dto) {
    UserProfile profile = em.find(UserProfile.class, userId);
    profile.setName(dto.getName());
    profile.setEmail(dto.getEmail());
    // OptimisticLockException if user updates from two devices
}
```

**Why Optimistic:**
- Users rarely update profile simultaneously
- Can show "concurrent edit detected" message
- Low conflict probability

### 11.4 Handling Lock Exceptions

```java
@Transactional
public void updateStockWithRetry(Long productId, int newStock) {
    int maxRetries = 3;
    int attempt = 0;
    
    while (attempt < maxRetries) {
        try {
            Product product = em.find(Product.class, productId);
            product.setStock(newStock);
            em.flush();
            return; // Success
        } catch (OptimisticLockException e) {
            attempt++;
            if (attempt >= maxRetries) {
                throw new RuntimeException("Max retries exceeded", e);
            }
            // Reload and retry
            em.clear(); // Clear PC to reload fresh state
        }
    }
}
```

**Advanced Retry Strategy with Exponential Backoff:**

```java
@Transactional
public void updateWithExponentialBackoff(Long productId, int newStock) {
    int maxRetries = 5;
    long baseDelayMs = 100; // Start with 100ms
    int attempt = 0;
    
    while (attempt < maxRetries) {
        try {
            Product product = em.find(Product.class, productId);
            product.setStock(newStock);
            em.flush();
            return;
        } catch (OptimisticLockException e) {
            attempt++;
            if (attempt >= maxRetries) {
                throw new ConcurrentModificationException(
                    "Failed after " + maxRetries + " retries", e
                );
            }
            
            // Exponential backoff: 100ms, 200ms, 400ms, 800ms, 1600ms
            long delayMs = baseDelayMs * (long) Math.pow(2, attempt - 1);
            try {
                Thread.sleep(delayMs);
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Interrupted during retry", ie);
            }
            
            em.clear(); // Clear Persistence Context
        }
    }
}
```

**Handling Pessimistic Lock Timeouts:**

```java
@Transactional
public void updateWithPessimisticLock(Long productId, int newStock) {
    Map<String, Object> hints = new HashMap<>();
    hints.put("javax.persistence.lock.timeout", 5); // 5 seconds
    
    try {
        Product product = em.find(
            Product.class, 
            productId, 
            LockModeType.PESSIMISTIC_WRITE,
            hints
        );
        product.setStock(newStock);
    } catch (PessimisticLockException | LockTimeoutException e) {
        throw new BusinessException(
            "Product is currently being modified by another transaction. Please try again.",
            e
        );
    }
}
```

**User-Friendly Error Handling:**

```java
public class LockExceptionHandler {
    
    @Transactional
    public void safeUpdate(Long productId, int newStock) {
        try {
            updateWithRetry(productId, newStock);
        } catch (OptimisticLockException e) {
            throw new UserFacingException(
                "This record was modified by another user. Please refresh and try again."
            );
        } catch (PessimisticLockException e) {
            throw new UserFacingException(
                "This record is currently locked. Please wait a moment and try again."
            );
        }
    }
}
```

### 11.5 Deadlock Handling

Deadlocks occur when two or more transactions wait indefinitely for each other's locks.

**Deadlock Example:**

```java
// Transaction 1
@Transactional
public void transfer1(Long fromId, Long toId) {
    Account from = em.find(Account.class, fromId, LockModeType.PESSIMISTIC_WRITE); // Locks A
    Account to = em.find(Account.class, toId, LockModeType.PESSIMISTIC_WRITE);   // Waits for B
    // ... transfer logic
}

// Transaction 2 (running concurrently)
@Transactional
public void transfer2(Long fromId, Long toId) {
    Account to = em.find(Account.class, toId, LockModeType.PESSIMISTIC_WRITE);   // Locks B
    Account from = em.find(Account.class, fromId, LockModeType.PESSIMISTIC_WRITE); // Waits for A
    // ... transfer logic
}

// Deadlock! T1 holds A, waits for B. T2 holds B, waits for A.
```

**Preventing Deadlocks with Consistent Lock Ordering:**

```java
@Service
public class AccountService {
    
    // Always lock accounts in the same order (by ID)
    @Transactional
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
        // Ensure consistent ordering: smaller ID first
        Long firstId = Math.min(fromId, toId);
        Long secondId = Math.max(fromId, toId);
        
        Account first = em.find(Account.class, firstId, LockModeType.PESSIMISTIC_WRITE);
        Account second = em.find(Account.class, secondId, LockModeType.PESSIMISTIC_WRITE);
        
        // Now perform the transfer
        if (fromId.equals(firstId)) {
            first.setBalance(first.getBalance().subtract(amount));
            second.setBalance(second.getBalance().add(amount));
        } else {
            second.setBalance(second.getBalance().subtract(amount));
            first.setBalance(first.getBalance().add(amount));
        }
    }
}
```

**Handling Deadlocks with Retry:**

```java
@Transactional
public void transferWithDeadlockRetry(Long fromId, Long toId, BigDecimal amount) {
    int maxRetries = 3;
    int attempt = 0;
    
    while (attempt < maxRetries) {
        try {
            transferFunds(fromId, toId, amount);
            return;
        } catch (PessimisticLockException e) {
            if (e.getMessage() != null && e.getMessage().contains("deadlock")) {
                attempt++;
                if (attempt >= maxRetries) {
                    throw new RuntimeException("Deadlock persisted after retries", e);
                }
                // Wait before retry
                try {
                    Thread.sleep(100 * attempt);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Interrupted", ie);
                }
                em.clear();
            } else {
                throw e; // Not a deadlock, re-throw
            }
        }
    }
}
```

**Database Deadlock Detection:**

Most databases automatically detect deadlocks and kill one transaction:

```java
@Transactional
public void handleDeadlock() {
    try {
        // Operation that might deadlock
    } catch (PessimisticLockException e) {
        // Check if it's a deadlock
        if (isDeadlock(e)) {
            // Retry the transaction
            log.warn("Deadlock detected, retrying...");
        } else {
            throw e;
        }
    }
}

private boolean isDeadlock(PessimisticLockException e) {
    String message = e.getMessage();
    return message != null && 
           (message.contains("deadlock") || 
            message.contains("Deadlock") ||
            message.contains("1213")); // MySQL deadlock error code
}
```

### 11.6 Lock Scope and Granularity

**Row-Level Locking (Default):**

```java
// Locks only the specific row
em.find(Product.class, 1L, LockModeType.PESSIMISTIC_WRITE);
// SQL: SELECT ... FROM product WHERE id = 1 FOR UPDATE
```

**Table-Level Locking (Not directly supported by JPA, use native SQL):**

```java
@Transactional
public void lockTableForBatchUpdate() {
    // Use native SQL for table-level lock
    em.createNativeQuery("LOCK TABLE product IN EXCLUSIVE MODE").executeUpdate();
    
    // Now safe to perform batch operations
    List<Product> products = em.createQuery("SELECT p FROM Product p", Product.class).getResultList();
    products.forEach(p -> p.setPrice(p.getPrice() * 1.1));
}
```

**Range Locking (for phantom read prevention):**

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferWithRangeLock(Long fromAccountId, Long toAccountId, BigDecimal amount) {
    // SERIALIZABLE isolation provides range locking
    // Prevents new accounts from being inserted in the range
    Account from = em.find(Account.class, fromAccountId);
    Account to = em.find(Account.class, toAccountId);
    
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
}
```

**Lock Granularity Trade-offs:**

| Granularity | Concurrency | Overhead | Use Case |
|-------------|-------------|----------|----------|
| **Table** | Lowest | Lowest | Bulk operations, maintenance |
| **Page** | Medium | Medium | Rarely used directly |
| **Row** | High | Medium | Most common |
| **Column** | Highest | Highest | Rare, specialized |

### 11.7 Common Pitfalls and Best Practices

**Pitfall 1: Forgetting @Version Field**

```java
// BAD: No version field, no optimistic locking
@Entity
public class Product {
    @Id
    private Long id;
    private Integer stock;
    // No @Version - concurrent updates silently overwrite each other
}
```

**Correct:**

```java
// GOOD: Always add @Version for entities that are updated concurrently
@Entity
public class Product {
    @Id
    private Long id;
    private Integer stock;
    
    @Version
    private Long version;
}
```

**Pitfall 2: Holding Locks Too Long**

```java
// BAD: Lock held during I/O operation
@Transactional
public void updateAndNotify(Long productId, int newStock) {
    Product product = em.find(Product.class, productId, LockModeType.PESSIMISTIC_WRITE);
    product.setStock(newStock);
    
    // BAD: Lock held while sending email (could take seconds)
    emailService.sendNotification(product);
}
```

**Correct:**

```java
// GOOD: Split transaction and I/O
@Transactional
public void updateProduct(Long productId, int newStock) {
    Product product = em.find(Product.class, productId, LockModeType.PESSIMISTIC_WRITE);
    product.setStock(newStock);
    // Transaction commits here, lock released
}

public void updateAndNotify(Long productId, int newStock) {
    updateProduct(productId, newStock);  // Lock released
    Product product = em.find(Product.class, productId);
    emailService.sendNotification(product);  // I/O outside lock
}
```

**Pitfall 3: Lock Ordering Inconsistency**

```java
// BAD: Inconsistent lock ordering causes deadlocks
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Account from = em.find(Account.class, fromId, LockModeType.PESSIMISTIC_WRITE);
    Account to = em.find(Account.class, toId, LockModeType.PESSIMISTIC_WRITE);
    // If another transaction transfers in reverse order, deadlock!
}
```

**Correct:**

```java
// GOOD: Always lock in consistent order
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Long firstId = Math.min(fromId, toId);
    Long secondId = Math.max(fromId, toId);
    
    Account first = em.find(Account.class, firstId, LockModeType.PESSIMISTIC_WRITE);
    Account second = em.find(Account.class, secondId, LockModeType.PESSIMISTIC_WRITE);
    
    // Perform transfer
}
```

**Pitfall 4: Not Handling Lock Exceptions**

```java
// BAD: Lock exceptions not handled
@Transactional
public void updateStock(Long productId, int newStock) {
    Product product = em.find(Product.class, productId);
    product.setStock(newStock);
    // OptimisticLockException will propagate to user as 500 error
}
```

**Correct:**

```java
// GOOD: Handle lock exceptions gracefully
@Transactional
public void updateStock(Long productId, int newStock) {
    try {
        Product product = em.find(Product.class, productId);
        product.setStock(newStock);
    } catch (OptimisticLockException e) {
        throw new UserFacingException(
            "Product was modified by another user. Please refresh and try again."
        );
    }
}
```

**Pitfall 5: Using Pessimistic Locking When Not Needed**

```java
// BAD: Unnecessary pessimistic locking
@Transactional
public void viewProduct(Long productId) {
    // Reading with pessimistic lock blocks other readers
    Product product = em.find(Product.class, productId, LockModeType.PESSIMISTIC_WRITE);
    return product;
}
```

**Correct:**

```java
// GOOD: Use optimistic for reads
@Transactional(readOnly = true)
public Product viewProduct(Long productId) {
    // No lock, high concurrency
    return em.find(Product.class, productId);
}
```

**Best Practices Summary:**

1. **Default to optimistic locking** - Use `@Version` for most entities
2. **Keep transactions short** - Minimize lock holding time
3. **Use consistent lock ordering** - Always acquire locks in the same order
4. **Handle lock exceptions gracefully** - Provide user-friendly error messages
5. **Use pessimistic locking sparingly** - Only when conflicts are likely
6. **Set appropriate timeouts** - Avoid indefinite waits
7. **Monitor lock contention** - Use database monitoring tools
8. **Test for deadlocks** - Use concurrent testing to identify deadlock scenarios
9. **Document locking strategy** - Make it clear which entities use which locking
10. **Consider isolation levels** - Sometimes SERIALIZABLE is simpler than manual locking

---

## 12. Cascade Operations

Cascade operations define how operations on a parent entity should propagate to its associated entities.

### 12.1 Cascade Types

```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items;
}
```

| Cascade Type | Operations Propagated | Use Case |
|---------------|----------------------|----------|
| **PERSIST** | `em.persist()` | Save parent + children together |
| **MERGE** | `em.merge()` | Reattach parent + children |
| **REMOVE** | `em.remove()` | Delete parent + children |
| **REFRESH** | `em.refresh()` | Reload parent + children from DB |
| **DETACH** | `em.detach()` | Detach parent + children from PC |
| **ALL** | All above operations | Full lifecycle management |

### 12.2 Cascade Examples

#### Cascade PERSIST

```java
Order order = new Order();
order.setId(1L);

OrderItem item = new OrderItem();
item.setProduct("Widget");
item.setOrder(order);  // Set relationship

order.getItems().add(item);

em.persist(order);  // Cascades to item → both saved
// Without cascade: item would not be saved
```

#### Cascade REMOVE

```java
Order order = em.find(Order.class, 1L);
em.remove(order);  // Cascades to items → all items deleted too
// Without cascade: items would become orphans
```

#### Cascade MERGE

```java
// Detached order with modified items
Order detachedOrder = ...; // from HTTP request

em.merge(detachedOrder);  // Reattaches order AND items
// Without cascade: items would remain detached
```

### 12.3 Orphan Removal

`orphanRemoval = true` automatically deletes child entities when removed from the collection:

```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @OneToMany(mappedBy = "order", 
               cascade = CascadeType.ALL, 
               orphanRemoval = true)  // ← Key setting
    private List<OrderItem> items;
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
        // orphanRemoval causes DELETE for item
    }
}
```

**Without orphanRemoval:**
```java
order.getItems().remove(item);
// Item still exists in DB, just FK set to null
// → Orphan row
```

**With orphanRemoval:**
```java
order.getItems().remove(item);
// Item is DELETED from DB
// → No orphans
```

### 12.4 Cascade vs orphanRemoval

| Aspect | Cascade REMOVE | orphanRemoval |
|--------|----------------|---------------|
| **Trigger** | `em.remove(parent)` | Remove from collection |
| **Scope** | All children | Specific child |
| **Use Case** | Delete entire tree | Remove specific node |
| **Can Combine** | Yes | Yes (with cascade) |

### 12.5 Common Cascade Patterns

```java
// Pattern 1: Parent-Child with full lifecycle
@Entity
public class Order {
    @OneToMany(mappedBy = "order", 
               cascade = CascadeType.ALL, 
               orphanRemoval = true)
    private List<OrderItem> items;
}

// Pattern 2: Parent-Child with no delete cascade
@Entity
public class Department {
    @OneToMany(mappedBy = "department", 
               cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    private List<Employee> employees;
    // Employees can exist without department
}

// Pattern 3: Reference only (no cascade)
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
    // Customer exists independently
}
```

---

## 13. Query Optimization

### 13.1 The N+1 Problem

The N+1 problem occurs when lazy loading triggers N additional queries for N parent entities.

**Theoretical Foundation:** The N+1 problem is a classic ORM performance anti-pattern that stems from the **object-relational impedance mismatch**. In SQL, you can fetch related data in a single query using JOINs. In object-oriented programming, associations are navigable object references, and lazy loading defers fetching until access. When you iterate over a collection and access lazy associations, each access triggers a separate query, leading to 1 + N queries total.

**Why It Happens:**

1. **Lazy Loading by Default**: JPA uses `FetchType.LAZY` by default for `@ManyToOne` and `@OneToMany` associations
2. **Transparent Access**: When you access a lazy association, Hibernate automatically executes a query
3. **No Batching**: Each lazy access results in a separate SQL query
4. **Developer Blindness**: The queries happen transparently, making the problem hard to spot

```java
// Query returns 10 orders
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class)
                        .getResultList();

// For each order, accessing customer triggers a query
for (Order order : orders) {
    Customer customer = order.getCustomer();  // ← N additional queries
}
// Total: 1 (orders) + 10 (customers) = 11 queries
```

**SQL Execution Timeline:**

```
Query 1: SELECT * FROM orders  (returns 10 rows)
Query 2: SELECT * FROM customer WHERE id = 1  (for order 1)
Query 3: SELECT * FROM customer WHERE id = 2  (for order 2)
Query 4: SELECT * FROM customer WHERE id = 1  (for order 3, same customer)
Query 5: SELECT * FROM customer WHERE id = 3  (for order 4)
...
Query 11: SELECT * FROM customer WHERE id = 5  (for order 10)

Total: 11 queries for 10 orders
```

**Performance Impact:**

```
N+1 Impact on Query Count:
┌─────────────┬──────────┬─────────────┬────────────────┐
│ N (Rows)    │ Queries  │ Latency (ms)│ Network Round-trips │
├─────────────┼──────────┼─────────────┼────────────────┤
│ 10          │ 11       │ ~50ms       │ 11             │
│ 100         │ 101      │ ~500ms      │ 101            │
│ 1000        │ 1001     │ ~5000ms     │ 1001           │
│ 10000       │ 10001    │ ~50000ms    │ 10001          │
└─────────────┴──────────┴─────────────┴────────────────┘

Note: Each query has ~1-5ms network latency + DB processing time
```

**Real-World Example:**

```java
// E-commerce scenario - displaying order history
@GetMapping("/orders")
public List<OrderDto> getUserOrders(@PathVariable Long userId) {
    List<Order> orders = orderRepository.findByUserId(userId);  // 1 query
    
    List<OrderDto> dtos = new ArrayList<>();
    for (Order order : orders) {
        OrderDto dto = new OrderDto();
        dto.setId(order.getId());
        dto.setStatus(order.getStatus());
        
        // N+1 problem: Each customer access triggers a query
        dto.setCustomerName(order.getCustomer().getName());  // N queries
        
        // Another N+1: Each items access triggers queries
        dto.setItemCount(order.getItems().size());  // Could be M queries
        
        dtos.add(dto);
    }
    return dtos;
}

// If user has 50 orders:
// - 1 query for orders
// - 50 queries for customers
// - 50+ queries for items
// Total: 100+ queries for a simple page load!
```

**Detecting N+1 Problems:**

**1. SQL Logging:**

```properties
# Enable SQL logging in application.properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

**2. Statistics:**

```java
// Enable Hibernate statistics
spring.jpa.properties.hibernate.generate_statistics=true

// Check statistics after execution
Statistics stats = em.getEntityManagerFactory().getStatistics();
long queryCount = stats.getQueryExecutionCount();
long entityFetchCount = stats.getEntityFetchCount();
```

**3. Monitoring Tools:**
- Hibernate Statistics
- Database query logs
- APM tools (New Relic, Datadog, etc.)
- Custom interceptors

### 13.2 Solving N+1 with JOIN FETCH

**Theoretical Foundation:** JOIN FETCH is a JPA feature that allows you to eagerly fetch associations in the same query as the parent entity. It translates to a SQL JOIN, eliminating the N+1 problem by fetching all data in a single database round-trip.

```java
// Single query with join fetch
String jpql = "SELECT o FROM Order o JOIN FETCH o.customer";
List<Order> orders = em.createQuery(jpql, Order.class)
                        .getResultList();

// No additional queries - customers already loaded
```

**SQL Generated:**
```sql
SELECT o.*, c.*
FROM order o
LEFT JOIN customer c ON o.customer_id = c.id
```

**How JOIN FETCH Works:**

1. **SQL Translation**: JPA translates `JOIN FETCH` to a SQL LEFT JOIN
2. **Result Set Processing**: Hibernate processes the joined result set and builds the object graph
3. **Persistence Context Population**: Both parent and child entities are loaded into the Persistence Context
4. **Association Wiring**: Hibernate automatically wires the associations based on the JOIN

**Multiple JOIN FETCH:**

```java
// Fetch multiple associations in one query
String jpql = "SELECT o FROM Order o " +
              "JOIN FETCH o.customer " +
              "JOIN FETCH o.items " +
              "JOIN FETCH o.items.product";
List<Order> orders = em.createQuery(jpql, Order.class).getResultList();
```

**SQL Generated:**
```sql
SELECT o.*, c.*, i.*, p.*
FROM order o
LEFT JOIN customer c ON o.customer_id = c.id
LEFT JOIN order_item i ON o.id = i.order_id
LEFT JOIN product p ON i.product_id = p.id
```

**Important: DISTINCT for Duplicate Results**

When using JOIN FETCH with collections, you may get duplicate parent entities in the result due to the SQL JOIN. Use DISTINCT to eliminate duplicates:

```java
// Without DISTINCT: May return duplicate orders
String jpql = "SELECT o FROM Order o JOIN FETCH o.items";
List<Order> orders = em.createQuery(jpql, Order.class).getResultList();
// If order has 3 items, you'll get the same order 3 times!

// With DISTINCT: Returns unique orders
String jpql = "SELECT DISTINCT o FROM Order o JOIN FETCH o.items";
List<Order> orders = em.createQuery(jpql, Order.class).getResultList();
// Returns each order once, with all items loaded
```

**SQL with DISTINCT:**
```sql
SELECT DISTINCT o.*, c.*, i.*
FROM order o
LEFT JOIN customer c ON o.customer_id = c.id
LEFT JOIN order_item i ON o.id = i.order_id
```

**JOIN FETCH vs Regular JOIN:**

```java
// Regular JOIN - only for WHERE clause, doesn't fetch association
String jpql1 = "SELECT o FROM Order o JOIN o.customer c WHERE c.name = :name";
// Customer is NOT loaded, only used for filtering

// JOIN FETCH - loads the association
String jpql2 = "SELECT o FROM Order o JOIN FETCH o.customer c WHERE c.name = :name";
// Customer IS loaded into Persistence Context
```

**Edge Cases and Limitations:**

**1. Cannot JOIN FETCH in subqueries:**

```java
// BAD: JOIN FETCH not allowed in subquery
String jpql = "SELECT o FROM Order o WHERE o.id IN " +
              "(SELECT o2.id FROM Order o2 JOIN FETCH o2.customer)";
// Throws exception
```

**2. Cannot use JOIN FETCH with scrollable results:**

```java
// BAD: Not compatible with scrollable results
Query query = em.createQuery("SELECT o FROM Order o JOIN FETCH o.customer");
query.setHint("org.hibernate.fetchSize", 100);
query.setHint("org.hibernate.readOnly", true);
ScrollableResults results = query.scroll(ScrollMode.FORWARD_ONLY);
// May cause issues
```

**3. Cartesian Product Warning:**

When fetching multiple collections, be aware of the cartesian product:

```java
// Order has many items, each item has many tags
String jpql = "SELECT o FROM Order o " +
              "JOIN FETCH o.items " +
              "JOIN FETCH o.items.tags";
// If order has 10 items and each item has 5 tags
// Result set has 50 rows (10 * 5)
// Hibernate deduplicates, but large result sets impact performance
```

**Performance Considerations:**

```java
// Fetching large collections can be expensive
// If Order has 1000 items:
String jpql = "SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id";
Order order = em.createQuery(jpql, Order.class)
                 .setParameter("id", 1L)
                 .getSingleResult();
// SQL returns 1000 rows, all loaded into memory
// Consider pagination or batch fetching instead
```

**Best Practices:**

1. **Use JOIN FETCH sparingly** - Only for associations you actually need
2. **Always use DISTINCT** with collection JOIN FETCH to avoid duplicates
3. **Avoid deep JOIN FETCH chains** - Can cause cartesian products
4. **Consider batch fetching** for large collections instead of JOIN FETCH
5. **Profile your queries** - Use EXPLAIN to verify SQL execution plans

### 13.3 EntityGraph for Dynamic Fetching

**Theoretical Foundation:** EntityGraph is a JPA 2.1 feature that provides a declarative way to define fetch plans dynamically at runtime. Unlike static `JOIN FETCH` in JPQL (which is baked into the query string), EntityGraph allows you to specify which associations to fetch per query invocation. This provides flexibility to optimize fetching based on use case without writing multiple query methods.

**Types of EntityGraph:**

1. **Named EntityGraph** - Defined as annotation on entity
2. **Dynamic EntityGraph** - Created programmatically at runtime
3. **Fetch Graph** - Only specified attributes are fetched (others lazy)
4. **Load Graph** - Specified attributes are fetched, others follow default fetch type

**Named EntityGraph:**

```java
@Entity
@NamedEntityGraph(
    name = "Order.withCustomerAndItems",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "items")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "items",
            attributeNodes = @NamedAttributeNode("product")
        )
    }
)
public class Order {
    @Id
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
    
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

**Usage with Named EntityGraph:**

```java
EntityGraph<?> graph = em.getEntityGraph("Order.withCustomerAndItems");
Map<String, Object> hints = new HashMap<>();
hints.put("javax.persistence.fetchgraph", graph);

Order order = em.find(Order.class, 1L, hints);
// Customer and items loaded in one query
```

**Dynamic EntityGraph (Programmatic):**

```java
// Create graph at runtime
EntityGraph<Order> graph = em.createEntityGraph(Order.class);
graph.addAttributeNodes("customer");

// Add subgraph for nested associations
Subgraph<OrderItem> itemsSubgraph = graph.addSubgraph("items");
itemsSubgraph.addAttributeNodes("product");

Map<String, Object> hints = new HashMap<>();
hints.put("javax.persistence.fetchgraph", graph);

List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class)
                         .setHint("javax.persistence.fetchgraph", graph)
                         .getResultList();
```

**Fetch Graph vs Load Graph:**

```java
// FETCH GRAPH: Only specified attributes are fetched
EntityGraph<Order> fetchGraph = em.createEntityGraph(Order.class);
fetchGraph.addAttributeNodes("customer");
hints.put("javax.persistence.fetchgraph", fetchGraph);
// Only customer is fetched, everything else is lazy (even if EAGER by default)

// LOAD GRAPH: Specified attributes are fetched, others follow default fetch type
EntityGraph<Order> loadGraph = em.createEntityGraph(Order.class);
loadGraph.addAttributeNodes("customer");
hints.put("javax.persistence.loadgraph", loadGraph);
// Customer is fetched, other associations follow their FetchType (EAGER/LAZY)
```

**Complex Subgraph Example:**

```java
@Entity
@NamedEntityGraph(
    name = "Order.full",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "items")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "items",
            attributeNodes = {
                @NamedAttributeNode("product"),
                @NamedAttributeNode(value = "product", subgraph = "product")
            }
        ),
        @NamedSubgraph(
            name = "product",
            attributeNodes = {
                @NamedAttributeNode("category"),
                @NamedAttributeNode("manufacturer")
            }
        )
    }
)
public class Order {
    @Id
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
    
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> items;
}

@Entity
public class OrderItem {
    @Id
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Product product;
}

@Entity
public class Product {
    @Id
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Manufacturer manufacturer;
}
```

**Usage with Repository (Spring Data JPA):**

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @EntityGraph(attributePaths = {"customer", "items"})
    Order findWithCustomerAndItemsById(Long id);
    
    @EntityGraph(value = "Order.full", type = EntityGraphType.FETCH)
    List<Order> findAllWithFullDetails();
}
```

**Dynamic Fetching Based on Use Case:**

```java
@Service
public class OrderService {
    
    // Lightweight - only order data
    public Order getOrderSummary(Long orderId) {
        return orderRepository.findById(orderId).orElse(null);
    }
    
    // With customer - for order details page
    @EntityGraph(attributePaths = {"customer"})
    public Order getOrderWithCustomer(Long orderId) {
        return orderRepository.findById(orderId).orElse(null);
    }
    
    // Full details - for order editing
    @EntityGraph(attributePaths = {"customer", "items", "items.product"})
    public Order getOrderWithFullDetails(Long orderId) {
        return orderRepository.findById(orderId).orElse(null);
    }
}
```

**EntityGraph vs JOIN FETCH:**

| Aspect | EntityGraph | JOIN FETCH |
|--------|-------------|------------|
| **Flexibility** | High (dynamic at runtime) | Low (static in query) |
| **Reusability** | High (define once, use anywhere) | Low (query-specific) |
| **Type Safety** | High (compile-time checked) | Low (string-based) |
| **Complexity** | Medium (annotation + code) | Simple (in JPQL) |
| **Nested Fetching** | Easy (subgraphs) | Verbose (multiple JOIN FETCH) |
| **Best For** | Dynamic fetching requirements | Static, well-defined queries |

**Performance Considerations:**

```java
// EntityGraph translates to JOIN FETCH internally
// The SQL generated is similar to JOIN FETCH
// However, EntityGraph provides more control over what's fetched

// Example: Conditional fetching
public List<Order> findOrders(boolean includeCustomer, boolean includeItems) {
    EntityGraph<Order> graph = em.createEntityGraph(Order.class);
    
    if (includeCustomer) {
        graph.addAttributeNodes("customer");
    }
    
    if (includeItems) {
        graph.addAttributeNodes("items");
    }
    
    Map<String, Object> hints = new HashMap<>();
    hints.put("javax.persistence.fetchgraph", graph);
    
    return em.createQuery("SELECT o FROM Order o", Order.class)
               .setHint("javax.persistence.fetchgraph", graph)
               .getResultList();
}
```

**Best Practices:**

1. **Use EntityGraph for dynamic fetching** - When you need to vary associations per use case
2. **Prefer named graphs for reuse** - Define common fetch plans as annotations
3. **Use FETCH graph for performance** - Only fetch what you need
4. **Avoid deep subgraphs** - Can cause cartesian products
5. **Combine with DTO projections** - For read-only operations, consider DTOs instead

### 13.4 Batch Fetching

**Theoretical Foundation:** Batch fetching is a Hibernate-specific optimization that loads multiple lazy associations in batches rather than one-by-one. It reduces the N+1 problem from 1 + N queries to 1 + ceil(N/batch_size) queries. This is particularly useful when you can't use JOIN FETCH (e.g., when you don't know which associations will be accessed at query time).

**How Batch Fetching Works:**

1. **Configuration**: Set batch size globally or per association
2. **Lazy Access Tracking**: Hibernate tracks which lazy associations are accessed
3. **Batch Loading**: When lazy associations are accessed, Hibernate loads them in batches
4. **IN Clause**: Uses SQL IN clause to load multiple entities in one query

**Global Configuration:**

```properties
# application.properties
spring.jpa.properties.hibernate.default_batch_fetch_size=20
```

**Per-Association Configuration:**

```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @Batch(size = 20)  // Hibernate-specific annotation
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
    
    @Batch(size = 50)
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

**Performance Comparison:**

```java
// Without batch fetching: 1 + N queries
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class).getResultList();
orders.get(0).getCustomer();  // Query 1
orders.get(1).getCustomer();  // Query 2
orders.get(2).getCustomer();  // Query 3
// ...
orders.get(99).getCustomer(); // Query 100
// Total: 101 queries

// With batch fetching (size=20): 1 + ceil(N/20) queries
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class).getResultList();
orders.get(0).getCustomer();  // Triggers batch load: SELECT * FROM customer WHERE id IN (1..20)
orders.get(1).getCustomer();  // Already loaded
orders.get(2).getCustomer();  // Already loaded
// ...
orders.get(20).getCustomer(); // Triggers next batch: SELECT * FROM customer WHERE id IN (21..40)
// For 100 orders: 1 + 5 = 6 queries instead of 101
```

**SQL Generated with Batch Fetching:**

```sql
-- Initial query
SELECT * FROM orders;

-- First batch (accessing first customer)
SELECT * FROM customer WHERE id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20);

-- Second batch (accessing 21st customer)
SELECT * FROM customer WHERE id IN (21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40);
```

**Performance Analysis:**

```
Query Count Comparison (N=100 orders):

┌─────────────────────┬──────────┬──────────┬─────────────┐
│ Strategy            │ Queries  │ Latency  │ Improvement │
├─────────────────────┼──────────┼──────────┼─────────────┤
│ No batch (N+1)      │ 101      │ ~500ms   │ Baseline    │
│ Batch size=10       │ 11       │ ~55ms    │ 9x faster   │
│ Batch size=20       │ 6        │ ~30ms    │ 17x faster  │
│ Batch size=50       │ 3        │ ~15ms    │ 33x faster  │
│ JOIN FETCH          │ 1        │ ~5ms     │ 100x faster │
└─────────────────────┴──────────┴──────────┴─────────────┘

Note: JOIN FETCH is still fastest, but batch fetching provides flexibility
```

**Choosing the Right Batch Size:**

```properties
# Small batch size (10-20): Good for moderate result sets
spring.jpa.properties.hibernate.default_batch_fetch_size=20

# Large batch size (50-100): Good for large result sets
spring.jpa.properties.hibernate.default_batch_fetch_size=50

# Very large batch size (200+): Can cause SQL IN clause limits
# Some databases limit IN clause size (e.g., Oracle: 1000)
```

**Trade-offs:**

| Batch Size | Query Count | Memory Usage | IN Clause Size | Best For |
|------------|-------------|--------------|----------------|----------|
| Small (10) | More queries | Lower | Small | Memory-constrained |
| Medium (20-50) | Balanced | Medium | Medium | General purpose |
| Large (100+) | Fewer queries | Higher | Large | Large datasets |

**Batch Fetching vs JOIN FETCH:**

```java
// JOIN FETCH: Single query, but loads all associations upfront
String jpql = "SELECT o FROM Order o JOIN FETCH o.customer";
List<Order> orders = em.createQuery(jpql, Order.class).getResultList();
// 1 query, but customer loaded even if not accessed

// Batch Fetching: Multiple queries, but loads on-demand
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class).getResultList();
// Only load customers when accessed
if (needCustomer) {
    orders.get(0).getCustomer();  // Triggers batch load
}
// Flexible, but more queries if accessed
```

**Batch Fetching for Collections:**

```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @Batch(size = 20)
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> items;
}

// When accessing items from multiple orders:
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class).getResultList();
orders.get(0).getItems();  // Triggers batch load for first 20 orders
orders.get(1).getItems();  // Already loaded
orders.get(20).getItems(); // Triggers next batch
```

**Configuration Best Practices:**

```properties
# Global default (applies to all associations)
spring.jpa.properties.hibernate.default_batch_fetch_size=25

# Override for specific associations if needed
# Use @Batch annotation on entity fields

# For collections, consider larger batch sizes
# For single-valued associations, smaller batch sizes are fine
```

**When to Use Batch Fetching:**

1. **When you don't know which associations will be accessed** - Lazy loading with batching
2. **When JOIN FETCH causes cartesian products** - Multiple collections
3. **When you need flexibility** - Different use cases require different associations
4. **When result sets are large** - JOIN FETCH with large collections is expensive

**When NOT to Use Batch Fetching:**

1. **When you know exactly what you need** - JOIN FETCH is more efficient
2. **For small result sets** - Overhead of batching isn't worth it
3. **When associations are always accessed** - EAGER fetch or JOIN FETCH
4. **When you need all data upfront** - JOIN FETCH is better

### 13.5 Query Hints

**Theoretical Foundation:** Query hints are vendor-specific hints that provide additional instructions to the JPA provider (Hibernate) about how to execute a query. They allow fine-tuning of query execution without changing the query semantics. Hints are passed as key-value pairs and can affect caching, fetch behavior, timeout, and other execution parameters.

**Common Hibernate Query Hints:**

```java
Query query = em.createQuery("SELECT o FROM Order o WHERE o.id = :id");
query.setParameter("id", 1L);

// Cache query results
query.setHint("org.hibernate.cacheable", true);

// Set fetch size for large result sets
query.setHint("org.hibernate.fetchSize", 100);

// Set timeout
query.setHint("javax.persistence.query.timeout", 5000);

// Read-only (no dirty checking)
query.setHint("org.hibernate.readOnly", true);
```

**Comprehensive Hint Reference:**

| Hint Name | Purpose | Default | Use Case |
|-----------|---------|---------|----------|
| `org.hibernate.cacheable` | Enable query cache | false | Frequently executed queries |
| `org.hibernate.cacheMode` | Cache mode | NORMAL | Control cache interaction |
| `org.hibernate.fetchSize` | JDBC fetch size | 0 (all) | Large result sets |
| `org.hibernate.readOnly` | Read-only entities | false | Read-only operations |
| `org.hibernate.comment` | SQL comment | null | Query debugging |
| `javax.persistence.query.timeout` | Query timeout (ms) | 0 (no timeout) | Prevent long-running queries |
| `javax.persistence.lock.timeout` | Lock timeout (ms) | 0 (wait forever) | Pessimistic locking |
| `org.hibernate.flushMode` | Flush mode | AUTO | Control flush behavior |
| `org.hibernate.cacheRegion` | Cache region name | default | Organize cache regions |
| `org.hibernate.fetchProfile` | Fetch profile | none | Predefined fetch plans |
| `org.hibernate.loadGraph` | Load graph | none | Dynamic fetching |
| `org.hibernate.fetchGraph` | Fetch graph | none | Dynamic fetching |

**Detailed Examples:**

**1. Query Cache Hint:**

```java
// Enable query caching for frequently executed queries
Query query = em.createQuery(
    "SELECT p FROM Product p WHERE p.category = :category"
);
query.setParameter("category", "ELECTRONICS");
query.setHint("org.hibernate.cacheable", true);

// Requires second-level cache to be enabled
// Results are cached in query cache region
```

**Configuration:**
```properties
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory
```

**2. Fetch Size Hint:**

```java
// For large result sets, set fetch size to reduce memory usage
Query query = em.createQuery("SELECT o FROM Order o", Order.class);
query.setHint("org.hibernate.fetchSize", 100);
query.setHint("org.hibernate.readOnly", true);

// Uses JDBC cursor-based fetching
// Reduces memory footprint by streaming results
```

**Best for:** Large exports, batch processing, reports

**3. Timeout Hint:**

```java
// Set timeout to prevent long-running queries
Query query = em.createQuery("SELECT o FROM Order o WHERE o.status = :status");
query.setParameter("status", "PENDING");
query.setHint("javax.persistence.query.timeout", 5000); // 5 seconds

try {
    List<Order> orders = query.getResultList();
} catch (QueryTimeoutException e) {
    // Handle timeout
    throw new BusinessException("Query timed out, please narrow your search");
}
```

**4. Read-Only Hint:**

```java
// Mark query as read-only to skip dirty checking
Query query = em.createQuery("SELECT o FROM Order o", Order.class);
query.setHint("org.hibernate.readOnly", true);
query.setHint("org.hibernate.cacheable", true);

// Performance benefits:
// - No dirty checking overhead
// - Entities not tracked for changes
// - Can use query cache
```

**5. Comment Hint (Debugging):**

```java
// Add comment to SQL for debugging
Query query = em.createQuery("SELECT o FROM Order o WHERE o.id = :id");
query.setParameter("id", 1L);
query.setHint("org.hibernate.comment", "Fetching order by ID for order detail page");

// SQL generated will include comment:
// /* Fetching order by ID for order detail page */
// SELECT * FROM orders WHERE id = ?
```

**6. Cache Mode Hint:**

```java
// Control cache interaction
Query query = em.createQuery("SELECT p FROM Product p WHERE p.id = :id");
query.setParameter("id", 1L);

// Don't put results in cache
query.setHint("org.hibernate.cacheMode", CacheMode.IGNORE);

// Put in cache but don't read from cache
query.setHint("org.hibernate.cacheMode", CacheMode.PUT);

// Read from cache but don't put in cache
query.setHint("org.hibernate.cacheMode", CacheMode.GET);

// Normal: both read and write
query.setHint("org.hibernate.cacheMode", CacheMode.NORMAL);
```

**7. Flush Mode Hint:**

```java
// Control when flush occurs
Query query = em.createQuery("SELECT o FROM Order o WHERE o.status = :status");
query.setParameter("status", "PENDING");

// Don't flush before this query
query.setHint("org.hibernate.flushMode", FlushModeType.COMMIT);

// Always flush before this query
query.setHint("org.hibernate.flushMode", FlushModeType.ALWAYS);
```

**8. Cache Region Hint:**

```java
// Organize cached queries into regions
Query query = em.createQuery("SELECT p FROM Product p WHERE p.category = :category");
query.setParameter("category", "ELECTRONICS");
query.setHint("org.hibernate.cacheable", true);
query.setHint("org.hibernate.cacheRegion", "product.byCategory");

// Allows separate cache configuration per region
```

**Spring Data JPA Query Hints:**

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @QueryHints({
        @QueryHint(name = "org.hibernate.cacheable", value = "true"),
        @QueryHint(name = "org.hibernate.readOnly", value = "true")
    })
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    List<Order> findByStatus(@Param("status") String status);
    
    @QueryHints({
        @QueryHint(name = "org.hibernate.fetchSize", value = "100"),
        @QueryHint(name = "org.hibernate.readOnly", value = "true")
    })
    List<Order> findAllWithHints();
}
```

**Performance Impact:**

```
Query Hint Performance Impact:

┌─────────────────────┬──────────┬─────────────┬──────────────┐
│ Hint                │ Impact   │ When to Use │ Risk         │
├─────────────────────┼──────────┼─────────────┼──────────────┤
│ cacheable           │ High     │ Frequent    │ Stale data   │
│ fetchSize           │ High     │ Large sets  | None         │
│ readOnly            │ Medium   │ Read-only   | None         │
│ timeout             │ Medium   │ All queries │ Too short?   │
│ comment             | Low      │ Debugging  | None         │
│ cacheMode           | High     | Advanced    | Complexity   │
│ flushMode           | Medium   | Specific    │ Side effects │
└─────────────────────┴──────────┴─────────────┴──────────────┘
```

**Best Practices:**

1. **Use cacheable for frequently executed queries** - Same parameters, same results
2. **Set fetchSize for large result sets** - Reduces memory footprint
3. **Use readOnly for read-only operations** - Eliminates dirty checking overhead
4. **Set timeout for user-facing queries** - Prevents slow queries from affecting UX
5. **Use comment for debugging** - Helps identify queries in database logs
6. **Profile before optimizing** - Use hints based on actual performance measurements
7. **Don't over-optimize** - Only use hints when you have a proven performance issue

### 13.6 DTO Projections

**Theoretical Foundation:** DTO (Data Transfer Object) projections allow you to select specific columns from the database without loading full entities. This is a fundamental optimization technique that reduces data transfer, memory usage, and eliminates the overhead of entity management (dirty checking, first-level cache, lazy loading). DTOs are particularly effective for read-only operations like reports, dashboards, and API responses.

Avoid loading entities entirely - use DTOs for read-only operations:

```java
// DTO class
public class OrderSummary {
    private Long id;
    private String status;
    private BigDecimal total;
    private String customerName;
    
    // Constructor
    public OrderSummary(Long id, String status, BigDecimal total, String customerName) {
        this.id = id;
        this.status = status;
        this.total = total;
        this.customerName = customerName;
    }
}

// Query with DTO projection
String jpql = "SELECT new com.example.OrderSummary(" +
              "o.id, o.status, o.total, c.name) " +
              "FROM Order o JOIN o.customer c";
List<OrderSummary> summaries = em.createQuery(jpql, OrderSummary.class)
                                 .getResultList();
```

**Benefits:**
- ✅ Single query
- ✅ No entities in Persistence Context (no dirty checking overhead)
- ✅ Only selected columns loaded
- ✅ No lazy loading issues

**Class-Based Projections:**

```java
// Constructor-based projection
public class OrderSummary {
    private Long id;
    private String status;
    private BigDecimal total;
    
    public OrderSummary(Long id, String status, BigDecimal total) {
        this.id = id;
        this.status = status;
        this.total = total;
    }
    
    // Getters
    public Long getId() { return id; }
    public String getStatus() { return status; }
    public BigDecimal getTotal() { return total; }
}

// Query
String jpql = "SELECT new com.example.OrderSummary(o.id, o.status, o.total) " +
              "FROM Order o";
List<OrderSummary> summaries = em.createQuery(jpql, OrderSummary.class).getResultList();
```

**Interface-Based Projections (Spring Data JPA):**

```java
// Interface-based projection (no implementation needed)
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
    String getCustomerName();
}

// Repository method
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Query("SELECT o.id as id, o.status as status, o.total as total, c.name as customerName " +
           "FROM Order o JOIN o.customer c")
    List<OrderSummary> findOrderSummaries();
}
```

**Spring Data JPA Projection Types:**

**1. Interface Projections (Closed):**

```java
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
}
```

**2. Interface Projections (Open/Dynamic):**

```java
public interface OrderSummary {
    Long getId();
    String getStatus();
    
    // Dynamic access to other properties
    @Value("#{target.customer.name}")
    String getCustomerName();
}
```

**3. Class-Based Projections (DTOs):**

```java
public class OrderSummary {
    private Long id;
    private String status;
    private BigDecimal total;
    
    // Required constructor
    public OrderSummary(Long id, String status, BigDecimal total) {
        this.id = id;
        this.status = status;
        this.total = total;
    }
    
    // Getters
    public Long getId() { return id; }
    public String getStatus() { return status; }
    public BigDecimal getTotal() { return total; }
}
```

**4. Tuple Projections:**

```java
// For ad-hoc queries without DTO classes
Query query = em.createQuery(
    "SELECT o.id, o.status, o.total FROM Order o"
);
List<Object[]> results = query.getResultList();

for (Object[] row : results) {
    Long id = (Long) row[0];
    String status = (String) row[1];
    BigDecimal total = (BigDecimal) row[2];
}
```

**Performance Comparison:**

```
Entity vs DTO Performance (1000 rows):

┌─────────────────────┬──────────┬──────────┬─────────────┐
│ Approach            │ Data Size│ Memory   │ Query Time  │
├─────────────────────┼──────────┼──────────┼─────────────┤
│ Full Entity         │ 100%     │ 100%     │ 100ms       │
│ DTO (3 columns)     │ 20%      │ 20%      │ 25ms        │
│ DTO (1 column)      │ 7%       │ 7%       │ 10ms        │
└─────────────────────┴──────────┴──────────┴─────────────┘

Note: Actual savings depend on entity complexity and column sizes
```

**Nested Projections:**

```java
// Nested DTO for related data
public class OrderDetail {
    private Long id;
    private String status;
    private CustomerSummary customer;
    
    public OrderDetail(Long id, String status, CustomerSummary customer) {
        this.id = id;
        this.status = status;
        this.customer = customer;
    }
}

public class CustomerSummary {
    private Long id;
    private String name;
    private String email;
    
    public CustomerSummary(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
}

// Query with nested DTO
String jpql = "SELECT new com.example.OrderDetail(" +
              "o.id, o.status, " +
              "new com.example.CustomerSummary(c.id, c.name, c.email)) " +
              "FROM Order o JOIN o.customer c";
```

**Spring Data JPA Nested Projections:**

```java
public interface OrderDetail {
    Long getId();
    String getStatus();
    CustomerSummary getCustomer();
}

public interface CustomerSummary {
    Long getId();
    String getName();
    String getEmail();
}

// Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Query("SELECT o.id as id, o.status as status, c as customer " +
           "FROM Order o JOIN o.customer c")
    List<OrderDetail> findOrderDetails();
}
```

**When to Use DTO Projections:**

1. **Read-only operations** - Reports, dashboards, API responses
2. **Large result sets** - Reduce memory footprint
3. **API endpoints** - Only return needed fields
4. **Aggregations** - Sum, count, avg operations
5. **Performance-critical queries** - Minimize data transfer

**When NOT to Use DTO Projections:**

1. **When you need to modify data** - Entities are required for updates
2. **When you need lazy loading** - DTOs don't support lazy loading
3. **When you need entity lifecycle** - Cascade operations, events
4. **Simple queries** - Overhead of DTO class may not be worth it

**Best Practices:**

1. **Use DTOs for read-only operations** - Eliminates entity overhead
2. **Select only needed columns** - Reduce data transfer
3. **Use interface projections for simple cases** - Less boilerplate
4. **Use class DTOs for complex projections** - More control
5. **Consider mapping frameworks** - MapStruct, ModelMapper for complex mappings
6. **Keep DTOs immutable** - Prevent accidental modifications

### 13.7 Criteria API vs HQL

**Theoretical Foundation:** The Criteria API and HQL/JPQL represent two different approaches to building queries in JPA. HQL/JPQL is a string-based query language similar to SQL, while the Criteria API is a type-safe, programmatic API for building queries. The choice between them depends on factors like type safety requirements, query complexity, dynamic nature of filters, and team preference.

| Aspect | HQL/JPQL | Criteria API |
|--------|----------|--------------|
| **Readability** | High (SQL-like) | Lower (verbose) |
| **Type Safety** | Compile-time string checking | Strong type safety |
| **Dynamic Queries** | String concatenation | Programmatic building |
| **Complex Queries** | Easier | More verbose |
| **Best For** | Static queries | Dynamic filters |

**HQL/JPQL Example:**

```java
// Simple, readable HQL query
String jpql = "SELECT o FROM Order o WHERE o.status = :status AND o.total > :minTotal";
List<Order> orders = em.createQuery(jpql, Order.class)
                         .setParameter("status", "PENDING")
                         .setParameter("minTotal", new BigDecimal("100"))
                         .getResultList();
```

**Criteria API Example:**

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> cq = cb.createQuery(Order.class);
Root<Order> order = cq.from(Order.class);

// Dynamic filters
List<Predicate> predicates = new ArrayList<>();

if (status != null) {
    predicates.add(cb.equal(order.get("status"), status));
}

if (minTotal != null) {
    predicates.add(cb.ge(order.get("total"), minTotal));
}

cq.where(predicates.toArray(new Predicate[0]));
List<Order> orders = em.createQuery(cq).getResultList();
```

**Detailed Comparison:**

**1. Type Safety:**

```java
// HQL: Runtime errors if entity/field names are wrong
String jpql = "SELECT o FROM Order o WHERE o.stattus = :status";  // Typo!
// Compiles fine, fails at runtime

// Criteria API: Compile-time errors
Root<Order> order = cq.from(Order.class);
order.get("stattus");  // Typo caught at compile time if using metamodel
```

**2. Dynamic Query Building:**

```java
// HQL: String manipulation (error-prone)
StringBuilder jpql = new StringBuilder("SELECT o FROM Order o WHERE 1=1");
Map<String, Object> params = new HashMap<>();

if (status != null) {
    jpql.append(" AND o.status = :status");
    params.put("status", status);
}

if (minTotal != null) {
    jpql.append(" AND o.total > :minTotal");
    params.put("minTotal", minTotal);
}

Query query = em.createQuery(jpql.toString());
params.forEach(query::setParameter);

// Criteria API: Type-safe dynamic building
List<Predicate> predicates = new ArrayList<>();
if (status != null) {
    predicates.add(cb.equal(order.get(Order_.status), status));
}
if (minTotal != null) {
    predicates.add(cb.ge(order.get(Order_.total), minTotal));
}
cq.where(predicates.toArray(new Predicate[0]));
```

**3. Complex Queries:**

```java
// HQL: More readable for complex joins
String jpql = "SELECT DISTINCT o FROM Order o " +
              "JOIN FETCH o.customer c " +
              "JOIN FETCH o.items i " +
              "WHERE c.name = :customerName " +
              "AND i.quantity > 0 " +
              "ORDER BY o.createdAt DESC";

// Criteria API: More verbose but type-safe
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> cq = cb.createQuery(Order.class);
Root<Order> order = cq.from(Order.class);

// Join
Join<Order, Customer> customer = order.join(Order_.customer);
Join<Order, OrderItem> items = order.join(Order_.items);

// Conditions
cq.where(
    cb.equal(customer.get(Customer_.name), customerName),
    cb.gt(items.get(OrderItem_.quantity), 0)
);

// Order by
cq.orderBy(cb.desc(order.get(Order_.createdAt)));

// Distinct
cq.distinct(true);
```

**4. Metamodel for Compile-Time Safety:**

```java
// Generate metamodel (Hibernate JPA 2.1 Metamodel Generator)
// Add to pom.xml:
// <plugin>
//   <groupId>org.hibernate.orm</groupId>
//   <artifactId>hibernate-jpamodelgen-plugin</artifactId>
// </plugin>

// Use metamodel for type-safe Criteria API
import com.example.domain.Order_;

CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> cq = cb.createQuery(Order.class);
Root<Order> order = cq.from(Order.class);

// Type-safe field access
cq.where(cb.equal(order.get(Order_.status), "PENDING"));
```

**5. Subqueries:**

```java
// HQL: Subqueries are readable
String jpql = "SELECT o FROM Order o WHERE o.total > " +
              "(SELECT AVG(o2.total) FROM Order o2)";

// Criteria API: Subqueries are more complex
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> cq = cb.createQuery(Order.class);
Root<Order> order = cq.from(Order.class);

Subquery<BigDecimal> subquery = cq.subquery(BigDecimal.class);
Root<Order> order2 = subquery.from(Order.class);
subquery.select(cb.avg(order2.get(Order_.total)));

cq.where(cb.gt(order.get(Order_.total), subquery));
```

**Performance Comparison:**

```
Query Building Performance:

┌─────────────────────┬──────────┬─────────────┬──────────────┐
│ Aspect              │ HQL      │ Criteria     │ Notes        │
├─────────────────────┼──────────┼─────────────┼──────────────┤
│ Parse Time          │ Low      │ High        │ Criteria parsed at compile │
│ Execution Time      │ Same     │ Same        │ Same SQL generated │
│ Memory Usage        │ Low      | Higher      │ Criteria uses more objects │
│ Caching             │ Better   | Worse       │ HQL easier to cache │
└─────────────────────┴──────────┴─────────────┴──────────────┘
```

**Decision Framework:**

```
Use HQL/JPQL when:
├─ Query is static (filters don't change)
├─ Query is complex (multiple joins, subqueries)
├─ Readability is important
├─ Team is familiar with SQL
└─ Performance is critical (better caching)

Use Criteria API when:
├─ Query is dynamic (filters built at runtime)
├─ Type safety is critical
├─ Query builder UI/DSL
├─ Metamodel available
└─ Compile-time checking required
```

**Best Practices:**

1. **Use HQL for static queries** - More readable and maintainable
2. **Use Criteria for dynamic queries** - Type-safe filter building
3. **Use QueryDSL instead of Criteria** - More readable than Criteria API
4. **Consider Specification (Spring Data)** - Combines type safety with readability
5. **Profile both approaches** - Performance difference is usually negligible

**Spring Data JPA Specification (Alternative):**

```java
// Type-safe query building with Spring Data
public interface OrderRepository extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> {
    
    default List<Order> findByStatusAndMinTotal(String status, BigDecimal minTotal) {
        return findAll((Root<Order> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            
            if (status != null) {
                predicates.add(cb.equal(root.get(Order_.status), status));
            }
            
            if (minTotal != null) {
                predicates.add(cb.ge(root.get(Order_.total), minTotal));
            }
            
            return cb.and(predicates.toArray(new Predicate[0]));
        });
    }
}
```

### 13.8 Database Indexing

**Theoretical Foundation:** Database indexing is a database-level optimization that creates data structures (typically B-trees) to accelerate data retrieval. While not a JPA/Hibernate feature, it's critical for query performance. Hibernate can generate indexes based on entity annotations, but proper indexing strategy requires understanding query patterns and database internals.

**Index Types:**

| Index Type | Description | Use Case |
|-------------|-------------|----------|
| **B-Tree** | Balanced tree, default for most databases | Equality, range queries |
| **Hash** | Hash-based lookup | Exact equality only |
| **GIN** | Generalized inverted index | Array values, full-text search |
| **GiST** | Generalized search tree | Geospatial, full-text |
| **Bitmap** | Bitmap for low-cardinality columns | Data warehousing |

**Indexing with JPA Annotations:**

```java
@Entity
@Table(
    name = "orders",
    indexes = {
        @Index(name = "idx_order_status", columnList = "status"),
        @Index(name = "idx_order_customer", columnList = "customer_id"),
        @Index(name = "idx_order_status_date", columnList = "status, created_at")
    }
)
public class Order {
    @Id
    private Long id;
    
    @Column(name = "status")
    private String status;
    
    @Column(name = "customer_id")
    private Long customerId;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "total")
    private BigDecimal total;
}
```

**Composite Index Ordering:**

```java
// Composite index on (status, created_at)
@Index(name = "idx_order_status_date", columnList = "status, created_at")

// Queries that use this index effectively:
// ✅ WHERE status = ? (uses index)
// ✅ WHERE status = ? AND created_at > ? (uses index)
// ✅ WHERE status = ? ORDER BY created_at (uses index)
// ❌ WHERE created_at > ? (index not used - status not in WHERE)
// ❌ WHERE created_at > ? ORDER BY status (index not used efficiently)
```

**Index Cardinality:**

```java
// High cardinality (many unique values) - Good for indexing
@Column(name = "customer_id")
@Index(name = "idx_customer")  // Each customer_id is unique

// Low cardinality (few unique values) - Poor for indexing
@Column(name = "status")
@Index(name = "idx_status")  // Only 5-10 unique values (PENDING, SHIPPED, etc.)
// Consider bitmap index for low cardinality
```

**Index Best Practices:**

1. **Index foreign keys** - Always index columns used in JOINs
2. **Index WHERE clause columns** - Columns frequently filtered
3. **Index ORDER BY columns** - Columns used for sorting
4. **Avoid over-indexing** - Each index slows down INSERT/UPDATE
5. **Monitor index usage** - Remove unused indexes

**Hibernate Index Generation:**

```properties
# Enable automatic index generation
spring.jpa.hibernate.ddl-auto=update

# Hibernate will create indexes based on @Index annotations
# For production, use validate or manual schema migration
```

### 13.9 Query Caching

**Theoretical Foundation:** Query caching is a Hibernate feature that caches the results of queries (not the entities themselves). When the same query is executed with the same parameters, Hibernate returns the cached result set instead of hitting the database. This is separate from the second-level cache which caches entity data.

**Query Cache vs Entity Cache:**

| Aspect | Query Cache | Entity Cache (L2) |
|--------|-------------|-------------------|
| **What's Cached** | Query results (entity IDs) | Entity data |
| **Scope** | Per query + parameters | Per entity |
| **Invalidation** | On any entity modification | On entity modification |
| **Use Case** | Frequently executed queries | Frequently accessed entities |

**Enabling Query Cache:**

```properties
# Enable query cache
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory
```

**Using Query Cache:**

```java
// Enable caching for specific query
Query query = em.createQuery(
    "SELECT p FROM Product p WHERE p.category = :category"
);
query.setParameter("category", "ELECTRONICS");
query.setHint("org.hibernate.cacheable", true);

// First execution: hits database
List<Product> products1 = query.getResultList();

// Second execution with same parameters: returns from cache
List<Product> products2 = query.getResultList();
```

**Query Cache Configuration:**

```properties
# Query cache region configuration
spring.jpa.properties.hibernate.cache.query_cache_factory=org.hibernate.cache.ehcache.EhCacheRegionFactory

# Set query cache region
query.setHint("org.hibernate.cacheRegion", "product.byCategory");
```

**Query Cache Limitations:**

1. **Only works with exact parameter matches** - Different parameters = cache miss
2. **Invalidated on any entity modification** - Even unrelated changes
3. **Not suitable for frequently changing data** - Cache invalidation overhead
4. **Requires second-level cache** - Query cache depends on L2 cache

**When to Use Query Cache:**

- Read-mostly data (reference tables)
- Frequently executed queries with same parameters
- Expensive queries (complex joins, aggregations)
- Data that changes infrequently

**When NOT to Use Query Cache:**

- Frequently changing data
- Queries with many different parameter combinations
- Write-heavy workloads
- When cache invalidation cost > query cost

### 13.10 Performance Monitoring

**Theoretical Foundation:** Performance monitoring is essential for identifying slow queries, N+1 problems, and other performance bottlenecks. Hibernate provides built-in statistics, and external tools can provide deeper insights into query performance.

**Hibernate Statistics:**

```properties
# Enable Hibernate statistics
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.properties.hibernate.session_factory_name_statistics=true
```

**Accessing Statistics:**

```java
@Service
public class QueryMonitor {
    
    @PersistenceContext
    private EntityManager em;
    
    public void logStatistics() {
        Statistics stats = em.getEntityManagerFactory().getStatistics();
        
        log.info("Entity Load Count: {}", stats.getEntityLoadCount());
        log.info("Entity Fetch Count: {}", stats.getEntityFetchCount());
        log.info("Query Execution Count: {}", stats.getQueryExecutionCount());
        log.info("Query Cache Hit Count: {}", stats.getQueryCacheHitCount());
        log.info("Query Cache Miss Count: {}", stats.getQueryCacheMissCount());
        log.info("Second Level Cache Hit Count: {}", stats.getSecondLevelCacheHitCount());
        log.info("Second Level Cache Miss Count: {}", stats.getSecondLevelCacheMissCount());
    }
}
```

**Slow Query Logging:**

```properties
# Log slow queries
spring.jpa.properties.hibernate.session_factory_name_statistics=true
spring.jpa.properties.hibernate.jdbc.batch_size=50

# Custom logging interceptor
@Interceptors({SlowQueryInterceptor.class})
```

**Database Query Logging:**

```properties
# Enable SQL logging
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

**Query Execution Plan Analysis:**

```java
// Get execution plan (database-specific)
Query query = em.createNativeQuery(
    "EXPLAIN ANALYZE SELECT * FROM orders WHERE status = :status"
);
query.setParameter("status", "PENDING");
List<Object[]> plans = query.getResultList();

for (Object[] plan : plans) {
    log.info("Execution Plan: {}", plan[0]);
}
```

**APM Tools Integration:**

- **New Relic** - Database query monitoring
- **Datadog** - Query performance tracking
- **Jaeger/Zipkin** - Distributed tracing
- **Micrometer** - Custom metrics

**Custom Metrics:**

```java
@Service
public class QueryMetrics {
    
    private final MeterRegistry meterRegistry;
    private final EntityManager em;
    
    @Timed("order.query.execution")
    public List<Order> findOrders(String status) {
        Query query = em.createQuery("SELECT o FROM Order o WHERE o.status = :status");
        query.setParameter("status", status);
        return query.getResultList();
    }
}
```

**Performance Monitoring Best Practices:**

1. **Enable statistics in development** - Not in production (performance impact)
2. **Monitor query execution time** - Identify slow queries
3. **Track N+1 problems** - Entity fetch count vs entity load count
4. **Monitor cache hit ratios** - Low hit ratio = cache not effective
5. **Set up alerts** - For slow queries or high error rates
6. **Regular performance reviews** - Weekly/monthly analysis
7. **Benchmark before optimization** - Measure improvement

**Common Performance Metrics:**

```
Key Performance Indicators:

┌─────────────────────────────┬──────────────┬─────────────┐
│ Metric                      │ Target       │ Alert       │
├─────────────────────────────┼──────────────┼─────────────┤
│ Query Execution Time       │ < 100ms      │ > 500ms     │
│ N+1 Ratio (fetch/load)     │ < 1.5        │ > 3.0       │
│ Cache Hit Ratio (L2)        │ > 80%        │ < 50%       │
│ Query Cache Hit Ratio       │ > 70%        │ < 30%       │
│ Connection Pool Usage      │ < 80%        │ > 90%       │
│ Transaction Duration       │ < 1s         │ > 5s        │
└─────────────────────────────┴──────────────┴─────────────┘
```

---

## 14. Second-Level Cache

### 14.1 Overview

The second-level cache (L2) is shared across all `Session` instances in the same `SessionFactory`. It stores entity data in a dehydrated (serialized) form.

```
┌─────────────────────────────────────────────────────────────┐
│                     SessionFactory (Application Scope)      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Second-Level Cache (L2)                  │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ EntityKey → Dehydrated State                   │  │  │
│  │  │ (User, 42) → {id:42, name:"Alice", age:30}     │  │  │
│  │  │ (Order, 100) → {id:100, status:"SHIPPED"}      │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
        ▲                                          ▲
        │                                          │
   ┌─────┴───────┐                           ┌─────┴───────┐
   │ Session 1   │                           │ Session N   │
   │ (L1 Cache)  │                           │ (L1 Cache)  │
   └─────────────┘                           └─────────────┘
```

### 14.2 Cache Providers

| Provider | Description | Configuration |
|----------|-------------|--------------|
| **Ehcache** | Popular, feature-rich | `hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory` |
| **Infinispan** | Distributed, clustering | `hibernate.cache.region.factory_class=org.hibernate.cache.infinispan.InfinispanRegionFactory` |
| **Hazelcast** | Distributed, cloud-native | `hibernate.cache.region.factory_class=com.hazelcast.hibernate.HazelcastLocalCacheRegionFactory` |
| **Caffeine** | High-performance, local | `hibernate.cache.region.factory_class=org.hibernate.cache.caffeine.CaffeineRegionFactory` |

### 14.3 Enabling L2 Cache

```properties
# Enable second-level cache
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory

# Enable query cache
spring.jpa.properties.hibernate.cache.use_query_cache=true
```

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    private Long id;
    private String name;
    private Integer stock;
}
```

### 14.4 Cache Concurrency Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **READ_ONLY** | Never updated, only loaded | Reference data, lookup tables |
| **READ_WRITE** | Read/write with locks | Frequently updated entities |
| **NONSTRICT_READ_WRITE** | Read/write without locks | Low contention, eventual consistency OK |
| **TRANSACTIONAL** | Full transactional support | Distributed caches, high consistency |

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    // ...
}
```

### 14.5 Cache Regions

Organize cached entities into regions for fine-grained control:

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "products")
public class Product {
    // ...
}

@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "reference_data")
public class Country {
    // ...
}
```

**Ehcache configuration (ehcache.xml):**
```xml
<config>
    <cache name="products"
           maxEntriesLocalHeap="10000"
           eternal="false"
           timeToIdleSeconds="300"
           timeToLiveSeconds="600"/>
    
    <cache name="reference_data"
           maxEntriesLocalHeap="1000"
           eternal="true"/>
</config>
```

### 14.6 Query Cache

Caches query results (entity IDs, not full entities):

```java
List<Product> products = em.createQuery(
    "SELECT p FROM Product p WHERE p.stock > 0", Product.class)
    .setHint("org.hibernate.cacheable", true)
    .getResultList();
```

**Important:** Query cache requires L2 cache for the entities. It only caches IDs; entity data comes from L2.

### 14.7 Cache Eviction

```java
// Evict specific entity from L2
em.getEntityManagerFactory().getCache().evict(Product.class, 1L);

// Evict all entities of a type
em.getEntityManagerFactory().getCache().evict(Product.class);

// Evict all from L2
em.getEntityManagerFactory().getCache().evictAll();
```

---

## 15. Batch Processing

### 15.1 JDBC Batching

Configure Hibernate to batch INSERT/UPDATE operations:

```properties
# Enable JDBC batching
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

```java
@Transactional
public void batchInsert(List<Product> products) {
    for (int i = 0; i < products.size(); i++) {
        em.persist(products.get(i]);
        
        // Flush and clear every 50 entities
        if (i % 50 == 0) {
            em.flush();
            em.clear();  // Prevent memory buildup
        }
    }
    em.flush();  // Flush remaining
}
```

**Without batching:**
```java
// 1000 INSERT statements sent individually
for (Product p : products) {
    em.persist(p);
}
```

**With batching (batch_size=50):**
```java
// 20 batches of 50 INSERT statements
for (Product p : products) {
    em.persist(p);
}
```

### 15.2 Stateless Session

For massive inserts/updates where Persistence Context overhead is unnecessary:

```java
StatelessSession session = entityManager.unwrap(Session.class)
                                        .getSessionFactory()
                                        .openStatelessSession();

Transaction tx = session.beginTransaction();

for (Product product : products) {
    session.insert(product);  // No dirty checking, no cascade, no events
}

tx.commit();
session.close();
```

**StatelessSession characteristics:**
- ✅ No first-level cache (no memory buildup)
- ✅ No dirty checking (faster)
- ✅ No cascade operations
- ✅ No lifecycle events (@PrePersist, etc.)
- ✅ Returns detached entities
- ❌ No lazy loading
- ❌ Doesn't work with @Version (optimistic locking)

### 15.3 Bulk Updates with JPQL

```java
// Single SQL UPDATE statement
int updated = em.createQuery(
    "UPDATE Product p SET p.price = p.price * 1.1 WHERE p.category = :category")
    .setParameter("category", "ELECTRONICS")
    .executeUpdate();

// Note: This bypasses the Persistence Context
// Entities in PC will have stale values
em.clear();  // Clear PC to avoid stale data
```

### 15.4 Performance Comparison

| Approach | 1000 Inserts | Memory | Dirty Checking | Cascade | Events |
|----------|--------------|--------|----------------|---------|--------|
| **Standard Session** | ~1000ms | High | Yes | Yes | Yes |
| **Standard + Batch** | ~200ms | High | Yes | Yes | Yes |
| **Standard + Batch + Clear** | ~200ms | Low | Yes | Yes | Yes |
| **Stateless Session** | ~50ms | None | No | No | No |
| **Bulk JPQL** | ~10ms | None | No | No | No |

### 15.5 Best Practices

```java
@Transactional
public void optimalBatchInsert(List<Product> products) {
    // Use SEQUENCE for ID generation (not IDENTITY)
    // Enable JDBC batching
    // Flush and clear periodically
    
    int batchSize = 50;
    for (int i = 0; i < products.size(); i++) {
        em.persist(products.get(i));
        
        if (i % batchSize == 0 && i > 0) {
            em.flush();
            em.clear();
        }
    }
    em.flush();
}
```

## Summary Table: All Key Concepts

| Concept | Description | Key Detail |
|---------|-------------|------------|
| **JPA** | Specification (interfaces) | Vendor-neutral API |
| **Hibernate** | Implementation | Provides Session, Caching, HQL |
| **EntityManager** | Per-request facade | Wraps Session, not thread-safe |
| **Persistence Context** | Identity map + snapshot map + action queue | = First-Level Cache |
| **First-Level Cache** | Same as Persistence Context | Always on, scoped to one Session |
| **Session** | Hibernate's native EntityManager | Adds `saveOrUpdate`, `evict`, Criteria |
| **Flush** | Sync PC → DB | Dirty check + ordered SQL execution |
| **TRANSIENT** | Created with `new`, no PC association | No ID, no DB row, GC-eligible |
| **MANAGED** | In Persistence Context | Identity guarantee, dirty checking, write-behind |
| **DETACHED** | Was managed, PC closed | No dirty checking, lazy = exception |
| **REMOVED** | Flagged for deletion | Still in PC until flush, then gone |
| **@Id** | Primary key marker | Used for EntityKey, all SQL WHERE clauses |
| **IDENTITY** | DB auto-increment | Immediate INSERT, breaks batching |
| **SEQUENCE** | DB sequence | Deferred INSERT, best for batching |
| **ASSIGNED** | Application sets ID | Framework (EntityListener, Factory) must assign |
| **Write-Behind** | Queue actions, execute at flush | INSERT uses latest field values |
| **Proxy** | Runtime subclass | Intercepts methods, triggers SELECT on first non-ID access |
| **Byte Enhancement** | Build-time class modification | Intercepts method reads/writes |
| **LazyInitializationException** | Session closed before access | Proxy has no session to execute SQL |
| **N+1 Problem** | Each lazy access = 1 query | Fix with JOIN FETCH or EntityGraph |
| **Dirty Tracking** | Inline field change detection | O(1) per entity vs O(n) snapshot comparison |
| **Transaction Isolation** | READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE | Controls concurrent transaction visibility |
| **Transaction Propagation** | REQUIRED, REQUIRES_NEW, MANDATORY, etc. | Spring @Transactional behavior |
| **Flush Mode** | AUTO, COMMIT, MANUAL, ALWAYS | When Persistence Context syncs to DB |
| **SINGLE_TABLE** | Single table with discriminator | Best performance, null columns |
| **JOINED** | Normalized tables with FK joins | Normalized schema, slower queries |
| **TABLE_PER_CLASS** | Separate table per concrete class | UNION queries, no discriminator |
| **Optimistic Locking** | @Version field, check on commit | Best for low contention |
| **Pessimistic Locking** | SELECT FOR UPDATE/FOR SHARE | Database locks, high contention |
| **Cascade Types** | PERSIST, MERGE, REMOVE, REFRESH, DETACH, ALL | Propagate operations to associations |
| **orphanRemoval** | Delete when removed from collection | Prevent orphan rows |
| **JOIN FETCH** | Load associations in same query | Solves N+1 problem |
| **EntityGraph** | Declarative fetch plan | Dynamic loading control |
| **Batch Fetching** | Load lazy associations in batches | Reduces query count |
| **DTO Projection** | Query to DTO constructor | No entities, no dirty checking |
| **Second-Level Cache** | Shared across SessionFactory | Cache providers, concurrency strategies |
| **Cache Regions** | Organize cached entities | Fine-grained cache control |
| **JDBC Batching** | Batch INSERT/UPDATE statements | Requires SEQUENCE, not IDENTITY |
| **Stateless Session** | No Persistence Context | Bulk operations, no cascade/events |

---

## Glossary of Key Terms

### A

**ACID Properties**
- **Definition**: Atomicity, Consistency, Isolation, Durability — the four properties that guarantee database transactions are processed reliably
- **Context**: Transactions must satisfy all four properties to ensure data integrity
- **See Also**: Section 9 (Transaction Management)

**Action Queue**
- **Definition**: A queue maintained by Hibernate's Session that holds pending database operations (INSERT, UPDATE, DELETE) to be executed at flush time
- **Purpose**: Implements the write-behind pattern, allowing operations to be batched and ordered correctly
- **See Also**: Section 2.4 (Persistence Context)

**Aggregation**
- **Definition**: A "whole-part" relationship where the child cannot exist independently of the parent
- **JPA Mapping**: `@OneToMany` with `cascade = CascadeType.ALL` and `orphanRemoval = true`
- **See Also**: Section 0.2 (Key ORM Concepts)

**Association**
- **Definition**: A relationship between entities where both can exist independently
- **JPA Mapping**: `@ManyToOne`, `@OneToOne`, `@OneToMany`, `@ManyToMany`
- **See Also**: Section 0.2 (Key ORM Concepts)

### B

**Bytecode Enhancement**
- **Definition**: Build-time modification of entity class bytecode to intercept method calls for lazy loading and dirty tracking
- **Purpose**: Enables lazy loading of basic fields and eliminates snapshot comparison for dirty checking
- **See Also**: Section 7 (Bytecode Enhancement Explained)

**Bidirectional Association**
- **Definition**: A relationship where both sides maintain references to each other
- **Example**: `Order` has `List<OrderItem>` and `OrderItem` has `Order`
- **Requirement**: One side must be marked as the "mappedBy" (owning) side

**Bulk Update**
- **Definition**: Executing a single SQL UPDATE/DELETE statement that affects multiple rows
- **JPA**: `em.createQuery("UPDATE ...").executeUpdate()`
- **Trade-off**: Bypasses Persistence Context, entities become stale
- **See Also**: Section 15.3 (Bulk Updates with JPQL)

### C

**Cache Concurrency Strategy**
- **Definition**: Strategy for handling concurrent access to cached entities in the second-level cache
- **Types**: READ_ONLY, READ_WRITE, NONSTRICT_READ_WRITE, TRANSACTIONAL
- **See Also**: Section 14.4 (Cache Concurrency Strategies)

**Cascade**
- **Definition**: Propagation of operations from a parent entity to its associated entities
- **Types**: PERSIST, MERGE, REMOVE, REFRESH, DETACH, ALL
- **See Also**: Section 12 (Cascade Operations)

**Connection Pool**
- **Definition**: A cache of database connections maintained for reuse, reducing connection establishment overhead
- **Common Implementations**: HikariCP, C3P0, DBCP
- **Hibernate Role**: Hibernate obtains connections from the pool via DataSource

**Dirty Checking**
- **Definition**: Automatic detection of which managed entities have changed since they were loaded
- **Implementation**: Compares current field values against snapshot taken at load time
- **See Also**: Section 2.4 (Persistence Context)

**Dirty Field**
- **Definition**: A field whose value has changed since the entity was loaded
- **Optimization**: With bytecode enhancement, only dirty fields are included in UPDATE statements

**Detached Entity**
- **Definition**: An entity that was previously managed but is no longer associated with a Persistence Context
- **Characteristics**: Has identity, not tracked for changes, lazy loading throws exception
- **Reattachment**: Use `em.merge()` to reattach
- **See Also**: Section 3.3 (DETACHED State)

**Domain Model**
- **Definition**: The conceptual layer of an application that captures business rules and relationships independently of persistence concerns
- **Implementation**: Entity classes with business logic
- **See Also**: Section 0.2 (Key ORM Concepts)

**DTO (Data Transfer Object)**
- **Definition**: An object that carries data between processes, typically used for read-only queries
- **Purpose**: Avoid loading full entities, reduce overhead, select only needed columns
- **See Also**: Section 13.6 (DTO Projections)

### E

**Entity**
- **Definition**: A persistent object that has its own identity and lifecycle, mapped to a database table
- **Requirements**: Must have `@Id` annotation, no-arg constructor (public or protected)
- **See Also**: Section 0.2 (Key ORM Concepts)

**Entity Graph**
- **Definition**: A declarative way to specify a fetch plan (which associations to load eagerly)
- **Usage**: `@NamedEntityGraph` annotation or dynamic `EntityGraph` API
- **See Also**: Section 13.3 (EntityGraph for Dynamic Fetching)

**Entity Key**
- **Definition**: A unique identifier used to look up entities in the Persistence Context (Identity Map)
- **Composition**: Entity class + primary key value
- **Example**: `EntityKey(Customer.class, 42L)`

**EntityManager**
- **Definition**: JPA interface for managing entity lifecycle and querying
- **Scope**: Typically one per transaction/request
- **Hibernate Equivalent**: `Session`
- **See Also**: Section 2.3 (EntityManager / Session)

**EntityManagerFactory**
- **Definition**: JPA interface for creating EntityManager instances
- **Scope**: One per application (per persistence unit)
- **Hibernate Equivalent**: `SessionFactory`
- **See Also**: Section 2.2 (EntityManagerFactory / SessionFactory)

**Fetch Join**
- **Definition**: A JOIN that loads the associated entity in the same query
- **JPQL Syntax**: `JOIN FETCH o.customer`
- **Purpose**: Solves N+1 problem by loading associations eagerly
- **See Also**: Section 13.2 (Solving N+1 with JOIN FETCH)

**Fetch Type**
- **Definition**: Strategy for when to load associations
- **Types**: EAGER (load immediately), LAZY (load on access)
- **Default**: `@ManyToOne` is EAGER, `@OneToMany` is LAZY
- **See Also**: Section 6 (Lazy Loading Mechanisms)

**First-Level Cache**
- **Definition**: Cache scoped to a single Session/EntityManager, also known as the Persistence Context
- **Characteristics**: Always enabled, stores full entity objects, implements Identity Map pattern
- **See Also**: Section 2.4 (Persistence Context)

**Flush**
- **Definition**: The process of synchronizing the Persistence Context state with the database
- **Triggers**: Explicit `em.flush()`, transaction commit, before query execution
- **Actions**: Dirty checking, ordering and executing queued SQL
- **See Also**: Section 5.8 (SQL Executed at Flush)

**Flush Mode**
- **Definition**: Controls when Hibernate automatically flushes the Persistence Context
- **Types**: AUTO (default), COMMIT, MANUAL, ALWAYS
- **See Also**: Section 9.5 (Flush Modes)

### G

**Generated Identifier**
- **Definition**: Primary key value generated by the database or Hibernate
- **Strategies**: IDENTITY (auto-increment), SEQUENCE, TABLE, AUTO
- **See Also**: Section 4.2 (ID Generation Strategies)

### H

**Hibernate**
- **Definition**: The most widely-used implementation of the JPA specification
- **Features**: Extended API, caching, interceptors, event system, native SQL
- **See Also**: Section 1 (JPA Specification vs Hibernate Implementation)

**Hydration**
- **Definition**: The process of converting database result set rows into entity objects
- **Inverse**: Dehydration (converting entity objects to database values)

### I

**Identity**
- **Definition**: The property that distinguishes one entity from another
- **In Java**: Memory address (`==`) or logical equality (`equals()`)
- **In Database**: Primary key
- **In ORM**: Primary key used as identity within Persistence Context
- **See Also**: Section 0.1 (Object-Relational Impedance Mismatch)

**Identity Map**
- **Definition**: A pattern that ensures each object is loaded only once per session
- **Implementation**: Map of EntityKey to Entity instance
- **Hibernate**: The Persistence Context IS an Identity Map
- **See Also**: Section 0.3 (Theoretical Foundations of Hibernate Architecture)

**Impedance Mismatch**
- **Definition**: The conceptual and structural differences between object-oriented programming and relational databases
- **Aspects**: Granularity, identity, inheritance, associations, navigation, state, encapsulation
- **Solution**: ORM frameworks like Hibernate
- **See Also**: Section 0.1 (What is Object-Relational Mapping?)

**Inheritance Mapping**
- **Definition**: Strategy for mapping class hierarchies to database tables
- **Strategies**: SINGLE_TABLE, JOINED, TABLE_PER_CLASS
- **See Also**: Section 10 (Inheritance Mapping Strategies)

**Insert Action**
- **Definition**: A queued operation to insert a new entity into the database
- **Queued**: When `em.persist()` is called
- **Executed**: At flush time
- **See Also**: Section 5.7 (INSERT Action Queued)

**Isolation Level**
- **Definition**: Degree to which one transaction must be isolated from other transactions
- **Types**: READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
- **Trade-off**: Higher isolation = less concurrency
- **See Also**: Section 9.2 (Transaction Isolation Levels)

### J

**JPA (Jakarta Persistence API)**
- **Definition**: Java specification for ORM, defining interfaces and annotations
- **Implementations**: Hibernate, EclipseLink, OpenJPA
- **Purpose**: Vendor-neutral API for database persistence
- **See Also**: Section 1 (JPA Specification vs Hibernate Implementation)

**JPQL (Java Persistence Query Language)**
- **Definition**: Object-oriented query language similar to SQL but operates on entities
- **Difference**: Uses entity names and property names, not table/column names
- **Example**: `SELECT o FROM Order o WHERE o.status = 'SHIPPED'`

**Join Fetch**
- **Definition**: JPQL syntax to load associations in the same query
- **Syntax**: `JOIN FETCH o.customer`
- **See Also**: Section 13.2 (Solving N+1 with JOIN FETCH)

### L

**Lazy Initialization**
- **Definition**: Deferring the loading of an object or association until it's actually accessed
- **Mechanism**: Proxies or bytecode enhancement
- **Exception**: `LazyInitializationException` if accessed outside session
- **See Also**: Section 6 (Lazy Loading Mechanisms)

**LazyInitializationException**
- **Definition**: Exception thrown when trying to access a lazy-loaded association after the session is closed
- **Cause**: Proxy has no session to execute the loading query
- **Solutions**: JOIN FETCH, EntityGraph, initialize before closing, Open Session in View
- **See Also**: Section 6.3 (LazyInitializationException)

**Lock Mode**
- **Definition**: Strategy for acquiring locks on entities during queries
- **Optimistic**: OPTIMISTIC, OPTIMISTIC_FORCE_INCREMENT (version-based)
- **Pessimistic**: PESSIMISTIC_READ, PESSIMISTIC_WRITE (database locks)
- **See Also**: Section 11 (Locking Mechanisms)

### M

**Managed Entity**
- **Definition**: An entity associated with an open Persistence Context
- **Characteristics**: Identity guarantee, dirty checking, write-behind
- **Transitions**: From Transient (persist), From Detached (merge), From Database (find/query)
- **See Also**: Section 3.2 (MANAGED State)

**Merge**
- **Definition**: Operation to reattach a detached entity and synchronize its state with the database
- **Behavior**: Returns a new managed instance; the detached instance remains detached
- **See Also**: Section 3.2 (MANAGED State)

**N+1 Problem**
- **Definition**: Performance issue where N additional queries are executed for N parent entities
- **Cause**: Lazy loading triggered in a loop
- **Solutions**: JOIN FETCH, EntityGraph, batch fetching
- **See Also**: Section 13.1 (The N+1 Problem)

### O

**Object-Relational Mapping (ORM)**
- **Definition**: Programming technique that converts data between object-oriented languages and relational databases
- **Purpose**: Bridge the impedance mismatch between objects and relations
- **See Also**: Section 0.1 (What is Object-Relational Mapping?)

**Optimistic Locking**
- **Definition**: Concurrency control strategy that assumes conflicts are rare, using version numbers to detect conflicts
- **Implementation**: `@Version` annotation
- **Conflict**: Throws `OptimisticLockException`
- **See Also**: Section 11.1 (Optimistic Locking)

**Orphan Removal**
- **Definition**: Automatic deletion of child entities when removed from the parent's collection
- **Configuration**: `orphanRemoval = true` on `@OneToMany`
- **See Also**: Section 12.3 (Orphan Removal)

**Owned Side**
- **Definition**: In a bidirectional association, the side that owns the foreign key column
- **Marked**: The side NOT marked with `mappedBy`
- **Responsibility**: Hibernate only looks at the owned side for SQL generation

### P

**Persistence Context**
- **Definition**: A set of managed entity instances within a Session, implementing the Identity Map pattern
- **Components**: Identity Map, Snapshot Map, Action Queue
- **Synonyms**: First-Level Cache, Session Cache
- **See Also**: Section 2.4 (Persistence Context)

**Pessimistic Locking**
- **Definition**: Concurrency control strategy that acquires database locks to prevent conflicts
- **Implementation**: `SELECT FOR UPDATE` or `SELECT FOR SHARE`
- **Use Case**: High contention scenarios
- **See Also**: Section 11.2 (Pessimistic Locking)

**Primary Key**
- **Definition**: Unique identifier for a database row, mapped to entity's `@Id` field
- **Types**: Simple (single column) or Composite (multiple columns)
- **Generation**: AUTO, IDENTITY, SEQUENCE, TABLE
- **See Also**: Section 4 (The @Id Annotation)

**Proxy**
- **Definition**: A runtime-generated subclass that intercepts method calls to implement lazy loading
- **Mechanism**: Subclasses the entity, overrides methods to trigger initialization
- **Limitations**: Cannot proxy final classes/methods, cannot lazily load basic fields
- **See Also**: Section 6.1 (Proxy-Based Lazy Loading)

**Propagation**
- **Definition**: In Spring's `@Transactional`, defines how transactions relate to each other
- **Types**: REQUIRED, REQUIRES_NEW, MANDATORY, SUPPORTS, NOT_SUPPORTED, NEVER, NESTED
- **See Also**: Section 9.3 (Transaction Propagation)

### Q

**Query Cache**
- **Definition**: Cache that stores the result set of queries (entity IDs, not full entities)
- **Requirement**: Requires second-level cache for the entities
- **Configuration**: `hibernate.cache.use_query_cache=true`
- **See Also**: Section 14.6 (Query Cache)

### R

**Read-Only Entity**
- **Definition**: An entity that will not be modified, allowing Hibernate to skip dirty checking
- **Configuration**: Query hint `org.hibernate.readOnly=true`
- **Benefit**: Performance improvement for read operations

**Removed Entity**
- **Definition**: A managed entity scheduled for deletion at the next flush
- **Transition**: From Managed via `em.remove()`
- **After Flush**: Becomes Transient (deleted from database)
- **See Also**: Section 3.4 (REMOVED State)

**Rollback**
- **Definition**: Cancelling a transaction, reverting all changes made during the transaction
- **Triggers**: RuntimeException, explicit `setRollbackOnly()`
- **See Also**: Section 9.4 (Transaction Rollback)

### S

**Second-Level Cache**
- **Definition**: Cache shared across all Session instances in a SessionFactory
- **Scope**: Application-wide
- **Content**: Dehydrated (serialized) entity state
- **Providers**: Ehcache, Infinispan, Hazelcast, Caffeine
- **See Also**: Section 14 (Second-Level Cache)

**Session**
- **Definition**: Hibernate's native API for entity management and querying
- **JPA Equivalent**: `EntityManager`
- **Scope**: Typically one per transaction/request
- **Features**: Extended API beyond JPA (saveOrUpdate, evict, Criteria)
- **See Also**: Section 2.3 (EntityManager / Session)

**SessionFactory**
- **Definition**: Hibernate's factory for creating Session instances
- **JPA Equivalent**: `EntityManagerFactory`
- **Scope**: One per application (per persistence unit)
- **Thread-safe**: Can be shared across threads
- **See Also**: Section 2.2 (EntityManagerFactory / SessionFactory)

**Snapshot**
- **Definition**: A copy of an entity's field values taken when it's loaded into the Persistence Context
- **Purpose**: Used for dirty checking — compare current values against snapshot
- **See Also**: Section 2.4 (Persistence Context)

**Stateless Session**
- **Definition**: A Hibernate API that bypasses the Persistence Context for bulk operations
- **Characteristics**: No first-level cache, no dirty checking, no cascade, no events
- **Use Case**: Massive inserts/updates where overhead is unnecessary
- **See Also**: Section 15.2 (Stateless Session)

**Strategy**
- **Definition**: In Hibernate, a pluggable component that handles specific functionality
- **Examples**: ID generation strategies, cache strategies, fetching strategies

### T

**Transaction**
- **Definition**: A logical unit of work that must be completed atomically (all or nothing)
- **Boundaries**: Defined by `begin()` and `commit()`/`rollback()`
- **Properties**: ACID (Atomicity, Consistency, Isolation, Durability)
- **See Also**: Section 9 (Transaction Management)

**Transient Entity**
- **Definition**: An entity created with `new` but not yet associated with a Persistence Context
- **Characteristics**: No ID (or unassigned), not tracked, no database row
- **Transition**: To Managed via `em.persist()`
- **See Also**: Section 3.1 (TRANSIENT State)

### U

**Unit of Work**
- **Definition**: Design pattern that maintains a list of objects affected by a business transaction
- **Hibernate Implementation**: The Session/EntityManager implements this pattern
- **Benefits**: Atomicity, consistency, identity map, lazy loading, dirty checking
- **See Also**: Section 0.3 (Theoretical Foundations of Hibernate Architecture)

**Update Action**
- **Definition**: A queued operation to update a modified entity in the database
- **Queued**: When dirty checking detects changes
- **Executed**: At flush time
- **See Also**: Section 5.8 (SQL Executed at Flush)

### V

**Value Object**
- **Definition**: An immutable object without identity, defined by its attributes
- **Characteristics**: No `@Id`, equality based on all fields
- **JPA Mapping**: `@Embeddable` for composition into entities
- **See Also**: Section 0.2 (Key ORM Concepts)

**Version**
- **Definition**: A field used for optimistic locking to detect concurrent modifications
- **Annotation**: `@Version`
- **Behavior**: Automatically incremented on each update
- **See Also**: Section 11.1 (Optimistic Locking)

### W

**Write-Behind**
- **Definition**: Pattern that defers database writes until the end of a transaction
- **Benefits**: Enables batching, reduces round-trips, ensures proper ordering
- **See Also**: Section 5.9 (Key Insight: Write-Behind Behavior)
