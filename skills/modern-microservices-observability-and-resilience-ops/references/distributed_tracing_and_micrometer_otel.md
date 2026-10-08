# Distributed Tracing, Micrometer & OpenTelemetry in Spring Boot 3.x+

This reference provides production configurations for distributed tracing with Micrometer Tracing, OpenTelemetry, structured JSON logging, and MDC trace propagation.

---

## 1. Dependencies Setup (`build.gradle.kts`)

```kotlin
// Micrometer Tracing Core & OpenTelemetry Bridge
implementation("io.micrometer:micrometer-tracing-bridge-otel")
implementation("io.opentelemetry:opentelemetry-exporter-otlp")

// Actuator & Prometheus Metrics
implementation("org.springframework.boot:spring-boot-starter-actuator")
runtimeOnly("io.micrometer:micrometer-registry-prometheus")

// Structured JSON Logging (Logstash Logback Encoder)
implementation("net.logstash.logback:logstash-logback-encoder:7.4")
```

---

## 2. Distributed Tracing Configuration (`application.yaml`)

```yaml
# Invariant: No default fallback values in placeholders. All defaults reside in .env.example!
management:
  tracing:
    sampling:
      probability: 1.0 # 100% in dev/staging; tune to 0.1 (10%) in high-throughput prod
    propagation:
      type: W3C # Invariant: W3C TraceContext headers (traceparent, tracestate)
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT}
      transport: http

logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
```

```bash
# .env.example - documented at repository root
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318/v1/traces
```

---

## 3. Structured JSON Logging with Logback (`src/main/resources/logback-spring.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="envProfile" source="spring.profiles.active"/>

    <appender name="CONSOLE_JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <version/>
                <logLevel/>
                <loggerName/>
                <threadName/>
                <message/>
                <mdc/> <!-- Injects traceId, spanId, customerId -->
                <stackTrace/>
                <pattern>
                    <pattern>
                        {
                            "service.name": "${appName}",
                            "environment": "${envProfile}"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE_JSON"/>
    </root>
</configuration>
```

---

## 4. Injecting `traceId` into RFC 7807 `ProblemDetail`

When an unhandled exception or business validation fails, the trace ID must be attached to the response so developers and support teams can look up the exact distributed trace immediately:

```java
package com.example.common.infrastructure.web;

import io.micrometer.tracing.Tracer;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.net.URI;
import java.time.Clock;

@RestControllerAdvice
public class GlobalProblemDetailExceptionHandler extends ResponseEntityExceptionHandler {

    private final Tracer tracer;
    private final Clock clock;

    public GlobalProblemDetailExceptionHandler(Tracer tracer, Clock clock) {
        this.tracer = tracer;
        this.clock = clock;
    }

    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusinessException(BusinessException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
        problem.setTitle("Business Validation Violation");
        problem.setType(URI.create("https://errors.example.com/business-violation"));
        problem.setProperty("timestamp", clock.instant());

        // Attach traceId to ProblemDetail extension properties
        if (tracer.currentSpan() != null) {
            problem.setProperty("traceId", tracer.currentSpan().context().traceId());
            problem.setProperty("spanId", tracer.currentSpan().context().spanId());
        }

        return problem;
    }
}
```
