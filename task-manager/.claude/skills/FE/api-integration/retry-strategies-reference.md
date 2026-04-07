# Retry Strategies Reference

Detailed patterns for implementing resilient retry logic in API clients.

## Exponential Backoff Algorithm

### Basic Implementation

```typescript
interface RetryConfig {
  maxRetries: number;
  baseDelay: number; // milliseconds
  maxDelay: number;
  retryableStatuses: number[];
}

const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxRetries: 3,
  baseDelay: 500,
  maxDelay: 30000,
  retryableStatuses: [408, 429, 500, 502, 503, 504],
};

function calculateBackoff(
  attempt: number,
  baseDelay: number,
  maxDelay: number
): number {
  const exponential = baseDelay * Math.pow(2, attempt);
  const jitter = Math.random() * baseDelay * 0.5; // 0-50% jitter
  return Math.min(exponential + jitter, maxDelay);
}

async function withRetry<T>(
  operation: () => Promise<T>,
  config: RetryConfig = DEFAULT_RETRY_CONFIG
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;

      // Check if error is retryable
      if (!isRetryableError(error, config.retryableStatuses)) {
        throw error;
      }

      // Don't wait after last attempt
      if (attempt < config.maxRetries) {
        const delay = calculateBackoff(
          attempt,
          config.baseDelay,
          config.maxDelay
        );
        await sleep(delay);
      }
    }
  }

  throw lastError!;
}

function isRetryableError(
  error: unknown,
  retryableStatuses: number[]
): boolean {
  if (error instanceof ApiError) {
    return retryableStatuses.includes(error.statusCode);
  }
  // Network errors are retryable
  if (error instanceof TypeError && error.message.includes('fetch')) {
    return true;
  }
  return false;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

## Respecting Retry-After Headers

When the server provides `Retry-After`, use it instead of calculated backoff:

```typescript
function getRetryDelay(
  response: Response,
  attempt: number,
  config: RetryConfig
): number {
  const retryAfter = response.headers.get('Retry-After');

  if (retryAfter) {
    // Retry-After can be seconds or HTTP date
    const seconds = parseInt(retryAfter, 10);
    if (!isNaN(seconds)) {
      return seconds * 1000; // Convert to ms
    }

    // Try parsing as HTTP date
    const date = new Date(retryAfter);
    if (!isNaN(date.getTime())) {
      return Math.max(0, date.getTime() - Date.now());
    }
  }

  // Fall back to exponential backoff
  return calculateBackoff(attempt, config.baseDelay, config.maxDelay);
}
```

## Idempotency Considerations

### Safe Methods (Always Retry)

- GET — No side effects
- HEAD — No side effects
- OPTIONS — No side effects

### Idempotent Methods (Safe to Retry)

- PUT — Replaces resource entirely
- DELETE — Deleting twice has same effect

### Non-Idempotent Methods (Retry with Caution)

- POST — May create duplicates
- PATCH — May apply changes multiple times

For POST/PATCH, only retry if:

1. Backend provides idempotency keys
2. Request failed before reaching server (network error)
3. Response was 408 timeout (server may not have processed)

```typescript
function shouldRetry(
  method: string,
  error: ApiError,
  hasIdempotencyKey: boolean
): boolean {
  // Always retry safe methods
  if (['GET', 'HEAD', 'OPTIONS'].includes(method)) {
    return error.isRetryable();
  }

  // Retry idempotent methods
  if (['PUT', 'DELETE'].includes(method)) {
    return error.isRetryable();
  }

  // For POST/PATCH, only retry if:
  // - Has idempotency key, OR
  // - Network error (never reached server), OR
  // - 408 timeout
  if (['POST', 'PATCH'].includes(method)) {
    if (hasIdempotencyKey) return error.isRetryable();
    if (error.statusCode === 408) return true;
    if (error.message.includes('network')) return true;
    return false;
  }

  return false;
}
```

## Circuit Breaker Pattern

Prevent cascading failures by stopping requests when service is consistently failing:

```typescript
enum CircuitState {
  CLOSED, // Normal operation
  OPEN, // Failing, reject immediately
  HALF_OPEN, // Testing if service recovered
}

class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private nextAttempt = Date.now();

  constructor(
    private threshold: number = 5, // Failures before opening
    private timeout: number = 60000, // ms before half-open
    private successThreshold: number = 2 // Successes to close
  ) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = CircuitState.HALF_OPEN;
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;

    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.successCount = 0;
      }
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.successCount = 0;

    if (this.failureCount >= this.threshold) {
      this.state = CircuitState.OPEN;
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

## Retry Budget

Limit total retry rate to prevent overwhelming the system:

```typescript
class RetryBudget {
  private retries = 0;
  private requests = 0;
  private windowStart = Date.now();

  constructor(
    private maxRetryRatio: number = 0.1, // 10% of requests can retry
    private windowMs: number = 60000 // 1 minute window
  ) {}

  canRetry(): boolean {
    this.resetWindowIfNeeded();

    const retryRatio = this.requests === 0 ? 0 : this.retries / this.requests;
    return retryRatio < this.maxRetryRatio;
  }

  recordRequest(): void {
    this.resetWindowIfNeeded();
    this.requests++;
  }

  recordRetry(): void {
    this.resetWindowIfNeeded();
    this.retries++;
  }

  private resetWindowIfNeeded(): void {
    const now = Date.now();
    if (now - this.windowStart > this.windowMs) {
      this.retries = 0;
      this.requests = 0;
      this.windowStart = now;
    }
  }
}
```

## Complete Retry-Enabled Client

```typescript
class ResilientApiClient {
  private circuitBreaker = new CircuitBreaker();
  private retryBudget = new RetryBudget();

  async request<T>(config: RequestConfig): Promise<T> {
    this.retryBudget.recordRequest();

    return this.circuitBreaker.execute(() =>
      withRetry(
        async () => {
          const response = await fetch(config.url, config);

          if (!response.ok) {
            throw new ApiError(
              response.status,
              await response.text()
            );
          }

          return response.json();
        },
        {
          maxRetries: 3,
          baseDelay: 500,
          maxDelay: 30000,
          retryableStatuses: [408, 429, 500, 502, 503, 504],
          shouldRetry: (error, attempt) => {
            if (!this.retryBudget.canRetry()) return false;
            this.retryBudget.recordRetry();
            return shouldRetry(config.method, error, !!config.idempotencyKey);
          },
        }
      )
    );
  }
}
```

## Testing Retry Logic

```typescript
describe('Retry Logic', () => {
  it('retries on 503 and succeeds', async () => {
    const mock = jest
      .fn()
      .mockRejectedValueOnce(new ApiError(503, 'Service Unavailable'))
      .mockRejectedValueOnce(new ApiError(503, 'Service Unavailable'))
      .mockResolvedValueOnce({ data: 'success' });

    const result = await withRetry(mock, {
      maxRetries: 3,
      baseDelay: 10,
      maxDelay: 100,
      retryableStatuses: [503],
    });

    expect(result).toEqual({ data: 'success' });
    expect(mock).toHaveBeenCalledTimes(3);
  });

  it('does not retry on 400', async () => {
    const mock = jest
      .fn()
      .mockRejectedValue(new ApiError(400, 'Bad Request'));

    await expect(
      withRetry(mock, {
        maxRetries: 3,
        baseDelay: 10,
        maxDelay: 100,
        retryableStatuses: [503],
      })
    ).rejects.toThrow('Bad Request');

    expect(mock).toHaveBeenCalledTimes(1);
  });

  it('respects Retry-After header', async () => {
    const retryAfterMs = 2000;
    const start = Date.now();

    const mock = jest
      .fn()
      .mockRejectedValueOnce(
        new ApiError(429, 'Too Many Requests', {
          headers: { 'Retry-After': '2' },
        })
      )
      .mockResolvedValueOnce({ data: 'success' });

    await withRetry(mock, {
      maxRetries: 1,
      baseDelay: 100,
      maxDelay: 5000,
      retryableStatuses: [429],
    });

    const elapsed = Date.now() - start;
    expect(elapsed).toBeGreaterThanOrEqual(retryAfterMs);
  });
});
```

## Key Takeaways

- **Always add jitter** to prevent thundering herd
- **Respect `Retry-After`** headers from server
- **Cap max delay** to prevent indefinite waits
- **Don't retry non-idempotent operations** without idempotency keys
- **Use circuit breakers** for sustained failures
- **Track retry budget** to prevent retry storms
- **Test retry logic** with controlled failures
