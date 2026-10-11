---
description: >-
  Automatically enforces modern Microservices architecture and modern Spring Boot 3.x+ engineering standards:
  service boundary & database-per-service isolation, declarative HTTP interfaces with RestClient, Resilience4j fault tolerance,
  event-driven messaging with transactional Outbox & Sagas, Spring Cloud Gateway & OAuth2/JWT security,
  distributed tracing with Micrometer/OTel, and contract-driven testing.
trigger: model_decision
---

# Modern Microservices with Spring Boot Guardrails & Skill Router

When architecting, developing, reviewing, refactoring, or testing microservices using modern Spring Boot (3.2+, 3.3+, 3.4+) and Java 21+, strictly adhere to Modern Microservices Engineering Standards and the corresponding specialized skills:

| When working on... | Consult & Activate Skill | Key Focus Areas |
| :--- | :--- | :--- |
| **Service Boundaries & DDD Modeling** | `modern-microservices-architecture-and-decomposition` | Domain-Driven Design (DDD) Bounded Contexts, Business Capability decomposition, database-per-service strict isolation, Anti-Corruption Layer (ACL), eliminating distributed monoliths, and API-First contract design (OpenAPI 3.1). |
| **Synchronous RPC & Inter-Service Calls** | `modern-microservices-inter-service-communication` | Declarative HTTP Interfaces (`@HttpExchange`) backed by fluent `RestClient`, connection pool tuning, Resilience4j fault tolerance (Circuit Breakers, Timeouts, Rate Limiters, Bulkheads, Retries with jitter), and binary RPC via Spring gRPC / Protobuf. |
| **Asynchronous Events & Data Consistency** | `modern-microservices-event-driven-and-data-consistency` | Spring for Apache Kafka (`KafkaTemplate`, `@KafkaListener`), Transactional Outbox pattern with Debezium CDC, Idempotent Consumers, Dead-Letter Topics (`.DLT`), Saga pattern (Orchestration vs Choreography), and schema registry evolution. |
| **Edge Gateways & Perimeter Security** | `modern-microservices-perimeter-gateway-and-security` | Spring Cloud Gateway (MVC & Reactive), Backend-for-Frontend (BFF) patterns, centralized rate limiting (Redis), OAuth2/OIDC Token Relay filter, downstream Spring Security Stateless JWT Resource Servers, and service-to-service mTLS. |
| **Observability, Operations & Testing** | `modern-microservices-observability-and-resilience-ops` | Distributed Tracing with Micrometer Tracing & OpenTelemetry (W3C `traceparent` propagation), JSON structured logging with MDC (`traceId`, `spanId`), Kubernetes liveness/readiness probes, graceful shutdown, and Consumer-Driven Contract testing (Pact / Spring Cloud Contract) with WireMock. |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Absolute Ban on Shared Databases (Database-per-Service Mandate)**:
   - Every microservice MUST own its datastore privately. Never share database instances, schemas, or tables across microservice boundaries.
   - Cross-service database queries (`JOIN`s across schemas) and foreign key constraints pointing to external service tables are strictly prohibited.
   - Microservices MUST share data exclusively through published public APIs (synchronous) or domain events (asynchronous).

2. **Declarative HTTP Interfaces & Modern `RestClient` over Legacy Clients**:
   - For synchronous inter-service HTTP communication, use Spring 6 / Boot 3+ Declarative HTTP Interfaces (`@HttpExchange`, `@GetExchange`, `@PostExchange`) configured on top of `RestClient`.
   - Never use the legacy `RestTemplate` (in maintenance mode) or deprecated Spring Cloud OpenFeign for greenfield microservices.
   - Always configure explicit connection and read timeouts on the underlying HTTP client (`JdkClientHttpRequestFactory` or `HttpComponentsClientHttpRequestFactory`).

3. **Mandatory Fault Tolerance & Circuit Breaking with Resilience4j**:
   - Never make synchronous network calls across microservice boundaries without explicit timeout, circuit breaker, and retry boundaries (using Resilience4j or Spring Cloud CircuitBreaker).
   - Downstream client calls MUST define fallback methods or graceful degradation mechanisms. Never allow a downstream service outage to cascade into thread exhaustion or unhandled 500 errors in upstream services.
   - Retries MUST use exponential backoff with jitter and apply solely to idempotent, transient failure codes (e.g., HTTP 502/503/504, connection resets; never HTTP 400/404/422).

4. **Absolute Ban on Dual Writes (Mandatory Transactional Outbox Pattern)**:
   - Never write to a local database and publish to a message broker (e.g., Kafka, RabbitMQ) inside the same application method without the Transactional Outbox pattern.
   - State mutation and outgoing event records MUST be persisted in the same local ACID database transaction into an `outbox` table.
   - Message dispatching to the broker MUST be decoupled via Change Data Capture (CDC with Debezium) or an asynchronous outbox publisher worker to guarantee at-least-once delivery without distributed 2PC.

5. **Idempotent Consumers & Dead-Letter Topic (DLT) Discipline**:
   - All asynchronous message listeners (`@KafkaListener`) MUST be designed to be idempotent to handle duplicated messages safely. Deduplication must leverage unique business keys or idempotency tables.
   - Message consumers MUST NOT silently swallow unhandled exceptions or drop poisoned pills.
   - Configure deterministic retry and error handling (e.g., Spring Kafka `@RetryableTopic` or default error handler) routing exhausted failures to a dedicated Dead-Letter Topic (`<topic>.DLT`) with original error headers preserved.

6. **Saga Pattern over Distributed Two-Phase Commit (2PC / XA)**:
   - Never attempt distributed ACID transactions (Two-Phase Commit, XA) across microservices.
   - Multi-service distributed workflows MUST be modeled as Sagas (either Choreography-based using domain events or Orchestration-based with a dedicated coordinator).
   - Every forward step in a Saga MUST define an explicit, idempotent compensating transaction (`undo` action) to revert state if an intermediate step fails.

7. **Perimeter Authentication & Downstream Stateless JWT Resource Servers**:
   - External clients MUST NOT access internal microservices directly. All external traffic must ingress through an API Gateway (Spring Cloud Gateway).
   - Authenticate external clients at the API Gateway using OAuth2 / OpenID Connect (OIDC).
   - The Gateway MUST use the Token Relay filter to inject verified JWT tokens into downstream requests.
   - Downstream microservices MUST run as stateless Spring Security Resource Servers (`oauth2ResourceServer().jwt()`), validating cryptographic token signatures locally without session stickiness.

8. **Distributed Tracing & W3C TraceContext Header Propagation**:
   - Every microservice MUST configure distributed tracing using Micrometer Tracing with an OpenTelemetry (OTel) or Brave tracer.
   - All outgoing synchronous HTTP requests (`RestClient`) and asynchronous messaging records (Kafka headers) MUST automatically propagate W3C TraceContext headers (`traceparent`, `tracestate`).
   - All logging MUST be structured (JSON) and automatically include `traceId` and `spanId` injected into MDC. RFC 7807 `ProblemDetail` error responses MUST include `traceId` in their error extension properties.

9. **Cloud-Native Kubernetes Probes & Graceful Shutdown**:
   - Microservices MUST enable Spring Boot Actuator health probes for container orchestrators (`management.endpoint.health.probes.enabled=true`).
   - Microservices MUST configure separate Liveness (`/actuator/health/liveness`) and Readiness (`/actuator/health/readiness`) endpoints. Liveness must verify internal container health; Readiness must verify availability to accept incoming traffic.
   - Microservices MUST configure graceful shutdown (`server.shutdown=graceful`) with a container-aligned termination grace period (`spring.lifecycle.timeout-per-shutdown-phase`).

10. **Consumer-Driven Contract (CDC) Testing & Downstream Virtualization**:
    - Never depend on unstable shared staging environments to verify inter-service compatibility.
    - Inter-service REST and event contracts MUST be verified in automated CI pipelines using Consumer-Driven Contracts (Spring Cloud Contract or Pact).
    - Downstream service dependencies in unit/slice tests MUST be virtualized using declarative stubs (WireMock) or Testcontainers for messaging/database dependencies.

11. **Backend for Frontend (BFF) Pattern & Edge Boundary Isolation**:
    - Avoid monolithic "god" API gateways that aggregate disparate domain models and complex business logic into a single deployment bottleneck.
    - Employ the Backend-for-Frontend (BFF) pattern to isolate client-specific concerns (Web, Mobile, External B2B APIs) into lightweight, purpose-built gateway instances.
    - Gateways must enforce strict edge rate limiting (via Redis-backed token buckets) and CORS policies before traffic enters the private internal network.

12. **Zero Hardcoded Secrets & Externalized Configuration**:
    - Microservices MUST NEVER hardcode service URLs, broker addresses, database credentials, or secret keys in source code or `application.yaml`.
    - Microservices MUST resolve configuration through immutable Java 21 `record`s annotated with `@ConfigurationProperties`. Absolute ban on `@Value` injection.
    - Strict placeholder discipline: Placeholders in Java classes must reference Spring property keys without default fallback values (e.g., `${app.bar.foo}`, NEVER `${APP_BAR_FOO}` or `${app.bar.foo:1s}`). In `application.yaml`, properties point to environment variables without default values (e.g., `app.bar.foo: ${APP_BAR_FOO}`, NEVER `${APP_BAR_FOO:1s}`).
    - All environment variables and their default/sample values MUST be defined and documented in `.env.example` at the repository root.

---

Consult the corresponding specialized skills for comprehensive architectural patterns, code recipes, configuration templates, and testing blueprints.
