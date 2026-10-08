# Declarative HTTP Interfaces & RestClient in Spring Boot 3.x+

This reference provides production configurations, error handling strategies, and virtual thread compatibility guidelines for HTTP interfaces in modern Spring Boot microservices.

---

## 1. Declarative Client vs. Traditional Clients

| Feature | Legacy `RestTemplate` | Spring Cloud OpenFeign | Spring 6 HTTP Interfaces + `RestClient` |
| :--- | :--- | :--- | :--- |
| **Status** | Maintenance mode | Legacy / Maintenance | **Modern Standard (Spring 6 / Boot 3+)** |
| **Virtual Threads Ready** | No (blocking legacy I/O) | Partial | **Yes (100% Virtual Thread friendly)** |
| **Native Image (GraalVM)** | Complex reflection | Reflection heavy | **First-class native image support** |
| **Type Safety** | Low (manual template methods) | High (interface-based) | **High (declarative interface contract)** |
| **External Dependencies** | Built-in | Heavy `spring-cloud-openfeign` | **Zero external dependencies (core Spring)** |

---

## 2. Advanced Error Handling & RFC 7807 Mapping

When a downstream service returns an HTTP 4xx or 5xx with `application/problem+json`, the upstream `RestClient` must parse the `ProblemDetail` payload rather than throwing generic `HttpClientErrorException`:

```java
package com.example.order.infrastructure.client.inventory;

import org.springframework.http.HttpStatusCode;
import org.springframework.http.ProblemDetail;
import org.springframework.web.client.RestClient;

public class InventoryClientErrorHandler {

    public static RestClient.ResponseSpec applyCustomHandling(RestClient.ResponseSpec spec) {
        return spec
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                ProblemDetail problem = response.body(ProblemDetail.class);
                String detail = problem != null ? problem.getDetail() : "Downstream Client Error";
                throw new DownstreamClientException(response.getStatusCode(), detail, problem);
            })
            .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                throw new DownstreamServiceUnavailableException(
                    "Inventory service unavailable: HTTP " + response.getStatusCode()
                );
            });
    }
}
```

---

## 3. Virtual Thread Compatibility & Thread Pinning Prevention

In Spring Boot 3.2+ with Java 21, enabling virtual threads:
```yaml
spring:
  threads:
    virtual:
      enabled: true
```
Requires ensuring that the underlying HTTP client does not pin platform carrier threads:
1. **`JdkClientHttpRequestFactory` (Recommended Default)**:
   - Built on Java 11+ `java.net.http.HttpClient`.
   - Written entirely in modern Java with non-blocking virtual thread-friendly locking (`ReentrantLock`).
   - Does **NOT** pin carrier threads during I/O blocking.
   ```java
   @Bean
   public RestClient.Builder restClientBuilder() {
       var httpClient = java.net.http.HttpClient.newBuilder()
               .connectTimeout(Duration.ofSeconds(2))
               .version(java.net.http.HttpClient.Version.HTTP_2)
               .build();
       
       return RestClient.builder()
               .requestFactory(new JdkClientHttpRequestFactory(httpClient));
   }
   ```
2. **`HttpComponentsClientHttpRequestFactory` (Apache HttpClient 5)**:
   - Supported when advanced connection pooling, custom SSL contexts, or proxy routing is required.
   - HttpClient 5.3+ has resolved synchronized carrier thread pinning for HTTP/1.1 and HTTP/2 pipelines.

---

## 4. W3C Distributed Tracing Header Propagation

When `micrometer-tracing` is on the classpath, `RestClient.Builder` automatically registers an `ObservationClientHttpRequestInterceptor`. 

**Critical Invariant**:
Always inject `RestClient.Builder` into configuration classes rather than calling `RestClient.create()`. Calling `RestClient.create()` bypasses Spring Boot's auto-configured observation interceptors, losing distributed trace propagation (`traceparent` header).

```java
@ConfigurationProperties(prefix = "services.customer")
public record ClientProperties(String customerServiceUrl) {}

// ✅ CORRECT: TraceContext, metrics, and timeouts injected automatically
@Bean
public CustomerClient customerClient(RestClient.Builder builder, ClientProperties props) {
    RestClient restClient = builder
            .baseUrl(props.customerServiceUrl())
            .build();
    // ... proxy factory adapter
}

// ❌ WRONG: Loses traceparent propagation, metrics, and default timeouts!
RestClient restClient = RestClient.create("http://customer-service:8080");
```

```yaml
# application.yaml - mapped without default fallbacks
services:
  customer:
    customer-service-url: ${SERVICES_CUSTOMER_URL}
```

```bash
# .env.example - documented at repository root
SERVICES_CUSTOMER_URL=http://customer-service:8080
```
