# Kubernetes Health Probes & Graceful Shutdown

This reference provides production configurations for Spring Boot Actuator Liveness/Readiness probes, graceful shutdown mechanics, and container orchestration alignment.

---

## 1. Spring Boot Actuator Probes Configuration (`application.yaml`)

```yaml
server:
  shutdown: graceful # Enables graceful shutdown of web server (Tomcat/Netty)

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s # Bounded window for finishing in-flight requests

management:
  endpoint:
    health:
      probes:
        enabled: true # Automatically activates /liveness and /readiness endpoints
      show-details: when_authorized
      roles: ACTUATOR_ADMIN
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
```

---

## 2. Difference Between Liveness & Readiness

| Dimension | Liveness Probe (`/actuator/health/liveness`) | Readiness Probe (`/actuator/health/readiness`) |
| :--- | :--- | :--- |
| **Endpoint** | `/actuator/health/liveness` | `/actuator/health/readiness` |
| **What It Checks** | JVM runtime state, internal deadlocks. | Datastore connections, Kafka listener ready, warm caches. |
| **On Failure** | Kubernetes sends `SIGTERM` and **restarts** pod. | Kubernetes **removes pod from Service endpoints** (no traffic). |
| **External Outage Impact** | MUST NOT fail if downstream DB is down! | MAY fail if mandatory local DB connection is broken. |

> **Warning**: Never include external downstream microservice healthchecks in your service's Liveness probe! If downstream service B goes down, failing the Liveness probe of service A causes Kubernetes to enter an infinite restart cascade (restart storm) across your entire cluster.

---

## 3. Kubernetes Deployment Pod Manifest Alignment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      terminationGracePeriodSeconds: 45 # Must be LARGER than Spring timeout-per-shutdown-phase (30s)
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.2.0
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sleep", "5"] # Allow kube-proxy to propagate endpoint removal before SIGTERM
```
- The `preStop` hook sleeps for 5 seconds to guarantee that kube-proxy and Ingress controllers have dropped the pod IP from active routing tables before Spring Boot receives `SIGTERM`.
- `terminationGracePeriodSeconds: 45` provides ample margin above Spring's `timeout-per-shutdown-phase: 30s`.
