# Bounded Contexts & Database-per-Service Architecture

This guide details domain decomposition techniques, boundary rules, and database isolation patterns for modern Spring Boot microservices.

---

## 1. Domain-Driven Design (DDD) Decomposition

### Bounded Context Identification
A Bounded Context defines a boundary within which a specific domain model applies and terms have an unambiguous meaning.

#### The Ambiguity Trap: Same Word, Different Concepts
In an enterprise e-commerce platform:
- In the **Catalog Context**, a `Product` represents marketing descriptions, categories, images, and brand specifications.
- In the **Inventory Context**, a `Product` is a `StockKeepingUnit` (SKU) with physical dimensions, weight, warehouse bin location, and reorder levels.
- In the **Pricing Context**, a `Product` is a price matrix with seasonal discounts, tier margins, and tax rules.

Trying to represent all three concerns in a single monolithic `Product` entity creates an unmaintainable monster. In microservices, each service creates its own specialized entity within its Bounded Context.

### Anti-Corruption Layer (ACL) Implementation in Spring Boot
When a downstream microservice consumes data from an external or legacy system, it MUST shield its internal domain model using an Anti-Corruption Layer:

```java
package com.example.order.infrastructure.adapter.acl;

import com.example.order.domain.model.CustomerId;
import com.example.order.domain.model.CustomerProfile;
import com.example.order.domain.port.outbound.CustomerLookupPort;
import org.springframework.stereotype.Component;

@Component
public class ExternalCustomerAclAdapter implements CustomerLookupPort {

    private final ExternalCustomerClient customerClient;
    private final CustomerProfileMapper mapper;

    public ExternalCustomerAclAdapter(ExternalCustomerClient customerClient, CustomerProfileMapper mapper) {
        this.customerClient = customerClient;
        this.mapper = mapper;
    }

    @Override
    public CustomerProfile findProfile(CustomerId customerId) {
        // Fetch raw response from external supplier context
        ExternalCustomerDto externalDto = customerClient.getCustomer(customerId.value());
        
        // Translate foreign model into pure internal domain entity
        return mapper.toDomain(externalDto);
    }
}
```

---

## 2. The Database-per-Service Pattern

### Anti-Pattern: The Shared Database
```text
[ Order Service ]       [ Inventory Service ]       [ Billing Service ]
         \                      |                      /
          \                     |                     /
           ▼                    ▼                    ▼
             [ Shared PostgreSQL Database Instance ]
                 - orders table
                 - inventory table
                 - invoices table
```
**Why this fails:**
- **Schema Lock-In**: Altering the `orders` table risks breaking queries run by `Billing Service`.
- **Resource Contention**: Heavy reporting in `Billing Service` exhausts connections or locks tables, starving `Order Service` of transaction throughput.
- **Leaked Business Invariants**: `Billing Service` can directly update order status without going through `Order Service` validation.

### Correct Pattern: Database-per-Service
```text
[ Order Service ]       [ Inventory Service ]       [ Billing Service ]
         |                      |                      |
         ▼                      ▼                      ▼
[ Order Database ]     [ Inventory Database ]   [ Billing Database ]
  (PostgreSQL)             (MongoDB / DynamoDB)       (PostgreSQL)
```

### Transitioning from Shared Cluster to Isolated DBs
During transitional phases where spinning up separate RDS instances is cost-prohibitive:
1. **Separate Schemas & Users**:
   - `order_db` owned exclusively by user `order_svc_user`.
   - `inventory_db` owned exclusively by user `inventory_svc_user`.
   - Revoke cross-database permissions: `REVOKE ALL ON DATABASE order_db FROM inventory_svc_user;`.
2. **Dedicated Connection Pools**:
   Configure separate HikariCP connection pools with strict bounds in each Spring Boot service:
   ```yaml
   spring:
     datasource:
       url: ${DATABASE_URL}
       username: ${DATABASE_USER}
       password: ${DATABASE_PASSWORD}
       hikari:
         maximum-pool-size: 20
         minimum-idle: 5
         connection-timeout: 30000
         idle-timeout: 600000
         max-lifetime: 1800000
   ```

   ```bash
   # .env.example - documented at repository root
   DATABASE_URL=jdbc:postgresql://postgres:5432/order_db
   DATABASE_USER=order_svc_user
   DATABASE_PASSWORD=secret_local_dev_password
   ```

---

## 3. Resolving Cross-Service Data Aggregation

In a database-per-service architecture, SQL `JOIN`s across services are impossible. Three patterns resolve cross-service data queries:

### Pattern A: API Composition
The API Gateway or an Aggregator Service queries each microservice synchronously via HTTP/gRPC and merges the results in memory.
- **Best For**: Low-latency, real-time, lightweight aggregation (e.g., aggregating order header + customer profile + shipping status for an Order Details screen).
- **Caveat**: Can lead to high latency and cascading failures if downstream services are slow.

### Pattern B: CQRS (Command Query Responsibility Segregation) & Read Projections
Services listen to domain events and build materialized views in their local datastore optimized for specific query access patterns.
- **Example**: `Order Query Service` consumes `OrderCreatedEvent`, `PaymentCapturedEvent`, and `ShipmentDispatchedEvent` from Kafka, writing to an Elasticsearch or MongoDB read model.
- **Best For**: Complex dashboards, full-text searches, and multi-aggregate reports.
- **Benefit**: Zero runtime coupling between query services and transactional services.

### Pattern C: Event Enrichment
When publishing domain events, upstream services include commonly needed reference data (e.g., customer name and email) in the event payload, preventing consumers from making synchronous round-trip lookup calls.
