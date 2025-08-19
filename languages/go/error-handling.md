---
title: Go Error Handling Patterns
description: Implement robust error handling with wrapping, sentinel errors, and proper propagation
tags: [go, errors, best-practices]
trigger:
  type: glob
  globs: ["**/*.go", "!**/*_test.go"]
languages: [go]
variables:
    extended: false
---

# Go Error Handling Patterns

{{if not .extended}}
## Core Principles
1.  **Check all errors**: No `_ = someFunc()`.
2.  **Add context**: Wrap errors with `fmt.Errorf("...: %w", err)`.
3.  **Handle errors once**: The caller either logs or propagates the error.
4.  **Fail fast**: Check errors immediately after the call.
5.  **Use `panic` only for unrecoverable errors**: such as programming mistakes.

## Creating Errors
- For simple, static errors, use `errors.New("message")`.
- For dynamic errors, use `fmt.Errorf("... %v", val)`.
- For errors that need to be checked, use a sentinel error: `var ErrNotFound = errors.New("not found")`.
- For structured error data, use a custom error type.

## Propagating Errors
- Add context using the `%w` verb to wrap the original error. This preserves the error chain for inspection with `errors.Is` and `errors.As`.
- Avoid generic messages like "failed to". Instead, describe the attempted operation.

```go
// Good: add context
if err != nil {
    return fmt.Errorf("database query for ID %s: %w", id, err)
}

// Bad: loses original error type
if err != nil {
    return fmt.Errorf("database query error: %v", err) // Should be %w
}
```

## Checking Errors
- Use `errors.Is(err, ErrTarget)` for sentinel error checks.
- Use `errors.As(err, &targetErr)` to check for a specific error type and access its fields.

```go
var targetErr *MyErrorType
if errors.As(err, &targetErr) {
    // Can now access fields of targetErr
}
```

## Error Messages
- Write error messages in lowercase without punctuation.
- Ensure they are specific and provide meaningful context.

## Handling Panics
- `panic` should only be used for unrecoverable situations, like programmer errors (e.g., index out of bounds).
- If a panic is necessary, provide a clear, informative message.
- Use `recover` only at the top level of a program or goroutine to log the error and gracefully shut down.
{{else}}
## Core Principles
1. **Never panic** for expected errors
2. **Check all errors** - no `_ = someFunc()`
3. **Add context** when propagating
4. **Handle once** - log OR return, not both
5. **Fail fast** - check errors immediately

## Error Creation Strategy

| Need matching? | Message type | Use this approach |
|---------------|--------------|-------------------|
| No | Static | `errors.New("fixed message")` |
| No | Dynamic | `fmt.Errorf("context: %v", val)` |
| Yes | Static | `var ErrXxx = errors.New("...")` |
| Yes | Dynamic | Custom error type |

## Sentinel Errors
```go
package myapp

import "errors"

// Define at package level for comparison
var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrInvalidInput  = errors.New("invalid input")
    ErrAlreadyExists = errors.New("already exists")
)

// Usage with errors.Is
if errors.Is(err, ErrNotFound) {
    // Handle not found case
    return nil, status.NotFound
}
```

## Error Wrapping
```go
// ✅ Good: Add context with %w
func processDatabaseQuery(id string) error {
    data, err := databaseQuery(id)
    if err != nil {
        return fmt.Errorf("database query for ID %s: %w", id, err)
    }
    return nil
}

// ❌ Bad: "failed to" is redundant
if err != nil {
    return fmt.Errorf("failed to database query: %w", err)
}

// ❌ Bad: Loses error chain
if err != nil {
    return fmt.Errorf("database query error: %v", err) // Use %w not %v
}
```

## Custom Error Types
```go
// For structured errors with fields
type MyAppError struct {
    Op   string // Operation attempted
    Kind string // Category of error
    Err  error  // Underlying error
}

func (e *MyAppError) Error() string {
    return fmt.Sprintf("%s: %s: %v", e.Op, e.Kind, e.Err)
}

func (e *MyAppError) Unwrap() error {
    return e.Err
}

// Usage
return &MyAppError{
    Op:   "GetUser",
    Kind: "validation",
    Err:  fmt.Errorf("invalid ID format"),
}
```

## Error Checking Patterns
```go
// ✅ Early return
func doWork() error {
    if err := step1(); err != nil {
        return fmt.Errorf("step1: %w", err)
    }
    
    if err := step2(); err != nil {
        return fmt.Errorf("step2: %w", err)
    }
    
    return step3()
}

// ✅ Check immediately after call
resp, err := http.Get(url)
if err != nil {
    return nil, fmt.Errorf("fetch %s: %w", url, err)
}
defer resp.Body.Close()

// ❌ Never ignore errors
_ = json.NewEncoder(w).Encode(data) // Bad!

// ✅ If truly ok to ignore, document why
// Safe to ignore: we're already returning an error
_ = resp.Body.Close() // Best effort cleanup
```

## Error Messages
```go
// ✅ Good error messages:
errors.New("cannot parse config")           // lowercase
errors.New("invalid user ID")               // no punctuation
fmt.Errorf("decode JSON at line %d", line)  // add context

// ❌ Bad error messages:
errors.New("Failed to parse config.")       // Capitalized with period
errors.New("ERROR: bad input")              // Redundant prefix
errors.New("something went wrong")          // Too vague
```

## Handle Errors Once
```go
// ❌ Bad: Logging AND returning
func fetchData() error {
    err := db.Query()
    if err != nil {
        log.Printf("query failed: %v", err) // Don't log here
        return err                          // Caller will handle
    }
    return nil
}

// ✅ Good: Let caller decide
func fetchData() error {
    if err := db.Query(); err != nil {
        return fmt.Errorf("fetch data: %w", err)
    }
    return nil
}

// Caller logs OR returns
if err := fetchData(); err != nil {
    log.Printf("operation failed: %v", err)
    // Don't return the error after logging
}
```

## Assertion Errors
```go
// Safe type assertion with error
func processValue(v interface{}) error {
    str, ok := v.(string)
    if !ok {
        return fmt.Errorf("expected string, got %T", v)
    }
    // Use str
    return nil
}
```

## Testing Errors
```go
func TestErrorCases(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr error
    }{
        {
            name:    "not found",
            input:   "missing",
            wantErr: ErrNotFound,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := process(tt.input)
            if !errors.Is(err, tt.wantErr) {
                t.Errorf("got error %v, want %v", err, tt.wantErr)
            }
        })
    }
}
```
{{end}}
