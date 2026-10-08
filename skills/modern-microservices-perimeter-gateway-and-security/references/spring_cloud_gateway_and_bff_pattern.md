# Spring Cloud Gateway Routing & Backend-for-Frontend (BFF) Pattern

This reference provides production configurations, Redis-backed rate limiting, route definitions, and architectural guidelines for the Backend-for-Frontend (BFF) pattern.

---

## 1. Spring Cloud Gateway Configuration (`application.yaml`)

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials, RETAIN_FIRST
        - TokenRelay=
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 50
            redis-rate-limiter.burstCapacity: 100
            redis-rate-limiter.requestedTokens: 1
            key-resolver: "#{@userKeyResolver}"
      routes:
        - id: order-service-route
          uri: http://order-service:8080
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=0
            - name: CircuitBreaker
              args:
                name: orderGatewayCircuitBreaker
                fallbackUri: forward:/fallback/orders

        - id: customer-service-route
          uri: http://customer-service:8080
          predicates:
            - Path=/api/v1/customers/**
          filters:
            - StripPrefix=0

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins:
              - "https://app.example.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders:
              - Authorization
              - Content-Type
              - X-Correlation-ID
            allowCredentials: true
            maxAge: 3600
```

---

## 2. Redis Key Resolver for Rate Limiting

Rate limiting can be resolved by authenticated user identity or client IP address:

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.core.context.SecurityContext;
import reactor.core.publisher.Mono;

@Configuration(proxyBeanMethods = false)
public class RateLimiterConfig {

    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> ReactiveSecurityContextHolder.getContext()
                .map(SecurityContext::getAuthentication)
                .filter(Authentication::isAuthenticated)
                .map(Authentication::getName)
                .switchIfEmpty(Mono.justOrEmpty(
                        exchange.getRequest().getRemoteAddress() != null
                                ? exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
                                : "anonymous"
                ));
    }
}
```

---

## 3. The Backend-for-Frontend (BFF) Pattern

Instead of maintaining a massive single API gateway serving desktop browsers, mobile apps, and IoT devices simultaneously:

```text
[ Web Desktop App ]   [ Mobile iOS/Android ]   [ B2B Partner API ]
         |                       |                       |
         ▼                       ▼                       ▼
   [ Web BFF ]             [ Mobile BFF ]          [ Partner BFF ]
  (Cookie session)        (Short-lived JWT)       (API Key + mTLS)
         \                       |                       /
          +----------------------+----------------------+
                                 |
                     [ Internal Microservices ]
                     - Order Service
                     - Catalog Service
                     - Billing Service
```

### Benefits of the BFF Pattern
- **Tailored Payload Sizing**: The Mobile BFF prunes verbose JSON fields to conserve cellular bandwidth; the Web BFF returns richer payloads for widescreen displays.
- **Protocol Optimization**: Mobile BFF can optimize using HTTP/2 Multiplexing or Protobuf, while Web BFF speaks standard JSON.
- **Independent Team Ownership**: The frontend web team owns `Web BFF`; the mobile team owns `Mobile BFF`. Changes do not require cross-team deployment scheduling.
