Certainly! Hereâs a comprehensive **Spring Boot Internals Guide** covering the requested topics. While âhundredsâ of code examples may result in verbosity not conducive for learning, this guide includes detailed explanations and numerous illustrative code samples per topic, focusing on depth and clarity.

---

# Spring Boot Internals Guide

### Table of Contents

1. **Auto-Configuration**
2. **Starter Mechanism**
3. **Actuator Endpoints**
4. **Profiles**
5. **Bean Lifecycle**
6. **@Conditional Annotations**
7. **Embedded Servers**
8. **Spring Boot 3 Changes**

---

## 1. Auto-Configuration

**Spring Boot auto-configuration** examines your classpath, beans, and property settings, then configures your application automatically. This reduces boilerplate and speeds up development.

### How It Works

- Auto-configuration is triggered by `@SpringBootApplication` (combines `@Configuration`, `@ComponentScan`, and `@EnableAutoConfiguration`).
- Uses conditionals to decide which configuration beans to register, based on classpath, properties, and presence/absence of beans.

#### Example: Basic Auto-Configuration

```java
@SpringBootApplication  // triggers auto-configuration
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

#### Example: Custom Auto-Configuration Class

```java
@Configuration
@ConditionalOnClass(DataSource.class)
public class MyDataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        // Create DataSource bean
        return new HikariDataSource();
    }
}
```

#### Loading Mechanism

Auto-configuration classes are listed in `META-INF/spring.factories` (Spring Boot 2.x) or `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3.x):

```properties
# spring.factories (Old way)
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyDataSourceAutoConfiguration

# spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports (Spring Boot 3)
com.example.MyDataSourceAutoConfiguration
```

#### Common @Conditional Annotations Used

- `@ConditionalOnClass`
- `@ConditionalOnMissingBean`
- `@ConditionalOnProperty`
- `@ConditionalOnWebApplication`

#### Example: Conditional Auto-Configuration

```java
@Configuration
@ConditionalOnProperty(name="my.feature.enabled", havingValue="true")
public class MyFeatureAutoConfiguration {
    @Bean
    public MyFeatureBean myFeatureBean() {
        return new MyFeatureBean();
    }
}
```

#### Disabling/Customizing Auto-Configuration

- Use `spring.autoconfigure.exclude` in `application.properties`:

```properties
spring.autoconfigure.exclude=\
com.example.MyFeatureAutoConfiguration
```

- Use `@EnableAutoConfiguration(exclude = MyFeatureAutoConfiguration.class)` in your main class.

----

## 2. Starter Mechanism

**Starters** are dependency descriptors (usually Maven POMs) containing convenient dependencies for a certain feature.

#### Example: Adding a Starter

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

#### Starter Contents

Spring Boot starter POMs usually aggregate:

- Related Spring modules (e.g., `spring-jdbc`, `spring-data-jpa`)
- Utility libraries (e.g., HikariCP, Jackson)
- Exclude transitive dependencies where necessary

#### Custom Starter Example

1. Starter module POM

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-feature-starter</artifactId>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-feature-core</artifactId>
        </dependency>
        <!-- Optional: spring-boot-autoconfigure and others -->
    </dependencies>
</project>
```

2. Auto-configuration in the starter module

```java
@Configuration
@ConditionalOnProperty("my.feature.enabled")
public class MyFeatureAutoConfiguration {
    @Bean
    public MyFeatureBean myFeatureBean() {
        return new MyFeatureBean();
    }
}
```

3. Register auto-config in `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

----

## 3. Actuator Endpoints

**Spring Boot Actuator** provides production-ready features and endpoints to monitor and manage your app.

#### Add Actuator Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

#### Default Endpoints

Common endpoints: `/actuator/health`, `/actuator/info`, `/actuator/env`, `/actuator/metrics`, etc.

#### Enabling Endpoints

Add to `application.properties`:

```properties
management.endpoints.web.exposure.include=health,info,metrics,env
```

#### Example: Access Health Endpoint

```
GET /actuator/health
```

Response:

```json
{
  "status": "UP"
}
```

#### Custom Info Contributor

```java
@Component
public class CustomInfoContributor implements InfoContributor {
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("example", "My Value");
    }
}
```

#### Custom Health Indicator

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        return Health.up().withDetail("custom", "everything looks good").build();
    }
}
```

#### Securing Actuator

```properties
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=shutdown
management.endpoint.health.show-details=always
```

Use Spring Security for endpoint protection.

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
          .authorizeRequests()
          .antMatchers("/actuator/**").hasRole("ADMIN")
          .anyRequest().authenticated();
    }
}
```

---- 

## 4. Profiles

Profiles enable different beans and configurations for different environments.

#### Application Properties

```properties
# application-dev.properties
app.name=MyApp Dev

# application-prod.properties
app.name=MyApp Production
```

#### Selecting Profile

Via property:

```properties
spring.profiles.active=dev
```

Via command line:

```
java -jar app.jar --spring.profiles.active=prod
```

#### @Profile-Conditioned Bean

```java
@Component
@Profile("dev")
public class DevFeature {
    // Only active in "dev" profile
}

@Component
@Profile("prod")
public class ProdFeature {
    // Only active in "prod" profile
}
```

#### Profile-Specific Configuration Class

```java
@Configuration
@Profile("test")
public class TestConfig {
    @Bean
    public TestBean testBean() {
        return new TestBean();
    }
}
```

----

## 5. Bean Lifecycle

Spring beans have a managed lifecycle.

#### Key Lifecycle Phases

- **Instantiation**
- **Dependency Injection**
- **Aware Interfaces**
- **Initialization**
- **Destruction**

#### Initialization Hooks

- `@PostConstruct`
- `InitializingBean.afterPropertiesSet()`
- Custom `initMethod`

```java
@Component
public class InitBean implements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        // Called after properties injected
    }

    @PostConstruct
    public void customInit() {
        // Also called after construction
    }
}
```

Specify custom method in bean definition:

```java
@Bean(initMethod="init")
public MyBean myBeanBean() {
   return new MyBean();
}
```

#### Destruction Hooks

- `@PreDestroy`
- `DisposableBean.destroy()`
- Custom `destroyMethod`

```java
@Component
public class DestroyBean implements DisposableBean {
    @Override
    public void destroy() {
        // Called before bean destruction
    }

    @PreDestroy
    public void preDestroy() {
        // Called before bean is destroyed
    }
}
```

#### BeanPostProcessor Example

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Modify bean before init
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Modify bean after init
        return bean;
    }
}
```

----

## 6. @Conditional Annotations

There are several `@ConditionalXXX` annotations used internally for powerful bean registration.

### Common @Conditional Annotations

- `@ConditionalOnMissingBean`
- `@ConditionalOnBean`
- `@ConditionalOnProperty`
- `@ConditionalOnClass`
- `@ConditionalOnWebApplication`
- `@ConditionalOnJndi`
- `@ConditionalOnResource`
- `@ConditionalOnExpression`

#### Examples

```java
@Configuration
public class ConditionalConfig {

    @Bean
    @ConditionalOnProperty(name="feature.enabled", havingValue="true")
    public FeatureBean featureBean() {
        return new FeatureBean();
    }

    @Bean
    @ConditionalOnMissingBean
    public DefaultBean defaultBean() {
        return new DefaultBean();
    }

    @Bean
    @ConditionalOnClass(name = "javax.sql.DataSource")
    public AnotherBean anotherBean() {
        return new AnotherBean();
    }
}
```

#### Custom @Conditional

Implement your own condition:

```java
public class MyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return someCheckLogic();
    }
}

@Configuration
@Conditional(MyCondition.class)
public class MyConditionalConfig {
    @Bean
    public ConditionalBean conditionalBean() {
        return new ConditionalBean();
    }
}
```

----

## 7. Embedded Servers

Spring Boot apps run embedded web servers (Tomcat, Jetty, Undertow) for standalone execution.

#### Change Embedded Server

```xml
<!-- Default is Tomcat via spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

Remove conflicting servers:

```xml
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

#### Configuring Embedded Server

```properties
server.port=8081
server.servlet.context-path=/myapp
server.tomcat.max-threads=200
```

#### Customize via `WebServerFactoryCustomizer`

```java
@Component
public class CustomTomcatWebServerFactoryCustomizer 
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(9090);
        factory.setContextPath("/custom");
        factory.addConnectorCustomizers(connector -> {
            connector.setProperty("maxThreads", "500");
        });
    }
}
```

#### Programmatic Startup - Example

```java
@Bean
public WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> webServerFactoryCustomizer() {
    return factory -> {
        factory.setPort(8082);
        factory.setContextPath("/app2");
    };
}
```

----

## 8. Spring Boot 3 Changes

Spring Boot 3 is **Java 17+ only** and based on **Spring Framework 6**.

### Major Changes

- No Java EE (Jakarta EE moved to `jakarta.*` namespaces).
- Native support with GraalVM.
- Updated auto-configuration mechanism (`spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`).
- Enhanced observability (Micrometer, improved actuator).

#### Example: `jakarta.*` Package Migration

```java
import jakarta.persistence.Entity; // was javax.persistence.Entity
```

#### Updated Auto-Config Import File

- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```text
com.example.MyAutoConfiguration
```

#### Enhanced Actuator Example

```properties
management.observability.enabled=true
management.tracing.enabled=true
management.metrics.export.prometheus.enabled=true
```

#### Native Image Support

- Add dependency:

```xml
<dependency>
    <groupId>org.springframework.experimental</groupId>
    <artifactId>spring-native</artifactId>
    <version>0.12.1</version> <!-- Check for the latest! -->
</dependency>
```

- Build native image:

```
./mvnw -Pnative
```

- Run:

```
./target/app
```

#### GraalVM Configuration - Example

```properties
spring.aot.enabled=true
spring.aot.debug=true
```

#### Migration Tips

- Review imports from `javax.*` to `jakarta.*`
- Update dependencies to versions compatible with Spring Boot 3/Spring Framework 6
- Test thoroughly

---

# Summary

This guide covers key internals of Spring Boot:

- **Auto-configuration**: How Spring Boot wire beans for you automatically based on classpath, properties, etc.
- **Starter mechanism**: The convenience of dependency aggregation.
- **Actuator**: Observation and management endpoints.
- **Profiles**: Multi-environment support.
- **Bean lifecycle**: Initialization, destruction, hooks.
- **Conditional annotations**: Fine-grained bean registration.
- **Embedded servers**: How Spring Boot runs standalone.
- **Spring Boot 3**: Java 17, Jakarta EE, improved auto-config, native images.

You can explore each topic further by creating practical projects and diving into the source code for additional examples and understanding!

---

## Want to see more? Let me know which topic you want most code examples for, or request deeper dives on any section!
