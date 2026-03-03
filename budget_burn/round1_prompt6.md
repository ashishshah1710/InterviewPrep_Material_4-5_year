Certainly! Creating a guide as comprehensive as youâve requested is a massive task, far exceeding a single reply limit. What I can do is **structure the complete guide** for you, and begin by detailing the first set of design patterns along with key principles and Spring examples. If you want the full content for all 23 patterns, weâll break it into partsâjust say âcontinueâ for the next section.

Below is **Part 1**: Foundations, SOLID Principles, Dependency Injection, Clean Architecture, and the first few GoF patterns, all with Java + Spring examples, explanations, and code.  

---

# Java Design Patterns Guide with Spring Examples

---

## Table of Contents

1. **Foundations**
   - SOLID Principles
   - Clean Architecture
   - Dependency Injection Internals (with Spring)

2. **Creational Patterns (GoF)**
   - Singleton
   - Factory Method
   - Abstract Factory
   - Builder
   - Prototype

3. **Structural Patterns (GoF)**
   - Adapter
   - Bridge
   - Composite
   - Decorator
   - Facade
   - Flyweight
   - Proxy

4. **Behavioral Patterns (GoF)**
   - Chain of Responsibility
   - Command
   - Interpreter
   - Iterator
   - Mediator
   - Memento
   - Observer
   - State
   - Strategy
   - Template Method
   - Visitor

---

## 1. SOLID Principles

### S: Single Responsibility Principle

**Definition:** A class should have only one reason to change.

**Spring Example (Service + Repository):**
```java
// Bad: Both business logic and DB code mixed
public class UserService {
    public void register(User user) {
        // Validate, send email, save user (all in one!)
    }
}

// Good: Separate responsibilities
@Service
public class UserService {
    private final UserRepository userRepository;
    // Constructor injection
    public UserService(UserRepository userRepository) { ... }
    public void register(User user) {
        // Business logic
        userRepository.save(user);
    }
}
@Repository
public interface UserRepository extends JpaRepository<User, Long> { }
```

---

### O: Open/Closed Principle

**Definition:** Software entities should be open for extension, but closed for modification.

**Spring Example (feature toggling strategies):**
```java
// Use interface and inject at runtime
public interface DiscountStrategy { double apply(double amount); }

@Component
public class ChristmasDiscount implements DiscountStrategy { ... }
@Component
public class RegularDiscount implements DiscountStrategy { ... }

@Service
public class OrderService {
    private final List<DiscountStrategy> strategies;
    public OrderService(List<DiscountStrategy> strategies) { ... }
    public double getDiscountedPrice(double amount) {
        for (DiscountStrategy strategy : strategies) {
            amount = strategy.apply(amount);
        }
        return amount;
    }
}
```

---

### L: Liskov Substitution Principle

**Definition:** Subtypes must be substitutable for their base types.

**Spring Example:**
```java
public interface NotificationService { void send(String msg); }
@Component
public class EmailNotification implements NotificationService { ... }
@Component
public class SmsNotification implements NotificationService { ... }
// Accept NotificationService in any Spring bean
```

---

### I: Interface Segregation Principle

**Definition:** Favor many client-specific interfaces over one general-purpose interface.

**Spring Example:**
```java
// Bad
public interface AllInOneRepository {
    void add(); void remove(); void update(); void backup();
}
// Good
public interface CrudRepository { void add(); void remove(); void update(); }
public interface BackupRepository { void backup(); }
```

---

### D: Dependency Inversion Principle

**Definition:** High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Spring Example:**
```java
public interface ReportGenerator { void generate(); }

@Component
public class PdfReportGenerator implements ReportGenerator { ... }

@Service
public class ReportingService {
    private final ReportGenerator reportGenerator;
    @Autowired
    public ReportingService(ReportGenerator reportGenerator) { ... }
}
```

---

## 2. Clean Architecture Principles

- **Layers:** Entities, Use Cases, Interface Adapters, Frameworks & Drivers.
- **Dependency Rule:** Inner layers are not aware of outer layers.

**Spring Example Project Structure:**
```
com.example
|
|---- domain         // Entities, interfaces (business rules)
|---- application    // Use cases (services, business logic)
|---- infrastructure // Data/JPA, messaging, external APIs
|---- web            // Controllers, DTOs
```

**Service Example:**
```java
// domain/User.java
public class User { ... }

// domain/UserRepository.java
public interface UserRepository { ... }

// application/UserService.java
@Service
public class UserService {
    private final UserRepository userRepository;
    ...
}

// infrastructure/JpaUserRepository.java
@Repository
public class JpaUserRepository implements UserRepository { ... }

// web/UserController.java
@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService userService;
    ...
}
```
*Notice beans are injected as interfaces, decoupling layers.*

---

## 3. Dependency Injection Internals (Spring)

- **DI Types:**
  - Constructor (preferred)
  - Setter
  - Field (not recommended for testability)

**How Spring manages beans:**
- Scans classes with annotations (`@Service`, `@Repository`, `@Component`, `@RestController`).
- Builds a dependency graph.
- Resolves dependencies (constructor/setter injection).

**Constructor Injection Example:**
```java
@Component
public class OrderService {
    private final PaymentService paymentService;
    @Autowired
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Spring Context Example (Manual):**
```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService orderService = ctx.getBean(OrderService.class);
```

---

## 4. Creational Patterns

### 4.1 Singleton

**Intent:** Ensure a class has only one instance and provide a global point of access.

**Classic Java:**
```java
public class ClassicSingleton {
    private static final ClassicSingleton INSTANCE = new ClassicSingleton();
    private ClassicSingleton() {}
    public static ClassicSingleton getInstance() { return INSTANCE; }
}
```

**Spring Singleton Scope (real-world):**
```java
@Service // or @Component
public class AppSettingsService {
    // One shared instance in context
}
```
**Spring internals:** All beans are singletons by default (unless configured otherwise):

```java
@Component
@Scope("singleton")
public class ReportService { ... }
```

**Testing Singleton behavior in Spring:**
```java
@Autowired private ReportService r1;
@Autowired private ReportService r2;
// r1 == r2 => true
```

---

### 4.2 Factory Method

**Intent:** Define an interface for creating an object, but let subclasses alter the type of objects that will be created.

**Java Example:**
```java
public abstract class Dialog {
    public void renderWindow() {
        Button okButton = createButton();
        okButton.render();
    }
    protected abstract Button createButton();
}

public class WindowsDialog extends Dialog {
    protected Button createButton() { return new WindowsButton(); }
}

public class WebDialog extends Dialog {
    protected Button createButton() { return new HtmlButton(); }
}
```

**Spring Example (using factories):**
```java
public interface Notification { void send(); }

public class EmailNotification implements Notification { ... }
public class SmsNotification implements Notification { ... }

@Component
public class NotificationFactory {
    public Notification createNotification(String type) {
        if ("EMAIL".equals(type)) return new EmailNotification();
        if ("SMS".equals(type)) return new SmsNotification();
        throw new IllegalArgumentException();
    }
}

@Service
public class AlertService {
    private final NotificationFactory notificationFactory;
    @Autowired
    public AlertService(NotificationFactory notificationFactory) { ... }
    public void alert(String type) {
        notificationFactory.createNotification(type).send();
    }
}
```
*In Spring Boot's `ObjectProvider.getIfAvailable()` and `@Bean` methods are also real-world factory methods.*

---

### 4.3 Abstract Factory

**Intent:** Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

**Java Example:**
```java
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}
public class WindowsFactory implements UIFactory {
    public Button createButton() { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}
public class MacFactory implements UIFactory {
    public Button createButton() { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}
```

**Spring Example (profiles for DB):**
```java
public interface DataSourceFactory { DataSource createDataSource(); }

@Profile("dev")
@Component
public class DevDataSourceFactory implements DataSourceFactory {
    public DataSource createDataSource() { /* dev DS */ }
}

@Profile("prod")
@Component
public class ProdDataSourceFactory implements DataSourceFactory {
    public DataSource createDataSource() { /* prod DS */ }
}
```

---

### 4.4 Builder

**Intent:** Separate the construction of a complex object from its representation.

**Java Example:**
```java
public class User {
    private String name;
    private String email;
    private int age;
    // Builder static class
    public static class Builder {
        private String name, email; private int age;
        public Builder setName(String name) { this.name = name; return this; }
        public Builder setEmail(String email) { this.email = email; return this; }
        public Builder setAge(int age) { this.age = age; return this; }
        public User build() { return new User(name, email, age); }
    }
    private User(String n, String e, int a) { ... }
}

// Usage
User user = new User.Builder().setName("Alice").setEmail("alice@test.com").setAge(30).build();
```

**Spring Example (RestTemplateBuilder):**
```java
@Autowired
public RestTemplateBean(RestTemplateBuilder builder) {
    this.restTemplate = builder.rootUri("https://api.example.com")
                              .basicAuthentication("user", "pass")
                              .build();
}
```

---

### 4.5 Prototype

**Intent:** Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype.

**Java Example:**
```java
public class Product implements Cloneable {
    private String name;
    public Product clone() throws CloneNotSupportedException {
        return (Product) super.clone();
    }
}
Product p1 = new Product("Book");
Product p2 = p1.clone();
```

**Spring 'prototype' scope (real-world):**
```java
@Component
@Scope("prototype")
public class Task { ... }
// Now each injection or ctx.getBean(Task.class) will create a new Task instance
```

---

## Continue?

This guide covers **foundation principles** and all **creational GoF patterns** with Java and Spring examples, design and code.

**Reply "continue" for the structural patterns section (Adapter, Decorator, Proxy, etc) with explanations and hundreds of code samples!**
