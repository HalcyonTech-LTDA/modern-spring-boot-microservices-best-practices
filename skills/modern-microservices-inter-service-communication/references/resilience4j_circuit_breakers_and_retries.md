# Resilience4j Circuit Breakers, Retries & Fallbacks

This reference provides production configurations, state transition mechanics, fallback choreography, and Micrometer metrics integration for Resilience4j in Spring Boot 3.x+.

---

## 1. Resilience4j Architecture & State Machine

The Circuit Breaker pattern prevents an application from repeatedly trying to execute an operation that is likely to fail:
```text
           [ CLOSED ]  <------------------------+
           (Normal)                             |
               |                                |
   Failure rate > threshold             Success rate met
               |                                |
               ▼                                |
            [ OPEN ] -------- Wait duration -> [ HALF-OPEN ]
        (Fast Failure)                       (Trial calls)
```

1. **Closed**: Requests pass through. Metrics (failure rate, slow call rate) are recorded over a sliding window.
2. **Open**: Requests fail fast immediately with `CallNotPermittedException`. No network calls hit downstream services. Fallback methods execute immediately.
3. **Half-Open**: After `wait-duration-in-open-state`, a limited number of trial calls pass through to probe downstream recovery.

---

## 2. Production `application.yaml` Configuration

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 50
        minimum-number-of-calls: 20
        failure-rate-threshold: 50.0
        slow-call-rate-threshold: 75.0
        slow-call-duration-threshold: 1500ms
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 10
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - org.springframework.web.client.ResourceAccessException
          - java.net.SocketTimeoutException
          - java.io.IOException
        ignore-exceptions:
          - com.example.common.exception.BusinessValidationException
    instances:
      inventoryService:
        base-config: default

  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 200ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        exponential-max-wait-duration: 2s
        randomize-wait-duration: true # Injects Jitter to prevent thundering herd
        retry-exceptions:
          - org.springframework.web.client.ResourceAccessException
          - java.net.SocketTimeoutException
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException # Never retry 4xx!
    instances:
      inventoryService:
        base-config: default

  timelimiter:
    configs:
      default:
        timeout-duration: 2s
        cancel-running-future: true
    instances:
      inventoryService:
        base-config: default
```

---

## 3. Decorating Service Invocations & Fallback Methods

```java
package com.example.order.infrastructure.adapter;

import com.example.order.domain.model.InventoryStatus;
import com.example.order.domain.port.outbound.InventoryCheckPort;
import com.example.order.infrastructure.client.inventory.InventoryClient;
import com.example.order.infrastructure.client.inventory.InventoryItemResponse;
import io.github.resilience4j.circuitbreaker.CallNotPermittedException;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class ResilientInventoryAdapter implements InventoryCheckPort {

    private static final Logger log = LoggerFactory.getLogger(ResilientInventoryAdapter.class);
    private final InventoryClient inventoryClient;

    public ResilientInventoryAdapter(InventoryClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }

    @Override
    @CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackCheckStock")
    @Retry(name = "inventoryService")
    public InventoryStatus checkStock(String sku) {
        InventoryItemResponse response = inventoryClient.checkStock(sku);
        return new InventoryStatus(response.sku(), response.availableQuantity() > 0);
    }

    /**
     * Fallback method MUST match target method signature + Throwable parameter
     */
    public InventoryStatus fallbackCheckStock(String sku, CallNotPermittedException ex) {
        log.warn("Inventory circuit breaker OPEN for SKU {}. Falling back to degraded mode.", sku);
        // Return conservative degraded response or consult read cache
        return new InventoryStatus(sku, false);
    }

    public InventoryStatus fallbackCheckStock(String sku, Exception ex) {
        log.error("Failed to check inventory for SKU {}. Error: {}", sku, ex.getMessage());
        return new InventoryStatus(sku, false);
    }
}
```

---

## 4. Micrometer Metrics & Actuator Integration

Resilience4j publishes comprehensive metrics via Micrometer:
- `resilience4j.circuitbreaker.state`: Current state (`0: CLOSED`, `1: OPEN`, `2: HALF_OPEN`).
- `resilience4j.circuitbreaker.calls`: Total call counter tagged by `kind` (successful, failed, not_permitted).
- `resilience4j.circuitbreaker.failure.rate`: Current percentage of failed calls.
- `resilience4j.retry.calls`: Retry attempt counts tagged by `kind` (successful_without_retry, successful_with_retry, failed_with_retry).

Expose circuit breaker health status in Spring Boot Actuator:
```yaml
management:
  health:
    circuitbreakers:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
```
Accessing `GET /actuator/health` displays the status of all registered circuit breakers, allowing Kubernetes readiness checks or alerting systems to identify degraded dependencies immediately.
