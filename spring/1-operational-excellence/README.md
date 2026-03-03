# Spring Boot Operational Excellence

Actuator, metrics, health checks, and structured logging for production Spring Boot services.

## Actuator

Spring Boot Actuator exposes operational endpoints out of the box. Expose only what you need in production.

### Key Endpoints

| Endpoint | What It Gives You |
|----------|-------------------|
| `/actuator/health` | Liveness and readiness probes. Auto-checks for DB, Kafka, Redis, disk space. |
| `/actuator/metrics` | Micrometer metrics. JVM, HTTP, custom business metrics. |
| `/actuator/info` | Build info, git commit, custom info contributors. |
| `/actuator/prometheus` | Prometheus scrape endpoint (with `micrometer-registry-prometheus`). |

### Alerting Rules

- Health endpoint returns DOWN — service cannot serve traffic.
- HTTP 5xx rate > 1% — application errors need investigation.
- JVM heap usage > 80% sustained — tune GC or increase memory.
- Thread pool active threads = max threads — requests are queuing.

## Structured Logging

Use structured JSON logging in production for machine parsing. Spring Boot 4 supports structured logging natively with Logback's `JsonLayout` or `logstash-logback-encoder`.

Include correlation IDs (trace ID from Micrometer Tracing) in every log line for cross-service tracing.

## Health Check Design

Use health groups to separate **liveness** (is the process alive?) from **readiness** (can it serve traffic?).

- Liveness: JVM is running, no deadlocks. Failing liveness = restart the pod.
- Readiness: database connected, Kafka broker reachable. Failing readiness = stop sending traffic.

## Dashboards

Recommended Grafana panels:
1. HTTP request rate by status code (line chart)
2. P99/P95/P50 response latency (heatmap)
3. JVM heap usage and GC pause time (line chart)
4. Active threads vs max threads (line chart)
5. Health check status (stat panel with alert)

---

## Code

### Actuator Exposure Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when_authorized
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, kafka
  info:
    git:
      mode: full
```

### Custom Health Indicator

```kotlin
@Component
class ExternalApiHealthIndicator(
    private val restClient: RestClient,
) : HealthIndicator {

    override fun health(): Health {
        return try {
            restClient.get().uri("/health").retrieve().toBodilessEntity()
            Health.up().build()
        } catch (e: Exception) {
            Health.down(e).build()
        }
    }
}
```

### Structured Logging Configuration

```yaml
logging:
  structured:
    format: logstash
  level:
    root: INFO
    com.myapp: DEBUG
```

### Custom Info Contributor

```kotlin
@Component
class AppInfoContributor : InfoContributor {

    override fun contribute(builder: Info.Builder) {
        builder.withDetail("app", mapOf(
            "name" to "my-service",
            "description" to "Order processing service",
        ))
    }
}
```
