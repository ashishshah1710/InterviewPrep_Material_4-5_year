Certainly! Hereâs a detailed guide on *Spring Boot Interview Questions for Candidates with 4-5 Years Experience*, focusing on auto-configuration internals, starter mechanism, actuator, profiles, bean lifecycle, and @Conditional annotations with explanations and code samples. The content is concise but covers the topics deeply for interview preparation.

---

# Spring Boot Interview Guide (4-5 Years Experience)

---

## Table of Contents

1. **Spring Boot Auto-Configuration Internals**
2. **Spring Boot Starter Mechanism**
3. **Spring Boot Actuator**
4. **Spring Boot Profiles**
5. **Bean Lifecycle in Spring Boot**
6. **@Conditional Annotations**

---

## 1. Auto-Configuration Internals

### Q1: **What is Spring Boot Auto-Configuration and how does it work internally?**

**Answer:**  
Spring Boot Auto-Configuration reduces boilerplate code by providing sensible default configurations based on project dependencies and environment. Internally, Spring Boot uses `@EnableAutoConfiguration` (usually via `@SpringBootApplication`) and `META-INF/spring.factories`âlisting classes with `@Configuration` and `@Conditional` logic.

#### Internal Steps:
1. **Classpath Scanning:** Scans dependencies in classpath.
2. **spring.factories:** Finds auto-configuration classes.
3. **Condition Evaluation:** Uses `@Conditional*` annotations to decide whether a configuration should load.
4. **Bean Registration:** Registers beans with configuration classes.

#### Key Classes:
- `org.springframework.boot.autoconfigure.EnableAutoConfiguration`
- `org.springframework.boot.autoconfigure.AutoConfigurationImportSelector`
- `spring.factories` (in Boot JAR)

#### Code Example:  
Custom auto-configuration:

```java
// src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.demo.config.MyAutoConfiguration

// com/example/demo/config/MyAutoConfiguration.java
@Configuration
@ConditionalOnClass(MyService.class)
public class MyAutoConfiguration {
    
    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

---

### Q2: **How does Spring Boot determine which configuration to apply?**

**Answer:**  
Spring Boot uses conditional annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc.) in auto-configuration classes to check:
- Presence of certain classes.
- Absence/presence of beans.
- Config property values.

---

### Q3: **How can you exclude an auto-configuration?**

**Answer:**
Use `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` or `spring.autoconfigure.exclude` property in `application.properties`.

---

### Q4: **Explain `@ConditionalOnProperty` with an example?**

**Answer:**  
Activates configuration based on a property value.

```java
@Configuration
@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
public class FeatureConfig {
    @Bean
    public FeatureBean featureBean() {
        return new FeatureBean();
    }
}
```
In `application.properties`:
```
feature.enabled=true
```

---

### Q5: **Describe the bootstrapping process of `@SpringBootApplication`**

**Answer:**  
`@SpringBootApplication` combines `@Configuration`, `@EnableAutoConfiguration`, `@ComponentScan`. On startup:
1. Scans beans.
2. Applies auto-configurations (from `spring.factories`).
3. Applies conditional logic.
4. Registers beans.

---

## 2. Starter Mechanism

### Q1: **What are Spring Boot starters and how do they work?**

**Answer:**  
Starters are pre-configured dependency descriptors (`Maven POMs`) that bring all necessary libraries for a specific purpose (web, JPA, etc).

Example:  
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
This pulls web, Jackson, Tomcat and related dependencies.

---

### Q2: **Can you create your own starter? How?**

**Answer:**
Build a new library that depends on desired dependencies and auto-configuration classes. Add a `spring.factories`.

```xml
<!-- Your starter project POM -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
  <groupId>com.example</groupId>
  <artifactId>your-library</artifactId>
</dependency>
```

---

### Q3: **Why does Spring Boot use starters rather than just dependencies?**

**Answer:**  
Starters simplify configuration, ensure compatibility, and reduce boilerplate and version conflicts.

---

## 3. Spring Boot Actuator

### Q1: **What is Spring Boot Actuator? What endpoints does it provide?**

**Answer:**  
Actuator exposes production-ready endpoints for monitoring and managing applications: health, metrics, info, env, mappings, etc.

#### Main Endpoints:
- `/actuator/health`
- `/actuator/info`
- `/actuator/metrics`
- `/actuator/mappings`
- `/actuator/env`
- `/actuator/loggers`

---

### Q2: **How do you enable actuator endpoints?**

**Answer:**  
Add dependency:
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Configure in `application.properties`:
```properties
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always
```

---

### Q3: **How to add a custom actuator endpoint?**

**Answer:**

```java
@Component
@Endpoint(id = "custom")
public class CustomEndpoint {

    @ReadOperation
    public String custom() {
        return "Custom actuator endpoint";
    }
}
```
Access at `/actuator/custom`.

---

### Q4: **How to secure actuator endpoints?**

**Answer:**  
Using Spring Security:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
Configure:

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .authorizeRequests()
        .antMatchers("/actuator/health").permitAll()
        .antMatchers("/actuator/**").hasRole("ADMIN");
}
```

---

### Q5: **What are health indicators? How can you write a custom one?**

**Answer:**  
Health indicators display health status.

Custom indicator:
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // custom check
        return Health.up().withDetail("custom", "ok").build();
    }
}
```
Output appears in `/actuator/health`.

---

## 4. Profiles

### Q1: **What is the purpose of Spring Profiles?**

**Answer:**  
Profiles allow you to separate configuration for different environments (dev, test, prod, etc).

---

### Q2: **How to define beans for a specific profile?**

**Answer:**
```java
@Bean
@Profile("prod")
public DataSource prodDataSource() {
    // production datasource
}
```
Enable profile via:
- `spring.profiles.active=prod` (application.properties)
- `--spring.profiles.active=prod` (command line)

---

### Q3: **How to use properties files for profiles?**

**Answer:**  
- `application.properties` (default)
- `application-dev.properties`
- `application-prod.properties`

Spring Boot selects based on `spring.profiles.active`.

---

### Q4: **Can you activate multiple profiles at once?**

**Answer:**  
Yes, comma-separated:
```
spring.profiles.active=dev,featureX
```
Beans for both profiles will be loaded.

---

### Q5: **How to set a default profile?**

**Answer:**  
Set `spring.profiles.default` property.

---

## 5. Bean Lifecycle in Spring Boot

### Q1: **What is the Spring Bean lifecycle?**

**Answer:**  
Lifecycle:
1. Instantiated (constructor)
2. Properties injected
3. `BeanNameAware`, `BeanFactoryAware`, etc.
4. `@PostConstruct` or `afterPropertiesSet`
5. Ready for use
6. `@PreDestroy` or `destroy`
7. Garbage collected

---

### Q2: **What are `@PostConstruct` and `@PreDestroy`?**

**Answer:**  
- `@PostConstruct`: Runs after bean initialization.
- `@PreDestroy`: Runs before bean destruction.

```java
@Component
public class LifecycleBean {
    @PostConstruct
    public void init() {
        // initialization
    }
    @PreDestroy
    public void cleanup() {
        // destroy tasks
    }
}
```

---

### Q3: **How to customize bean initialization and destruction?**

**Answer:**  
- Implement `InitializingBean` and `DisposableBean`
- Specify `initMethod` and `destroyMethod` in `@Bean`

```java
@Bean(initMethod = "customInit", destroyMethod = "customDestroy")
public SomeBean someBean() {
    return new SomeBean();
}
```

---

### Q4: **How does Spring Boot manage bean scopes?**

**Answer:**  
Default is singleton. You can specify `@Scope("prototype")`, etc.

```java
@Component
@Scope("prototype")
public class PrototypeBean { ... }
```

---

### Q5: **What is the role of BeanPostProcessor?**

**Answer:**  
Allows you to modify bean instances before/after initialization. E.g., for custom logic or proxies.

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) { ... }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) { ... }
}
```

---

## 6. @Conditional Annotations

### Q1: **Explain `@Conditional` annotation and its usages.**

**Answer:**  
`@Conditional` enables/disable beans based on custom logic. Multiple built-in annotations:
- `@ConditionalOnClass`
- `@ConditionalOnMissingBean`
- `@ConditionalOnProperty`
- `@ConditionalOnResource`
etc.

---

### Q2: **Write a custom `@Conditional` annotation.**

**Answer:**  
```java
public class MyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata atm) {
        return System.getProperty("my.condition") != null;
    }
}

@Configuration
@Conditional(MyCondition.class)
public class MyConfig {
    @Bean
    public MyService myService() { ... }
}
```

---

### Q3: **Difference between `@ConditionalOnBean` and `@ConditionalOnMissingBean`?**

**Answer:**  
- `@ConditionalOnBean`: Bean is registered *only* if a bean of specified type exists.
- `@ConditionalOnMissingBean`: Bean is registered *only* if a bean is NOT found.

---

### Q4: **How does `@ConditionalOnExpression` work?**

**Answer:**  
Activates bean if SpEL expression evaluates to true.

```java
@ConditionalOnExpression("${feature.enabled:true}")
```

---

### Q5: **Can you combine multiple `@Conditional*` annotations?**

**Answer:**  
Yes. Beans are registered **only if all conditions are satisfied**.

```java
@Configuration
@ConditionalOnClass(SomeClass.class)
@ConditionalOnProperty(name = "enabled", havingValue = "true")
public class MultiConditionalConfig { ... }
```

---

# Sample Interview Scenarios & Coding Questions

---

## Scenario 1: **Custom Starter, Auto-Configuration with Property Conditional**

**Task:**  
Build a custom starter that enables a bean if a property is set.

### Steps:

**a) Starter POM:**
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
  <groupId>com.example</groupId>
  <artifactId>custom-lib</artifactId>
</dependency>
```

**b) Auto-Configuration Class:**
```java
@Configuration
@ConditionalOnProperty(name = "custom.feature.enabled", havingValue = "true")
public class CustomFeatureAutoConfig {
    @Bean
    public CustomFeature customFeature() { return new CustomFeature(); }
}
```

**c) spring.factories:**
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.starter.CustomFeatureAutoConfig
```

**d) Usage:**
Enable in `application.properties`:
```
custom.feature.enabled=true
```

---

## Scenario 2: **Profile-Based Bean Example**

```java
@Component
@Profile("test")
public class TestMailSender implements MailSender { ... }

@Component
@Profile("prod")
public class ProdMailSender implements MailSender { ... }
```
Switch profile via `spring.profiles.active=test`.

---

## Scenario 3: **Bean Lifecycle with Logging**

```java
@Component
public class LifecycleLogger implements InitializingBean, DisposableBean {
    @Override
    public void afterPropertiesSet() { System.out.println("Bean initialized"); }
    @Override
    public void destroy() { System.out.println("Bean destroyed"); }

    @PostConstruct
    public void postConstruct() { System.out.println("Post construct"); }
    @PreDestroy
    public void preDestroy() { System.out.println("Pre destroy"); }
}
```

---

## Scenario 4: **Custom Health Indicator (Actuator)**

```java
@Component
public class DiskSpaceHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        long freeDiskSpace = checkDiskSpace();
        if (freeDiskSpace > 100 * 1024 * 1024) // 100MB
            return Health.up().withDetail("freeSpace", freeDiskSpace).build();
        else
            return Health.down().withDetail("freeSpace", freeDiskSpace).build();
    }
}
```

---

# Conclusion

For **4-5 years experienced Spring Boot developer interviews**, focus on:

- **Auto-configuration**: Internals, conditionals, customization, exclusion.
- **Starter mechanism**: Creating starters, dependencies, purpose.
- **Actuator**: Endpoints, custom endpoints, security, health indicators.
- **Profiles**: Configuration separation, bean loading, activation.
- **Bean Lifecycle**: Initialization, destruction, custom methods, post-processors.
- **@Conditional annotations**: Built-in and custom conditions, practical examples.

Mastering these concepts, backed with code examples and scenarios, ensures interview readiness and effective hands-on skills.

---

*(If you require expanded explanations or a deeper dive into any topicâincluding more code samples, real-life interview Q&A, or mock questionsâlet me know!)*
