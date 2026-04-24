# spring-boot-engineer

Spring Boot 3.x expert for Java and Kotlin. Turns Junie into a senior Spring engineer that writes idiomatic code **and** catches the production pitfalls most tutorials skip.

## Philosophy

This extension is intentionally thin. Spring Boot's happy path (`@RestController`, `@GetMapping`, `JpaRepository`, `SecurityFilterChain` boilerplate) is what LLMs already produce well — duplicating that here would just inflate context. Every file covers **project policy + the traps that bite in production**, not a rewrite of the docs.

## What it covers

- **Web layer** — DTO boundary rules, `ProblemDetail` error model, pagination, `RestClient` / `WebClient` choice, CORS with Spring Security.
- **Data access** — N+1 diagnosis, `spring.jpa.open-in-view=false`, `@Transactional` proxy rules, fetch-join + pagination (`HHH90003004`), HikariCP tuning.
- **Security** — Spring Security 6 filter chain, CSRF policy for cookie vs stateless, OAuth2 resource server + JWT, method security, `/actuator/*` hardening.
- **Testing** — test slices (`@WebMvcTest`, `@DataJpaTest`, `@RestClientTest`), `@MockitoBean` replacing deprecated `@MockBean`, Testcontainers with `@ServiceConnection`.
- **Migrations** — Flyway / Liquibase, expand-contract rename/drop, `CREATE INDEX CONCURRENTLY`, baseline-on-migrate for legacy DBs.
- **Scheduling & observability** — `@Scheduled` in a clustered deployment (ShedLock), health probes done right, Micrometer cardinality, virtual threads trade-offs.
- **Kotlin** — `kotlin-spring` / `kotlin-jpa` plugins, `@field:` validation, `suspend` controllers, `data class` vs `@Entity`.
- **Event-driven** — `@TransactionalEventListener` phases, outbox pattern, Kafka idempotence.
- **Resilience** — Resilience4j annotation order, `fallbackMethod` signature, retry + circuit breaker interaction.
- **Reactive (WebFlux)** — when it's actually the right choice, blocking traps, `Schedulers.boundedElastic()`, context propagation.
- **Cloud native** — `spring.config.import` (not `bootstrap.yml`), `@RefreshScope` limits, Gateway on WebFlux, when to skip Spring Cloud on Kubernetes.

## Pitfalls it catches

- `@Transactional` self-invocation, private / final / static, checked-exception rollback, `@Async` + `@Transactional`, `readOnly = true` + writes.
- N+1 queries and `spring.jpa.open-in-view=true` (the silent production killer).
- `JOIN FETCH` + `Pageable` (`HHH90003004` — Hibernate paginates in memory).
- Kotlin + Spring runtime failures when `plugin.spring` / `plugin.jpa` are missing (`InstantiationException`, CGLIB proxy failures).
- Bean Validation silently ignored on Kotlin `data class` without `@field:`.
- Blocking JDBC / `.block()` / blocking filters inside WebFlux and Spring Cloud Gateway.
- Returning JPA entities from controllers (lazy-loading blowups, API coupling).
- `@Scheduled` duplicating jobs across cluster replicas without a distributed lock.
- Exposing all actuator endpoints publicly; high-cardinality Micrometer labels.
- `.csrf.disable()` on cookie-session apps (real CSRF surface).
- Editing already-applied Flyway migrations; running `flyway clean` in shared environments.

## Installation

Add this extension to Junie via your configuration settings. The `SKILL.md` hub loads additional references on demand from `skills/spring-boot-engineer/references/`.

## Requirements

- Spring Boot 3.x project (`spring-boot-starter-parent` in Maven, or `org.springframework.boot` Gradle plugin).
- Java 17+ (Spring Boot 3 requirement); Java 21 recommended for records + pattern matching.
- For Kotlin projects using Spring / JPA: `kotlin-spring` and `kotlin-jpa` compiler plugins.
