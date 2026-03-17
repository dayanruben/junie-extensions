---
name: "validation-patterns"
description: "Bean Validation (Jakarta Validation) patterns with Spring Boot. Use when user asks about @Valid, @Validated, custom validators, validation groups, or handling MethodArgumentNotValidException."
---

# Validation Patterns Skill

Best practices for input validation in Spring Boot using Jakarta Bean Validation.

## Scope and Version Notes

- This is the canonical skill for Bean Validation guidance in Spring Boot projects.
- Examples are simplified pattern fragments; adapt imports, exception payloads, and framework-specific wiring to the current project.
- For authentication/authorization and security filters, defer to `security-audit`. For controller/service structure, defer to `spring-boot-patterns`.

## When to Use
- User adds validation annotations (`@NotNull`, `@Size`, etc.)
- Questions about `@Valid` vs `@Validated`
- Custom constraint implementation
- Validation groups for create/update scenarios
- Service-layer validation (not just controller)

---

## Basic Annotations

```java
// ✅ GOOD: Request DTO with standard constraints
public record CreateUserRequest(

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2–100 characters")
    String name,

    @NotBlank
    @Email(message = "Must be a valid email address")
    String email,

    @NotNull
    @Min(value = 18, message = "Must be at least 18 years old")
    @Max(value = 120)
    Integer age,

    @NotNull
    @Positive(message = "Amount must be positive")
    BigDecimal amount,

    @NotNull
    @Future(message = "Delivery date must be in the future")
    LocalDate deliveryDate,

    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$", message = "Invalid phone number")
    String phone
) {}
```

**Kotlin — use `@field:` prefix:**
```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 100)
    val name: String,

    @field:Email
    @field:NotBlank
    val email: String,
)
```

---

## Controller Validation

```java
@RestController
@RequestMapping("/api/v1/users")
@Validated   // ← needed for @PathVariable / @RequestParam constraints
public class UserController {

    // @Valid triggers validation on request body
    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(userService.create(request));
    }

    // @Validated at class level + constraint on path variable
    @GetMapping("/{id}")
    public UserResponse getById(@PathVariable @Positive Long id) {
        return userService.findById(id);
    }
}
```

| Annotation | Where to use |
|---|---|
| `@Valid` | Method parameter (request body, nested objects) |
| `@Validated` | On class (enables method-level / param constraints) |
| `@Validated(Group.class)` | Validation groups (create vs update) |

---

## Error Handling

```java
// ✅ GOOD: Structured validation error response
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {

        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();

        return ResponseEntity.badRequest()
            .body(new ValidationErrorResponse("VALIDATION_ERROR", errors));
    }

    // For @Validated on path/query params
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ValidationErrorResponse> handleConstraintViolation(
            ConstraintViolationException ex) {

        List<FieldError> errors = ex.getConstraintViolations().stream()
            .map(cv -> new FieldError(
                cv.getPropertyPath().toString(),
                cv.getMessage()))
            .toList();

        return ResponseEntity.badRequest()
            .body(new ValidationErrorResponse("VALIDATION_ERROR", errors));
    }
}

public record ValidationErrorResponse(String code, List<FieldError> errors) {}
public record FieldError(String field, String message) {}
```

---

## Custom Validators

```java
// Step 1: Define annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email is already registered";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Step 2: Implement validator
@Component
@RequiredArgsConstructor
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) return true;  // Let @NotBlank handle null
        return !userRepository.existsByEmail(email);
    }
}

// Step 3: Use it
public record CreateUserRequest(
    @NotBlank
    @Email
    @UniqueEmail
    String email
) {}
```

---

## Validation Groups

Use groups when create and update have different validation requirements.

```java
// Define marker interfaces
public interface OnCreate {}
public interface OnUpdate {}

// Apply groups to DTO
public record UserRequest(

    @NotBlank(groups = OnCreate.class)   // Required on create only
    String password,

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    String name,

    @Null(groups = OnCreate.class)       // Must be absent on create
    @NotNull(groups = OnUpdate.class)    // Must be present on update
    Long id
) {}

// Use @Validated(Group.class) in controller
@PostMapping
public UserResponse create(@Validated(OnCreate.class) @RequestBody UserRequest request) {
    return userService.create(request);
}

@PutMapping("/{id}")
public UserResponse update(@PathVariable Long id,
                           @Validated(OnUpdate.class) @RequestBody UserRequest request) {
    return userService.update(id, request);
}
```

---

## Service-Layer Validation

```java
// ✅ GOOD: Validate in service for business rules
@Service
@Validated   // ← enables method validation via AOP proxy
@RequiredArgsConstructor
public class OrderService {

    // Validate method parameters
    public OrderResponse createOrder(@Valid CreateOrderRequest request) {
        // Bean Validation runs before method body
        validateBusinessRules(request);
        // ...
    }

    // Validate return value
    @Valid
    public OrderResponse findById(@Positive Long id) {
        return orderRepository.findById(id).orElseThrow();
    }

    private void validateBusinessRules(CreateOrderRequest request) {
        if (request.quantity() > getAvailableStock(request.productId())) {
            throw new BusinessException("INSUFFICIENT_STOCK",
                "Requested quantity exceeds available stock");
        }
    }
}
```

---

## Nested Object Validation

```java
// ✅ Use @Valid to cascade validation into nested objects
public record CreateOrderRequest(

    @NotNull
    @Valid                          // ← cascade into nested object
    AddressRequest shippingAddress,

    @NotEmpty
    @Valid                          // ← cascade into each list element
    List<OrderItemRequest> items
) {}

public record AddressRequest(
    @NotBlank String street,
    @NotBlank String city,
    @Pattern(regexp = "\\d{5}(-\\d{4})?") String zip
) {}
```

---

## Common Mistakes

```java
// ❌ BAD: Missing @Valid — validation silently skipped
@PostMapping
public ResponseEntity<UserResponse> create(@RequestBody CreateUserRequest request) {
    return ResponseEntity.ok(userService.create(request));
}

// ❌ BAD: @Validated on method instead of class for path params
@GetMapping("/{id}")
public UserResponse getById(@PathVariable @Positive Long id) { ... }
// This only works if @Validated is on the controller class

// ❌ BAD: Forgetting @field: in Kotlin
data class Request(
    @NotBlank val name: String  // Applies to constructor param, NOT the field!
)
// ✅ GOOD:
data class Request(
    @field:NotBlank val name: String
)
```

---

## Quick Reference

| Need | Solution |
|---|---|
| Request body validation | `@Valid @RequestBody` |
| Path/query param validation | `@Validated` on class + constraint on param |
| Custom constraint | `@Constraint(validatedBy = ...)` + `ConstraintValidator` |
| Different rules for create/update | Validation groups |
| Nested object | `@Valid` on the nested field |
| Service-layer validation | `@Validated` on service class |

---

## Related Skills
- `spring-boot-patterns` - Controller and exception handler patterns
- `security-audit` - Input sanitization and injection prevention
