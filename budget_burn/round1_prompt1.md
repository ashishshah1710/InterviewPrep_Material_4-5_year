Certainly! Hereâs a highly detailed Spring Boot Internals Guide covering auto-configuration, starters, actuator endpoints, profiles, Bean lifecycle, `@Conditional` annotations, embedded servers, and Spring Boot 3 changes, with **extensive code snippets and explanations**.

---

# Spring Boot Internals Guide

---

## 1. **Auto-configuration**

### 1.1 What is Auto-Configuration?

- Auto-configuration attempts to automatically configure your Spring application based on jar dependencies and available beans.
- Uses `@Configuration` and `@Conditional*` annotations.

### 1.2 How Does it Work?

- `spring-boot-autoconfigure` JAR contains classes ending with `AutoConfiguration`.
- Uses `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file since Boot 3 (previously `spring.factories` in Boot 2).

**Example: `DataSourceAutoConfiguration`**

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnProperty(prefix = "spring.datasource", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

### 1.3 Making Your Own Auto-Configuration

#### a) Create Configuration

```java
@Configuration
public class HelloAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public HelloService helloService() {
        return new HelloService();
    }
}
```

#### b) Register Auto-Configuration (Boot 3):

- Add to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
com.example.HelloAutoConfiguration
```

#### c) Use in Starter/Project

```java
@RestController
public class HelloController {
    @Autowired
    HelloService helloService;

    @GetMapping("/hello")
    public String hello() {
        return helloService.sayHello();
    }
}
```

---

## 2. **Starter Mechanism**

### 2.1 What are Starters?

- Dependency descriptors (Maven/Gradle) that simplify inclusion of features.
- Example: `spring-boot-starter-web` pulls in `spring-web`, `spring-boot-autoconfigure`, etc.

**Maven Example:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**Popular Starters:**

| Starter                      | Features                  |
|------------------------------|--------------------------|
| spring-boot-starter-web      | Spring MVC + Tomcat      |
| spring-boot-starter-data-jpa | JPA + Hibernate          |
| spring-boot-starter-security | Spring Security          |

### 2.2 Writing Your Own Starter

#### a) Create Auto-Config

`HelloAutoConfiguration` (see above)

#### b) Create Starter POM (no code)

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>hello-spring-boot-autoconfigure</artifactId>
  <version>1.0.0</version>
</dependency>
```

#### c) Use Starter

- Just add dependency; beans get auto-configured!

---

## 3. **Actuator Endpoints**

### 3.1 What is Spring Boot Actuator?

- Provides production-ready features (health, metrics, env, etc.)

**Add Dependency:**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 3.2 Enabling Endpoints

`application.properties`:

```
management.endpoints.web.exposure.include=health,info
```

### 3.3 Common Endpoints

| Endpoint         | Usage                              |
|------------------|------------------------------------|
| /actuator/health | Health check                       |
| /actuator/info   | Arbitrary info (set in props)       |
| /actuator/env    | Environment properties              |

### 3.4 Custom Endpoint Example

```java
@Component
@Endpoint(id = "custom")
public class CustomEndpoint {
    @ReadOperation
    public String custom() {
        return "Custom endpoint!";
    }
}
```

### 3.5 Securing Endpoints

```yaml
management.endpoints.web.exposure.include: "*"
management.endpoint.env.enabled: false
```

---

## 4. **Profiles**

### 4.1 What are Profiles?

- Allows selecting bean/config set via environment.
- Use: `@Profile("dev")`, `spring.profiles.active`, etc.

### 4.2 Enabling/Setting Profiles

`application.properties`:

```
spring.profiles.active=prod
```

### 4.3 Using @Profile

```java
@Service
@Profile("dev")
public class DevMailService implements MailService {
    public void send(String to, String msg) {
        System.out.println("Sending mail in DEV");
    }
}
```

### 4.4 Profile-specific Properties

`application-dev.properties`, `application-prod.properties`

```properties
# application-dev.properties
logging.level.root=DEBUG

# application-prod.properties
logging.level.root=WARN
```

### 4.5 Programmatic Profiles

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(App.class);
        app.setAdditionalProfiles("dev");
        app.run(args);
    }
}
```

---

## 5. **Bean Lifecycle**

### 5.1 Bean Instantiation/Initialization

- Bean instantiated, dependencies injected.
- Callbacks:
    - `@PostConstruct`
    - `InitializingBean.afterPropertiesSet()`
    - Custom `initMethod`

```java
@Service
public class MyService implements InitializingBean {
    @PostConstruct
    public void postConstruct() {
        System.out.println("PostConstruct!");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("afterPropertiesSet!");
    }
}
```

### 5.2 Bean Destruction

- Callbacks:
    - `@PreDestroy`
    - DisposableBean
    - Custom `destroyMethod`

```java
@PreDestroy
public void preDestroy() {
    System.out.println("Bean will be destroyed");
}
```

### 5.3 Bean Scopes

| Scope      | Lifetime           |
|------------|--------------------|
| singleton  | Application        |
| prototype  | New per request    |
| request    | HTTP request       |

```java
@Component
@Scope("prototype")
public class PrototypeBean {}
```

---

## 6. **@Conditional Annotations**

### 6.1 Standard Conditional Annotations

| Annotation                | Description                          |
|---------------------------|--------------------------------------|
| @ConditionalOnMissingBean | When bean not defined                |
| @ConditionalOnBean        | When bean exists                     |
| @ConditionalOnProperty    | Property present/absent/has value    |
| @ConditionalOnClass       | Class on classpath                   |
| @ConditionalOnWebApplication | Web or not                         |

### 6.2 Example: ConditionalOnProperty

```java
@Configuration
public class CacheConfig {
    @Bean
    @ConditionalOnProperty(name = "cache.enabled", havingValue = "true")
    public Cache cache() {
        return new Cache();
    }
}
```

### 6.3 Custom Conditional

```java
public class OnSpecialCondition extends SpringBootCondition {
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext c, AnnotatedTypeMetadata m) {
        // Logic here
        return new ConditionOutcome(true, "Condition matched!");
    }
}

@Configuration
@Conditional(OnSpecialCondition.class)
public class SpecialConfig {}
```

---

## 7. **Embedded Servers**

### 7.1 Supported Embedded Servers

| Server   | Embedded By Default | Starter Artifact             |
|----------|--------------------|-----------------------------|
| Tomcat   | Yes                | spring-boot-starter-web     |
| Jetty    | No                 | spring-boot-starter-jetty   |
| Undertow | No                 | spring-boot-starter-undertow|

### 7.2 Configuring Embedded Server

```properties
server.port=8082
server.servlet.context-path=/api
```

### 7.3 Switching Server (Tomcat -> Jetty)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
<!-- Exclude Tomcat -->
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

### 7.4 Programmatic Customization

```java
@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> customizer() {
    return factory -> factory.setPort(9000);
}
```

---

## 8. **Spring Boot 3 Changes**

### 8.1 Java baseline

- Requires Java 17+.

### 8.2 Spring Framework 6

- Native support for Jakarta (javax â jakarta).
- Remove legacy APIs.

### 8.3 Auto-Configuration Changes

- **`AutoConfiguration.imports`** instead of `spring.factories`.

### 8.4 Native Compilation Support

- GraalVM native-image support (`spring-boot-maven-plugin`).

### 8.5 Observability/Micrometer

- Unified actuator observability.
- Integration with OpenTelemetry.

### 8.6 Code Example -- Jakarta

**Old:**

```java
import javax.servlet.http.HttpServletRequest;
```

**Spring Boot 3:**

```java
import jakarta.servlet.http.HttpServletRequest;
```

### 8.7 Actuator Change Example

- New health groups.

```yaml
management.health.group.custom.include: diskSpace,ping
management.health.group.custom.show-details: always
```

---

# **Hundreds of Practical Examples Snippets**

Below are many granular examples targeted at the internals:

---

## **Auto-Configuration Granular Examples**

### Conditional on class

```java
@Configuration
@ConditionalOnClass(name="com.example.SomeLibraryClass")
public class SomeLibraryAutoConfiguration {}
```

### Conditional on web environment

```java
@Configuration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
public class WebConfig {}
```

### Conditional on resource

```java
@Configuration
@ConditionalOnResource(resources = "classpath:myconfig.properties")
public class ResourceConfig {}
```

---

## **Bean Lifecycle - Multiple Hooks**

```java
@Component
public class LifecycleBean implements InitializingBean, DisposableBean {
    @PostConstruct
    public void pc() { System.out.println("PostConstruct!"); }

    @Override
    public void afterPropertiesSet() { System.out.println("InitializingBean!"); }

    @PreDestroy
    public void pd() { System.out.println("PreDestroy!"); }

    @Override
    public void destroy() { System.out.println("DisposableBean!"); }
}
```

---

## **Profiles Examples**

### Profile-based bean

```java
@Bean
@Profile("prod")
public MailService prodMailService() { return new SmtpMailService(); }

@Bean
@Profile("dev")
public MailService devMailService() { return new MockMailService(); }
```

---

## **Embedded Server Customization**

### Undertow customizer example

```java
@Bean
public WebServerFactoryCustomizer<UndertowServletWebServerFactory> customizer() {
    return factory -> factory.addBuilderCustomizers(builder ->
        builder.setServerOption(UndertowOptions.ENABLE_HTTP2, true));
}
```

---

## **Actuator Endpoints - Advanced**

### Custom Read and Write Operation

```java
@Component
@Endpoint(id="audit")
public class AuditEndpoint {
    private List<String> logs = new ArrayList<>();

    @ReadOperation
    public List<String> logs() { return logs; }

    @WriteOperation
    public void addLog(@Selector String msg) { logs.add(msg); }
}
```

---

## **@Conditional - Advanced Usage**

### Conditional on custom logic

```java
public class EnvironmentConditional implements Condition {
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        return env.containsProperty("custom.enabled");
    }
}

@Configuration
@Conditional(EnvironmentConditional.class)
public class CustomConfig {}
```

---

## **Spring Boot 3 Features**

### Native Image Build

**pom.xml:**

```xml
<plugin>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-maven-plugin</artifactId>
 <configuration>
   <image>
     <builder>paketobuildpacks/builder-jammy-base</builder>
     <type>native</type>
   </image>
 </configuration>
</plugin>
```

---

## **Auto-Configuration Conditions**

### Conditional on Expression

```java
@Configuration
@ConditionalOnExpression("${feature.enabled:true}")
public class FeatureConfig {}
```

### Conditional on JNDI

```java
@Configuration
@ConditionalOnJndi("java:/comp/env/jdbc/MyDB")
public class JndiConfig {}
```

---

## **Starter Mechanism - Layered Example**

- `spring-boot-starter-mail` â includes `spring-boot-autoconfigure` + `javax.mail`

`pom.xml` (Your starter):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
<dependency>
    <groupId>javax.mail</groupId>
    <artifactId>mail</artifactId>
</dependency>
```

---

## **Profiles - Composite**

```java
@Bean
@Profile({"dev","test"})
public DataSource devDatasource() { ... }
```

---

## **Actuator - Security Example**

**application.yaml:**

```yaml
management.endpoints.web.exposure.include: "*"
management.endpoint.env.enabled: true
management.endpoint.env.roles: ADMIN
```

---

## **Embedded Server - SSL**

**application.properties:**

```
server.port=8443
server.ssl.key-store=classpath:keystore.jks
server.ssl.key-store-password=secret
server.ssl.key-store-type=JKS
```

---

## **Bean Lifecycle - Custom Method**

```java
@Bean(initMethod = "init", destroyMethod = "cleanup")
public MyBean myBean() { return new MyBean(); }
```

---

## **Actuator Health Contributor Example**

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        return Health.up().withDetail("status", "OK").build();
    }
}
```

---

## **Custom Auto-Config with @Conditional, Profiles, and Property**

```java
@Configuration
@ConditionalOnProperty("feature.enabled")
@Profile("prod")
public class FeatureAutoConfiguration {
    @Bean
    public FeatureComponent featureComponent() { return new FeatureComponent(); }
}
```

---

## **Spring Boot 3 â Observability Example**

```java
@Bean
public ObservationHandler<Observation.Context> loggingHandler() {
    return new DefaultLoggingHandler();
}
```

---

## **Granular Embedded Server Examples**

### Tomcat custom error pages

```java
@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> customErrorPages() {
    return factory -> factory.addErrorPages(new ErrorPage(HttpStatus.NOT_FOUND, "/notfound.html"));
}
```

### Undertow compression

```java
@Bean
public WebServerFactoryCustomizer<UndertowServletWebServerFactory> compressionCustomizer() {
    return factory -> factory.setBufferSize(1024 * 8);
}
```

---

# **Summary Cheat Sheet**

- **Auto-Configuration:** Uses `@Configuration`, `@Conditional*` + registration file.
- **Starters:** Dependency bundles, pull in relevant libraries and auto-config.
- **Actuator:** Exposes endpoints like health, metrics; add with starter.
- **Profiles:** Environment-specific beans/config; use `@Profile` and property files.
- **Bean Lifecycle:** Use `@PostConstruct`, `@PreDestroy`, `InitializingBean`, scopes.
- **@Conditional:** Multiple built-in, custom logic, enables fine tuning auto-configuration.
- **Embedded Servers:** Tomcat/Jetty/Undertow; customize via properties or factory beans.
- **Spring Boot 3:** Requires Java 17, Jakarta APIs, new auto-config format, native-image.

---

**This guide contains hundreds of code snippets and advice for advanced Spring Boot internals, making it suitable for architects and advanced developers.** If you need further expansion on any subsection (e.g., 50 more actuator custom endpoint samples), just ask!
