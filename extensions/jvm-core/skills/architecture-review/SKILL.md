---
name: "architecture-review"
description: "Macro-level architecture guidance for Kotlin/Java backends: package structure, module boundaries, dependency direction, and layering trade-offs. Use when reviewing or changing project structure, boundaries, or large refactorings."
---

# Architecture Review Skill

Use this skill when the main question is not a single class implementation, but whether the overall structure is easy to evolve safely.

## Scope and Boundaries

- Use this skill for package structure, module boundaries, dependency direction, layering, and architecture smell review.
- Use `spring-boot-patterns` for Spring Boot component roles and high-level application structure.
- Use `api-design-patterns` for HTTP contract design.
- Use `jpa-patterns` for persistence-layer specifics.
- Treat examples as pattern fragments; adapt naming, package layout, and module conventions to the current project.

## When to Use

- Reviewing package organization or module structure
- Planning a refactoring that crosses multiple features or layers
- Checking whether responsibilities are leaking between controllers, services, repositories, and infrastructure
- Evaluating whether a project is over-layered or under-structured

## Core Principles

- Prefer feature cohesion over large horizontal buckets.
- Keep dependency direction easy to explain.
- Separate API, application, domain, and infrastructure concerns when the project complexity justifies it.
- Prefer the simplest structure that still keeps change localized.
- Avoid introducing architectural patterns that the current team and codebase do not need yet.

## Common Organization Options

### Package by Feature

Prefer for most growing business applications:

```text
com.example.app/
  orders/
    OrderController.kt
    OrderService.kt
    OrderRepository.kt
    OrderMapper.kt
  customers/
    CustomerController.kt
    CustomerService.kt
    CustomerRepository.kt
```

Why it helps:
- related code stays together
- feature extraction is easier later
- reviewers can inspect one feature in one place

### Package by Layer

Acceptable for small projects or narrow services, but watch for growth:

```text
com.example.app/
  controller/
  service/
  repository/
  model/
```

Risk signs:
- very large `service` or `repository` packages
- one change touches many distant packages
- feature boundaries become unclear

### Explicit Domain / Application / Infrastructure Split

Prefer only when complexity really demands stronger boundaries:

```text
com.example.app/
  domain/
  application/
  infrastructure/
  web/
```

Use when:
- there are multiple adapters or external integrations
- business rules must stay isolated from framework code
- module extraction is likely

## Architecture Smells to Check

- Controllers contain business rules, transactions, or persistence details.
- Services know HTTP, SQL, or queue protocol details directly.
- Domain code depends on framework annotations or infrastructure clients without a good reason.
- One `common`, `shared`, or `util` package keeps growing without clear ownership.
- Circular dependencies appear between features or modules.
- A change in one feature requires edits across many unrelated packages.

## Dependency Direction Heuristics

- Web/API layer may depend on application services, but not on repository implementation details.
- Persistence and external clients should support the domain/application flow, not define it.
- Shared modules should provide stable abstractions, not become a dumping ground.
- If a dependency direction is hard to justify in one sentence, reconsider it.

## Review Checklist

- Can you explain the responsibility of each top-level package or module quickly?
- Are related classes grouped by feature or by a boundary with a clear reason?
- Do dependencies mostly point inward toward more stable code?
- Is any package acting as a catch-all for unrelated helpers?
- Would a new feature require editing only one bounded area, or many cross-cutting folders?
- Is the current structure simpler than the next more abstract alternative?

## Refactoring Guidance

- Prefer incremental moves over architecture rewrites.
- Move one feature or boundary at a time and keep behavior unchanged during structural refactors.
- Add module or package boundaries only when they prevent recurring confusion or coupling.
- If the project is still small, prefer clearer naming and feature grouping over introducing many layers.

## Anti-Patterns

- Forcing hexagonal or clean architecture on a small CRUD service without real pressure.
- Creating deep package trees with one class per folder.
- Introducing a `common` module for code that is not actually stable and shared.
- Mixing architecture refactoring with large behavior changes in the same step.

## Outcome Standard

An architecture review is done when you can state:

- what the main structural boundaries are
- where the current structure helps or hurts changeability
- which smells are real versus acceptable trade-offs
- the smallest structural improvement worth making next