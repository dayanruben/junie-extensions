---
name: "spring-boot-patterns"
description: "High-level Spring Boot architecture and API patterns. Use when creating controllers, services, DTO boundaries, exception handling, or project structure. For deep validation, JPA, security, and testing guidance, defer to the dedicated skills."
---

# Spring Boot Patterns Skill

Best practices and patterns for Spring Boot applications.

## Scope and Version Notes

- Treat this skill as the high-level guide for project structure, controllers, services, DTO boundaries, and exception handling.
- For deep or version-sensitive topics, defer to canonical skills:
  - `validation-patterns` for Bean Validation details
  - `jpa-patterns` for entity mapping, fetch plans, and transaction pitfalls (**lives in the `sql-database` extension**)
  - `webflux-patterns` for reactive controllers, WebClient, Mono/Flux, and blocking anti-patterns
  - `security-audit` for Spring Security and secure coding guidance
  - `spring-testing` for test slices, Boot test annotations, and Testcontainers
- Examples in this file are pattern fragments, not copy-paste-ready production code; adapt imports, beans, package names, and framework versions to the current project.

## When to Use
- User says "create controller" / "add service" / "Spring Boot help"
- Reviewing Spring Boot code
- Setting up new Spring Boot project structure

## Project Structure

For detailed guidance on package organization, module boundaries, and when to prefer package-by-feature vs package-by-layer, see `architecture-review`.

A common starting structure for small-to-medium Spring Boot services (package-by-layer):

```
src/main/java/com/example/myapp/
├── MyAppApplication.java          # @SpringBootApplication
├── config/                        # Configuration classes
├── controller/                    # REST controllers
├── service/                       # Business logic
├── repository/                    # Data access
├── model/                         # Entities
├── dto/                           # Request/response DTOs
├── exception/                     # Custom exceptions + handler
└── util/                          # Utilities
```

As the project grows, prefer **package-by-feature** to keep related code together (see `architecture-review` for trade-offs).

---

## Controller Patterns

### REST Controller Template
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor  // Lombok for constructor injection
public class UserController {

    private final UserService userService;

    @GetMapping
    public ResponseEntity<List<UserResponse>> getAll() {
        return ResponseEntity.ok(userService.findAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getById(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserResponse> create(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse created = userService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.getId())
            .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        return ResponseEntity.ok(userService.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Controller Best Practices

| Practice | Example |
|----------|---------|
| Versioned API | `/api/v1/users` |
| Plural nouns | `/users` not `/user` |
| Status codes | 200=OK, 201=Created, 204=NoContent, 404=NotFound |
| Validation | `@Valid` on request body — see `validation-patterns` skill |

### ❌ Anti-patterns
```java
// ❌ Business logic in controller
@PostMapping
public User create(@RequestBody User user) {
    user.setCreatedAt(LocalDateTime.now());  // Logic belongs in service
    return userRepository.save(user);         // Direct repo access
}

// ❌ Returning entity directly (exposes internals)
@GetMapping("/{id}")
public User getById(@PathVariable Long id) {
    return userRepository.findById(id).get();
}
```

---

## Service Patterns

### Service Interface + Implementation
```java
// Interface
public interface UserService {
    List<UserResponse> findAll();
    UserResponse findById(Long id);
    UserResponse create(CreateUserRequest request);
    UserResponse update(Long id, UpdateUserRequest request);
    void delete(Long id);
}

// Implementation
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)  // Default read-only
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    @Override
    public List<UserResponse> findAll() {
        return userRepository.findAll().stream()
            .map(userMapper::toResponse)
            .toList();
    }

    @Override
    public UserResponse findById(Long id) {
        return userRepository.findById(id)
            .map(userMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    @Override
    @Transactional  // Write transaction
    public UserResponse create(CreateUserRequest request) {
        User user = userMapper.toEntity(request);
        User saved = userRepository.save(user);
        return userMapper.toResponse(saved);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("User", id);
        }
        userRepository.deleteById(id);
    }
}
```

### Service Best Practices

- Interface + Impl for testability
- `@Transactional(readOnly = true)` at class level
- `@Transactional` for write methods
- Throw domain exceptions, not generic ones
- Use mappers (MapStruct) for entity ↔ DTO conversion

---

## Repository Patterns

### JPA Repository
```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query
    Optional<User> findByEmail(String email);

    List<User> findByActiveTrue();

    // Custom query
    @Query("SELECT u FROM User u WHERE u.department.id = :deptId")
    List<User> findByDepartmentId(@Param("deptId") Long departmentId);

    // Native query (use sparingly)
    @Query(value = "SELECT * FROM users WHERE created_at > :date",
           nativeQuery = true)
    List<User> findRecentUsers(@Param("date") LocalDate date);

    // Exists check (more efficient than findBy)
    boolean existsByEmail(String email);

    // Count
    long countByActiveTrue();
}
```

### Repository Best Practices

- Prefer this section for repository boundaries only; move to `jpa-patterns` for fetch strategy, N+1, relationship tuning, and transaction edge cases.
- Use derived queries when possible
- `Optional` for single results
- `existsBy` instead of `findBy` for existence checks
- Avoid native queries unless necessary
- Use `@EntityGraph` for fetch optimization

---

## DTO Patterns

### Request/Response DTOs
```java
// Request DTO with validation (full validation guidance lives in validation-patterns)
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    String name,

    @NotBlank
    @Email(message = "Invalid email format")
    String email,

    @NotNull
    @Min(18)
    Integer age
) {}

// Response DTO
public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {}
```

### MapStruct Mapper
```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    UserResponse toResponse(User entity);

    List<UserResponse> toResponseList(List<User> entities);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    User toEntity(CreateUserRequest request);
}
```

---

## Skill Boundaries

- Prefer `validation-patterns` when changing constraints, validation groups, nested validation, or error payloads.
- Prefer `jpa-patterns` (in `sql-database` extension) when modifying entities, relationships, fetch mode, transaction scope, or Hibernate tuning.
- Prefer `webflux-patterns` when working with reactive controllers, WebClient, Mono/Flux chains, SSE, or mixing blocking/reactive code.
- Prefer `security-audit` when touching authentication, authorization, secrets, CSRF, CORS, headers, or security filters.
- Prefer `spring-testing` when adding or reviewing Spring tests.

---

## Exception Handling

### Custom Exceptions
```java
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String resource, Long id) {
        super(String.format("%s not found with id: %d", resource, id));
    }
}

public class BusinessException extends RuntimeException {

    private final String code;

    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }
}
```

### Global Exception Handler
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    // For structured validation error handling, see validation-patterns skill
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_ERROR", errors.toString()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}

public record ErrorResponse(String code, String message) {}
```

---

## Configuration Patterns

### Application Properties
```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # Never 'create' in production!
    show-sql: false

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400000
```

### Configuration Properties Class
```java
// ✅ Record-based (Spring Boot 3+) — immutable, no getters/setters needed
@ConfigurationProperties(prefix = "app.jwt")
@Validated
public record JwtProperties(
    @NotBlank String secret,
    @Min(60000) long expiration
) {}
```

### Profile-Specific Configuration
```
src/main/resources/
├── application.yml           # Common config
├── application-dev.yml       # Development
├── application-test.yml      # Testing
└── application-prod.yml      # Production
```

---

## Common Annotations Quick Reference

| Annotation | Purpose |
|------------|---------|
| `@RestController` | REST controller (combines @Controller + @ResponseBody) |
| `@Service` | Business logic component |
| `@Repository` | Data access component |
| `@Configuration` | Configuration class |
| `@RequiredArgsConstructor` | Lombok: constructor injection |
| `@Transactional` | Transaction management |
| `@Valid` | Trigger validation |
| `@ConfigurationProperties` | Bind properties to class |
| `@Profile("dev")` | Profile-specific bean |
| `@Scheduled` | Scheduled tasks |

## Documentation with Swagger / OpenAPI
Using `springdoc-openapi` for interactive API documentation.

### Configuration
```kotlin
// build.gradle.kts
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.+")  // use latest 2.x
```

### Accessing UI
- Swagger UI: `http://localhost:8080/swagger-ui.html`
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`

### Enhancing Documentation
```java
@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "User Management", description = "Operations related to users")
public class UserController {

    @Operation(summary = "Get user by ID", description = "Returns user details if found")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "User found"),
        @ApiResponse(responseCode = "404", description = "User not found")
    })
    @GetMapping("/{id}")
    public UserResponse getById(@PathVariable Long id) { ... }
}
```

---

## Quick Reference Card

| Layer | Responsibility | Annotations |
|-------|---------------|-------------|
| Controller | HTTP handling, validation | `@RestController`, `@Valid` |
| Service | Business logic, transactions | `@Service`, `@Transactional` |
| Repository | Data access | `@Repository`, extends `JpaRepository` |
| DTO | Data transfer | Records with validation annotations |
| Config | Configuration | `@Configuration`, `@ConfigurationProperties` |
| Exception | Error handling | `@RestControllerAdvice` |

---

## Kotlin Spring Boot Patterns

### Controller (Kotlin)
```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(private val userService: UserService) {  // constructor injection, no @RequiredArgsConstructor

    @GetMapping("/{id}")
    fun getById(@PathVariable id: Long): UserResponse = userService.findById(id)

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse =
        userService.create(request)

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun delete(@PathVariable id: Long) = userService.delete(id)
}
```

### Service (Kotlin)
```kotlin
@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository,
    private val userMapper: UserMapper,
) {
    fun findById(id: Long): UserResponse =
        userRepository.findById(id)
            .map(userMapper::toResponse)
            .orElseThrow { ResourceNotFoundException("User", id) }

    @Transactional
    fun create(request: CreateUserRequest): UserResponse {
        val user = userMapper.toEntity(request)
        return userMapper.toResponse(userRepository.save(user))
    }
}
```

### DTOs as data classes
```kotlin
// Request DTO
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")  // Note: @field: prefix needed in Kotlin
    @field:Size(min = 2, max = 100)
    val name: String,

    @field:Email(message = "Invalid email format")
    @field:NotBlank
    val email: String,

    @field:Min(18)
    val age: Int,
)

// Response DTO — immutable data class
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val createdAt: LocalDateTime,
)
```

### Configuration Properties (Kotlin)
```kotlin
// ✅ Immutable data class with constructor binding (Spring Boot 3+)
@ConfigurationProperties(prefix = "app.jwt")
@Validated
data class JwtProperties(
    @field:NotBlank
    val secret: String,

    @field:Min(60000)
    val expiration: Long = 86400000,
)
```

### Coroutine-based async (Spring WebFlux / suspend)

For reactive Kotlin patterns with WebFlux (`suspend`, `Mono`, `Flux`, coroutines), see the `webflux-patterns` skill.

### Kotlin-specific Anti-patterns
```kotlin
// ❌ Bad: nullable entity field without safe access
@Entity
class User {
    @Id @GeneratedValue
    var id: Long? = null  // Use Long? carefully — prefer separate persistence model

    var name: String = ""  // ✅ Non-nullable with default
}

// ❌ Bad: lateinit abuse
@Service
class UserService {
    @Autowired
    lateinit var userRepository: UserRepository  // ❌ Use constructor injection instead
}

// ✅ Good: constructor injection (Spring auto-detects single constructor)
@Service
class UserService(private val userRepository: UserRepository)
```
