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
