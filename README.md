<div align="center">

<a href="https://www.halcyontech.com.br/" target="_blank" rel="noopener noreferrer">
  <img src="assets/halcyon-horizontal-cor.svg" alt="Halcyon Tech" height="60">
</a>

# Modern Spring Boot Microservices Best Practices Plugin
### Distributed Microservices Architecture, Resilience4j & Observability

[![Antigravity Plugin](https://img.shields.io/badge/Antigravity-Plugin%20v1.0.0-4F46E5.svg)](https://antigravity.google)
[![License: MIT](https://img.shields.io/badge/License-MIT-10B981.svg)](./LICENSE)
[![Website](https://img.shields.io/badge/Website-halcyontech.com.br-008080.svg)](https://www.halcyontech.com.br/)
[![Organization](https://img.shields.io/badge/Organization-HalcyonTech--LTDA-black.svg)](https://github.com/HalcyonTech-LTDA)

<p align="center">
  <b>Comprehensive engineering standards, architectural patterns, and production guardrails for modern Microservices using modern Spring Boot 3.x+: database-per-service isolation, declarative HTTP interfaces with RestClient, Resilience4j fault tolerance, transactional Outbox & Saga distributed data patterns, Spring Cloud Gateway with OAuth2/JWT token relay, distributed tracing with Micrometer & OpenTelemetry, and consumer-driven contract testing.</b>
</p>

</div>

---

## Installation & Verification

### Antigravity CLI (`agy`)

Install directly into your Antigravity environment via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-spring-boot-microservices-best-practices
```

Verify that the plugin is active and adheres to the strict official schema:

```bash
agy plugin list
agy plugin validate --strict ./modern-spring-boot-microservices-best-practices
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

---

## What's Included (Component Catalog)

### Contextual Rules (`rules/`)
Automatically loaded to govern and protect project contexts:
- **[`modern-spring-boot-microservices.md`](rules/modern-spring-boot-microservices.md)**: >-

### Procedural Skills (`skills/`)
On-demand actionable workflows and runbooks:
- **[`/modern-microservices-architecture-and-decomposition`](skills/modern-microservices-architecture-and-decomposition/SKILL.md)**: Use this skill when defining microservice boundaries, applying Domain-Driven Design (DDD) Bounded Contexts, implementing the database-per-service pattern, preventing distributed monoliths, creating Anti-Corruption Layers (ACL), or authoring API-First contracts with OpenAPI 3.1.
- **[`/modern-microservices-event-driven-and-data-consistency`](skills/modern-microservices-event-driven-and-data-consistency/SKILL.md)**: Use this skill when designing asynchronous messaging, event-driven architectures with Apache Kafka, implementing the Transactional Outbox pattern with Debezium CDC, handling idempotent message consumption, managing Dead-Letter Topics (DLT), or orchestrating distributed transactions with the Saga pattern.
- **[`/modern-microservices-inter-service-communication`](skills/modern-microservices-inter-service-communication/SKILL.md)**: Use this skill when implementing synchronous RPC communication between Spring Boot microservices: declarative HTTP interfaces (@HttpExchange), RestClient configuration, connection pool tuning, Resilience4j fault tolerance (CircuitBreaker, Retry, RateLimiter, Bulkhead, TimeLimiter), fallback mechanisms, or high-performance gRPC.
- **[`/modern-microservices-observability-and-resilience-ops`](skills/modern-microservices-observability-and-resilience-ops/SKILL.md)**: Use this skill when implementing distributed tracing with Micrometer Tracing and OpenTelemetry (OTel), configuring structured JSON logging with MDC trace propagation, enabling Kubernetes liveness/readiness health probes, configuring graceful shutdown, or setting up Consumer-Driven Contract testing (Pact / Spring Cloud Contract) with WireMock.
- **[`/modern-microservices-perimeter-gateway-and-security`](skills/modern-microservices-perimeter-gateway-and-security/SKILL.md)**: Use this skill when configuring API gateways with Spring Cloud Gateway (MVC or Reactive), designing Backend-for-Frontend (BFF) architectures, implementing OAuth2/OIDC Token Relay, securing downstream Spring Boot microservices as stateless JWT Resource Servers, or establishing service-to-service mTLS.

---

## Directory Hierarchy

```text
modern-spring-boot-microservices-best-practices/
├── plugin.json              # Official Antigravity manifest (schema v1)
├── LICENSE                  # MIT License (Copyright 2026 HalcyonTech LTDA)
├── .gitignore               # System ignores & installed_version.json exclusions
├── .gitattributes           # Deterministic line endings (* text=auto eol=lf)
├── README.md                # Golden Standard documentation & runbook
├── assets/
│   ├── logo.svg             # Antigravity marketplace square icon
│   └── halcyon-horizontal-cor.svg # Official Halcyon Tech brand logo (white background)             # Official Halcyon Tech corporate vector symbol
├── rules/                   # Contextual rules discovered automatically
│   └── modern-spring-boot-microservices.md
├── skills/                  # Actionable procedural skills (Progressive Disclosure)
│   └── modern-microservices-architecture-and-decomposition/
│       └── SKILL.md
│   └── modern-microservices-event-driven-and-data-consistency/
│       └── SKILL.md
│   └── modern-microservices-inter-service-communication/
│       └── SKILL.md
│   └── modern-microservices-observability-and-resilience-ops/
│       └── SKILL.md
│   └── modern-microservices-perimeter-gateway-and-security/
│       └── SKILL.md
```

---

## 💡 Prompt Recipes & Use Cases

```markdown
- "Design microservice boundaries following Domain-Driven Design and database-per-service"
- "Implement Resilience4j circuit breakers, retries, and declarative HTTP interfaces"
- "Configure distributed tracing with Micrometer, OpenTelemetry, and structured logging"
```

---

## 📄 License & Corporate Governance

Distributed under the **MIT License**. Copyright (c) 2026 **HalcyonTech LTDA**.  
Engineered with precision and excellence by [Halcyon Tech](https://www.halcyontech.com.br/) &mdash; [GitHub Organization](https://github.com/HalcyonTech-LTDA).
