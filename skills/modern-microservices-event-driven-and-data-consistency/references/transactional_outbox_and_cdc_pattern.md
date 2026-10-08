# Transactional Outbox Pattern & CDC Implementation

This reference details the Transactional Outbox pattern, database schema design, atomic persistence in Spring Boot 3.x+, and event publishing via Change Data Capture (Debezium) or a polling worker.

---

## 1. Outbox Database Schema (PostgreSQL Example)

```sql
CREATE TABLE outbox_event (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(128) NOT NULL,
    payload JSONB NOT NULL,
    trace_id VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_outbox_unprocessed ON outbox_event(created_at) WHERE processed_at IS NULL;
```

---

## 2. Atomic Persistence with Clean Architecture & Outbound Port

To uphold Screaming Clean Architecture and the zero framework leakage invariant, the application layer remains pure Java with zero framework annotations (`@Service`, `@Transactional`, `@Entity`). It communicates with the database and outbox through outbound ports:

### Outbound Port (`domain.port.outbound`)
```java
package com.example.order.domain.port.outbound;

import com.example.order.domain.model.DomainEvent;

public interface DomainEventOutboxPort {
    void append(DomainEvent event);
}
```

### Pure Application Service (`application.service`)
```java
package com.example.order.application.service;

import com.example.order.domain.model.Order;
import com.example.order.domain.model.OrderCreatedEvent;
import com.example.order.domain.port.outbound.DomainEventOutboxPort;
import com.example.order.domain.port.outbound.OrderRepository;

import java.time.Clock;

public class CreateOrderUseCase {

    private final OrderRepository orderRepository;
    private final DomainEventOutboxPort outboxPort;
    private final Clock clock;

    public CreateOrderUseCase(
            OrderRepository orderRepository,
            DomainEventOutboxPort outboxPort,
            Clock clock
    ) {
        this.orderRepository = orderRepository;
        this.outboxPort = outboxPort;
        this.clock = clock;
    }

    public Order execute(CreateOrderCommand command) {
        // 1. Business domain state mutation
        Order order = Order.create(command.customerId(), command.items());
        orderRepository.save(order);

        // 2. Domain event creation with injected clock for deterministic testing
        OrderCreatedEvent event = new OrderCreatedEvent(
                order.getId(),
                order.getCustomerId(),
                order.getTotalAmount(),
                clock.instant()
        );

        // 3. Atomically stage event in the outbox
        outboxPort.append(event);

        return order;
    }
}
```

### Infrastructure Outbox Adapter (`infrastructure.outbox`)
```java
package com.example.order.infrastructure.outbox;

import com.example.order.domain.model.DomainEvent;
import com.example.order.domain.port.outbound.DomainEventOutboxPort;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.micrometer.tracing.Tracer;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.util.UUID;

@Component
public class JpaOutboxEventAdapter implements DomainEventOutboxPort {

    private final OutboxEventJpaRepository outboxRepository;
    private final ObjectMapper objectMapper;
    private final Tracer tracer;
    private final Clock clock;

    public JpaOutboxEventAdapter(
            OutboxEventJpaRepository outboxRepository,
            ObjectMapper objectMapper,
            Tracer tracer,
            Clock clock
    ) {
        this.outboxRepository = outboxRepository;
        this.objectMapper = objectMapper;
        this.tracer = tracer;
        this.clock = clock;
    }

    @Override
    @Transactional
    public void append(DomainEvent event) {
        try {
            String payloadJson = objectMapper.writeValueAsString(event);
            String currentTraceId = tracer.currentSpan() != null ? tracer.currentSpan().context().traceId() : null;

            OutboxEventEntity outboxRecord = new OutboxEventEntity(
                    UUID.randomUUID(),
                    event.aggregateType(),
                    event.aggregateId(),
                    event.eventType(),
                    payloadJson,
                    currentTraceId,
                    clock.instant(),
                    null
            );
            outboxRepository.save(outboxRecord);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize Outbox event", e);
        }
    }
}
```

---

## 3. Option A: Change Data Capture (Debezium + Kafka Connect)

Change Data Capture (CDC) directly monitors the PostgreSQL Write-Ahead Log (WAL) or MySQL binlog:
- **Zero Query Overhead**: No continuous `SELECT ... FROM outbox_event` polling queries hitting the primary database.
- **Sub-Second Latency**: Events stream to Kafka within milliseconds of the database transaction committing.
- **Debezium Outbox Event Router**: Automatically unwraps the payload, routing to dynamic Kafka topics based on `aggregate_type` or `event_type`.

### Debezium PostgreSQL Connector Configuration
```json
{
  "name": "order-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "plugin.name": "pgoutput",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "${file:/secrets/db-credentials.properties:password}",
    "database.dbname": "order_db",
    "table.include.list": "public.outbox_event",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.route.topic.replacement": "${routedByValue}.events",
    "transforms.outbox.table.fields.additional.placement": "trace_id:header:traceparent"
  }
}
```

---

## 4. Option B: Scheduled Polling Worker (Lightweight Fallback)

For architectures without Kafka Connect infrastructure, a scheduled background worker can poll unprocessed events:

```java
package com.example.order.infrastructure.outbox;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.util.List;

@Component
public class OutboxPollingPublisher {

    private static final Logger log = LoggerFactory.getLogger(OutboxPollingPublisher.class);
    private final OutboxEventJpaRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final Clock clock;

    public OutboxPollingPublisher(
            OutboxEventJpaRepository outboxRepository,
            KafkaTemplate<String, String> kafkaTemplate,
            Clock clock
    ) {
        this.outboxRepository = outboxRepository;
        this.kafkaTemplate = kafkaTemplate;
        this.clock = clock;
    }

    // Invariant: Placeholders in Java must reference property keys without default fallback values
    @Scheduled(fixedDelayString = "${app.outbox.polling-interval}")
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEventEntity> pendingEvents = outboxRepository.findTop50ByProcessedAtIsNullOrderByCreatedAtAsc();

        for (OutboxEventEntity event : pendingEvents) {
            try {
                String targetTopic = event.getAggregateType().toLowerCase() + ".events";
                kafkaTemplate.send(targetTopic, event.getAggregateId(), event.getPayload()).get();
                
                event.setProcessedAt(clock.instant());
                outboxRepository.save(event);
            } catch (Exception ex) {
                log.error("Failed to publish outbox event id: {}. Will retry on next poll.", event.getId(), ex);
                break; // Break loop to maintain event ordering
            }
        }
    }
}
```

### Externalized Configuration (`application.yaml` & `.env.example`)
```yaml
# application.yaml - mapped directly to env var without defaults
app:
  outbox:
    polling-interval: ${APP_OUTBOX_POLLING_INTERVAL}
```

```bash
# .env.example - documented at repository root
APP_OUTBOX_POLLING_INTERVAL=500ms
```
