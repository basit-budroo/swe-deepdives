# Spring AOP Proxy & Transaction Management — Complete Guide

## Table of Contents
0. [AOP Fundamentals](#0-aop-fundamentals)
1. [Spring AOP Architecture](#1-spring-aop-architecture)
2. [Proxy Creation Mechanisms](#2-proxy-creation-mechanisms)
3. [Transaction Management with AOP](#3-transaction-management-with-aop)
4. [Proxy Internals Deep Dive](#4-proxy-internals-deep-dive)
5. [Common Pitfalls and Solutions](#5-common-pitfalls-and-solutions)
6. [Advanced Topics](#6-advanced-topics)

---

## 0. AOP Fundamentals

### 0.1 What is AOP?

**Aspect-Oriented Programming (AOP)** is a programming paradigm that aims to increase modularity by allowing the separation of **cross-cutting concerns**. It does so by adding additional behavior to existing code (advice) without modifying the code itself.

**The Core Problem: Cross-Cutting Concerns**

Cross-cutting concerns are aspects of a program that affect other concerns. They cannot be cleanly decomposed from the rest of the system and result in scattering or tangling.

| Traditional OOP | AOP Approach |
|-----------------|--------------|
| Concerns scattered across classes | Concerns centralized in aspects |
| Code duplication (logging, security) | Single point of maintenance |
| Business logic mixed with infrastructure | Clean separation of concerns |

**Example Without AOP:**
```java
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // Logging concern
        log.info("Creating order for customer: {}", request.getCustomerId());
        long startTime = System.currentTimeMillis();
        
        // Security concern
        if (!securityService.hasPermission("ORDER_CREATE")) {
            throw new AccessDeniedException();
        }
        
        // Transaction concern
        Transaction tx = transactionManager.begin();
        
        try {
            // Business logic
            Order order = new Order(request);
            orderRepository.save(order);
            
            tx.commit();
            log.info("Order created in {}ms", System.currentTimeMillis() - startTime);
            return order;
        } catch (Exception e) {
            tx.rollback();
            log.error("Failed to create order", e);
            throw e;
        }
    }
}
```

**Example With AOP:**
```java
@Service
public class OrderService {
    
    @Transactional
    @LogExecutionTime
    @PreAuthorize("hasPermission('ORDER_CREATE')")
    public Order createOrder(OrderRequest request) {
        // Only business logic
        Order order = new Order(request);
        return orderRepository.save(order);
    }
}
```

### 0.2 Core AOP Concepts (Simplified)

#### Real-World Analogy: Building Security System

Think of AOP like installing a security system in a building:

- **Your business code** = The rooms and offices where work happens
- **AOP aspects** = Security cameras, badge readers, alarms installed at entrances
- **Cross-cutting concerns** = Security (needed everywhere, but not part of the actual work)

---

#### Join Point (The "Where")

**Simple definition:** A specific moment in your code where something happens - usually when a method is called.

**Analogy:** Every doorway in a building is a "join point" - it's a place where you can install security measures.

```java
// Join point: when createOrder method is called
public Order createOrder(OrderRequest request) {
    // The moment this method starts running = join point
}
```

**In Spring AOP:** Join points are always when a method starts running (Spring doesn't support other types).

---

#### Pointcut (The "Which")

**Simple definition:** A rule that picks which methods to apply AOP to. It's like a filter.

**Analogy:** A rule that says "Install cameras only at main entrances, not storage closets."

```java
// Pointcut: "Apply this to ALL methods in the service package"
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}
```

**Translation:** `*.*.*(..)` means:
- First `*` = any return type
- Second `*` = any class name
- Third `*` = any method name
- `(..)` = any parameters

---

#### Advice (The "What")

**Simple definition:** The actual code that runs at the join point. It's what you want to do (log, check security, start transaction, etc.).

**Analogy:** The actual security action - checking a badge, recording video, sounding an alarm.

| Advice Type | When It Runs | Simple Example |
|-------------|--------------|----------------|
| **@Before** | Before method starts | "Check if user is logged in before letting them in" |
| **@After** | After method finishes (success or failure) | "Log that the method finished" |
| **@AfterReturning** | After method succeeds | "Record how long it took" |
| **@AfterThrowing** | After method crashes | "Send alert email about the error" |
| **@Around** | Before AND after method | "Start transaction, run method, commit transaction" |

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint joinPoint) {
    System.out.println("About to run: " + joinPoint.getSignature().getName());
}
```

---

#### Aspect (The "Package")

**Simple definition:** A class that groups together pointcuts and advice. It's a module for a specific cross-cutting concern.

**Analogy:** The entire security system package - all cameras, all badge readers, all rules in one system.

```java
@Aspect  // "This is an AOP aspect"
@Component
public class LoggingAspect {
    
    // Pointcut: which methods?
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}
    
    // Advice: what to do?
    @Before("serviceMethods()")
    public void logBefore(JoinPoint joinPoint) {
        log.info("Executing: {}", joinPoint.getSignature());
    }
}
```

---

#### Target (The "Original Object")

**Simple definition:** The actual object with your business logic. The one being "watched" by AOP.

**Analogy:** The actual room or office where work happens (not the security camera watching it).

```java
@Service
public class OrderService {
    // This is the TARGET - the real object with business logic
    public Order createOrder(OrderRequest request) {
        // Business code here
    }
}
```

---

#### Proxy (The "Wrapper")

**Simple definition:** A fake object that looks like your target but intercepts calls to run advice first. Spring creates this automatically.

**Analogy:** A security guard at the door. You think you're walking into the room, but you're actually talking to the guard first, who checks your badge before letting you in.

```
Client calls: orderService.createOrder()
                ↓
            PROXY (the guard)
                ↓
         Runs advice (checks security, starts transaction)
                ↓
         Calls TARGET.createOrder() (actual business logic)
                ↓
         Runs more advice (commits transaction, logs result)
                ↓
         Returns result to client
```

**Key point:** Your code never knows it's talking to a proxy - it just works!

---

#### Quick Summary

| Term | Simple Meaning | Real-World Analogy |
|------|----------------|-------------------|
| **Join Point** | Where AOP can hook in | A doorway |
| **Pointcut** | Which methods get AOP | Rule: "all main entrances" |
| **Advice** | What AOP does | Badge reader, camera, alarm |
| **Aspect** | Group of pointcuts + advice | Entire security system |
| **Target** | Your actual business object | The room/office |
| **Proxy** | Wrapper that intercepts calls | Security guard at door |

---

## 1. Spring AOP Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Application Layer                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Client Code                                              │  │
│  │  orderService.createOrder(request);                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Spring AOP Layer                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  AOP Proxy (JDK Dynamic Proxy or CGLIB)                   │  │
│  │  - Intercepts method calls                                │  │
│  │  - Invokes advice chain                                   │  │
│  │  - Delegates to target object                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Advisor Chain                                           │  │
│  │  - Transaction Advisor                                   │  │
│  │  - Security Advisor                                      │  │
│  │  - Logging Advisor                                       │  │
│  │  - Caching Advisor                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Target Object                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  OrderService (actual bean instance)                      │  │
│  │  - Contains business logic                                 │  │
│  │  - Unaware of AOP                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Spring AOP vs Full AOP

| Feature | Spring AOP | Full AOP (AspectJ) |
|---------|------------|-------------------|
| **Implementation** | Runtime proxies | Compile-time or load-time weaving |
| **Join Points** | Method execution only | Method execution, field access, constructor execution, etc. |
| **Proxy Type** | JDK Dynamic Proxy or CGLIB | No proxy (bytecode modification) |
| **Performance** | Slight overhead (method call indirection) | Minimal overhead (inline) |
| **Complexity** | Simpler, easier to use | More complex, requires compiler |
| **Self-invocation** | Not intercepted | Intercepted |

### 1.3 When Spring Uses Which Proxy

```
Target Object
     │
     ├─ Implements at least one interface
     │   └─ JDK Dynamic Proxy used
     │       - Proxies all interfaces
     │       - Cannot proxy class-specific methods
     │
     └─ Does NOT implement any interface
         └─ CGLIB Proxy used
             - Proxies the class
             - Subclasses the target
             - Can proxy all methods
```

```java
// Case 1: Interface-based → JDK Dynamic Proxy
public interface OrderService {
    Order createOrder(OrderRequest request);
}

@Service
public class OrderServiceImpl implements OrderService {
    @Override
    public Order createOrder(OrderRequest request) {
        // ...
    }
}
// Proxy: implements OrderService, delegates to OrderServiceImpl

// Case 2: Class-based → CGLIB Proxy
@Service
public class OrderService {
    public Order createOrder(OrderRequest request) {
        // ...
    }
}
// Proxy: extends OrderService, overrides createOrder()
```

---

## 2. Proxy Creation Mechanisms

### 2.1 JDK Dynamic Proxy

**How It Works:**
JDK Dynamic Proxy creates a proxy class at runtime that implements the specified interfaces and delegates method calls to an `InvocationHandler`.

```java
// JDK Dynamic Proxy Example (without Spring)
public interface OrderService {
    Order createOrder(OrderRequest request);
}

public class OrderServiceImpl implements OrderService {
    @Override
    public Order createOrder(OrderRequest request) {
        return new Order(request);
    }
}

// Create proxy programmatically
OrderService target = new OrderServiceImpl();

OrderService proxy = (OrderService) Proxy.newProxyInstance(
    OrderService.class.getClassLoader(),
    new Class<?>[] { OrderService.class },
    new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("Before: " + method.getName());
            Object result = method.invoke(target, args);
            System.out.println("After: " + method.getName());
            return result;
        }
    }
);

proxy.createOrder(new OrderRequest());
// Output:
// Before: createOrder
// After: createOrder
```

**Spring's Usage:**
```java
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = false)  // Use JDK proxy
public class AppConfig {
}
```

**Limitations:**
- Can only proxy interfaces
- Cannot proxy class-specific methods
- Cannot proxy final classes or methods
- Self-invocation (calling methods within the same class) bypasses proxy

### 2.2 CGLIB Proxy

**How It Works:**
CGLIB (Code Generation Library) creates a proxy by subclassing the target class and overriding methods.

```java
// CGLIB Proxy Example (without Spring)
public class OrderService {
    public Order createOrder(OrderRequest request) {
        return new Order(request);
    }
}

// Create proxy programmatically
OrderService target = new OrderService();

Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(OrderService.class);
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = proxy.invoke(target, args);
        System.out.println("After: " + method.getName());
        return result;
    }
});

OrderService proxy = (OrderService) enhancer.create();
proxy.createOrder(new OrderRequest());
// Output:
// Before: createOrder
// After: createOrder
```

**Spring's Usage:**
```java
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)  // Use CGLIB
public class AppConfig {
}
```

**Advantages:**
- Can proxy classes without interfaces
- Can proxy class-specific methods
- More flexible than JDK proxy

**Limitations:**
- Cannot proxy final classes (cannot subclass)
- Cannot proxy final methods (cannot override)
- Slight performance overhead compared to JDK proxy
- Constructor called twice (once for proxy, once for target)

### 2.3 Proxy Creation in Spring

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
    @Bean
    public OrderService orderService() {
        return new OrderServiceImpl();
    }
    
    @Bean
    public TransactionInterceptor transactionInterceptor() {
        return new TransactionInterceptor(transactionManager);
    }
}

// Spring's proxy creation process:
// 1. Scan for @Transactional annotations
// 2. Identify beans that need proxying
// 3. Determine proxy type (JDK or CGLIB)
// 4. Create proxy with advisor chain
// 5. Register proxy as bean instead of target
```

**Proxy Creation Flow:**
```
Bean Definition
     │
     ├─ Has @Transactional or other AOP annotations?
     │   ├─ Yes → Create Proxy
     │   │   ├─ Determine proxy type (JDK or CGLIB)
     │   │   ├─ Build advisor chain
     │   │   ├─ Create proxy instance
     │   │   └─ Register proxy as bean
     │   │
     │   └─ No → Register target as bean
```

---

## 3. Transaction Management with AOP

### 3.1 @Transactional Annotation Processing

```java
@Service
public class OrderService {
    
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        readOnly = false,
        rollbackFor = { Exception.class }
    )
    public Order createOrder(OrderRequest request) {
        // Business logic
    }
}
```

**Spring's Processing:**

1. **Annotation Detection**
```java
// Spring scans for @Transactional annotations
// Creates TransactionAttribute for each annotated method
TransactionAttribute attr = new DefaultTransactionAttribute();
attr.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
attr.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
attr.setTimeout(30);
attr.setReadOnly(false);
```

2. **Advisor Creation**
```java
// Spring creates BeanFactoryTransactionAttributeSourceAdvisor
Advisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor(
    transactionAttributeSource,  // Finds @Transactional annotations
    transactionInterceptor        // Handles transaction logic
);
```

3. **Proxy Creation**
```java
// Spring wraps the bean in a proxy with the advisor
OrderService proxy = (OrderService) ProxyFactory.getProxy(
    target, 
    new Advisor[] { advisor }
);
```

### 3.2 TransactionInterceptor Internals

```java
public class TransactionInterceptor implements MethodInterceptor {
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 1. Get transaction attributes from @Transactional
        TransactionAttribute txAttr = getTransactionAttributeSource()
            .getTransactionAttribute(invocation.getMethod(), invocation.getThis().getClass());
        
        // 2. Get TransactionManager
        PlatformTransactionManager tm = getTransactionManager();
        
        // 3. Join existing transaction or create new one
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr);
        
        try {
            // 4. Proceed with method invocation
            Object retVal = invocation.proceed();
            
            // 5. Commit if no exception
            commitTransactionAfterReturning(txInfo);
            
            return retVal;
            
        } catch (Throwable ex) {
            // 6. Rollback on exception
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // 7. Cleanup
            cleanupTransactionInfo(txInfo);
        }
    }
}
```

### 3.3 Transaction Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    Transaction Lifecycle                         │
│                                                                  │
│  1. Method Invocation                                            │
│     │                                                            │
│     ├─ Proxy intercepts call                                    │
│     ├─ TransactionInterceptor.invoke() called                   │
│     │                                                            │
│  2. Transaction Begin                                            │
│     │                                                            │
│     ├─ Check if transaction exists                              │
│     │   ├─ Yes: Join existing transaction                       │
│     │   └─ No: Create new transaction                           │
│     │                                                            │
│     ├─ Get connection from DataSource                            │
│     ├─ connection.setAutoCommit(false)                          │
│     ├─ Set isolation level                                      │
│     ├─ Set read-only flag                                       │
│     ├─ Set timeout                                               │
│     │                                                            │
│  3. Method Execution                                             │
│     │                                                            │
│     ├─ invocation.proceed()                                     │
│     ├─ Target method executes                                   │
│     ├─ Database operations use the same connection              │
│     │                                                            │
│  4. Transaction Commit (if no exception)                         │
│     │                                                            │
│     ├─ EntityManager.flush() (if JPA)                           │
│     ├─ connection.commit()                                      │
│     ├─ Database makes changes permanent                         │
│     ├─ Locks released                                           │
│     │                                                            │
│  5. Transaction Rollback (if exception)                          │
│     │                                                            │
│     ├─ connection.rollback()                                     │
│     ├─ Database discards changes                                │
│     ├─ Locks released                                           │
│     │                                                            │
│  6. Cleanup                                                      │
│     │                                                            │
│     ├─ Connection returned to pool                              │
│     ├─ EntityManager closed (if JPA)                            │
│     └─ Transaction synchronization cleanup                      │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Transaction Propagation in Action

```java
@Service
public class OrderService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void createOrder(OrderRequest request) {
        // Transaction A created here
        orderRepository.save(new Order(request));
        
        // Joins Transaction A
        paymentService.processPayment(request);
        
        // Joins Transaction A
        inventoryService.reserveItems(request.getItems());
        
        // Transaction A commits here
    }
}

@Service
public class PaymentService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processPayment(OrderRequest request) {
        // Transaction A suspended
        // Transaction B created here
        paymentRepository.save(new Payment(request));
        
        // Transaction B commits here
        // Transaction A resumes
    }
}
```

**Execution Flow:**
```
createOrder()
  │
  ├─ Transaction A begins
  ├─ save(order) → in Transaction A
  ├─ processPayment()
  │   ├─ Transaction A suspended
  │   ├─ Transaction B begins
  │   ├─ save(payment) → in Transaction B
  │   ├─ Transaction B commits
  │   └─ Transaction A resumes
  ├─ reserveItems() → in Transaction A
  └─ Transaction A commits
```

---

## 4. Proxy Internals Deep Dive

### 4.1 Generated Proxy Class Structure

**JDK Dynamic Proxy Class (simplified):**
```java
// Generated by Proxy.newProxyInstance()
public final class $Proxy0 extends Proxy implements OrderService {
    
    private static Method m0;
    private static Method m1;
    
    static {
        try {
            m0 = OrderService.class.getMethod("createOrder", OrderRequest.class);
            m1 = Object.class.getMethod("equals", Object.class);
        } catch (NoSuchMethodException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
    
    public $Proxy0(InvocationHandler h) {
        super(h);
    }
    
    @Override
    public Order createOrder(OrderRequest request) {
        try {
            return (Order) h.invoke(this, m0, new Object[] { request });
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
    
    @Override
    public boolean equals(Object obj) {
        try {
            return (Boolean) h.invoke(this, m1, new Object[] { obj });
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
}
```

**CGLIB Proxy Class (simplified):**
```java
// Generated by Enhancer
public class OrderService$$EnhancerBySpringCGLIB$$12345678 extends OrderService {
    
    private Dispatcher dispatcher;
    private MethodInterceptor interceptor;
    
    static {
        // CGLIB initializes method indices
    }
    
    public OrderService$$EnhancerBySpringCGLIB$$12345678() {
        // Calls super constructor
    }
    
    @Override
    public Order createOrder(OrderRequest request) {
        try {
            // Calls interceptor instead of super method
            return (Order) interceptor.intercept(
                this,
                CGLIB$createOrder$0,
                new Object[] { request },
                new CGLIB$createOrder$0$Proxy()
            );
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
    
    // Original method (for calling from interceptor)
    final Order CGLIB$createOrder$0(OrderRequest request) {
        return super.createOrder(request);
    }
    
    // equals(), hashCode(), toString() also overridden
}
```

### 4.2 Advisor Chain Execution

```java
// Multiple advisors on a single method
@Service
@Transactional
@Cacheable
@PreAuthorize("hasRole('ADMIN')")
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // Business logic
    }
}

// Advisor chain order (from outermost to innermost):
// 1. Security Advisor (outermost)
// 2. Transaction Advisor
// 3. Caching Advisor
// 4. Target method (innermost)
```

**Execution Flow:**
```
Client Call
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Security Advisor (outermost)                                  │
│ - Check @PreAuthorize                                         │
│ - If fails, throw AccessDeniedException                       │
│ - If passes, proceed to next advisor                          │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Transaction Advisor                                           │
│ - Begin transaction                                           │
│ - Proceed to next advisor                                     │
│ - On return: commit                                           │
│ - On exception: rollback                                      │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Caching Advisor                                                │
│ - Check cache                                                 │
│ - If hit, return cached value (skip target)                   │
│ - If miss, proceed to target                                  │
│ - On return: cache result                                     │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Target Method                                                  │
│ - Execute business logic                                      │
│ - Return result                                               │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
Result propagates back through advisors
```

### 4.3 ReflectiveMethodInvocation

```java
// Spring's MethodInvocation implementation
public class ReflectiveMethodInvocation implements MethodInvocation {
    
    private final Object target;
    private final Method method;
    private final Object[] arguments;
    private final List<InterceptorAndDynamicMethodMatcher> interceptors;
    private int currentInterceptorIndex = -1;
    
    @Override
    public Object proceed() throws Throwable {
        // Start from -1, so first call increments to 0
        if (this.currentInterceptorIndex == this.interceptors.size() - 1) {
            // All advisors executed, invoke target
            return invokeJoinpoint();
        }
        
        // Get next advisor
        InterceptorAndDynamicMethodMatcher interceptorAndDynamicMethodMatcher =
            this.interceptors.get(++this.currentInterceptorIndex);
        
        MethodInterceptor interceptor = interceptorAndDynamicMethodMatcher.interceptor;
        
        // Invoke advisor
        return interceptor.invoke(this);
    }
    
    protected Object invokeJoinpoint() throws Throwable {
        // Invoke actual target method
        return this.method.invoke(this.target, this.arguments);
    }
}
```

---

## 5. Common Pitfalls and Solutions

### 5.1 Self-Invocation Problem

**The Problem:**
When a method calls another method within the same class, the call bypasses the proxy.

```java
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(OrderRequest request) {
        // This method is proxied, transaction works
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Self-invocation - bypasses proxy!
        sendNotification(order);  // ← Not transactional!
    }
    
    @Transactional
    public void sendNotification(Order order) {
        // This method is NOT called through proxy
        // Transaction annotation is ignored
        notificationRepository.save(new Notification(order));
    }
}
```

**Why It Happens:**
```
Client
  │
  ▼
Proxy (OrderService$$EnhancerBySpringCGLIB$$12345678)
  │
  ├─ createOrder() → Interceptor chain → Target.createOrder()
  │                                              │
  │                                              └─ sendNotification()
  │                                                 Direct call to target,
  │                                                 not through proxy
```

**Solution 1: Self-Injection**
```java
@Service
public class OrderService {
    
    @Autowired
    private OrderService self;  // Inject proxy reference
    
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Call through proxy
        self.sendNotification(order);  // ← Transactional!
    }
    
    @Transactional
    public void sendNotification(Order order) {
        notificationRepository.save(new Notification(order));
    }
}
```

**Solution 2: AopContext**
```java
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)
public class AppConfig {
}

@Service
public class OrderService {
    
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Get current proxy
        OrderService proxy = (OrderService) AopContext.currentProxy();
        proxy.sendNotification(order);  // ← Transactional!
    }
    
    @Transactional
    public void sendNotification(Order order) {
        notificationRepository.save(new Notification(order));
    }
}
```

**Solution 3: Extract to Separate Service**
```java
@Service
public class OrderService {
    
    @Autowired
    private NotificationService notificationService;
    
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Call different service (always through proxy)
        notificationService.sendNotification(order);  // ← Transactional!
    }
}

@Service
public class NotificationService {
    
    @Transactional
    public void sendNotification(Order order) {
        notificationRepository.save(new Notification(order));
    }
}
```

### 5.2 Final Classes/Methods

**The Problem:**
CGLIB cannot proxy final classes or methods.

```java
@Service
public final class OrderService {  // ← Cannot be proxied by CGLIB
    
    @Transactional
    public final Order createOrder(OrderRequest request) {  // ← Cannot be overridden
        return orderRepository.save(new Order(request));
    }
}
```

**Solution:**
Remove `final` modifier from classes and methods that need AOP.

```java
@Service
public class OrderService {  // ← Can be proxied
    
    @Transactional
    public Order createOrder(OrderRequest request) {  // ← Can be overridden
        return orderRepository.save(new Order(request));
    }
}
```

### 5.3 Private Methods

**The Problem:**
Spring AOP cannot intercept private methods.

```java
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        processOrder(order);  // ← Private, not intercepted
    }
    
    @Transactional
    private void processOrder(Order order) {  // ← Annotation ignored
        // Transaction won't work
    }
}
```

**Solution:**
Use package-private or protected methods instead of private.

```java
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        processOrder(order);  // ← Package-private, intercepted
    }
    
    @Transactional
    void processOrder(Order order) {  // ← Works
        // Transaction works
    }
}
```

### 5.4 Constructor Issues with CGLIB

**The Problem:**
CGLIB creates a subclass, so the constructor is called twice.

```java
@Service
public class OrderService {
    
    public OrderService() {
        System.out.println("Constructor called");
        // This prints twice with CGLIB proxy
    }
}
```

**Solution:**
Avoid expensive initialization in constructors. Use `@PostConstruct` instead.

```java
@Service
public class OrderService {
    
    public OrderService() {
        // Lightweight constructor
    }
    
    @PostConstruct
    public void init() {
        // Expensive initialization here
        // Called only once
    }
}
```

---

## 6. Advanced Topics

### 6.1 Custom Advisors

```java
@Component
public class CustomAdvisor implements PointcutAdvisor {
    
    @Override
    public Pointcut getPointcut() {
        return new AspectJExpressionPointcut() {{
            setExpression("@annotation(com.example.CustomAnnotation)");
        }};
    }
    
    @Override
    public Advice getAdvice() {
        return new MethodInterceptor() {
            @Override
            public Object invoke(MethodInvocation invocation) throws Throwable {
                System.out.println("Custom advice before");
                Object result = invocation.proceed();
                System.out.println("Custom advice after");
                return result;
            }
        };
    }
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

### 6.2 Introduction (Mixin) with AOP

```java
// Introduce new interface to existing beans
public interface Auditable {
    void setCreatedBy(String user);
    String getCreatedBy();
}

@Aspect
@Component
public class AuditableIntroduction {
    
    @DeclareParents(
        value = "com.example.service.*",
        defaultImpl = DefaultAuditable.class
    )
    public static Auditable mixin;
}

// Now all service beans implement Auditable
orderService.setCreatedBy("john");
```

### 6.3 Load-Time Weaving (AspectJ)

For scenarios where Spring AOP is insufficient (e.g., field access, constructor interception):

```java
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}

// Requires JVM agent:
// -javaagent:/path/to/spring-instrument.jar
```

### 6.4 Compile-Time Weaving (AspectJ)

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.14.0</version>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

---

## Summary

**Key Concepts:**
- Spring AOP uses runtime proxies (JDK Dynamic Proxy or CGLIB)
- Proxies intercept method calls and execute advice chains
- @Transactional is implemented via TransactionInterceptor advisor
- Self-invocation bypasses proxy (use self-injection or extract service)
- CGLIB cannot proxy final classes/methods
- Private methods cannot be intercepted

**Proxy Selection:**
- Interface-based beans → JDK Dynamic Proxy
- Class-based beans → CGLIB Proxy
- Force CGLIB: `@EnableAspectJAutoProxy(proxyTargetClass = true)`

**Transaction Flow:**
1. Proxy intercepts method call
2. TransactionInterceptor begins transaction
3. Method executes
4. TransactionInterceptor commits or rolls back
5. Connection returned to pool

**Best Practices:**
- Avoid self-invocation (use separate service or self-injection)
- Don't use final on classes/methods needing AOP
- Use @PostConstruct for initialization
- Keep transactions short
- Understand advisor ordering
