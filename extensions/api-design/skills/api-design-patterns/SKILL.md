---
name: "api-design-patterns"
description: "HTTP API design patterns: resource modeling, contracts, validation boundaries, error models, idempotency, pagination, and backward-compatible evolution. Use when designing or reviewing external or internal APIs."
---

# API Design Patterns Skill

Design APIs that are predictable, evolvable, and easy for clients to use correctly.

## Scope and Boundaries

- Use this skill for HTTP API contracts, resource naming, request/response modeling, versioning, idempotency, and error semantics.
- Use `spring-boot-patterns` for controller/service layering and high-level Spring Boot structure.
- Use `validation-patterns` for Bean Validation annotations, custom validators, and validation groups.
- Use `security-audit` for authentication, authorization, sensitive data handling, and secure defaults.
- Treat examples as pattern fragments; adapt media types, framework annotations, and package structure to the current project.

## When to Use

- Designing a new REST or HTTP API
- Reviewing an existing controller contract
- Adding pagination, filtering, sorting, or partial updates
- Standardizing error responses
- Planning backward-compatible API changes

## Core Principles

- Model resources and actions from the client’s point of view.
- Prefer consistent naming and payload shape across endpoints.
- Keep transport DTOs separate from persistence entities.
- Prefer explicit contracts over “smart” overloaded behavior.
- Design for backward compatibility by default.

## Resource Modeling

- Use plural nouns for collections: `/users`, `/orders`, `/invoices`
- Use nested resources only when the parent-child relationship is real and stable: `/orders/{orderId}/items`
- Avoid RPC-style endpoints unless the operation is not naturally resource-based

Prefer:
- `POST /orders`
- `GET /orders/{id}`
- `POST /orders/{id}/cancel`

Avoid:
- `POST /createOrder`
- `GET /getOrderById?id=...`

## Request and Response DTOs

- Use separate DTOs for create/update/request/response shapes.
- Keep server-managed fields out of create requests.
- Return stable identifiers and relevant state in responses.
- Avoid leaking internal enum names, database IDs semantics, or persistence-only fields unless they are part of the contract.

```kotlin
data class CreateOrderRequest(
    val customerId: String,
    val items: List<CreateOrderItemRequest>,
    val note: String?,
)

data class OrderResponse(
    val id: String,
    val status: String,
    val createdAt: Instant,
    val totalAmount: BigDecimal,
)
```

## HTTP Semantics

- `GET` for safe reads
- `POST` for creation or non-idempotent commands
- `PUT` for full replacement when the semantics are truly full replacement
- `PATCH` for partial updates
- `DELETE` for delete semantics, including soft-delete if that is the public contract

Prefer explicit command endpoints when a domain action is clearer than an ambiguous partial update.

Examples:
- `PATCH /users/{id}` for patching profile fields
- `POST /orders/{id}/cancel` for a domain command with business rules

## Status Codes and Error Model

- `200 OK` for successful reads and updates returning a body
- `201 Created` for successful creation, usually with `Location`
- `202 Accepted` for asynchronous processing
- `204 No Content` for successful operations with no body
- `400 Bad Request` for malformed payloads or contract violations
- `401 Unauthorized` for missing/invalid authentication
- `403 Forbidden` for authenticated caller lacking permission
- `404 Not Found` when the resource is absent or intentionally hidden
- `409 Conflict` for version/state conflicts or duplicate commands
- `422 Unprocessable Entity` only if your platform uses it consistently for semantic validation failures

Prefer one stable error shape:

```kotlin
data class ApiErrorResponse(
    val code: String,
    val message: String,
    val details: List<ApiErrorDetail> = emptyList(),
    val traceId: String? = null,
)

data class ApiErrorDetail(
    val field: String?,
    val issue: String,
)
```

## Validation Boundary

- Validate transport-level constraints at the API boundary.
- Validate business invariants in the application/domain layer.
- Return field-level details for client-fixable input errors.
- Defer annotation-level guidance to `validation-patterns`.

## Pagination, Filtering, and Sorting

- Prefer explicit query parameters: `page`, `size`, `sort`, `filter`.
- Cap page size to protect the service.
- Define stable sort semantics; do not rely on database default ordering.
- Prefer cursor pagination for large, append-heavy datasets; offset pagination is acceptable for simpler admin/read screens.

Document:
- default page size
- max page size
- supported sort fields
- filtering operators and case sensitivity

## Idempotency and Retries

- Assume clients and gateways may retry requests.
- For non-idempotent operations with retry risk, prefer idempotency keys or explicit deduplication.
- Make duplicate handling explicit in the contract.

Examples:
- `POST /payments` with `Idempotency-Key` header
- `409 Conflict` or replaying the original response for duplicate commands, depending on the contract

## Versioning and Evolution

- Prefer backward-compatible additive changes first.
- Avoid removing fields or changing field meaning without a migration plan.
- Version only when compatibility cannot be preserved.
- Keep deprecation explicit and time-bound.

Usually safe changes:
- adding optional response fields
- adding optional request fields with sensible defaults
- adding new enum values only if clients are prepared for unknown values

Risky changes:
- renaming fields
- changing validation rules incompatibly
- changing pagination defaults silently
- changing synchronous behavior to async without contract updates

## Checklist Before Shipping

- Are resource names and verbs consistent with the rest of the API?
- Are DTOs explicit and decoupled from entities?
- Are status codes and error bodies consistent?
- Are pagination and sorting semantics documented?
- Are retries and duplicates handled safely?
- Is the change backward compatible for existing clients?

## Anti-Patterns

- Returning entities directly from controllers
- Mixing transport validation and business invariants in one opaque error
- Overusing generic `Map<String, Any>` payloads for stable contracts
- Hiding major behavioral changes behind the same endpoint contract
- Using one endpoint that behaves differently based on many unrelated flags