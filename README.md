# Modern Spring Boot Microservices Best Practices Plugin

An Antigravity plugin establishing automated engineering guardrails, architectural patterns, resilience standards, distributed data consistency disciplines, perimeter security, and full-spectrum observability for modern Microservices using **Spring Boot 3.x+** (3.2+, 3.3+, 3.4+) and **Java 21+**.

---

## Installation

Install directly in Antigravity via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-spring-boot-microservices-best-practices
```

Verify that the plugin is installed and ready:

```bash
agy plugin list
```

---
## Overview

Transitioning from a monolithic architecture to distributed microservices promises developer autonomy, granular scalability, and rapid deployment velocity. However, without rigorous architectural discipline and cloud-native standards, microservice ecosystems rapidly degrade into critical production failure modes:

- **The Distributed Monolith**: Overly fragmented "nano-services" coupled by shared databases, synchronous RPC chains, and co-dependent deployments.
- **The Dual-Write Data Corruption Trap**: Updating a local database and publishing a Kafka/RabbitMQ message in a single method, leading to irrecoverable data loss and split-brain states when network blips occur.
- **Cascading Outages**: Synchronous inter-service calls without circuit breakers, timeouts, or jittered retries, causing a slow downstream dependency to exhaust all worker threads across the entire mesh.
- **Perimeter Vulnerabilities & Security Drift**: Leaking internal service endpoints to the public internet, duplicating authentication logic, or using stateful sessions in downstream microservices.
- **Observability Blind Spots**: Disconnected logs and traces without W3C TraceContext propagation, making it impossible to diagnose cross-service latency regressions or distributed transaction failures.
- **Flaky Staging Environments**: Relying on brittle, shared end-to-end environments to test inter-service contracts instead of automated Consumer-Driven Contracts (CDC).

This plugin establishes production-grade engineering standards for Spring Boot microservices across five foundational pillars:

1. **Architecture Boundaries & Domain Decomposition (`modern-microservices-architecture-and-decomposition`)**: Domain-Driven Design (DDD) Bounded Contexts, business capability mapping, strict database-per-service isolation, Anti-Corruption Layers (ACL), and API-First contract design (OpenAPI 3.1).
2. **Synchronous Inter-Service Communication (`modern-microservices-inter-service-communication`)**: Declarative HTTP Interfaces (`@HttpExchange`) configured on top of fluent `RestClient`, connection pool tuning, Resilience4j fault tolerance (Circuit Breakers, Timeouts, Rate Limiters, Bulkheads, Retries with jitter), and binary RPC via Spring gRPC.
3. **Event-Driven Architecture & Distributed Consistency (`modern-microservices-event-driven-and-data-consistency`)**: Spring for Apache Kafka (`KafkaTemplate`, `@KafkaListener`), Transactional Outbox pattern with Debezium CDC, Idempotent Consumers, Dead-Letter Topics (`.DLT`), Saga pattern (Orchestration vs Choreography), and schema registry evolution.
4. **Perimeter Gateway & Zero-Trust Security (`modern-microservices-perimeter-gateway-and-security`)**: Spring Cloud Gateway (MVC & Reactive), Backend-for-Frontend (BFF) patterns, centralized rate limiting (Redis), OAuth2/OIDC Token Relay filter, downstream Spring Security Stateless JWT Resource Servers, and service-to-service mTLS.
5. **Observability, Operations & Contract Testing (`modern-microservices-observability-and-resilience-ops`)**: Distributed Tracing with Micrometer Tracing & OpenTelemetry (W3C `traceparent` propagation), JSON structured logging with MDC (`traceId`, `spanId`), Kubernetes liveness/readiness probes, graceful shutdown, and Consumer-Driven Contract testing (Pact / Spring Cloud Contract) with WireMock.

---

## What's Included

### 1. Automatic Guardrails (`rules/modern-spring-boot-microservices.md`)
Automatically enforced across all microservice projects:
- **Database-per-Service Mandate**: Absolute ban on shared databases, cross-schema SQL joins, and cross-service foreign keys.
- **Declarative HTTP Interfaces & RestClient**: Ban on legacy `RestTemplate` and deprecated OpenFeign in favor of Spring 6 Declarative HTTP Interfaces.
- **Mandatory Resilience4j Boundaries**: Strict circuit breaking, timeouts, and jittered retries on all synchronous inter-service calls.
- **Zero Dual Writes (Transactional Outbox)**: Mandatory atomic local database persistence with decoupled CDC or polling event dispatchers.
- **Idempotent Consumers & DLT Discipline**: Idempotency guards via business keys/deduplication tables and non-blocking retry routing to Dead-Letter Topics (`<topic>.DLT`).
- **Sagas over Distributed 2PC**: Absolute ban on distributed Two-Phase Commit (XA); mandatory Sagas with idempotent compensating transactions.
- **Edge Gateway Token Relay & Stateless JWT Servers**: Perimeter authentication with Spring Cloud Gateway and downstream stateless JWT Resource Servers.
- **Distributed Tracing & W3C TraceContext Propagation**: Mandatory Micrometer Tracing with OpenTelemetry, MDC injection of `traceId`/`spanId`, and RFC 7807 `ProblemDetail` trace integration.
- **Cloud-Native Kubernetes Probes & Graceful Shutdown**: Actuator Liveness and Readiness probes paired with `server.shutdown: graceful`.
- **Consumer-Driven Contract (CDC) Testing**: Fast, isolated contract verification (Pact / Spring Cloud Contract) avoiding shared staging bottlenecks.
- **Backend-for-Frontend (BFF) Pattern**: Dedicated client-tailored gateways preventing monolithic edge bottlenecks.
- **Zero Hardcoded Secrets & Strict Placeholder Discipline**: 12-factor configuration via immutable `@ConfigurationProperties` records (ban on `@Value`) and strict placeholders without default fallback values (`.env.example` discipline).

---

### 2. Specialized Skills (`skills/`)

#### 🧱 `modern-microservices-architecture-and-decomposition`
Guides domain boundary modeling, bounded context identification, database isolation, and API contract design:
- `references/bounded_contexts_and_database_per_service.md`: DDD Bounded Contexts, aggregate boundaries, Anti-Corruption Layer (ACL) adapters, database isolation rules, and CQRS read projections.
- `references/api_first_and_contract_design.md`: OpenAPI 3.1 specification blueprints, tolerant reader rules, backward compatibility guidelines, and Gradle code generation pipelines.

#### ⚡ `modern-microservices-inter-service-communication`
Guides synchronous inter-service communication, declarative HTTP clients, connection pools, and Resilience4j fault tolerance:
- `references/declarative_http_interfaces_and_restclient.md`: Declarative `@HttpExchange` interfaces, Apache HttpClient 5 / JDK connection pooling, RFC 7807 error handling, and Virtual Thread safety.
- `references/resilience4j_circuit_breakers_and_retries.md`: Circuit Breaker state machines, `application.yaml` configurations, retry backoff with jitter, fallback methods, and Micrometer metrics.

#### 🔄 `modern-microservices-event-driven-and-data-consistency`
Guides asynchronous event-driven messaging, Transactional Outbox, idempotency, and distributed Sagas:
- `references/transactional_outbox_and_cdc_pattern.md`: Outbox schema DDL, atomic JPA/JdbcClient persistence, Debezium CDC connector configuration, and polling fallback workers.
- `references/idempotent_consumer_and_dead_letter_topics.md`: Database deduplication table pattern, Spring Kafka `@RetryableTopic` choreography, Dead-Letter Topic handlers, and poison-pill recovery.
- `references/saga_pattern_and_distributed_transactions.md`: Banning 2PC/XA, Choreography vs Orchestration Sagas, Spring Boot Saga Orchestrator state machine, and compensating transaction rules.

#### 🛡️ `modern-microservices-perimeter-gateway-and-security`
Guides API gateway architecture, routing, rate limiting, and zero-trust perimeter security:
- `references/spring_cloud_gateway_and_bff_pattern.md`: Spring Cloud Gateway configuration, Redis token bucket rate limiting, global CORS, and Backend-for-Frontend (BFF) architecture.
- `references/oauth2_token_relay_and_jwt_resource_server.md`: Gateway OAuth2 login & TokenRelay filter, downstream stateless Spring Security Resource Server setup, and method-level security (`@PreAuthorize`).

#### 🔭 `modern-microservices-observability-and-resilience-ops`
Guides distributed tracing, structured logging, Kubernetes lifecycle probes, and Consumer-Driven Contract testing:
- `references/distributed_tracing_and_micrometer_otel.md`: Micrometer Tracing with OpenTelemetry, OTLP exporters, Logstash JSON Logback encoder, and MDC trace propagation in RFC 7807 `ProblemDetail`.
- `references/health_probes_and_graceful_shutdown.md`: Spring Boot Actuator Liveness/Readiness probes, graceful shutdown timeouts, and Kubernetes Deployment manifests.
- `references/contract_testing_and_service_virtualization.md`: Pact consumer & provider tests, Testcontainers `@ServiceConnection`, and WireMock service virtualization.

---

## Directory Layout

```text
modern-spring-boot-microservices-best-practices/
├── plugin.json
├── README.md
├── rules/
│   └── modern-spring-boot-microservices.md
└── skills/
    ├── modern-microservices-architecture-and-decomposition/
    │   ├── SKILL.md
    │   └── references/
    │       ├── api_first_and_contract_design.md
    │       └── bounded_contexts_and_database_per_service.md
    ├── modern-microservices-event-driven-and-data-consistency/
    │   ├── SKILL.md
    │   └── references/
    │       ├── idempotent_consumer_and_dead_letter_topics.md
    │       ├── saga_pattern_and_distributed_transactions.md
    │       └── transactional_outbox_and_cdc_pattern.md
    ├── modern-microservices-inter-service-communication/
    │   ├── SKILL.md
    │   └── references/
    │       ├── declarative_http_interfaces_and_restclient.md
    │       └── resilience4j_circuit_breakers_and_retries.md
    ├── modern-microservices-observability-and-resilience-ops/
    │   ├── SKILL.md
    │   └── references/
    │       ├── contract_testing_and_service_virtualization.md
    │       ├── distributed_tracing_and_micrometer_otel.md
    │       └── health_probes_and_graceful_shutdown.md
    └── modern-microservices-perimeter-gateway-and-security/
        ├── SKILL.md
        └── references/
            ├── oauth2_token_relay_and_jwt_resource_server.md
            └── spring_cloud_gateway_and_bff_pattern.md
```

---

## Core Technologies & Compatibility Baseline

- **Language Baseline**: Java 21+ LTS (Virtual Threads, Pattern Matching, Records).
- **Framework Baseline**: Spring Boot 3.2+, 3.3+, 3.4+, Spring Cloud 2023.0+ (Leyton).
- **Communication**: Declarative HTTP Interfaces (`@HttpExchange`), `RestClient`, Spring gRPC.
- **Resilience**: Resilience4j 2.2+, Spring Cloud CircuitBreaker.
- **Event-Driven**: Spring for Apache Kafka 3.x, Debezium CDC, Kafka Connect.
- **Perimeter & Security**: Spring Cloud Gateway (MVC & WebFlux), Spring Security 6.x OAuth2 Resource Server.
- **Observability**: Micrometer 1.13+, Micrometer Tracing 1.3+, OpenTelemetry (OTel), Logstash Logback Encoder.
- **Testing**: JUnit 5 Jupiter, AssertJ, Testcontainers 1.20+, Pact 4.6+ / Spring Cloud Contract, WireMock 3.x.

---

## License

This project is licensed under the [MIT License](LICENSE) - see the LICENSE file for details.
Copyright (c) 2026 Halcyon Tech.
