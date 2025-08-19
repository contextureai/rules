---
title: Go Testing Best Practices
description: Write idiomatic table-driven tests with proper helpers and assertions
tags: [go, testing, test, best-practices]
trigger:
  type: glob
  globs: ["**/*_test.go"]
languages: [go]
variables:
  extended: "false"
---

# Go Testing Best Practices

{{if not .extended}}
## Table-Driven Tests
Use table-driven tests for comprehensive coverage. Define a slice of test case structs and iterate through them with `t.Run`.

```go
func TestCalculate(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {"happy path", "1+1", 2, false},
        {"invalid input", "abc", 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Calculate(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("Calculate() error = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("Calculate() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

## Test Naming
- Test functions: `Test<FunctionName>`.
- Benchmarks: `Benchmark<FunctionName>`.
- Examples: `Example<FunctionName>`.

## Test Helpers
- Call `t.Helper()` at the start of any helper function to ensure error line numbers point to the test case, not the helper.
- Use helpers for setup/teardown logic.

```go
func setup(t *testing.T) (teardown func()) {
    t.Helper()
    // ... setup ...
    return func() { /* ... teardown ... */ }
}
```

## `t.Error` vs `t.Fatal`
- Use `t.Error` or `t.Errorf` to report a non-fatal failure and continue with the current test.
- Use `t.Fatal` or `t.Fatalf` only when a failure is critical and the test cannot proceed (e.g., setup fails).

## Parallel Tests
To run subtests in parallel, call `t.Parallel()` within the `t.Run` block. Remember to capture the range variable.

```go
for _, tt := range tests {
    tt := tt // Capture range variable
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // ... test logic ...
    })
}
```

## Test Data
- Store test data files in a `testdata` directory. This directory is ignored by Go build tools.
- Use `go-cmp` for clear diffs in complex data structure comparisons.

## Benchmarks
- Perform setup outside the `for` loop.
- Call `b.ResetTimer()` after setup to exclude it from measurements.
{{else}}
## Table-Driven Tests

The standard Go pattern for comprehensive testing:

```go
func TestCalculate(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {
            name:  "empty input",
            input: "",
            want:  0,
        },
        {
            name:    "invalid input",
            input:   "invalid",
            wantErr: true,
        },
        {
            name:  "valid input",  
            input: "valid",
            want:  100,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Calculate(tt.input)
            
            // Check error expectation
            if (err != nil) != tt.wantErr {
                t.Errorf("Calculate() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            
            // Check result
            if got != tt.want {
                t.Errorf("Calculate(%v) = %v, want %v", tt.input, got, tt.want)
            }
        })
    }
}
```

## Test Naming Conventions

```go
// Test functions: Test<FunctionName>[_<Scenario>]
func TestNewCalculate(t *testing.T) {}
func TestCalculate_EmptyInput(t *testing.T) {}
func TestCalculate_Integration(t *testing.T) {}

// Benchmarks: Benchmark<FunctionName>[_<Scenario>]
func BenchmarkCalculate(b *testing.B) {}
func BenchmarkCalculate_Parallel(b *testing.B) {}

// Examples: Example<FunctionName>[_<Scenario>]
func ExampleCalculate() {
    result := Calculate("input")
    fmt.Println(result)
    // Output: expected output
}
```

## Test Helpers

```go
// Mark helpers with t.Helper() for accurate failure lines
func mustCalculate(t *testing.T, input string) int {
    t.Helper() // Points to caller on failure
    
    result, err := Calculate(input)
    if err != nil {
        t.Fatalf("{{.functionName}}(%v) failed: %v", input, err)
    }
    return result
}

// Setup/teardown helpers
func setupTest(t *testing.T) (*TestEnv, func()) {
    t.Helper()
    
    env := &TestEnv{
        // Initialize
    }
    
    cleanup := func() {
        // Cleanup resources
    }
    
    return env, cleanup
}

// Usage
func TestWithSetup(t *testing.T) {
    env, cleanup := setupTest(t)
    defer cleanup()
    
    // Test with env
}
```

## Error vs Fatal

```go
// ✅ Use Error to continue testing
func TestMultipleChecks(t *testing.T) {
    result := compute()
    
    if result.Field1 != expected1 {
        t.Errorf("Field1 = %v, want %v", result.Field1, expected1)
    }
    
    if result.Field2 != expected2 {
        t.Errorf("Field2 = %v, want %v", result.Field2, expected2)
    }
    // Both errors will be reported
}

// ✅ Use Fatal only when continuation is impossible
func TestWithFile(t *testing.T) {
    f, err := os.Open("testdata/input.txt")
    if err != nil {
        t.Fatalf("cannot open test file: %v", err)
    }
    defer f.Close()
    
    // Can't continue without file
}
```

## Parallel Tests

```go
func TestParallelCalculate(t *testing.T) {
    tests := []struct{
        name  string
        input string
    }{
        {"case1", "a"},
        {"case2", "b"},
    }
    
    for _, tt := range tests {
        tt := tt // Capture range variable
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Run subtests in parallel
            
            // Test code here
            result := Calculate(tt.input)
            // Assertions
        })
    }
}
```

## Test Data Organization

```go
// Use testdata/ directory for test files
func TestWithTestdata(t *testing.T) {
    golden := filepath.Join("testdata", "golden.json")
    want, err := os.ReadFile(golden)
    if err != nil {
        t.Fatalf("read golden file: %v", err)
    }
    
    got := generateOutput()
    
    if diff := cmp.Diff(string(want), got); diff != "" {
        t.Errorf("output mismatch (-want +got):\n%s", diff)
    }
}
```

## Black-box Testing

```go
// In separate package to test public API only
package myapp_test

import (
    "testing"
    "myapp"
)

func TestPublicAPI(t *testing.T) {
    // Can only access exported functions
    result := myapp.Calculate("input")
    
    if result != expected {
        t.Errorf("got %v, want %v", result, expected)
    }
}
```

## Benchmark Best Practices

```go
func BenchmarkCalculate(b *testing.B) {
    // Setup outside timer
    input := prepareInput()
    
    b.ResetTimer() // Reset after setup
    
    for i := 0; i < b.N; i++ {
        Calculate(input)
    }
}

// Sub-benchmarks
func BenchmarkCalculateSizes(b *testing.B) {
    sizes := []int{10, 100, 1000}
    
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            input := makeInput(size)
            b.ResetTimer()
            
            for i := 0; i < b.N; i++ {
                Calculate(input)
            }
        })
    }
}
```
{{end}}
