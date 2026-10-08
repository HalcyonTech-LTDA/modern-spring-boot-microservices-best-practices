---
name: modern-microservices-perimeter-gateway-and-security
description: >-
  Use this skill when configuring API gateways with Spring Cloud Gateway (MVC or Reactive), designing Backend-for-Frontend (BFF) architectures, implementing OAuth2/OIDC Token Relay, securing downstream Spring Boot microservices as stateless JWT Resource Servers, or establishing service-to-service mTLS.
---

# API Gateway, Perimeter Security & Token Relay

This skill guides the design, routing, rate limiting, and zero-trust security configuration of microservices using Spring Cloud Gateway and Spring Security Resource Server.

---

## 1. API Gateway Responsibilities & Boundaries

The API Gateway is the single public entry point into the microservices landscape. It acts as an architectural boundary between public clients and private internal networks:

### Core Responsibilities
- **Request Routing & Path Rewriting**: Forwarding external URL paths (e.g., `/api/v1/orders/**`) to internal service locations (`http://order-service:8080`).
- **Edge Security & Authentication**: Authenticating users via OAuth2/OIDC, terminating user sessions, and minting or forwarding cryptographically signed JWT tokens downstream.
- **Centralized Rate Limiting**: Protecting downstream microservices from denial of service and abusive traffic using Redis token bucket rate limiting.
- **CORS Management**: Enforcing uniform Cross-Origin Resource Sharing headers at the edge, removing CORS handling from individual microservices.
- **TLS Termination**: Offloading external TLS certificates and routing unencrypted traffic inside secure private VPC networks, or upgrading to mTLS for zero-trust internal traffic.

### Antipattern: The "God" Gateway
Never embed business logic, domain transformations, or heavy database access inside the API Gateway. Keep the gateway lean and focused exclusively on routing, security, and cross-cutting edge policies.

---

## 2. Spring Cloud Gateway Flavors

Spring Boot 3 supports two architectural flavors of Spring Cloud Gateway:
1. **Spring Cloud Gateway (Reactive / WebFlux)**:
   - High-concurrency, non-blocking Netty runtime.
   - Recommended for massive ingress traffic and thousands of concurrent streaming connections.
2. **Spring Cloud Gateway Server MVC (Virtual Threads)**:
   - Introduced in Spring Cloud 2023.0+ on top of Spring MVC and Java 21 Virtual Threads.
   - Simplifies development by avoiding reactive programming (`Mono`/`Flux`) while delivering high concurrency via virtual threads.

---

## 3. Perimeter Authentication & Token Relay Architecture

```text
[ Browser / Mobile Client ]
            |
            | 1. Request with Session / Bearer Token
            ▼
[ Spring Cloud Gateway ]  <--- Authenticates with Identity Provider (Keycloak / Okta / Auth0)
            |
            | 2. Injects validated JWT via TokenRelay filter
            ▼
[ Internal Virtual Network ]
            |
            +---- Authorization: Bearer <signed-jwt> ----> [ Order Microservice ] (Resource Server)
            |
            +---- Authorization: Bearer <signed-jwt> ----> [ Customer Microservice ] (Resource Server)
```

### Invariant Security Rules
- **No Anonymous Access Downstream**: Downstream microservices must reject any request lacking a valid `Authorization: Bearer <token>` header.
- **Stateless Verification**: Downstream Resource Servers do not consult the OAuth2 Identity Provider on every request. They validate JWTs locally using cached public keys (`jwk-set-uri`).
- **Scope & Role Enforcement**: Controllers enforce granular permissions using method security:
  ```java
  @PreAuthorize("hasAuthority('SCOPE_orders:write')")
  ```

---

## References

For complete deep-dive blueprints and production configurations, consult:
- [Spring Cloud Gateway Routing & Backend-for-Frontend (BFF) Pattern](./references/spring_cloud_gateway_and_bff_pattern.md): Spring Cloud Gateway route definitions, Redis rate limiting, CORS configuration, and BFF topology.
- [OAuth2 Token Relay & Stateless JWT Resource Server Security](./references/oauth2_token_relay_and_jwt_resource_server.md): TokenRelay filter setup, Spring Security Resource Server configuration, and custom JWT authentication converter.
