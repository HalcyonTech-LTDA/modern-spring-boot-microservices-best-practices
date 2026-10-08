---
name: modern-microservices-observability-and-resilience-ops
description: >-
  Use this skill when implementing distributed tracing with Micrometer Tracing and OpenTelemetry (OTel), configuring structured JSON logging with MDC trace propagation, enabling Kubernetes liveness/readiness health probes, configuring graceful shutdown, or setting up Consumer-Driven Contract testing (Pact / Spring Cloud Contract) with WireMock.
---

# Microservices Observability, Cloud Lifecycle & Contract Testing

This skill guides the implementation of production-grade distributed tracing, structured logging, cloud-native container health choreography, and Consumer-Driven Contract testing for Spring Boot 3.x+ microservices.

---

## 1. The Three Pillars of Distributed Observability

In a distributed microservice mesh, tracking requests across service boundaries requires unified observability:

### 1. Distributed Tracing (Micrometer Tracing + OpenTelemetry)
- **Trace Context Propagation**: W3C TraceContext standards (`traceparent`, `tracestate`).
- **Span Management**: Automatic span creation across incoming HTTP requests, outgoing `RestClient` calls, and Kafka producer/consumer records.
- **Trace Export**: Direct streaming to OpenTelemetry Collector, Tempo, Zipkin, or AWS X-Ray via OTLP (`opentelemetry-exporter-otlp`).

### 2. Structured JSON Logging with MDC Injection
- Automatically inject `traceId`, `spanId`, and `service.name` into every log statement via Logback MDC (Mapped Diagnostic Context).
- Format logs as JSON (e.g., using Logstash Logback encoder or OpenTelemetry appenders) for ingestion into Elasticsearch, Grafana Loki, or CloudWatch.
- Inject `traceId` into RFC 7807 `ProblemDetail` error responses so frontend users can quote a single trace ID for incident investigations.

### 3. Application Metrics & SLA Monitoring
- Micrometer dimensional metrics exposed via `/actuator/prometheus`.
- Key Microservices Metrics: Request count, P95/P99 latency timers, HTTP error rates, circuit breaker states, HikariCP connection pool utilization, Kafka consumer lag.

---

## 2. Cloud-Native Lifecycle: Health Probes & Graceful Shutdown

Kubernetes container orchestrators require fine-grained lifecycle signals:

### Health Probes Discipline
- **Liveness Probe (`/actuator/health/liveness`)**: Verifies the JVM process is alive and not deadlocked. If it fails, Kubernetes **restarts** the container.
- **Readiness Probe (`/actuator/health/readiness`)**: Verifies the service has completed initialization, database connections are established, and caches are warm. If it fails, Kubernetes **removes** the pod from Service routing endpoints without restarting it.

### Graceful Shutdown Choreography
When a pod receives `SIGTERM`:
1. Kubernetes marks the pod as unready and stops routing new traffic.
2. Spring Boot stops accepting new HTTP connections.
3. In-flight requests are allowed to complete within a bounded grace period (`spring.lifecycle.timeout-per-shutdown-phase: 30s`).
4. HikariCP pools and Kafka consumers close cleanly without dropping transactions.

---

## 3. Microservices Testing Strategy & Consumer-Driven Contracts

### The Microservices Testing Pyramid
```text
               /  E2E Smoke Tests  \        (1-2 critical happy paths)
              /---------------------\
             /  Contract Tests (Pact) \     (API compatibility in CI)
            /---------------------------\
           / Component (Testcontainers)  \   (Isolated service with real DB/Kafka)
          /-------------------------------\
         /  Slice & Unit Tests (@WebMvc)   \ (Fast, deterministic, isolated)
        /-----------------------------------\
```

### Consumer-Driven Contract (CDC) Testing
- Downstream consumer creates a contract (`Pact` or `Spring Cloud Contract`) defining the exact requests it sends and responses it expects.
- The contract is verified automatically in the upstream provider's CI pipeline without deploying both services simultaneously.
- Prevents breaking API changes from ever merging into `main`.

---

## References

For complete deep-dive blueprints and production configurations, consult:
- [Distributed Tracing, Micrometer & OpenTelemetry in Spring Boot 3.x+](./references/distributed_tracing_and_micrometer_otel.md): Micrometer Tracing setup, OpenTelemetry exporter, JSON logback configuration, and MDC propagation.
- [Kubernetes Health Probes & Graceful Shutdown](./references/health_probes_and_graceful_shutdown.md): Spring Boot Actuator probe configuration, custom `AvailabilityChangeEvent` listeners, and Kubernetes Pod YAML.
- [Consumer-Driven Contract Testing & Service Virtualization](./references/contract_testing_and_service_virtualization.md): Pact / Spring Cloud Contract consumer & provider tests, Testcontainers `@ServiceConnection`, and WireMock stubbing.
