---
title: Go Style Guide Summary
description: Quick reference for Go idioms and style decisions
tags: [go, style, quick-reference]
trigger:
  type: glob
  globs: ["**/*.go"]
languages: [go]
variables:
  extended: "false"
---

# Go Style Guide Quick Reference

{{if not eq .extended "true" }}
**Clarity > Cleverness** • **Simplicity > Complexity** • **Explicit > Implicit**

## Naming
- **Packages**: `lowercase`, no underscores (e.g., `net/http`). Avoid generics like `util`.
- **Exported**: `PascalCase` (e.g., `WriteFile`).
- **Unexported**: `camelCase` (e.g., `writeFile`).
- **Constants**: `MixedCaps` (e.g., `MaxRetries`), not `MAX_RETRIES`.
- **Interfaces**: `-er` suffix for single method (e.g., `Reader`); descriptive for multi-method.
- **Getters**: No `Get` prefix (e.g., `user.Name()`).
- **Errors**: `ErrXxx` for sentinel errors (e.g., `var ErrNotFound = errors.New(...)`).

## Must-Do Patterns
- **Check every `error`**.
- **Use early returns** (guard clauses) to reduce nesting.
- **`defer` cleanup** immediately after resource acquisition.
- **`context.Context`** must be the first parameter.
- **Accept interfaces, return structs**.

## Never-Do Patterns
- **`panic` for expected errors**.
- **Ignore errors** with `_`.
- **Store `context.Context` in structs**.
- **Embed mutexes**; use a named field `mu sync.Mutex`.
- **Start goroutines without a way to stop them**.

## Project Layout
- `cmd/`: Application entry points.
- `internal/`: Private packages, not importable by other projects.
- `pkg/`: Public library packages, intended for external use.

## Error Handling
- Use `fmt.Errorf("...: %w", err)` to wrap errors with context.
- Use `errors.Is(err, ErrTarget)` to check for sentinel errors.
- Use `errors.As(err, &target)` to check for a specific error type.

## Testing
- Use table-driven tests with `t.Run` for subtests.
- Call `t.Helper()` in test helper functions.
- Use `t.Fatal` when a test cannot continue; otherwise, use `t.Error`.

## Concurrency
- Protect shared data with `sync.Mutex` or `sync.RWMutex`.
- Use `sync.WaitGroup` to wait for goroutines to finish.
- Use `context` for cancellation and timeouts.

## Performance
- **Measure first** with benchmarks and profiling.
- Use `strings.Builder` for building strings in loops.
- **Pre-allocate** slices and maps when the size is known.
{{else}}
## Essential Go Philosophy
**Clarity > Cleverness** • **Simplicity > Complexity** • **Explicit > Implicit**

## Naming Checklist

| Type | Convention | Example |
|------|------------|---------|
| **Package** | lowercase, no underscore | `tabwriter` ✅ `tab_writer` ❌ |
| **Exported** | PascalCase | `WriteFile` `HTTPServer` |
| **Unexported** | camelCase | `writeFile` `httpServer` |
| **Constants** | MixedCaps | `MaxRetries` not `MAX_RETRIES` |
| **Interfaces** | -er suffix or descriptive | `Reader` `Handler` `DataStore` |
| **Getters** | no Get prefix | `user.Name()` not `user.GetName()` |
| **Receivers** | 1-2 chars | `(c *Client)` not `(this *Client)` |
| **Errors** | ErrXxx | `var ErrNotFound = errors.New(...)` |

## Critical Patterns

### ✅ Always Do
```go
// Check all errors
if err != nil {
    return fmt.Errorf("context: %w", err)
}

// Early returns
if invalid {
    return ErrInvalid
}
// Happy path here

// Defer cleanup immediately
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()

// Context as first param
func Process(ctx context.Context, data string) error

// Accept interfaces, return structs
func NewServer(store Store) *Server
```

### ❌ Never Do
```go
// Never panic for errors
panic(err) // Use error returns

// Never ignore errors
_ = doWork() // Handle or document why

// Never store context in structs
type Server struct {
    ctx context.Context // Bad!
}

// Never embed mutex
type Cache struct {
    sync.Mutex // Use named field
}

// Never fire-and-forget goroutines
go doWork() // Must have lifecycle control
```

## Project Structure
```
myproject/
├── cmd/[app]/         # Entry points
├── internal/          # Private packages
├── pkg/              # Public packages
├── go.mod
└── README.md
```

## Import Groups
```go
import (
    // Standard library
    "fmt"
    "net/http"
    
    // External packages
    "github.com/pkg/errors"
    
    // Internal packages
    "myproject/internal/config"
)
```

## Testing Pattern
```go
func TestFeature(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    string
        wantErr bool
    }{
        {"empty", "", "", false},
        {"invalid", "bad", "", true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Feature(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("error = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

## Error Handling
```go
// Sentinel errors
var (
    ErrNotFound = errors.New("not found")
    ErrInvalid  = errors.New("invalid")
)

// Wrap with context
if err != nil {
    return fmt.Errorf("fetch user %d: %w", id, err)
}

// Check specific errors
if errors.Is(err, ErrNotFound) {
    // Handle not found
}
```

## Concurrency Safety
```go
// Safe goroutine with cancellation
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case task := <-tasks:
            process(task)
        }
    }
}

// Named mutex fields
type Safe struct {
    mu   sync.RWMutex
    data map[string]string
}
```

## Interface Design
```go
// Small, focused interfaces
type Reader interface {
    Read([]byte) (int, error)
}

// Interface in consumer package
package consumer
type Store interface {
    Get(string) ([]byte, error)
}

// Return concrete types
package provider
func NewRedis() *Redis {
    return &Redis{}
}

// Compile-time check
var _ Store = (*Redis)(nil)
```

## Documentation
```go
// Package http provides HTTP client and server implementations.
package http

// Process handles the data according to the configuration.
// It returns an error if the data is invalid or processing fails.
func Process(data []byte) error {
    // Implementation
}
```

## Performance Tips

| Do | Don't |
|----|-------|
| `strings.Builder` for concatenation | String += in loops |
| Preallocate slices `make([]T, 0, size)` | Grow without hint |
| `sync.Pool` for temporary objects | Create/destroy repeatedly |
| Benchmark before optimizing | Premature optimization |

## Common Code Review Comments

1. **"Return early to reduce nesting"**
2. **"Don't panic in libraries"**
3. **"Add context to errors"**
4. **"Check all errors"**
5. **"Interface should be in consumer package"**
6. **"Missing documentation on exported type"**
7. **"Use table-driven tests"**
8. **"Avoid global state"**
9. **"Context should be first parameter"**
10. **"Goroutine leak - no way to stop it"**

## Style Decision Matrix

| Situation | Google Says | Uber Says | Use This |
|-----------|------------|-----------|----------|
| Line length | No limit | 99 chars | Match codebase |
| Global vars | Standard | `_` prefix | Be consistent |
| Atomics | sync/atomic | go.uber.org/atomic | Uber's for safety |
| Assertion libs | No | Sometimes | Prefer standard |

## Quick Audit Checklist

- [ ] All exported names documented?
- [ ] Errors handled or explicitly ignored?
- [ ] Context passed, not stored?
- [ ] Goroutines can be stopped?
- [ ] Interfaces small and focused?
- [ ] No panic for normal errors?
- [ ] Tests table-driven?
- [ ] Package names meaningful?
- [ ] Early returns used?
- [ ] Resources deferred after acquisition?
{{end}}
