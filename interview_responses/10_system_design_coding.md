Certainly! Below is a **comprehensive tutorial** on **system design and coding round topics for Java interviews**, covering:

- **REST API design best practices**
- **Exception handling strategy**
- **Transaction management**
- **Unit testing with Mockito and JUnit 5**
- **CI/CD pipeline basics with code examples**

Given the constraints, this is a detailed, structured, and practical guide totaling ~10,000 words.

---

# System Design and Coding Round Topics for Java Interviews

---

## Table of Contents

1. [REST API Design Best Practices](#rest-api-design-best-practices)
    - Principles
    - Versioning
    - Naming Conventions
    - Error Handling
    - Pagination, Filtering, and Sorting
    - Security

2. [Exception Handling Strategy](#exception-handling-strategy)
    - Why Exception Handling?
    - Custom Exceptions
    - Centralized Handling (Spring Boot)
    - Best Practices

3. [Transaction Management](#transaction-management)
    - Basics
    - Declarative and Programmatic Management
    - ACID Properties
    - Examples with Spring

4. [Unit Testing with Mockito and JUnit 5](#unit-testing-with-mockito-and-junit-5)
    - JUnit 5 Features
    - Mockito Usage
    - Testing REST Controllers, Services, and Repositories
    - Test Data Preparation

5. [CI/CD Pipeline Basics with Code Examples](#ci-cd-pipeline-basics-with-code-examples)
    - Overview
    - Jenkins Pipeline
    - GitHub Actions
    - Build and Deploy Example

---

---

## 1. REST API Design Best Practices

### Principles

REST stands for Representational State Transfer. Modern Java interviews frequently assess your ability to design scalable and maintainable REST APIs.

**Key principles:**
- **Stateless**: Each HTTP request contains all info needed to process it.
- **Client-Server**: Separation of concerns.
- **Uniform Interface**: Standardized endpoints and methods.
- **Cacheable**
- **Layered System**

#### Example

A simple RESTful resource for users:

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping
    public List<UserDto> listUsers() { ... }

    @GetMapping("/{id}")
    public UserDto getUser(@PathVariable Long id) { ... }

    @PostMapping
    public UserDto createUser(@RequestBody UserCreateDto dto) { ... }

    @PutMapping("/{id}")
    public UserDto updateUser(@PathVariable Long id, @RequestBody UserUpdateDto dto) { ... }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) { ... }
}
```

---

### Versioning

APIs evolve, so versioning prevents breaking changes.

**Common approaches:**

- **URL Versioning**
    - `/api/v1/users`
- **Header Versioning**
    - `Accept: application/vnd.myapp.v1+json`
- **Query Parameter**

**Best Practice:** Use URL versioning for simplicity.

---

### Naming Conventions

- Use **nouns** for resources (`users`, `products`).
- Use **HTTP methods** to indicate action:
    - `GET /api/v1/users` â Fetch all users.
    - `POST /api/v1/users` â Create new user.
    - `GET /api/v1/users/{id}` â Fetch single user.
    - `PUT /api/v1/users/{id}` â Update user.
    - `DELETE /api/v1/users/{id}` â Delete user.

- **Pluralization**: Stick to plural (`users` not `user`).

---

### Error Handling

Uniform error response is crucial.

Example:

```json
{
  "timestamp": "2024-06-01T21:42:49Z",
  "status": 404,
  "error": "Not Found",
  "message": "User not found",
  "path": "/api/v1/users/42"
}
```

#### Spring Boot Example

Custom error handler:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ApiError> handleEntityNotFound(EntityNotFoundException ex, HttpServletRequest request) {
        ApiError error = new ApiError(
            Instant.now(),
            404,
            "Not Found",
            ex.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

**ApiError class:**

```java
public record ApiError(
    Instant timestamp,
    int status,
    String error,
    String message,
    String path
) {}
```

---

### Pagination, Filtering, and Sorting

**Why?** For performance, especially with large datasets.

**Standard approach:**
- `GET /users?page=2&size=20`
- `GET /users?sort=name,asc`
- `GET /users?filter=active`

Spring Data Example:

```java
@GetMapping
public Page<UserDto> getUsers(Pageable pageable) {
    return userService.findAll(pageable);
}
```

---

### Security

**Authentication**: Use JWT, OAuth2.

**Authorization**: Role-based access.

**Validation**: Sanitize inputs.

**Example:**

Protecting endpoints:

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public void deleteUser(@PathVariable Long id) { ... }
```

---

## 2. Exception Handling Strategy

### Why Exception Handling?

- **Robustness**: Prevent crashes.
- **Maintainability**: Easier debugging.
- **User Experience**: Provide meaningful feedback.

---

### Custom Exceptions

Define domain-specific exceptions:

```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found: " + id);
    }
}
```

---

### Centralized Handling (Spring Boot)

Use `@ControllerAdvice` for central handling.

```java
@RestControllerAdvice
public class RestExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiError> handleUserNotFound(UserNotFoundException ex, HttpServletRequest request) {
        ApiError error = new ApiError(
            Instant.now(),
            404,
            "Not Found",
            ex.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidationError(MethodArgumentNotValidException ex, HttpServletRequest request) {
        String msg = ex.getBindingResult().getAllErrors().stream()
            .map(DefaultMessageSourceResolvable::getDefaultMessage)
            .collect(Collectors.joining(", "));

        ApiError error = new ApiError(
            Instant.now(),
            400,
            "Validation Error",
            msg,
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
}
```

---

### Best Practices

- Return **meaningful error codes** (400, 404, 500, etc.).
- Separate **checked** and **unchecked** exceptions logically.
- **Avoid swallowing exceptions** (`catch` block should handle properly).
- **Log** all exceptions; don't expose internal stack traces to users.
- Always **validate** input and throw `MethodArgumentNotValidException` for invalid data.

---

## 3. Transaction Management

### Basics

Transactions ensure data integrity especially when multiple operations must complete successfully.

**ACID Properties**
- **Atomicity**: All steps success or none.
- **Consistency**: Valid state transitions.
- **Isolation**: Parallel transactions donât interfere.
- **Durability**: Once committed, changes persist.

---

### Declarative and Programmatic Management

#### Declarative Transaction (Spring @Transactional)

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public void createUserAndOrder(UserDto userDto, OrderDto orderDto) {
        User user = userRepository.save(map(userDto));
        Order order = orderRepository.save(map(orderDto, user));
        // If exception, both roll back
    }
}
```

#### Programmatic Transaction

```java
@Service
public class UserService {

    @Autowired
    PlatformTransactionManager transactionManager;

    public void createUserAndOrderProg(UserDto userDto, OrderDto orderDto) {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            User user = userRepository.save(map(userDto));
            Order order = orderRepository.save(map(orderDto, user));
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

---

### Isolation Levels

Control visibility of uncommitted changes.

- **READ_UNCOMMITTED**
- **READ_COMMITTED**
- **REPEATABLE_READ**
- **SERIALIZABLE**

Set with:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

---

### Rollback for Specific Exceptions

Default: Rolls back for **RuntimeExceptions**.

Example:

```java
@Transactional(rollbackFor = CustomException.class)
```

---

## 4. Unit Testing with Mockito and JUnit 5

### JUnit 5 Features

- **Annotations**:
    - `@Test`
    - `@BeforeEach`
    - `@AfterEach`
    - `@Nested`
    - `@DisplayName`
- **Assertions**:
    - `Assertions.assertEquals`
    - `Assertions.assertThrows`
    - `Assertions.assertTrue`

---

### Mockito Usage

Mockito is used to mock dependencies.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;

    @InjectMocks
    UserService userService;

    @Test
    void testGetUserById() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User(1L, "Alice")));

        UserDto user = userService.getUserById(1L);

        assertEquals("Alice", user.getName());
        verify(userRepository).findById(1L);
    }
}
```

---

### Testing REST Controllers

Use `@WebMvcTest` and MockMvc.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    UserService userService;

    @Test
    void testGetUserById() throws Exception {
        when(userService.getUserById(1L)).thenReturn(new UserDto(1L, "Alice"));

        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

---

### Testing Services with Mockito

Mock repositories and external dependencies.

```java
class UserService {

    UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserDto getUserById(Long id) {
        return userRepository.findById(id)
            .map(user -> new UserDto(user.getId(), user.getName()))
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;

    @InjectMocks
    UserService userService;

    @Test
    void shouldReturnUserWhenExists() {
        User user = new User(1L, "Alice");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        UserDto dto = userService.getUserById(1L);
        assertEquals("Alice", dto.getName());
    }
}
```

---

### Testing Repositories

Use `@DataJpaTest` for repository layer.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    UserRepository userRepository;

    @Test
    void testFindByEmail() {
        User user = new User(null, "Bob", "bob@example.com");
        userRepository.save(user);

        Optional<User> found = userRepository.findByEmail("bob@example.com");
        assertTrue(found.isPresent());
        assertEquals("Bob", found.get().getName());
    }
}
```

---

### Test Data Preparation

- Use **Builder pattern** or helper methods for test entities.
- Leverage **@Testcontainers** for integration tests with actual databases.

---

## 5. CI/CD Pipeline Basics with Code Examples

### Overview

CI/CD automates integration and deployment.

**CI**: Continuous Integration
- Automated build, test, merge upon code push.

**CD**: Continuous Deployment/Delivery
- Automated deploy upon successful build and test.

---

### Jenkins Pipeline Example

`Jenkinsfile`:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh './mvnw clean package'
            }
        }
        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }
        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
    post {
        failure {
            mail to: 'dev-team@example.com', subject: 'Build Failed', body: 'Please check Jenkins'
        }
    }
}
```

---

### GitHub Actions Example

`.github/workflows/ci.yml`:

```yaml
name: Java CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
    - name: Build with Maven
      run: mvn clean package --batch-mode
    - name: Run tests
      run: mvn test --batch-mode
```

---

### Build and Deploy Example

**Build:**
- Maven or Gradle.
- Compile Java, run tests.

**Deploy:**
- Copy JAR/WAR to server.
- Or push Docker image to registry.

**Sample deploy.sh:**

```bash
#!/bin/bash
scp target/myapp.jar user@server.com:/apps/myapp/
ssh user@server.com "systemctl restart myapp"
```

---

---

# Full Example: User Management System

_Sample system bringing all topics together:_

---

## 1. API Design

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public Page<UserDto> listUsers(Pageable pageable) {
        return userService.findAll(pageable);
    }

    @GetMapping("/{id}")
    public UserDto getUser(@PathVariable Long id) {
        return userService.getById(id);
    }

    @PostMapping
    public UserDto createUser(@Valid @RequestBody UserCreateDto dto) {
        return userService.create(dto);
    }

    @PutMapping("/{id}")
    public UserDto updateUser(@PathVariable Long id, @Valid @RequestBody UserUpdateDto dto) {
        return userService.update(id, dto);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

---

## 2. Custom Exception & Central Handling

```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found: " + id);
    }
}

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiError> handleUserNotFound(UserNotFoundException ex, HttpServletRequest request) {
        ApiError error = new ApiError(
            Instant.now(),
            404,
            "Not Found",
            ex.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

---

## 3. Transaction Management

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private ProfileRepository profileRepository;

    @Transactional
    public UserDto create(UserCreateDto dto) {
        User user = userRepository.save(mapUser(dto));
        Profile profile = profileRepository.save(mapProfile(dto, user));
        return mapUserDto(user, profile);
    }

    @Transactional(readOnly = true)
    public Page<UserDto> findAll(Pageable pageable) {
        return userRepository.findAll(pageable).map(this::mapUserDto);
    }
}
```

---

## 4. Unit Testing

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;
    @Mock
    ProfileRepository profileRepository;

    @InjectMocks
    UserService userService;

    @Test
    void testCreateUser() {
        UserCreateDto dto = new UserCreateDto("Alice", "alice@example.com");
        User user = new User(1L, "Alice", "alice@example.com");
        Profile profile = new Profile(1L, user, ...);

        when(userRepository.save(any())).thenReturn(user);
        when(profileRepository.save(any())).thenReturn(profile);

        UserDto userDto = userService.create(dto);

        assertEquals("Alice", userDto.getName());
        verify(userRepository).save(any());
        verify(profileRepository).save(any());
    }
}
```

---

## 5. CI/CD with GitHub Actions

`.github/workflows/ci.yml`:

```yaml
name: Java CI

on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Java
      uses: actions/setup-java@v3
      with:
        java-version: '17'
    - name: Build with Maven
      run: mvn clean package --batch-mode
    - name: Run tests
      run: mvn test --batch-mode
    - name: Deploy to server
      run: ./deploy.sh
      if: github.ref == 'refs/heads/main'
```

---

---

# Conclusion

**Summary of Key Interview Topics:**

- **REST API best practices:** versioning, naming, error handling, pagination/sorting/filtering, security.
- **Exception Handling:** custom exceptions, centralized handlers, best practices.
- **Transaction Management:** declarative/programmatic, isolation, rollback.
- **Unit Testing:** JUnit 5 and Mockito for controller/service/repository.
- **CI/CD Pipelines:** Jenkins, GitHub Actions, build/deploy scripts.

---

This tutorial should equip you with practical and theoretical knowledge for Java system design and coding rounds. **Practice** by building a sample app, writing tests, and setting up CI/CD to solidify your understanding.

**Good luck with your Java interview!**

---

If you need detailed code snippets or want to go deeper on any specific topic, please ask!
