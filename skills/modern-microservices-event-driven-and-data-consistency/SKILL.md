---
name: modern-microservices-event-driven-and-data-consistency
description: >-
  Use this skill when designing asynchronous messaging, event-driven architectures with Apache Kafka, implementing the Transactional Outbox pattern with Debezium CDC, handling idempotent message consumption, managing Dead-Letter Topics (DLT), or orchestrating distributed transactions with the Saga pattern.
---

# Event-Driven Architecture, Outbox Pattern & Distributed Consistency

This skill guides the design and implementation of event-driven microservices, asynchronous messaging with Spring for Apache Kafka, the Transactional Outbox pattern, idempotent message consumers, and distributed transactions using Sagas.

---

## 1. The Dual-Write Problem & The Outbox Pattern

### The Fatal Dual-Write Antipattern
In microservices, updating a local database and publishing a message to a broker in a single method is fundamentally broken:
```java
// ❌ FATAL ANTIPATTERN: The Dual-Write Trap
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);               // DB write succeeds
    kafkaTemplate.send("order-created", order); // Network failure or app crashes!
    // Result: Database has order, but no event was published. System is permanently inconsistent!
}
```
If you reverse the order (publish first, then save to DB), the DB write might fail due to a constraint violation after the message was already broadcast to the world.

### The Solution: Transactional Outbox Pattern
1. Write the business entity AND an event record into an `outbox` table in the **same atomic local database transaction**.
2. A dedicated publisher process (Change Data Capture via Debezium or a transactional polling worker) reads the `outbox` table and publishes events to Apache Kafka.
3. Once published, the event is marked processed or deleted.
4. **Guarantees**: At-least-once message delivery without distributed Two-Phase Commit (2PC / XA).

---

## 2. Spring for Apache Kafka Standards

### Publisher Configuration
- **Idempotent Producer**: Set `enable.idempotence=true` (Spring Boot default since Kafka 3.0) to prevent duplicate messages during producer retries.
- **Acks All**: Set `acks=all` (`-1`) to ensure all in-sync replicas (ISRs) acknowledge message persistence before returning success.
- **Message Keys**: Always provide a meaningful partition key (e.g., `customerId` or `orderId`) to guarantee strict ordering for related events within the same partition.

### Consumer Configuration
- **Explicit Acknowledgment**: Prefer `AckMode.MANUAL_IMMEDIATE` or `RECORD` to ensure messages are acknowledged only after business processing succeeds.
- **Virtual Thread Concurrency**: Enable virtual threads on `@KafkaListener` containers to scale IO-bound event consumers efficiently without massive thread pool overhead.
- **Non-Blocking Retries & Dead-Letter Topics**: Never block the consumer partition on poisoned pills or unhandled exceptions. Use Spring Kafka's `@RetryableTopic` to route transient errors through backoff retry topics to a Dead-Letter Topic (`<topic>.DLT`).

---

## 3. Idempotent Consumer Discipline

Because the Outbox pattern and message brokers provide **at-least-once delivery**, consumers MUST anticipate receiving duplicate messages:
- **Deduplication via Unique Key**: Track processed `message_id` or `idempotency_key` in a dedicated deduplication table or Redis set.
- **Naturally Idempotent Operations**: Design domain operations to be state-based rather than incremental (e.g., `SET status = 'PAID'` instead of `DECREMENT balance`).
- **Conditional Updates**: Apply optimistic locking with version checks: `UPDATE account SET balance = :val, version = version + 1 WHERE id = :id AND version = :expectedVersion`.

---

## 4. The Saga Pattern for Distributed Workflows

Distributed ACID transactions across microservices fail because:
- Two-Phase Commit (2PC / XA) holds locks across network boundaries, causing extreme latency and vulnerability to coordinator crashes.
- Microservices using different databases (Postgres, MongoDB, Redis) cannot participate in uniform XA transactions.

### Saga Strategy Selection
| Strategy | Description | Best For | Trade-offs |
| :--- | :--- | :--- | :--- |
| **Choreography** | Services react to domain events published by peers. No centralized coordinator. | Simple workflows (2-4 services), loosely coupled domains. | Difficult to trace complete workflow state; risk of cyclic dependencies. |
| **Orchestration** | A central Saga Orchestrator coordinates steps, issues commands, and listens for replies. | Complex workflows (5+ steps), high-audit financial processes, branching logic. | Orchestrator service can become a bottleneck if not kept lean. |

### Compensating Transactions
Every forward action in a Saga MUST define an idempotent compensating action (`undo`):
- Forward: `ReserveCreditCommand` -> Compensating: `ReleaseCreditCommand`.
- Forward: `ReserveInventoryCommand` -> Compensating: `ReleaseInventoryCommand`.

---

## References

For complete deep-dive blueprints and production code implementations, consult:
- [Transactional Outbox Pattern & CDC Implementation](./references/transactional_outbox_and_cdc_pattern.md): Outbox table DDL, atomic JPA/JdbcClient persistence, Debezium CDC configuration, and polling fallback.
- [Idempotent Consumers & Dead-Letter Topics (DLT) in Spring Kafka](./references/idempotent_consumer_and_dead_letter_topics.md): Spring `@RetryableTopic`, Dead-Letter Topic architecture, and Redis/DB idempotency guards.
- [The Saga Pattern & Distributed Transaction Management](./references/saga_pattern_and_distributed_transactions.md): State-machine-based Saga Orchestrator implementation with Spring Boot and compensating transaction handling.
