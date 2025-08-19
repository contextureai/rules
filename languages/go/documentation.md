---
title: Go Documentation Standards
description: Write clear, idiomatic Go documentation and comments
tags: [go, documentation, best-practices]
trigger:
  type: glob
  globs: ["**/*.go"]
languages: [go]
variables:
    extended: false
---

# Go Documentation Standards

{{if not .extended}}
## General Rules
- Document every exported name (function, type, constant, variable).
- Comments for exported names must start with the name of the item being described (e.g., `// Server processes...`).
- Write complete sentences with proper punctuation.
- Use code fences for example code.

## Package Comments
Every package needs a package comment. It should be a concise summary of the package's functionality. For more extensive documentation, create a `doc.go` file.

```go
// Package server provides utilities for processing data.
// It supports concurrent workers and configurable retry logic.
package server
```

## Types and Functions
- Document what the type or function does.
- For functions, explain the parameters and return values, especially any error conditions.

```go
// Config defines configuration for a Server.
type Config struct {
    // Workers specifies the number of concurrent workers.
    // Defaults to runtime.NumCPU() if zero.
    Workers int
}

// NewServer creates a Server with the given configuration.
// It returns an error if the configuration is invalid.
func NewServer(cfg Config) (*Server, error) {
    // ...
}
```

## Constants and Variables
Document blocks of constants and variables, and individual items if their meaning is not obvious from the name.

```go
// State represents the server's operational state.
type State int

const (
    // StateIdle indicates the server is waiting for tasks.
    StateIdle State = iota
    // StateRunning indicates the server is actively processing.
    StateRunning
)
```

## Implementation Comments
Use comments inside functions to explain *why* the code is written a certain way, not *what* it does. Good comments clarify complex logic, business reasons, or trade-offs.

```go
// Pre-allocate slice to avoid reallocations.
// Average input size is ~1024 items.
results := make([]Result, 0, 1024)
```

## Deprecation Notices
To mark an exported name as deprecated, add a `// Deprecated:` directive as the first sentence of its doc comment. Explain what to use instead.

```go
// Deprecated: Use NewServer instead. This function will be removed in v2.0.0.
func CreateServer() *Server { /* ... */ }
```
{{else}}
## Documentation Rules

1. **Start with the name** being documented
2. **Complete sentences** with proper grammar
3. **Document all exported** names
4. **Package comment** required

## Package Documentation

```go
// Package server provides utilities for processing data
// with concurrent workers and configurable retry logic.
//
// The main entry point is the Server type:
//
//	handler := server.NewServer(config)
//	result, err := handler.Process(input)
//
// For concurrent processing, use the Pool:
//
//	pool := server.NewPool(workers)
//	pool.Submit(tasks)
package server
```

For extensive docs, use `doc.go`:
```go
// doc.go

/*
Package server implements a high-performance processing system.

# Overview

This package provides three main components:

  - Server: Core processing logic
  - Pool: Concurrent task execution
  - Retry: Automatic retry with backoff

# Configuration

Configure the processor using the Config struct:

	cfg := server.Config{
	    Workers:    10,
	    MaxRetries: 3,
	    Timeout:    30 * time.Second,
	}

# Examples

See the examples directory for complete usage examples.

# Performance

The processor uses worker pools to achieve high throughput.
Benchmarks show 10,000 ops/sec on standard hardware.
*/
package server
```

## Type Documentation

```go
// Server processes incoming requests with automatic
// retry logic and concurrent execution support.
type Server struct {
    config   Config
    pool     *WorkerPool
    metrics  *Metrics
}

// Config defines configuration options for Server.
type Config struct {
    // Workers specifies the number of concurrent workers.
    // Default is runtime.NumCPU().
    Workers int
    
    // MaxRetries specifies maximum retry attempts.
    // Default is 3. Use 0 to disable retries.
    MaxRetries int
    
    // Timeout specifies the maximum duration for processing.
    // Default is 30 seconds. Use 0 for no timeout.
    Timeout time.Duration
}

// State represents the current state of the handler.
type State int

// Possible states for the handler.
const (
    // StateIdle indicates the handler is ready.
    StateIdle State = iota
    
    // StateProcessing indicates active processing.
    StateProcessing
    
    // StateStopped indicates the handler has stopped.
    StateStopped
)
```

## Function Documentation

```go
// NewServer creates a new handler with the given configuration.
// It returns an error if the configuration is invalid.
func NewServer(cfg Config) (*Server, error) {
    if err := cfg.Validate(); err != nil {
        return nil, fmt.Errorf("invalid config: %w", err)
    }
    return &Server{config: cfg}, nil
}

// Process executes the main processing logic on the input.
// It returns the processed result or an error if processing fails.
//
// The processing steps are:
//  1. Validate input
//  2. Transform data  
//  3. Apply business logic
//  4. Return result
//
// Example:
//
//	result, err := handler.Process(input)
//	if err != nil {
//	    log.Printf("processing failed: %v", err)
//	}
func (h *Server) Process(input Input) (Result, error) {
    // Implementation
}

// ProcessWithContext is like Process but accepts a context
// for cancellation and timeout control.
func (h *Server) ProcessWithContext(
    ctx context.Context,
    input Input,
) (Result, error) {
    // Implementation
}
```

## Method Documentation

```go
// Start begins processing with the configured number of workers.
// It returns immediately and processes tasks asynchronously.
// Use Stop to gracefully shutdown.
func (h *Server) Start() error {
    // Implementation
}

// Stop gracefully shuts down the handler, waiting for active
// tasks to complete. It returns an error if shutdown fails.
func (h *Server) Stop() error {
    // Implementation
}

// unexportedMethod doesn't require a comment but can have one
// for complex logic that helps future maintainers.
func (h *Server) unexportedMethod() {
    // Implementation
}
```

## Interface Documentation

```go
// Serverer defines the interface for types that can
// process data. Implementations must be safe for concurrent use.
type Serverer interface {
    // Process processes the input and returns a result.
    // Implementations should return an error for invalid input.
    Process(Input) (Result, error)
}

// Store defines methods for data persistence.
// All methods must be safe for concurrent use.
type Store interface {
    // Get retrieves the value for key.
    // It returns ErrNotFound if the key doesn't exist.
    Get(key string) (interface{}, error)
    
    // Set stores the value for key.
    // It overwrites any existing value.
    Set(key string, value interface{}) error
    
    // Delete removes the key and its value.
    // It returns nil even if the key doesn't exist.
    Delete(key string) error
}
```

## Deprecated Items

```go
// Deprecated: Use NewServer instead.
// This function will be removed in v2.0.
func CreateServer() *Server {
    return NewServer(DefaultConfig())
}

// ServerV1 is the legacy handler implementation.
//
// Deprecated: Use Server instead. V1 will be removed
// in the next major version. Migration guide:
// https://example.com/migration
type ServerV1 struct {
    // fields
}
```

## Comment Formatting

```go
// ✅ Good comments
// Server implements the processing logic.
// This is a complete sentence with proper capitalization.

// ❌ Bad comments  
// handler impl  // Too brief, not a sentence
// THE HANDLER TYPE  // Don't use all caps
// this processes stuff  // No capitalization
// Processes the data.  // Missing subject

// ✅ Multi-line comments
// Process handles the core processing logic.
// It validates input, applies transformations, and returns
// the result. Processing may take up to 30 seconds for
// large inputs.

// ✅ Implementation comments (explain why, not what)
func (h *Server) process() {
    // Use a buffered channel to prevent goroutine blocking
    // when the consumer is slow. Size 100 based on benchmarks.
    ch := make(chan Task, 100)
    
    // Pre-allocate slice to avoid reallocations during growth.
    // Average case has ~1000 items based on production metrics.
    results := make([]Result, 0, 1000)
}
```

## Special Comments

```go
// TODO(username): Implement retry logic with exponential backoff.

// BUG(username): This panics on nil input. See issue #123.

// NOTE: This algorithm is O(n²) but inputs are always small (<100).

// FIXME: Race condition when workers > 10. Needs mutex.
```

## Example Functions

```go
// Example demonstrates basic usage of Server.
func ExampleServer() {
    handler := NewServer(DefaultConfig())
    result, err := handler.Process(Input{Data: "test"})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(result)
    // Output: processed: test
}

// Example_concurrent shows concurrent processing.
func ExampleServer_concurrent() {
    handler := NewServer(Config{Workers: 5})
    
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            handler.Process(Input{ID: id})
        }(i)
    }
    wg.Wait()
    // Output:
    // Processing completed
}

// ExampleProcess demonstrates the Process method.
func ExampleServer_Process() {
    h := &Server{}
    result, _ := h.Process(Input{})
    fmt.Println(result)
    // Output: default result
}
```
{{end}}
