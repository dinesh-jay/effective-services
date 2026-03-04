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
