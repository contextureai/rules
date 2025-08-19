---
title: Go Concurrency Patterns
description: Safe goroutine management, channel patterns, and synchronization best practices
tags: [go, concurrency, goroutines]
trigger:
  type: model
  description: When implementing concurrent code, goroutines, channels, or synchronization
languages: [go]
variables:
    extended: false
---

# Go Concurrency Patterns

{{if not .extended}}
## Goroutine Lifecycle
Every goroutine must be managed: know when it will stop, and have a way to wait for it to complete. Never start a goroutine without a clear termination plan. `sync.WaitGroup` is the standard tool for this.

## `context` for Cancellation
Use `context.Context` to manage cancellation across API boundaries and goroutines. The `select` statement is key for handling context completion.

```go
func operation(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err() // Propagate cancellation error
    case result := <-longRunningTask():
        // ... process result
    }
    return nil
}
```

## Channels
- **Direction**: Specify channel direction (`<-chan T` for receive-only, `chan<- T` for send-only) to enforce API contracts at compile time.
- **Buffering**: Default to unbuffered channels (`make(chan T)`). Use buffered channels (`make(chan T, N)`) only for specific queuing needs, and document the reason for the buffer size.
- **Ownership**: The goroutine that creates a channel is responsible for closing it. Closing a channel signals completion to receivers.

## `select` Statement
- The `select` statement lets a goroutine wait on multiple communication operations.
- A `default` case makes a `select` non-blocking.
- Use `select` with a `time.After` channel for timeouts.

## Synchronization with `sync` Package
- **`sync.Mutex`**: Use a `sync.Mutex` or `sync.RWMutex` to protect shared data. Keep critical sections (the code between `Lock` and `Unlock`) as short as possible. Never embed a mutex; use a named field `mu`.
- **`sync.WaitGroup`**: Use a `WaitGroup` to wait for a collection of goroutines to finish.
- **`sync.Once`**: Use `Once` for one-time initialization.
- **`sync/atomic`**: Use atomic operations for simple counters or flags to avoid locking overhead.

## Common Patterns & Anti-Patterns
- **Worker Pools**: Use a fixed number of goroutines to process a variable number of tasks from a channel. This controls resource usage.
- **Goroutine Leaks**: A goroutine can leak if it blocks indefinitely on a channel or lock. Ensure every channel send has a guaranteed receiver, or use `context` for cancellation.
- **Race Conditions**: A race condition occurs when multiple goroutines access the same data concurrently, and at least one of the accesses is a write. Protect shared data with mutexes or atomic operations. Use the `-race` detector during testing.
{{else}}
## Goroutine Lifecycle Rules

**Cardinal rule**: Never fire-and-forget. Every goroutine must have:
1. Clear termination condition
2. Way to signal completion
3. Mechanism to wait for completion

## Safe Goroutine Pattern

```go
type TaskWorker struct {
    tasks chan Task
    stop  chan struct{}
    done  chan struct{}
    wg    sync.WaitGroup
}

func NewTaskWorker() *TaskWorker {
    return &TaskWorker{
        tasks: make(chan Task),
        stop:  make(chan struct{}),
        done:  make(chan struct{}),
    }
}

func (w *TaskWorker) Start() {
    w.wg.Add(1)
    go w.run()
}

func (w *TaskWorker) run() {
    defer w.wg.Done()
    defer close(w.done)
    
    for {
        select {
        case task := <-w.tasks:
            w.process(task)
        case <-w.stop:
            return
        }
    }
}

func (w *TaskWorker) Stop() {
    close(w.stop)
    w.wg.Wait() // Wait for completion
}
```

## Channel Patterns

### Channel Direction
```go
// ✅ Always specify direction when possible
func sender(out chan<- string) {
    out <- "message"
}

func receiver(in <-chan string) {
    msg := <-in
}

// ❌ Don't use bidirectional unless necessary
func process(ch chan string) {} // Avoid
```

### Channel Sizing
```go
// Unbuffered: synchronous communication
ch := make(chan int)

// Buffer of 1: async send, select default case
ch := make(chan int, 1)

// Specific size: must document why
// Buffer size 10 matches worker pool
workers := make(chan Work, 10)
```

## Worker Pool Pattern

```go
func TaskPool(tasks []Task) {
    var wg sync.WaitGroup
    taskChan := make(chan Task, len(tasks))
    
    // Start 10 workers
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for task := range taskChan {
                log.Printf("Worker %d processing task", id)
                processTask(task)
            }
        }(i)
    }
    
    // Send tasks
    for _, task := range tasks {
        taskChan <- task
    }
    close(taskChan)
    
    // Wait for completion
    wg.Wait()
}
```

## Context for Cancellation

```go
func TaskWithContext(ctx context.Context) error {
    // Create cancellable context
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    resultChan := make(chan Result, 1)
    errChan := make(chan error, 1)
    
    go func() {
        result, err := longRunningTask()
        if err != nil {
            errChan <- err
            return
        }
        resultChan <- result
    }()
    
    select {
    case result := <-resultChan:
        return processResult(result)
    case err := <-errChan:
        return fmt.Errorf("task failed: %w", err)
    case <-ctx.Done():
        return fmt.Errorf("task timeout: %w", ctx.Err())
    }
}
```

## Mutex Patterns

```go
// ✅ Named mutex field
type SafeTaskStore struct {
    mu    sync.RWMutex // Named, not embedded
    data  map[string]interface{}
    stats Stats
}

// ✅ Minimal critical section
func (s *SafeTaskStore) Get(key string) (interface{}, bool) {
    s.mu.RLock()
    val, ok := s.data[key]
    s.mu.RUnlock() // Unlock before processing
    
    if ok {
        // Process outside lock
        return process(val), true
    }
    return nil, false
}

// ✅ Defer for cleanup
func (s *SafeTaskStore) Update(key string, value interface{}) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    s.data[key] = value
    s.stats.Updates++
}

// ❌ Never embed mutex
type Bad struct {
    sync.Mutex // Don't embed
    data string
}
```

## Once for Initialization

```go
type TaskManager struct {
    once     sync.Once
    instance *Processor
    initErr  error
}

func (m *TaskManager) GetProcessor() (*Processor, error) {
    m.once.Do(func() {
        m.instance, m.initErr = initializeProcessor()
    })
    return m.instance, m.initErr
}
```

## Select Patterns

```go
// Non-blocking send
select {
case ch <- value:
    // Sent successfully
default:
    // Channel full, handle accordingly
}

// Timeout pattern
select {
case result := <-resultChan:
    return result
case <-time.After(5 * time.Second):
    return ErrTimeout
}

// First response wins
func fastestTask() string {
    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)
    
    go func() { ch1 <- fetchTaskFromSource1() }()
    go func() { ch2 <- fetchTaskFromSource2() }()
    
    select {
    case result := <-ch1:
        return result
    case result := <-ch2:
        return result
    }
}
```

## Atomic Operations

```go
import "sync/atomic"

type TaskCounter struct {
    count int64
}

func (c *TaskCounter) Increment() {
    atomic.AddInt64(&c.count, 1)
}

func (c *TaskCounter) Get() int64 {
    return atomic.LoadInt64(&c.count)
}

// Prefer go.uber.org/atomic for type safety
import "go.uber.org/atomic"

type BetterTaskCounter struct {
    count atomic.Int64
}

func (c *BetterTaskCounter) Increment() {
    c.count.Inc()
}
```

## Common Anti-Patterns

```go
// ❌ Goroutine leak
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch // Blocks forever if ch never receives
    }()
    // ch is never written to
}

// ✅ Fix: Provide cancellation
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            // Process
        case <-ctx.Done():
            return
        }
    }()
}

// ❌ Race condition
var counter int
for i := 0; i < 100; i++ {
    go func() {
        counter++ // Race!
    }()
}

// ✅ Fix: Use synchronization
var counter atomic.Int64
for i := 0; i < 100; i++ {
    go func() {
        counter.Inc()
    }()
}
```
{{end}}
