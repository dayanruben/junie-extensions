# JPA Query Optimization Reference

## Fix LazyInitializationException

```java
// Option 1: JOIN FETCH in query
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);

// Option 2: @Transactional on service method
@Transactional(readOnly = true)
public OrderDTO getOrderWithItems(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    order.getItems().size();  // safe — inside transaction
    return mapper.toDTO(order);
}

// Option 3: DTO projection (best for read-only)
@Query("SELECT new com.example.dto.OrderDTO(o.id, o.status, SIZE(o.items)) FROM Order o WHERE o.id = :id")
Optional<OrderDTO> findOrderDTO(@Param("id") Long id);
```

---

## Pagination

```java
// ✅ Always paginate large result sets
public interface OrderRepository extends JpaRepository<Order, Long> {
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
}

// Usage
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());
Page<Order> page = orderRepository.findByStatus(OrderStatus.PENDING, pageable);

// Slice — lighter than Page (no COUNT query)
Slice<Order> slice = orderRepository.findByStatus(status, pageable);
```

---

## DTO Projections — Fetch Only What You Need

```java
// Interface projection — Spring Data generates proxy
public interface OrderSummary {
    Long getId();
    String getStatus();
    @Value("#{target.customer.name}")
    String getCustomerName();
}

List<OrderSummary> findByStatus(OrderStatus status);

// Class projection with constructor expression
public record OrderDTO(Long id, String status, String customerName) {}

@Query("SELECT new com.example.dto.OrderDTO(o.id, o.status, o.customer.name) " +
       "FROM Order o WHERE o.status = :status")
List<OrderDTO> findOrderDTOs(@Param("status") OrderStatus status);
```

---

## Bulk Operations

```java
// ✅ Bulk update — no entity loading
@Modifying
@Query("UPDATE Order o SET o.status = :status WHERE o.createdAt < :before")
int archiveOldOrders(@Param("status") OrderStatus status, @Param("before") LocalDateTime before);

// ✅ Bulk delete
@Modifying
@Query("DELETE FROM Order o WHERE o.status = 'CANCELLED' AND o.createdAt < :before")
int deleteCancelledOrders(@Param("before") LocalDateTime before);

// Usage — must be in @Transactional
@Transactional
public void archiveOrders() {
    int count = orderRepository.archiveOldOrders(
        OrderStatus.ARCHIVED,
        LocalDateTime.now().minusYears(1)
    );
    log.info("Archived {} orders", count);
}
```

**Note:** `@Modifying` does **not** clear the persistence context by default (`clearAutomatically = false`). Add `clearAutomatically = true` if you need the context cleared after the bulk operation to avoid stale entities in the same transaction.

---

## Exists and Count

```java
// ✅ Prefer existsBy over findBy for existence checks — no entity hydration
boolean existsByEmail(String email);      // SELECT 1 FROM ...
long countByDepartmentId(Long deptId);    // SELECT COUNT(*) FROM ...

// ❌ Wasteful
Optional<User> user = findByEmail(email);
if (user.isPresent()) { ... }            // loaded full entity for nothing
```

---

## @EntityGraph for Multiple Associations

```java
// Named graph on entity
@Entity
@NamedEntityGraph(
    name = "Order.full",
    attributeNodes = {
        @NamedAttributeNode("items"),
        @NamedAttributeNode(value = "customer", subgraph = "customer-address"),
    },
    subgraphs = @NamedSubgraph(name = "customer-address", attributeNodes = @NamedAttributeNode("address"))
)
public class Order { ... }

// Use in repository
@EntityGraph("Order.full")
Optional<Order> findById(Long id);

// Ad-hoc graph
@EntityGraph(attributePaths = {"items", "customer.address"})
List<Order> findByStatus(OrderStatus status);
```

---

## Detecting Slow Queries

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100  # log queries >100ms

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.stat: DEBUG
```

```java
// Programmatic statistics
Statistics stats = entityManagerFactory.unwrap(SessionFactory.class).getStatistics();
log.info("Queries: {}, slow: {}", stats.getQueryExecutionCount(), stats.getQueryExecutionMaxTime());
```
