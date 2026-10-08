---
name: modern-microservices-inter-service-communication
description: >-
  Use this skill when implementing synchronous RPC communication between Spring Boot microservices: declarative HTTP interfaces (@HttpExchange), RestClient configuration, connection pool tuning, Resilience4j fault tolerance (CircuitBreaker, Retry, RateLimiter, Bulkhead, TimeLimiter), fallback mechanisms, or high-performance gRPC.
---

# Microservices Synchronous Inter-Service Communication & Fault Tolerance

This skill guides the implementation of resilient synchronous inter-service communication across Spring Boot 3.x+ microservices using modern Declarative HTTP Interfaces, fluent `RestClient`, and Resilience4j fault tolerance patterns.

---

## 1. Modern Synchronous Communication Standards

### Ban on Legacy Clients
- **Legacy `RestTemplate`**: In maintenance mode as of Spring Framework 5/6. Banned for greenfield microservices.
- **Spring Cloud OpenFeign**: Heavy legacy dependency, slower startup, reflection-based, and lagging behind Spring Boot 3's native AOT / Virtual Thread features. Banned in favor of native Spring 6 HTTP Interfaces.

### Modern Baseline: Declarative HTTP Interfaces (`@HttpExchange`)
Spring Framework 6 and Spring Boot 3 introduce first-class Declarative HTTP Interfaces:
1. Define a Java interface annotated with `@HttpExchange`, `@GetExchange`, `@PostExchange`, `@PutExchange`, `@DeleteExchange`.
2. Configure a fluent `RestClient` bean with customized timeouts and connection pools.
3. Adapt the `RestClient` to the interface using `HttpServiceProxyFactory`.

---

## 2. Declarative HTTP Interface Setup Recipe

```java
package com.example.order.infrastructure.client.inventory;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;
import java.util.List;

@HttpExchange("/api/v1/inventory")
public interface InventoryClient {

    @GetExchange("/items/{sku}")
    InventoryItemResponse checkStock(@PathVariable("sku") String sku);

    @PostExchange("/reserve")
    ReservationResponse reserveStock(ReserveStockRequest request);
}
```

### Factory Configuration with Connection Pool & Timeouts
```java
package com.example.order.infrastructure.client.inventory;

import org.apache.hc.client5.http.impl.classic.CloseableHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManagerBuilder;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

import java.time.Duration;

@Configuration(proxyBeanMethods = false)
public class InventoryClientConfig {

    /**
     * Invariant: Never use @Value for properties. Always bind external configuration
     * to immutable Java records via @ConfigurationProperties.
     */
    @ConfigurationProperties(prefix = "services.inventory")
    public record InventoryClientProperties(String baseUrl) {}

    @Bean
    public InventoryClient inventoryClient(
            RestClient.Builder builder,
            InventoryClientProperties properties
    ) {
        // Production Apache HttpClient 5 Connection Manager with strict pools
        var connectionManager = PoolingHttpClientConnectionManagerBuilder.create()
                .setMaxConnTotal(200)
                .setMaxConnPerRoute(50)
                .build();

        CloseableHttpClient httpClient = HttpClients.custom()
                .setConnectionManager(connectionManager)
                .build();

        var requestFactory = new HttpComponentsClientHttpRequestFactory(httpClient);
        requestFactory.setConnectTimeout(Duration.ofMillis(1000));
        requestFactory.setConnectionRequestTimeout(Duration.ofMillis(500));

        RestClient restClient = builder
                .baseUrl(properties.baseUrl())
                .requestFactory(requestFactory)
                .defaultHeader("Accept", "application/json")
                .defaultHeader("User-Agent", "order-service/1.0")
                .build();

        var adapter = RestClientAdapter.create(restClient);
        var factory = HttpServiceProxyFactory.builderFor(adapter).build();

        return factory.createClient(InventoryClient.class);
    }
}
```

```yaml
# application.yaml - Invariant: No default fallback values in placeholders!
services:
  inventory:
    base-url: ${SERVICES_INVENTORY_BASE_URL}
```

---

## 3. Resilience4j Fault Tolerance Choreography

Every network boundary is subject to latency spikes, packet loss, and downstream outages. Resilience4j patterns prevent cascading system collapse:

### Pattern Matrix
| Pattern | Purpose | When to Apply |
| :--- | :--- | :--- |
| **Circuit Breaker** (`@CircuitBreaker`) | Trips open after repeated failures, shedding load and failing fast. | All synchronous downstream RPC calls. |
| **Time Limiter** (`@TimeLimiter`) | Cancels calls exceeding max duration, preventing thread hogging. | Every synchronous network interaction. |
| **Retry** (`@Retry`) | Re-executes requests on transient network or 503 errors with backoff. | Idempotent queries (GET) only; never non-idempotent mutations without idempotency keys. |
| **Rate Limiter** (`@RateLimiter`) | Throttles invocations to match downstream SLA limits. | Consuming rate-limited third-party partner APIs. |
| **Bulkhead** (`@Bulkhead`) | Isolates concurrent executions into bounded pools/semaphores. | Preventing slow downstream endpoints from saturating all worker threads. |

### Configuration Rules
- **Exponential Backoff with Jitter**: Avoid thundering herds by adding randomized jitter to retry intervals.
- **Selective Retries**: Never retry HTTP 4xx (Bad Request, Unauthorized, Not Found, Unprocessable Entity). Only retry transient 5xx or `ResourceAccessException` / `SocketTimeoutException`.
- **Meaningful Fallbacks**: Return cached data, default degraded responses, or queue for async processing. Never hide critical errors behind empty success responses.

---

## References

For complete deep-dive blueprints and advanced configuration recipes, consult:
- [Declarative HTTP Interfaces & RestClient in Spring Boot 3.x+](./references/declarative_http_interfaces_and_restclient.md): Complete HTTP interface configuration, error mapping to RFC 7807, and Virtual Thread safety.
- [Resilience4j Circuit Breakers, Retries & Fallbacks](./references/resilience4j_circuit_breakers_and_retries.md): Resilience4j production YAML configs, state transitions, fallback methods, and Micrometer metrics.
