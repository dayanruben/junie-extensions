---
name: "debugging-investigation-patterns"
description: "Workflow for debugging JVM applications: reproduce issues, narrow scope, use logs/metrics/traces, and validate fixes. Use when investigating incidents, flaky behavior, or unclear failures."
---

# Debugging Investigation Patterns Skill

Use this skill for investigation work: understand the failure mode, collect the smallest useful evidence set, form a hypothesis, and prove or disprove it with targeted checks.

## Scope and Boundaries

- Use this skill for debugging workflow, incident triage, hypothesis-driven investigation, and safe observability usage.
- Use `spring-actuator-patterns` for Spring Boot Actuator and Micrometer configuration details.
- Use `performance-patterns` when the problem is already known to be throughput, latency, CPU, memory, or caching related.
- Use `concurrency-patterns` when the issue is specifically about thread safety, coroutines, race conditions, or backpressure.
- Treat examples as pattern fragments; adapt tooling, libraries, and framework APIs to the current project.

## When to Use

- A bug is reported but the root cause is unknown
- Tests are flaky or fail only in CI / only locally
- Logs show symptoms, but not the cause
- A service returns wrong data, intermittent errors, or unexpected timeouts
- You need to build a minimal reproducer before changing code

## Investigation Workflow

1. Define the observed failure precisely.
   - What was expected?
   - What actually happened?
   - In which environment, inputs, and timeframe?
2. Reproduce with the smallest possible scenario.
   - Prefer one endpoint, one use case, one dataset, one failing test.
3. Narrow the blast radius.
   - Is it request-specific, tenant-specific, time-dependent, concurrency-related, or environment-specific?
4. Collect evidence before editing code.
   - Relevant logs
   - Metrics / counters / timers
   - Recent config or dependency changes
   - Stack traces and correlation IDs
5. Form one concrete hypothesis at a time.
6. Run the smallest validation that can falsify that hypothesis.
7. Only implement a fix after the cause is demonstrated.
8. Re-run the reproducer and confirm the fix does not only hide the symptom.

## Triage Checklist

- Check whether the issue is deterministic or intermittent.
- Check whether it started after a deploy, config change, schema change, or dependency upgrade.
- Check whether the failure is isolated to one node / region / queue / external dependency.
- Check whether bad input, missing validation, or stale cache can explain the behavior.
- Check whether retries, timeouts, circuit breakers, or async boundaries are masking the first failure.

## Logging Guidelines

- Prefer structured logs with stable field names.
- Include correlation identifiers, request identifiers, tenant or aggregate identifiers when available.
- Log state transitions and decision points, not every line of execution.
- Avoid logging secrets, tokens, raw personal data, or huge payloads.
- Prefer a temporary focused log around the suspected branch instead of broad noisy logging.

```kotlin
private val logger = LoggerFactory.getLogger(OrderService::class.java)

fun submit(command: CreateOrderCommand): OrderResult {
    logger.info(
        "Submitting order: orderId={}, customerId={}, itemsCount={}",
        command.orderId,
        command.customerId,
        command.items.size,
    )

    val inventoryAvailable = inventoryGateway.reserve(command.items)
    if (!inventoryAvailable) {
        logger.warn("Inventory reservation failed: orderId={}", command.orderId)
        return OrderResult.rejected("inventory_unavailable")
    }

    return OrderResult.accepted(command.orderId)
}
```

## Metrics and Traces During Investigation

- Prefer adding short-lived counters/timers when logs alone cannot show frequency or latency.
- Compare successful and failed paths using the same dimensions.
- If tracing is available, follow one failing request across service boundaries before changing code.
- If metrics are missing, add the smallest metric that can confirm or reject the current hypothesis.

Good investigation questions:
- Did error rate change first, or latency?
- Is one dependency slower than the others?
- Do failures correlate with one endpoint, status code, or tenant?

## Minimal Reproducer Strategy

- Start from the smallest failing automated test or CLI reproduction.
- Freeze non-essential variables: seed data, clocks, random values, time zones, and concurrency level.
- Remove unrelated integrations until the failure disappears.
- If the reproducer becomes too broad, split it into smaller cases until one case explains the bug.

## Common Failure Patterns

### Wrong data, no exception
- Compare read model vs write model
- Check mapping, serialization, timezone, null/default handling
- Check stale cache or eventual consistency window

### Intermittent failure
- Check shared mutable state, ordering assumptions, retries, and background jobs
- Check test isolation and cleanup
- Check clock and time-based logic

### Only in production or CI
- Check environment variables, profiles, secrets, feature flags, and missing migrations
- Check resource limits, startup ordering, and external dependency availability

### Timeout or slowdown
- Check downstream latency, pool exhaustion, lock contention, and unexpected N+1 queries
- Defer to `performance-patterns` if the issue is confirmed to be primarily performance-related

## Anti-Patterns

- Do not patch code before you can describe the failure mode precisely.
- Do not add broad debug logging everywhere and keep it permanently.
- Do not assume the last changed file is the root cause.
- Do not stop at the first symptom if retries or wrappers can hide the original failure.
- Do not “fix” flaky tests by relaxing assertions or adding sleeps without proving timing is the root cause.

## Outcome Standard

An investigation is done when you can state all of the following:

- exact trigger or input shape
- failing component or boundary
- evidence that supports the root-cause hypothesis
- smallest fix or mitigation that addresses the cause
- validation that the original reproducer now passes