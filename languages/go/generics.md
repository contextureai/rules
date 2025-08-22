---
title: Go Generics and Type Parameters
description: Use generics judiciously where they provide clear value without unnecessary complexity
tags: [go, generics]
trigger:
  type: model
  description: When implementing generic types, functions with type parameters, or type constraints
languages: [go]
variables:
  extended: false
---
{{if not .extended}}
## When to Use Generics
Use generics when you have functions or data structures that work with multiple different types in the exact same way.
- **Good use cases**: General-purpose data structures (e.g., slices, maps, channels), and functions that operate on them (e.g., `map`, `filter`, `reduce`).
- **When to avoid**: Don't use generics for the sake of abstraction. If you only ever use a function with one type, it doesn't need to be generic. Interfaces often provide a better solution for abstracting over behavior.

## Generic Functions
A generic function has type parameters, listed in square brackets before the regular function parameters.

```go
// T is a type parameter, constrained by constraints.Ordered.
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

## Generic Types
A generic type has type parameters, listed in square brackets after the type name.

```go
// A Container of any type T.
type Container[T any] struct {
    items []T
}

// Methods can use the type's parameters.
func (c *Container[T]) Add(item T) {
    c.items = append(c.items, item)
}
```

## Type Constraints
Constraints are interfaces that define the set of types a type parameter can accept.
- The `any` constraint allows any type.
- The `comparable` constraint allows types that can be compared with `==` and `!=`.
- The `constraints` package provides common constraints like `constraints.Ordered` (for `<`, `>`, etc.) and `constraints.Integer`.
- You can define your own constraints as interfaces. An interface used as a constraint can include arbitrary types in a union.

```go
// This interface can be used as a constraint for numeric types.
type Number interface {
    constraints.Integer | constraints.Float
}
```

## Best Practices
- **Start with concrete types**. Only introduce generics when you have a clear case of duplicated logic for different types.
- **Keep constraints minimal**. A function should only require the operations it actually uses.
- **Type parameters cannot be used on methods**. If a method needs to be generic, make it a standalone function instead.
- **Use type inference** where possible, but provide explicit type arguments if it improves clarity.
{{else}}
## Core Principles

1. **Start concrete, generalize when needed**
2. **Avoid premature abstraction**
3. **Don't create DSLs with generics**
4. **Prefer existing language features when sufficient**

## When to Use Generics

✅ **Good Use Cases:**
- General-purpose data structures (lists, trees, graphs)
- Type-safe containers and collections
- Channel utilities and synchronization primitives
- Functions operating on slices/maps generically

❌ **Avoid Generics For:**
- Single type instantiation in practice
- Creating domain-specific languages
- Error handling frameworks
- Assertion libraries for testing

## Basic Generic Patterns

```go
// Generic function
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// Generic type
type Container[T any] struct {
    items []T
}

// Methods on generic types
func (c *Container[T]) Add(item T) {
    c.items = append(c.items, item)
}

func (c *Container[T]) Get(index int) T {
    return c.items[index]
}
```

## Type Constraints

```go
// Built-in constraints
import "constraints"

func Sum[T constraints.Integer | constraints.Float](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}

// Custom constraints
type Number interface {
    constraints.Integer | constraints.Float | constraints.Complex
}

// Type sets
type Stringable interface {
    ~string // Types with underlying type string
}

// Method constraints
type Hasher[T any] interface {
    Hash() T
}
```

## Generic Best Practices

### Start Without Generics
```go
// ❌ Premature generalization
type Stack[T any] struct {
    items []T
}

// ✅ Start concrete if only one type needed
type IntStack struct {
    items []int
}

// Generalize later if multiple types needed
type Stack[T any] struct {
    items []T
}
```

### Avoid Complex Type Inference
```go
// ❌ Hard to understand
func Process[T any, R comparable, S ~[]T](fn func(T) R, items S) map[R]S

// ✅ Clearer with explicit types
func GroupBy[T any, K comparable](items []T, keyFunc func(T) K) map[K][]T
```

### Type Parameters in Methods
```go
// ❌ Type parameters on methods not allowed
func (c *Container[T]) Transform[R any](fn func(T) R) []R // Compile error

// ✅ Use type parameters on functions
func Transform[T, R any](c *Container[T], fn func(T) R) []R {
    result := make([]R, 0, len(c.items))
    for _, item := range c.items {
        result = append(result, fn(item))
    }
    return result
}
```

## Common Generic Patterns

### Zero Value Handling
```go
func GetOrDefault[T any](m map[string]T, key string, defaultVal T) T {
    if val, ok := m[key]; ok {
        return val
    }
    return defaultVal
}

// Using zero value
func GetOrZero[T any](m map[string]T, key string) T {
    return m[key] // Returns zero value if not found
}
```

### Slice Operations
```go
func Filter[T any](slice []T, predicate func(T) bool) []T {
    var result []T
    for _, item := range slice {
        if predicate(item) {
            result = append(result, item)
        }
    }
    return result
}

func Map[T, R any](slice []T, transform func(T) R) []R {
    result := make([]R, len(slice))
    for i, item := range slice {
        result[i] = transform(item)
    }
    return result
}
```

### Channel Helpers
```go
func Merge[T any](done <-chan struct{}, channels ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    
    wg.Add(len(channels))
    for _, ch := range channels {
        go func(c <-chan T) {
            defer wg.Done()
            for {
                select {
                case val, ok := <-c:
                    if !ok {
                        return
                    }
                    select {
                    case out <- val:
                    case <-done:
                        return
                    }
                case <-done:
                    return
                }
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

## Type Inference

```go
// Explicit type arguments
result := Min[int](10, 20)

// Type inference (preferred when unambiguous)
result := Min(10, 20) // Inferred as int

// Sometimes explicit is clearer
var items = Map[string, int](names, func(s string) int {
    return len(s)
})
```

## Avoiding Generic Anti-patterns

### Don't Create Error DSLs
```go
// ❌ Bad: Generic error handling framework
type Result[T any] struct {
    value T
    err   error
}

func (r Result[T]) Map[R any](fn func(T) R) Result[R] {
    // Complex monadic error handling
}

// ✅ Good: Standard Go error handling
value, err := doSomething()
if err != nil {
    return fmt.Errorf("failed: %w", err)
}
```

### Don't Over-Abstract
```go
// ❌ Too abstract
type Functor[F[_] any, A, B any] interface {
    Map(func(A) B) F[B]
}

// ✅ Concrete and clear
func MapStrings(strs []string, fn func(string) string) []string {
    result := make([]string, len(strs))
    for i, s := range strs {
        result[i] = fn(s)
    }
    return result
}
```

## Migration Strategy

When adding generics to existing code:

1. **Identify duplication**: Look for copy-pasted code with type changes
2. **Extract common behavior**: Find the shared algorithm
3. **Add type parameters**: Generalize only what varies
4. **Maintain compatibility**: Keep existing APIs when possible

```go
// Before: Duplicated functions
func SortInts(x []int) { sort.Ints(x) }
func SortStrings(x []string) { sort.Strings(x) }

// After: Generic function
func Sort[T constraints.Ordered](x []T) {
    sort.Slice(x, func(i, j int) bool {
        return x[i] < x[j]
    })
}
```

## Documentation

Always include examples with generic code:

```go
// ExampleMin demonstrates the Min function with different types.
func ExampleMin() {
    fmt.Println(Min(10, 20))           // int
    fmt.Println(Min(3.14, 2.71))       // float64
    fmt.Println(Min("apple", "banana")) // string
    // Output:
    // 10
    // 2.71
    // apple
}
```
{{end}}
