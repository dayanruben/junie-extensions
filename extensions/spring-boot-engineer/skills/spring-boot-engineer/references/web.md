# Web Layer — policy & pitfalls

Generic REST / Spring MVC knowledge (`@RestController`, `@GetMapping`, `@PathVariable`, `@Valid`) is assumed. This file only covers decisions that typically go wrong.

## DTO boundary

- **Never return or accept `@Entity` types from controllers.** Exposes persistence details, causes `LazyInitializationException` / `could not initialize proxy — no Session`, couples API to schema.
- Request / response DTOs → Java `record` or Kotlin `data class`. One `FooRequest` and one `FooResponse` per endpoint; don't reuse the same DTO for both directions.
- Mapping entity ↔ DTO lives in the service (or a dedicated mapper), not in the controller.

## Validation

- Every mutating endpoint: `@Valid @RequestBody` on the argument **and** constraints on the DTO fields (`@NotBlank`, `@Email`, `@Size`, `@Pattern`, `@Min/@Max`).
- `@Validated` on the controller class is required for validating `@PathVariable` / `@RequestParam` (method-level validation).
- Kotlin: use `@field:NotBlank` on `data class` properties — bare `@NotBlank` on a Kotlin property targets the constructor parameter and is silently ignored by Jakarta Validation.

## Error handling

- Use `ProblemDetail` (RFC 7807) — built-in since Spring 6. Do not hand-roll `ErrorResponse` records.
- One `@RestControllerAdvice` with handlers for: `MethodArgumentNotValidException` (400 + field errors), domain `NotFoundException` (404), `DataIntegrityViolationException` (409), fallback `Exception` (500, log full stack, return generic message).
- Never leak stack traces or SQL fragments to clients.
- Spring Boot 3.2+: set `spring.mvc.problemdetails.enabled=true` to let Spring produce `ProblemDetail` for framework exceptions automatically.

## Pagination

- Prefer `Pageable` + `@PageableDefault(size = 20, sort = "id")` over manual `page` / `size` params.
- Return `Page<DTO>` when the UI needs total count; return `Slice<DTO>` when it only needs next/prev (cheaper — skips `COUNT(*)`).
- Cap max page size via `spring.data.web.pageable.max-page-size=100` to prevent clients requesting `?size=1000000`.

## Outbound HTTP

- For new code use `RestClient` (Spring 6.1+, synchronous, fluent) or declarative `@HttpExchange` interfaces. `RestTemplate` is in maintenance — avoid in green-field code.
- Use `WebClient` only in WebFlux apps (or when you actually need reactive / streaming). Inside MVC it works but adds complexity for no benefit.
- Always set a timeout. Default `RestClient` / `WebClient` have **no read timeout** — requests can hang forever.

## CORS

- Prefer a single `CorsConfigurationSource` bean wired into `SecurityFilterChain` over `WebMvcConfigurer.addCorsMappings`. When security is on, the filter-chain config is the one that actually fires first; the MVC-level one gets bypassed for preflights.
- Never use `allowedOrigins("*")` together with `allowCredentials(true)` — Spring throws at startup, and browsers reject the response anyway.

## Deprecations to avoid

- `WebMvcConfigurerAdapter` → implement `WebMvcConfigurer` directly.
- `antMatchers(...)` in security / MVC config → `requestMatchers(...)`.
- `@RequestMapping(method = RequestMethod.GET)` on a method → `@GetMapping` / `@PostMapping` / ...
