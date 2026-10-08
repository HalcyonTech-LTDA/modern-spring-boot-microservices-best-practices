# Idempotent Consumers & Dead-Letter Topics (DLT) in Spring Kafka

This reference details message deduplication strategies, Spring Kafka `@RetryableTopic` choreography, Dead-Letter Topic (DLT) handling, and poison-pill recovery.

---

## 1. Idempotency Guard Implementation

### Database Deduplication Table Pattern
```sql
CREATE TABLE processed_message (
    message_id VARCHAR(128) PRIMARY KEY,
    consumer_group VARCHAR(64) NOT NULL,
    processed_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Idempotent Consumer Service
```java
package com.example.payment.infrastructure.kafka;

import com.example.payment.application.service.PaymentApplicationService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.RetryableTopic;
import org.springframework.kafka.retrytopic.DltStrategy;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.retry.annotation.Backoff;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class OrderCreatedEventListener {

    private static final Logger log = LoggerFactory.getLogger(OrderCreatedEventListener.class);
    private final PaymentApplicationService paymentService;
    private final JdbcClient jdbcClient;

    public OrderCreatedEventListener(PaymentApplicationService paymentService, JdbcClient jdbcClient) {
        this.paymentService = paymentService;
        this.jdbcClient = jdbcClient;
    }

    @RetryableTopic(
            attempts = "4",
            backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000),
            dltStrategy = DltStrategy.FAIL_ON_ERROR,
            include = { TransientNetworkException.class }
    )
    @KafkaListener(topics = "order.events", groupId = "payment-service")
    @Transactional
    public void onOrderCreated(
            @Payload OrderCreatedEvent payload,
            @Header(KafkaHeaders.RECEIVED_KEY) String messageKey,
            @Header(value = "messageId", required = false) String messageId
    ) {
        String deduplicationId = messageId != null ? messageId : messageKey;

        // Atomically attempt to register the processed message
        int inserted = jdbcClient.sql("""
                INSERT INTO processed_message (message_id, consumer_group)
                VALUES (:id, 'payment-service')
                ON CONFLICT (message_id) DO NOTHING
                """)
                .param("id", deduplicationId)
                .update();

        if (inserted == 0) {
            log.info("Duplicate event detected with ID: {}. Skipping execution safely.", deduplicationId);
            return;
        }

        // Execute core business logic
        paymentService.processPaymentForOrder(payload);
    }
}
```

---

## 2. Spring Kafka Non-Blocking Retry Architecture (`@RetryableTopic`)

Blocking consumer retries (`Thread.sleep()` in the listener) stall the partition and risk triggering consumer group rebalances.

### Non-Blocking Retry Mechanism
With `@RetryableTopic(attempts = "4")`:
```text
[ order.events ] (Main Topic - attempt 1)
       |
       +--- On Transient Failure ---> [ order.events-retry-1000 ] (Delay: 1s)
                                              |
                                              +--- On Transient Failure ---> [ order.events-retry-2000 ] (Delay: 2s)
                                                                                    |
                                                                                    +--- Final Failure ---> [ order.events-dlt ] (DLT)
```
- Other messages on `order.events` continue to be processed without head-of-line blocking.
- Each retry topic has an independent consumer with appropriate backoff delays.
- Poisoned messages arrive safely in `order.events-dlt`.

---

## 3. Consuming and Alerting on Dead-Letter Topics (DLT)

A dedicated DLT consumer processes failed messages for observability, manual replay, or alerting:

```java
package com.example.payment.infrastructure.kafka;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.DltHandler;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class PaymentDltHandler {

    private static final Logger log = LoggerFactory.getLogger(PaymentDltHandler.class);

    @DltHandler
    public void processDltMessage(
            @Payload String payload,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage,
            @Header(KafkaHeaders.EXCEPTION_STACKTRACE) String stackTrace
    ) {
        log.error("POISON PILL ROUTED TO DLT! Source Topic: {}, Error: {}, Payload: {}",
                topic, exceptionMessage, payload);
        
        // Publish alert to PagerDuty / Slack or write to dead-letter review dashboard
    }
}
```
