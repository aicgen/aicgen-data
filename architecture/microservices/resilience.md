# Microservices Resilience

## Circuit Breaker Pattern

```text
Class CircuitBreaker:
  State: CLOSED | OPEN | HALF_OPEN
  
  Method Execute(operation):
    If State is OPEN:
      If TimeoutExpired:
        State = HALF_OPEN
      Else:
        Throw Error("Circuit Open")
    
    Try:
      Result = operation()
      OnSuccess()
      Return Result
    Catch Error:
      OnFailure()
      Throw Error
```

## Retry with Exponential Backoff

```text
Function Retry(operation, maxAttempts, baseDelay):
  For attempt in 1..maxAttempts:
    Try:
      return operation()
    Catch Error:
      If attempt == maxAttempts: Throw Error
      
      # Exponential Backoff + Jitter
      delay = baseDelay * (2 ^ attempt) + RandomJitter()
      Sleep(delay)
```

## Bulkhead Pattern

```text
# Isolate resources to prevent cascading failures
Class Bulkhead:
  MaxConcurrent = 5
  Active = 0
  
  Method Execute(operation):
    If Active >= MaxConcurrent:
      Throw Error("Bulkhead Full")
      
    Active++
    Try:
      return operation()
    Finally:
      Active--

# Usage: Separate bulkheads per dependency
PaymentBulkhead = New Bulkhead(5)
EmailBulkhead = New Bulkhead(10)
```

## Timeouts

```text
Function WithTimeout(operation, timeoutMs):
  Race:
    1. Result = operation()
    2. Sleep(timeoutMs) -> Throw Error("Timeout")

# Always set timeouts for external calls
Result = WithTimeout(UserService.GetUser(id), 5000)
```

## Graceful Degradation

```text
Function GetProductRecommendations(userId):
  Try:
    return RecommendationService.GetPersonalized(userId)
  Catch Error:
    # Fallback to cached popular items
    Log("Recommendation service unavailable")
    return GetPopularProducts()

# Partial responses instead of complete failure
Function GetDashboard(userId):
  User = GetUser(userId) OR null
  Orders = GetOrders(userId) OR []
  Stats = GetStats(userId) OR null

  return { User, Orders, Stats }
```

## Health Checks

```text
Endpoint GET /health:
  Checks = [
    CheckDatabase(),
    CheckRedis(),
    CheckExternalAPI()
  ]
  
  Healthy = All(Checks) passed
  
  Return HTTP 200/503 {
    status: Healthy ? "healthy" : "degraded",
    checks: { ...details... }
  }
```
