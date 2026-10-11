---
description: >-
  Automatically enforces modern Microservices architecture and modern Spring Boot 3.x+ engineering standards:
  service boundary & database-per-service isolation, declarative HTTP interfaces with RestClient, Resilience4j fault tolerance,
  event-driven messaging with transactional Outbox & Sagas, Spring Cloud Gateway & OAuth2/JWT security,
  distributed tracing with Micrometer/OTel, and contract-driven testing.
trigger: model_decision
---

# Modern Microservices with Spring Boot Guardrails & Skill Router

When architecting, developing, reviewing, refactoring, or testing microservices using modern Spring Boot and Java 21+, strictly adhere to Modern Microservices Engineering Standards and the corresponding specialized skills:

| When working on... | Consult & Activate Skill | Key Focus Areas |
| :--- | :--- | :--- |
| **Service Boundaries & DDD Modeling** | `modern-microservices-architecture-and-decomposition` | DDD Bounded Contexts, database-per-service strict isolation, Anti-Corruption Layer (ACL), and API-First contract design (OpenAPI 3.1). |
| **Synchronous RPC & Inter-Service Calls** | `modern-microservices-inter-service-communication` | Declarative HTTP Interfaces (`@HttpExchange`) with `RestClient`, connection pool tuning, Resilience4j fault tolerance, and gRPC. |
| **Asynchronous Events & Data Consistency** | `modern-microservices-event-driven-and-data-consistency` | Spring for Apache Kafka, Transactional Outbox pattern with Debezium CDC, Idempotent Consumers, Dead-Letter Topics (`.DLT`), and Sagas. |
| **Edge Gateways & Perimeter Security** | `modern-microservices-perimeter-gateway-and-security` | Spring Cloud Gateway, BFF patterns, Redis rate limiting, OAuth2/OIDC Token Relay filter, stateless JWT Resource Servers, and mTLS. |
| **Observability, Operations & Testing** | `modern-microservices-observability-and-resilience-ops` | Distributed Tracing (Micrometer Tracing/OTel), JSON logging with MDC, Kubernetes probes, graceful shutdown, and Contract testing. |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Strict Datastore & Boundary Isolation (Database-per-Service)**: Every microservice must privately own its datastore. Never share databases, tables, or cross-service `JOIN`s. Multi-service distributed workflows must use Sagas (strictly ban distributed 2PC/XA transactions).
2. **Declarative HTTP Interfaces with Mandatory Resilience**: Synchronous inter-service communication must use Declarative HTTP Interfaces (`@HttpExchange`) on `RestClient` (prohibit legacy `RestTemplate` and OpenFeign). All network calls must be wrapped with Resilience4j timeouts and circuit breakers.
3. **Transactional Outbox & Idempotent Asynchronous Messaging**: Prohibit application-level dual writes. State changes and event publishing must be coordinated atomically via the Transactional Outbox pattern. Message consumers must be idempotent and route unrecoverable failures to Dead-Letter Topics (`.DLT`).
4. **Perimeter Authentication, BFF & Stateless Resource Servers**: External traffic must ingress via an API Gateway or Backend-for-Frontend (BFF) utilizing OAuth2 Token Relay. Downstream microservices must operate as stateless JWT Resource Servers validating signatures locally.
5. **Distributed Tracing & W3C TraceContext Propagation**: All synchronous and asynchronous communications must automatically propagate W3C TraceContext headers (`traceparent`). Applications must output structured JSON logs with MDC-injected `traceId` and `spanId`.
6. **Cloud-Native Lifecycle, Configuration Hygiene & Contract Testing**: Enable Kubernetes liveness/readiness probes with graceful shutdown. Bind configuration strictly to immutable records without default placeholder fallbacks (`.env.example` discipline). Verify inter-service contracts via Consumer-Driven Contracts (Pact/Spring Cloud Contract) and virtualize dependencies with WireMock.

Consult the corresponding specialized skills for comprehensive architectural patterns, code blueprints, configuration templates, and testing blueprints.
