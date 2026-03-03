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
