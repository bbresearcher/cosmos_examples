# Go Context Guide

## What is Context in Go?

Context in Go is a fundamental concept for managing request lifecycles, cancellation, deadlines, and passing request-scoped values across API boundaries. It's defined in the `context` package and is essential for writing robust concurrent applications.

## Core Context Interface

The `context.Context` interface provides four key methods:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

## Main Use Cases

### 1. Cancellation and Timeouts

Context allows you to signal when operations should be cancelled or have exceeded their allowed time:

```go
// Timeout context
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Cancellation context
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// Using context in a function
func fetchData(ctx context.Context) error {
    select {
    case <-time.After(2*time.Second): // simulate work
        return nil
    case <-ctx.Done():
        return ctx.Err() // returns context.Canceled or context.DeadlineExceeded
    }
}
```

### 2. Request-Scoped Values

Context can carry request-scoped data like user IDs, trace IDs, or authentication tokens:

```go
type contextKey string

const userIDKey contextKey = "userID"

// Setting a value
ctx := context.WithValue(context.Background(), userIDKey, "user123")

// Retrieving a value
func getUserID(ctx context.Context) string {
    if userID, ok := ctx.Value(userIDKey).(string); ok {
        return userID
    }
    return ""
}
```

### 3. Goroutine Coordination

Context helps coordinate multiple goroutines and propagate cancellation signals:

```go
func processWithWorkers(ctx context.Context) error {
    var wg sync.WaitGroup
    
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            worker(ctx, workerID)
        }(i)
    }
    
    wg.Wait()
    return ctx.Err()
}

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d stopping: %v\n", id, ctx.Err())
            return
        default:
            // Do work
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

## Common Context Functions

### Creating Contexts

- `context.Background()` - Root context, typically used at program start
- `context.TODO()` - Placeholder when you're unsure which context to use
- `context.WithCancel(parent)` - Returns a context that can be manually cancelled
- `context.WithTimeout(parent, duration)` - Automatically cancels after duration
- `context.WithDeadline(parent, time)` - Cancels at specific time
- `context.WithValue(parent, key, value)` - Adds a key-value pair

## Best Practices

### Always pass context as first parameter

```go
func DoSomething(ctx context.Context, arg1 string, arg2 int) error {
    // implementation
}
```

### Check for cancellation in long-running operations

```go
func longRunningTask(ctx context.Context) error {
    for i := 0; i < 1000; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // do work
        }
    }
    return nil
}
```

### Don't store contexts in structs

```go
// Bad
type Server struct {
    ctx context.Context
}

// Good - pass context to methods
func (s *Server) HandleRequest(ctx context.Context) error {
    // use ctx here
}
```

## Real-World Example

Here's how context is commonly used in HTTP handlers:

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Create context with timeout
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()
    
    // Add request ID for tracing
    requestID := generateRequestID()
    ctx = context.WithValue(ctx, "requestID", requestID)
    
    // Call business logic
    result, err := businessLogic(ctx, getUserInput(r))
    if err != nil {
        if err == context.DeadlineExceeded {
            http.Error(w, "Request timeout", http.StatusRequestTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(result)
}
```

## Understanding Context: Not Just Global Storage

Context is more nuanced than just "global storage." It's better understood as **request-scoped metadata** that flows through your call stack, not general-purpose storage.

### Key Distinctions from Global Storage

#### 1. Scoped, Not Global

```go
// This is scoped to a specific request/operation
ctx := context.WithValue(parentCtx, "userID", "123")
processRequest(ctx) // Only this call chain has access

// Not like a global variable that everything can access
var globalUserID string // ❌ This is truly global
```

#### 2. Immutable Chain

```go
// Each WithValue creates a new context, doesn't modify existing ones
ctx1 := context.WithValue(context.Background(), "key1", "value1")
ctx2 := context.WithValue(ctx1, "key2", "value2")
// ctx1 still only has key1, ctx2 has both keys
```

#### 3. Primary Purpose is Control, Not Data

```go
// Context's main job is lifecycle management
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Data storage is secondary and should be minimal
ctx = context.WithValue(ctx, "traceID", generateTraceID())
```

### What Should and Shouldn't Go in Context

#### ✅ Good context values:
- Request IDs, trace IDs
- User authentication tokens
- Request deadlines/timeouts
- Cancellation signals
- Cross-cutting concerns (logging, metrics)

#### ❌ Bad context values:
- Business logic data
- Function parameters
- Configuration that doesn't change per request
- Large objects or complex data structures

### Better Mental Model

Think of context as a **control channel** that happens to carry some metadata:

```go
func processOrder(ctx context.Context, orderData OrderInfo) error {
    // orderData = actual business data (passed explicitly)
    // ctx = control info (timeouts, user identity, tracing)
    
    select {
    case <-ctx.Done():
        return ctx.Err() // Control: operation cancelled
    default:
        userID := ctx.Value("userID").(string) // Metadata: who made request
        return fulfillOrder(ctx, orderData, userID)
    }
}
```

### Why This Distinction Matters

If you treat context like global storage, you might:
- Put too much data in it (performance impact)
- Make functions hard to test (hidden dependencies)
- Create tight coupling between layers
- Make code harder to reason about

Context works best when you think of it as "the environment this operation is running in" rather than "a place to stash data." The call stack flows the context down, but the context primarily exists to let you control and coordinate that flow, not to replace proper parameter passing.

## Conclusion

Context is thread-safe and designed to be passed down through call stacks, making it an essential tool for building scalable, well-behaved Go applications that can handle cancellation, timeouts, and request-scoped data properly.