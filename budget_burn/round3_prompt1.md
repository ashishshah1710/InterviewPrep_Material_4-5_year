Producing hundreds of examples in a single answer is not feasible due to space constraints, but here's an **extremely detailed guide** covering every requested Spring Boot internal topic with focused examples, deep explanations, and references to code samples for practical insight. Use this as an exhaustive deep dive for advanced Spring Boot concepts.

---

# Spring Boot Internals: Detailed Guide

---

## 1. **Auto-Configuration**

### **What Is Auto-Configuration?**

Spring Boot simplifies application setup by dynamically configuring beans based on classpath entries, property settings, and environment variables. Auto-configuration is powered by `@EnableAutoConfiguration`, typically via `@SpringBootApplication`.

### **How It Works**

- Spring Boot scans `META-INF/spring.factories` (for Spring Boot <3.x) or `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (for Spring Boot 3.x).
- It locates and loads `AutoConfiguration` classes marked with `@Configuration` and `@Conditional...` annotations.

### **Example: Custom Auto-Configuration**

```java
// src/main/java/com/example/MyServiceAutoConfiguration.java
@Configuration
@ConditionalOnProperty(name = "my.service.enabled", havingValue = "true")
public class MyServiceAutoConfiguration {
    @Bean
    public MyService myService() {
        return new MyService();
    }
}

// src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyServiceAutoConfiguration
```

### **Built-In Example: DataSource**

- Auto-configuration (`DataSourceAutoConfiguration`) creates a DataSource if HikariCP is on the classpath and certain properties are set.

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(DataSource.class)
@Conditional(DatabaseCondition.class)
public class DataSourceAutoConfiguration { ... }
```

### **Override Mechanisms**

- Add your own beans with the same name/type to override auto-configured ones.
- Use `spring.autoconfigure.exclude` in `application.properties` to blacklist classes.

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## 2. **Starter Mechanism**

### **What Is a Starter?**

A starter is a dependency (usually with `spring-boot-starter-` prefix) that acts as an aggregation of transitive dependencies required for a specific feature (web, security, JPA, batch, etc.).

### **How Starters Work**

- Starter brings relevant libraries into your projectânot much code, mostly dependencies.
- Also brings necessary auto-configurations on the classpath.

### **Example: spring-boot-starter-web**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**Includes:**
- Spring Web MVC
- Jackson
- Tomcat (default embedded server)
- Logging (`logback`)
- Validation libraries

### **Custom Starter Example**

Suppose you want a starter for Redis:

```xml
<!-- pom.xml for your starter project -->
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
  </dependency>
</dependencies>
```

Additionally provide auto-config (see section 1).

---

## 3. **Actuator Endpoints**

### **What Are Actuator Endpoints?**

Actuator endpoints provide management and monitoring capabilities, exposing REST APIs to interact with your Spring Boot app.

### **Setup**

Add dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### **Default Endpoints**

- `/actuator/health`
- `/actuator/info`
- `/actuator/metrics`
- `/actuator/env`
- `/actuator/mappings`
- `/actuator/loggers`
- `/actuator/beans`

**Configure Exposure:**

```properties
management.endpoints.web.exposure.include=health,info,metrics,beans
management.endpoints.web.exposure.exclude=mappings,env
```

### **Securing Endpoints**

```java
@EnableWebSecurity
public class ActuatorSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
            .antMatchers("/actuator/health").permitAll()
            .antMatchers("/actuator/**").hasRole("ADMIN")
            .and()
            .httpBasic();
    }
}
```

### **Custom Endpoint Example**

```java
@Component
@Endpoint(id = "custom")
public class CustomEndpoint {
    @ReadOperation
    public String custom() {
        return "Hello Actuator";
    }
}
```

Now `/actuator/custom` will work.

---

## 4. **Profiles**

Profiles offer a way to segregate bean definitions and configuration by environment (dev, staging, prod).

### **Define Profile-Based Beans**

```java
@Configuration
public class DatasourceConfig {
    @Bean
    @Profile("dev")
    public DataSource devDataSource() { ... }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() { ... }
}
```

### **Activate Profiles**

- Via properties: `spring.profiles.active=dev`
- JVM: `-Dspring.profiles.active=prod`
- Programmatically:

  ```java
  SpringApplication app = new SpringApplication(MyApplication.class);
  app.setAdditionalProfiles("dev");
  app.run(args);
  ```

### **Profile-Specific Properties**

`application-dev.properties` and `application-prod.properties` will auto-apply based on active profile.

---

## 5. **Bean Lifecycle**

### **Bean Instantiation**

- Beans are created at startup by scanning classes annotated with `@Component`, `@Service`, `@Repository`, `@Controller`, or via `@Bean` methods.

### **Bean Lifecycle Hooks**

- **Initialization:**
  - `@PostConstruct` method
  - Implement `InitializingBean`
  - `initMethod` for `@Bean`
- **Destruction:**
  - `@PreDestroy` method
  - Implement `DisposableBean`
  - `destroyMethod` for `@Bean`

#### **Example**

```java
@Component
public class ExampleBean implements InitializingBean, DisposableBean {
    @PostConstruct
    public void postConstruct() { ... }

    @Override
    public void afterPropertiesSet() { ... }

    @PreDestroy
    public void preDestroy() { ... }

    @Override
    public void destroy() { ... }
}
```

#### **ApplicationContextAware Example**

Get the context:

```java
@Component
public class ContextAwareBean implements ApplicationContextAware {
    @Override
    public void setApplicationContext(ApplicationContext ctx) { ... }
}
```

---

## 6. **@Conditional Annotations**

Spring Boot heavily uses `@Conditional...` to control bean registration.

### **Common Conditional Annotations**

- `@ConditionalOnClass`
- `@ConditionalOnMissingBean`
- `@ConditionalOnBean`
- `@ConditionalOnProperty`
- `@ConditionalOnResource`
- `@ConditionalOnExpression`
- `@ConditionalOnWebApplication`
- `@ConditionalOnNotWebApplication`

#### **Example: ConditionalOnClass**

```java
@Configuration
@ConditionalOnClass(name = "com.example.ExternalLibrary")
public class ExternalLibAutoConfig {}
```

#### **Example: ConditionalOnMissingBean**

```java
@Bean
@ConditionalOnMissingBean(DataSource.class)
public DataSource defaultDataSource() { ... }
```

#### **Example: ConditionalOnProperty**

```java
@Bean
@ConditionalOnProperty(prefix = "app.feature", name = "enabled", havingValue = "true")
public FeatureBean featureBean() { ... }
```

#### **Custom Conditional**

```java
public class LinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata meta) {
        return ctx.getEnvironment().getProperty("os.name").contains("Linux");
    }
}
@Configuration
@Conditional(LinuxCondition.class)
public class LinuxConfig { ... }
```

---

## 7. **Embedded Servers**

Spring Boot embeds application servers like Tomcat, Jetty, or Undertow, offering out-of-the-box configuration.

### **Default Server**

- Tomcat for `spring-boot-starter-web`.

### **Switch Embedded Server**

```xml
<!-- Switch to Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
<!-- Remove Tomcat dependency (optional) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### **Customizing Server**

```properties
server.port=8081
server.servlet.context-path=/api
server.tomcat.max-threads=200
server.undertow.accesslog.enabled=true
```

### **Programmatic Configuration Example**

```java
@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> customizer() {
    return factory -> factory.setPort(9090);
}
```

---

## 8. **Spring Boot 3 Changes**

Spring Boot 3.x (and Spring Framework 6) introduces major updates:

### **Java Version**

- Requires Java 17+.

### **Jakarta Namespace**

- All imports switch from `javax.*` to `jakarta.*`.

### **Observability**

- Unified observability API (`io.micrometer`), replaces `actuator` metrics and traces.

### **Native Support**

- Native images with GraalVM via Spring AOT (Ahead-of-Time Compilation).

### **Enhanced Auto-Configuration Mechanism**

- `spring.factories` replaced by `AutoConfiguration.imports`.

### **Example: Auto-Configuration Metadata**

```java
// Simple auto-config class
@Configuration
public class MyAutoConfiguration { ... }

// In src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyAutoConfiguration
```

### **Miscellaneous**

- Removed deprecated stuff from SB2
- Supports virtual threads

---

## 9. **Hundreds of Code Examples**

Given space, below are focused representative examples for each section. For hundreds of code samples, refer to [Spring Boot Samples Repository](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-samples), [docs](https://docs.spring.io/spring-boot/docs/current/reference/html/), or local clone.

---

### **Auto-Configuration Example**

```java
@Configuration
@ConditionalOnResource(resources="classpath:/myfile.txt")
public class ResourceAutoConfig { ... }
```

---

### **Starter Example**

```xml
<!-- spring-boot-starter-data-jpa brings Hibernate, DataSource AutoConfig, etc. -->
```

---

### **Actuator Health Check Example**

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        if (checkStatus()) {
            return Health.up().withDetail("custom", "Available").build();
        }
        return Health.down().withDetail("custom", "Unavailable").build();
    }
}
```

---

### **Profile + Bean Example**

```java
@Profile("prod")
@Bean
public MyProdBean prodBean() { ... }

@Profile("dev")
@Bean
public MyDevBean devBean() { ... }
```

---

### **Bean Lifecycle Example**

```java
@Bean(initMethod = "customInit", destroyMethod = "customDestroy")
public MyBean bean() { ... }
```

---

### **Conditional Annotations Example**

```java
@Bean
@ConditionalOnExpression("${app.feature.enable:true}")
public FeatureBean featureBean() { ... }
```

---

### **Embedded Server Example**

```properties
server.port=9999
server.servlet.context-path=/myapp
```

### **Custom Embedded Server Configuration**

```java
@Bean
public WebServerFactoryCustomizer<UndertowServletWebServerFactory> undertowCustomizer() {
    return factory -> {
        factory.setPort(7070);
        factory.addBuilderCustomizers(builder -> builder.setServerOption(UndertowOptions.ENABLE_HTTP2, true));
    };
}
```

---

### **Spring Boot 3: Native Configuration**

```xml
<dependency>
    <groupId>org.springframework.experimental</groupId>
    <artifactId>spring-native</artifactId>
</dependency>
```

---

## **References & Further Learning**

- [Spring Boot External Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Boot GitHub Repository](https://github.com/spring-projects/spring-boot)
- [Spring Framework 6 Migration Guide](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-6-Migration-Guide)

---

## **Summary Table**

| Feature                | Key Annotation/API                      | Config Location                 | Example Given    |
|------------------------|-----------------------------------------|---------------------------------|------------------|
| Auto-Config            | `@Configuration`, `@Conditional*`       | spring.factories / AutoConfig   | â                |
| Starter Mechanism      | Maven dependency                        | pom.xml                         | â                |
| Actuator Endpoints     | `@Endpoint`, `@ReadOperation`           | application.properties          | â                |
| Profiles               | `@Profile`, `spring.profiles.active`    | application-{profile}.properties| â                |
| Bean Lifecycle         | `@PostConstruct`, `InitializingBean`    | Java code                       | â                |
| Conditional Annotations| `@ConditionalOn*`                       | Java                            | â                |
| Embedded Servers       | `Tomcat`, `Jetty`, `Undertow`           | application.properties          | â                |
| Spring Boot 3 Changes  | Java 17, jakarta.*, AOT, Observability  | pom.xml                         | â                |

---

**For hundreds of code examples, clone the [Spring Boot samples](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-samples) and the [actuator samples](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-actuator-autoconfigure).**

---

### **If you need an extremely advanced, internal topic with deep source code navigation (like building your own starter, writing custom conditions, full lifecycle tracing, etc.), request a specific area and I'll give even more code samples with deep dive explanations.**
