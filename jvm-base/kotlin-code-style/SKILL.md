---
name: "kotlin-code-style"
description: "Idiomatic Kotlin patterns that are often misused: scope functions, sealed classes, extension functions. Use when writing or reviewing Kotlin code."
---

# Kotlin Code Style Skill

## Scope and Boundaries

- Use this skill for idiomatic Kotlin patterns: scope functions, sealed classes, extensions, null safety, collections.
- Use `spring-boot-patterns` for Kotlin-specific Spring Boot patterns (controllers, services, DTOs, `@field:` annotations).
- Use `concurrency-patterns` for coroutines and Flow.
- Use `test-quality` (Kotest/MockK reference) for Kotlin test patterns.

## When to Use
- Writing new Kotlin code
- Reviewing Kotlin code for idiomatic style
- Migrating Java code to Kotlin

---

## Scope Functions

Choosing the wrong scope function is a common source of confusion:

| Function | Context | Returns | Use case |
|----------|---------|---------|----------|
| `let` | `it` | lambda result | Null checks, transformations |
| `run` | `this` | lambda result | Object config + compute result |
| `apply` | `this` | receiver | Object initialization |
| `also` | `it` | receiver | Side effects (logging) |
| `with` | `this` | lambda result | Operating on non-null object |
---

## Sealed Classes / When Expressions

Use sealed classes for exhaustive state modeling — compiler enforces all cases in `when`.

```kotlin
sealed class Result<out T> {
  data class Success<T>(val data: T) : Result<T>()
  data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
  data object Loading : Result<Nothing>()
}

// ✅ when is exhaustive — no else needed
fun handle(result: Result<User>) = when (result) {
  is Result.Success -> showUser(result.data)
  is Result.Error -> showError(result.message)
  Result.Loading -> showSpinner()
}
```

---

## Extension Functions

Prefer extension functions over utility classes — they stay close to the type they extend.

```kotlin
// ✅ Extension on domain type
fun String.toSlug(): String = lowercase().replace(" ", "-").replace(Regex("[^a-z0-9-]"), "")
fun LocalDate.isWeekend(): Boolean = dayOfWeek in listOf(DayOfWeek.SATURDAY, DayOfWeek.SUNDAY)

// ✅ Extension on collection
fun List<User>.activeOnly(): List<User> = filter { it.isActive }

// ❌ Utility class — harder to discover, breaks fluent chains
object StringUtils {
  fun toSlug(s: String): String = s.lowercase().replace(" ", "-")
}
```

---

## Data Classes

Use `data class` for value objects and DTOs — compiler generates `equals`, `hashCode`, `toString`, and `copy`.

```kotlin
// ✅ Immutable data class
data class CreateOrderRequest(
    val userId: Long,
    val items: List<OrderItem>,
    val shippingAddress: Address
)

// ✅ copy() for non-destructive updates
val updated = request.copy(shippingAddress = newAddress)

// ❌ Regular class for value objects — boilerplate and error-prone
class CreateOrderRequest(val userId: Long, val items: List<OrderItem>)
```

---

## Null Safety

Prefer safe operators over `!!`. Use `!!` only when a null value is a programming error that should crash immediately.

```kotlin
// ✅ Safe call + Elvis for defaults
val name = user?.profile?.displayName ?: "Anonymous"

// ✅ let for null-conditional execution
user?.email?.let { sendWelcome(it) }

// ✅ requireNotNull / checkNotNull for invariants
val config = requireNotNull(env["CONFIG_PATH"]) { "CONFIG_PATH must be set" }

// ❌ !! hides the real problem — avoid unless null is truly impossible
val name = user!!.profile!!.displayName
```

---

## `when` as Expression

Use `when` as an expression to eliminate `if-else` chains and ensure exhaustiveness with sealed types.

```kotlin
// ✅ when as expression — compiler enforces all branches return a value
val label = when (order.status) {
    OrderStatus.PENDING   -> "Awaiting payment"
    OrderStatus.CONFIRMED -> "Processing"
    OrderStatus.SHIPPED   -> "On the way"
    OrderStatus.CANCELLED -> "Cancelled"
}

// ✅ when with smart cast
fun describe(value: Any): String = when (value) {
    is Int    -> "integer: $value"
    is String -> "string of length ${value.length}"
    else      -> "unknown"
}
```

---

## Idiomatic Collections

Prefer Kotlin collection functions over manual loops.

```kotlin
// ✅ Declarative transformations
val activeEmails = users
    .filter { it.isActive }
    .map { it.email }

// ✅ groupBy for aggregation
val byStatus: Map<OrderStatus, List<Order>> = orders.groupBy { it.status }

// ✅ associate for building maps
val userById: Map<Long, User> = users.associate { it.id to it }

// ✅ firstOrNull instead of find + null check
val admin = users.firstOrNull { it.role == Role.ADMIN }

// ❌ Manual loop where a collection function suffices
val emails = mutableListOf<String>()
for (user in users) {
    if (user.isActive) emails.add(user.email)
}
```

---

## `object` and `companion object`

```kotlin
// ✅ object — singleton, stateless utilities, constants
object OrderDefaults {
    const val MAX_ITEMS = 100
    const val EXPIRY_DAYS = 30L
}

// ✅ companion object — factory methods and class-level constants
class Order private constructor(val id: Long, val items: List<OrderItem>) {
    companion object {
        fun create(items: List<OrderItem>): Order = Order(generateId(), items)
    }
}

// ❌ companion object with mutable state — use a proper service instead
companion object {
    var counter = 0  // shared mutable state, not thread-safe
}
```
