# Spring Framework — Complete Deep Dive Guide

## Table of Contents
0. [Introduction to Spring](#0-introduction-to-spring)
1. [Core Concepts: IoC and DI](#1-core-concepts-ioc-and-di)
2. [Spring Container Architecture](#2-spring-container-architecture)
3. [Bean Definition and Lifecycle](#3-bean-definition-and-lifecycle)
4. [Dependency Injection Mechanisms](#4-dependency-injection-mechanisms)
5. [Spring Annotations Comprehensive Guide](#5-spring-annotations-comprehensive-guide)
6. [Configuration Methods](#6-configuration-methods)
7. [Spring Boot Fundamentals](#7-spring-boot-fundamentals)
8. [Spring MVC Internals](#8-spring-mvc-internals)
9. [Spring Data JPA](#9-spring-data-jpa)
10. [Spring Security Fundamentals](#10-spring-security-fundamentals)
11. [Advanced Spring Topics](#11-advanced-spring-topics)
12. [Spring Performance and Best Practices](#12-spring-performance-and-best-practices)
13. [Glossary of Key Terms](#13-glossary-of-key-terms)

---

## 0. Introduction to Spring

### 0.1 What is Spring Framework?

**Spring Framework** is a comprehensive programming and configuration model for modern Java-based enterprise applications. It provides infrastructure support for developing Java applications, focusing on productivity and runtime performance.

**The Problem Spring Solves:**

Before Spring (early 2000s), Java EE (J2EE) development was complex and cumbersome:

```java
// Before Spring - JNDI lookup pattern (complex and error-prone)
public class OrderService {
    private OrderRepository orderRepository;
    
    public OrderService() throws NamingException {
        // JNDI lookup - tight coupling to application server
        InitialContext ctx = new InitialContext();
        this.orderRepository = (OrderRepository) ctx.lookup("java:comp/env/OrderRepository");
    }
    
    public void createOrder(Order order) {
        // Manual transaction management
        UserTransaction ut = (UserTransaction) new InitialContext()
            .lookup("java:comp/UserTransaction");
        try {
            ut.begin();
            orderRepository.save(order);
            ut.commit();
        } catch (Exception e) {
            ut.rollback();
            throw new RuntimeException(e);
        }
    }
}
```

**Problems with this approach:**
- **Heavy reliance on JNDI** - Required application server, difficult to test outside container
- **Complex XML configuration** - Hundreds of lines of deployment descriptors
- **Difficult unit testing** - Tight coupling to container, requires mocking JNDI
- **Boilerplate code** - Repeated patterns for lookups, transactions, error handling
- **Inconsistent APIs** - Different APIs for different technologies (EJB, JMS, JDBC)
- **Checked exceptions** - Forced to handle many checked exceptions
- **Invasive APIs** - Business code forced to extend framework classes

**Spring's Solution:**

```java
// With Spring - clean and testable
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    
    // Dependency injection - no JNDI lookups
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    // Declarative transaction management
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        // Transaction automatically committed/rolled back
    }
}
```

**Spring's Core Principles:**

1. **Inversion of Control (IoC)**: Container manages object lifecycle, not the application
2. **Dependency Injection (DI)**: Loose coupling through dependency injection
3. **Aspect-Oriented Programming (AOP)**: Cross-cutting concerns separated from business logic
4. **Simplified Testing**: Easy unit and integration testing with POJOs
5. **Consistent Abstraction**: Unified API for various technologies
6. **Non-invasive**: Business classes don't need to extend framework classes
7. **POJO-based**: Plain Old Java Objects, no special interfaces required

### 0.2 Spring Module Architecture

Spring is modular, allowing you to pick and choose which modules to use. This modular design reduces application bloat and keeps dependencies minimal.

**Module Dependencies:**
```
Core Container (Required)
    │
    ├─ AOP (Optional - depends on Core)
    ├─ Aspects (Optional - depends on Core, AOP)
    ├─ Instrument (Optional - depends on Core)
    ├─ Messaging (Optional - depends on Core)
    ├─ Data Access (Optional - depends on Core)
    │   ├─ JDBC (depends on Core, TX)
    │   ├─ TX (depends on Core)
    │   ├─ ORM (depends on Core, TX)
    │   └─ OXM (depends on Core)
    ├─ Web (Optional - depends on Core)
    │   ├─ Web MVC (depends on Web, Core)
    │   └─ WebSocket (depends on Web)
    └─ Test (Optional - depends on Core)
```

**Module Details:**

**1. Core Container (spring-core, spring-beans, spring-context, spring-expression)**
- **spring-core**: IoC and Dependency Injection basics, utilities
- **spring-beans**: BeanFactory, BeanDefinition, bean instantiation
- **spring-context**: ApplicationContext, event publishing, i18n, resource loading
- **spring-expression**: Spring Expression Language (SpEL) for dynamic queries

**2. AOP and Instrumentation (spring-aop, spring-aspects, spring-instrument)**
- **spring-aop**: AOP Alliance interface, proxy-based AOP
- **spring-aspects**: Integration with AspectJ
- **spring-instrument**: Classloader-based instrumentation for byte-code weaving

**3. Messaging (spring-messaging, spring-jms)**
- **spring-messaging**: Message abstraction, channels, patterns
- **spring-jms**: JMS integration, message listener containers

**4. Data Access / Integration (spring-jdbc, spring-tx, spring-orm, spring-oxm, spring-jms)**
- **spring-jdbc**: JdbcTemplate, NamedParameterJdbcTemplate, exception translation
- **spring-tx**: Transaction management, programmatic and declarative
- **spring-orm**: Hibernate, JPA, JDO, iBatis integration
- **spring-oxm**: Object/XML mapping (JAXB, Castor, XStream)

**5. Web Layer (spring-web, spring-webmvc, spring-websocket)**
- **spring-web**: Web application context, multipart file upload, web utilities
- **spring-webmvc**: Spring MVC framework, controllers, view resolution
- **spring-websocket**: WebSocket support, STOMP messaging

**6. Test (spring-test)**
- Unit testing support, integration testing with Spring context
- Mock MVC for web layer testing
- Test context framework, test execution listeners

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Framework                              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Core Container                                          │  │
│  │  - spring-core                                           │  │
│  │  - spring-beans                                          │  │
│  │  - spring-context                                        │  │
│  │  - spring-expression (SpEL)                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  AOP and Instrumentation                                  │  │
│  │  - spring-aop                                             │  │
│  │  - spring-aspects                                         │  │
│  │  - spring-instrument                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Messaging                                                │  │
│  │  - spring-messaging                                       │  │
│  │  - spring-jms                                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Data Access / Integration                               │  │
│  │  - spring-jdbc                                            │  │
│  │  - spring-tx                                              │  │
│  │  - spring-orm                                             │  │
│  │  - spring-oxm                                             │  │
│  │  - spring-jms                                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Web Layer                                                │  │
│  │  - spring-web                                             │  │
│  │  - spring-webmvc                                          │  │
│  │  - spring-websocket                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Test                                                     │  │
│  │  - spring-test                                            │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Core Concepts: IoC and DI

### 1.1 Inversion of Control (IoC)

**Inversion of Control (IoC)** is a design principle where the control flow of a program is inverted: instead of the programmer controlling the flow (calling libraries), the framework calls the programmer's code.

**Theoretical Foundation:**

IoC is also known as the **Hollywood Principle** - "Don't call us, we'll call you." This is a fundamental shift from traditional imperative programming where the application code controls the flow.

**Traditional Control Flow (Library Call):**
```
Application Code
     │
     ├─ Creates objects
     ├─ Manages dependencies
     ├─ Calls library methods
     └─ Controls flow
```

**IoC Control Flow (Framework Callback):**
```
Framework
     │
     ├─ Creates objects
     ├─ Manages dependencies
     ├─ Calls application code
     └─ Controls flow
```

**Concrete Example Without IoC:**
```java
// Traditional approach: Tight coupling, hard to test
public class OrderService {
    private OrderRepository orderRepository;
    private PaymentService paymentService;
    private NotificationService notificationService;
    
    public OrderService() {
        // Direct instantiation - tight coupling
        this.orderRepository = new JdbcOrderRepository();
        this.paymentService = new StripePaymentService();
        this.notificationService = new EmailNotificationService();
    }
    
    public void createOrder(Order order) {
        // Business logic mixed with infrastructure
        orderRepository.save(order);
        paymentService.process(order.getPayment());
        notificationService.send(order.getCustomer().getEmail());
    }
}
```

**Problems:**
- Cannot change implementation without recompiling
- Cannot test with mock implementations
- Violates Single Responsibility Principle
- Hard to reuse in different contexts

**Concrete Example With IoC:**
```java
// IoC approach: Loose coupling, easy to test
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    // Dependencies injected - loose coupling
    public OrderService(OrderRepository orderRepository,
                        PaymentService paymentService,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
    
    public void createOrder(Order order) {
        // Pure business logic
        orderRepository.save(order);
        paymentService.process(order.getPayment());
        notificationService.send(order.getCustomer().getEmail());
    }
}
```

**Benefits:**
- Can change implementation via configuration
- Can test with mock implementations
- Follows Single Responsibility Principle
- Reusable in different contexts

**IoC Implementation Patterns:**

1. **Dependency Injection (DI)** - Dependencies passed in (Spring's approach)
2. **Service Locator** - Dependencies looked up from registry
3. **Template Method** - Framework calls abstract methods
4. **Strategy Pattern** - Algorithm selected at runtime

**Spring's IoC Container:**

Spring implements IoC through its **IoC Container**, which:
- Instantiates, configures, and assembles objects
- Manages object lifecycle
- Handles dependency resolution
- Provides configuration metadata (XML, annotations, Java code)

**IoC vs DI:**

| Aspect | IoC | DI |
|--------|-----|-----|
| **Type** | Design principle | Pattern to implement IoC |
| **Scope** | Broader principle | Specific technique |
| **Implementation** | Can be DI, Service Locator, etc. | One way to implement IoC |
| **Spring** | Uses IoC principle | Implements DI pattern |

### 1.2 Dependency Injection (DI)

**Dependency Injection (DI)** is a pattern to implement IoC, where dependencies are "injected" into a class rather than the class creating them.

**Theoretical Foundation:**

DI follows the **Dependency Inversion Principle** from SOLID:
- High-level modules should not depend on low-level modules
- Both should depend on abstractions
- Abstractions should not depend on details
- Details should depend on abstractions

**DI Types in Detail:**

**1. Constructor Injection (Recommended)**

Constructor injection passes dependencies through the class constructor.

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    // Constructor injection - ensures all dependencies are available
    public OrderService(OrderRepository orderRepository, 
                        PaymentService paymentService,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
}
```

**Advantages:**
- **Immutable dependencies** - Final fields guarantee dependencies can't be changed
- **Clearly shows required dependencies** - Constructor signature reveals all requirements
- **Easy to test** - Can't create instance without providing all dependencies
- **Enables circular dependency detection** - Spring detects circular dependencies at startup
- **Thread-safe by default** - Immutable objects are inherently thread-safe
- **Validates dependencies at construction** - Fail fast if dependencies are missing
- **Works well with Lombok** - `@RequiredArgsConstructor` generates constructor

**When to Use:**
- For required dependencies (non-null)
- For immutable objects
- For thread-safe services
- For domain objects with invariants

**Spring 4.3+ Enhancement:**
```java
// Before Spring 4.3: Required @Autowired
@Autowired
public OrderService(OrderRepository orderRepository) {
    this.orderRepository = orderRepository;
}

// Spring 4.3+: @Autowired optional on single constructor
public OrderService(OrderRepository orderRepository) {
    this.orderRepository = orderRepository;
}
```

**2. Setter Injection**

Setter injection passes dependencies through setter methods.

```java
@Service
public class OrderService {
    private OrderRepository orderRepository;
    private PaymentService paymentService;
    
    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Advantages:**
- **Optional dependencies** - Can leave unset if not needed
- **Can change dependencies after construction** - Allows reconfiguration
- **Works with mutable dependencies** - Supports state changes
- **Supports circular dependencies** - Resolves some circular dependency scenarios
- **Clear method naming** - `setXxx` pattern is recognizable

**Disadvantages:**
- **Dependencies can be null** - No guarantee of initialization
- **Less explicit** - Harder to see required dependencies at a glance
- **Not thread-safe by default** - Mutable state requires synchronization
- **Requires null checks** - Must validate dependencies before use
- **Can be incomplete** - Not all setters may be called

**When to Use:**
- For optional dependencies
- For dependencies that may change during runtime
- For legacy code migration
- For JMX-managed beans
- For prototype-scoped beans

**3. Field Injection (Not Recommended)**

Field injection uses reflection to set dependencies directly on fields.

```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // ← Avoid this
    
    @Autowired
    private PaymentService paymentService;
}
```

**Disadvantages:**
- **Cannot make fields final** - Dependencies can be reassigned
- **Harder to test** - Requires reflection or Spring test context
- **Hides dependencies** - Dependencies not visible in constructor
- **Cannot use constructor validation** - No validation at construction time
- **Not supported in Spring Boot 3.x by default** - Requires explicit configuration
- **Violates encapsulation** - Direct field access bypasses encapsulation
- **Prevents immutability** - Cannot create truly immutable objects
- **Difficult to reason about** - Hidden dependencies make code harder to understand

**When (Rarely) to Use:**
- Legacy code that cannot be refactored
- Framework code where constructor injection is not possible
- Prototype-scoped beans with no constructor

**Why Spring Boot 3.x Discourages Field Injection:**
Spring Boot 3.x removed constructor injection requirement for single constructors, making constructor injection the default and de-emphasizing field injection.

---

## 2. Spring Container Architecture

### 2.1 ApplicationContext Overview

The **ApplicationContext** is the central interface in Spring that provides configuration for an application. It is built on top of BeanFactory and adds more enterprise-specific features.

**BeanFactory vs ApplicationContext:**

**BeanFactory** is the root interface for Spring's IoC container. It provides:
- Basic bean instantiation and wiring
- Lazy loading of beans (beans created only when requested)
- Minimal features, lightweight

**ApplicationContext** extends BeanFactory and adds:
- Eager loading of singleton beans (beans created at startup)
- Internationalization (i18n) support via MessageSource
- Event publishing via ApplicationEventPublisher
- Resource loading via ResourceLoader
- Environment abstraction (profiles, properties)
- Enterprise-specific features (JNDI, EJB integration)

**When to Use BeanFactory:**
- Resource-constrained environments (embedded systems)
- When you need lazy initialization
- When you don't need enterprise features

**When to Use ApplicationContext:**
- Enterprise applications
- When you need event publishing
- When you need i18n support
- When you need resource loading
- Most Spring applications (default choice)

```
┌─────────────────────────────────────────────────────────────────┐
│                    ApplicationContext Hierarchy                    │
│                                                                  │
│  BeanFactory (interface)                                         │
│      │                                                            │
│      ├─ HierarchicalBeanFactory                                 │
│      │   │                                                        │
│      │   └─ ConfigurableBeanFactory                              │
│      │       │                                                    │
│      │       └─ AbstractBeanFactory                             │
│      │           │                                                │
│      │           ├─ DefaultListableBeanFactory                   │
│      │           │                                                │
│      │           └─ AbstractRefreshableApplicationContext        │
│      │               │                                            │
│      │               └─ ApplicationContext (interface)             │
│      │                   │                                        │
│      │                   ├─ ConfigurableApplicationContext       │
│      │                   │                                        │
│      │                   ├─ WebApplicationContext (interface)     │
│      │                   │   │                                    │
│      │                   │   └─ ...                              │
│      │                   │                                        │
│      │                   ├─ AnnotationConfigApplicationContext   │
│      │                   │                                        │
│      │                   ├─ ClassPathXmlApplicationContext       │
│      │                   │                                        │
│      │                   └─ FileSystemXmlApplicationContext      │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 ApplicationContext Features in Detail

| Feature | Description | Internal Mechanism |
|---------|-------------|-------------------|
| **Bean Instantiation** | Creates and configures beans | Reflection, ConstructorResolver, AutowireResolver |
| **Dependency Injection** | Wires beans together | DependencyDescriptor, AutowiredAnnotationBeanPostProcessor |
| **Bean Lifecycle Management** | Manages bean lifecycle callbacks | BeanPostProcessors, InitializingBean, DisposableBean |
| **Internationalization** | Message source for i18n | MessageSource, ResourceBundleMessageSource |
| **Event Publishing** | Application event mechanism | ApplicationEventMulticaster, Observer pattern |
| **Resource Loading** | Loads resources from various sources | ResourceLoader, ResourcePatternResolver |
| **Environment Abstraction** | Profiles and properties | Environment, PropertySources, ConfigurableEnvironment |

**ApplicationContext Internals:**

```java
// ApplicationContext internal structure (simplified)
public abstract class AbstractApplicationContext {
    
    // Bean definition registry
    private final DefaultListableBeanFactory beanFactory;
    
    // Message source for i18n
    private MessageSource messageSource;
    
    // Event multicaster
    private ApplicationEventMulticaster applicationEventMulticaster;
    
    // Resource pattern resolver
    private ResourcePatternResolver resourcePatternResolver;
    
    // Environment abstraction
    private ConfigurableEnvironment environment;
    
    // Lifecycle processors
    private List<LifecycleProcessor> lifecycleProcessors;
    
    // Bean post processors
    private final List<BeanPostProcessor> beanPostProcessors;
}
```

**BeanFactory Internals:**

```java
// DefaultListableBeanFactory internal structure (simplified)
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory {
    
    // Bean definition registry: bean name -> BeanDefinition
    private final Map<String, BeanDefinition> beanDefinitionMap;
    
    // Singleton bean registry: bean name -> bean instance
    private final Map<String, Object> singletonObjects;
    
    // Early singleton bean registry (for circular dependencies)
    private final Map<String, Object> earlySingletonObjects;
    
    // Singleton bean factories (for circular dependencies)
    private final Map<String, ObjectFactory<?>> singletonFactories;
    
    // Bean post processors
    private final List<BeanPostProcessor> beanPostProcessors;
    
    // Autowire candidate resolver
    private AutowireCandidateResolver autowireCandidateResolver;
}
```

### 2.3 ApplicationContext Implementations

**1. AnnotationConfigApplicationContext**

Used for Java-based configuration:

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}

// Create context
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService orderService = context.getBean(OrderService.class);
```

**2. ClassPathXmlApplicationContext**

Used for XML-based configuration:

```java
// Create context from XML
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
OrderService orderService = context.getBean(OrderService.class);
```

**3. FileSystemXmlApplicationContext**

Used for XML configuration from file system:

```java
// Create context from file system path
ApplicationContext context = new FileSystemXmlApplicationContext("/path/to/applicationContext.xml");
```

**4. GenericWebApplicationContext**

Used for web applications:

```java
// In Spring MVC
public class WebAppInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) {
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);
        context.setServletContext(servletContext);
        context.refresh();
    }
}
```

### 2.4 Hierarchical Contexts

Spring supports hierarchical application contexts, allowing parent-child relationships.

**Hierarchy Benefits:**
- Child contexts can access beans from parent contexts
- Parent contexts cannot access beans from child contexts
- Enables modular application design
- Common beans defined once in parent context

**Example:**

```java
// Parent context (root)
AnnotationConfigApplicationContext parentContext = new AnnotationConfigApplicationContext();
parentContext.register(DataSourceConfig.class, SecurityConfig.class);
parentContext.refresh();

// Child context (web)
AnnotationConfigWebApplicationContext childContext = new AnnotationConfigWebApplicationContext();
childContext.setParent(parentContext);
childContext.register(WebConfig.class, ServiceConfig.class);
childContext.refresh();
```

**Bean Resolution in Hierarchy:**
```
Child Context
    │
    ├─ Look for bean in child context
    │   ├─ Found? Return bean
    │   └─ Not found? Continue to parent
    │
    └─ Look for bean in parent context
        ├─ Found? Return bean
        └─ Not found? Throw NoSuchBeanDefinitionException
```

**Use Cases:**
- **Root context**: Infrastructure beans (DataSource, TransactionManager)
- **Child context**: Web-specific beans (Controllers, ViewResolvers)
- **Multiple child contexts**: Different modules or applications sharing common infrastructure

---

## 3. Bean Definition and Lifecycle

### 3.1 Bean Definition

A **BeanDefinition** is the recipe for creating a bean instance. It contains configuration metadata that tells Spring how to create, configure, and manage a bean.

**BeanDefinition Properties:**

```java
// BeanDefinition structure
BeanDefinition beanDefinition = BeanDefinitionBuilder
    .rootBeanDefinition(OrderService.class)
    .setScope(BeanDefinition.SCOPE_SINGLETON)
    .setLazyInit(false)
    .setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE)
    .setDependencyChecker(new SimpleAutowireCandidateResolver())
    .addPropertyValue("timeout", 30)
    .addConstructorArgValue("default")
    .setInitMethodName("init")
    .setDestroyMethodName("cleanup")
    .setPrimary(false)
    .setRole(BeanDefinition.ROLE_APPLICATION)
    .setDescription("Order service bean")
    .getBeanDefinition();
```

**BeanDefinition Internal Structure:**

```java
// AbstractBeanDefinition internal structure (simplified)
public abstract class AbstractBeanDefinition implements BeanDefinition {
    
    // Bean class name
    private volatile Object beanClass;
    
    // Scope: singleton, prototype, etc.
    private String scope = SCOPE_DEFAULT;
    
    // Lazy initialization flag
    private Boolean lazyInit;
    
    // Autowire mode: no, by_name, by_type, constructor
    private int autowireMode = AUTOWIRE_NO;
    
    // Dependency check mode
    private int dependencyCheck = DEPENDENCY_CHECK_NONE;
    
    // Constructor argument values
    private ConstructorArgumentValues constructorArgumentValues;
    
    // Property values
    private MutablePropertyValues propertyValues;
    
    // Method to call after initialization
    private String initMethodName;
    
    // Method to call before destruction
    private String destroyMethodName;
    
    // Whether this bean is primary candidate for autowiring
    private boolean primary = false;
    
    // Bean role: application, infrastructure, support
    private int role = ROLE_APPLICATION;
    
    // Bean description
    private String description;
    
    // Whether bean is synthetic (framework-generated)
    private boolean synthetic = false;
}
```

**BeanDefinition Sources:**

1. **Component Scanning**: `@Component` and stereotype annotations
2. **@Bean Methods**: Java configuration methods
3. **XML Configuration**: `<bean>` elements
4. **Groovy Bean Definition DSL**: Groovy-based configuration
5. **Programmatic**: Direct BeanDefinition creation

**BeanDefinition Registry:**

The `BeanDefinitionRegistry` is the interface for registering bean definitions:

```java
public interface BeanDefinitionRegistry extends AliasRegistry {
    
    // Register a new bean definition
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition);
    
    // Remove a bean definition
    void removeBeanDefinition(String beanName);
    
    // Get a bean definition
    BeanDefinition getBeanDefinition(String beanName);
    
    // Check if bean definition exists
    boolean containsBeanDefinition(String beanName);
    
    // Get all bean names
    String[] getBeanDefinitionNames();
    
    // Get bean names by type
    String[] getBeanNamesForType(Class<?> type);
}
```

### 3.2 Bean Scopes

Bean scope determines the lifecycle and visibility of a bean instance.

**Scope Comparison:**

| Scope | Description | Default | Thread-Safe | Use Case |
|-------|-------------|---------|-------------|----------|
| `singleton` | One instance per container | Yes | Requires synchronization | Stateless services, shared resources |
| `prototype` | New instance each time requested | No | Each instance independent | Stateful objects, commands |
| `request` | One instance per HTTP request (web) | No | Request-scoped | Request-specific data |
| `session` | One instance per HTTP session (web) | No | Session-scoped | User session data |
| `application` | One instance per ServletContext (web) | No | Application-scoped | Shared web resources |
| `websocket` | One instance per WebSocket (web) | No | WebSocket-scoped | WebSocket connections |

**Singleton Scope (Default):**

```java
@Service
@Scope("singleton")  // or omit annotation
public class OrderService {
    private final OrderRepository orderRepository;
    
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    public Order createOrder(Order order) {
        return orderRepository.save(order);
    }
}
```

**Characteristics:**
- One instance per Spring container
- Created at container startup (unless lazy)
- Shared across all requests
- Must be thread-safe (use synchronization or stateless design)

**Prototype Scope:**

```java
@Service
@Scope("prototype")
public class OrderProcessor {
    private Order currentOrder;
    
    public void process(Order order) {
        this.currentOrder = order;
        // Process order
    }
}
```

**Characteristics:**
- New instance each time requested
- Created on demand (not at startup)
- Not managed by container after creation
- Client responsible for cleanup
- Not thread-safe (each instance independent)

**Request Scope (Web):**

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart {
    private List<OrderItem> items = new ArrayList<>();
    
    public void addItem(OrderItem item) {
        items.add(item);
    }
    
    public List<OrderItem> getItems() {
        return items;
    }
}
```

**Characteristics:**
- One instance per HTTP request
- Automatically cleared at end of request
- Requires proxy mode for injection into singletons
- Thread-safe within request

**Session Scope (Web):**

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserPreferences {
    private String theme = "light";
    private String language = "en";
    
    // getters and setters
}
```

**Characteristics:**
- One instance per HTTP session
- Persists across multiple requests
- Requires proxy mode for injection into singletons
- Session invalidation destroys bean

**Custom Scope:**

You can define custom scopes:

```java
public class ThreadScope implements Scope {
    
    private final ThreadLocal<Map<String, Object>> threadScope = 
        ThreadLocal.withInitial(HashMap::new);
    
    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        return threadScope.get().computeIfAbsent(name, k -> objectFactory.getObject());
    }
    
    @Override
    public Object remove(String name) {
        return threadScope.get().remove(name);
    }
    
    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        // Cleanup callback
    }
    
    @Override
    public Object resolveContextualObject(String key) {
        return null;
    }
    
    @Override
    public String getConversationId() {
        return Thread.currentThread().getName();
    }
}

// Register custom scope
@Configuration
public class ScopeConfig implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        beanFactory.registerScope("thread", new ThreadScope());
    }
}

// Use custom scope
@Bean
@Scope("thread")
public RequestData requestData() {
    return new RequestData();
}
```

### 3.3 Bean Lifecycle - Complete Flow

The bean lifecycle in Spring is a well-defined sequence of steps that a bean goes through from creation to destruction.

```
┌─────────────────────────────────────────────────────────────────┐
│                 Bean Lifecycle (Singleton)                       │
│                                                                  │
│  1. Instantiation                                                │
│     │                                                            │
│     ├─ Container creates bean instance                           │
│     ├─ Calls constructor (no-arg or with args)                   │
│     │                                                            │
│  2. Population of Properties                                     │
│     │                                                            │
│     ├─ Inject dependencies via constructor/setter/field          │
│     ├─ Set bean name and factory                                │
│     ├─ Call BeanPostProcessors before initialization            │
│     │                                                            │
│  3. Initialization                                              │
│     │                                                            │
│     ├─ Call @PostConstruct annotated methods                   │
│     ├─ Call InitializingBean.afterPropertiesSet()                │
│     ├─ Call custom init-method                                   │
│     │                                                            │
│  4. Bean Ready for Use                                          │
│     │                                                            │
│     ├─ Bean is fully initialized and ready                       │
│     ├─ Can be used by application                              │
│     │                                                            │
│  5. Destruction                                                  │
│     │                                                            │
│     ├─ Call @PreDestroy annotated methods                      │
│     ├─ Call DisposableBean.destroy()                            │
│     ├─ Call custom destroy-method                               │
│     ├─ Bean instance eligible for GC                            │
└─────────────────────────────────────────────────────────────────┘
```

**Detailed Lifecycle with Code Examples:**

```java
// Complete bean lifecycle example
@Service
public class OrderService implements BeanNameAware, BeanFactoryAware, 
                                           ApplicationContextAware, InitializingBean, DisposableBean {
    
    private OrderRepository orderRepository;
    private String beanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;
    
    // 1. Constructor called
    public OrderService(OrderRepository orderRepository) {
        System.out.println("1. Constructor called");
        this.orderRepository = orderRepository;
    }
    
    // 2. BeanNameAware callback
    @Override
    public void setBeanName(String name) {
        System.out.println("2. setBeanName called: " + name);
        this.beanName = name;
    }
    
    // 3. BeanFactoryAware callback
    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        System.out.println("3. setBeanFactory called");
        this.beanFactory = beanFactory;
    }
    
    // 4. ApplicationContextAware callback
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        System.out.println("4. setApplicationContext called");
        this.applicationContext = applicationContext;
    }
    
    // 5. @PostConstruct callback
    @PostConstruct
    public void init() {
        System.out.println("5. @PostConstruct called");
        // Initialization logic
    }
    
    // 6. InitializingBean callback
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("6. afterPropertiesSet called");
        // Initialization logic
    }
    
    // 7. Custom init-method (if specified in @Bean)
    public void customInit() {
        System.out.println("7. customInit called");
        // Custom initialization
    }
    
    // Bean is now ready for use
    
    // 8. @PreDestroy callback
    @PreDestroy
    public void preDestroy() {
        System.out.println("8. @PreDestroy called");
        // Cleanup logic
    }
    
    // 9. DisposableBean callback
    @Override
    public void destroy() throws Exception {
        System.out.println("9. destroy called");
        // Cleanup logic
    }
    
    // 10. Custom destroy-method (if specified in @Bean)
    public void customDestroy() {
        System.out.println("10. customDestroy called");
        // Custom cleanup
    }
}
```

**BeanPostProcessors in Lifecycle:**

BeanPostProcessors are called during bean creation and allow custom modification to bean instances.

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        // Modify bean before initialization
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("After init: " + beanName);
        // Modify bean after initialization
        return bean;
    }
}
```

**Lifecycle Order with BeanPostProcessors:**
```
1. Instantiation
   │
2. BeanPostProcessor.postProcessBeforeInitialization() (for each processor)
   │
3. @PostConstruct
   │
4. InitializingBean.afterPropertiesSet()
   │
5. Custom init-method
   │
6. BeanPostProcessor.postProcessAfterInitialization() (for each processor)
   │
7. Bean ready for use
   │
8. @PreDestroy
   │
9. DisposableBean.destroy()
   │
10. Custom destroy-method
```

**Built-in BeanPostProcessors:**

| BeanPostProcessor | Purpose |
|-------------------|---------|
| `AutowiredAnnotationBeanPostProcessor` | Processes @Autowired annotations |
| `CommonAnnotationBeanPostProcessor` | Processes @PostConstruct, @PreDestroy, @Resource |
| `ApplicationContextAwareProcessor` | Processes ApplicationContextAware |
| `ConfigurationClassPostProcessor` | Processes @Configuration classes |
| `RequiredAnnotationBeanPostProcessor` | Processes @Required annotations |
| `InitDestroyAnnotationBeanPostProcessor` | Processes @PostConstruct, @PreDestroy (alternative) |

### 3.4 Lifecycle Callbacks

**Initialization Callbacks:**

Spring provides three ways to define initialization callbacks:

```java
@Service
public class OrderService implements InitializingBean, DisposableBean {
    
    // Method 1: @PostConstruct (recommended)
    @PostConstruct
    public void init() {
        System.out.println("@PostConstruct called");
        // Initialization logic
    }
    
    // Method 2: InitializingBean interface
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("afterPropertiesSet called");
        // Initialization logic
    }
    
    // Method 3: Custom init-method
    public void customInit() {
        System.out.println("customInit called");
        // Initialization logic
    }
}

// Configuration to specify custom init-method
@Configuration
public class AppConfig {
    @Bean(initMethod = "customInit")
    public OrderService orderService() {
        return new OrderService();
    }
}
```

**Execution Order:**
1. Constructor
2. Dependency injection
3. BeanPostProcessors before initialization
4. @PostConstruct
5. afterPropertiesSet()
6. custom init-method
7. BeanPostProcessors after initialization

**Best Practice:** Use @PostConstruct as it:
- Is part of JSR-250 standard (portable)
- Doesn't couple to Spring interface
- Can have multiple methods
- Works with annotation-based configuration

**Destruction Callbacks:**

```java
@Service
public class OrderService implements DisposableBean {
    
    // Method 1: @PreDestroy (recommended)
    @PreDestroy
    public void cleanup() {
        System.out.println("@PreDestroy called");
        // Cleanup logic
    }
    
    // Method 2: DisposableBean interface
    @Override
    public void destroy() throws Exception {
        System.out.println("destroy() called");
        // Cleanup logic
    }
    
    // Method 3: Custom destroy-method
    public void customDestroy() {
        System.out.println("customDestroy called");
        // Cleanup logic
    }
}

// Configuration to specify custom destroy-method
@Configuration
public class AppConfig {
    @Bean(destroyMethod = "customDestroy")
    public OrderService orderService() {
        return new OrderService();
    }
}
```

**Execution Order:**
1. @PreDestroy
2. destroy()
3. custom destroy-method

**Important Notes:**
- Destruction callbacks are only called for singleton beans
- Prototype-scoped beans are not managed by container after creation
- Client must call cleanup for prototype beans
- @PreDestroy is preferred (JSR-250 standard)

**Shutdown Hook:**

For non-web applications, register a shutdown hook to ensure graceful shutdown:

```java
public class Application {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = 
            new AnnotationConfigApplicationContext(AppConfig.class);
        
        // Register shutdown hook for JVM shutdown
        context.registerShutdownHook();
        
        // Use the context
        OrderService orderService = context.getBean(OrderService.class);
        
        // Close the context (triggers destruction callbacks)
        context.close();
    }
}
```

**Lifecycle for Prototype Beans:**

```java
@Service
@Scope("prototype")
public class PrototypeService implements DisposableBean {
    
    @PostConstruct
    public void init() {
        System.out.println("Prototype init called");
    }
    
    @PreDestroy
    public void preDestroy() {
        System.out.println("Prototype @PreDestroy - NEVER CALLED!");
    }
    
    @Override
    public void destroy() throws Exception {
        System.out.println("Prototype destroy - NEVER CALLED!");
    }
}

// Client must manage lifecycle
PrototypeService service = context.getBean(PrototypeService.class);
// Use service
service.cleanup();  // Client must call cleanup manually
```

---

## 4. Dependency Injection Mechanisms

### 4.1 Autowiring Modes

Autowiring allows Spring to automatically resolve dependencies. While autowiring is primarily used in XML configuration, understanding it is important for legacy code.

**Autowiring Modes:**

| Mode | Description | Example | Use Case |
|------|-------------|---------|----------|
| `no` | No autowiring, must specify explicitly | `<bean autowire="no">` | Full control, explicit dependencies |
| `byName` | Autowire by property name matching bean name | `<bean autowire="byName">` | Legacy code, explicit naming convention |
| `byType` | Autowire by property type matching bean type | `<bean autowire="byType">` | When only one bean of type exists |
| `constructor` | Autowire by constructor argument types | `<bean autowire="constructor">` | Constructor-based dependency injection |

**byName Example:**

```xml
<bean id="orderRepository" class="com.example.JdbcOrderRepository"/>
<bean id="orderService" class="com.example.OrderService" autowire="byName">
    <!-- orderRepository property matches bean name -->
</bean>
```

```java
public class OrderService {
    private OrderRepository orderRepository;  // Matches bean name
    
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

**byType Example:**

```xml
<bean class="com.example.JdbcOrderRepository"/>
<bean class="com.example.OrderService" autowire="byType">
    <!-- Autowires by type -->
</bean>
```

**Limitations of Autowiring:**
- **Ambiguity**: Multiple beans of same type cause confusion
- **Explicitness**: Less explicit about dependencies
- **Documentation**: Harder to see what dependencies are required
- **Refactoring**: Renaming classes can break autowiring

**Modern Approach:**

Modern Spring applications prefer explicit dependency injection via @Autowired and constructor injection over autowiring modes.

### 4.2 @Autowired Annotation

**@Autowired** is the primary annotation for dependency injection in Spring. It can be applied to constructors, fields, setter methods, and configuration methods.

**How @Autowired Works Internally:**

```java
// Spring's internal processing of @Autowired
public class AutowiredAnnotationBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) {
        // 1. Find all @Autowired annotations on the bean
        InjectionMetadata metadata = findAutowiringMetadata(bean.getClass());
        
        // 2. For each @Autowired field/method
        for (InjectedElement element : metadata.getInjectedElements()) {
            // 3. Resolve dependency from container
            Object dependency = resolveDependency(element.getDependencyDescriptor());
            
            // 4. Inject the dependency
            element.inject(bean, dependency);
        }
        
        return pvs;
    }
}
```

**Constructor Injection with @Autowired:**

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    // Spring 4.3+: @Autowired optional on single constructor
    public OrderService(OrderRepository orderRepository,
                        PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
    
    // Multiple constructors: specify @Autowired on one
    @Autowired
    public OrderService(OrderRepository orderRepository) {
        this(orderRepository, new DefaultPaymentService());
    }
    
    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
}
```

**Setter Injection with @Autowired:**

```java
@Service
public class OrderService {
    private OrderRepository orderRepository;
    private PaymentService paymentService;
    
    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Field Injection with @Autowired (Not Recommended):**

```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // Avoid this
}
```

**Optional Dependencies:**

```java
@Service
public class OrderService {
    
    @Autowired(required = false)
    private AuditService auditService;  // Null if not found
    
    public void audit(Order order) {
        if (auditService != null) {
            auditService.log(order);
        }
    }
}
```

**@Autowired on Configuration Methods:**

```java
@Configuration
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
    
    @Bean
    @Autowired  // Dependencies injected from other @Bean methods
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

**@Autowired Resolution Order:**

When resolving a dependency marked with @Autowired, Spring follows this order:

1. **By Type**: Find bean of matching type
2. **By Qualifier**: If multiple beans, use @Qualifier to select
3. **By Name**: If still ambiguous, use bean name
4. **Primary**: If still ambiguous, use @Primary bean
5. **Fail**: If still ambiguous, throw NoUniqueBeanDefinitionException

### 4.3 @Qualifier Annotation

**@Qualifier** resolves ambiguity when multiple beans of the same type exist. It specifies which bean should be injected.

**Basic Usage:**

```java
@Repository
@Qualifier("jdbc")
public class JdbcOrderRepository implements OrderRepository {
    // JDBC implementation
}

@Repository
@Qualifier("jpa")
public class JpaOrderRepository implements OrderRepository {
    // JPA implementation
}

@Service
public class OrderService {
    
    @Autowired
    public OrderService(@Qualifier("jpa") OrderRepository orderRepository) {
        this.orderRepository = orderRepository;  // Uses JPA implementation
    }
}
```

**@Qualifier on Collections:**

```java
@Repository
@Qualifier("primary")
public class PrimaryOrderHandler implements OrderHandler {
    public void handle(Order order) { /* ... */ }
}

@Repository
@Qualifier("secondary")
public class SecondaryOrderHandler implements OrderHandler {
    public void handle(Order order) { /* ... */ }
}

@Service
public class OrderProcessor {
    
    @Autowired
    @Qualifier("primary")
    private OrderHandler primaryHandler;
    
    @Autowired
    private List<OrderHandler> allHandlers;  // All OrderHandler beans
    
    @Autowired
    @Qualifier("primary")
    private List<OrderHandler> primaryHandlers;  // Only @Qualifier("primary")
}
```

**Custom @Qualifier Annotations:**

You can create custom qualifier annotations for type safety:

```java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface JdbcRepository {
}

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface JpaRepository {
}

@Repository
@JdbcRepository
public class JdbcOrderRepository implements OrderRepository {
    // JDBC implementation
}

@Repository
@JpaRepository
public class JpaOrderRepository implements OrderRepository {
    // JPA implementation
}

@Service
public class OrderService {
    
    @Autowired
    public OrderService(@JpaRepository OrderRepository orderRepository) {
        this.orderRepository = orderRepository;  // Type-safe qualifier
    }
}
```

**@Qualifier vs @Named:**

- `@Qualifier`: Generic qualifier, can specify any name
- `@Named`: JSR-250 annotation, equivalent to @Qualifier with bean name

```java
// Equivalent
@Autowired
@Qualifier("orderService")
private OrderService service;

@Autowired
@Named("orderService")
private OrderService service;
```

### 4.4 @Primary Annotation

**@Primary** marks a bean as the primary candidate for autowiring when multiple beans of the same type exist. It's a global preference for a specific implementation.

**Basic Usage:**

```java
@Repository
@Primary
public class JpaOrderRepository implements OrderRepository {
    // Primary implementation - used by default
}

@Repository
public class JdbcOrderRepository implements OrderRepository {
    // Fallback implementation - used only with @Qualifier
}

@Service
public class OrderService {
    
    @Autowired
    public OrderService(OrderRepository orderRepository) {
        // Injects JpaOrderRepository (marked as @Primary)
        this.orderRepository = orderRepository;
    }
}
```

**@Primary vs @Qualifier:**

| Aspect | @Primary | @Qualifier |
|--------|----------|------------|
| **Scope** | Global preference | Local selection |
| **Multiple** | Only one primary per type | Multiple qualifiers allowed |
| **Flexibility** | Less flexible | More flexible |
| **Use Case** | Default implementation | Specific use case |

**Example with Both:**

```java
@Repository
@Primary
public class JpaOrderRepository implements OrderRepository {
    // Default for most use cases
}

@Repository
@Qualifier("jdbc")
public class JdbcOrderRepository implements OrderRepository {
    // Used when explicitly requested
}

@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;  // Uses @Primary (JPA)
    
    @Autowired
    @Qualifier("jdbc")
    private OrderRepository jdbcRepository;  // Uses JDBC
}
```

**@Primary on Configuration Methods:**

```java
@Configuration
public class RepositoryConfig {
    
    @Bean
    @Primary
    public OrderRepository jpaOrderRepository(EntityManager em) {
        return new JpaOrderRepository(em);
    }
    
    @Bean
    public OrderRepository jdbcOrderRepository(DataSource ds) {
        return new JdbcOrderRepository(ds);
    }
}
```

### 4.5 @Value Annotation

**@Value** injects values from properties files, environment variables, or SpEL expressions into bean fields or method parameters.

**Property Value Injection:**

```java
@Service
public class OrderService {
    
    @Value("${order.timeout:30}")
    private int timeout;  // Default value if property not found
    
    @Value("${app.name}")
    private String appName;  // Required - throws exception if not found
    
    @Value("${order.processing.enabled:true}")
    private boolean processingEnabled;  // Boolean parsing
}
```

**application.properties:**
```properties
order.timeout=30
app.name=Order Management System
order.processing.enabled=true
```

**System Properties Injection:**

```java
@Service
public class OrderService {
    
    @Value("#{systemProperties['user.home']}")
    private String userHome;
    
    @Value("#{systemProperties['java.version']}")
    private String javaVersion;
    
    @Value("#{systemEnvironment['PATH']}")
    private String path;
}
```

**SpEL Expressions:**

```java
@Service
public class OrderService {
    
    @Value("#{T(java.lang.Math).random() * 100}")
    private double randomValue;
    
    @Value("#{orderRepository.count()}")
    private long orderCount;
    
    @Value("'Hello ' + '${app.name}")
    private String greeting;
    
    @Value("${order.timeout} > 30 ? 'high' : 'low'")
    private String priority;
}
```

**Environment Variables:**

```java
@Service
public class OrderService {
    
    @Value("${DB_URL}")
    private String dbUrl;  // Environment variable DB_URL
    
    @Value("${DB_URL:jdbc:mysql://localhost:3306/db}")
    private String dbUrlWithDefault;
}
```

**@Value on Constructor Parameters:**

```java
@Service
public class OrderService {
    private final int timeout;
    private final String appName;
    
    public OrderService(@Value("${order.timeout:30}") int timeout,
                       @Value("${app.name}") String appName) {
        this.timeout = timeout;
        this.appName = appName;
    }
}
```

**@Value on @Bean Methods:**

```java
@Configuration
public class AppConfig {
    
    @Value("${db.url}")
    private String dbUrl;
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(dbUrl, "user", "password");
    }
}
```

**@Value vs @Autowired:**

| Aspect | @Value | @Autowired |
|--------|--------|------------|
| **Injects** | Primitive values, Strings, SpEL | Beans (objects) |
| **Source** | Properties, environment, SpEL | Spring container |
| **Type** | Field, method parameter | Field, method, constructor |
| **Default** | Can specify default value | Can specify required=false |

**Best Practices:**
- Use @Value for configuration properties
- Use @Autowired for bean dependencies
- Provide default values for optional properties
- Use @ConfigurationProperties for complex configuration

---

## 5. Spring Annotations Comprehensive Guide

### 5.1 Core Annotations

#### @Component
Marks a class as a Spring-managed bean. This is the generic stereotype annotation.

```java
@Component
public class OrderValidator {
    public boolean validate(Order order) {
        return order != null && order.getItems() != null;
    }
}
```

**How @Component Works:**

```java
// Spring's component scanning process
public class ClassPathBeanDefinitionScanner {
    
    public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        // 1. Scan classpath for classes
        Set<BeanDefinition> candidates = new HashSet<>();
        
        // 2. For each class, check for @Component annotation
        for (Class<?> clazz : scan(basePackage)) {
            if (clazz.isAnnotationPresent(Component.class)) {
                // 3. Create BeanDefinition
                BeanDefinition bd = createBeanDefinition(clazz);
                candidates.add(bd);
            }
        }
        
        return candidates;
    }
}
```

**Stereotype Annotations (Specialized @Component):**

| Annotation | Purpose | Processing | Additional Behavior |
|------------|---------|------------|---------------------|
| `@Service` | Service layer business logic | Same as @Component | None |
| `@Repository` | DAO layer data access | Same as @Component | Exception translation |
| `@Controller` | Presentation layer (Spring MVC) | Same as @Component | Request mapping |
| `@RestController` | REST API endpoints | Same as @Controller | Response body serialization |

**@Repository Exception Translation:**

```java
@Repository
public class OrderRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public Order findById(Long id) {
        try {
            return entityManager.find(Order.class, id);
        } catch (PersistenceException ex) {
            // @Repository enables automatic exception translation
            // PersistenceException → Spring DataAccessException
            throw new DataRetrievalFailureException("Failed to find order", ex);
        }
    }
}
```

#### @Configuration
Indicates that a class declares one or more @Bean methods and is a source of bean definitions.

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

**@Configuration Internals:**

```java
// Spring treats @Configuration classes specially
public class ConfigurationClassPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 1. Find all @Configuration classes
        for (String beanName : beanFactory.getBeanNamesForType(ConfigurationClass.class)) {
            ConfigurationClass configClass = beanFactory.getBean(beanName);
            
            // 2. Enhance @Configuration class with CGLIB
            // This ensures @Bean methods are called only once (singleton semantics)
            ConfigurationClass enhancedClass = enhanceConfigurationClass(configClass);
            
            // 3. Process @Bean methods
            processBeanMethods(enhancedClass);
        }
    }
}
```

**Why CGLIB Enhancement?**

```java
@Configuration
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());  // Calls dataSource() method
    }
}

// Without CGLIB: dataSource() called each time jdbcTemplate() is referenced
// With CGLIB: dataSource() called once, result cached, reused
```

#### @Bean
Declares a bean within a @Configuration class. The method name becomes the bean name.

```java
@Bean
@Scope("prototype")
public OrderProcessor orderProcessor() {
    return new OrderProcessor();
}
```

**@Bean Parameters:**

```java
@Configuration
public class AppConfig {
    
    @Bean
    @Scope("singleton")
    @Lazy
    @Primary
    @Description("Main data source")
    @DependsOn("dataSource")
    public OrderService orderService(OrderRepository orderRepository) {
        return new OrderService(orderRepository);
    }
}
```

**@Bean Lifecycle Methods:**

```java
@Bean(initMethod = "init", destroyMethod = "cleanup")
public OrderService orderService() {
    OrderService service = new OrderService();
    return service;
}
```

### 5.2 Component Scanning

#### @ComponentScan

**@ComponentScan** configures component scanning directives to automatically detect and register beans.

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Service.class)
    },
    excludeFilters = {
        @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = TestService.class)
    }
)
public class AppConfig {
}
```

**Component Scanning Internals:**

```java
// Spring's component scanning process
public class ClassPathBeanDefinitionScanner {
    
    public int scan(String... basePackages) {
        int beanCount = 0;
        
        for (String basePackage : basePackages) {
            // 1. Convert package to classpath path
            String packageSearchPath = ClasspathScanningCandidateComponentLoader.convertClassNameToResourcePath(basePackage);
            
            // 2. Scan for resources in classpath
            Resource[] resources = getResourceLoader().getResources(packageSearchPath);
            
            // 3. Load candidate classes
            for (Resource resource : resources) {
                if (resource.isReadable()) {
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    
                    // 4. Check if component
                    if (isCandidateComponent(metadataReader)) {
                        // 5. Create BeanDefinition
                        BeanDefinition bd = createBeanDefinition(metadataReader);
                        beanFactory.registerBeanDefinition(beanName, bd);
                        beanCount++;
                    }
                }
            }
        }
        
        return beanCount;
    }
}
```

**Filter Types:**

| Filter Type | Description | Example |
|-------------|-------------|---------|
| `ANNOTATION` | Filter by annotation presence | `@ComponentScan.Filter(type = ANNOTATION, classes = Service.class)` |
| `ASSIGNABLE_TYPE` | Filter by class type | `@ComponentScan.Filter(type = ASSIGNABLE_TYPE, classes = OrderService.class)` |
| `ASPECTJ` | Filter by AspectJ expression | `@ComponentScan.Filter(type = ASPECTJ, pattern = "com.example..*")` |
| `REGEX` | Filter by regex pattern | `@ComponentScan.Filter(type = REGEX, pattern = ".*Service")` |
| `CUSTOM` | Custom filter implementation | `@ComponentScan.Filter(type = CUSTOM, classes = MyFilter.class)` |

**@Indexed (Spring 5+):**

Spring 5 introduced component indexing to speed up startup:

```java
@Indexed  // Enables component index creation
@Service
public class OrderService {
    // ...
}
```

Without @Indexed, Spring scans all classes at startup. With @Indexed, Spring reads a pre-generated index file, significantly improving startup time for large applications.

### 5.3 Configuration Annotations

#### @Import

**@Import** imports other configuration classes, allowing modular configuration.

```java
@Configuration
@Import({DataSourceConfig.class, SecurityConfig.class, CacheConfig.class})
public class AppConfig {
    // Combines all imported configurations
}
```

**@Import with @Configuration:**

```java
@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}

@Configuration
@Import(DatabaseConfig.class)
public class AppConfig {
    // DatabaseConfig's beans are available here
}
```

**@Import with ImportSelector:**

```java
public class CustomImportSelector implements ImportSelector {
    
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // Conditionally import configurations
        if (isDevelopment()) {
            return new String[]{DevConfig.class.getName()};
        } else {
            return new String[]{ProdConfig.class.getName()};
        }
    }
}

@Configuration
@Import(CustomImportSelector.class)
public class AppConfig {
    // Configuration selected dynamically
}
```

**@Import with ImportBeanDefinitionRegistrar:**

```java
public class CustomBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                        BeanDefinitionRegistry registry) {
        // Programmatically register beans
        BeanDefinition bd = BeanDefinitionBuilder.rootBeanDefinition(CustomService.class).getBeanDefinition();
        registry.registerBeanDefinition("customService", bd);
    }
}

@Configuration
@Import(CustomBeanDefinitionRegistrar.class)
public class AppConfig {
    // Custom beans registered programmatically
}
```

#### @PropertySource

**@PropertySource** loads properties from external files.

```java
@Configuration
@PropertySource("classpath:application.properties")
@PropertySource("classpath:database.properties")
public class AppConfig {
    
    @Autowired
    private Environment environment;
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(
            environment.getProperty("db.url"),
            environment.getProperty("db.username"),
            environment.getProperty("db.password")
        );
    }
}
```

**@PropertySource with encoding:**

```java
@PropertySource(value = "classpath:application.properties", encoding = "UTF-8")
public class AppConfig {
}
```

**@PropertySources (multiple sources):**

```java
@PropertySources({
    @PropertySource("classpath:default.properties"),
    @PropertySource("classpath:override.properties")
})
public class AppConfig {
    // Later sources override earlier ones
}
```

#### @Profile

**@Profile** indicates that a component is only eligible for registration when one or more specified profiles are active.

```java
@Configuration
@Profile("development")
public class DevConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource("jdbc:h2:mem:devdb", "sa", "");
    }
}

@Configuration
@Profile("production")
public class ProdConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource("jdbc:mysql://prod-db:3306/app", "user", "password");
    }
}
```

**Multiple Profiles:**

```java
@Configuration
@Profile({"development", "testing"})
public class DevConfig {
    // Active when development OR testing profile is active
}

@Configuration
@Profile("!production")
public class NonProdConfig {
    // Active when production profile is NOT active
}
```

**Activating Profiles:**

```java
// Method 1: application.properties
spring.profiles.active=development

// Method 2: Environment variable
export SPRING_PROFILES_ACTIVE=development

// Method 3: JVM argument
-Dspring.profiles.active=development

// Method 4: Programmatically
SpringApplication.run(App.class, "--spring.profiles.active=development");
```

### 5.4 Conditional Annotations

Conditional annotations allow beans to be registered based on specific conditions.

#### @Conditional

**@Conditional** is the base annotation for conditional bean registration.

```java
@Configuration
public class AppConfig {
    
    @Bean
    @Conditional(WindowsCondition.class)
    public WindowsService windowsService() {
        return new WindowsService();
    }
}

public class WindowsCondition implements Condition {
    
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return context.getEnvironment()
                   .getProperty("os.name")
                   .contains("Windows");
    }
}
```

#### @ConditionalOnProperty

**@ConditionalOnProperty** checks if a property has a specific value.

```java
@Configuration
@ConditionalOnProperty(
    name = "feature.cache.enabled",
    havingValue = "true",
    matchIfMissing = false
)
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager();
    }
}
```

**Parameters:**
- `name`: Property name
- `havingValue`: Expected value
- `matchIfMissing`: Whether to match if property is absent

#### @ConditionalOnClass

**@ConditionalOnClass** checks if a specified class is on the classpath.

```java
@Configuration
@ConditionalOnClass(name = "com.mysql.cj.jdbc.Driver")
public class MySqlConfig {
    
    @Bean
    public DataSource mysqlDataSource() {
        return new HikariDataSource("jdbc:mysql://localhost:3306/db", "user", "password");
    }
}
```

#### @ConditionalOnMissingBean

**@ConditionalOnMissingBean** checks if a bean is missing.

```java
@Configuration
public class DefaultConfig {
    
    @Bean
    @ConditionalOnMissingBean(DataSource.class)
    public DataSource defaultDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}
```

#### @ConditionalOnBean

**@ConditionalOnBean** checks if a bean exists.

```java
@Configuration
@ConditionalOnBean(DataSource.class)
public class JpaConfig {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        // ...
    }
}
```

#### @ConditionalOnExpression

**@ConditionalOnExpression** uses SpEL expression for complex conditions.

```java
@Configuration
public class ConditionalConfig {
    
    @Bean
    @ConditionalOnExpression("'${feature.enabled}' == 'true' and '${env}' == 'production'")
    public ProductionFeature productionFeature() {
        return new ProductionFeature();
    }
}
```

#### @ConditionalOnJava

**@ConditionalOnJava** checks Java version.

```java
@Configuration
@ConditionalOnJava(JavaVersion.EIGHT)
public class Java8Config {
    // Only active on Java 8+
}

@Configuration
@ConditionalOnJava(range = JavaVersion.NINE)
public class Java9PlusConfig {
    // Only active on Java 9+
}
```

### 5.5 Lifecycle Annotations

#### @PostConstruct

**@PostConstruct** annotates a method to be called after dependency injection is complete.

```java
@Service
public class OrderService {
    
    @PostConstruct
    public void init() {
        System.out.println("OrderService initialized");
        // Initialization logic here
    }
}
```

**Rules for @PostConstruct:**
- Method must have no parameters
- Method must return void
- Method must not throw checked exceptions
- Method can be public, protected, package-private, or private
- Can be applied to multiple methods in the same class

#### @PreDestroy

**@PreDestroy** annotates a method to be called before the bean is destroyed.

```java
@Service
public class OrderService {
    
    @PreDestroy
    public void cleanup() {
        System.out.println("OrderService cleanup");
        // Cleanup logic here
    }
}
```

**Rules for @PreDestroy:**
- Same rules as @PostConstruct
- Only called for singleton beans
- Not called for prototype beans (client must manage lifecycle)

### 5.6 Transaction Annotations

#### @Transactional

**@Transactional** defines transaction boundaries for methods.

```java
@Service
public class OrderService {
    
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        readOnly = false,
        rollbackFor = { Exception.class },
        noRollbackFor = { BusinessException.class }
    )
    public Order createOrder(OrderRequest request) {
        // Transactional method
        Order order = new Order(request);
        orderRepository.save(order);
        paymentService.process(order.getPayment());
        return order;
    }
    
    @Transactional(readOnly = true)
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElse(null);
    }
}
```

**Propagation Types:**

| Propagation | Description | When to Use |
|-------------|-------------|-------------|
| `REQUIRED` | Join existing transaction or create new | Default for most cases |
| `REQUIRES_NEW` | Always create new transaction | Independent transactions |
| `SUPPORTS` | Join if exists, else non-transactional | Optional transactions |
| `NOT_SUPPORTED` | Execute non-transactionally | Read-only operations |
| `MANDATORY` | Must be called within existing transaction | Enforce transaction requirement |
| `NEVER` | Must not be called within transaction | Non-transactional operations |
| `NESTED` | Create nested savepoint | Partial rollback |

**Isolation Levels:**

| Isolation | Description | Performance |
|-----------|-------------|-------------|
| `DEFAULT` | Database default | Database dependent |
| `READ_UNCOMMITTED` | Can read uncommitted changes | Highest performance, lowest consistency |
| `READ_COMMITTED` | Can't read uncommitted changes | Good balance |
| `REPEATABLE_READ` | Can't read uncommitted or non-repeatable reads | Medium consistency |
| `SERIALIZABLE` | Full isolation | Lowest performance, highest consistency |

**Transaction Internals:**

```java
// Spring's transaction interceptor
public class TransactionInterceptor implements MethodInterceptor {
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 1. Get transaction attributes
        TransactionAttribute txAttr = getTransactionAttribute(invocation.getMethod());
        
        // 2. Begin transaction
        TransactionInfo txInfo = createTransactionIfNecessary(txAttr);
        
        try {
            // 3. Proceed with method
            Object retVal = invocation.proceed();
            
            // 4. Commit
            commitTransactionAfterReturning(txInfo);
            return retVal;
            
        } catch (Exception ex) {
            // 5. Rollback
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
    }
}
```

### 5.7 MVC Annotations

#### @RestController

**@RestController** combines @Controller and @ResponseBody. It indicates that the return value of methods should be written directly to the HTTP response body.

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.getOrder(id);
        // Automatically serialized to JSON
    }
    
    @PostMapping
    public Order createOrder(@RequestBody @Valid OrderRequest request) {
        return orderService.createOrder(request);
    }
}
```

**@RestController Internals:**

```java
// @RestController is a meta-annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}
```

**@Controller vs @RestController:**

| Aspect | @Controller | @RestController |
|--------|------------|---------------|
| **Return Value** | View name (String) | Response body (Object) |
| **Serialization** | Manual (ModelAndView) | Automatic (Jackson) |
| **Use Case** | HTML rendering | REST APIs |

#### @RequestMapping

**@RequestMapping** maps HTTP requests to handler methods.

```java
@Controller
@RequestMapping("/orders")
public class OrderController {
    
    @RequestMapping(method = RequestMethod.GET)
    public String listOrders(Model model) {
        model.addAttribute("orders", orderService.getAllOrders());
        return "orders/list";
    }
    
    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public String getOrder(@PathVariable Long id, Model model) {
        model.addAttribute("order", orderService.getOrder(id));
        return "orders/detail";
    }
}
```

**@RequestMapping Parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `value` | URL path | `@RequestMapping("/orders")` |
| `method` | HTTP method | `@RequestMapping(method = RequestMethod.POST)` |
| `params` | Request parameters | `@RequestMapping(params = "action=create")` |
| `headers` | Request headers | `@RequestMapping(headers = "Content-Type=application/json")` |
| `consumes` | Content-Type accepted | `@RequestMapping(consumes = "application/json")` |
| `produces` | Content-Type produced | `@RequestMapping(produces = "application/json")` |

#### @RequestBody

**@RequestBody** binds HTTP request body to method parameter.

```java
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    return orderService.createOrder(request);
}
```

**@RequestBody Internals:**

```java
// Spring uses HttpMessageConverter to deserialize request body
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
    
    @Override
    public Object resolveArgument(MethodParameter parameter, 
                                  ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest) {
        // 1. Read request body
        HttpInputMessage inputMessage = createInputMessage(webRequest);
        
        // 2. Find appropriate message converter
        HttpMessageConverter<?> converter = selectConverter(parameter, inputMessage);
        
        // 3. Convert to Java object
        return converter.read(parameter.getParameterType(), inputMessage);
    }
}
```

#### @PathVariable

**@PathVariable** binds URI template variable to method parameter.

```java
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderService.getOrder(id);
}

@GetMapping("/orders/{id}/items/{itemId}")
public OrderItem getItem(@PathVariable Long id, @PathVariable Long itemId) {
    return orderService.getItem(id, itemId);
}
```

**@PathVariable with Custom Name:**

```java
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable("orderId") Long id) {
    return orderService.getOrder(id);
}
```

#### @RequestParam

**@RequestParam** binds HTTP request parameter to method parameter.

```java
@GetMapping("/orders")
public List<Order> searchOrders(
    @RequestParam(required = false) String status,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size
) {
    return orderService.search(status, page, size);
}
```

**@RequestParam Parameters:**
- `value`: Parameter name
- `required`: Whether parameter is required (default: true)
- `defaultValue`: Default value if not present

---

## 6. Configuration Methods

### 6.1 XML Configuration (Legacy)

XML configuration was the primary method before Java config became popular. It's still supported but considered legacy.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           https://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.example"/>

    <bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
        <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/db"/>
        <property name="username" value="user"/>
        <property name="password" value="password"/>
    </bean>

    <bean id="orderService" class="com.example.OrderService">
        <constructor-arg ref="orderRepository"/>
    </bean>

</beans>
```

**Pros of XML Configuration:**
- Centralized configuration
- No recompilation needed for changes
- Separation of configuration from code

**Cons of XML Configuration:**
- Verbose
- No type safety
- Harder to refactor
- No IDE support

### 6.2 Java Configuration (Modern)

Java configuration is the recommended approach for modern Spring applications.

```java
@Configuration
@ComponentScan(basePackages = "com.example")
@PropertySource("classpath:application.properties")
@Import({SecurityConfig.class, CacheConfig.class})
public class AppConfig {
    
    @Value("${db.url}")
    private String dbUrl;
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(dbUrl);
        return dataSource;
    }
}
```

**Pros of Java Configuration:**
- Type-safe
- IDE support (autocomplete, refactoring)
- Less verbose
- Can use Java logic

**Cons of Java Configuration:**
- Requires recompilation for changes
- Configuration mixed with code

### 6.3 Annotation Configuration

Annotation configuration relies on component scanning and annotations.

```java
@Configuration
@ComponentScan("com.example")
@EnableTransactionManagement
@EnableJpaRepositories("com.example.repository")
public class AppConfig {
    // Components auto-detected via @ComponentScan
}
```

**Pros of Annotation Configuration:**
- Minimal configuration
- Declarative
- Easy to understand

**Cons of Annotation Configuration:**
- Scattered configuration
- Harder to see overall structure

### 6.4 Configuration Comparison

| Aspect | XML | Java | Annotation |
|--------|-----|------|------------|
| **Type Safety** | No | Yes | Yes |
| **Readability** | Verbose | Clear | Very clear |
| **Refactoring** | Difficult | Easy | Easy |
| **IDE Support** | Limited | Excellent | Excellent |
| **Mixing Config** | Possible | Possible | Possible |
| **Centralization** | Yes | Yes | No |
| **Recommended** | No | Yes | Yes |

**Recommendation:** Use Java configuration with annotation-based component scanning for the best balance of type safety and convenience.

---

## 7. Spring Boot Fundamentals

### 7.1 What is Spring Boot?

**Spring Boot** is an opinionated framework for building Spring applications with minimal configuration. It eliminates boilerplate code and provides sensible defaults.

**The Problem Spring Boot Solves:**

Before Spring Boot, setting up a Spring application required:
- Manual dependency management (many JARs with compatible versions)
- Extensive XML configuration
- Manual setup of embedded servers
- Manual configuration of data sources, JPA, security, etc.
- Boilerplate code for common patterns

**Spring Boot's Solution:**

```java
// Before Spring Boot - extensive configuration
@Configuration
@EnableWebMvc
@ComponentScan("com.example")
@EnableTransactionManagement
@EnableJpaRepositories("com.example.repository")
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost:3306/db");
        ds.setUsername("user");
        ds.setPassword("password");
        return ds;
    }
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean emf = new LocalContainerEntityManagerFactoryBean();
        emf.setDataSource(dataSource());
        emf.setPackagesToScan("com.example.entity");
        emf.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        return emf;
    }
    
    // ... many more beans
}

// With Spring Boot - minimal configuration
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**Key Features:**
- **Auto-configuration**: Automatically configures Spring based on classpath
- **Embedded servers**: Tomcat, Jetty, Undertow embedded by default
- **Starters**: Curated dependency sets
- **Production-ready**: Metrics, health checks, externalized configuration
- **No XML**: Convention over configuration
- **Opinionated defaults**: Sensible defaults that can be overridden

### 7.2 @SpringBootApplication

**@SpringBootApplication** is a convenience annotation that adds all of the following:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**@SpringBootApplication is a meta-annotation composed of:**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration  // @Configuration
@EnableAutoConfiguration  // Enables auto-configuration
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})
public @interface SpringBootApplication {
    // ...
}
```

**Each Annotation's Role:**

1. **@SpringBootConfiguration**: Marks the class as a configuration class (same as @Configuration)
2. **@EnableAutoConfiguration**: Enables Spring Boot's auto-configuration mechanism
3. **@ComponentScan**: Enables component scanning of the package containing this class

**SpringApplication.run() Internals:**

```java
public class SpringApplication {
    
    public ConfigurableApplicationContext run(String... args) {
        // 1. Create SpringApplication instance
        SpringApplication application = new SpringApplication(primarySources);
        
        // 2. Prepare environment
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        
        // 3. Create ApplicationContext
        ConfigurableApplicationContext context = createApplicationContext();
        
        // 4. Prepare context
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        
        // 5. Refresh context (triggers bean creation)
        refreshContext(context);
        
        // 6. Call after refresh listeners
        afterRefresh(context, applicationArguments);
        
        return context;
    }
}
```

### 7.3 Auto-Configuration

Auto-configuration automatically configures Spring application based on dependencies on the classpath.

**How It Works:**

```
1. Spring Boot scans classpath for JARs
2. Finds JARs (e.g., spring-boot-starter-web, spring-boot-starter-data-jpa)
3. Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
4. For each auto-configuration class, evaluates @ConditionalOn* annotations
5. Creates beans that match conditions
```

**Auto-Configuration Example:**

```java
// Spring Boot's DataSourceAutoConfiguration
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingBean(DataSource.class)
    @Conditional(PooledDataSourceCondition.class)
    static class PooledDataSourceConfiguration {
        
        @Bean
        @ConditionalOnMissingBean(DataSource.class)
        @Conditional(PooledDataSourceCondition.class)
        DataSource dataSource(DataSourceProperties properties) {
            return createDataSource(properties);
        }
    }
}
```

**Conditional Evaluation:**

```java
// PooledDataSourceCondition checks if connection pool is available
static class PooledDataSourceCondition extends SpringBootCondition {
    
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, 
                                            AnnotatedTypeMetadata metadata) {
        ConditionMessage.Builder message = ConditionMessage.forCondition("DataSource");
        
        if (DataSourceConfigured.hasDataSource(context.getClassLoader())) {
            return ConditionOutcome.match(message.found("connection pool"));
        }
        
        return ConditionOutcome.noMatch(message.didNotFind("connection pool"));
    }
}
```

**Auto-Configuration Report:**

To see which auto-configurations were applied and why:

```properties
# Enable auto-configuration report
debug=true
```

Or programmatically:

```java
@Autowired
private AutoConfigurationReport autoConfigurationReport;

public void printReport() {
    autoConfigurationReport.getConditionAndOutcomeBySource().forEach((source, outcomes) -> {
        System.out.println(source + ": " + outcomes);
    });
}
```

**Disabling Auto-Configuration:**

```java
@SpringBootApplication
@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class})
public class Application {
}
```

Or in properties:

```properties
spring.autoconfigure.exclude=com.example.MyAutoConfiguration
```

### 7.4 Spring Boot Starters

Starters are convenient dependency descriptors that bring in related dependencies.

**Common Starters:**

| Starter | Description | Key Dependencies |
|---------|-------------|-----------------|
| `spring-boot-starter` | Core starter, auto-configuration support | spring-core, spring-context, spring-boot |
| `spring-boot-starter-web` | Web applications (Spring MVC, Tomcat) | spring-webmvc, spring-boot-starter-tomcat |
| `spring-boot-starter-data-jpa` | JPA with Hibernate | spring-data-jpa, hibernate, spring-jdbc |
| `spring-boot-starter-security` | Spring Security | spring-security-config, spring-security-web |
| `spring-boot-starter-data-jdbc` | JDBC with HikariCP | spring-jdbc, HikariCP |
| `spring-boot-starter-validation` | Validation with Hibernate Validator | hibernate-validator |
| `spring-boot-starter-cache` | Caching support | spring-context-support, caffeine |
| `spring-boot-starter-actuator` | Production-ready features | spring-boot-actuator, micrometer |
| `spring-boot-starter-test` | Testing (JUnit, Mockito, etc.) | spring-test, junit, mockito |
| `spring-boot-starter-webflux` | Reactive web applications | spring-webflux, spring-boot-starter-reactor-netty |

**Starter Internals:**

```xml
<!-- spring-boot-starter-web pom.xml (simplified) -->
<project>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
    </dependencies>
</project>
```

**Creating Custom Starters:**

```xml
<!-- custom-starter pom.xml -->
<project>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>custom-library</artifactId>
        </dependency>
    </dependencies>
</project>
```

```java
// Auto-configuration for custom library
@Configuration
@ConditionalOnClass(CustomService.class)
@EnableConfigurationProperties(CustomProperties.class)
public class CustomAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public CustomService customService(CustomProperties properties) {
        return new CustomService(properties);
    }
}
```

```properties
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.CustomAutoConfiguration
```

### 7.5 External Configuration

Spring Boot provides extensive support for external configuration, allowing you to configure your application from various sources.

**Configuration Sources (in order of precedence):**

1. Command line arguments
2. SPRING_APPLICATION_JSON (environment variable)
3. JNDI attributes from java:comp/env
4. Java System properties
5. OS environment variables
6. RandomValuePropertySource
7. application-{profile}.properties outside packaged jar
8. application-{profile}.properties inside packaged jar
9. application.properties outside packaged jar
10. application.properties inside packaged jar
11. @PropertySource annotations
12. Default properties

**application.properties:**

```properties
# Server configuration
server.port=8080
server.servlet.context-path=/api
server.compression.enabled=true

# Database configuration
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5

# JPA configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.properties.hibernate.format_sql=true

# Logging configuration
logging.level.com.example=DEBUG
logging.level.org.springframework.web=INFO
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n
```

**application.yml:**

```yaml
server:
  port: 8080
  servlet:
    context-path: /api
  compression:
    enabled: true

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
        format_sql: true

logging:
  level:
    com.example: DEBUG
    org.springframework.web: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
```

**@ConfigurationProperties:**

For type-safe configuration binding:

```java
@ConfigurationProperties(prefix = "order")
public class OrderProperties {
    private int timeout = 30;
    private boolean enabled = true;
    private int maxItems = 100;
    private Retry retry = new Retry();
    
    // getters and setters
    
    public static class Retry {
        private int maxAttempts = 3;
        private long delay = 1000;
        
        // getters and setters
    }
}

@Configuration
@EnableConfigurationProperties(OrderProperties.class)
public class AppConfig {
}
```

**application.properties:**

```properties
order.timeout=30
order.enabled=true
order.max-items=100
order.retry.max-attempts=3
order.retry.delay=1000
```

**Environment Variables:**

```bash
# Environment variables override properties
export SERVER_PORT=9090
export SPRING_DATASOURCE_URL=jdbc:mysql://prod-db:3306/mydb
export SPRING_PROFILES_ACTIVE=production
```

**Command Line Arguments:**

```bash
java -jar app.jar --server.port=9090 --spring.profiles.active=production
```

---

## 8. Spring MVC Internals

### 8.1 Spring MVC Architecture

Spring MVC follows the Model-View-Controller pattern with a front controller design.

**Detailed Request Flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring MVC Request Flow                        │
│                                                                  │
│  1. HTTP Request                                                │
│     │                                                            │
│     ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  DispatcherServlet (Front Controller)                     │  │
│  │  - Receives all HTTP requests                              │  │
│  │  - Delegates to other components                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│     │                                                            │
│     ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  HandlerMapping                                           │  │
│  │  - Maps request URL to controller method                │  │
│  │  - Uses @RequestMapping annotations                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│     │                                                            │
│     ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  HandlerAdapter                                            │  │
│  │  - Invokes controller method                              │  │
│  │  - Handles method arguments                              │  │
│  │  - Handles return values                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│     │                                                            │
│     ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Controller                                               │  │
│  │  - Business logic                                        │  │
│  │  - Returns data or view name                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│     │                                                            │
│     ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ViewResolver                                             │  │
│  │  - Resolves view name to View implementation            │  │
│  │  - For REST APIs, returns data directly                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│     │                                                            │
│     ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  View                                                     │  │
│  │  - Renders response (HTML, JSON, etc.)                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│     │                                                            │
│     ▼                                                            │
│  7. HTTP Response                                               │
└─────────────────────────────────────────────────────────────────┘
```

**DispatcherServlet Internals:**

```java
public class DispatcherServlet extends FrameworkServlet {
    
    @Override
    protected void doService(HttpServletRequest request, HttpServletResponse response) {
        // 1. Process request
        processRequest(request, response);
    }
    
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {
        // 2. Get handler
        HandlerExecutionChain mappedHandler = getHandler(processedRequest);
        
        // 3. Get handler adapter
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
        
        // 4. Apply pre-handle interceptors
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
        }
        
        // 5. Actually invoke the handler
        ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
        
        // 6. Apply post-handle interceptors
        mappedHandler.applyPostHandle(processedRequest, response, mv);
        
        // 7. Process result (render view or write response)
        processDispatchResult(processedRequest, response, mappedHandler, mv);
    }
}
```

### 8.2 Controller Method Arguments

Spring MVC supports various method arguments for controller methods. The framework uses argument resolvers to populate these arguments.

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    // Path variable
    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.getOrder(id);
    }
    
    // Request parameter
    @GetMapping
    public List<Order> search(
        @RequestParam(required = false) String status,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size
    ) {
        return orderService.search(status, page, size);
    }
    
    // Request body
    @PostMapping
    public Order createOrder(@RequestBody @Valid OrderRequest request) {
        return orderService.createOrder(request);
    }
    
    // Request header
    @GetMapping
    public List<Order> getOrders(@RequestHeader("Authorization") String token) {
        return orderService.getOrders(token);
    }
    
    // Cookie value
    @GetMapping
    public String getSession(@CookieValue("JSESSIONID") String sessionId) {
        return sessionId;
    }
    
    // HttpServletRequest
    @GetMapping
    public void handleRequest(HttpServletRequest request) {
        // Access raw request
    }
    
    // Principal (authenticated user)
    @GetMapping("/me")
    public User getCurrentUser(Principal principal) {
        return userService.getUser(principal.getName());
    }
    
    // Locale
    @GetMapping
    public String getLocale(Locale locale) {
        return locale.toString();
    }
    
    // Model (for view rendering)
    @GetMapping("/orders")
    public String listOrders(Model model) {
        model.addAttribute("orders", orderService.getAllOrders());
        return "orders/list";
    }
    
    // RedirectAttributes
    @PostMapping("/orders")
    public String createOrder(OrderRequest request, RedirectAttributes redirectAttributes) {
        Order order = orderService.createOrder(request);
        redirectAttributes.addFlashAttribute("message", "Order created");
        return "redirect:/orders/" + order.getId();
    }
}
```

**HandlerMethodArgumentResolver Internals:**

```java
// Spring uses HandlerMethodArgumentResolver to resolve method arguments
public interface HandlerMethodArgumentResolver {
    
    boolean supportsParameter(MethodParameter parameter);
    
    Object resolveArgument(MethodParameter parameter,
                          ModelAndViewContainer mavContainer,
                          NativeWebRequest webRequest,
                          WebDataBinderFactory binderFactory) throws Exception;
}

// Example: RequestParamMethodArgumentResolver
public class RequestParamMethodArgumentResolver implements HandlerMethodArgumentResolver {
    
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestParam.class);
    }
    
    @Override
    public Object resolveArgument(MethodParameter parameter, ...)
            throws Exception {
        // 1. Get parameter name
        String name = getParameterName(parameter);
        
        // 2. Get value from request
        String value = webRequest.getParameter(name);
        
        // 3. Convert to target type
        return conversionService.convert(value, parameter.getParameterType());
    }
}
```

### 8.3 Exception Handling

Spring MVC provides several ways to handle exceptions in controllers.

**@ExceptionHandler (Controller-level):**

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.getOrder(id);
    }
    
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) {
        ErrorResponse error = new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

**@RestControllerAdvice (Global):**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) {
        ErrorResponse error = new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", ex.getMessage());
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleMethodArgumentNotValid(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.toList());
        ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", errors);
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        ErrorResponse error = new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

**Exception Handling Internals:**

```java
// DispatcherServlet exception handling
public class DispatcherServlet extends FrameworkServlet {
    
    @Override
    protected void processDispatchResult(HttpServletRequest request, 
                                      HttpServletResponse response,
                                      HandlerExecutionChain mappedHandler,
                                      ModelAndView mv) {
        try {
            // Process result
            render(mv, request, response);
        } catch (Exception ex) {
            // Handle exception
            processHandlerException(request, response, mappedHandler.getHandler(), ex);
        }
    }
    
    protected ModelAndView processHandlerException(HttpServletRequest request,
                                                HttpServletResponse response,
                                                Object handler,
                                                Exception ex) {
        // 1. Try @ExceptionHandler in handler
        ModelAndView mav = triggerHandlerException(handler, ex);
        if (mav != null) return mav;
        
        // 2. Try @ExceptionHandler in @ControllerAdvice
        mav = triggerHandlerExceptionResolver(handler, ex);
        if (mav != null) return mav;
        
        // 3. Use default exception handling
        return doResolveException(request, response, handler, ex);
    }
}
```

---

## 9. Spring Data JPA

### 9.1 Repository Interfaces

Spring Data JPA provides repository abstractions to significantly reduce the amount of boilerplate code required to implement data access layers.

**Repository Hierarchy:**

```
Repository (Marker Interface)
    │
    ├─ CrudRepository (CRUD operations)
    │   │
    │   ├─ PagingAndSortingRepository (Pagination + Sorting)
    │   │   │
    │   │   └─ JpaRepository (JPA-specific features)
    │   │
    │   └─ ListCrudRepository (List-based CRUD)
    │
    └─ ListPagingAndSortingRepository (List-based pagination)
```

**CrudRepository Methods:**

```java
@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);              // Save single entity
    <S extends T> Iterable<S> saveAll(Iterable<S> entities);  // Save multiple
    Optional<T> findById(ID id);               // Find by ID
    boolean existsById(ID id);                 // Check existence
    Iterable<T> findAll();                      // Find all
    Iterable<T> findAllById(Iterable<ID> ids);  // Find by IDs
    long count();                               // Count all
    void deleteById(ID id);                     // Delete by ID
    void delete(T entity);                      // Delete entity
    void deleteAll(Iterable<? extends T> entities);  // Delete multiple
    void deleteAll();                           // Delete all
}
```

**PagingAndSortingRepository Methods:**

```java
@NoRepositoryBean
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
    Iterable<T> findAll(Sort sort);           // Find all with sorting
    Page<T> findAll(Pageable pageable);        // Find all with pagination
}
```

**JpaRepository Additional Methods:**

```java
@NoRepositoryBean
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID> {
    List<T> findAll();                         // Find all as List
    List<T> findAll(Sort sort);               // Find all with sorting as List
    List<T> findAllById(Iterable<ID> ids);    // Find by IDs as List
    List<T> saveAll(Iterable<S> entities);     // Save all as List
    void flush();                              // Flush persistence context
    T saveAndFlush(T entity);                  // Save and flush
    void deleteInBatch(Iterable<T> entities);  // Delete in batch
    T getOne(ID id);                           // Get entity reference (lazy)
    T getById(ID id);                         // Get entity (eager)
}
```

### 9.2 Repository Methods

Spring Data JPA provides three ways to define queries: method name derivation, @Query annotation, and custom implementation.

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // ========== CRUD methods (inherited) ==========
    // save(S entity), findById(ID id), findAll(), count(), deleteById(ID id)
    
    // ========== Query Derivation ==========
    List<Order> findByCustomer_Id(Long customerId);
    List<Order> findByStatus(String status);
    List<Order> findByStatusAndCreatedDateBetween(String status, Date start, Date end);
    List<Order> findByCustomer_Email(String email);
    List<Order> findByStatusAndCustomer_Id(String status, Long customerId);
    
    // ========== Pagination ==========
    Page<Order> findByStatus(String status, Pageable pageable);
    Slice<Order> findByStatus(String status, Pageable pageable);
    List<Order> findByStatus(String status, Sort sort);
    
    // ========== Count Queries ==========
    long countByStatus(String status);
    boolean existsByStatus(String status);
    
    // ========== Limiting Results ==========
    List<Order> findTop10ByStatusOrderByCreatedDateDesc(String status);
    Order findFirstByCustomer_IdOrderByCreatedDateDesc(Long customerId);
    
    // ========== @Query Annotation ==========
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    List<Order> findByStatusWithQuery(@Param("status") String status);
    
    // ========== Native Query ==========
    @Query(value = "SELECT * FROM orders WHERE status = ?1", nativeQuery = true)
    List<Order> findByStatusNative(String status);
    
    // ========== Modifying Query ==========
    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") String status);
    
    // ========== Entity Graph ==========
    @EntityGraph(attributePaths = {"customer", "items"})
    Order findWithCustomerAndItemsById(Long id);
    
    // ========== Projections ==========
    <T> List<T> findByStatus(String status, Class<T> type);
}
```

### 9.3 Query Derivation Keywords

**Complete Keyword Reference:**

| Keyword | Sample | JPQL Snippet |
|---------|--------|--------------|
| `findBy` | `findByStatus` | `WHERE x.status = ?1` |
| `readBy` | `readByStatus` | `WHERE x.status = ?1` |
| `queryBy` | `queryByStatus` | `WHERE x.status = ?1` |
| `countBy` | `countByStatus` | `SELECT COUNT(*) WHERE x.status = ?1` |
| `existsBy` | `existsByStatus` | `SELECT COUNT(*) > 0 WHERE x.status = ?1` |
| `And` | `findByStatusAndDate` | `WHERE x.status = ?1 AND x.date = ?2` |
| `Or` | `findByStatusOrStatus` | `WHERE x.status = ?1 OR x.status = ?2` |
| `Between` | `findByDateBetween` | `WHERE x.date BETWEEN ?1 AND ?2` |
| `LessThan` | `findByAmountLessThan` | `WHERE x.amount < ?1` |
| `LessThanEqual` | `findByAmountLessThanEqual` | `WHERE x.amount <= ?1` |
| `GreaterThan` | `findByAmountGreaterThan` | `WHERE x.amount > ?1` |
| `GreaterThanEqual` | `findByAmountGreaterThanEqual` | `WHERE x.amount >= ?1` |
| `After` | `findByDateAfter` | `WHERE x.date > ?1` |
| `Before` | `findByDateBefore` | `WHERE x.date < ?1` |
| `IsNull` | `findByDateIsNull` | `WHERE x.date IS NULL` |
| `IsNotNull` | `findByDateIsNotNull` | `WHERE x.date IS NOT NULL` |
| `Like` | `findByNameLike` | `WHERE x.name LIKE ?1` |
| `NotLike` | `findByNameNotLike` | `WHERE x.name NOT LIKE ?1` |
| `StartingWith` | `findByNameStartingWith` | `WHERE x.name LIKE ?1%` |
| `EndingWith` | `findByNameEndingWith` | `WHERE x.name LIKE %?1` |
| `Containing` | `findByNameContaining` | `WHERE x.name LIKE %?1%` |
| `OrderBy` | `findByStatusOrderByDateDesc` | `WHERE x.status = ?1 ORDER BY x.date DESC` |
| `Not` | `findByStatusNot` | `WHERE x.status <> ?1` |
| `In` | `findByIdIn` | `WHERE x.id IN ?1` |
| `NotIn` | `findByIdNotIn` | `WHERE x.id NOT IN ?1` |
| `True` | `findByActiveTrue` | `WHERE x.active = true` |
| `False` | `findByActiveFalse` | `WHERE x.active = false` |
| `IgnoreCase` | `findByNameIgnoreCase` | `WHERE LOWER(x.name) = LOWER(?1)` |

**Complex Query Derivation Example:**

```java
// Method name
List<Order> findByCustomer_EmailAndStatusAndCreatedDateBetweenOrderByCreatedDateDesc(
    String email, 
    String status, 
    Date start, 
    Date end
);

// Generated JPQL
SELECT o FROM Order o 
WHERE o.customer.email = ?1 
  AND o.status = ?2 
  AND o.createdDate BETWEEN ?3 AND ?4 
ORDER BY o.createdDate DESC
```

**Repository Internals:**

```java
// Spring Data uses JDK dynamic proxies to create repository implementations
public class JpaRepositoryFactory extends RepositoryFactorySupport {
    
    @Override
    protected Object getTargetRepository(RepositoryInformation information) {
        // 1. Create repository implementation
        SimpleJpaRepository<?, ?> repository = getTargetRepositoryViaReflection(information);
        
        // 2. Set entity manager
        repository.setEntityManager(entityManager);
        
        return repository;
    }
    
    @Override
    protected Class<?> getRepositoryBaseClass(RepositoryMetadata metadata) {
        return SimpleJpaRepository.class;
    }
}
```

---

## 10. Spring Security Fundamentals

### 10.1 Spring Security Architecture

Spring Security is a powerful and highly customizable authentication and access-control framework.

**Detailed Filter Chain:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Security Filter Chain                   │
│                                                                  │
│  HTTP Request                                                   │
│     │                                                            │
│     ▼                                                            │
│  SecurityContextPersistenceFilter                               │
│  - Loads SecurityContext from session (if exists)               │
│  - Saves SecurityContext to session after request              │
│     │                                                            │
│     ▼                                                            │
│  LogoutFilter                                                   │
│  - Processes logout requests                                    │
│  - Clears SecurityContext and session                           │
│     │                                                            │
│     ▼                                                            │
│  UsernamePasswordAuthenticationFilter                            │
│  - Processes form-based login                                   │
│  - Creates Authentication token                                 │
│     │                                                            │
│     ▼                                                            │
│  BasicAuthenticationFilter                                      │
│  - Processes HTTP Basic authentication                           │
│  - Extracts credentials from Authorization header              │
│     │                                                            │
│     ▼                                                            │
│  SecurityContextHolderAwareRequestFilter                         │
│  - Wraps request to check SecurityContext                        │
│     │                                                            │
│     ▼                                                            │
│  AnonymousAuthenticationFilter                                   │
│  - Creates anonymous Authentication if none exists                │
│     │                                                            │
│     ▼                                                            │
│  ExceptionTranslationFilter                                      │
│  - Converts Spring Security exceptions to HTTP responses        │
│     │                                                            │
│     ▼                                                            │
│  FilterSecurityInterceptor                                       │
│  - Performs authorization checks                                │
│  - Uses AccessDecisionManager                                    │
│     │                                                            │
│     ├─ AccessDecisionManager                                    │
│     │   ├─ AffirmativeBased (any voter grants)                  │
│     │   ├─ ConsensusBased (majority grants)                     │
│     │   └─ UnanimousBased (all voters grant)                    │
│     │                                                            │
│     └─ FilterInvocationSecurityMetadataSource                  │
│         │                                                        │
│         └─ ConfigAttribute (@Secured, @PreAuthorize)             │
│                                                                  │
│  HTTP Response                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**SecurityContextHolder Internals:**

```java
// SecurityContextHolder stores security context
public class SecurityContextHolder {
    
    // Strategy for storing context (ThreadLocal by default)
    private static SecurityContextHolderStrategy strategy;
    
    static {
        // Default: ThreadLocal strategy
        strategy = new ThreadLocalSecurityContextHolderStrategy();
    }
    
    public static SecurityContext getContext() {
        return strategy.getContext();
    }
    
    public static void setContext(SecurityContext context) {
        strategy.setContext(context);
    }
}

// SecurityContext holds Authentication
public class SecurityContext implements Serializable {
    
    private Authentication authentication;
    
    public Authentication getAuthentication() {
        return authentication;
    }
    
    public void setAuthentication(Authentication authentication) {
        this.authentication = authentication;
    }
}

// Authentication represents authenticated user
public interface Authentication extends Principal {
    
    Collection<? extends GrantedAuthority> getAuthorities();  // Roles/permissions
    Object getCredentials();                                       // Password (usually null after auth)
    Object getDetails();                                           // Additional details (IP, session ID)
    Object getPrincipal();                                         // User identity
    boolean isAuthenticated();                                      // Authenticated status
}
```

### 10.2 Security Configuration

**Modern Spring Security 6 Configuration:**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Enables @PreAuthorize, @PostAuthorize, etc.
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            // Form login
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            // Logout
            .logout(logout -> logout
                .logoutSuccessUrl("/login")
                .permitAll()
            )
            // HTTP Basic
            .httpBasic(Customizer.withDefaults())
            // CSRF (disabled for REST APIs)
            .csrf(csrf -> csrf.disable());
        
        return http.build();
    }
    
    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.builder()
            .username("user")
            .password(passwordEncoder().encode("password"))
            .roles("USER")
            .build();
        
        UserDetails admin = User.builder()
            .username("admin")
            .password(passwordEncoder().encode("admin"))
            .roles("ADMIN")
            .build();
        
        return new InMemoryUserDetailsManager(user, admin);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**Custom UserDetailsService:**

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));
        
        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())
            .roles(user.getRoles().toArray(new String[0]))
            .build();
    }
}
```

### 10.3 Method Security

Method security enables fine-grained security checks at the method level using annotations.

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}

@Service
public class OrderService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) {
        orderRepository.deleteById(id);
    }
    
    @PreAuthorize("#order.customer.id == authentication.principal.id")
    public Order updateOrder(Order order) {
        return orderRepository.save(order);
    }
    
    @PostFilter("filterObject.customer.id == authentication.principal.id")
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }
}
```

**Additional Method Security Annotations:**

```java
@Service
public class OrderService {
    
    @PostAuthorize("returnObject.customer.id == authentication.principal.id")
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElse(null);
    }
    
    @PreFilter("filterObject.customer.id == authentication.principal.id")
    public void bulkUpdate(List<Order> orders) {
        orderRepository.saveAll(orders);
    }
    
    @Secured("ROLE_ADMIN")  // Legacy annotation
    public void deleteOrder(Long id) {
        orderRepository.deleteById(id);
    }
}
```

**Method Security Internals:**

```java
// Spring Security uses AOP to enforce method security
public class MethodSecurityInterceptor extends AbstractSecurityInterceptor implements MethodInterceptor {
    
    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        // 1. Get security attributes from annotation
        InterceptorStatusToken token = beforeInvocation(mi);
        
        try {
            // 2. Invoke method
            Object result = mi.proceed();
            
            // 3. Post-invocation checks (e.g., @PostAuthorize)
            return afterInvocation(token, result);
            
        } finally {
            // 4. Clean up
            finallyInvocation(token);
        }
    }
}
```

---

## 11. Advanced Spring Topics

### 11.1 Spring Expression Language (SpEL)

SpEL is a powerful expression language for querying and manipulating object graphs at runtime.

**SpEL Features:**
- Literal expressions
- Method and property access
- Arithmetic, logical, and relational operators
- Regular expressions
- Collection selection and projection
- Bean references
- Type constructors
- Method invocation
- Assignment
- Collection filtering
- Safe navigation

```java
@Service
public class OrderService {
    
    // Literal expressions
    @Value("#{100}")
    private int literal;
    
    // System properties
    @Value("#{systemProperties['user.home']}")
    private String userHome;
    
    // Math expressions
    @Value("#{T(java.lang.Math).random() * 100.0}")
    private double randomValue;
    
    // Bean references
    @Value("#{orderService.defaultTimeout * 2}")
    private int extendedTimeout;
    
    // Boolean expressions
    @Value("#{orderService.defaultTimeout > 20 ? 'high' : 'low'}")
    private String priority;
    
    // Safe navigation
    @Value("#{customer?.address?.city}")
    private String city;
    
    public int getDefaultTimeout() {
        return 30;
    }
}
```

**SpEL in Security:**

```java
@PreAuthorize("#order.customer.username == authentication.name")
public Order updateOrder(Order order) {
    return orderRepository.save(order);
}
```

### 11.2 Profiles

Profiles allow conditional bean registration based on environment (development, test, production, etc.).

```java
@Configuration
@Profile("development")
public class DevConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource("jdbc:h2:mem:devdb", "sa", "");
    }
}

@Configuration
@Profile("production")
public class ProdConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource("jdbc:mysql://prod-db:3306/app", "user", "password");
    }
}

@Configuration
@Profile({"test", "integration"})
public class TestConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource("jdbc:h2:mem:testdb", "sa", "");
    }
}
```

**Activating Profiles:**

```properties
# application.properties
spring.profiles.active=development

# Or programmatically
SpringApplication.run(App.class, "--spring.profiles.active=production");
```

### 11.3 Application Events

Spring provides an event publishing and listening mechanism for loose coupling between components.

**Custom Event:**

```java
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    
    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
    
    public Order getOrder() {
        return order;
    }
}
```

**Event Publisher:**

```java
@Service
public class OrderService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));
        return order;
    }
}
```

**Event Listener:**

```java
@Component
public class OrderEventListener {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        Order order = event.getOrder();
        notificationService.sendNotification(order);
    }
    
    @EventListener(condition = "#event.order.total > 1000")
    public void handleLargeOrder(OrderCreatedEvent event) {
        // Only handle orders over 1000
    }
}
```

**Async Event Processing:**

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}

@Component
public class OrderEventListener {
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Process asynchronously
    }
}
```

---

## 12. Spring Performance and Best Practices

### 12.1 Performance Tips

1. **Use constructor injection** - Ensures immutability and easier testing
2. **Avoid field injection** - Makes testing difficult and hides dependencies
3. **Use @Lazy for heavy beans** - Defer initialization until needed
4. **Scope beans appropriately** - Use prototype for stateful beans
5. **Use caching** - Reduce database calls with @Cacheable
6. **Batch database operations** - Use JDBC batch for bulk inserts
7. **Use connection pooling** - HikariCP is recommended
8. **Avoid N+1 queries** - Use JOIN FETCH or EntityGraph
9. **Use async processing** - @Async for long-running tasks
10. **Monitor with Actuator** - Track metrics and health
11. **Use component indexing** - @Indexed for faster startup in large applications
12. **Optimize bean scope** - Use request/session scope for web-specific beans
13. **Use lazy loading** - Configure JPA lazy loading for associations
14. **Tune connection pool** - Adjust pool size based on load
15. **Use query caching** - Enable second-level cache for frequently accessed data

### 12.2 Best Practices

**Configuration:**
- Use Java configuration over XML
- Use @ComponentScan for automatic discovery
- Externalize configuration with @PropertySource
- Use profiles for environment-specific config
- Keep configuration classes focused
- Use @ConfigurationProperties for type-safe configuration
- Use @PropertySource with encoding for non-ASCII characters

**Dependency Injection:**
- Prefer constructor injection
- Use final fields for required dependencies
- Use @Qualifier for ambiguity resolution
- Use @Lazy for circular dependencies
- Avoid @Autowired on constructors (Spring 4.3+)
- Use @Primary for default implementations
- Use @Value for configuration properties

**Transactions:**
- Keep transactions short
- Use @Transactional on service layer
- Specify rollbackFor appropriately
- Use readOnly for read-only operations
- Understand propagation behavior
- Use REQUIRES_NEW for independent transactions
- Use NESTED for partial rollback

**Testing:**
- Use @SpringBootTest for integration tests
- Use @MockBean for mocking beans
- Use @DataJpaTest for repository tests
- Use @WebMvcTest for controller tests
- Use @ConfigurationProperties for configuration testing
- Use @TestConfiguration for test-specific beans
- Use @DirtiesContext for context reloading

**Security:**
- Use method-level security with @PreAuthorize
- Use hasRole() for role-based checks
- Use #expression for parameter-based checks
- Implement custom UserDetailsService
- Use PasswordEncoder for password hashing
- Configure CSRF appropriately
- Use HTTPS in production

**Data Access:**
- Use Spring Data JPA repositories
- Use JOIN FETCH to avoid N+1
- Use @EntityGraph for fetch plans
- Use pagination for large result sets
- Use batch inserts for bulk operations
- Use second-level cache for read-heavy data
- Use @Transactional for transaction boundaries

**General:**
- Follow SOLID principles
- Keep methods small and focused
- Use meaningful names
- Document complex logic
- Write unit tests
- Use logging appropriately
- Handle exceptions gracefully
- Use appropriate HTTP status codes
- Validate input data
- Use DTOs for external APIs

---

## 13. Glossary of Key Terms

| Term | Definition |
|------|------------|
| **IoC** | Inversion of Control - design principle where framework controls flow |
| **DI** | Dependency Injection - pattern to implement IoC |
| **Bean** | Object managed by Spring container |
| **ApplicationContext** | Central interface for Spring configuration |
| **BeanFactory** | Root interface for accessing Spring bean container |
| **AOP** | Aspect-Oriented Programming - separation of cross-cutting concerns |
| **Proxy** | Object used to implement AOP advice |
| **Join Point** | Well-defined point during program execution |
| **Pointcut** | Expression that matches join points |
| **Advice** | Action taken at a join point |
| **Aspect** | Module encapsulating pointcuts and advice |
| **StereoType Annotations** | Specialized @Component (@Service, @Repository, @Controller) |
| **Auto-configuration** | Spring Boot feature for automatic configuration |
| **Starter** | Convenient dependency descriptor |
| **Actuator** | Production-ready features for monitoring |
| **Profile** | Logical group of bean definitions |
| **SpEL** | Spring Expression Language |
| **Transaction** | Atomic unit of work |
| **Propagation** | Behavior of transaction with existing transaction |
| **Isolation** | Degree of isolation between concurrent transactions |
| **Repository** | Data access abstraction in Spring Data |
| **Entity** | JPA persistent object |
| **EntityManager** | JPA interface for interacting with persistence context |
| **Persistence Context** | Set of managed entity instances |
| **Lazy Loading** | Deferring data fetching until needed |
| **Eager Loading** | Loading data immediately |
| **JOIN FETCH** | JPA feature to eagerly fetch associations |
| **EntityGraph** - Declarative fetch plan |
| **Filter** | Servlet-level request interceptor |
| **Interceptor** - Spring MVC request interceptor |
| **HandlerMapping** - Maps requests to handlers |
| **HandlerAdapter** - Invokes handler methods |
| **ViewResolver** - Resolves view names to views |
| **SecurityContext** - Holds security information |
| **Authentication** - Verification of identity |
| **Authorization** - Granting access to resources |
