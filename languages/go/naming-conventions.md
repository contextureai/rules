---
title: Go Naming Conventions
description: Enforce idiomatic Go naming for packages, functions, variables, and types
tags: [go, style, naming]
trigger:
  type: glob
  globs: ["**/*.go", "!**/*_test.go"]
languages: [go]
variables:
  extended: "false"
---
# Go Naming Conventions

{{if not .extended}}
## Package Names
- **lowercase**, **short**, **single-word**, **singular**: `net/url`, not `net/urls`
- **descriptive**: avoid generic names: `util`, `common`

## Function, Method & Variable Names
- `PascalCase` for exported, `camelCase` for unexported.
- **Getters omit `Get`**: `c.Name()` not `c.GetName()`.
- **Setters keep `Set`**: `c.SetName("new")`.
- Short names in small scopes (`i`, `r`), descriptive names in large scopes (`responseBody`).
- Common abbreviations: `r` for `io.Reader`, `ctx` for `context.Context`, `err` for `error`.

## Receiver Names
- Use a 1-2 character abbreviation of the type, consistently.
```go
func (c *Client) Method() {}      // ✅
func (this *Client) Method() {}  // ❌
```

## Constants
- Use `MixedCaps`, not `SCREAMING_SNAKE_CASE`. Follow export naming (`PascalCase`/`camelCase`).
```go
const DefaultTimeout = 30 * time.Second // ✅ Exported
const defaultPort = 8080                // ✅ Unexported
const DEFAULT_TIMEOUT = 30              // ❌
```

## Interfaces
- Suffix single-method interfaces with `-er`: `Reader`, `Writer`.
- Use descriptive names for multi-method interfaces: `ReadWriteCloser`.

## Initialisms & Acronyms
- Keep case consistent (all upper or all lower).
```go
var URLPath string // ✅ Not UrlPath
var userID int     // ✅ Not userId
```

## Errors
- Error variables: `var ErrNotFound = errors.New("not found")`
- Error types: `type ValidationError struct { ... }`
{{else}}
## Package Names
- **lowercase only**: `tabwriter` not `tab_writer` or `TabWriter`
- **short, single words**: prefer `http` over `hypertext_protocol`
- **not plural**: `net/url` not `net/urls`
- **avoid generics**: never `util`, `common`, `misc`, `api`, `types`, `interfaces`

## Function & Method Names
```go
// Exported: PascalCase
func ClientConfig() {}
func NewClient() *Client {}

// Unexported: camelCase  
func clientHelper() {}
func parseClientData() {}

// Getters: no Get prefix
func (c *Client) Name() string { return c.name }     // ✅
func (c *Client) GetName() string { return c.name }  // ❌

// Setters: keep Set prefix
func (c *Client) SetName(name string) { c.name = name }
```

## Variable Names
```go
// Short names in small scopes
for i := 0; i < 10; i++ {} // ✅
for index := 0; index < 10; index++ {} // ❌ too verbose

// Descriptive names in large scopes
var responseBodyBuffer []byte // package-level
var buf []byte // function-level

// Common short names
var r io.Reader
var w io.Writer  
var ctx context.Context
var err error
```

## Receiver Names
```go
// 1-2 char abbreviation of type
func (c *Client) Method() {} // ✅
func (cl *Client) Method() {} // ✅ if 'c' ambiguous

// Never use these
func (this *Client) Method() {} // ❌
func (self *Client) Method() {} // ❌
func (me *Client) Method() {} // ❌
```

## Constants
```go
// MixedCaps, not SCREAMING_SNAKE_CASE
const DefaultTimeout = 30 * time.Second // ✅
const DEFAULT_TIMEOUT = 30 * time.Second // ❌
const kDefaultTimeout = 30 * time.Second // ❌

// Exported
const MaxRetries = 3
const MinConnections = 2

// Unexported
const defaultPort = 8080
const bufferSize = 1024
```

## Interfaces
```go
// Single method: -er suffix
type Reader interface { Read([]byte) (int, error) }
type Writer interface { Write([]byte) (int, error) }
type Closer interface { Close() error }

// Multiple methods: descriptive name
type ReadWriteCloser interface {
    Read([]byte) (int, error)
    Write([]byte) (int, error)
    Close() error
}
```

## Initialisms & Acronyms
```go
// Consistent case throughout
var URLPath string  // ✅ not UrlPath
var userID int      // ✅ not userId  
type HTTPServer {}  // ✅ not HttpServer
type XMLParser {}   // ✅ not XmlParser

// In unexported
var httpClient *http.Client // ✅ not httpClient
var xmlData []byte          // ✅ not xmlData
```

## Error Variables
```go
// Package-level errors: ErrXxx
var ErrNotFound = errors.New("not found")
var ErrInvalidInput = errors.New("invalid input")

// Error types: XxxError
type ValidationError struct {
    Field string
    Reason string
}
```
{{end}}
