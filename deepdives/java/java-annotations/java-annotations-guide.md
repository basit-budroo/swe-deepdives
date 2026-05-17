# Comprehensive Guide to Java Annotations

## Table of Contents
1. [Introduction to Annotations](#introduction-to-annotations)
2. [Annotation Syntax and Structure](#annotation-syntax-and-structure)
3. [Meta-Annotations](#meta-annotations)
4. [Retention Policies](#retention-policies)
5. [Annotation Targets](#annotation-targets)
6. [Built-in Java Annotations](#built-in-java-annotations)
7. [Creating Custom Annotations](#creating-custom-annotations)
8. [Annotation Processing Internals](#annotation-processing-internals)
9. [Runtime Reflection with Annotations](#runtime-reflection-with-annotations)
10. [Bytecode Representation](#bytecode-representation)
11. [Annotation Inheritance](#annotation-inheritance)
12. [Practical Examples](#practical-examples)
13. [Performance Considerations](#performance-considerations)

---

## Introduction to Annotations

### What are Annotations?

Annotations in Java are a form of metadata that provide data about a program but are not part of the program itself. They have no direct effect on the operation of the code they annotate.

**Key Characteristics:**
- Introduced in Java 5 (JDK 1.5)
- Start with the `@` symbol
- Can be applied to classes, methods, fields, parameters, packages, and more
- Do not affect program execution directly
- Used by compilers, runtime tools, and frameworks

### Why Annotations Exist

Before annotations, Java used:
- **XML configuration files** (verbose, separate from code)
- **Naming conventions** (error-prone, no validation)
- **Marker interfaces** (limited functionality)

Annotations solve these problems by:
- Keeping metadata close to the code
- Providing type-safe metadata
- Enabling compile-time checking
- Supporting automated processing

### Annotation vs. Comments

```java
// This is a comment - ignored by compiler
@Deprecated  // This is an annotation - processed by compiler/tools
public void oldMethod() {}
```

**Difference:** Comments are ignored by the compiler, while annotations are retained and can be processed by the compiler, annotation processors, or runtime.

---

## Annotation Syntax and Structure

### Basic Annotation Syntax

```java
@interface MyAnnotation {
    // Annotation elements (similar to methods)
    String value();
    int count() default 1;
}
```

### Using Annotations

```java
@MyAnnotation(value = "example", count = 5)
public class MyClass {
    
    @MyAnnotation("single value")  // When 'value' is the only element
    public void myMethod() {}
}
```

### Annotation Elements

Annotation elements are declared like abstract methods:

```java
@interface ComplexAnnotation {
    String name();              // Required element
    int age() default 0;        // Optional with default
    String[] roles();           // Array type
    Class<?> type() default Object.class;  // Class type
    AnnotationType annotation() default @AnnotationType;  // Nested annotation
    enum Color { RED, GREEN, BLUE }
    Color color() default Color.RED;  // Enum type
}
```

### Element Types

Annotation elements can only be of these types:
- Primitive types: `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`
- `String`
- `Class`
- `Enum`
- Other annotations
- Arrays of the above types

**NOT allowed:** Objects, collections, generic types, or other complex types.

### The Special `value` Element

When an annotation has a single element named `value`, you can omit the element name:

```java
@interface Simple {
    String value();
}

// These are equivalent:
@Simple("text")
@Simple(value = "text")
```

### Default Values

Default values make elements optional:

```java
@interface Config {
    String name();
    int timeout() default 5000;
    boolean enabled() default true;
}

@Config(name = "service")  // timeout and enabled use defaults
public class Service {}
```

**Rules for default values:**
- Must be compile-time constants
- Cannot be `null` (use empty string or array instead)
- All non-default elements must be specified when using the annotation

---

## Meta-Annotations

Meta-annotations are annotations that apply to other annotations. They are defined in `java.lang.annotation` package.

### @Retention

Defines how long the annotation is retained:

```java
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.SOURCE)
@interface SourceOnly {}

@Retention(RetentionPolicy.CLASS)
@interface ClassOnly {}

@Retention(RetentionPolicy.RUNTIME)
@interface RuntimeVisible {}
```

### @Target

Specifies where the annotation can be applied:

```java
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;

@Target(ElementType.TYPE)  // Classes, interfaces, enums
@interface TypeAnnotation {}

@Target({ElementType.METHOD, ElementType.CONSTRUCTOR})
@interface MethodAnnotation {}

@Target({
    ElementType.TYPE,
    ElementType.FIELD,
    ElementType.METHOD,
    ElementType.PARAMETER,
    ElementType.CONSTRUCTOR,
    ElementType.LOCAL_VARIABLE,
    ElementType.ANNOTATION_TYPE,
    ElementType.PACKAGE,
    ElementType.TYPE_PARAMETER,  // Since Java 8
    ElementType.TYPE_USE         // Since Java 8
})
@interface Everywhere {}
```

### @Documented

Indicates that annotations should be included in Javadoc:

```java
import java.lang.annotation.Documented;

@Documented
@interface Important {
    String description();
}
```

### @Inherited

Indicates that the annotation type is automatically inherited:

```java
import java.lang.annotation.Inherited;

@Inherited
@interface ParentAnnotation {}

@ParentAnnotation
class Parent {}

class Child extends Parent {}  // Child inherits the annotation
```

**Important:** `@Inherited` only works for class inheritance, not for interfaces or methods.

### @Repeatable (Java 8+)

Allows the same annotation to be applied multiple times:

```java
import java.lang.annotation.Repeatable;

@interface Authors {
    Author[] value();
}

@Repeatable(Authors.class)
@interface Author {
    String name();
}

// Usage:
@Author(name = "Alice")
@Author(name = "Bob")
public class Book {}
```

---

## Retention Policies

The `@Retention` meta-annotation determines when annotations are discarded. This is crucial for understanding annotation internals.

### RetentionPolicy.SOURCE

```java
@Retention(RetentionPolicy.SOURCE)
@interface SourceAnnotation {}
```

**Lifecycle:**
- Retained only in source code
- Discarded by the compiler
- Not present in the compiled `.class` file
- **Use case:** Compiler warnings, code generation tools

**Internal behavior:**
```
Source Code → Compiler → [ANNOTATION DISCARDED] → Bytecode
```

### RetentionPolicy.CLASS (Default)

```java
@Retention(RetentionPolicy.CLASS)  // This is the default
@interface ClassAnnotation {}
```

**Lifecycle:**
- Retained in source code
- Written to the `.class` file
- Not loaded into the JVM at runtime
- **Use case:** Bytecode manipulation tools, post-compilation processing

**Internal behavior:**
```
Source Code → Compiler → Bytecode (with annotation) → JVM Runtime [ANNOTATION NOT LOADED]
```

### RetentionPolicy.RUNTIME

```java
@Retention(RetentionPolicy.RUNTIME)
@interface RuntimeAnnotation {}
```

**Lifecycle:**
- Retained in source code
- Written to the `.class` file
- Loaded into the JVM at runtime
- Accessible via reflection
- **Use case:** Frameworks (Spring, Hibernate), runtime introspection

**Internal behavior:**
```
Source Code → Compiler → Bytecode (with annotation) → JVM Runtime (annotation loaded)
```

### Retention Policy Comparison

| Policy | Source | Class File | Runtime | Reflection Access |
|--------|--------|------------|---------|-------------------|
| SOURCE | ✓ | ✗ | ✗ | ✗ |
| CLASS | ✓ | ✓ | ✗ | ✗ |
| RUNTIME | ✓ | ✓ | ✓ | ✓ |

### Choosing the Right Policy

```java
// SOURCE: For compile-time checks
@Retention(SOURCE)
@interface SuppressWarnings { }

// CLASS: For bytecode tools
@Retention(CLASS)
@interface BytecodeMarker { }

// RUNTIME: For frameworks and reflection
@Retention(RUNTIME)
@interface Component { }
```

---

## Annotation Targets

The `@Target` meta-annotation restricts where annotations can be applied.

### ElementType.TYPE

```java
@Target(ElementType.TYPE)
@interface ClassLevel {}
```

Applies to:
- Classes
- Interfaces
- Enums
- Annotation types

```java
@ClassLevel
public class MyClass {}
```

### ElementType.FIELD

```java
@Target(ElementType.FIELD)
@interface FieldLevel {}
```

Applies to:
- Instance fields
- Static fields

```java
public class Example {
    @FieldLevel
    private String name;
}
```

### ElementType.METHOD

```java
@Target(ElementType.METHOD)
@interface MethodLevel {}
```

Applies to:
- Instance methods
- Static methods

```java
public class Example {
    @MethodLevel
    public void doSomething() {}
}
```

### ElementType.PARAMETER

```java
@Target(ElementType.PARAMETER)
@interface ParamLevel {}
```

Applies to method parameters:

```java
public class Example {
    public void method(@ParamLevel String param) {}
}
```

### ElementType.CONSTRUCTOR

```java
@Target(ElementType.CONSTRUCTOR)
@interface ConstructorLevel {}
```

Applies to constructors:

```java
public class Example {
    @ConstructorLevel
    public Example() {}
}
```

### ElementType.LOCAL_VARIABLE

```java
@Target(ElementType.LOCAL_VARIABLE)
@interface LocalVarLevel {}
```

Applies to local variables:

```java
public void method() {
    @LocalVarLevel
    int x = 10;
}
```

### ElementType.ANNOTATION_TYPE

```java
@Target(ElementType.ANNOTATION_TYPE)
@interface MetaAnnotation {}
```

Applies to other annotations:

```java
@MetaAnnotation
@interface MyAnnotation {}
```

### ElementType.PACKAGE

```java
@Target(ElementType.PACKAGE)
@interface PackageLevel {}
```

Applied in `package-info.java`:

```java
@PackageLevel
package com.example;
```

### ElementType.TYPE_PARAMETER (Java 8+)

```java
@Target(ElementType.TYPE_PARAMETER)
@interface TypeParamAnnotation {}
```

Applies to type parameter declarations:

```java
public class Box<@TypeParamAnnotation T> {}
```

### ElementType.TYPE_USE (Java 8+)

```java
@Target(ElementType.TYPE_USE)
@interface TypeUseAnnotation {}
```

Applies to any use of a type:

```java
public class Example {
    List<@TypeUseAnnotation String> list;  // In type argument
    public <@TypeUseAnnotation T> void method() {}  // Type parameter
    String @TypeUseAnnotation [] array;  // Array component type
    public void method() throws @TypeUseAnnotation Exception {}  // Exception type
}
```

---

## Built-in Java Annotations

Java provides several built-in annotations in the `java.lang` package.

### @Override

```java
class Parent {
    public void method() {}
}

class Child extends Parent {
    @Override
    public void method() {}  // Compiler verifies this overrides a parent method
    
    // @Override  // Compile error: no method to override
    public void newMethod() {}
}
```

**Purpose:** Ensures a method actually overrides a superclass method.

**Internal behavior:**
- Compiler checks if the annotated method overrides a method in a superclass or implements an interface method
- If not, compilation fails
- This is a SOURCE-level annotation (RetentionPolicy.SOURCE)

### @Deprecated

```java
class OldClass {
    @Deprecated
    public void oldMethod() {
        // This method should not be used anymore
    }
}

// Usage generates warning:
OldClass obj = new OldClass();
obj.oldMethod();  // Warning: oldMethod() has been deprecated
```

**Purpose:** Marks elements as obsolete, encouraging alternative usage.

**Internal behavior:**
- RetentionPolicy.RUNTIME (accessible via reflection)
- Compiler generates warnings when deprecated elements are used
- Javadoc includes deprecated elements in special section

**Customizing deprecation:**

```java
@Deprecated(since = "1.5", forRemoval = true)
public void veryOldMethod() {}
```

### @SuppressWarnings

```java
class Example {
    @SuppressWarnings("unchecked")
    public void method() {
        List list = new ArrayList();  // No unchecked warning
        list.add("string");
    }
    
    @SuppressWarnings({"unchecked", "rawtypes"})
    public void anotherMethod() {
        List list = new ArrayList();  // No warnings
    }
}
```

**Purpose:** Instructs compiler to suppress specific warnings.

**Common warning types:**
- `"unchecked"` - unchecked generic operations
- `"rawtypes"` - raw types
- `"deprecation"` - use of deprecated elements
- `"unused"` - unused variables/methods
- `"serial"` - missing serialVersionUID

**Internal behavior:**
- RetentionPolicy.SOURCE
- Compiler-specific (not standardized)
- Only affects the annotated element and its children

### @SafeVarargs (Java 7+)

```java
@SafeVarargs
public final void method(String... args) {
    // Asserts that the method doesn't perform unsafe operations
}
```

**Purpose:** Suppresses warnings for varargs methods with generic types.

**Use case:** When you're certain your varargs method won't cause heap pollution.

### @FunctionalInterface (Java 8+)

```java
@FunctionalInterface
interface MyFunctionalInterface {
    void singleMethod();
    
    // Can have default methods
    default void defaultMethod() {}
    
    // Can have Object methods
    @Override
    boolean equals(Object obj);
    
    // void anotherMethod();  // Compile error: more than one abstract method
}
```

**Purpose:** Ensures the interface has exactly one abstract method (functional interface).

**Internal behavior:**
- Compiler verifies the interface meets functional interface criteria
- RetentionPolicy.RUNTIME

### @Native (Java 8+)

```java
class Example {
    @Native
    public static final int CONSTANT = 123;
}
```

**Purpose:** Indicates a constant field is referenced from native code.

**Internal behavior:**
- Hint to native code generators
- RetentionPolicy.SOURCE

---

## Creating Custom Annotations

### Step-by-Step Custom Annotation

Let's create a comprehensive custom annotation for validation:

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Validate {
    String message() default "Invalid value";
    int min() default Integer.MIN_VALUE;
    int max() default Integer.MAX_VALUE;
    boolean required() default true;
    String pattern() default "";
}
```

### Using the Custom Annotation

```java
public class User {
    @Validate(message = "Name cannot be empty", required = true)
    private String name;
    
    @Validate(min = 0, max = 120, message = "Invalid age")
    private int age;
    
    @Validate(pattern = "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$", 
             message = "Invalid email format")
    private String email;
    
    // Constructors, getters, setters...
}
```

### Complex Custom Annotation with Nested Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Entity {
    String tableName() default "";
    
    Column[] columns() default {};
    
    Index[] indexes() default {};
}

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Column {
    String name();
    String type() default "VARCHAR";
    boolean nullable() default true;
    int length() default 255;
}

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Index {
    String name();
    String[] columns();
    boolean unique() default false;
}
```

**Usage:**

```java
@Entity(
    tableName = "users",
    columns = {
        @Column(name = "id", type = "BIGINT", nullable = false),
        @Column(name = "username", length = 50, nullable = false),
        @Column(name = "email", length = 100)
    },
    indexes = {
        @Index(name = "idx_username", columns = {"username"}, unique = true)
    }
)
public class User {
    private Long id;
    private String username;
    private String email;
}
```

### Annotation with Enum Values

```java
public enum AccessLevel {
    PUBLIC, PRIVATE, PROTECTED, PACKAGE
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Security {
    AccessLevel level() default AccessLevel.PUBLIC;
    String[] roles() default {};
}
```

**Usage:**

```java
@Security(level = AccessLevel.PRIVATE, roles = {"ADMIN", "MODERATOR"})
public class AdminPanel {
    
    @Security(level = AccessLevel.PUBLIC)
    public void viewInfo() {}
    
    @Security(level = AccessLevel.PROTECTED, roles = {"ADMIN"})
    public void deleteData() {}
}
```

---

## Annotation Processing Internals

Annotation processing is the mechanism by which annotations are read and processed. This happens at compile-time.

### The Annotation Processing Pipeline

```
Source Code (.java)
    ↓
Java Parser (AST)
    ↓
Symbol Resolution
    ↓
Annotation Processing Round 1
    ↓
[Generate new source files]
    ↓
Annotation Processing Round 2
    ↓
[Generate new source files]
    ↓
...
    ↓
Code Generation (.class files)
```

### Creating an Annotation Processor

```java
import javax.annotation.processing.*;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.*;
import javax.tools.Diagnostic;
import java.util.Set;
import java.util.HashSet;

@SupportedAnnotationTypes("com.example.Validate")
@SupportedSourceVersion(SourceVersion.RELEASE_11)
public class ValidateProcessor extends AbstractProcessor {
    
    @Override
    public boolean process(Set<? extends TypeElement> annotations, 
                          RoundEnvironment roundEnv) {
        
        for (TypeElement annotation : annotations) {
            Set<? extends Element> annotatedElements = 
                roundEnv.getElementsAnnotatedWith(annotation);
            
            for (Element element : annotatedElements) {
                if (element.getKind() == ElementKind.FIELD) {
                    VariableElement field = (VariableElement) element;
                    
                    // Get annotation values
                    Validate validate = field.getAnnotation(Validate.class);
                    
                    // Generate validation code
                    generateValidationCode(field, validate);
                }
            }
        }
        
        return true;  // Claim these annotations
    }
    
    private void generateValidationCode(VariableElement field, Validate validate) {
        processingEnv.getMessager().printMessage(
            Diagnostic.Kind.NOTE,
            "Processing field: " + field.getSimpleName() +
            " with message: " + validate.message()
        );
        // In a real processor, you would generate Java source files here
    }
}
```

### Registering the Processor

Create `META-INF/services/javax.annotation.processing.Processor`:

```
com.example.ValidateProcessor
```

### Processing Rounds

Annotation processing happens in rounds:

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, 
                      RoundEnvironment roundEnv) {
    
    // Check if this is the last round
    if (roundEnv.processingOver()) {
        // Clean up or final generation
        return false;
    }
    
    // Process annotations
    // ...
    
    // Return true if we processed the annotations
    // Return false to let other processors handle them
    return true;
}
```

### Generating Source Files

```java
private void generateValidatorClass(TypeElement classElement) 
    throws IOException {
    
    String className = classElement.getSimpleName() + "Validator";
    
    JavaFileObject builderFile = processingEnv.getFiler()
        .createSourceFile("com.example.generated." + className);
    
    try (PrintWriter out = new PrintWriter(builderFile.openWriter())) {
        out.println("package com.example.generated;");
        out.println();
        out.println("public class " + className + " {");
        out.println("    public static boolean validate(Object obj) {");
        out.println("        // Generated validation logic");
        out.println("        return true;");
        out.println("    }");
        out.println("}");
    }
}
```

### The AbstractProcessor API

Key methods and their purposes:

```java
public abstract class AbstractProcessor implements Processor {
    
    // Initialize the processor with processing environment
    public synchronized void init(ProcessingEnvironment processingEnv) {
        this.processingEnv = processingEnv;
        // Access to: Messager, Filer, Elements, Types
    }
    
    // Return the annotation types this processor supports
    public Set<String> getSupportedAnnotationTypes() {
        return Set.of("com.example.*");
    }
    
    // Return the source version this processor supports
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.RELEASE_11;
    }
    
    // Process the annotations
    public abstract boolean process(Set<? extends TypeElement> annotations,
                                    RoundEnvironment roundEnv);
}
```

### ProcessingEnvironment Components

```java
@SupportedAnnotationTypes("*")
@SupportedSourceVersion(SourceVersion.RELEASE_11)
public class MyProcessor extends AbstractProcessor {
    
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        
        // Messager: For reporting errors, warnings, notes
        Messager messager = processingEnv.getMessager();
        
        // Filer: For creating new source, class, resource files
        Filer filer = processingEnv.getFiler();
        
        // Elements: Utility methods for operating on program elements
        Elements elementUtils = processingEnv.getElementUtils();
        
        // Types: Utility methods for operating on types
        Types typeUtils = processingEnv.getTypeUtils();
    }
}
```

### Element Hierarchy

The `javax.lang.model.element` package represents the program structure:

```
Element (interface)
├── PackageElement (packages)
├── TypeElement (classes, interfaces, enums, annotations)
├── VariableElement (fields, parameters, local variables)
├── ExecutableElement (methods, constructors)
├── TypeParameterElement (type parameters)
└── ModuleElement (modules, Java 9+)
```

### Example: Processing All Fields

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, 
                      RoundEnvironment roundEnv) {
    
    // Get all elements annotated with @Validate
    Set<? extends Element> validatedFields = 
        roundEnv.getElementsAnnotatedWith(Validate.class);
    
    for (Element element : validatedFields) {
        // Get enclosing class
        TypeElement enclosingClass = (TypeElement) element.getEnclosingElement();
        
        // Get package
        PackageElement packageElement = 
            (PackageElement) enclosingClass.getEnclosingElement();
        
        processingEnv.getMessager().printMessage(
            Diagnostic.Kind.NOTE,
            "Package: " + packageElement.getQualifiedName() +
            ", Class: " + enclosingClass.getSimpleName() +
            ", Field: " + element.getSimpleName()
        );
    }
    
    return true;
}
```

---

## Runtime Reflection with Annotations

Runtime reflection allows you to read annotations at runtime using the Reflection API.

### Reading Class Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Author {
    String name();
    String date();
}

@Author(name = "John Doe", date = "2024")
public class Book {}

// Reading the annotation:
public class Main {
    public static void main(String[] args) {
        Class<Book> bookClass = Book.class;
        
        if (bookClass.isAnnotationPresent(Author.class)) {
            Author author = bookClass.getAnnotation(Author.class);
            System.out.println("Author: " + author.name());
            System.out.println("Date: " + author.date());
        }
    }
}
```

### Reading Method Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface LogExecution {
    String level() default "INFO";
}

public class Service {
    @LogExecution(level = "DEBUG")
    public void process() {}
}

// Reading method annotations:
public class Main {
    public static void main(String[] args) throws Exception {
        Class<Service> serviceClass = Service.class;
        Method method = serviceClass.getMethod("process");
        
        if (method.isAnnotationPresent(LogExecution.class)) {
            LogExecution log = method.getAnnotation(LogExecution.class);
            System.out.println("Log level: " + log.level());
        }
    }
}
```

### Reading Field Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Inject {
    String value() default "";
}

public class Configuration {
    @Inject("database.url")
    private String dbUrl;
    
    @Inject
    private String timeout;
}

// Reading field annotations:
public class Main {
    public static void main(String[] args) throws Exception {
        Class<Configuration> configClass = Configuration.class;
        
        for (Field field : configClass.getDeclaredFields()) {
            if (field.isAnnotationPresent(Inject.class)) {
                Inject inject = field.getAnnotation(Inject.class);
                System.out.println("Field: " + field.getName());
                System.out.println("Value: " + inject.value());
            }
        }
    }
}
```

### Reading Parameter Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@interface NotNull {}

public class Validator {
    public void validate(@NotNull String input) {}
}

// Reading parameter annotations:
public class Main {
    public static void main(String[] args) throws Exception {
        Class<Validator> validatorClass = Validator.class;
        Method method = validatorClass.getMethod("validate", String.class);
        
        Parameter[] parameters = method.getParameters();
        for (Parameter param : parameters) {
            if (param.isAnnotationPresent(NotNull.class)) {
                System.out.println(param.getName() + " is @NotNull");
            }
        }
    }
}
```

### Reading Multiple Annotations of Same Type (Java 8+)

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Repeatable(Authors.class)
@interface Author {
    String name();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Authors {
    Author[] value();
}

@Author(name = "Alice")
@Author(name = "Bob")
public class MultiAuthorBook {}

// Reading repeated annotations:
public class Main {
    public static void main(String[] args) {
        Class<MultiAuthorBook> bookClass = MultiAuthorBook.class;
        
        // Method 1: Using getAnnotationsByType
        Author[] authors = bookClass.getAnnotationsByType(Author.class);
        for (Author author : authors) {
            System.out.println("Author: " + author.name());
        }
        
        // Method 2: Using the container annotation
        if (bookClass.isAnnotationPresent(Authors.class)) {
            Authors authorsContainer = bookClass.getAnnotation(Authors.class);
            for (Author author : authorsContainer.value()) {
                System.out.println("Author: " + author.name());
            }
        }
    }
}
```

### Getting All Annotations

```java
public class Main {
    public static void main(String[] args) {
        Class<String> stringClass = String.class;
        
        // Get all annotations (including inherited)
        Annotation[] annotations = stringClass.getAnnotations();
        for (Annotation annotation : annotations) {
            System.out.println(annotation.annotationType().getSimpleName());
        }
        
        // Get only directly present annotations
        Annotation[] directAnnotations = stringClass.getDeclaredAnnotations();
        for (Annotation annotation : directAnnotations) {
            System.out.println(annotation.annotationType().getSimpleName());
        }
    }
}
```

### Annotation Reflection Methods Summary

| Method | Description | Includes Inherited |
|--------|-------------|-------------------|
| `getAnnotation(Class<T>)` | Get specific annotation | Yes |
| `getAnnotations()` | Get all annotations | Yes |
| `getDeclaredAnnotation(Class<T>)` | Get specific annotation (direct only) | No |
| `getDeclaredAnnotations()` | Get all annotations (direct only) | No |
| `isAnnotationPresent(Class<T>)` | Check if annotation present | Yes |
| `getAnnotationsByType(Class<T>)` | Get repeated annotations | Yes |

### Dynamic Annotation Invocation

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Cacheable {
    long ttl() default 3600;
}

public class CacheExample {
    @Cacheable(ttl = 7200)
    public String getData() {
        return "data";
    }
}

// Dynamically reading annotation values:
public class Main {
    public static void main(String[] args) throws Exception {
        Method method = CacheExample.class.getMethod("getData");
        Cacheable cacheable = method.getAnnotation(Cacheable.class);
        
        // Dynamically invoke annotation methods
        Method ttlMethod = cacheable.annotationType().getMethod("ttl");
        long ttl = (Long) ttlMethod.invoke(cacheable);
        
        System.out.println("TTL: " + ttl);
    }
}
```

### Building a Simple Framework with Annotations

```java
// Annotation definition
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Route {
    String path();
    String method() default "GET";
}

// Controller
class UserController {
    @Route(path = "/users", method = "GET")
    public String getAllUsers() {
        return "[{\"id\": 1, \"name\": \"John\"}]";
    }
    
    @Route(path = "/users/{id}", method = "GET")
    public String getUser(String id) {
        return "{\"id\": " + id + ", \"name\": \"John\"}";
    }
}

// Simple framework
public class SimpleFramework {
    public static void main(String[] args) throws Exception {
        registerController(UserController.class);
    }
    
    public static void registerController(Class<?> controllerClass) {
        for (Method method : controllerClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Route.class)) {
                Route route = method.getAnnotation(Route.class);
                System.out.println("Registered route: " + 
                    route.method() + " " + route.path() + 
                    " -> " + controllerClass.getSimpleName() + 
                    "." + method.getName());
            }
        }
    }
}
```

---

## Bytecode Representation

Understanding how annotations are stored in bytecode provides insight into annotation internals.

### Bytecode Annotation Structure

Annotations are stored in the `.class` file in specific attributes:

```
ClassFile {
    ...
    RuntimeVisibleAnnotations_attribute {  // For RUNTIME retention
        u2 attribute_name_index;
        u4 attribute_length;
        u2 num_annotations;
        annotation annotations[num_annotations];
    }
    
    RuntimeInvisibleAnnotations_attribute {  // For CLASS retention
        // Same structure
    }
}
```

### Annotation Structure in Bytecode

```
annotation {
    u2 type_index;           // Index to CONSTANT_Utf8 (annotation type)
    u2 num_element_value_pairs;
    {
        u2 element_name_index;  // Index to CONSTANT_Utf8 (element name)
        element_value value;    // Element value
    } [num_element_value_pairs];
}
```

### Element Value Types

```
element_value {
    u1 tag;  // Indicates the type
    union {
        u2 const_value_index;           // For primitives, String
        enum_const_value;               // For enums
        class_info_index;               // For Class
        annotation;                     // For nested annotations
        array_value;                    // For arrays
    } value;
}

tag values:
B: byte
C: char
D: double
F: float
I: int
J: long
S: short
Z: boolean
s: String
e: enum_const_value
c: class_info_index
@: annotation
[: array_value
```

### Examining Bytecode with javap

Let's examine the bytecode of an annotated class:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface MyAnnotation {
    String value();
}

@MyAnnotation("example")
public class AnnotatedClass {}
```

Compile and examine:

```bash
javac AnnotatedClass.java
javap -v AnnotatedClass
```

**Output (relevant parts):**

```
RuntimeVisibleAnnotations:
  0: #12()
    com.example.MyAnnotation
    value="example"
```

### Comparing Retention Policies in Bytecode

**SOURCE retention:**
```java
@Retention(SOURCE)
@interface SourceAnno {}
@SourceAnno
class TestSource {}
```
```
javap -v TestSource
# No annotation attributes in bytecode
```

**CLASS retention:**
```java
@Retention(CLASS)
@interface ClassAnno {}
@ClassAnno
class TestClass {}
```
```
javap -v TestClass
RuntimeInvisibleAnnotations:
  0: #12()
    com.example.ClassAnno
```

**RUNTIME retention:**
```java
@Retention(RUNTIME)
@interface RuntimeAnno {}
@RuntimeAnno
class TestRuntime {}
```
```
javap -v TestRuntime
RuntimeVisibleAnnotations:
  0: #12()
    com.example.RuntimeAnno
```

### Constant Pool for Annotations

Annotations reference the constant pool heavily:

```
Constant pool:
   #1 = Class              #2             // com/example/AnnotatedClass
   #2 = Utf8               com/example/AnnotatedClass
   #3 = Class              #4             // java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               SourceFile
   #6 = Utf8               AnnotatedClass.java
   #7 = Utf8               RuntimeVisibleAnnotations
   #8 = Utf8               Lcom/example/MyAnnotation;
   #9 = Utf8               value
  #10 = Utf8               example
  #11 = Utf8               com/example/MyAnnotation
  #12 = Utf8               Lcom/example/MyAnnotation;
```

### Method Annotations in Bytecode

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Log {}

class Test {
    @Log
    public void method() {}
}
```

**Bytecode output:**
```
public void method();
  descriptor: ()V
  flags: (0x0001) ACC_PUBLIC
  RuntimeVisibleAnnotations:
    0: #14()
      com.example.Log
```

### Field Annotations in Bytecode

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Inject {}

class Test {
    @Inject
    private String field;
}
```

**Bytecode output:**
```
private java.lang.String field;
  descriptor: Ljava/lang/String;
  flags: (0x0002) ACC_PRIVATE
  RuntimeVisibleAnnotations:
    0: #14()
      com.example.Inject
```

### Parameter Annotations in Bytecode

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@interface Valid {}

class Test {
    public void method(@Valid String param) {}
}
```

**Bytecode output:**
```
public void method(java.lang.String);
  descriptor: (Ljava/lang/String;)V
  flags: (0x0001) ACC_PUBLIC
  RuntimeVisibleParameterAnnotations:
    parameter 0:
      0: #14()
        com.example.Valid
```

### Type Annotations (Java 8+) in Bytecode

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE_USE)
@interface TypeAnno {}

class Test {
    List<@TypeAnno String> list;
}
```

**Bytecode output:**
```
RuntimeVisibleTypeAnnotations:
  0: #14(): FIELD, location=[TYPE_ARGUMENT(0)]
    com.example.TypeAnno
```

---

## Annotation Inheritance

Understanding how annotations behave with inheritance is crucial for framework design.

### @Inherited Behavior

The `@Inherited` meta-annotation affects only class-level annotations:

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface ParentAnno {
    String value();
}

@ParentAnno("parent")
class Parent {}

class Child extends Parent {}

public class Main {
    public static void main(String[] args) {
        // Child inherits ParentAnno from Parent
        System.out.println(Child.class.isAnnotationPresent(ParentAnno.class));
        // Output: true
        
        ParentAnno anno = Child.class.getAnnotation(ParentAnno.class);
        System.out.println(anno.value());
        // Output: parent
    }
}
```

### @Inherited Limitations

**1. Does not work for interfaces:**

```java
@Inherited
@interface MyAnno {}

@MyAnno
interface MyInterface {}

class MyClass implements MyInterface {}

public class Main {
    public static void main(String[] args) {
        System.out.println(MyClass.class.isAnnotationPresent(MyAnno.class));
        // Output: false - interfaces don't inherit annotations
    }
}
```

**2. Does not work for methods:**

```java
@Inherited
@interface MethodAnno {}

class Parent {
    @MethodAnno
    public void method() {}
}

class Child extends Parent {
    @Override
    public void method() {}
}

public class Main {
    public static void main(String[] args) throws Exception {
        Method parentMethod = Parent.class.getMethod("method");
        Method childMethod = Child.class.getMethod("method");
        
        System.out.println(parentMethod.isAnnotationPresent(MethodAnno.class));
        // Output: true
        
        System.out.println(childMethod.isAnnotationPresent(MethodAnno.class));
        // Output: false - methods don't inherit annotations
    }
}
```

**3. Only affects direct inheritance:**

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Anno {}

@Anno
class GrandParent {}

class Parent extends GrandParent {}

class Child extends Parent {}

public class Main {
    public static void main(String[] args) {
        System.out.println(Child.class.isAnnotationPresent(Anno.class));
        // Output: true - inherited through chain
    }
}
```

### Overriding Inherited Annotations

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Anno {
    String value();
}

@Anno("parent")
class Parent {}

@Anno("child")
class Child extends Parent {}

public class Main {
    public static void main(String[] args) {
        Anno anno = Child.class.getAnnotation(Anno.class);
        System.out.println(anno.value());
        // Output: child - child's annotation overrides parent's
    }
}
```

### Getting All Annotations (Including Inherited)

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface InheritedAnno {}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface DirectAnno {}

@InheritedAnno
@DirectAnno
class Parent {}

class Child extends Parent {}

public class Main {
    public static void main(String[] args) {
        // getAnnotations() includes inherited annotations
        Annotation[] all = Child.class.getAnnotations();
        for (Annotation a : all) {
            System.out.println(a.annotationType().getSimpleName());
        }
        // Output:
        // DirectAnno
        // InheritedAnno
        
        // getDeclaredAnnotations() does NOT include inherited
        Annotation[] direct = Child.class.getDeclaredAnnotations();
        for (Annotation a : direct) {
            System.out.println(a.annotationType().getSimpleName());
        }
        // Output:
        // DirectAnno
    }
}
```

### Practical Framework Pattern

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Component {
    String name() default "";
}

@Component(name = "baseService")
class BaseService {}

class UserService extends BaseService {}

public class Framework {
    public static void main(String[] args) {
        // Scan for components
        scanComponents(UserService.class);
    }
    
    public static void scanComponents(Class<?> clazz) {
        if (clazz.isAnnotationPresent(Component.class)) {
            Component component = clazz.getAnnotation(Component.class);
            String name = component.name().isEmpty() ? 
                clazz.getSimpleName() : component.name();
            System.out.println("Found component: " + name);
        }
    }
}
// Output: Found component: baseService
```

---

## Practical Examples

### Example 1: Dependency Injection Framework

```java
// Annotation definitions
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Inject {
    String name() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Component {
    String value() default "";
}

// Services
@Component("userService")
class UserService {
    public void greet() {
        System.out.println("Hello from UserService!");
    }
}

@Component
class OrderService {
    @Inject("userService")
    private UserService userService;
    
    public void processOrder() {
        userService.greet();
        System.out.println("Processing order...");
    }
}

// Simple DI Container
class DIContainer {
    private static final Map<String, Object> beans = new HashMap<>();
    
    public static void scanAndInitialize(String basePackage) 
        throws Exception {
        // In a real implementation, you would scan the package
        // For simplicity, we'll manually register
        registerBean(UserService.class);
        registerBean(OrderService.class);
        
        // Inject dependencies
        performDependencyInjection();
    }
    
    private static void registerBean(Class<?> clazz) 
        throws Exception {
        if (clazz.isAnnotationPresent(Component.class)) {
            Component component = clazz.getAnnotation(Component.class);
            String beanName = component.value().isEmpty() ? 
                clazz.getSimpleName() : component.value();
            Object instance = clazz.getDeclaredConstructor().newInstance();
            beans.put(beanName, instance);
            System.out.println("Registered bean: " + beanName);
        }
    }
    
    private static void performDependencyInjection() 
        throws Exception {
        for (Object bean : beans.values()) {
            for (Field field : bean.getClass().getDeclaredFields()) {
                if (field.isAnnotationPresent(Inject.class)) {
                    Inject inject = field.getAnnotation(Inject.class);
                    String dependencyName = inject.name().isEmpty() ? 
                        field.getType().getSimpleName() : inject.name();
                    
                    Object dependency = beans.get(dependencyName);
                    if (dependency != null) {
                        field.setAccessible(true);
                        field.set(bean, dependency);
                        System.out.println("Injected " + dependencyName + 
                            " into " + bean.getClass().getSimpleName());
                    }
                }
            }
        }
    }
    
    @SuppressWarnings("unchecked")
    public static <T> T getBean(Class<T> clazz) {
        String beanName = clazz.getSimpleName();
        return (T) beans.get(beanName);
    }
    
    public static <T> T getBean(String name) {
        return (T) beans.get(name);
    }
}

// Usage
public class DIExample {
    public static void main(String[] args) throws Exception {
        DIContainer.scanAndInitialize("com.example");
        
        OrderService orderService = DIContainer.getBean(OrderService.class);
        orderService.processOrder();
    }
}
```

### Example 2: Validation Framework

```java
// Annotation definitions
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Validate {
    String message() default "Invalid value";
    int min() default Integer.MIN_VALUE;
    int max() default Integer.MAX_VALUE;
    boolean required() default false;
    String pattern() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Email {
    String message() default "Invalid email";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Validated {}

// Model class
@Validated
class UserForm {
    @Validate(required = true, message = "Name is required")
    private String name;
    
    @Validate(min = 18, max = 120, message = "Age must be between 18 and 120")
    private int age;
    
    @Email
    private String email;
    
    // Constructors, getters, setters
    public UserForm() {}
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}

// Validation result
class ValidationResult {
    private final List<String> errors = new ArrayList<>();
    
    public void addError(String error) {
        errors.add(error);
    }
    
    public boolean isValid() {
        return errors.isEmpty();
    }
    
    public List<String> getErrors() {
        return errors;
    }
}

// Validator
class Validator {
    public static ValidationResult validate(Object obj) {
        ValidationResult result = new ValidationResult();
        
        Class<?> clazz = obj.getClass();
        
        // Only validate if @Validated is present
        if (!clazz.isAnnotationPresent(Validated.class)) {
            return result;
        }
        
        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            
            try {
                Object value = field.get(obj);
                
                // @Validate annotation
                if (field.isAnnotationPresent(Validate.class)) {
                    Validate validate = field.getAnnotation(Validate.class);
                    validateField(field, value, validate, result);
                }
                
                // @Email annotation
                if (field.isAnnotationPresent(Email.class)) {
                    Email email = field.getAnnotation(Email.class);
                    validateEmail(field, value, email, result);
                }
                
            } catch (IllegalAccessException e) {
                result.addError("Cannot access field: " + field.getName());
            }
        }
        
        return result;
    }
    
    private static void validateField(Field field, Object value, 
                                     Validate validate, 
                                     ValidationResult result) {
        String fieldName = field.getName();
        
        // Required check
        if (validate.required() && (value == null || 
            (value instanceof String && ((String) value).isEmpty()))) {
            result.addError(validate.message());
            return;
        }
        
        if (value == null) return;  // Skip other validations if null and not required
        
        // Min/Max for numbers
        if (value instanceof Number) {
            Number num = (Number) value;
            if (num.intValue() < validate.min()) {
                result.addError(fieldName + " must be at least " + validate.min());
            }
            if (num.intValue() > validate.max()) {
                result.addError(fieldName + " must be at most " + validate.max());
            }
        }
        
        // Pattern validation for strings
        if (value instanceof String && !validate.pattern().isEmpty()) {
            String str = (String) value;
            if (!str.matches(validate.pattern())) {
                result.addError(fieldName + " does not match required pattern");
            }
        }
    }
    
    private static void validateEmail(Field field, Object value, 
                                      Email email, 
                                      ValidationResult result) {
        if (value == null) return;
        
        String emailPattern = "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$";
        if (!value.toString().matches(emailPattern)) {
            result.addError(email.message());
        }
    }
}

// Usage
public class ValidationExample {
    public static void main(String[] args) {
        UserForm form = new UserForm();
        form.setName("");  // Invalid: required
        form.setAge(15);   // Invalid: below min
        form.setEmail("invalid-email");  // Invalid: not an email
        
        ValidationResult result = Validator.validate(form);
        
        if (result.isValid()) {
            System.out.println("Form is valid!");
        } else {
            System.out.println("Validation errors:");
            for (String error : result.getErrors()) {
                System.out.println("  - " + error);
            }
        }
    }
}
```

### Example 3: Aspect-Oriented Programming (AOP) Style

```java
// Annotation definitions
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Before {
    String value();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface After {
    String value();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Around {
    String value();
}

// Service class
class BankingService {
    @Before("logTransaction")
    @After("logTransaction")
    public void transfer(String from, String to, double amount) {
        System.out.println("Transferring " + amount + " from " + 
                          from + " to " + to);
    }
    
    @Around("securityCheck")
    public void sensitiveOperation() {
        System.out.println("Performing sensitive operation");
    }
}

// AOP Processor
class AOPProcessor {
    private static final Map<String, Runnable> advice = new HashMap<>();
    
    static {
        advice.put("logTransaction", () -> 
            System.out.println("Transaction logged"));
        advice.put("securityCheck", () -> 
            System.out.println("Security check passed"));
    }
    
    public static <T> T createProxy(Class<T> interfaceClass, T target) {
        return (T) Proxy.newProxyInstance(
            interfaceClass.getClassLoader(),
            new Class<?>[] { interfaceClass },
            (proxy, method, args) -> {
                // Process @Before
                if (method.isAnnotationPresent(Before.class)) {
                    Before before = method.getAnnotation(Before.class);
                    advice.get(before.value()).run();
                }
                
                // Process @Around
                if (method.isAnnotationPresent(Around.class)) {
                    Around around = method.getAnnotation(Around.class);
                    advice.get(around.value()).run();
                }
                
                // Execute actual method
                Object result = method.invoke(target, args);
                
                // Process @After
                if (method.isAnnotationPresent(After.class)) {
                    After after = method.getAnnotation(After.class);
                    advice.get(after.value()).run();
                }
                
                return result;
            }
        );
    }
}

// Usage
public class AOPExample {
    public static void main(String[] args) throws Exception {
        BankingService service = new BankingService();
        
        // In a real AOP framework, this would be done automatically
        // For simplicity, we're calling methods directly with manual advice
        Method transferMethod = BankingService.class.getMethod(
            "transfer", String.class, String.class, double.class);
        
        // Check for @Before
        if (transferMethod.isAnnotationPresent(Before.class)) {
            Before before = transferMethod.getAnnotation(Before.class);
            System.out.println("Before advice: " + before.value());
        }
        
        // Execute method
        transferMethod.invoke(service, "Alice", "Bob", 100.0);
        
        // Check for @After
        if (transferMethod.isAnnotationPresent(After.class)) {
            After after = transferMethod.getAnnotation(After.class);
            System.out.println("After advice: " + after.value());
        }
    }
}
```

### Example 4: Serialization/Deserialization

```java
// Annotation definitions
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Serializable {}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Serialize {
    String name() default "";
    String format() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Ignore {}

// Model class
@Serializable
class Person {
    @Serialize(name = "full_name")
    private String name;
    
    @Serialize(format = "yyyy-MM-dd")
    private Date birthDate;
    
    @Serialize
    private int age;
    
    @Ignore
    private String password;
    
    // Constructors, getters, setters
    public Person() {}
    
    public Person(String name, Date birthDate, int age, String password) {
        this.name = name;
        this.birthDate = birthDate;
        this.age = age;
        this.password = password;
    }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public Date getBirthDate() { return birthDate; }
    public void setBirthDate(Date birthDate) { this.birthDate = birthDate; }
    
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}

// Serializer
class JsonSerializer {
    public static String serialize(Object obj) throws Exception {
        Class<?> clazz = obj.getClass();
        
        if (!clazz.isAnnotationPresent(Serializable.class)) {
            throw new IllegalArgumentException(
                "Class must be annotated with @Serializable");
        }
        
        StringBuilder json = new StringBuilder("{");
        
        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            
            // Skip @Ignore fields
            if (field.isAnnotationPresent(Ignore.class)) {
                continue;
            }
            
            // Only serialize @Serialize fields
            if (field.isAnnotationPresent(Serialize.class)) {
                Serialize serialize = field.getAnnotation(Serialize.class);
                String fieldName = serialize.name().isEmpty() ? 
                    field.getName() : serialize.name();
                
                Object value = field.get(obj);
                String formattedValue = formatValue(value, serialize);
                
                if (json.length() > 1) {
                    json.append(",");
                }
                
                json.append("\"").append(fieldName).append("\":")
                    .append(formattedValue);
            }
        }
        
        json.append("}");
        return json.toString();
    }
    
    private static String formatValue(Object value, Serialize serialize) {
        if (value == null) {
            return "null";
        }
        
        if (value instanceof String) {
            return "\"" + value + "\"";
        }
        
        if (value instanceof Date) {
            SimpleDateFormat sdf = new SimpleDateFormat(
                serialize.format().isEmpty() ? "yyyy-MM-dd" : serialize.format());
            return "\"" + sdf.format((Date) value) + "\"";
        }
        
        return String.valueOf(value);
    }
}

// Usage
public class SerializationExample {
    public static void main(String[] args) throws Exception {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        Person person = new Person(
            "John Doe",
            sdf.parse("1990-05-15"),
            34,
            "secret123"
        );
        
        String json = JsonSerializer.serialize(person);
        System.out.println(json);
        // Output: {"full_name":"John Doe","birthDate":"1990-05-15","age":34}
    }
}
```

---

## Performance Considerations

### Annotation Processing Overhead

**Compile-time:**
- Annotation processors add to compilation time
- Complex processors can significantly slow builds
- Incremental compilation helps mitigate this

**Runtime:**
- `isAnnotationPresent()` is relatively fast
- `getAnnotation()` involves reflection overhead
- Caching annotation data improves performance

### Reflection Performance

```java
// Benchmark: Reading annotations
public class AnnotationBenchmark {
    public static void main(String[] args) throws Exception {
        Class<TestClass> clazz = TestClass.class;
        
        // Warm up
        for (int i = 0; i < 10000; i++) {
            clazz.isAnnotationPresent(MyAnnotation.class);
        }
        
        // Benchmark
        long start = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            clazz.isAnnotationPresent(MyAnnotation.class);
        }
        long end = System.nanoTime();
        
        System.out.println("Time: " + (end - start) / 1_000_000 + " ms");
    }
}

@MyAnnotation
class TestClass {}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface MyAnnotation {}
```

**Typical results:** ~50-200ms for 1 million checks

### Optimization Strategies

**1. Cache annotation lookups:**

```java
class AnnotationCache {
    private static final Map<Class<?>, Boolean> cache = new ConcurrentHashMap<>();
    
    public static boolean hasAnnotation(Class<?> clazz, 
                                       Class<? extends Annotation> annotation) {
        return cache.computeIfAbsent(clazz, 
            c -> c.isAnnotationPresent(annotation));
    }
}
```

**2. Use SOURCE retention when possible:**

```java
// Better: No runtime overhead
@Retention(SOURCE)
@interface CompileTimeCheck {}

// Worse: Runtime overhead
@Retention(RUNTIME)
@interface RuntimeCheck {}
```

**3. Batch annotation processing:**

```java
// Instead of multiple lookups
if (field.isAnnotationPresent(A.class)) { }
if (field.isAnnotationPresent(B.class)) { }
if (field.isAnnotationPresent(C.class)) { }

// Use single call
for (Annotation anno : field.getAnnotations()) {
    if (anno instanceof A) { }
    else if (anno instanceof B) { }
    else if (anno instanceof C) { }
}
```

### Memory Considerations

- RUNTIME annotations increase class file size
- Each annotation consumes constant pool entries
- Complex nested annotations multiply this effect

**Example:**

```java
// Minimal memory footprint
@interface Simple {
    String value();
}

// Larger memory footprint
@interface Complex {
    String a();
    String b();
    String c();
    int d();
    int e();
    boolean f();
    Class<?> g();
    Nested h();
    String[] i();
}
```

### When to Use Annotations vs. Alternatives

| Use Case | Annotations | XML | Code |
|----------|-------------|-----|------|
| Compile-time validation | ✓ | ✗ | ✓ |
| Runtime configuration | ✓ | ✓ | ✗ |
| Framework integration | ✓ | ✓ | ✗ |
| Simple metadata | ✓ | ✗ | ✗ |
| Complex configuration | ✗ | ✓ | ✓ |
| Performance-critical | ✗ | ✓ | ✓ |

### Best Practices

1. **Use SOURCE retention for compile-time checks** - no runtime overhead
2. **Cache annotation lookups** - avoid repeated reflection
3. **Keep annotations simple** - minimize memory footprint
4. **Prefer RUNTIME over CLASS** - CLASS has limited use cases
5. **Document annotation behavior** - especially for custom annotations
6. **Use @Inherited judiciously** - understand its limitations
7. **Consider annotation processors for code generation** - reduce runtime overhead

---

## Summary

Java annotations are a powerful metadata mechanism that enables:

- **Compile-time processing** via annotation processors
- **Runtime introspection** via reflection
- **Framework integration** (Spring, Hibernate, etc.)
- **Code generation** and automation
- **Declarative programming** patterns

**Key takeaways:**
- Annotations are metadata, not code
- Retention policy determines annotation lifecycle
- @Target restricts where annotations can be applied
- Reflection enables runtime annotation access
- Annotation processors enable compile-time code generation
- Understanding bytecode representation helps with debugging
- Performance considerations are important for production use

**Common patterns:**
- Dependency injection (@Inject, @Autowired)
- Validation (@Valid, @NotNull)
- ORM mapping (@Entity, @Table)
- AOP (@Transactional, @Async)
- Serialization (@JsonIgnore, @XmlElement)
- Testing (@Test, @Before)

This guide provides a comprehensive foundation for understanding and using Java annotations effectively in your projects.
