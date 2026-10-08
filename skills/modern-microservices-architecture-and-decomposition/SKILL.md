---
name: modern-microservices-architecture-and-decomposition
description: >-
  Use this skill when defining microservice boundaries, applying Domain-Driven Design (DDD) Bounded Contexts, implementing the database-per-service pattern, preventing distributed monoliths, creating Anti-Corruption Layers (ACL), or authoring API-First contracts with OpenAPI 3.1.
---

# Microservices Architecture, Boundaries & Decomposition

This skill guides the architectural decomposition of systems into autonomous, resilient microservices using Domain-Driven Design (DDD) principles and modern Spring Boot 3.x+ engineering standards.

---

## 1. Core Principles & Boundary Sizing

Decomposing a system into microservices requires balancing business autonomy with operational complexity. Sizing boundaries incorrectly leads to two failure modes:
1. **The Distributed Monolith**: Too fine-grained ("nano-services") where services share databases, communicate synchronously in deep call chains, and cannot be deployed independently.
2. **The Tangled Monolith**: Overly coarse-grained services where disparate business contexts are bound together, preventing teams from innovating autonomously.

### The Golden Rules of Microservice Decomposition
- **Organize Around Business Capabilities**: Group services by business capabilities (e.g., Billing, Inventory, Fulfillment, Identity) rather than technical tiers (e.g., UI, Business Logic, Database).
- **Domain-Driven Design (DDD) Bounded Contexts**: Every microservice represents a single Bounded Context (or a coherent set of closely related aggregates). Ubiquitous language is explicit and isolated inside the context.
- **High Cohesion, Loose Coupling**: Elements that change together stay together. Code within a service changes frequently together; code across services changes independently.
- **Autonomous Lifecycle**: A microservice must be deployable, scalable, and testable without coordinating deployments with other services.

---

## 2. Database-per-Service Pattern

Shared databases across microservices completely violate decoupling and autonomy:
- Schema updates break multiple services simultaneously.
- Table locks and connection pool starvation in one service bring down other services.
- Data access bypassing domain boundaries violates business invariants.

### Strict Data Isolation Rules
1. **Private Schema Ownership**: Each microservice owns its datastore exclusively. No other microservice may access its database directly.
2. **Polyglot Persistence**: Choose the storage engine best suited for the bounded context (e.g., PostgreSQL for transactions in Order Service, MongoDB for document schemas in Catalog Service, Redis for cache/sessions).
3. **No Cross-Service SQL Joins**: Never join tables across microservice databases. Aggregate data across boundaries through published APIs, domain events, or Read-Model Projections (CQRS).
4. **Logical vs. Physical Separation**: In early growth phases, microservices may share a database cluster but MUST have separate logical databases and distinct database user credentials preventing cross-schema queries. In production, physically isolate database instances for high-throughput domains.

---

## 3. Context Mapping & Anti-Corruption Layer (ACL)

When microservices interact across domain boundaries, models must not leak or contaminate internal domain logic.

| DDD Context Relationship | Description | Spring Boot Implementation |
| :--- | :--- | :--- |
| **Shared Kernel** | Small shared subset of domain model (e.g., common value objects). | Shared lightweight library containing only immutable Java `record`s. Use with extreme caution. |
| **Customer / Supplier** | Upstream (supplier) provides APIs tailored to downstream (customer). | Downstream requests custom projections; upstream publishes versioned OpenAPI contracts. |
| **Conformist** | Downstream accepts upstream's model as-is without translation. | Downstream binds directly to upstream DTOs (advisable only when upstream is a mature internal platform). |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream concepts into its own domain language. | Explicit Adapter and Mapper (MapStruct / native mappers) isolating the internal domain from external payloads. |
| **Open Host Service / Published Language** | Upstream defines a well-documented public protocol for all consumers. | Upstream exposes versioned OpenAPI 3.1 REST APIs or AsyncAPI schemas (Avro/JSON Schema). |

---

## 4. API-First Contract Design

Microservices must treat their APIs as public products:
- **Design Before Coding**: Write OpenAPI 3.1 specifications or AsyncAPI definitions before implementing controller endpoints.
- **Semantic Versioning**: Follow `MAJOR.MINOR.PATCH`. Major versions for breaking changes (e.g., `/v1/orders` vs `/v2/orders`), minor for backward-compatible additions, patch for bug fixes.
- **Backward Compatibility Guarantee**: Never remove or rename existing JSON fields in minor/patch releases. Always make new fields optional.
- **Code Generation (Optional but Recommended)**: Use `openapi-generator-maven-plugin` or Gradle equivalent to generate controller interfaces and DTOs from the canonical OpenAPI spec.

---

## References

For comprehensive blueprints, patterns, and code examples, consult:
- [Bounded Contexts & Database-per-Service Architecture](./references/bounded_contexts_and_database_per_service.md): In-depth boundary decomposition rules, anti-patterns, and database isolation strategies.
- [API-First Contract Design & Semantic Evolution](./references/api_first_and_contract_design.md): OpenAPI 3.1 specification design, backward compatibility guidelines, and Spring Boot code generation.
