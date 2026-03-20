---
name: "webflux-patterns"
description: "Spring WebFlux reactive programming patterns: pure reactive flows, Mono.defer() lazy errors, operator selection, parallel operations with Mono.zip(), WebClient, SSE, scheduler rules, and blocking anti-patterns. Use when user works with reactive Spring (WebFlux, R2DBC) or hits issues like .block() deadlocks, imperative code in reactive chains, stream not executing, or mixing blocking I/O with reactive."
---

# Spring WebFlux Patterns Skill

This skill provides comprehensive guidance for writing idiomatic, production-ready reactive code with Spring WebFlux in Java. It enforces pure reactive programming patterns and eliminates common anti-patterns that break reactive streams.

## Scope and Boundaries

- Use this skill for reactive controller design, WebClient, error handling, SSE, reactive security, and operator patterns.
- Use `spring-testing` → `webflux-testing.md` for testing reactive code (WebTestClient, StepVerifier).
- Use `spring-boot-patterns` for imperative (Servlet/MVC) controller patterns.
- Examples are pattern fragments — adapt imports and versions to the current project.

## When to Use
- User works with `spring-boot-starter-webflux`
- Developing or refactoring Spring WebFlux applications
- Working with Mono and Flux reactive types
- `.block()` deadlock / `IllegalStateException: block()/blockFirst()/blockLast() are blocking` errors
- Mixing blocking code with reactive (JDBC, legacy libs)
- Reviewing reactive code for anti-patterns
- SSE / streaming responses
- Reactive security configuration

---

## WebFlux vs MVC — When to Choose

| Factor | Spring MVC (Servlet) | Spring WebFlux (Reactive) |
|---|---|---|
| I/O model | Thread-per-request | Non-blocking, event loop |
| Blocking calls (JDBC, files) | Natural | Requires `Schedulers.boundedElastic()` |
| Concurrency under load | Limited by thread pool | High (few threads, many connections) |
| Learning curve | Low | Higher |
| Best for | CRUD services, teams new to reactive | High-concurrency I/O-bound services, streaming |

> **Rule of thumb**: If you're using Spring Data JPA + JDBC, stay with MVC. Switch to WebFlux + R2DBC when you need non-blocking end-to-end.

---

## Core Principles

### 1. Analysis Before Implementation
Before writing any code, analyze existing code, define an implementation plan, ask clarifying questions, and make explicit design decisions. See [references/BEFORE_CODING.md](references/BEFORE_CODING.md) for the complete workflow.

### 2. Pure Reactive Flow
Never use imperative constructs (`if`, `throw`, nested operations). Replace with reactive operators (`filter`, `flatMap`, `switchIfEmpty`, `Mono.error()`). All errors must be lazy using `Mono.defer()`. See [references/REACTIVE_PATTERNS.md](references/REACTIVE_PATTERNS.md) for detailed patterns.

### 3. No Literals
Never use string or number literals in code. Use constants for all values and enums for error messages. This ensures maintainability and type safety.

### 4. Helper Methods
Extract complex operations into single-responsibility helper methods. Main methods should read as high-level reactive flows.

### 5. Parallel Operations
Use `Mono.zip()` for independent parallel validations. Use `Flux.merge()` for independent parallel operations. Never serialize what can run in parallel.

### 6. Clean Imports and Types
Always use short imported names. Organize imports: Java standard library → external libraries → project classes.

---

## Quick Reference

**Common Anti-Patterns to Avoid:**
- ❌ `if (condition) { ... } else { ... }` → ✅ Use `filter()`, `switchIfEmpty()`
- ❌ `throw new Exception()` → ✅ Use `Mono.defer(() -> Mono.error())`
- ❌ `.then(Mono.just(x))` mid-flow → ✅ Breaks fail-fast, restructure flow
- ❌ Nested `flatMap` chains → ✅ Extract to helper methods
- ❌ `Mono<Void>` mid-flow → ✅ Only at final return
- ❌ `.block()` on Netty thread → ✅ Deadlock — keep chain reactive
- ❌ Blocking JDBC/RestTemplate on event loop → ✅ Use `Schedulers.boundedElastic()` or R2DBC
- ❌ `subscribe()` inside a reactive chain → ✅ Detached stream, errors swallowed — use `flatMap`

**Essential Patterns:**
```java
// Lazy error handling (mandatory pattern)
return Optional.ofNullable(data)
    .filter(d -> !d.isEmpty())
    .map(Mono::just)
    .orElse(Mono.defer(() -> Mono.error(BusinessType.EMPTY_DATA.build())));

// Repository with fallback
return repository.findById(id)
    .switchIfEmpty(Mono.defer(() -> Mono.error(BusinessType.NOT_FOUND.build(id))));

// Parallel validations
return Mono.zip(
    validateField1(data),
    validateField2(data),
    validateField3(data)
).thenReturn(data);
```

---

## Operator Selection Guide

**Is the operation synchronous (no I/O, no async calls)?**
- Yes → Use `map()`
- No → Continue

**Does it return Mono/Flux?**
- Yes → Use `flatMap()`
- No → Use `map()`

**Need to filter elements?**
- Use `filter()` + `switchIfEmpty()`

**Need to handle empty streams?**
- Use `switchIfEmpty()`

**Need to run operations in parallel?**
- Independent Monos → `Mono.zip()`
- Independent Fluxes → `Flux.merge()`

**Need lazy evaluation (especially errors)?**
- Use `Mono.defer()`

| Operator | Behavior | Use case |
|---|---|---|
| `map()` | Synchronous 1:1 transformation | Transforming data without I/O |
| `flatMap()` | Async 1:1, concurrent | DB calls, HTTP calls, returns Mono/Flux |
| `concatMap()` | Async 1:1, sequential ordered | Order-sensitive operations, DB writes |
| `switchMap()` | Cancels previous on new item | Live search, latest-value-wins |
| `filter()` | Conditional filtering | Keeping elements matching predicate |
| `switchIfEmpty()` | Fallback for empty streams | Resource not found, default value |
| `zip()` | Combine parallel Monos | Independent parallel lookups/validations |
| `merge()` | Combine Fluxes as they arrive | Unordered parallel Flux operations |
| `defer()` | Lazy evaluation | Mandatory for error wrapping |

---

## Schedulers — Threading Rules

| Scenario | Scheduler |
|---|---|
| Non-blocking async work | Default (Netty event loop) — do nothing |
| **Blocking I/O (JDBC, files, legacy libs)** | `Schedulers.boundedElastic()` |
| CPU-intensive computation | `Schedulers.parallel()` |

```java
// ✅ Wrapping a blocking call safely
public Mono<User> findUserBlocking(Long id) {
    return Mono.fromCallable(() -> legacyJdbcDao.findById(id))
               .subscribeOn(Schedulers.boundedElastic());
}
```

> `publishOn(scheduler)` — affects downstream operators.
> `subscribeOn(scheduler)` — affects the source subscription (use this when wrapping blocking sources).

---

## Error Handling Philosophy

All errors must be lazy to maintain reactive semantics. Never use `throw` directly in reactive chains. Always wrap errors in `Mono.defer()`.

**Why lazy errors matter:**
- Preserves fail-fast behavior in reactive chains
- Allows proper error propagation through operators
- Enables retry and fallback strategies
- Maintains backpressure semantics

---

## Detailed Documentation

For comprehensive guidance, refer to these resources:

- **[references/BEFORE_CODING.md](references/BEFORE_CODING.md)** - Pre-implementation analysis workflow and planning checklist
- **[references/REACTIVE_PATTERNS.md](references/REACTIVE_PATTERNS.md)** - Complete reactive patterns, anti-patterns, and transformations
- **[references/BEST_PRACTICES.md](references/BEST_PRACTICES.md)** - Code style, imports, constants, and organizational rules
- **[references/TROUBLESHOOTING.md](references/TROUBLESHOOTING.md)** - Common issues, debugging techniques, and solutions (includes BlockHound, Reactor debug mode, context propagation)
- **[references/SPRING-INFRASTRUCTURE.md](references/SPRING-INFRASTRUCTURE.md)** - Spring-specific: controllers, WebClient, SSE, reactive security, R2DBC

---

## Related Skills
- `spring-boot-patterns` — imperative MVC patterns
- `spring-testing` → `webflux-testing.md` — testing with WebTestClient and StepVerifier
- `spring-cloud-patterns` — Gateway (built on WebFlux), Resilience4j
