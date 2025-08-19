---
title: Go Performance and Anti-patterns
description: Optimize performance correctly and avoid common Go anti-patterns
tags: [go, performance, optimization, anti-patterns, best-practices]
trigger:
  type: model
  description: When optimizing Go code performance or reviewing for common mistakes
languages: [go]
variables:
    extended: false
---

# Go Performance and Anti-patterns

{{if not .extended}}
## Performance Principles
1.  **Measure First**: Use benchmarks (`go test -bench`) and profiling (`pprof`) to identify actual bottlenecks. Don't guess.
2.  **Optimize Algorithms**: A better algorithm (e.g., O(n)) is more impactful than micro-optimizing a bad one (e.g., O(n²)).
3.  **Prioritize Clarity**: Don't sacrifice readable code for minor performance gains. Document any non-obvious optimizations with benchmark data.

## Common Optimizations
- **String Concatenation**: In loops, use `strings.Builder` instead of the `+` operator to avoid repeated memory allocations. For simple joins, `strings.Join` is sufficient.
- **Pre-allocation**: When the size is known, pre-allocate slices (`make([]T, len, cap)`) and maps (`make(map[K]V, size)`) to prevent reallocations and resizing.
- **Buffer Reuse**: Use `sync.Pool` to reuse memory for temporary buffers (like `bytes.Buffer`) in high-throughput scenarios.

## Common Anti-Patterns
1.  **Using `panic` for Errors**: `panic` is for unrecoverable errors. For expected errors, always return an `error` value.
2.  **Ignoring Errors**: Never ignore an error with `_`. Always check and handle them.
3.  **Goroutine Leaks**: Ensure every goroutine has a clear exit condition. Use `context` for cancellation in long-running goroutines.
4.  **Global Variables**: Avoid global mutable state. Use dependency injection to pass dependencies (like configuration or database connections) explicitly.
5.  **`interface{}` Overuse**: Prefer specific types over the empty interface. Use generics (Go 1.18+) for type-safe, general-purpose code.
6.  **Generic Package Names**: Avoid names like `util`, `common`, or `helpers`. Package names should be descriptive and specific.
7.  **`Get` Prefix on Getters**: In Go, getters are named after the noun they retrieve (e.g., `user.Name()` not `user.GetName()`).
{{else}}
## Performance Optimization Rules

1. **Measure first** - Use benchmarks and pprof
2. **Optimize algorithms** before micro-optimizations
3. **Readability over minor gains** unless proven critical
4. **Document optimizations** with benchmarks

## String Building Performance

```go
// ❌ BAD: String concatenation in loop (O(n²))
func badConcat(items []string) string {
    var result string
    for _, item := range items {
        result += item + ", " // Creates new string each time
    }
    return result
}

// ✅ GOOD: strings.Builder (O(n))
func goodBuilder(items []string) string {
    var b strings.Builder
    b.Grow(len(items) * 10) // Pre-allocate if size known
    
    for i, item := range items {
        if i > 0 {
            b.WriteString(", ")
        }
        b.WriteString(item)
    }
    return b.String()
}

// ✅ OK: strings.Join for simple cases
func simpleJoin(items []string) string {
    return strings.Join(items, ", ")
}

// ✅ OK: fmt.Sprintf for formatting
func formatUser(u User) string {
    return fmt.Sprintf("%s (%d)", u.Name, u.ID)
}
```

## Slice Preallocation

```go
// ❌ BAD: Growing slice without preallocation
func badSlice() []User {
    var users []User
    for i := 0; i < 1000; i++ {
        users = append(users, fetchUser(i))
    }
    return users
}

// ✅ GOOD: Preallocate when size is known
func goodSlice() []User {
    users := make([]User, 0, 1000)
    for i := 0; i < 1000; i++ {
        users = append(users, fetchUser(i))
    }
    return users
}

// ✅ BETTER: Direct indexing if size is fixed
func betterSlice() []User {
    users := make([]User, 1000)
    for i := 0; i < 1000; i++ {
        users[i] = fetchUser(i)
    }
    return users
}
```

## Map Preallocation

```go
// ✅ Preallocate maps when size is predictable
func processItems(items []Item) map[string]Result {
    // Preallocate with expected size
    results := make(map[string]Result, len(items))
    
    for _, item := range items {
        results[item.ID] = processItem(item)
    }
    return results
}

// Check existence before access
if val, ok := myMap[key]; ok {
    process(val)
}
```

## Common Anti-patterns

### 1. Panic for Error Handling
```go
// ❌ NEVER: Panic for normal errors
func badUser(id int) User {
    user, err := fetchUser(id)
    if err != nil {
        panic(err) // Don't do this!
    }
    return user
}

// ✅ GOOD: Return errors
func goodUser(id int) (User, error) {
    user, err := fetchUser(id)
    if err != nil {
        return User{}, fmt.Errorf("fetch user %d: %w", id, err)
    }
    return user, nil
}
```

### 2. Ignoring Errors
```go
// ❌ NEVER: Ignore errors with blank identifier
_ = json.NewEncoder(w).Encode(data) // Bad!

// ✅ GOOD: Handle or explicitly document
if err := json.NewEncoder(w).Encode(data); err != nil {
    log.Printf("encode response: %v", err)
    // Decide: return error, write error response, etc.
}
```

### 3. Goroutine Leaks
```go
// ❌ BAD: Leaking goroutine
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch // Blocks forever!
    }()
    // Never send to ch
}

// ✅ GOOD: Provide cancellation
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            process(val)
        case <-ctx.Done():
            return
        }
    }()
}
```

### 4. Incorrect Mutex Usage
```go
// ❌ BAD: Embedded mutex (exposes Lock/Unlock)
type BadCache struct {
    sync.Mutex // Embedded - avoid!
    data map[string]string
}

// ✅ GOOD: Named mutex field
type GoodCache struct {
    mu   sync.RWMutex // Named field
    data map[string]string
}

// ❌ BAD: Defer in loop
for _, item := range items {
    mu.Lock()
    defer mu.Unlock() // Defers stack up!
    process(item)
}

// ✅ GOOD: Explicit unlock or extract function
for _, item := range items {
    processWithLock(item, &mu)
}

func processWithLock(item Item, mu *sync.Mutex) {
    mu.Lock()
    defer mu.Unlock()
    process(item)
}
```

### 5. Global Mutable State
```go
// ❌ BAD: Global variables
var (
    globalConfig Config
    globalClient *http.Client
)

// ✅ GOOD: Dependency injection
type Server struct {
    config Config
    client *http.Client
}

func NewServer(cfg Config, client *http.Client) *Server {
    return &Server{
        config: cfg,
        client: client,
    }
}
```

### 6. Empty Interface Overuse
```go
// ❌ BAD: Unnecessary empty interface
func process(data interface{}) {
    // Immediately type assert anyway
    str := data.(string)
    // ...
}

// ✅ GOOD: Be specific
func process(data string) {
    // Direct use
}

// ✅ OK: Genuine need for any type
type Cache struct {
    data map[string]interface{} // Actually stores various types
}
```

### 7. Get Prefix on Getters
```go
// ❌ BAD: Get prefix
func (u *User) GetName() string {
    return u.name
}

// ✅ GOOD: Direct noun
func (u *User) Name() string {
    return u.name
}
```

### 8. Generic Package Names
```go
// ❌ BAD package names
package util     // What utilities?
package common   // Common to what?
package helper   // Helps with what?
package base     // Base of what?

// ✅ GOOD: Specific names
package auth     // Authentication
package retry    // Retry logic
package cache    // Caching
```

## Interface Performance

```go
// Interface conversions have cost
type Writer interface {
    Write([]byte) (int, error)
}

// ❌ Repeated interface conversions
for i := 0; i < 1000; i++ {
    w := Writer(myWriter) // Conversion cost
    w.Write(data)
}

// ✅ Convert once
w := Writer(myWriter)
for i := 0; i < 1000; i++ {
    w.Write(data)
}
```

## Reflection Performance

```go
// ⚠️ Reflection is slow - avoid in hot paths
func slowGeneric(v interface{}) {
    rv := reflect.ValueOf(v)
    // Reflection operations are expensive
}

// ✅ Prefer type assertions or type switches
func fastSpecific(v interface{}) {
    switch x := v.(type) {
    case string:
        // Fast direct access
    case int:
        // Fast direct access
    }
}
```

## Memory Optimization

```go
// ✅ Clear slice references to allow GC
func processUsers(users []User) {
    for i, user := range users {
        process(user)
        users[i] = User{} // Clear reference
    }
}

// ✅ Reuse buffers with sync.Pool
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func useBuffer() {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    
    // Use buffer
}
```

## Benchmark Template

```go
func BenchmarkUserProcessing(b *testing.B) {
    // Setup outside of timer
    users := make([]User, 1000)
    
    b.ResetTimer() // Start timing after setup
    
    for i := 0; i < b.N; i++ {
        _ = processUsers(users)
    }
    
    b.ReportAllocs() // Report memory allocations
}

// Run: go test -bench=. -benchmem
```
{{end}}
