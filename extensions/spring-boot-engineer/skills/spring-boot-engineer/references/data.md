# Data Access — policy & pitfalls

Generic JPA / Spring Data knowledge (`JpaRepository`, `@Query`, derived queries, `@Entity`, `@OneToMany`) is assumed. This file covers the decisions that actually break production apps.

## Must-fix config

- **Always set `spring.jpa.open-in-view=false`.** The default is `true`, which keeps the Hibernate `Session` open for the entire HTTP request. This hides lazy-loading bugs, holds a DB connection under load, and is the single biggest cause of slow Spring apps.
- `spring.jpa.hibernate.ddl-auto=validate` in prod (never `update` / `create`). Use Flyway / Liquibase for schema changes.
- Log SQL in dev only (`spring.jpa.show-sql=false` or use `spring.jpa.properties.hibernate.format_sql=true` with `logging.level.org.hibernate.SQL=DEBUG`). Production with `show-sql=true` destroys throughput.

## Transactions — read this before writing `@Transactional`

- **Self-invocation doesn't start a transaction.** `this.methodA()` calling `this.methodB()` where only `methodB` has `@Transactional` — the proxy is bypassed, no transaction begins. Fix: extract to a separate bean, or inject `self` via `@Lazy`.
- `@Transactional` works only on **`public` non-`final` methods** of a Spring bean. Private / package-private / final / static → silently ignored. Kotlin: the `kotlin-spring` plugin opens classes so CGLIB proxies work.
- **Checked exceptions do not roll back by default.** Only `RuntimeException` and `Error` do. For checked exceptions: `@Transactional(rollbackFor = Exception.class)` (or a tighter superclass).
- `@Transactional(readOnly = true)` on queries — lets Hibernate skip dirty checking and flushes. **Never write inside `readOnly = true`** — the flush is skipped silently, updates disappear.
- `@Async` + `@Transactional` on the same method: the async thread doesn't inherit transaction context. Entity becomes detached → `LazyInitializationException`. Split: sync method opens tx, async method runs outside it (or use `@TransactionalEventListener(phase = AFTER_COMMIT)`).
- `@Transactional` on `@PostConstruct` is ignored — proxy isn't ready yet.
- Long-running transactions are an anti-pattern. Keep `@Transactional` blocks short; do network calls / external HTTP **outside** the transaction.

## N+1 queries — the default killer

Default `@OneToMany` / `@ManyToOne` are lazy. Iterating a list and touching a lazy association = one query per entity.

Fixes (pick one, in order of preference):

1. **Projection interface / DTO query** — select only what you need, no entities:
   ```java
   interface UserView { String getName(); String getEmail(); }
   List<UserView> findBy();
   ```
2. **`@EntityGraph`** on the repository method — declarative, per-query:
   ```java
   @EntityGraph(attributePaths = {"orders", "orders.items"})
   List<User> findAll();
   ```
3. **`JOIN FETCH`** in JPQL when EntityGraph isn't flexible enough:
   ```java
   @Query("select u from User u join fetch u.orders where u.active = true")
   List<User> findActiveWithOrders();
   ```

Never solve N+1 with `FetchType.EAGER` on the entity — it pollutes every query, not just the one that needs it.

Detection: enable `spring.jpa.properties.hibernate.generate_statistics=true` + `logging.level.org.hibernate.stat=DEBUG` in dev, or use Hypersistence Utils / Datasource Proxy to log query counts per request.

## Pagination + fetch join

`@Query("... join fetch ...")` + `Pageable` together emits `HHH90003004` ("firstResult/maxResults specified with collection fetch; applying in memory") — Hibernate loads **the entire collection** and paginates in Java. Two-step fix:

1. Page IDs: `@Query("select u.id from User u") Page<Long> pageIds(Pageable p);`
2. Fetch entities for that page: `@Query("select u from User u join fetch u.orders where u.id in :ids") List<User> findByIds(List<Long> ids);`

## Entity hygiene

- `equals` / `hashCode` based on `id` is unsafe before `persist()` (id is null). For JPA entities prefer a business key, or use `Objects.hash(getClass())` + id-aware equals that handles null.
- `@GeneratedValue(strategy = GenerationType.IDENTITY)` disables Hibernate's JDBC batch inserts. For bulk inserts use `SEQUENCE` + `hibernate.jdbc.batch_size=50` + `hibernate.order_inserts=true` / `order_updates=true`.
- Bidirectional `@OneToMany` + `@ManyToOne`: manage both sides in helper methods (`addOrder(order) { orders.add(order); order.setUser(this); }`). Forgetting one side → dangling FKs on flush.
- Kotlin entities require `kotlin-jpa` plugin (generates no-arg constructor). Without it: `InstantiationException: No default constructor for entity`.

## Repository patterns

- Use interface projections for read-only queries; Spring builds the proxy automatically.
- `Specification<T>` for dynamic filters (search forms) instead of concatenating JPQL.
- Derived query methods (`findByEmailAndActiveTrue`) stay readable up to ~3 conditions; beyond that, switch to `@Query` or `Specification`.
- `@Modifying @Query("update ...")` requires `clearAutomatically = true` (or manual `em.clear()`) — otherwise stale entities remain in the persistence context and subsequent reads see the pre-update state.

## Connection pool (HikariCP defaults rarely fit prod)

- Default `maximum-pool-size=10`. Size it as `pool = ((core_count * 2) + effective_spindle_count)` — usually 10–30 per instance, **not** "more is better" (DB connection count is expensive).
- Always set `spring.datasource.hikari.connection-timeout` (default 30s) and `max-lifetime` (shorter than DB/proxy idle timeout).
- Leak detection in non-prod: `spring.datasource.hikari.leak-detection-threshold=5000`.

## Cheat table — common errors and their cause

| Error / symptom | Typical cause |
|---|---|
| `LazyInitializationException` | Accessing lazy association outside transaction (and OSIV is off, which is correct). Fetch it eagerly via `EntityGraph` / `JOIN FETCH` / projection. |
| `could not initialize proxy - no Session` | Same as above. |
| Update "runs" but nothing changes in DB | Called from another method in the same bean (self-invocation); or inside `readOnly = true`; or inside `@Async` method without proper tx propagation. |
| `HHH90003004` in logs, slow pagination | `JOIN FETCH` + `Pageable` together. Use two-step ID paging. |
| Mysterious `OptimisticLockException` on every save | Missing `@Version` column but concurrent updates; or entity re-attached from a cache without refreshing version. |
| `InstantiationException: No default constructor` | Kotlin `@Entity` without `kotlin-jpa` plugin. |
