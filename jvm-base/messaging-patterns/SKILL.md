---
name: "messaging-patterns"
description: "Kafka and RabbitMQ integration with Spring (spring-kafka, spring-amqp). Use when user works with message queues, event-driven architecture, consumers/producers, or asks about message delivery guarantees."
---

# Messaging Patterns Skill

Best practices for message-driven applications with Spring Kafka and Spring AMQP (RabbitMQ).

## Scope and Boundaries

- Use this skill for Kafka and RabbitMQ producer/consumer patterns, DLQ, idempotency, and transactional outbox.
- Use `spring-boot-patterns` for general service and transaction patterns.
- Use `performance-patterns` for connection pool and throughput tuning.
- Examples are pattern fragments; adapt serialization, topic names, and error handling to the current project.

## When to Use
- User is producing or consuming Kafka/RabbitMQ messages
- Questions about idempotency, retries, or dead-letter queues
- Event-driven architecture design
- Message serialization/deserialization issues
- Consumer group configuration

---

## Spring Kafka

### Producer

```java
// ✅ GOOD: Typed KafkaTemplate with error handling
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publish(OrderEvent event) {
        kafkaTemplate.send("orders", event.orderId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event: {}", event.orderId(), ex);
                } else {
                    log.debug("Published to partition {} offset {}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

```yaml
# application.yml — producer config
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all          # Wait for all replicas
      retries: 3
      properties:
        enable.idempotence: true   # Exactly-once semantics
        max.in.flight.requests.per.connection: 1
```

### Consumer

```java
// ✅ GOOD: Consumer with manual acknowledgment and error handling
@Component
@Slf4j
public class OrderEventConsumer {

    @KafkaListener(
        topics = "orders",
        groupId = "order-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            Acknowledgment ack) {
        try {
            processOrder(event);
            ack.acknowledge();
        } catch (RecoverableException e) {
            log.warn("Recoverable error for order {}, will retry", event.orderId(), e);
            // Don't ack — rethrow to let the error handler (e.g. DefaultErrorHandler)
            // manage retries and DLQ routing
            throw e;
        } catch (Exception e) {
            log.error("Non-recoverable error for order {}", event.orderId(), e);
            ack.acknowledge();  // Ack to skip poison pill; log for manual investigation
        }
    }
}
```

```yaml
# application.yml — consumer config
spring:
  kafka:
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.example.events"
    listener:
      ack-mode: MANUAL_IMMEDIATE
      concurrency: 3
```

### Dead Letter Queue (DLQ)

```java
// ✅ GOOD: Configure DLQ for failed messages
@Configuration
public class KafkaConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {

        var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
        factory.setConsumerFactory(consumerFactory);

        // Retry 3 times, then send to DLQ
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())),
            new FixedBackOff(1000L, 3L)
        ));

        return factory;
    }
}
```

### Kafka Streams (for event processing)

```java
@Configuration
@EnableKafkaStreams
public class OrderStreamProcessor {

    @Bean
    public KStream<String, OrderEvent> processOrders(StreamsBuilder builder) {
        return builder
            .stream("orders", Consumed.with(Serdes.String(), orderSerde()))
            .filter((key, event) -> event.status() == OrderStatus.PENDING)
            .mapValues(event -> event.withStatus(OrderStatus.PROCESSING))
            .peek((key, event) -> log.info("Processing order {}", key))
            .to("orders-processing", Produced.with(Serdes.String(), orderSerde()));
    }
}
```

---

## Spring AMQP (RabbitMQ)

### Configuration

```java
// ✅ GOOD: Declare exchanges, queues, and bindings as beans
@Configuration
public class RabbitMQConfig {

    public static final String ORDER_EXCHANGE = "orders.exchange";
    public static final String ORDER_QUEUE    = "orders.queue";
    public static final String ORDER_DLQ      = "orders.dlq";
    public static final String ORDER_ROUTING  = "orders.created";

    @Bean
    TopicExchange orderExchange() {
        return new TopicExchange(ORDER_EXCHANGE);
    }

    @Bean
    Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", ORDER_DLQ)
            .build();
    }

    @Bean
    Queue deadLetterQueue() {
        return QueueBuilder.durable(ORDER_DLQ).build();
    }

    @Bean
    Binding orderBinding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue).to(orderExchange).with(ORDER_ROUTING);
    }
}
```

### Producer

```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrderCreated(OrderEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_ROUTING,
            event,
            message -> {
                message.getMessageProperties().setMessageId(event.orderId());
                message.getMessageProperties().setContentType("application/json");
                return message;
            }
        );
    }
}
```

### Consumer

```java
@Component
@Slf4j
public class OrderEventConsumer {

    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE)
    public void consume(OrderEvent event, Channel channel,
                        @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
        try {
            processOrder(event);
            channel.basicAck(deliveryTag, false);
        } catch (RecoverableException e) {
            log.warn("Retry for order {}", event.orderId(), e);
            channel.basicNack(deliveryTag, false, true);  // requeue=true
        } catch (Exception e) {
            log.error("Non-recoverable for order {}", event.orderId(), e);
            channel.basicNack(deliveryTag, false, false); // send to DLQ
        }
    }
}
```

```yaml
# application.yml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: 5672
    username: ${RABBITMQ_USER:guest}
    password: ${RABBITMQ_PASS:guest}
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10           # Max unacked messages per consumer
        concurrency: 2
        max-concurrency: 5
```

---

## Common Patterns

### Idempotent Consumer

```java
// ✅ Always design consumers to be idempotent
@Service
@RequiredArgsConstructor
public class OrderProcessor {

    private final ProcessedEventRepository processedEvents;

    @Transactional
    public void process(OrderEvent event) {
        if (processedEvents.existsByEventId(event.eventId())) {
            log.debug("Duplicate event {}, skipping", event.eventId());
            return;
        }
        // ... process
        processedEvents.save(new ProcessedEvent(event.eventId()));
    }
}
```

### Transactional Outbox Pattern

```java
// ✅ Avoid dual-write: save event in the same transaction as domain change
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        // Store event in the same transaction — a background job publishes it
        outboxRepository.save(OutboxEvent.of("ORDER_CREATED", order));

        return order;
    }
}
```

---

## Quick Reference

| Concern | Kafka | RabbitMQ |
|---|---|---|
| Message ordering | Per partition (by key) | Per queue |
| Replay | Yes (configurable retention) | No (once consumed) |
| Routing | Topic/partition | Exchange → routing key |
| DLQ | `.DLT` topic convention | `x-dead-letter-exchange` arg |
| Idempotence | `enable.idempotence=true` | Manual dedup logic |

---

## Related Skills
- `spring-boot-patterns` - Service and transaction patterns
- `performance-patterns` - Connection pool and throughput tuning
