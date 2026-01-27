# Java Spring Boot Development Rubric

## Overview

This rubric provides reusable patterns, standards, and best practices for developing high-quality Java Spring Boot applications that are:
- Type-safe with proper use of generics and Spring type system
- Well-structured using Spring Boot conventions and patterns
- Properly tested with clear, maintainable test coverage (JUnit, Mockito)
- Securely designed with Spring Security and input validation
- Performant with appropriate database patterns and caching
- Easy to maintain and extend following SOLID principles

Use this document as a reference when writing new Spring Boot code, reviewing implementations, or establishing team standards.

---

## Table of Contents

1. [When to Apply This Rubric](#when-to-apply-this-rubric)
2. [Core Principles](#core-principles)
3. [Type System and Generics](#type-system-and-generics)
4. [Spring Boot Patterns](#spring-boot-patterns)
5. [Code Organization and Structure](#code-organization-and-structure)
6. [Error Handling and Exceptions](#error-handling-and-exceptions)
7. [Testing Standards](#testing-standards)
8. [Documentation Standards](#documentation-standards)
9. [Security Best Practices](#security-best-practices)
10. [Performance Considerations](#performance-considerations)
11. [Dependencies Management](#dependencies-management)
12. [Self-Assessment Checklist](#self-assessment-checklist)

---

## When to Apply This Rubric

### Apply Strictly When:
- Writing production Spring Boot applications
- Creating shared libraries or components used across projects
- Building applications with security or performance requirements
- Any code that processes user input or handles business logic
- Applications with authentication, payments, or sensitive data

### Core Expectations (Always):
- Proper use of Spring annotations and dependency injection
- Clean architecture with separation of concerns (Controller → Service → Repository)
- Comprehensive test coverage (unit and integration tests)
- JavaDoc on public APIs and complex logic
- Proper exception handling and logging
- No hardcoded configuration or credentials

### May Be Relaxed For:
- Quick prototypes or POCs (mark as such in README)
- Throwaway code in development environments
- Internal utilities with clear scope and single developer
- Third-party integrations when source is fixed

---

## Core Principles

| Principle | Description | Examples |
|-----------|-------------|----------|
| **Dependency Injection** | Use Spring's DI container; never use `new` for managed beans | `@Autowired`, `@Inject`, constructor injection (preferred) |
| **Separation of Concerns** | Each layer has a single responsibility | Controllers handle HTTP, Services handle business logic, Repositories handle data access |
| **SOLID Principles** | Follow Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion | Small focused classes, interface-based dependencies |
| **Configuration Over Convention** | Externalize configuration; make behavior configurable | Use `application.yml`, `@ConfigurationProperties`, environment variables |
| **Fail Fast** | Validate inputs early, throw specific exceptions for invalid states | Check preconditions at method start; use custom exceptions |
| **Defensive at Boundaries** | Validate all external inputs (HTTP requests, API responses, file content) | Use validation annotations, DTO validation, API response checking |
| **Immutability Where Possible** | Use immutable DTOs, records, and value objects | Records for simple data, `@Value` objects for domain models |
| **Clear Error Messages** | Exceptions and logs should be actionable by the next developer | Include context, actual vs expected values |
| **Testing at Multiple Levels** | Unit tests for logic, integration tests for database and Spring, e2e for workflows | Pyramid testing strategy |

---

## Type System and Generics

### Proper Use of Generics

Always use generics for collections and custom types. Never use raw types.

```java
// ❌ WRONG - Raw type, loses type information
List users = repository.findAll();
Map config = loadConfiguration();

// ✅ CORRECT - Proper generic types
List<User> users = repository.findAll();
Map<String, String> config = loadConfiguration();

// ✅ CORRECT - Wildcard types where appropriate
public List<? extends User> getActiveUsers() {
    return repository.findByStatus("ACTIVE");
}

public <T extends Entity> T save(T entity) {
    return repository.save(entity);
}
```

### Type-Safe Collections

```java
// ❌ WRONG - Unsafe cast
List items = repository.findAll();
Item item = (Item) items.get(0);

// ✅ CORRECT - Type-safe from the start
List<Item> items = repository.findAll();
Item item = items.get(0);  // No cast needed

// ✅ CORRECT - Generic methods
public <T> T findById(Long id, Class<T> type) {
    return repository.findById(id)
        .map(type::cast)
        .orElseThrow(() -> new EntityNotFoundException(
            f"${type.getSimpleName()} not found with id ${id}"
        ));
}
```

### Optional Usage

Use `Optional` to represent potentially absent values, never return null.

```java
// ❌ WRONG - Using null for absence
public User getUser(Long id) {
    return repository.findById(id).orElse(null);
}

// ✅ CORRECT - Use Optional
public Optional<User> getUser(Long id) {
    return repository.findById(id);
}

// ✅ CORRECT - Chain operations on Optional
User user = repository.findById(userId)
    .filter(u -> u.isActive())
    .orElseThrow(() -> new UserNotFoundException(
        f"Active user not found with id ${userId}"
    ));
```

---

## Spring Boot Patterns

### Dependency Injection

Always prefer constructor injection over field injection. It makes dependencies explicit and enables testing.

```java
// ❌ WRONG - Field injection, hard to test
@Service
public class UserService {
    @Autowired
    private UserRepository repository;

    @Autowired
    private EmailService emailService;
}

// ✅ CORRECT - Constructor injection (preferred)
@Service
public class UserService {
    private final UserRepository repository;
    private final EmailService emailService;

    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}

// ✅ ALSO CORRECT - Lombok reduces boilerplate
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository repository;
    private final EmailService emailService;
}
```

### Layered Architecture

Organize code in distinct layers with clear responsibilities:

```
com.example.myapp/
├── controller/          # HTTP endpoints, request/response handling
│   ├── UserController.java
│   └── dto/
│       ├── CreateUserRequest.java
│       └── UserResponse.java
├── service/            # Business logic
│   ├── UserService.java
│   └── EmailService.java
├── repository/         # Data access
│   ├── UserRepository.java
│   └── UserJpaRepository.java (extends JpaRepository)
├── entity/             # JPA entities
│   └── User.java
├── config/             # Spring configuration
│   ├── SecurityConfig.java
│   └── CacheConfig.java
├── exception/          # Custom exceptions
│   ├── UserNotFoundException.java
│   └── ValidationException.java
└── util/               # Utilities
    └── DateUtils.java
```

### REST Controllers

```java
// ✅ CORRECT - RESTful endpoint with proper status codes
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;

    // GET /api/v1/users/{id}
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(new UserResponse(user));
    }

    // POST /api/v1/users
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(request);
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(new UserResponse(user));
    }

    // PUT /api/v1/users/{id}
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        User user = userService.update(id, request);
        return ResponseEntity.ok(new UserResponse(user));
    }

    // DELETE /api/v1/users/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Service Layer

```java
// ✅ CORRECT - Service with clear business logic
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository repository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;

    public User create(CreateUserRequest request) {
        // Validate
        if (repository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(
                f"User with email ${request.getEmail()} already exists"
            );
        }

        // Create
        User user = new User(
            request.getName(),
            request.getEmail(),
            passwordEncoder.encode(request.getPassword())
        );
        user = repository.save(user);

        // Notify
        emailService.sendWelcomeEmail(user);

        return user;
    }

    public Optional<User> findById(Long id) {
        return repository.findById(id);
    }

    public void delete(Long id) {
        User user = repository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(
                f"User not found with id ${id}"
            ));

        repository.delete(user);
        emailService.sendGoodbyeEmail(user);
    }
}
```

### Data Access Layer

Use Spring Data JPA for most database operations:

```java
// ✅ CORRECT - Spring Data repository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByStatus(UserStatus status);
    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.createdAt >= ?1")
    List<User> findRecentUsers(LocalDateTime since);
}

// ✅ CORRECT - Custom repository for complex queries
@Repository
@RequiredArgsConstructor
public class UserRepositoryCustom {
    private final EntityManager entityManager;

    public List<User> searchUsers(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);

        List<Predicate> predicates = new ArrayList<>();

        if (criteria.getName() != null) {
            predicates.add(cb.like(
                root.get("name"),
                f"%${criteria.getName()}%"
            ));
        }

        if (criteria.getStatus() != null) {
            predicates.add(cb.equal(
                root.get("status"),
                criteria.getStatus()
            ));
        }

        query.where(cb.and(predicates.toArray(new Predicate[0])));
        return entityManager.createQuery(query).getResultList();
    }
}
```

### Configuration Properties

Externalize configuration using `@ConfigurationProperties`:

```java
// ✅ CORRECT - Type-safe configuration
@Configuration
@ConfigurationProperties(prefix = "app.email")
@Data
@NoArgsConstructor
public class EmailProperties {
    private String from;
    private String smtpHost;
    private int smtpPort;
    private String smtpUsername;
    private String smtpPassword;
    private int retryCount = 3;
    private int timeoutSeconds = 30;
}

// application.yml
/*
app:
  email:
    from: noreply@example.com
    smtpHost: smtp.gmail.com
    smtpPort: 587
    smtpUsername: ${SMTP_USERNAME}
    smtpPassword: ${SMTP_PASSWORD}
    retryCount: 5
*/
```

---

## Code Organization and Structure

### Package Structure

Organize packages by feature/domain, not by layer:

```
com.example.ecommerce/
├── user/
│   ├── controller/
│   │   └── UserController.java
│   ├── service/
│   │   └── UserService.java
│   ├── repository/
│   │   └── UserRepository.java
│   ├── entity/
│   │   └── User.java
│   └── dto/
│       ├── CreateUserRequest.java
│       └── UserResponse.java
├── product/
│   ├── controller/
│   ├── service/
│   ├── repository/
│   └── entity/
└── order/
    ├── controller/
    ├── service/
    └── entity/
```

### Entity Design

```java
// ✅ CORRECT - JPA entity with proper design
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true, length = 255)
    private String email;

    @Column(nullable = false)
    private String passwordHash;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private UserStatus status;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
}

// ✅ CORRECT - Record for simple data transfer
public record UserResponse(
    Long id,
    String name,
    String email,
    UserStatus status,
    LocalDateTime createdAt
) {
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(),
            user.getName(),
            user.getEmail(),
            user.getStatus(),
            user.getCreatedAt()
        );
    }
}
```

### Naming Conventions

```java
// ✅ CORRECT naming
public class UserService { }                    // Classes: PascalCase
public interface UserRepository { }             // Interfaces: PascalCase
private static final int MAX_RETRIES = 3;     // Constants: UPPER_CASE
private String userName;                       // Variables: camelCase
public void processUser() { }                  // Methods: camelCase
@Component("userProcessor")                   // Bean names: camelCase

// ❌ WRONG naming
public class user_service { }                  // Should be UserService
private static final int max_retries = 3;    // Should be MAX_RETRIES
```

---

## Error Handling and Exceptions

### Creating Custom Exceptions

```java
// ✅ CORRECT - Specific, informative exceptions
public class EntityNotFoundException extends RuntimeException {
    public EntityNotFoundException(String message) {
        super(message);
    }

    public EntityNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }

    public static EntityNotFoundException forEntity(String entity, Long id) {
        return new EntityNotFoundException(
            f"${entity} not found with id ${id}"
        );
    }
}

public class ValidationException extends RuntimeException {
    private final Map<String, String> errors;

    public ValidationException(String message, Map<String, String> errors) {
        super(message);
        this.errors = errors;
    }

    public Map<String, String> getErrors() {
        return errors;
    }
}

public class DuplicateEmailException extends RuntimeException {
    public DuplicateEmailException(String email) {
        super(f"User with email '${email}' already exists");
    }
}
```

### Global Exception Handling

```java
// ✅ CORRECT - Centralized exception handling
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            EntityNotFoundException e,
            HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            e.getMessage(),
            request.getRequestURI(),
            LocalDateTime.now()
        );
        log.warn("Entity not found: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException e,
            HttpServletRequest request) {
        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );

        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            request.getRequestURI(),
            LocalDateTime.now(),
            errors
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(
            Exception e,
            HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An unexpected error occurred",
            request.getRequestURI(),
            LocalDateTime.now()
        );
        log.error("Unexpected error", e);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(error);
    }
}

@Data
@AllArgsConstructor
public class ErrorResponse {
    private int status;
    private String message;
    private String path;
    private LocalDateTime timestamp;
    private Map<String, String> errors;

    public ErrorResponse(int status, String message, String path, LocalDateTime timestamp) {
        this.status = status;
        this.message = message;
        this.path = path;
        this.timestamp = timestamp;
        this.errors = new HashMap<>();
    }
}
```

### Service-Level Exception Handling

```java
// ✅ CORRECT - Specific exception handling in services
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {
    private final UserRepository repository;
    private final ExternalUserServiceClient client;

    public User create(CreateUserRequest request) {
        // Validate
        if (repository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }

        // Try external service with fallback
        try {
            User user = client.createRemoteUser(request);
            return repository.save(user);
        } catch (ExternalServiceException e) {
            log.warn("External service unavailable, creating locally", e);
            User user = new User(request.getName(), request.getEmail(), ...);
            return repository.save(user);
        }
    }
}
```

---

## Testing Standards

### Unit Testing with JUnit 5

```java
// ✅ CORRECT - JUnit 5 test structure
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService service;

    @Test
    @DisplayName("Should create user successfully")
    void testCreateUserSuccess() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "John Doe",
            "john@example.com",
            "password123"
        );
        User expectedUser = new User(1L, "John Doe", "john@example.com", ...);

        when(repository.existsByEmail(request.getEmail()))
            .thenReturn(false);
        when(repository.save(any(User.class)))
            .thenReturn(expectedUser);

        // Act
        User result = service.create(request);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getName()).isEqualTo("John Doe");
        verify(emailService).sendWelcomeEmail(expectedUser);
    }

    @Test
    @DisplayName("Should throw exception for duplicate email")
    void testCreateUserDuplicateEmail() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "Jane Doe",
            "jane@example.com",
            "password123"
        );

        when(repository.existsByEmail(request.getEmail()))
            .thenReturn(true);

        // Act & Assert
        assertThrows(DuplicateEmailException.class, () -> {
            service.create(request);
        });

        verify(repository, never()).save(any());
        verify(emailService, never()).sendWelcomeEmail(any());
    }

    @ParameterizedTest
    @ValueSource(strings = {
        "",           // Empty
        " ",          // Whitespace
        "a",          // Too short
        "a".repeat(101)  // Too long
    })
    @DisplayName("Should reject invalid names")
    void testCreateUserInvalidName(String name) {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            name,
            "test@example.com",
            "password123"
        );

        // Act & Assert
        assertThrows(ValidationException.class, () -> {
            service.create(request);
        });
    }
}
```

### Integration Testing with Spring Boot Test

```java
// ✅ CORRECT - Spring Boot integration test
@SpringBootTest
@ActiveProfiles("test")
class UserControllerIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository repository;

    @Autowired
    private ObjectMapper objectMapper;

    @BeforeEach
    void setup() {
        repository.deleteAll();
    }

    @Test
    @DisplayName("Should create user via REST API")
    void testCreateUserViaApi() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "Alice",
            "alice@example.com",
            "password123"
        );

        // Act
        ResponseEntity<UserResponse> response = restTemplate.postForEntity(
            "/api/v1/users",
            request,
            UserResponse.class
        );

        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getName()).isEqualTo("Alice");

        List<User> users = repository.findAll();
        assertThat(users).hasSize(1);
        assertThat(users.get(0).getEmail()).isEqualTo("alice@example.com");
    }

    @Test
    @DisplayName("Should return 400 for invalid request")
    void testCreateUserInvalidRequest() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "",  // Invalid: empty name
            "invalid-email",  // Invalid: not an email
            "short"  // Invalid: password too short
        );

        // Act
        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
            "/api/v1/users",
            request,
            ErrorResponse.class
        );

        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        assertThat(response.getBody().getErrors()).isNotEmpty();
    }
}
```

### Test Fixtures and Builders

```java
// ✅ CORRECT - Builder pattern for test data
public class UserTestBuilder {
    private Long id = 1L;
    private String name = "Test User";
    private String email = "test@example.com";
    private String passwordHash = "hashed_password";
    private UserStatus status = UserStatus.ACTIVE;

    public UserTestBuilder withId(Long id) {
        this.id = id;
        return this;
    }

    public UserTestBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserTestBuilder withStatus(UserStatus status) {
        this.status = status;
        return this;
    }

    public User build() {
        User user = new User();
        user.setId(id);
        user.setName(name);
        user.setEmail(email);
        user.setPasswordHash(passwordHash);
        user.setStatus(status);
        return user;
    }
}

// Usage in tests
User user = new UserTestBuilder()
    .withEmail("custom@example.com")
    .withStatus(UserStatus.INACTIVE)
    .build();
```

---

## Documentation Standards

### JavaDoc Standards

Document public APIs and complex logic only. Don't document obvious code.

```java
// ✅ CORRECT - Document public methods
/**
 * Create a new user account.
 *
 * @param request the user creation request containing name, email, and password
 * @return the newly created user
 * @throws DuplicateEmailException if a user with the same email already exists
 * @throws ValidationException if the request contains invalid data
 */
public User create(CreateUserRequest request) {
    // Implementation
}

// ✅ SIMPLE PUBLIC METHOD - One-liner is sufficient
/**
 * Find a user by their unique identifier.
 */
public Optional<User> findById(Long id) {
    return repository.findById(id);
}

// ❌ WRONG - Redundant documentation
/**
 * Gets the user ID.
 * @return the ID
 */
public Long getId() {
    return id;
}

// ✅ CORRECT - Document non-obvious behavior
/**
 * Search for users with advanced filtering.
 *
 * Results are cached for 5 minutes to reduce database load.
 * Cache is invalidated when any user is modified.
 *
 * @param criteria search criteria (partial matching on name, exact on status)
 * @return users matching the criteria, ordered by creation date descending
 */
public List<User> searchUsers(UserSearchCriteria criteria) {
    // Implementation
}
```

### Class-Level Documentation

```java
// ✅ CORRECT - Document complex classes
/**
 * Service for managing user accounts.
 *
 * Handles user creation, authentication, and profile management.
 * Integrates with external email service for notifications.
 *
 * Thread-safe: all methods can be called concurrently.
 */
@Service
public class UserService {
    // ...
}

// ✅ CORRECT - Document complex entities
/**
 * User account entity.
 *
 * Immutable after creation except for status updates.
 * Timestamps are automatically managed by the ORM.
 */
@Entity
public class User {
    // ...
}
```

---

## Security Best Practices

### Input Validation

Always validate input at the controller/API boundary:

```java
// ✅ CORRECT - Validation annotations
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        // Request is validated before this method is called
        return ResponseEntity.ok(new UserResponse(...));
    }
}

// ✅ CORRECT - Request DTO with validation
@Data
@NoArgsConstructor
public class CreateUserRequest {
    @NotBlank(message = "Name is required")
    @Size(min = 1, max = 100, message = "Name must be 1-100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be 8-100 characters")
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$",
        message = "Password must contain uppercase, lowercase, digit, and special character"
    )
    private String password;
}

// ✅ CORRECT - Custom validator for complex rules
@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    @Autowired
    private UserRepository repository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) {
            return true;
        }
        return !repository.existsByEmail(email);
    }
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email must be unique";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### Spring Security Configuration

```java
// ✅ CORRECT - Spring Security configuration
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors()
                .and()
            .csrf()
                .disable()
            .exceptionHandling()
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .and()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
            .authorizeRequests()
                .antMatchers("/", "/favicon.ico").permitAll()
                .antMatchers("/api/v1/auth/**").permitAll()
                .antMatchers("/api/v1/public/**").permitAll()
                .anyRequest().authenticated()
                .and()
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### No Hardcoded Secrets

```java
// ❌ WRONG - Hardcoded secrets
public class ApiConfig {
    private static final String API_KEY = "sk-1234567890abcdef";
    private static final String DB_PASSWORD = "admin123";
}

// ✅ CORRECT - Load from environment
@Configuration
@RequiredArgsConstructor
public class ApiConfig {

    @Value("${app.api-key}")
    private String apiKey;

    @Value("${spring.datasource.password}")
    private String dbPassword;

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate template = new RestTemplate();
        template.getInterceptors().add((request, body, execution) -> {
            request.getHeaders().set("Authorization", "Bearer " + apiKey);
            return execution.execute(request, body);
        });
        return template;
    }
}

// application.yml - Never commit with real values!
/*
app:
  api-key: ${API_KEY}
spring:
  datasource:
    password: ${DB_PASSWORD}
*/
```

### SQL Injection Prevention

Always use parameterized queries. Never concatenate SQL strings.

```java
// ❌ VULNERABLE - SQL injection risk
@Repository
public class UserRepositoryUnsafe {

    @Query(nativeQuery = true,
        value = "SELECT * FROM users WHERE email = '" + email + "'")
    User findByEmailUnsafe(String email);
}

// ✅ SECURE - Parameterized query
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.email = ?1")
    Optional<User> findByEmail(String email);

    // Or simply use Spring Data derived query names
    Optional<User> findByEmail(String email);
}

// ✅ SECURE - Named parameters with @Query
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.email = :email AND u.status = :status")
    List<User> findByEmailAndStatus(
        @Param("email") String email,
        @Param("status") UserStatus status
    );
}
```

---

## Performance Considerations

### Database Query Optimization

```java
// ❌ WRONG - N+1 query problem
@Service
public class UserService {

    public List<UserResponse> getAllUsers() {
        List<User> users = repository.findAll();  // Query 1
        return users.stream().map(user -> {
            // Each iteration causes a query for orders
            List<Order> orders = user.getOrders();  // N queries
            return UserResponse.from(user, orders);
        }).collect(Collectors.toList());
    }
}

// ✅ CORRECT - Eager loading with JOIN FETCH
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT DISTINCT u FROM User u LEFT JOIN FETCH u.orders")
    List<User> findAllWithOrders();
}

// ✅ CORRECT - Using Projections for read-only queries
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT new com.example.dto.UserSummary(u.id, u.name, u.email) " +
           "FROM User u WHERE u.status = ?1")
    List<UserSummary> findSummariesByStatus(UserStatus status);
}
```

### Pagination

```java
// ✅ CORRECT - Implement pagination for large result sets
@Service
public class UserService {

    public Page<UserResponse> listUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by("createdAt").descending());

        Page<User> users = repository.findAll(pageable);
        return users.map(UserResponse::from);
    }
}

// ✅ CORRECT - Use in controller
@GetMapping("/users")
public ResponseEntity<Page<UserResponse>> listUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    Page<UserResponse> users = userService.listUsers(page, size);
    return ResponseEntity.ok(users);
}
```

### Caching

```java
// ✅ CORRECT - Strategic caching
@Service
@CacheConfig(cacheNames = "users")
public class UserService {

    @Cacheable(key = "#id")
    public Optional<User> findById(Long id) {
        return repository.findById(id);
    }

    @CachePut(key = "#user.id")
    public User update(User user) {
        return repository.save(user);
    }

    @CacheEvict(key = "#id")
    public void delete(Long id) {
        repository.deleteById(id);
    }

    @CacheEvict(allEntries = true)
    public void clearAllCache() {
        // Invalidate all user cache
    }
}

// application.yml - Configure cache
/*
spring:
  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 minutes
*/
```

### Batch Processing

```java
// ✅ CORRECT - Efficient batch inserts
@Service
public class UserImportService {

    private static final int BATCH_SIZE = 100;

    public void importUsers(List<CreateUserRequest> requests) {
        List<User> batch = new ArrayList<>();

        for (CreateUserRequest request : requests) {
            batch.add(new User(request.getName(), request.getEmail(), ...));

            if (batch.size() == BATCH_SIZE) {
                repository.saveAll(batch);
                batch.clear();
            }
        }

        if (!batch.isEmpty()) {
            repository.saveAll(batch);
        }
    }
}
```

---

## Dependencies Management

### Maven POM Configuration

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>myapp</artifactId>
    <version>1.0.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.7.0</version>
            <scope>runtime</scope>
        </dependency>

        <!-- Utilities -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.30</version>
            <scope>provided</scope>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-junit-jupiter</artifactId>
            <version>5.5.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Dependency Version Management

Always pin dependency versions:

```xml
<!-- ✅ CORRECT - Pinned versions -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.2.0</version>
</dependency>

<!-- ❌ WRONG - Loose versions -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>  <!-- Could be any version! -->
```

---

## Self-Assessment Checklist

Use this checklist when reviewing Spring Boot code (yours or others'):

### Spring Fundamentals (6 items)
- [ ] All Spring beans use constructor injection (no field injection)
- [ ] No manual `new` of managed beans (use DI)
- [ ] Proper use of `@Service`, `@Repository`, `@Controller` stereotypes
- [ ] `@Transactional` used only where needed (read vs read-write)
- [ ] Configuration externalized via `application.yml` or `@ConfigurationProperties`
- [ ] No hardcoded configuration values or secrets

### Type Safety (5 items)
- [ ] All generics properly typed (no raw types)
- [ ] `Optional` used for potentially null values (not null returns)
- [ ] No unnecessary type casting
- [ ] Enums used instead of string constants
- [ ] Type-safe collections throughout

### Layered Architecture (6 items)
- [ ] Clear separation: Controller → Service → Repository
- [ ] Controllers handle HTTP only, not business logic
- [ ] Services contain business logic, orchestrate repositories
- [ ] Repositories handle data access, use Spring Data where possible
- [ ] DTOs used for API boundaries (not entities exposed directly)
- [ ] Custom exceptions for domain-specific errors

### Error Handling (5 items)
- [ ] Custom exceptions extend appropriate base classes
- [ ] Global exception handler (`@RestControllerAdvice`) configured
- [ ] Error responses include status, message, and path
- [ ] No generic `Exception` catches, specific exceptions caught
- [ ] Logging at appropriate levels (warn for expected, error for unexpected)

### Testing (6 items)
- [ ] Unit tests for business logic with mocks
- [ ] Integration tests for Spring context and database
- [ ] Test fixtures or builders used to reduce duplication
- [ ] Both success and error cases tested
- [ ] Test names clearly describe what's being tested
- [ ] 70%+ code coverage on core logic

### Database Access (5 items)
- [ ] Spring Data JPA used for standard operations
- [ ] `@Query` used for complex queries with parameterization
- [ ] No N+1 query problems (use FETCH JOIN or projections)
- [ ] Pagination implemented for large result sets
- [ ] Proper use of database indexes and constraints

### Security (5 items)
- [ ] All external input validated at boundaries
- [ ] Spring Security configured for authentication/authorization
- [ ] No hardcoded passwords or API keys
- [ ] SQL injection prevented (parameterized queries)
- [ ] CORS configured if needed

### Documentation (4 items)
- [ ] Public methods have JavaDoc
- [ ] Complex algorithms documented with explanation of *why*
- [ ] No comments restating what code clearly shows
- [ ] Configuration and setup documented

### Performance (4 items)
- [ ] No obvious N+1 query problems
- [ ] Caching used for expensive operations
- [ ] Large data sets use pagination
- [ ] Appropriate data structures and algorithms

### Code Quality (5 items)
- [ ] SOLID principles followed
- [ ] Classes have single responsibility
- [ ] No duplication (DRY principle)
- [ ] Naming is clear and consistent
- [ ] Code is testable and loosely coupled

### Total: 52 verification items

---

## Quick Reference

### Essential Spring Boot Annotations

```java
// Dependency Injection
@Component, @Service, @Repository, @Controller, @RestController
@Autowired, @Inject, @RequiredArgsConstructor

// HTTP Mapping
@GetMapping, @PostMapping, @PutMapping, @DeleteMapping, @PatchMapping
@RequestParam, @PathVariable, @RequestBody, @ResponseBody

// Configuration
@Configuration, @ConfigurationProperties, @Bean, @Value
@EnableCaching, @EnableScheduling, @EnableAsync

// Data Access
@Entity, @Table, @Column, @Id, @GeneratedValue
@OneToMany, @ManyToOne, @ManyToMany, @JoinColumn

// Testing
@SpringBootTest, @WebMvcTest, @DataJpaTest, @ExtendWith
@Mock, @MockBean, @InjectMocks, @Spy

// Other
@Transactional, @Cacheable, @CacheEvict, @Async, @Scheduled
@Validated, @RestControllerAdvice, @ExceptionHandler
```

### Common Exception Hierarchy

```
Exception
├── RuntimeException
│   ├── IllegalArgumentException
│   ├── IllegalStateException
│   └── [Custom exceptions: UserNotFoundException, ValidationException, etc.]
└── Checked Exception (discouraged in Spring)
```

### Testing Pyramid

```
     /\
    /  \  E2E Tests (few)
   /────\
  /      \  Integration Tests (some)
 /────────\
/__________\ Unit Tests (many)
```

---

## Version History

| Version | Date       | Changes                                                    |
| ------- | ---------- | ---------------------------------------------------------- |
| 1.0     | 2026-01-27 | Initial rubric created for Java Spring Boot development   |

---

## Related Documents

- `STD-001-prompt-creation-rubric.md` — For prompt engineering
- `STD-002-python-development-rubric.md` — For Python development
- [Spring Boot Official Documentation](https://spring.io/projects/spring-boot)
- [Spring Data JPA Documentation](https://spring.io/projects/spring-data-jpa)
- [Spring Security Documentation](https://spring.io/projects/spring-security)
