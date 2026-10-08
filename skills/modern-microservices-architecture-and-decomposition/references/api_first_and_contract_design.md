# API-First Contract Design & Semantic Evolution

This reference covers API-First development workflows, OpenAPI 3.1 specifications, backward compatibility rules, and code generation patterns in modern Spring Boot 3.x+ microservices.

---

## 1. The API-First Philosophy

In distributed microservices, APIs represent binding contractual commitments between autonomous teams.
- **Code-First Antipattern**: Writing Spring `@RestController`s, then auto-generating Swagger UI docs after the fact. This leads to accidental breaking changes, leaky internal implementation details, and uncoordinated client disruptions.
- **API-First Pattern**: Designing, reviewing, and versioning the OpenAPI 3.1 specification before writing a single line of application code. Consumers can mock the API and begin frontend or integration development immediately.

---

## 2. OpenAPI 3.1 Specification Blueprint

```yaml
openapi: 3.1.0
info:
  title: Order Processing Service API
  version: 1.2.0
  description: Public API for creating and querying orders in the Order Context.
servers:
  - url: /api/v1
    description: Production Gateway Route
paths:
  /orders:
    post:
      summary: Place a new order
      operationId: placeOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PlaceOrderRequest'
      responses:
        '201':
          description: Order successfully created
          headers:
            Location:
              schema:
                type: string
                format: uri
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '400':
          description: Invalid request payload
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetail'
        '422':
          description: Business validation rule violation
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetail'

components:
  schemas:
    PlaceOrderRequest:
      type: object
      required:
        - customerId
        - items
        - idempotencyKey
      properties:
        customerId:
          type: string
          format: uuid
        idempotencyKey:
          type: string
          format: uuid
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItemRequest'

    OrderItemRequest:
      type: object
      required:
        - sku
        - quantity
      properties:
        sku:
          type: string
          pattern: '^[A-Z]{3}-[0-9]{4}$'
        quantity:
          type: integer
          minimum: 1
          maximum: 100

    OrderResponse:
      type: object
      required:
        - orderId
        - customerId
        - status
        - totalAmount
        - createdAt
      properties:
        orderId:
          type: string
          format: uuid
        customerId:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING, CONFIRMED, CANCELLED]
        totalAmount:
          type: string
          example: "149.99"
          description: Monetary amount formatted as exact decimal string
        createdAt:
          type: string
          format: date-time

    ProblemDetail:
      type: object
      required:
        - type
        - title
        - status
      properties:
        type:
          type: string
          format: uri
        title:
          type: string
        status:
          type: integer
        detail:
          type: string
        instance:
          type: string
          format: uri
```

---

## 3. Backward Compatibility & Evolution Discipline

When changing an API in production, adhere to the **Tolerant Reader Pattern** and following invariant rules:

### Non-Breaking Changes (Permitted in Minor/Patch Versions)
- Adding a new optional request parameter or body property.
- Adding a new property to a response payload.
- Adding a new query parameter with a sensible default.
- Adding an entirely new endpoint path.

### Breaking Changes (Requires New Major Version URL or Header)
- Removing or renaming an existing endpoint, field, or query parameter.
- Making a previously optional field required.
- Changing the data type of an existing field (e.g., integer to string).
- Changing status codes (e.g., changing 200 OK to 204 No Content).
- Adding a new enum value to a response if consumers do not support open enums.

---

## 4. Gradle Code Generation Pipeline

Configure OpenAPI code generation in Gradle (`build.gradle.kts`):

```kotlin
plugins {
    id("org.openapi.generator") version "7.8.0"
}

openApiGenerate {
    generatorName.set("spring")
    inputSpec.set(layout.projectDirectory.file("specs/order-service-api-v1.yaml").asFile.path)
    outputDir.set(layout.buildDirectory.dir("generated/openapi").map { it.asFile.path })
    apiPackage.set("com.example.order.infrastructure.web.api")
    modelPackage.set("com.example.order.infrastructure.web.dto")
    configOptions.set(mapOf(
        "interfaceOnly" to "true",
        "useSpringBoot3" to "true",
        "useTags" to "true",
        "openApiNullable" to "false",
        "skipDefaultInterface" to "true"
    ))
}

sourceSets["main"].java.srcDir(layout.buildDirectory.dir("generated/openapi/src/main/java"))
tasks.compileJava { dependsOn(tasks.openApiGenerate) }
```
This produces clean, framework-independent Java interfaces and DTOs, keeping controller implementations strictly aligned with the contract.
