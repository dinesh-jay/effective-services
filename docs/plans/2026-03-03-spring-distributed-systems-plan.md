# Spring & Distributed Systems Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add `spring/` and `distributed-systems/` top-level sections with READMEs, operational excellence guides, 5 patterns, and 5 anti-patterns each.

**Architecture:** Two new top-level folders following existing conventions. Each folder gets a README (philosophy, when to use, table of contents), an operational excellence sub-folder, and numbered pattern/anti-pattern files. No `4-spring-boot-starter` sub-folder for either section — spring IS Spring Boot, and distributed-systems patterns use inline code.

**Tech Stack:** Kotlin, Spring Boot 4, Resilience4j (for distributed-systems), Testcontainers, Micrometer, Gradle Kotlin DSL.

**Conventions (from existing files):**
- Pattern files: `# Title` → `## What` → `## Why` → `## How` → `## Key Configuration` or `## Key Considerations` → `---` → `## Code` with fenced kotlin/yaml blocks
- Anti-pattern files: `# Title — Anti-Patterns` → numbered sections each with `**What people do:**` → `**Why it fails:**` → `**Instead:**`
- READMEs: `# Title` → intro sentence → `## Philosophy` → `## When to Use` → `## When NOT to Use` → `## What's in This Library` (table) → `## Tech Stack`
- Operational excellence: `# Title` → sections for Metrics (table), Alerting Rules, Tracing/Health/Dashboards → `---` → `## Code`

---

### Task 1: Spring README

**Files:**
- Create: `spring/README.md`

**Step 1: Create the file**

```markdown
# Spring Boot

Best practices for building production-ready services with Spring Boot 4.

## Philosophy

Spring Boot is an **opinionated framework**. Lean into its conventions. The framework has sensible defaults for configuration, error handling, health checks, and metrics — override them only when you understand what the default does and why it doesn't fit your case.

Prefer **auto-configuration over manual wiring**. Use `@ConfigurationProperties` over `@Value`. Test with real dependencies (Testcontainers) over mocks. Let Spring manage the lifecycle — fight the framework and you lose.

## When to Use Spring Boot

- **REST APIs and web services** — embedded server, auto-configured Jackson, exception handling out of the box.
- **Microservices** — Actuator for observability, externalized configuration, Docker-friendly fat JARs.
- **Event-driven services** — first-class integration with Kafka (spring-kafka), RabbitMQ (spring-amqp), and Temporal.
- **Batch processing** — Spring Batch with restart, skip, and retry built in.
- **Any JVM service that needs production readiness** — health checks, metrics, tracing, graceful shutdown come free.

## When NOT to Use Spring Boot

- **Serverless functions with cold-start constraints** — Spring's startup overhead (1-3s) is too slow. Use Micronaut, Quarkus, or plain Kotlin.
- **Libraries or SDKs** — Spring Boot is for applications, not libraries. A library should not force Spring on its consumers.
- **Simple CLI tools** — a framework designed for long-running services is overkill for a script.
- **When the team doesn't know Spring** — Spring's magic (auto-configuration, proxies, AOP) is productive when understood, but a debugging nightmare when not.

## What's in This Library

| Section | Description |
|---|---|
| [1-operational-excellence/](./1-operational-excellence/) | Actuator, metrics, health checks, structured logging |
| [2-patterns/](./2-patterns/) | Error handling, integration testing, graceful shutdown, configuration, auto-configuration |
| [3-anti-patterns/](./3-anti-patterns/) | Common Spring Boot mistakes and what to do instead |

## Tech Stack

- **Kotlin** — concise, null-safe, interoperable with the Java ecosystem.
- **Spring Boot 4** — framework baseline for all examples.
- **Testcontainers** — real dependencies in integration tests.
- **Micrometer** — vendor-neutral metrics facade.
- **Gradle Kotlin DSL** — build scripts as code with type-safe accessors.
```

**Step 2: Commit**

```bash
git add spring/README.md
git commit -m "docs(spring): add spring section README"
```

---

### Task 2: Spring Operational Excellence

**Files:**
- Create: `spring/1-operational-excellence/README.md`

**Step 1: Create the file**

```markdown
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
```

**Step 2: Commit**

```bash
git add spring/1-operational-excellence/README.md
git commit -m "docs(spring): add operational excellence guide"
```

---

### Task 3: Spring Pattern — Structured Error Handling

**Files:**
- Create: `spring/2-patterns/1-structured-error-handling.md`

**Step 1: Create the file**

```markdown
# Structured Error Handling

## What

Centralize all exception-to-HTTP-response mapping in a single `@ControllerAdvice` class using Spring's `ProblemDetail` (RFC 9457). Every error response follows the same structure.

## Why

Without centralized error handling, each controller handles exceptions differently. Some return plain strings, others return custom JSON, and unhandled exceptions leak stack traces. Clients cannot reliably parse error responses because the format varies.

`ProblemDetail` is a standard (RFC 9457) that gives error responses a consistent shape: `type`, `title`, `status`, `detail`, and optional extension fields.

## How

Create a single `@RestControllerAdvice` class that catches domain exceptions and maps each to a `ProblemDetail` with the appropriate HTTP status. Spring Boot 4 supports `ProblemDetail` natively — set `spring.mvc.problemdetails.enabled=true` to get default handling for standard Spring exceptions.

For domain-specific exceptions, define a sealed hierarchy and handle each in the advice class.

## Key Considerations

- Return `ProblemDetail` from all error paths — including validation errors, 404s, and 500s.
- Set `type` to a URI that identifies the error class (e.g., `urn:error:order-not-found`). Clients switch on `type`, not `status`.
- Never expose internal details (stack traces, SQL errors) in production. Use `detail` for user-facing messages only.
- Log the full exception server-side before returning the sanitized response.

---

## Code

### Enable ProblemDetail Support

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

### Domain Exceptions

```kotlin
sealed class DomainException(message: String) : RuntimeException(message)

class OrderNotFoundException(val orderId: String)
    : DomainException("Order not found: $orderId")

class InsufficientStockException(val productId: String, val requested: Int, val available: Int)
    : DomainException("Insufficient stock for product $productId: requested $requested, available $available")

class PaymentDeclinedException(val orderId: String, val reason: String)
    : DomainException("Payment declined for order $orderId: $reason")
```

### Global Exception Handler

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(OrderNotFoundException::class)
    fun handleNotFound(ex: OrderNotFoundException): ProblemDetail {
        log.warn("Order not found: {}", ex.orderId)
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.message!!).apply {
            title = "Order Not Found"
            type = URI.create("urn:error:order-not-found")
            setProperty("orderId", ex.orderId)
        }
    }

    @ExceptionHandler(InsufficientStockException::class)
    fun handleInsufficientStock(ex: InsufficientStockException): ProblemDetail {
        log.warn("Insufficient stock: {}", ex.message)
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.message!!).apply {
            title = "Insufficient Stock"
            type = URI.create("urn:error:insufficient-stock")
            setProperty("productId", ex.productId)
        }
    }

    @ExceptionHandler(Exception::class)
    fun handleUnexpected(ex: Exception): ProblemDetail {
        log.error("Unexpected error", ex)
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred",
        ).apply {
            title = "Internal Server Error"
            type = URI.create("urn:error:internal")
        }
    }
}
```
```

**Step 2: Commit**

```bash
git add spring/2-patterns/1-structured-error-handling.md
git commit -m "docs(spring): add structured error handling pattern"
```

---

### Task 4: Spring Pattern — Testcontainers Integration Testing

**Files:**
- Create: `spring/2-patterns/2-testcontainers-integration-testing.md`

**Step 1: Create the file**

```markdown
# Testcontainers Integration Testing

## What

Use Testcontainers to run real infrastructure (Postgres, Kafka, Redis) in Docker during integration tests. Replace mocks of infrastructure with actual instances. Use Spring Boot's `@ServiceConnection` for zero-config container wiring.

## Why

Mocks lie. A mock `JdbcTemplate` does not catch SQL syntax errors, missing columns, or transaction isolation bugs. A mock `KafkaTemplate` does not catch serialization failures or topic misconfiguration. Integration tests with real infrastructure catch the bugs that matter — the ones that happen in production.

Testcontainers starts throwaway Docker containers per test run and tears them down automatically.

## How

Add Testcontainers dependencies. Define containers as `@Bean` in a test configuration class with `@ServiceConnection` — Spring Boot auto-configures the datasource, Kafka bootstrap servers, or Redis connection from the container. No manual property overrides needed.

Use `@SpringBootTest` for full-context tests. Use slice tests (`@DataJpaTest`, `@WebMvcTest`) when you only need part of the context.

## Key Considerations

- **Reuse containers across tests** — set `testcontainers.reuse.enable=true` in `~/.testcontainers.properties` for faster local development.
- **Slice tests for speed** — use `@DataJpaTest` with Testcontainers for repository tests. Full `@SpringBootTest` for end-to-end flows.
- **CI compatibility** — Testcontainers requires Docker. Most CI systems provide Docker-in-Docker or a Docker socket.
- **Startup time** — containers add 5-15s to test startup. Acceptable for integration tests, too slow for unit tests.

---

## Code

### Dependencies (Gradle)

```kotlin
testImplementation("org.springframework.boot:spring-boot-testcontainers")
testImplementation("org.testcontainers:junit-jupiter")
testImplementation("org.testcontainers:postgresql")
testImplementation("org.testcontainers:kafka")
```

### Test Configuration

```kotlin
@TestConfiguration(proxyBeanMethods = false)
class TestContainersConfig {

    @Bean
    @ServiceConnection
    fun postgres(): PostgreSQLContainer<*> {
        return PostgreSQLContainer("postgres:16-alpine")
    }

    @Bean
    @ServiceConnection
    fun kafka(): KafkaContainer {
        return KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))
    }
}
```

### Integration Test

```kotlin
@SpringBootTest
@Import(TestContainersConfig::class)
class OrderServiceIntegrationTest {

    @Autowired lateinit var orderService: OrderService
    @Autowired lateinit var orderRepository: OrderRepository

    @Test
    fun `creates order and persists to database`() {
        val order = orderService.createOrder(CreateOrderRequest(
            productId = "PROD-1",
            quantity = 2,
        ))

        assertThat(order.id).isNotNull()
        assertThat(orderRepository.findById(order.id!!)).isPresent
    }
}
```

### Slice Test with Testcontainers

```kotlin
@DataJpaTest
@Import(TestContainersConfig::class)
class OrderRepositoryTest {

    @Autowired lateinit var repository: OrderRepository

    @Test
    fun `finds orders by status`() {
        repository.save(Order(status = OrderStatus.PENDING))
        repository.save(Order(status = OrderStatus.COMPLETED))

        val pending = repository.findByStatus(OrderStatus.PENDING)

        assertThat(pending).hasSize(1)
        assertThat(pending.first().status).isEqualTo(OrderStatus.PENDING)
    }
}
```
```

**Step 2: Commit**

```bash
git add spring/2-patterns/2-testcontainers-integration-testing.md
git commit -m "docs(spring): add testcontainers integration testing pattern"
```

---

### Task 5: Spring Pattern — Graceful Shutdown

**Files:**
- Create: `spring/2-patterns/3-graceful-shutdown.md`

**Step 1: Create the file**

```markdown
# Graceful Shutdown

## What

When the application receives a shutdown signal (SIGTERM), it stops accepting new requests, drains in-flight requests, and then exits cleanly. No requests are dropped. No database transactions are left half-committed.

## Why

In Kubernetes, a rolling deployment sends SIGTERM to the old pod while routing traffic to the new one. Without graceful shutdown, in-flight requests get killed mid-execution — users see 502s, database transactions roll back, and Kafka offsets may not commit.

## How

Spring Boot 4 supports graceful shutdown natively. Enable it in configuration, set a timeout, and coordinate with the health check so the load balancer stops sending traffic before the server stops accepting it.

The shutdown sequence:
1. SIGTERM received
2. Readiness probe returns DOWN — load balancer stops routing new traffic
3. Server stops accepting new connections
4. In-flight requests complete (up to the timeout)
5. Application context closes, beans are destroyed
6. Process exits

## Key Considerations

- **Set a shutdown timeout** — without a timeout, a stuck request blocks shutdown indefinitely. 30s is a reasonable default.
- **Coordinate with Kubernetes** — set `terminationGracePeriodSeconds` in the pod spec to be longer than the Spring shutdown timeout.
- **Drain non-HTTP work** — Kafka consumers, scheduled tasks, and thread pools also need graceful shutdown. Use `@PreDestroy` or `DisposableBean`.
- **Pre-stop hook** — add a short sleep (5s) in the Kubernetes pre-stop hook to give the load balancer time to deregister the pod before Spring starts draining.

---

## Code

### Enable Graceful Shutdown

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### Kubernetes Deployment Snippet

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        periodSeconds: 5
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        periodSeconds: 10
```

### Graceful Shutdown for Custom Thread Pool

```kotlin
@Configuration
class AsyncConfig {

    @Bean(destroyMethod = "shutdown")
    fun taskExecutor(): ThreadPoolTaskExecutor {
        return ThreadPoolTaskExecutor().apply {
            corePoolSize = 4
            maxPoolSize = 8
            setWaitForTasksToCompleteOnShutdown(true)
            setAwaitTerminationSeconds(30)
            setThreadNamePrefix("async-")
        }
    }
}
```
```

**Step 2: Commit**

```bash
git add spring/2-patterns/3-graceful-shutdown.md
git commit -m "docs(spring): add graceful shutdown pattern"
```

---

### Task 6: Spring Pattern — Configuration Properties

**Files:**
- Create: `spring/2-patterns/4-configuration-properties.md`

**Step 1: Create the file**

```markdown
# Configuration Properties

## What

Use `@ConfigurationProperties` to bind externalized configuration to type-safe Kotlin data classes with validation. One class per configuration concern, validated at startup.

## Why

`@Value("${some.property}")` scattered across the codebase is fragile: no type safety, no validation, no discoverability. You find out a property is missing at runtime when the code path executes, not at startup. Typos in property names fail silently.

`@ConfigurationProperties` gives you a single source of truth per configuration concern, validated eagerly at startup. IDE support, refactoring, and documentation come free.

## How

Create a Kotlin data class annotated with `@ConfigurationProperties(prefix = "app.feature")`. Use `@Validated` with Jakarta Bean Validation annotations to enforce constraints. Enable property scanning with `@ConfigurationPropertiesScan` on your main application class (or `@EnableConfigurationProperties` on specific classes).

## Key Considerations

- **One class per concern** — `OrderProperties`, `PaymentProperties`, not one giant `AppProperties`.
- **Validate at startup** — add `@Validated` and use `@NotBlank`, `@Positive`, `@URL` etc. A missing required property fails fast at startup.
- **Use `data class` with defaults** — sensible defaults reduce required configuration. Override in environment-specific profiles.
- **Prefix naming** — use `app.<feature>` to namespace your properties and avoid collisions with Spring's own properties.

---

## Code

### Configuration Properties Class

```kotlin
@Validated
@ConfigurationProperties(prefix = "app.orders")
data class OrderProperties(
    @field:NotBlank
    val serviceName: String = "order-service",

    @field:Positive
    val maxRetries: Int = 3,

    @field:Positive
    val timeoutSeconds: Long = 30,

    @field:NotBlank
    val deadLetterTopic: String = "orders.DLT",
)
```

### Application Class

```kotlin
@SpringBootApplication
@ConfigurationPropertiesScan
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}
```

### Usage in Service

```kotlin
@Service
class OrderService(
    private val orderProperties: OrderProperties,
) {
    fun processOrder(orderId: String) {
        val timeout = Duration.ofSeconds(orderProperties.timeoutSeconds)
        // use typed, validated config
    }
}
```

### YAML Configuration

```yaml
app:
  orders:
    service-name: order-service
    max-retries: 5
    timeout-seconds: 60
    dead-letter-topic: orders.DLT
```

### Profile Override

```yaml
# application-prod.yaml
app:
  orders:
    max-retries: 10
    timeout-seconds: 120
```
```

**Step 2: Commit**

```bash
git add spring/2-patterns/4-configuration-properties.md
git commit -m "docs(spring): add configuration properties pattern"
```

---

### Task 7: Spring Pattern — Custom Auto-Configuration

**Files:**
- Create: `spring/2-patterns/5-custom-auto-configuration.md`

**Step 1: Create the file**

```markdown
# Custom Auto-Configuration

## What

Write your own Spring Boot starter that auto-configures beans from properties. Consumers add the dependency and get sensible defaults — no explicit `@Bean` definitions needed. Every bean is `@ConditionalOnMissingBean` so consumers can override.

## Why

When multiple services need the same infrastructure wiring (HTTP clients, error handlers, metrics setup), copy-pasting configuration across projects leads to drift. A starter centralizes the wiring: one place to update, one place to test, consistent behavior everywhere.

## How

Create a separate module (e.g., `my-starter`). Define a `@ConfigurationProperties` class for the knobs you want to expose. Write an `@AutoConfiguration` class that creates beans conditionally. Register the auto-configuration class in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

## Key Considerations

- **`@ConditionalOnMissingBean` on every bean** — consumers must be able to replace your defaults without fighting your auto-configuration.
- **`@ConditionalOnClass` for optional dependencies** — only configure a bean if the required library is on the classpath.
- **No `@ComponentScan`** — auto-configurations must be explicit about what they register. Scanning pulls in unintended beans.
- **Test with `ApplicationContextRunner`** — verify beans are created, overridden, and conditionally excluded without starting a full Spring context.

---

## Code

### Configuration Properties

```kotlin
@ConfigurationProperties(prefix = "app.http-client")
data class HttpClientProperties(
    val connectTimeoutMs: Long = 5000,
    val readTimeoutMs: Long = 10000,
    val maxRetries: Int = 3,
)
```

### Auto-Configuration

```kotlin
@AutoConfiguration
@EnableConfigurationProperties(HttpClientProperties::class)
@ConditionalOnClass(RestClient::class)
class HttpClientAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    fun restClient(properties: HttpClientProperties): RestClient {
        return RestClient.builder()
            .requestFactory(clientHttpRequestFactory(properties))
            .build()
    }

    private fun clientHttpRequestFactory(properties: HttpClientProperties): ClientHttpRequestFactory {
        return SimpleClientHttpRequestFactory().apply {
            setConnectTimeout(Duration.ofMillis(properties.connectTimeoutMs))
            setReadTimeout(Duration.ofMillis(properties.readTimeoutMs))
        }
    }
}
```

### Registration File

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```text
com.myapp.starter.HttpClientAutoConfiguration
```

### Test with ApplicationContextRunner

```kotlin
class HttpClientAutoConfigurationTest {

    private val runner = ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(HttpClientAutoConfiguration::class.java))

    @Test
    fun `creates RestClient with default properties`() {
        runner.run { context ->
            assertThat(context).hasSingleBean(RestClient::class.java)
        }
    }

    @Test
    fun `backs off when user defines own RestClient`() {
        runner
            .withBean(RestClient::class.java, { RestClient.create() })
            .run { context ->
                assertThat(context).hasSingleBean(RestClient::class.java)
            }
    }
}
```
```

**Step 2: Commit**

```bash
git add spring/2-patterns/5-custom-auto-configuration.md
git commit -m "docs(spring): add custom auto-configuration pattern"
```

---

### Task 8: Spring Anti-Patterns

**Files:**
- Create: `spring/3-anti-patterns/1-structured-error-handling.md`
- Create: `spring/3-anti-patterns/2-testcontainers-integration-testing.md`
- Create: `spring/3-anti-patterns/3-graceful-shutdown.md`
- Create: `spring/3-anti-patterns/4-configuration-properties.md`
- Create: `spring/3-anti-patterns/5-custom-auto-configuration.md`

**Step 1: Create all five anti-pattern files**

`spring/3-anti-patterns/1-structured-error-handling.md`:

```markdown
# Structured Error Handling — Anti-Patterns

## 1. Catching Exceptions in Every Controller

**What people do:** Every controller method has its own try-catch block returning a custom error map.

**Why it fails:** Error response format varies by controller. Some return `{"error": "..."}`, others `{"message": "..."}`, others plain strings. Clients cannot write a single error parser. When a new exception type appears, every controller needs updating.

**Instead:** Use a single `@RestControllerAdvice` with `ProblemDetail`. One place handles all exceptions, one format for all errors.

## 2. Leaking Stack Traces in Responses

**What people do:** Let Spring's default error handling return the full exception stacktrace in the response body.

**Why it fails:** Stack traces expose internal class names, library versions, and SQL queries. This is a security risk (information disclosure) and useless to API consumers.

**Instead:** Handle all exceptions in `@RestControllerAdvice`. Log the full exception server-side. Return a sanitized `ProblemDetail` with a user-facing message.

## 3. Using HTTP Status Codes as the Only Error Signal

**What people do:** Return 400 for every client error, 500 for every server error, with no additional context.

**Why it fails:** Clients cannot distinguish between "validation failed" and "resource not found" — both are 400. Debugging requires reading log timestamps to correlate server logs with client errors.

**Instead:** Use `ProblemDetail.type` as a machine-readable error identifier (e.g., `urn:error:order-not-found`). Clients switch on `type`, not just `status`.
```

`spring/3-anti-patterns/2-testcontainers-integration-testing.md`:

```markdown
# Integration Testing — Anti-Patterns

## 1. Mocking the Database

**What people do:** Mock `JdbcTemplate` or `JpaRepository` in service tests and verify method calls.

**Why it fails:** Mocks don't catch SQL syntax errors, missing columns, wrong transaction isolation, or N+1 queries. The test passes but the code fails against a real database.

**Instead:** Use Testcontainers with `@DataJpaTest` or `@SpringBootTest`. Test against a real PostgreSQL instance that matches production.

## 2. Sharing a Single Test Database Across All Tests

**What people do:** Point all tests at a shared H2 or PostgreSQL instance without isolation.

**Why it fails:** Tests leak state. Test A inserts rows that cause Test B to fail. Tests pass individually but fail when run together. Order-dependent test suites are a maintenance nightmare.

**Instead:** Use Testcontainers with a fresh container per test class (or `@Transactional` to roll back after each test). Each test starts with a clean state.

## 3. Using H2 as a Test Database

**What people do:** Use H2 in-memory mode for tests because it is fast and requires no Docker.

**Why it fails:** H2 is not PostgreSQL. Window functions, JSON operators, CTEs, and many DDL features differ. Tests pass on H2 and fail on the real database. You end up maintaining two sets of SQL — one for H2 compatibility and one for production.

**Instead:** Use Testcontainers with the same database engine and version as production. The 5-10 second startup cost is worth the confidence.
```

`spring/3-anti-patterns/3-graceful-shutdown.md`:

```markdown
# Graceful Shutdown — Anti-Patterns

## 1. No Shutdown Configuration

**What people do:** Deploy to Kubernetes with default Spring Boot settings. SIGTERM kills the process immediately.

**Why it fails:** In-flight HTTP requests get 502 errors. Database transactions roll back. Kafka consumer offsets are not committed, causing reprocessing on restart.

**Instead:** Set `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase=30s`. Spring drains in-flight requests before stopping.

## 2. Kubernetes terminationGracePeriodSeconds Too Short

**What people do:** Set Spring's shutdown timeout to 30s but leave Kubernetes `terminationGracePeriodSeconds` at the default 30s.

**Why it fails:** Kubernetes sends SIGTERM and starts the 30s countdown. The pre-stop hook, load balancer deregistration, and Spring's drain phase all share that 30s. If they exceed it, Kubernetes sends SIGKILL — ungraceful shutdown.

**Instead:** Set `terminationGracePeriodSeconds` to at least Spring's timeout + pre-stop hook duration + buffer (e.g., 60s).

## 3. Ignoring Non-HTTP Workloads

**What people do:** Enable graceful shutdown for the HTTP server but forget about Kafka consumers, scheduled tasks, and custom thread pools.

**Why it fails:** The HTTP server drains cleanly, but background threads keep running until SIGKILL. Kafka consumer offsets are not committed. Scheduled tasks execute partially.

**Instead:** Use `@PreDestroy` or `DisposableBean` for custom thread pools. Set `setWaitForTasksToCompleteOnShutdown(true)` on `ThreadPoolTaskExecutor`. Kafka consumers shut down through Spring's lifecycle automatically when using spring-kafka.
```

`spring/3-anti-patterns/4-configuration-properties.md`:

```markdown
# Configuration Properties — Anti-Patterns

## 1. Using @Value Everywhere

**What people do:** Inject properties with `@Value("${app.feature.timeout}")` directly in service classes.

**Why it fails:** No type safety — a typo in the property name fails silently with a null or default. No validation — a negative timeout or missing required value is only caught at runtime. Properties are scattered across the codebase with no single source of truth.

**Instead:** Use `@ConfigurationProperties` with a data class. One class per concern, validated at startup, discoverable in one place.

## 2. One Giant Configuration Class

**What people do:** Create a single `AppProperties` class that holds every property in the application.

**Why it fails:** The class grows into a god object. Changes to unrelated features touch the same file. Testing requires constructing the entire config even when you only need one property. Auto-complete shows 50 properties when you need one.

**Instead:** One `@ConfigurationProperties` class per concern: `OrderProperties`, `PaymentProperties`, `NotificationProperties`. Each class is small, focused, and independently testable.

## 3. No Startup Validation

**What people do:** Define configuration properties without `@Validated` or validation annotations.

**Why it fails:** A missing required property or invalid value is not caught until the code path executes — possibly in production, possibly at 2 AM. The error message is a `NullPointerException` deep in business logic, not a clear "property X is required".

**Instead:** Add `@Validated` to the properties class and use `@NotBlank`, `@Positive`, `@URL` etc. The application fails fast at startup with a clear message.
```

`spring/3-anti-patterns/5-custom-auto-configuration.md`:

```markdown
# Custom Auto-Configuration — Anti-Patterns

## 1. No @ConditionalOnMissingBean

**What people do:** Define beans in auto-configuration without `@ConditionalOnMissingBean`.

**Why it fails:** Consumers who define their own bean get a `BeanDefinitionOverrideException` or unpredictable behavior depending on bean registration order. The starter is no longer a sensible default — it is a mandate.

**Instead:** Always use `@ConditionalOnMissingBean` on every bean in an auto-configuration. The consumer's explicit bean definition always wins.

## 2. Using @ComponentScan in Auto-Configuration

**What people do:** Add `@ComponentScan` to the auto-configuration class to pick up `@Component` and `@Service` classes from the starter module.

**Why it fails:** `@ComponentScan` is greedy — it scans the package and all sub-packages. If the consumer's code is in a parent or sibling package, the scan pulls in unintended beans from the consumer's application. Debugging unexpected bean registrations is painful.

**Instead:** Explicitly declare every bean in the `@AutoConfiguration` class with `@Bean` methods. No scanning, no surprises.

## 3. Not Testing Conditional Logic

**What people do:** Write auto-configuration with `@ConditionalOnClass`, `@ConditionalOnProperty`, etc. but only test the happy path (all conditions met).

**Why it fails:** When a condition is not met, the bean should be absent — but you never verified that. A refactoring breaks a condition, a bean is always created, and the starter forces a dependency that should be optional.

**Instead:** Use `ApplicationContextRunner` to test both paths: condition met (bean present) and condition not met (bean absent). Test overrides too — verify that a user-defined bean replaces the auto-configured one.
```

**Step 2: Commit**

```bash
git add spring/3-anti-patterns/
git commit -m "docs(spring): add anti-patterns for all spring patterns"
```

---

### Task 9: Distributed Systems README

**Files:**
- Create: `distributed-systems/README.md`

**Step 1: Create the file**

```markdown
# Distributed Systems

Best practices for building resilient distributed services.

## Philosophy

Distributed systems fail in **partial, unpredictable ways**. A network call can succeed, fail, or hang indefinitely — and you cannot tell which until a timeout fires. Design for failure: isolate blast radius with bulkheads, fail fast with circuit breakers, and degrade gracefully with fallbacks.

Every inter-service call is a liability. Each one introduces latency, a failure mode, and a debugging surface. Make calls resilient by default — retry transient failures, shed load when overwhelmed, and circuit-break when a dependency is down.

## When to Use These Patterns

- **Service-to-service HTTP/gRPC calls** — any synchronous call across a network boundary.
- **Integration with external APIs** — third-party services are unreliable by definition.
- **Microservice architectures** — the more services, the more failure modes.
- **Any system with SLA requirements** — resilience patterns are how you meet uptime guarantees.

## When NOT to Use These Patterns

- **Single-process monolith** — in-process method calls do not need circuit breakers or retries.
- **Fire-and-forget async messaging** — Kafka and message queues handle retries at the infrastructure level.
- **When simplicity matters more** — a startup with two services and no SLA does not need a full resilience stack. Add patterns as failure modes emerge.

## What's in This Library

| Section | Description |
|---|---|
| [1-operational-excellence/](./1-operational-excellence/) | Distributed tracing, cross-service observability, SLOs |
| [2-patterns/](./2-patterns/) | Circuit breaker, retry, rate limiting, bulkhead, service discovery |
| [3-anti-patterns/](./3-anti-patterns/) | Common mistakes and what to do instead |

## Tech Stack

- **Kotlin** — concise, null-safe, interoperable with the Java ecosystem.
- **Spring Boot 4** — framework baseline for all examples.
- **Resilience4j** — lightweight, modular resilience library for the JVM.
- **Micrometer** — metrics from Resilience4j decorators.
- **Gradle Kotlin DSL** — build scripts as code with type-safe accessors.
```

**Step 2: Commit**

```bash
git add distributed-systems/README.md
git commit -m "docs(distributed-systems): add distributed systems section README"
```

---

### Task 10: Distributed Systems Operational Excellence

**Files:**
- Create: `distributed-systems/1-operational-excellence/README.md`

**Step 1: Create the file**

```markdown
# Distributed Systems Operational Excellence

Distributed tracing, cross-service observability, and resilience metrics for production distributed services.

## Distributed Tracing

Use OpenTelemetry with Micrometer Tracing to propagate trace context across service boundaries. Every HTTP request generates a trace ID that flows through all downstream calls — HTTP, Kafka, gRPC.

### Key Metrics to Monitor

| Metric | What It Tells You |
|--------|-------------------|
| `resilience4j.circuitbreaker.state` | Circuit state per backend: 0=closed, 1=open, 2=half-open. Open = dependency is down. |
| `resilience4j.circuitbreaker.failure.rate` | Failure percentage. Approaches the threshold before the breaker opens. |
| `resilience4j.retry.calls` | Retry attempts by outcome (success, retry, failure). High retries = flaky dependency. |
| `resilience4j.bulkhead.available.concurrent.calls` | Available capacity. Zero = all permits used, requests are being rejected. |
| `resilience4j.ratelimiter.available.permissions` | Remaining rate limit permits. Zero = throttling active. |
| `http.client.requests` (by target service) | Latency and error rate per downstream dependency. |

### Alerting Rules

- Circuit breaker OPEN for > 1 minute — dependency is down, investigate.
- Retry rate > 20% of total calls — dependency is flaky, may need attention.
- Bulkhead saturation (available = 0) for > 30s — increase permits or scale dependency.
- Rate limiter rejections > 0 — traffic exceeds budget, check for load spikes.
- Cross-service P99 latency > SLO — degraded performance, check tracing.

## Health Checks

Expose the state of circuit breakers and bulkheads via Actuator health indicators. A circuit breaker in OPEN state should make the readiness probe return DOWN for that specific dependency (not the entire service).

## Dashboards

Recommended Grafana panels:
1. Circuit breaker state by backend (state timeline)
2. Request rate and error rate by downstream service (line chart)
3. Retry attempts vs successes (stacked bar)
4. Bulkhead utilization (gauge)
5. Cross-service trace latency distribution (heatmap)

---

## Code

### OpenTelemetry Tracing Configuration

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # lower in production (e.g., 0.1)
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
```

### Resilience4j Metrics Configuration

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        register-health-indicator: true
    metrics:
      enabled: true
  retry:
    metrics:
      enabled: true
  bulkhead:
    metrics:
      enabled: true
```

### Circuit Breaker Health Indicator

```kotlin
@Component
class CircuitBreakerHealthIndicator(
    private val circuitBreakerRegistry: CircuitBreakerRegistry,
) : HealthIndicator {

    override fun health(): Health {
        val openBreakers = circuitBreakerRegistry.allCircuitBreakers
            .filter { it.state == CircuitBreaker.State.OPEN }
            .map { it.name }

        return if (openBreakers.isEmpty()) {
            Health.up().build()
        } else {
            Health.down()
                .withDetail("openCircuitBreakers", openBreakers)
                .build()
        }
    }
}
```
```

**Step 2: Commit**

```bash
git add distributed-systems/1-operational-excellence/README.md
git commit -m "docs(distributed-systems): add operational excellence guide"
```

---

### Task 11: Distributed Systems Pattern — Circuit Breaker

**Files:**
- Create: `distributed-systems/2-patterns/1-circuit-breaker.md`

**Step 1: Create the file**

```markdown
# Circuit Breaker

## What

Wrap calls to a downstream service with a circuit breaker that monitors failure rates. When failures exceed a threshold, the breaker **opens** and fails fast without making the call. After a cooldown, it moves to **half-open** and lets a few calls through to test recovery.

## Why

Without a circuit breaker, a failing downstream service causes every caller to hang until the timeout fires. Threads pile up waiting for responses that will never come. The caller's thread pool saturates, and the failure cascades to its own callers. One failing service takes down the entire call chain.

A circuit breaker stops the cascade: once the failure rate proves the dependency is down, it fails immediately. Callers get a fast error instead of a slow timeout.

## How

Use Resilience4j's `CircuitBreaker` with Spring Boot auto-configuration. Annotate the method that makes the downstream call with `@CircuitBreaker`. Configure failure rate threshold, wait duration in open state, and the number of calls in the sliding window.

Provide a fallback method that returns a degraded response (cached data, default value, or a meaningful error) instead of propagating the exception.

## Key Considerations

- **Sliding window size** — too small and the breaker flaps on a few errors. Too large and it reacts slowly. 10-20 calls is a reasonable starting point.
- **Failure rate threshold** — 50% is the default. Lower it for critical dependencies where even 20% failures indicate a problem.
- **Wait duration in open state** — how long before the breaker tries half-open. 30-60s balances fast recovery with not hammering a struggling service.
- **Record only relevant exceptions** — don't count 400 Bad Request as a failure (that is a client bug, not a dependency failure). Only count 5xx and timeouts.

---

## Code

### Dependencies (Gradle)

```kotlin
implementation("io.github.resilience4j:resilience4j-spring-boot3")
implementation("io.github.resilience4j:resilience4j-micrometer")
```

### Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.myapp.exceptions.BadRequestException
```

### Service with Circuit Breaker

```kotlin
@Service
class PaymentClient(
    private val restClient: RestClient,
) {

    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallback")
    fun chargePayment(request: PaymentRequest): PaymentResponse {
        return restClient.post()
            .uri("/payments")
            .body(request)
            .retrieve()
            .body(PaymentResponse::class.java)!!
    }

    private fun fallback(request: PaymentRequest, ex: Exception): PaymentResponse {
        log.warn("Payment service unavailable, circuit open: {}", ex.message)
        return PaymentResponse(
            status = PaymentStatus.PENDING,
            message = "Payment service temporarily unavailable, will retry",
        )
    }
}
```
```

**Step 2: Commit**

```bash
git add distributed-systems/2-patterns/1-circuit-breaker.md
git commit -m "docs(distributed-systems): add circuit breaker pattern"
```

---

### Task 12: Distributed Systems Pattern — Retry with Backoff

**Files:**
- Create: `distributed-systems/2-patterns/2-retry-with-backoff.md`

**Step 1: Create the file**

```markdown
# Retry with Backoff

## What

Automatically retry failed calls to a downstream service with increasing delays between attempts. Add jitter to prevent synchronized retries from multiple callers.

## Why

Transient failures — network blips, brief service restarts, garbage collection pauses — are normal in distributed systems. A single retry often succeeds. But retrying immediately and indefinitely turns a transient failure into a sustained load spike. Exponential backoff gives the downstream service time to recover. Jitter prevents the thundering herd when all callers retry at the same intervals.

## How

Use Resilience4j's `Retry` with Spring Boot auto-configuration. Configure max attempts, wait duration, and exponential backoff. Combine with `@CircuitBreaker` — the circuit breaker wraps the retry. If all retries fail, the circuit breaker counts it as one failure.

## Key Considerations

- **Max attempts** — 3 is usually enough. More than 5 means the failure is not transient.
- **Exponential backoff** — double the wait each attempt (1s, 2s, 4s). Gives the dependency time to recover.
- **Jitter** — add randomness to the backoff to desynchronize retries from multiple callers.
- **Retry budget** — at the system level, limit retries to a percentage of total traffic (e.g., 20%). Without a budget, retries during an outage multiply load by the retry count.
- **Idempotency** — only retry idempotent operations. Retrying a non-idempotent POST can create duplicate resources.
- **Ordering** — place retry inside circuit breaker: `CircuitBreaker(Retry(call))`. The circuit breaker sees one call (with retries) as one attempt.

---

## Code

### Configuration

```yaml
resilience4j:
  retry:
    instances:
      payment-service:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2.0
        enable-randomized-wait: true
        randomized-wait-factor: 0.5
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.myapp.exceptions.BadRequestException
```

### Service with Retry and Circuit Breaker

```kotlin
@Service
class PaymentClient(
    private val restClient: RestClient,
) {

    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallback")
    @Retry(name = "payment-service")
    fun chargePayment(request: PaymentRequest): PaymentResponse {
        return restClient.post()
            .uri("/payments")
            .body(request)
            .retrieve()
            .body(PaymentResponse::class.java)!!
    }

    private fun fallback(request: PaymentRequest, ex: Exception): PaymentResponse {
        log.warn("Payment service unavailable after retries: {}", ex.message)
        return PaymentResponse(status = PaymentStatus.PENDING)
    }
}
```
```

**Step 2: Commit**

```bash
git add distributed-systems/2-patterns/2-retry-with-backoff.md
git commit -m "docs(distributed-systems): add retry with backoff pattern"
```

---

### Task 13: Distributed Systems Pattern — Rate Limiting

**Files:**
- Create: `distributed-systems/2-patterns/3-rate-limiting.md`

**Step 1: Create the file**

```markdown
# Rate Limiting

## What

Limit the rate at which your service calls a downstream dependency or accepts incoming requests. Prevent overwhelming a service beyond its capacity.

## Why

Without rate limiting, a traffic spike propagates through the entire system. A marketing campaign doubles traffic, your service forwards all of it to the payment provider, the payment provider throttles you with 429s, and your service burns retries on throttled requests — making the overload worse.

Rate limiting protects both directions: **outbound** (don't overwhelm dependencies) and **inbound** (don't let callers overwhelm you).

## How

Use Resilience4j's `RateLimiter` for client-side rate limiting. Configure the number of permissions per period and the timeout for acquiring a permit. When the limit is reached, calls either wait (up to the timeout) or fail immediately.

For server-side rate limiting, use Spring's `RateLimiter` or an API gateway. Resilience4j is best for controlling outbound call rates.

## Key Considerations

- **Per-instance vs global** — Resilience4j rate limiting is per-JVM instance. If you have 4 instances with a limit of 100/s each, the downstream sees 400/s. For global rate limiting, use Redis-backed solutions or an API gateway.
- **Timeout vs fail-fast** — `timeoutDuration > 0` queues requests up to the timeout. `timeoutDuration = 0` rejects immediately. Choose based on whether callers can wait.
- **Combine with circuit breaker** — rate limiter prevents overload, circuit breaker handles failures. Use both: `CircuitBreaker(RateLimiter(call))`.
- **Monitor rejections** — a high rejection rate means either the limit is too low or traffic is genuinely too high. Alert on sustained rejections.

---

## Code

### Configuration

```yaml
resilience4j:
  ratelimiter:
    instances:
      payment-service:
        limit-for-period: 50          # 50 calls per period
        limit-refresh-period: 1s       # reset every second
        timeout-duration: 500ms        # wait up to 500ms for a permit
```

### Service with Rate Limiting

```kotlin
@Service
class PaymentClient(
    private val restClient: RestClient,
) {

    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallback")
    @RateLimiter(name = "payment-service")
    @Retry(name = "payment-service")
    fun chargePayment(request: PaymentRequest): PaymentResponse {
        return restClient.post()
            .uri("/payments")
            .body(request)
            .retrieve()
            .body(PaymentResponse::class.java)!!
    }

    private fun fallback(request: PaymentRequest, ex: Exception): PaymentResponse {
        log.warn("Payment service call failed: {}", ex.message)
        return PaymentResponse(status = PaymentStatus.PENDING)
    }
}
```
```

**Step 2: Commit**

```bash
git add distributed-systems/2-patterns/3-rate-limiting.md
git commit -m "docs(distributed-systems): add rate limiting pattern"
```

---

### Task 14: Distributed Systems Pattern — Bulkhead Isolation

**Files:**
- Create: `distributed-systems/2-patterns/4-bulkhead-isolation.md`

**Step 1: Create the file**

```markdown
# Bulkhead Isolation

## What

Limit the number of concurrent calls to a downstream service. If one dependency is slow, only its allocated threads/permits are consumed — other dependencies remain unaffected.

## Why

Without bulkheads, all outbound calls share the same thread pool. A slow dependency (e.g., payment service responding in 30s instead of 100ms) consumes all available threads. The inventory service, notification service, and every other dependency are now starved of threads. One slow service takes down all outbound communication.

Bulkheads isolate the blast radius: the payment service can consume at most N threads, leaving the rest available for healthy dependencies.

## How

Use Resilience4j's `Bulkhead` (semaphore-based) or `ThreadPoolBulkhead` (thread-pool-based). The semaphore bulkhead limits concurrent calls on the caller's thread. The thread pool bulkhead runs calls on a dedicated thread pool with a bounded queue.

Semaphore bulkhead is simpler and lower overhead. Thread pool bulkhead provides full isolation but adds thread-switching cost.

## Key Considerations

- **Semaphore vs thread pool** — use semaphore for most cases. Use thread pool when you need true isolation (e.g., a dependency that blocks the calling thread).
- **Max concurrent calls** — size based on the dependency's capacity and expected latency. Too low = unnecessary rejections. Too high = no protection.
- **Max wait duration** — how long to wait for a permit. Zero fails fast. Set to a small value (100-500ms) if callers can tolerate brief queuing.
- **Combine with other patterns** — typical ordering: `CircuitBreaker(Bulkhead(RateLimiter(Retry(call))))`.

---

## Code

### Configuration (Semaphore Bulkhead)

```yaml
resilience4j:
  bulkhead:
    instances:
      payment-service:
        max-concurrent-calls: 10
        max-wait-duration: 100ms
      inventory-service:
        max-concurrent-calls: 20
        max-wait-duration: 0ms         # fail fast
```

### Service with Bulkhead

```kotlin
@Service
class PaymentClient(
    private val restClient: RestClient,
) {

    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallback")
    @Bulkhead(name = "payment-service")
    @Retry(name = "payment-service")
    fun chargePayment(request: PaymentRequest): PaymentResponse {
        return restClient.post()
            .uri("/payments")
            .body(request)
            .retrieve()
            .body(PaymentResponse::class.java)!!
    }

    private fun fallback(request: PaymentRequest, ex: Exception): PaymentResponse {
        log.warn("Payment service bulkhead full or failing: {}", ex.message)
        return PaymentResponse(status = PaymentStatus.PENDING)
    }
}
```

### Thread Pool Bulkhead (When Full Isolation Needed)

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      legacy-service:
        max-thread-pool-size: 5
        core-thread-pool-size: 3
        queue-capacity: 10
        keep-alive-duration: 100ms
```
```

**Step 2: Commit**

```bash
git add distributed-systems/2-patterns/4-bulkhead-isolation.md
git commit -m "docs(distributed-systems): add bulkhead isolation pattern"
```

---

### Task 15: Distributed Systems Pattern — Service Discovery

**Files:**
- Create: `distributed-systems/2-patterns/5-service-discovery.md`

**Step 1: Create the file**

```markdown
# Service Discovery

## What

Services find each other dynamically at runtime instead of using hardcoded URLs. A service registry maintains the list of available instances and their network locations. Callers query the registry to get a healthy instance.

## Why

Hardcoded URLs break when services scale (new instances), move (new IPs), or fail (dead instances). Updating URLs requires redeployment. Service discovery makes the system self-healing: new instances register automatically, failed instances are deregistered, and callers always get a healthy target.

## How

Two approaches:

**Client-side discovery** — the caller queries the registry and selects an instance. Spring Cloud `DiscoveryClient` + `@LoadBalanced RestClient` handles this. The caller has full control over load balancing.

**Server-side discovery** — a load balancer sits between the caller and the registry. The caller sends requests to the load balancer, which routes to a healthy instance. Kubernetes Services work this way.

In Kubernetes, prefer server-side discovery (Kubernetes Services). Outside Kubernetes, use client-side discovery with Spring Cloud.

## Key Considerations

- **Kubernetes-native** — if you are on Kubernetes, use Kubernetes Services. No need for Eureka, Consul, or other registries. DNS-based discovery is built in.
- **Health checks** — the registry must remove unhealthy instances. In Kubernetes, readiness probes handle this. In Eureka, configure heartbeat intervals and eviction.
- **Client-side load balancing** — Spring Cloud LoadBalancer supports round-robin and random. For more sophisticated strategies, configure a custom `ServiceInstanceListSupplier`.
- **DNS caching** — JVM caches DNS lookups. In Kubernetes, set `networkaddress.cache.ttl=10` to pick up new pods quickly.

---

## Code

### Kubernetes Service Discovery (Server-Side)

No code needed — Kubernetes DNS resolves service names automatically:

```yaml
# application.yml
app:
  payment-service:
    url: http://payment-service.default.svc.cluster.local:8080
```

```kotlin
@Service
class PaymentClient(
    @Value("\${app.payment-service.url}") private val baseUrl: String,
) {
    private val restClient = RestClient.builder().baseUrl(baseUrl).build()

    fun chargePayment(request: PaymentRequest): PaymentResponse {
        return restClient.post()
            .uri("/payments")
            .body(request)
            .retrieve()
            .body(PaymentResponse::class.java)!!
    }
}
```

### Spring Cloud Client-Side Discovery

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-kubernetes-client-all")
    // or for non-Kubernetes:
    // implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
}
```

```kotlin
@Configuration
class RestClientConfig {

    @Bean
    @LoadBalanced
    fun restClientBuilder(): RestClient.Builder {
        return RestClient.builder()
    }
}
```

```kotlin
@Service
class PaymentClient(
    restClientBuilder: RestClient.Builder,
) {
    // "payment-service" resolves via the service registry
    private val restClient = restClientBuilder.baseUrl("http://payment-service").build()

    fun chargePayment(request: PaymentRequest): PaymentResponse {
        return restClient.post()
            .uri("/payments")
            .body(request)
            .retrieve()
            .body(PaymentResponse::class.java)!!
    }
}
```

### JVM DNS Cache Configuration

```kotlin
// In application startup or via JVM args: -Dnetworkaddress.cache.ttl=10
Security.setProperty("networkaddress.cache.ttl", "10")
```
```

**Step 2: Commit**

```bash
git add distributed-systems/2-patterns/5-service-discovery.md
git commit -m "docs(distributed-systems): add service discovery pattern"
```

---

### Task 16: Distributed Systems Anti-Patterns

**Files:**
- Create: `distributed-systems/3-anti-patterns/1-circuit-breaker.md`
- Create: `distributed-systems/3-anti-patterns/2-retry-with-backoff.md`
- Create: `distributed-systems/3-anti-patterns/3-rate-limiting.md`
- Create: `distributed-systems/3-anti-patterns/4-bulkhead-isolation.md`
- Create: `distributed-systems/3-anti-patterns/5-service-discovery.md`

**Step 1: Create all five anti-pattern files**

`distributed-systems/3-anti-patterns/1-circuit-breaker.md`:

```markdown
# Circuit Breaker — Anti-Patterns

## 1. No Circuit Breaker at All

**What people do:** Call downstream services directly with only a timeout. Hope for the best.

**Why it fails:** When the dependency is down, every call hangs until the timeout fires. With a 10s timeout and 100 concurrent requests, you have 100 threads blocked for 10s each. Your thread pool saturates and the failure cascades to your callers. One failing dependency takes down the entire service.

**Instead:** Wrap every outbound service call with a circuit breaker. When the failure rate exceeds the threshold, fail immediately instead of waiting for the timeout.

## 2. Circuit Breaking on Client Errors

**What people do:** Configure the circuit breaker to record all exceptions, including 400 Bad Request and 404 Not Found.

**Why it fails:** Client errors (4xx) are not dependency failures — they are bugs in the caller's request. A batch of malformed requests opens the circuit breaker, and now valid requests are also rejected. The dependency is healthy but the breaker says it is down.

**Instead:** Only record server errors (5xx) and timeouts. Use `ignore-exceptions` or `record-exceptions` to exclude client errors.

## 3. No Fallback

**What people do:** Let the circuit breaker throw `CallNotPermittedException` when open, which propagates as a 500 to the caller.

**Why it fails:** The whole point of a circuit breaker is graceful degradation. Returning a 500 is not graceful — it is just a faster failure. The caller still sees an error and may cascade it further.

**Instead:** Provide a fallback method that returns a degraded response: cached data, a default value, or a meaningful error with a retry-after hint.
```

`distributed-systems/3-anti-patterns/2-retry-with-backoff.md`:

```markdown
# Retry with Backoff — Anti-Patterns

## 1. Retrying Without Backoff

**What people do:** Retry immediately on failure, with no delay between attempts.

**Why it fails:** If the dependency is overloaded, immediate retries add more load at the worst possible time. 100 callers each retrying 3 times immediately means 300 requests hit the struggling service simultaneously. The overload gets worse, not better.

**Instead:** Use exponential backoff (1s, 2s, 4s) with jitter. Give the dependency time to recover between attempts.

## 2. Unbounded Retries

**What people do:** Set max retries to a high number (10+) or retry indefinitely.

**Why it fails:** During an outage, each caller generates 10x the normal load in retries. If the outage lasts minutes, retry traffic dominates the network. When the dependency recovers, it faces a wall of queued retries that can immediately overload it again.

**Instead:** Limit retries to 3 attempts. If 3 retries fail, the issue is not transient. Use a circuit breaker to stop retrying entirely when the failure rate proves the dependency is down.

## 3. Retrying Non-Idempotent Operations

**What people do:** Retry POST requests that create resources or trigger side effects (send email, charge payment).

**Why it fails:** The first request may have succeeded but the response was lost (network timeout). The retry creates a duplicate order, sends a second email, or charges the customer twice.

**Instead:** Only retry idempotent operations (GET, PUT with idempotency key). For non-idempotent operations, use an idempotency key pattern — the server deduplicates based on a client-provided key.
```

`distributed-systems/3-anti-patterns/3-rate-limiting.md`:

```markdown
# Rate Limiting — Anti-Patterns

## 1. No Rate Limiting

**What people do:** Forward all incoming traffic to downstream dependencies without any rate control.

**Why it fails:** A traffic spike (marketing campaign, bot scraping, retry storm) propagates through the entire system. The downstream dependency gets overwhelmed, starts returning errors, and those errors trigger retries — amplifying the overload. This is the thundering herd problem.

**Instead:** Rate-limit outbound calls to match the dependency's capacity. Rate-limit inbound traffic to match your own capacity. Shed excess load early rather than letting it cascade.

## 2. Per-Instance Limits Without Coordination

**What people do:** Set a rate limit of 100/s per instance, deploy 10 instances, and the downstream sees 1000/s when the provider's limit is 500/s.

**Why it fails:** Per-instance rate limiting does not account for the total number of instances. Auto-scaling makes it worse — adding instances increases the aggregate rate.

**Instead:** Divide the global limit by the expected instance count with a safety margin. For precise global limiting, use a centralized rate limiter (Redis-backed or API gateway).

## 3. No Backpressure Signal

**What people do:** Rate-limit and silently drop or reject requests without informing the caller.

**Why it fails:** The caller does not know it is being rate-limited. It may retry the rejected request, increasing load. Monitoring does not capture the rejected traffic, hiding the real demand signal.

**Instead:** Return HTTP 429 with a `Retry-After` header. Log and metric every rejection. Callers can back off or queue. Operators can see the true traffic demand.
```

`distributed-systems/3-anti-patterns/4-bulkhead-isolation.md`:

```markdown
# Bulkhead Isolation — Anti-Patterns

## 1. Shared Thread Pool for All Dependencies

**What people do:** Use the default HTTP client thread pool (or Tomcat's thread pool) for all outbound calls.

**Why it fails:** One slow dependency consumes all available threads. A payment service responding in 30s instead of 100ms holds 300x more threads per request. The shared pool saturates and every other dependency — inventory, notifications, user service — is starved. The entire service hangs.

**Instead:** Use Resilience4j bulkheads to limit concurrent calls per dependency. Each dependency gets its own concurrency budget. A slow payment service consumes at most N threads, leaving the rest available.

## 2. Bulkhead Too Large

**What people do:** Set max concurrent calls to 100 "just in case" when the dependency can only handle 20 concurrent requests.

**Why it fails:** The bulkhead does not protect the dependency — it allows 100 concurrent calls, all of which timeout because the dependency can only handle 20. You still saturate threads, just slightly fewer than without a bulkhead.

**Instead:** Size the bulkhead based on the dependency's actual capacity and expected latency. If the dependency handles 20 concurrent requests with 100ms latency, set the bulkhead to 20-25.

## 3. No Monitoring of Bulkhead Saturation

**What people do:** Configure bulkheads but do not alert when they saturate.

**Why it fails:** Bulkhead saturation means requests are being rejected. Without monitoring, you do not know traffic exceeds the dependency's capacity. The service appears healthy (no crashes) but is silently dropping requests.

**Instead:** Monitor `resilience4j.bulkhead.available.concurrent.calls`. Alert when it stays at zero for more than 30 seconds. Either increase the bulkhead, scale the dependency, or investigate the latency increase.
```

`distributed-systems/3-anti-patterns/5-service-discovery.md`:

```markdown
# Service Discovery — Anti-Patterns

## 1. Hardcoded Service URLs

**What people do:** Put IP addresses or hostnames directly in configuration: `http://10.0.1.42:8080/api`.

**Why it fails:** When the service moves to a new IP (scaling, migration, restart), the caller breaks. Updating requires redeployment of every caller. In dynamic environments (Kubernetes, auto-scaling), IPs change constantly.

**Instead:** Use DNS-based service discovery (Kubernetes Services) or a service registry (Eureka, Consul). Services register themselves and callers resolve by name, not IP.

## 2. Long DNS Cache TTL

**What people do:** Use the JVM's default DNS cache (which may cache indefinitely) or set a long TTL.

**Why it fails:** When a service instance dies and a new one starts with a different IP, the caller keeps sending traffic to the dead IP because the DNS entry is cached. The JVM's default `networkaddress.cache.ttl` is 30s for successful lookups, but some environments override this to infinity.

**Instead:** Set `networkaddress.cache.ttl=10` in Kubernetes environments. DNS should resolve fresh every 10 seconds to pick up new pods.

## 3. No Health Checks on Registered Instances

**What people do:** Register service instances but do not configure health checks in the registry.

**Why it fails:** A crashed instance stays registered. The load balancer or client-side discovery sends traffic to a dead instance, getting timeouts or connection refused errors. Users see intermittent failures that are hard to diagnose because "some requests work and some don't."

**Instead:** Configure health checks at every level: readiness probes in Kubernetes, heartbeats in Eureka, health endpoints in the service. Unhealthy instances must be deregistered within seconds.
```

**Step 2: Commit**

```bash
git add distributed-systems/3-anti-patterns/
git commit -m "docs(distributed-systems): add anti-patterns for all distributed systems patterns"
```

---

### Task 17: Update Root README

**Files:**
- Modify: `README.md`

**Step 1: Update the README to include the new sections**

Add the new sections to the root README's description:

```markdown
# service-excellence
A living library of patterns for building production ready services.

## Sections

| Section | Description |
|---|---|
| [kafka/](./kafka/) | Event streaming with Apache Kafka |
| [temporal/](./temporal/) | Durable execution with Temporal |
| [spring/](./spring/) | Spring Boot framework patterns |
| [distributed-systems/](./distributed-systems/) | Resilience and communication patterns for distributed services |
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update root README with spring and distributed-systems sections"
```

---

Plan complete and saved to `docs/plans/2026-03-03-spring-distributed-systems-plan.md`.

Two execution options:

**1. Subagent-Driven (this session)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Parallel Session (separate)** — Open a new session with executing-plans, batch execution with checkpoints.

Which approach?