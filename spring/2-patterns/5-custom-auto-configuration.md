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
