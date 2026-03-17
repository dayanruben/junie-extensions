---
name: "concurrency-patterns"
description: "Kotlin coroutines, Project Reactor, thread safety patterns. Use when working with async code, coroutines, or reactive streams."
---

# Concurrency Patterns Skill

Use this skill when correctness depends on async execution, thread safety, cancellation, or backpressure.

## Scope and Boundaries

- Use this skill for coroutines, Reactor, shared-state safety, and concurrency review.
- Use `debugging-investigation-patterns` when the failure mode is still unclear.
- Use `performance-patterns` for throughput and resource tuning after correctness is understood.
- Treat examples as pattern fragments; adapt dispatchers, schedulers, and framework integrations to the current stack.

## When to Use
- Writing async or concurrent code
- Reviewing coroutine or reactive code
- Fixing thread-safety issues
- Investigating race conditions, deadlocks, cancellation bugs, or backpressure problems

## Kotlin Coroutines

### Structured Concurrency

```kotlin
// ✅ Good — use coroutineScope for parallel work
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val users = async { userService.fetchAll() }
    val stats = async { statsService.fetch() }
    Dashboard(users.await(), stats.await())
}

// ❌ Bad — GlobalScope leaks coroutines
GlobalScope.launch { doWork() }
```

### Dispatcher Selection

```kotlin
// ✅ IO-bound work
withContext(Dispatchers.IO) { database.query() }

// ✅ CPU-bound work
withContext(Dispatchers.Default) { heavyComputation() }

// ✅ Unconfined / custom dispatcher for special cases
withContext(Dispatchers.Unconfined) { /* resumes in caller thread */ }
```

### Flow

```kotlin
// ✅ Good — cold stream
fun userUpdates(): Flow<User> = flow {
    while (true) {
        emit(userRepository.findLatest())
        delay(5_000)
    }
}

// ✅ Collect with cancellation safety
userUpdates()
    .catch { e -> logger.error("Error", e) }
    .collect { user -> process(user) }
```

### Coroutine Review Heuristics

- Prefer structured concurrency over detached jobs.
- Make cancellation propagation explicit for long-running work.
- Avoid blocking calls on default or event-loop threads.
- Prefer immutable state handoff; if state is shared, make synchronization obvious.

---

## Thread Safety

```kotlin
// ✅ Good — use StateFlow for shared mutable state
private val _state = MutableStateFlow(initialState)
val state: StateFlow<State> = _state.asStateFlow()

// ✅ Good — atomic operations
private val counter = AtomicLong(0)
fun increment() = counter.incrementAndGet()

// ❌ Bad — non-atomic read-modify-write
private var counter = 0
fun increment() { counter++ } // race condition
```

Review checklist:
- Is any mutable state accessed from multiple threads or coroutines?
- Is compound state update atomic?
- Are locks, atomics, or message-passing boundaries clear?
- Could retries or duplicate delivery execute the same code concurrently?

---

## Project Reactor

```kotlin
// ✅ Good — compose operators
fun getActiveUsers(): Flux<User> = userRepository.findAll()
    .filter { it.isActive }
    .flatMap { user -> enrichWithProfile(user) }
    .onErrorResume { e -> Flux.empty<User>().also { logger.error("Error", e) } }

// ✅ Good — avoid blocking in reactive pipeline
// ❌ Bad
mono.map { blockingCall() } // blocks reactor thread
// ✅ Good
mono.flatMap { Mono.fromCallable { blockingCall() }.subscribeOn(Schedulers.boundedElastic()) }
```

## Common Failure Patterns

### Race conditions
- Non-atomic read-modify-write on shared state
- Assuming callbacks always execute in the same order
- Updating caches or accumulators from parallel flows without synchronization

### Cancellation leaks
- Launching child jobs outside the request or use-case scope
- Ignoring cancellation when wrapping blocking I/O
- Swallowing `CancellationException` as a generic error

### Backpressure issues
- Unbounded buffering between producer and consumer
- Parallelism higher than downstream capacity
- Blocking work placed on event-loop threads

## Anti-Patterns

- Using `GlobalScope` or fire-and-forget jobs for business-critical work without lifecycle ownership
- Mixing blocking and non-blocking code without an explicit boundary
- Hiding shared mutable state behind helper methods instead of making coordination visible
- Increasing parallelism to “fix” latency before proving the bottleneck
