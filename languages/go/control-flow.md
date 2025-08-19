---
title: Go Control Flow Best Practices
description: Use early returns, minimal nesting, and clear control flow patterns
tags: [go, best-practices]
trigger:
  type: glob
  globs: ["**/*.go", "!**/*_test.go"]
languages: [go]
variables:
    extended: false
---

# Go Control Flow Best Practices

{{if not .extended}}
## Early Returns (Guard Clauses)
Handle error conditions and preconditions at the beginning of a function to reduce nesting. This keeps the "happy path" at the lowest level of indentation, improving readability.

```go
func process(input string) error {
    if input == "" {
        return errors.New("empty input")
    }
    // ... happy path ...
    return nil
}
```

## `defer` for Cleanup
Use `defer` immediately after acquiring a resource to guarantee its release. `defer` is ideal for closing files, unlocking mutexes, and other cleanup tasks. Deferred calls are executed in Last-In, First-Out (LIFO) order.

```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close() // Guaranteed to run
```

## Loops
- Prefer `for ... range` for iterating over slices, arrays, maps, and channels.
- To modify elements in a slice, iterate over the index: `for i := range items`.
- Use `continue` to skip to the next iteration and `break` to exit a loop. Use labels (`break outer`) to exit nested loops.

## `switch` Statements
- `switch` provides a clean alternative to long `if-else` chains.
- Go's `switch` cases do not fall through by default; use the `fallthrough` keyword sparingly if you need that behavior.
- A `switch` without an expression is a cleaner way to write an `if-else-if` chain.
- Use a type switch (`switch v := i.(type)`) to handle different concrete types of an interface variable.

## Short Statement in `if`
Use the "if short statement" to scope variables to the conditional block. This is idiomatic for error handling and map lookups.

```go
// Scopes 'err' to the if block
if err := doWork(); err != nil {
    return err
}

// Scopes 'val' and 'ok'
if val, ok := myMap[key]; ok {
    // ... use val ...
}
```

## Reduce Nesting
Deeply nested code is hard to read. Flatten control flow by:
- Using early returns.
- Using `continue` in loops.
- Extracting nested logic into separate functions.
{{else}}
## Early Returns (Guard Clauses)

**Principle**: Handle errors and edge cases first, keep happy path at minimal indentation.

```go
// ✅ GOOD: Early returns, minimal nesting
func processData(input string) (Result, error) {
    // Validate inputs first
    if input == "" {
        return Result{}, errors.New("empty input")
    }
    
    if len(input) > MaxSize {
        return Result{}, errors.New("input too large")
    }
    
    // Check preconditions
    if !isReady() {
        return Result{}, errors.New("system not ready")
    }
    
    // Happy path with minimal indentation
    data, err := parse(input)
    if err != nil {
        return Result{}, fmt.Errorf("parse: %w", err)
    }
    
    result := transform(data)
    return result, nil
}

// ❌ BAD: Nested happy path
func processDataBad(input string) (Result, error) {
    if input != "" {
        if len(input) <= MaxSize {
            if isReady() {
                data, err := parse(input)
                if err == nil {
                    result := transform(data)
                    return result, nil
                }
                return Result{}, err
            }
            return Result{}, errors.New("not ready")
        }
        return Result{}, errors.New("too large")
    }
    return Result{}, errors.New("empty")
}
```

## Resource Management with Defer

```go
// ✅ Defer immediately after resource acquisition
func processFile(path string) error {
    file, err := openFile(path)
    if err != nil {
        return fmt.Errorf("open file: %w", err)
    }
    defer file.Close() // Immediately after successful open
    
    // Multiple defers execute in LIFO order
    lock.Lock()
    defer lock.Unlock() // Will run before Close()
    
    return file.Process()
}

// ✅ Defer with error checking
func saveFile(path string, data []byte) (err error) {
    file, err := createFile(path)
    if err != nil {
        return fmt.Errorf("create: %w", err)
    }
    
    defer func() {
        // Capture close error if no other error
        if cerr := file.Close(); cerr != nil && err == nil {
            err = fmt.Errorf("close: %w", cerr)
        }
    }()
    
    if _, err = file.Write(data); err != nil {
        return fmt.Errorf("write: %w", err)
    }
    
    return nil
}
```

## Loop Patterns

```go
// ✅ Range loops
for i, item := range items {
    process(i, item)
}

// ✅ Index only
for i := range items {
    items[i] = transform(items[i])
}

// ✅ Value only (discard index)
for _, item := range items {
    process(item)
}

// ✅ Infinite loop
for {
    if shouldStop() {
        break
    }
    doWork()
}

// ✅ Traditional loop when needed
for i := 0; i < len(items); i++ {
    if i > 0 {
        // Need access to previous item
        compare(items[i-1], items[i])
    }
}

// ❌ Avoid manual iteration of slices
for i := 0; i < len(items); i++ {
    process(items[i]) // Use range instead
}
```

## Switch Statements

```go
// ✅ Switch with multiple cases
switch value {
case "a", "A":
    return processA()
case "b", "B":
    return processB()
default:
    return processDefault()
}

// ✅ Switch without expression
switch {
case x < 0:
    return "negative"
case x == 0:
    return "zero"
case x > 0:
    return "positive"
}

// ✅ Type switch
switch v := value.(type) {
case string:
    return len(v)
case int:
    return v
case []byte:
    return len(v)
case nil:
    return 0
default:
    return -1
}

// ✅ Fallthrough (use sparingly)
switch n {
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("one or two")
default:
    fmt.Println("other")
}
```

## Conditional Expressions

```go
// Go doesn't have ternary operator
// ❌ Cannot do: result = condition ? valueA : valueB

// ✅ Use if-else for simple conditions
var result string
if condition {
    result = "valueA"
} else {
    result = "valueB"
}

// ✅ Or use a function for reusable logic
func ternary(condition bool, ifTrue, ifFalse string) string {
    if condition {
        return ifTrue
    }
    return ifFalse
}
result := ternary(x > 0, "positive", "non-positive")

// ✅ Initialize and check in if statement
if err := doWork(); err != nil {
    return fmt.Errorf("work failed: %w", err)
}

// ✅ Check existence and use
if val, ok := myMap[key]; ok {
    process(val)
}
```

## Break and Continue

```go
// ✅ Break from nested loops with labels
outer:
for i := range items {
    for j := range items[i] {
        if items[i][j] == target {
            break outer // Breaks both loops
        }
    }
}

// ✅ Continue to next iteration
for _, item := range items {
    if shouldSkip(item) {
        continue
    }
    
    // Process only non-skipped items
    process(item)
}

// ✅ Break from select in loop
for {
    select {
    case msg := <-ch:
        if msg == "stop" {
            break // Only breaks select, not loop!
        }
        process(msg)
    case <-done:
        return // Use return to exit loop
    }
}
```

## Select for Channel Operations

```go
// ✅ Select with timeout
select {
case result := <-ch:
    return result, nil
case <-time.After(5 * time.Second):
    return nil, errors.New("timeout")
}

// ✅ Non-blocking receive
select {
case msg := <-ch:
    process(msg)
default:
    // Channel empty, continue
}

// ✅ Priority select with default
for {
    select {
    case <-highPriority:
        handleHighPriority()
    default:
        select {
        case <-highPriority:
            handleHighPriority()
        case <-lowPriority:
            handleLowPriority()
        case <-time.After(1 * time.Second):
            // Neither has data
        }
    }
}
```

## Complex Conditionals

```go
// ✅ Extract complex conditions to functions
func isValidInput(data Data) bool {
    return data != nil &&
           len(data.Items) > 0 &&
           data.Status == StatusReady &&
           time.Since(data.Timestamp) < MaxAge
}

// Use the extracted function
if !isValidInput(data) {
    return errors.New("invalid input")
}

// ✅ Multi-line conditions
if err != nil ||
   response.StatusCode != http.StatusOK ||
   response.Body == nil {
    return errors.New("bad response")
}
```

## Avoiding Deep Nesting

```go
// ❌ BAD: Deep nesting
func processNested(items []Item) error {
    for _, item := range items {
        if item.IsValid() {
            data, err := item.GetData()
            if err == nil {
                for _, d := range data {
                    if d.NeedsProcessing() {
                        result := process(d)
                        if result.Success {
                            save(result)
                        }
                    }
                }
            }
        }
    }
    return nil
}

// ✅ GOOD: Flatten with early continues
func processFlat(items []Item) error {
    for _, item := range items {
        if !item.IsValid() {
            continue
        }
        
        data, err := item.GetData()
        if err != nil {
            log.Printf("get data: %v", err)
            continue
        }
        
        processItemData(data)
    }
    return nil
}

func processItemData(data []Data) {
    for _, d := range data {
        if !d.NeedsProcessing() {
            continue
        }
        
        result := process(d)
        if !result.Success {
            continue
        }
        
        save(result)
    }
}
```
{{end}}
