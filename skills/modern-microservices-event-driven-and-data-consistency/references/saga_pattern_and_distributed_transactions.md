# The Saga Pattern & Distributed Transaction Management

This reference details the implementation of distributed transactions across microservices using the Saga Pattern, contrasting Choreography vs. Orchestration, and defining compensating transactions.

---

## 1. Why Distributed 2PC/XA Is Banned in Microservices

Two-Phase Commit (2PC) relies on a centralized transaction manager locking database records across microservices:
1. **Blocking Latency**: Locks are held across multiple network roundtrips. Any slow network hop causes widespread locking cascades.
2. **Single Point of Failure**: If the coordinator crashes between the prepare and commit phases, participating resources remain locked indefinitely.
3. **Impedance Mismatch with Polyglot Storage**: Modern datastores (MongoDB, DynamoDB, Kafka, Redis) do not support the XA protocol.

---

## 2. Choreography-Based Saga (Event-Driven)

In a Choreography Saga, microservices publish and subscribe to domain events without a central coordinator.

```text
[ Order Service ]       [ Payment Service ]       [ Inventory Service ]
       |                         |                          |
       |--- OrderCreated ------->|                          |
       |                         |--- PaymentCaptured ----->|
       |                         |                          |--- InventoryReserved --> [ Order Confirmed ]
       |                         |                          |
       |                         |                          |--- OutOfStock --------->
       |                         |<-- RefundPayment --------|
       |<-- OrderCancelled ------|
```

### When to Choose Choreography
- Simple business processes with 2 to 4 participating services.
- Highly event-driven, loosely coupled ecosystems.
- Low operational overhead (no orchestrator infrastructure to maintain).

---

## 3. Orchestration-Based Saga (Command-Driven)

In an Orchestration Saga, a dedicated orchestrator service manages the workflow state machine, explicitly issuing commands and listening for replies.

```java
package com.example.order.saga;

import org.springframework.stereotype.Component;

@Component
public class OrderFulfillmentSagaOrchestrator {

    private final PaymentCommandSender paymentSender;
    private final InventoryCommandSender inventorySender;
    private final OrderSagaRepository sagaRepository;

    public OrderFulfillmentSagaOrchestrator(
            PaymentCommandSender paymentSender,
            InventoryCommandSender inventorySender,
            OrderSagaRepository sagaRepository
    ) {
        this.paymentSender = paymentSender;
        this.inventorySender = inventorySender;
        this.sagaRepository = sagaRepository;
    }

    public void startSaga(OrderSagaState state) {
        state.setStatus(SagaStatus.PAYMENT_PENDING);
        sagaRepository.save(state);
        paymentSender.sendAuthorizePaymentCommand(state.getOrderId(), state.getTotalAmount());
    }

    public void onPaymentAuthorized(PaymentAuthorizedReply reply) {
        OrderSagaState state = sagaRepository.findByOrderId(reply.orderId());
        state.setStatus(SagaStatus.INVENTORY_PENDING);
        sagaRepository.save(state);
        inventorySender.sendReserveInventoryCommand(state.getOrderId(), state.getItems());
    }

    public void onInventoryReservationFailed(InventoryFailedReply reply) {
        OrderSagaState state = sagaRepository.findByOrderId(reply.orderId());
        state.setStatus(SagaStatus.COMPENSATING);
        sagaRepository.save(state);

        // TRIGGER COMPENSATING TRANSACTIONS
        paymentSender.sendRefundPaymentCommand(state.getOrderId(), state.getTotalAmount());
    }

    public void onPaymentRefunded(PaymentRefundedReply reply) {
        OrderSagaState state = sagaRepository.findByOrderId(reply.orderId());
        state.setStatus(SagaStatus.FAILED);
        sagaRepository.save(state);
    }
}
```

---

## 4. Invariant Rules for Compensating Transactions

1. **Idempotency**: A compensating transaction (e.g., `RefundPayment`) may be invoked multiple times due to network retries. It MUST be strictly idempotent.
2. **Cannot Fail Due to Business Rules**: A compensating transaction must never be rejected by business validation (e.g., cannot fail refund because account is in a certain state). If it encounters a technical error, it must be retried until it succeeds.
3. **Semantic Inversion**: If an operation reserved funds, compensation releases them. If an operation dispatched an email, compensation sends a follow-up cancellation email.
4. **Audit Logging**: Every state transition in a Saga MUST be durably recorded with timestamps and correlation IDs for troubleshooting and compliance.
