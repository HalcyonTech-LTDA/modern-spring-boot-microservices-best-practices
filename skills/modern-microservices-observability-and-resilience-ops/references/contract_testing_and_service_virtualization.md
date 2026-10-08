# Consumer-Driven Contract Testing & Service Virtualization

This reference covers Consumer-Driven Contracts (Pact / Spring Cloud Contract), Testcontainers integration, and WireMock service virtualization in modern microservices architectures.

---

## 1. Why End-to-End Staging Environments Fail

Testing microservices by deploying all 20+ services into a shared staging cluster has critical flaws:
- **Flaky & Slow**: One flaky service or slow deployment stalls all teams.
- **Data Pollution**: Teams step on each other's test data in shared databases.
- **Late Feedback**: Contract incompatibilities are discovered after code is merged rather than in PR pipelines.

---

## 2. Consumer-Driven Contract Testing with Pact

### Consumer Test (Order Service generates Pact Contract)
```java
package com.example.order.client;

import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.core.model.RequestResponsePact;
import au.com.dius.pact.core.model.annotations.Pact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.web.client.ClientHttpRequestFactories;
import org.springframework.boot.web.client.ClientHttpRequestFactorySettings;
import org.springframework.http.client.ClientHttpRequestFactory;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "inventory-service")
class InventoryClientPactTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact checkStockPact(PactDslWithProvider builder) {
        return builder
                .given("item SKU-1234 exists with stock")
                .uponReceiving("a request for stock of SKU-1234")
                .path("/api/v1/inventory/items/SKU-1234")
                .method("GET")
                .willRespondWith()
                .status(200)
                .headers(java.util.Map.of("Content-Type", "application/json"))
                .body("""
                        {
                            "sku": "SKU-1234",
                            "availableQuantity": 15
                        }
                        """)
                .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "checkStockPact")
    void testCheckStockContract(au.com.dius.pact.consumer.MockServer mockServer) {
        RestClient restClient = RestClient.builder().baseUrl(mockServer.getUrl()).build();
        var adapter = RestClientAdapter.create(restClient);
        InventoryClient client = HttpServiceProxyFactory.builderFor(adapter).build().createClient(InventoryClient.class);

        InventoryItemResponse response = client.checkStock("SKU-1234");

        assertThat(response.sku()).isEqualTo("SKU-1234");
        assertThat(response.availableQuantity()).isEqualTo(15);
    }
}
```

### Provider Verification (Inventory Service verifies contract in CI)
```java
package com.example.inventory;

import au.com.dius.pact.provider.junit5.HttpTestTarget;
import au.com.dius.pact.provider.junit5.PactVerificationContext;
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junitsupport.Provider;
import au.com.dius.pact.provider.junitsupport.State;
import au.com.dius.pact.provider.junitsupport.loader.PactFolder;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestTemplate;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("inventory-service")
@PactFolder("target/pacts")
class InventoryProviderContractTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    void setup(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

    @State("item SKU-1234 exists with stock")
    void toDefaultState() {
        // Prepare mock or test data in the provider
    }
}
```

---

## 3. Component Testing with Testcontainers `@ServiceConnection`

Run full component tests with real PostgreSQL and Kafka instances launched via Testcontainers:

```java
package com.example.order;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.kafka.KafkaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderServiceComponentTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer("apache/kafka-native:3.8.0");

    @Test
    void contextLoads() {
        // Verifies Spring context, Flyway/Liquibase migrations, and Kafka connectivity
    }
}
```

---

## 4. Downstream Service Virtualization with WireMock (Modern WireMock Standards)

When isolating a microservice during component and slice integration tests, downstream HTTP dependencies must be virtualized using declarative WireMock stubs adhering to Modern WireMock Engineering Standards:

### 1. Strict Separation of Routing (`mappings/`) and Payloads (`__files/`)
Never inline JSON bodies directly inside `mappings/` files. Always decouple payloads into `__files/` and reference via `bodyFileName`.

#### Specific Mapping Rule (`src/test/resources/wiremock/mappings/inventory/check-stock.json`)
```json
{
  "priority": 1,
  "request": {
    "method": "GET",
    "urlPathPattern": "/api/v1/inventory/items/([a-zA-Z0-9-]+)",
    "headers": {
      "Accept": {
        "contains": "application/json"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "bodyFileName": "inventory/stock-available.json",
    "transformers": ["response-template"]
  }
}
```

#### Decoupled Payload (`src/test/resources/wiremock/__files/inventory/stock-available.json`)
```json
{
  "sku": "{{request.pathSegments.[3]}}",
  "availableQuantity": 50,
  "inStock": true,
  "warehouse": "US-EAST-01"
}
```

#### Fallback Low-Priority Stub (`src/test/resources/wiremock/mappings/inventory/fallback-not-found.json`)
```json
{
  "priority": 10,
  "request": {
    "method": "GET",
    "urlPathPattern": "/api/v1/inventory/items/.*"
  },
  "response": {
    "status": 404,
    "headers": {
      "Content-Type": "application/problem+json"
    },
    "bodyFileName": "inventory/not-found-problem.json"
  }
}
```

### 2. Spring Boot Integration with WireMock Testcontainers
```java
package com.example.order.client;

import com.example.order.infrastructure.client.inventory.InventoryClient;
import com.example.order.infrastructure.client.inventory.InventoryItemResponse;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.wiremock.integrations.testcontainers.WireMockContainer;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderInventoryClientIntegrationTest {

    @Container
    static WireMockContainer wiremock = new WireMockContainer("wiremock/wiremock:3.9.1-alpine")
            .withMappingFromResource("mappings", "wiremock/mappings")
            .withFileFromResource("__files", "wiremock/__files");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("services.inventory.base-url", wiremock::getBaseUrl);
    }

    @Autowired
    private InventoryClient inventoryClient;

    @Test
    void shouldReturnStockWhenInventoryExists() {
        InventoryItemResponse item = inventoryClient.checkStock("SKU-9999");

        assertThat(item.sku()).isEqualTo("SKU-9999");
        assertThat(item.availableQuantity()).isEqualTo(50);
    }
}
```
