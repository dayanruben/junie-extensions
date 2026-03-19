---
name: "jpa-patterns"
description: "JPA/Hibernate patterns and common pitfalls (N+1, lazy loading, transactions, queries). Use when user has JPA performance issues, LazyInitializationException, or asks about entity relationships and fetching strategies."
---

# JPA Patterns Skill

## Scope and Version Notes

- This is the canonical skill for JPA/Hibernate-specific guidance.
- Examples are pattern fragments; verify imports, transaction boundaries, and ORM-version details against the current project.
- Use `spring-boot-patterns` for high-level API/service structure and `sql-patterns` for raw SQL and migration advice.

## When to Use
- User mentions "N+1 problem" / "too many queries" / slow endpoints
- `LazyInitializationException` errors
- Questions about fetch strategies, transactions, entity relationships
- Query optimization

## Quick Reference

| Problem | Symptom | Solution |
|---------|---------|----------|
| N+1 queries | Many SELECTs in logs | `JOIN FETCH`, `@EntityGraph` |
| LazyInitializationException | Error outside transaction | JOIN FETCH, DTO projection, `@Transactional` |
| Slow queries | Performance issues | Pagination, projections, indexes |
| Dirty checking overhead | Slow bulk reads | `readOnly = true`, DTO |
| Concurrent update conflicts | Lost updates | `@Version` (optimistic locking) |

---

## N+1 Problem — Most Common Issue

```java
// ❌ BAD: N+1 — 1 query for authors + N queries for books
List<Author> authors = authorRepository.findAll();
authors.forEach(a -> System.out.println(a.getBooks().size())); // N queries!

// ✅ Solution 1: JOIN FETCH
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// ✅ Solution 2: @EntityGraph (declarative)
@EntityGraph(attributePaths = {"books"})
List<Author> findAll();

// ✅ Solution 3: Batch fetching (global setting)
// application.properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

Detect N+1:
```yaml
logging.level.org.hibernate.SQL: DEBUG
logging.level.org.hibernate.orm.jdbc.bind: TRACE
```

---

## Lazy Loading — Default Strategy

```java
// ✅ Always use LAZY (override EAGER defaults on @ManyToOne)
@ManyToOne(fetch = FetchType.LAZY)   // override default EAGER
private Customer customer;

@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)  // already default
private List<OrderItem> items;
```

Fix `LazyInitializationException` → see [references/QUERY-OPTIMIZATION.md](references/QUERY-OPTIMIZATION.md)

---

## Transactions — Key Rules

```java
@Service
@Transactional(readOnly = true)   // default for all methods
public class OrderService {

    @Transactional                // override for writes
    public Order create(CreateOrderRequest req) { ... }
}
```

```java
// ❌ COMMON MISTAKE: @Transactional on self-call is ignored
public void process(Long id) {
    updateOrder(id);  // proxy bypassed — NO transaction!
}
@Transactional
public void updateOrder(Long id) { ... }
```

Full transaction details → [references/TRANSACTIONS.md](references/TRANSACTIONS.md)

---

## Entity Relationships

```java
// ✅ Bidirectional — always sync both sides
@Entity
public class Author {
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books = new ArrayList<>();

    public void addBook(Book book) { books.add(book); book.setAuthor(this); }
    public void removeBook(Book book) { books.remove(book); book.setAuthor(null); }
}
```

Full relationship guide → [references/RELATIONSHIPS.md](references/RELATIONSHIPS.md)

---

## Performance Checklist

- [ ] No N+1 queries (use JOIN FETCH or `@EntityGraph`)
- [ ] `FetchType.LAZY` by default (especially `@ManyToOne`)
- [ ] Pagination for large result sets
- [ ] DTO projections for read-only queries
- [ ] Bulk operations for batch updates/deletes
- [ ] `@Version` for entities with concurrent access
- [ ] Indexes on frequently queried columns
- [ ] No lazy fields in `toString()`
- [ ] `readOnly = true` on read transactions
