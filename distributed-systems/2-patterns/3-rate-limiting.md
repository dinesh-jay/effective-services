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
