---
title: Go Type System and Memory Management
description: Master Go's type system, memory semantics, and conversion patterns
tags: [go, types, memory, pointers]
trigger:
  type: glob
  globs: ["**/*.go"]
languages: [go]
variables:
  extended: false
---
{{if not .extended}}
## Pointers vs. Values
- **Pass by value** for small, immutable data (basic types, small structs). All arguments in Go are passed by value, meaning a copy is made.
- **Pass by pointer** to:
    1.  **Modify the original value**.
    2.  **Avoid copying large structs**.
- Slices, maps, channels, and strings are reference types (they contain pointers internally), so you typically don't need to pass a pointer to them.

## Method Receivers (Pointer vs. Value)
- Use a **pointer receiver** (`func (s *MyStruct)`) if the method needs to modify the receiver. This is the most common choice.
- Use a **value receiver** (`func (s MyStruct)`) only if the method does not modify the receiver and the struct is small and immutable.
- **Be consistent**: If any method on a type has a pointer receiver, all methods on that type should have a pointer receiver.

## Slices vs. Arrays
- **Arrays** are fixed-size and are value types. A copy is made on assignment or function call. Prefer slices over arrays in most cases.
- **Slices** are dynamic and are reference types. They hold a pointer to an underlying array.
    - `len(s)` is the number of elements in the slice.
    - `cap(s)` is the capacity of the underlying array.
    - **Beware**: Reslicing (`s[1:3]`) creates a new slice that shares the same underlying array. Modifying one can affect the other.
    - Use `append` to grow a slice. If the capacity is exceeded, a new, larger array is allocated.

## Maps
- Maps are reference types.
- A `nil` map cannot have keys added to it; you must initialize it with `make(map[K]V)`.
- Iteration order over a map is not guaranteed.
- **Maps are not safe for concurrent use**. Protect concurrent access with `sync.RWMutex` or use `sync.Map`.
- You cannot take the address of a map element (`&myMap["key"]`).

## Type Assertions and Switches
- A **type assertion** (`x.(T)`) extracts the underlying concrete value from an interface.
- Always use the "comma, ok" idiom to safely check the type: `val, ok := x.(string)`. The single-value form will panic if the assertion fails.
- A **type switch** provides a clean way to perform different actions based on the dynamic type of an interface variable.

```go
switch v := i.(type) {
case int:
    // v is an int
case string:
    // v is a string
default:
    // i is some other type
}
```
{{else}}
## Pointer vs Value Semantics

### Pass by Value
```go
// ✅ Small, immutable types: pass by value
func UpdateTime(t time.Time) time.Time {
    return t.Add(time.Hour)
}

// ✅ Basic types: always by value
func Double(n int) int {
    return n * 2
}

// ❌ Don't pass pointers to save bytes
func Bad(s *string) { // string is already a reference type
    fmt.Println(*s)
}

// ✅ Pass string directly
func Good(s string) {
    fmt.Println(s)
}
```

### Pass by Pointer
```go
// ✅ Large structs: use pointers
type Data struct {
    // Many fields...
    Items [1000]Item
}

func Process(d *Data) error {
    // Modify d
}

// ✅ When mutation is needed
func (c *Counter) Increment() {
    c.value++
}

// ✅ Protocol buffers: always pointers
func SendMessage(msg *pb.Message) error {
    return proto.Marshal(msg)
}
```

## Arrays vs Slices

### Arrays: Fixed Size, Value Type
```go
// Arrays are values, not references
var a [3]int = [3]int{1, 2, 3}
b := a  // Copies entire array
b[0] = 99
fmt.Println(a[0]) // Still 1

// Pass pointer to avoid copying
func Sum(arr *[100]float64) float64 {
    var sum float64
    for _, v := range *arr {
        sum += v
    }
    return sum
}
```

### Slices: Dynamic, Reference Type
```go
// Slices are references to underlying arrays
var s []int

// Three ways to create
s = []int{1, 2, 3}           // Literal
s = make([]int, 10)          // Length 10, capacity 10
s = make([]int, 10, 100)     // Length 10, capacity 100

// Slice operations
s = append(s, 4, 5, 6)
subslice := s[1:3]  // Shares underlying array!

// ⚠️ Beware of shared backing arrays
original := []int{1, 2, 3, 4, 5}
slice1 := original[1:3]  // [2, 3]
slice2 := original[2:4]  // [3, 4]
slice1[1] = 99            // Modifies original and slice2!
```

### Slice Tricks
```go
// Copy a slice
dst := make([]int, len(src))
copy(dst, src)

// Or using append
dst := append([]int(nil), src...)

// Delete element at index i
s = append(s[:i], s[i+1:]...)

// Insert element at index i
s = append(s[:i], append([]int{x}, s[i:]...)...)

// Clear slice but keep capacity
s = s[:0]

// Release underlying array for GC
s = nil
```

## Map Patterns

### Map Basics
```go
// Declaration and initialization
var m map[string]int           // nil map, can't write
m = make(map[string]int)        // Empty map, can write
m = map[string]int{}            // Empty map, can write
m = map[string]int{             // Literal
    "foo": 1,
    "bar": 2,
}

// Check existence
value, ok := m[key]
if !ok {
    // Key doesn't exist
}

// Delete
delete(m, key) // Safe even if key doesn't exist

// Iterate (random order!)
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}
```

### Map Gotchas
```go
// ❌ Maps are not safe for concurrent use
// Use sync.Map or mutex

type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    val, ok := sm.m[key]
    return val, ok
}

// ⚠️ Can't take address of map elements
m := map[string]struct{ x int }{"foo": {x: 1}}
// ptr := &m["foo"].x  // Compile error!

// Workaround: Use pointer values
m2 := map[string]*struct{ x int }{"foo": {x: 1}}
m2["foo"].x = 2  // OK
```

## Type Assertions and Switches

### Type Assertions
```go
// Safe assertion with comma-ok
var i interface{} = "hello"

// ✅ Safe
s, ok := i.(string)
if !ok {
    // Handle type mismatch
}

// ❌ Unsafe (panics if wrong type)
s := i.(string)

// Check for interface implementation
var w io.Writer = &bytes.Buffer{}
if closer, ok := w.(io.Closer); ok {
    closer.Close()
}
```

### Type Switches
```go
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %q\n", v)
    case []byte:
        fmt.Printf("Bytes: %x\n", v)
    case nil:
        fmt.Println("nil value")
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

// Type switch for error handling
switch err := err.(type) {
case *os.PathError:
    fmt.Printf("Path error: %s\n", err.Path)
case *json.SyntaxError:
    fmt.Printf("JSON syntax error at offset %d\n", err.Offset)
default:
    fmt.Printf("Unknown error: %v\n", err)
}
```

## Reflection

### When to Use Reflection
```go
// ✅ OK: Serialization/deserialization
json.Marshal(v)

// ✅ OK: Generic programming before generics
func Contains(slice interface{}, item interface{}) bool {
    s := reflect.ValueOf(slice)
    for i := 0; i < s.Len(); i++ {
        if reflect.DeepEqual(s.Index(i).Interface(), item) {
            return true
        }
    }
    return false
}

// ❌ Avoid: When type is known
// Use type assertion or type switch instead
```

### Reflection Patterns
```go
// Check if nil
func IsNil(i interface{}) bool {
    if i == nil {
        return true
    }
    v := reflect.ValueOf(i)
    switch v.Kind() {
    case reflect.Ptr, reflect.Map, reflect.Slice, reflect.Chan, reflect.Func:
        return v.IsNil()
    }
    return false
}

// Get struct tags
type User struct {
    Name string `json:"name" validate:"required"`
}

func getTags(v interface{}) {
    t := reflect.TypeOf(v)
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        jsonTag := field.Tag.Get("json")
        validateTag := field.Tag.Get("validate")
        fmt.Printf("%s: json=%q validate=%q\n", 
            field.Name, jsonTag, validateTag)
    }
}
```

## Type Conversions

### Basic Conversions
```go
// Numeric conversions (explicit)
var i int = 42
var f float64 = float64(i)
var u uint = uint(i)

// String conversions
var b []byte = []byte("hello")
var s string = string(b)

// ⚠️ Beware of precision loss
var big int64 = 1<<62
var small int32 = int32(big) // Truncation!
```

### Interface Conversions
```go
// Any type to interface{} (automatic)
var x interface{} = 42

// Interface to concrete (needs assertion)
i := x.(int)

// Interface to interface
var r io.Reader = &bytes.Buffer{}
var rw io.ReadWriter = r.(io.ReadWriter)
```

## String Internals

```go
// Strings are immutable
s := "hello"
// s[0] = 'H'  // Compile error!

// Convert to []byte for mutation
b := []byte(s)
b[0] = 'H'
s = string(b)

// Efficient string building
var b strings.Builder
for i := 0; i < 1000; i++ {
    b.WriteString("x")
}
result := b.String()

// String interning (careful!)
// Substrings may keep entire original in memory
huge := strings.Repeat("x", 1000000)
small := huge[0:10] // Still references huge!

// Force copy to release memory
small = string([]byte(huge[0:10]))
```

## Method Sets

```go
type T struct{}

func (t T) ValueMethod() {}
func (t *T) PointerMethod() {}

// Method sets:
// - Type T: ValueMethod only
// - Type *T: Both ValueMethod and PointerMethod

var t T
var pt *T

t.ValueMethod()     // OK
t.PointerMethod()   // OK (auto-reference)
pt.ValueMethod()    // OK (auto-dereference)
pt.PointerMethod()  // OK

// But for interfaces:
type Interface interface {
    PointerMethod()
}

var i Interface
i = t   // Error! T doesn't have PointerMethod in method set
i = &t  // OK
i = pt  // OK
```

## Memory Management Tips

```go
// Clear references in slices before truncating
for i := range slice[newLen:] {
    slice[newLen+i] = nil // Allow GC
}
slice = slice[:newLen]

// Reuse allocations with sync.Pool
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

// Be aware of escape analysis
// go build -gcflags="-m" to see what escapes to heap

// Stack allocation (good)
func sum(nums []int) int {
    total := 0 // Stays on stack
    for _, n := range nums {
        total += n
    }
    return total
}

// Heap allocation (when necessary)
func newBuffer() *bytes.Buffer {
    buf := &bytes.Buffer{} // Escapes to heap
    return buf
}
```
{{end}}
