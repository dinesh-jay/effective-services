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
