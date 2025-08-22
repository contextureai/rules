---
title: Go Context Usage Patterns
description: Proper context propagation, cancellation, and value storage patterns
tags: [go, context, best-practices]
trigger:
  type: glob
  globs: ["**/*.go", "!**/*_test.go"]
languages: [go]
variables:
  extended: false
---
{{if not .extended}}
## Core Rules
1.  **Pass `context.Context` as the first parameter** in a function call, right after the receiver for methods. Name it `ctx`.
2.  **Never store `context.Context` in a struct**. Pass it explicitly through the call chain.
3.  **Never pass a `nil` `context.Context`**. If you are unsure what to use, pass `context.TODO()`.
4.  **Propagate the existing context**, don't create a new one from `context.Background()` within a function.
5.  **Call `cancel`**: For any context created with `context.WithCancel`, `WithTimeout`, or `WithDeadline`, you must call its `cancel` function to release resources. The idiomatic way is with `defer cancel()`.

## Cancellation
- The primary purpose of `context` is to enable cancellation. Downstream functions should check for cancellation and stop work.
- Use `select` with `ctx.Done()` to handle cancellation in long-running operations or when waiting on channels.

```go
func longOperation(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err() // e.g., context.Canceled or context.DeadlineExceeded
    case <-time.After(10 * time.Second):
        // ... operation completed
    }
    return nil
}
```

## Timeouts and Deadlines
- Use `context.WithTimeout` for operations that should not exceed a certain duration.
- Use `context.WithDeadline` for operations that must be completed by a specific time.
- Always `defer cancel()` when creating these contexts.

## Context Values
- Use context values only for request-scoped data that transits process and API boundaries, such as a request ID or security credentials.
- **Do not use context values to pass optional parameters to functions**. This makes the API obscure. Pass them as explicit arguments instead.
- For type safety and to avoid key collisions, define a custom, unexported type for your context keys.

```go
// Use an unexported type for the key to prevent collisions.
type contextKey string

const requestIDKey = contextKey("requestID")

// Setter function
func withRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

// Getter function
func getRequestID(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(requestIDKey).(string)
    return id, ok
}
```
{{else}}
## Context Rules

1. **First parameter** (after receiver)
2. **Never store in structs**
3. **Never nil** - use context.TODO() if unsure
4. **Pass through** - don't create new root contexts
5. **Cancel what you create**

## Parameter Position

```go
// ✅ CORRECT: Context is first parameter
func requestWithContext(ctx context.Context, id string) error {
    return process(ctx, id)
}

// ✅ CORRECT: After receiver for methods  
func (s *API) HandleRequest(ctx context.Context, req Request) (Response, error) {
    return s.process(ctx, req)
}

// ❌ WRONG: Context not first
func process(id string, ctx context.Context) error { // Bad position
    return nil
}

// ❌ WRONG: Storing context in struct
type BadAPI struct {
    ctx context.Context // Never do this!
    data string
}
```

## Context Creation

```go
// ✅ Root contexts (only in main, init, or tests)
func main() {
    ctx := context.Background()
    if err := run(ctx); err != nil {
        log.Fatal(err)
    }
}

func TestFeature(t *testing.T) {
    ctx := context.Background() // OK in tests
    result := process(ctx)
}

// ✅ TODO when refactoring
func oldFunction() {
    // Temporary during migration
    ctx := context.TODO()
    newFunction(ctx)
}

// ❌ Don't create new root contexts arbitrarily
func handler(ctx context.Context) {
    ctx = context.Background() // Wrong! Breaks cancellation chain
    process(ctx)
}
```

## Cancellation Patterns

```go
// ✅ Cancel what you create
func requestWithTimeout(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel() // Always call cancel
    
    return doRequest(ctx)
}

// ✅ Manual cancellation
func handleWithAPI(ctx context.Context) error {
    ctx, cancel := context.WithCancel(ctx)
    
    // Cancel on condition
    go func() {
        <-shutdownSignal
        cancel()
    }()
    
    return longRunningRequest(ctx)
}

// ✅ Deadline propagation
func processRequest(ctx context.Context) error {
    deadline, ok := ctx.Deadline()
    if ok {
        log.Printf("Operation must complete by %v", deadline)
    }
    
    // Respect context cancellation
    select {
    case <-ctx.Done():
        return ctx.Err()
    case result := <-doWork():
        return processResult(result)
    }
}
```

## Context Values

```go
// Define typed keys for context values
type contextKey string

const (
    requestIDKey contextKey = "requestID"
    userIDKey    contextKey = "userID"
)

// ✅ Set context values
func withRequestID(ctx context.Context, requestID string) context.Context {
    return context.WithValue(ctx, requestIDKey, requestID)
}

// ✅ Get context values safely
func getRequestID(ctx context.Context) string {
    if id, ok := ctx.Value(requestIDKey).(string); ok {
        return id
    }
    return ""
}

// ❌ Don't use context for optional parameters
func badDesign(ctx context.Context) {
    // Don't do this - pass as parameter instead
    config := ctx.Value("config").(Config)
}

// ✅ Use context only for request-scoped values
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := generateID()
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## HTTP Server Context

```go
func (s *API) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // ✅ Get context from request
    ctx := r.Context()
    
    // ✅ Add timeout for this endpoint
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()
    
    // Process with context
    result, err := s.processRequest(ctx)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(result)
}
```

## Database Operations

```go
// ✅ Pass context to database operations
func (s *Store) GetRequest(ctx context.Context, id string) (*Request, error) {
    query := `SELECT * FROM requests WHERE id = $1`
    
    row := s.db.QueryRowContext(ctx, query, id)
    
    var item Request
    err := row.Scan(&item.ID, &item.Name)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound
        }
        return nil, fmt.Errorf("query request: %w", err)
    }
    
    return &item, nil
}

// ✅ Transaction with context
func (s *Store) UpdateRequest(ctx context.Context, item *Request) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback() // No-op if committed
    
    query := `UPDATE requests SET name = $2 WHERE id = $1`
    if _, err := tx.ExecContext(ctx, query, item.ID, item.Name); err != nil {
        return fmt.Errorf("update: %w", err)
    }
    
    return tx.Commit()
}
```

## Goroutine Context Propagation

```go
// ✅ Pass context to goroutines
func processRequests(ctx context.Context, items []Request) error {
    g, ctx := errgroup.WithContext(ctx)
    
    for _, item := range items {
        item := item // Capture loop variable
        g.Go(func() error {
            return processItem(ctx, item)
        })
    }
    
    return g.Wait()
}

// ✅ Respect cancellation in goroutines
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            log.Printf("Worker cancelled: %v", ctx.Err())
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            process(ctx, job)
        }
    }
}
```

## Testing with Context

```go
func TestRequestWithTimeout(t *testing.T) {
    // Create test context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()
    
    // Test cancellation behavior
    err := longRunningRequest(ctx)
    if !errors.Is(err, context.DeadlineExceeded) {
        t.Errorf("expected deadline exceeded, got %v", err)
    }
}

func TestRequestCancellation(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    
    // Cancel after delay
    go func() {
        time.Sleep(50 * time.Millisecond)
        cancel()
    }()
    
    err := processRequest(ctx)
    if !errors.Is(err, context.Canceled) {
        t.Errorf("expected canceled, got %v", err)
    }
}
```

## Context Anti-patterns

```go
// ❌ NEVER: Optional function parameters via context
func badFunc(ctx context.Context) {
    verbose := ctx.Value("verbose").(bool) // Wrong!
}

// ✅ GOOD: Explicit parameters
func goodFunc(ctx context.Context, verbose bool) {
    // Use verbose directly
}

// ❌ NEVER: Pass nil context
processRequest(nil, data) // Wrong!

// ✅ GOOD: Use TODO if migrating
processRequest(context.TODO(), data)

// ❌ NEVER: Store context in struct
type BadService struct {
    ctx context.Context // Never!
}

// ✅ GOOD: Pass context to methods
type GoodService struct {
    // fields, but no context
}

func (s *GoodService) Process(ctx context.Context) error {
    // Use ctx here
}
```
{{end}}
