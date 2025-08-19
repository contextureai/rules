---
title: Go Interface Design Principles
description: Design small, focused interfaces in the right package with proper compliance checks
tags: [go, interfaces, architecture]
trigger:
  type: glob
  globs: ["**/*.go", "!**/*_test.go"]
languages: [go]
variables:
    extended: false
---

# Go Interface Design Principles

{{if not .extended}}
## Core Principles
1.  **Accept interfaces, return structs**: Functions should accept the minimal interface needed, but return concrete types. This gives the caller flexibility without sacrificing utility.
2.  **Define interfaces in the consuming package**: An interface should be defined by the package that uses it, not the package that implements it. This keeps dependencies pointing from the consumer to the provider.
3.  **Keep interfaces small**: The smaller the interface, the more flexible it is. Aim for 1-3 methods. Single-method interfaces are ideal (e.g., `io.Reader`, `io.Writer`).
4.  **Don't create an interface until you need one**: Avoid premature abstraction. Wait until you have multiple concrete types that need to be treated polymorphically.

## Naming Conventions
- Single-method interfaces are often named with an "-er" suffix (e.g., `Reader`, `Stringer`).
- For interfaces with multiple methods, choose a name that describes its purpose (e.g., `ReadWriteCloser`).
- Do not use a prefix like `I` (e.g., `IReader`).

## Interface Compliance
Verify that a type implements an interface at compile time with a blank assignment.

```go
// This line will fail to compile if RedisStorage does not satisfy the Storage interface.
var _ Storage = (*RedisStorage)(nil)
```

## `interface{}` (Empty Interface)
- Avoid using `interface{}` when a more specific type is known. It defeats static type checking.
- `interface{}` is acceptable for truly generic data structures or when working with dynamic data formats like JSON.
- In Go 1.18+, prefer type parameters (generics) over `interface{}` for type-safe generic programming.

## Interface Segregation Principle
Clients should not be forced to depend on methods they do not use. Break down large interfaces into smaller, more specific ones.

```go
// Instead of one large interface:
type ReadWriterDeleter interface {
    Read()
    Write()
    Delete()
}

// Prefer smaller, role-based interfaces:
type Reader interface { Read() }
type Writer interface { Write() }
type Deleter interface { Delete() }
```
{{else}}
## Core Interface Rules

1. **Accept interfaces, return structs**
2. **Define interfaces where used, not where implemented**
3. **Keep interfaces small** (1-3 methods ideal)
4. **Don't define interfaces before needed** (YAGNI)

## Interface Placement

```go
// ✅ GOOD: Interface in consumer package
// consumer/consumer.go
package consumer

// Define interface where it's used
type Storage interface {
    Get(key string) ([]byte, error)
    Set(key string, value []byte) error
}

func ProcessStorage(s Storage) error {
    data, err := s.Get("key")
    if err != nil {
        return fmt.Errorf("get data: %w", err)
    }
    // Process data
    return nil
}

// ✅ GOOD: Return concrete type
// provider/provider.go
package provider

type RedisStorage struct {
    client *redis.Client
}

// Return concrete type, not interface
func NewRedisStorage() *RedisStorage {
    return &RedisStorage{
        client: redis.NewClient(),
    }
}

func (r *RedisStorage) Get(key string) ([]byte, error) {
    return r.client.Get(key)
}

// ❌ BAD: Interface in provider package
// Don't define interface next to implementation
```

## Small, Focused Interfaces

```go
// ✅ Single-method interfaces (best)
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

type Closer interface {
    Close() error
}

// ✅ Compose when multiple methods needed
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// ❌ Large interfaces (avoid)
type StorageManager interface {
    Create(string) error
    Read(string) ([]byte, error)
    Update(string, []byte) error
    Delete(string) error
    List() ([]string, error)
    Count() int
    Clear() error
    Backup() error
    Restore(string) error
    // Too many methods!
}
```

## Interface Compliance Checks

```go
// Compile-time verification
var _ Storage = (*RedisStorage)(nil)
var _ Storage = (*MemoryStorage)(nil)

// Full implementation
type RedisStorage struct {
    // fields
}

func (r *RedisStorage) Get(key string) ([]byte, error) {
    // implementation
}

func (r *RedisStorage) Set(key string, value []byte) error {
    // implementation
}
```

## Interface Naming Conventions

```go
// Single method: -er suffix
type Getter interface {
    Get(string) ([]byte, error)
}

// Standard library patterns
type Stringer interface {
    String() string
}

type Handler interface {
    Handle(Context) error  
}

// Multiple methods: descriptive name
type StorageStore interface {
    Get(string) ([]byte, error)
    Set(string, []byte) error
    Delete(string) error
}

// ❌ Avoid I prefix or Interface suffix
type IStorage interface {} // Bad
type StorageInterface interface {} // Bad
```

## Empty Interface Usage

```go
// ❌ Avoid empty interface where possible
func process(data interface{}) {} // Too generic

// ✅ Be specific when you can
func process(data []byte) {}

// ✅ Empty interface OK for truly generic containers
type Cache struct {
    mu    sync.RWMutex
    items map[string]interface{} // Genuine need for any type
}

// ✅ Use type parameters in Go 1.18+
type Cache[T any] struct {
    mu    sync.RWMutex
    items map[string]T
}
```

## Interface Segregation

```go
// ✅ Separate interfaces for different consumers
// For readers
type StorageReader interface {
    Get(key string) ([]byte, error)
}

// For writers
type StorageWriter interface {
    Set(key string, value []byte) error
    Delete(key string) error
}

// For admins
type StorageAdmin interface {
    Clear() error
    Stats() Statistics
}

// Concrete type can implement all
type FullStorage struct{}

var _ StorageReader = (*FullStorage)(nil)
var _ StorageWriter = (*FullStorage)(nil)
var _ StorageAdmin = (*FullStorage)(nil)
```

## Testing with Interfaces

```go
// Define minimal interface for testing
type StorageMock struct {
    GetFunc func(string) ([]byte, error)
    SetFunc func(string, []byte) error
}

func (m *StorageMock) Get(key string) ([]byte, error) {
    if m.GetFunc != nil {
        return m.GetFunc(key)
    }
    return nil, nil
}

func (m *StorageMock) Set(key string, value []byte) error {
    if m.SetFunc != nil {
        return m.SetFunc(key, value)
    }
    return nil
}

// Use in tests
func TestProcess(t *testing.T) {
    mock := &StorageMock{
        GetFunc: func(key string) ([]byte, error) {
            return []byte("test data"), nil
        },
    }
    
    err := ProcessStorage(mock)
    if err != nil {
        t.Errorf("unexpected error: %v", err)
    }
}
```

## Interface Evolution

```go
// Start small
type StorageV1 interface {
    Get(string) ([]byte, error)
}

// Extend via embedding when truly needed
type StorageV2 interface {
    StorageV1
    Set(string, []byte) error
}

// Or create new interface
type ExtendedStorage interface {
    Get(string) ([]byte, error)
    GetBatch([]string) (map[string][]byte, error)
}
```
{{end}}
