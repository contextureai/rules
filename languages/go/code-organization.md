---
title: Go Code Organization and Imports
description: Structure packages, imports, and files following Go conventions
tags: [go, packages]
trigger:
  type: glob
  globs: ["**/*.go"]
languages: [go]
variables:
  extended: "false"
---

# Go Code Organization and Imports

{{if not .extended}}
## Import Grouping
Group imports into three blocks, separated by a blank line, in this order:
1.  Standard library packages
2.  Third-party packages
3.  Internal project packages

Within each group, imports should be alphabetized. `goimports` automates this.

## Import Aliasing
- Only alias an import to avoid a name collision.
- A common exception is aliasing protobuf packages (e.g., `pb "path/to/api/proto"`).
- Avoid dot imports (`. "package"`) outside of test files, as they obscure the origin of top-level identifiers.
- Blank imports (`_ "package"`) are used for their side effects (e.g., registering a database driver) and should typically only appear in `main` packages.

## Project Layout
A standard Go project layout helps with maintainability:
- `cmd/`: Main application entry points. Each subdirectory is a runnable command.
- `internal/`: Private application and library code. It cannot be imported by other projects. This is the best place for most of your code.
- `pkg/`: Public library code intended for external use. Only put code here if you intend for other projects to import it.
- `api/`: API definitions (e.g., protobuf files, OpenAPI specs).

## Package Naming
- Use short, single-word, all-lowercase names.
- The name should describe the package's purpose (e.g., `http`, `json`).
- Avoid generic names like `util`, `common`, or `helpers`.

## Code Layout Within a File
Structure your files logically for readability:
1.  Package documentation comment
2.  `import` statements
3.  `const` declarations
4.  `var` declarations
5.  Type declarations (`interface`s before `struct`s)
6.  Constructor functions (e.g., `New...`) immediately follow their type.
7.  Methods, grouped by receiver type.
8.  Other functions.

## Circular Dependencies
A circular dependency (package `a` imports `b`, and `b` imports `a`) is a compile error. To break the cycle:
1.  **Use an interface**: Define an interface in one of the packages (usually the "consumer") and have the other ("provider") implement it.
2.  **Move code**: If two packages are highly coupled, consider merging them.
{{else}}
## Import Organization

```go
package server

import (
    // Standard library (alphabetical)
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
    
    // External packages (alphabetical)
    "github.com/gorilla/mux"
    "go.uber.org/zap"
    "google.golang.org/grpc"
    
    // Internal packages (alphabetical)
    "github.com/myorg/myproject/internal/config"
    "github.com/myorg/myproject/internal/database"
    "github.com/myorg/myproject/pkg/auth"
)
```

## Import Rules

```go
// ✅ Good imports
import (
    "fmt"
    "strings"
    
    "github.com/pkg/errors"
)

// ❌ Bad: mixed grouping
import (
    "github.com/pkg/errors"
    "fmt"
    "strings"
)

// ✅ Alias only when necessary
import (
    client "github.com/conflicting/http" // Avoid name conflict
    pb "github.com/myorg/myproject/proto/service"   // Proto convention
)

// ❌ Unnecessary aliases
import (
    f "fmt"           // Don't alias standard library
    myhttp "net/http" // Unnecessary
)

// ✅ Blank imports only in main or test
import (
    _ "github.com/lib/pq"       // DB driver in main
    _ "embed"                    // Embed files
)

// ❌ Don't use dot imports except in tests
import . "fmt" // Avoid in production code
```

## Package Structure

```
github.com/myorg/myproject/
├── cmd/
│   └── server/
│       └── main.go          # Application entry point
├── internal/                # Private packages
│   ├── config/
│   ├── database/
│   └── handlers/
├── pkg/                     # Public packages
│   ├── auth/
│   └── middleware/
├── api/                     # API definitions
│   └── proto/
├── scripts/                 # Build/install scripts
├── testdata/               # Test fixtures
├── go.mod
├── go.sum
└── README.md
```

## File Organization Within Package

```go
// server.go - Main type and constructor
package server

type Server struct {
    config Config
    logger *zap.Logger
}

func NewServer(cfg Config) *Server {
    return &Server{
        config: cfg,
        logger: zap.NewNop(),
    }
}

// methods.go - Type methods
func (s *Server) Start(ctx context.Context) error {
    // Implementation
}

func (s *Server) Stop() error {
    // Implementation
}

// handlers.go - HTTP handlers (if applicable)
func (s *Server) handleRequest(w http.ResponseWriter, r *http.Request) {
    // Implementation
}

// utils.go - Package-level utilities
func validateServerConfig(cfg Config) error {
    // Implementation
}
```

## Type and Function Ordering

```go
package server

// 1. Package documentation
// Package server provides server functionality.

// 2. Imports
import (
    "context"
    "fmt"
)

// 3. Constants
const (
    DefaultPort = 8080
    MaxConnections = 100
)

// 4. Variables
var (
    ErrInvalidConfig = errors.New("invalid configuration")
)

// 5. Types (interfaces first)
type Store interface {
    Get(string) (interface{}, error)
}

// 6. Struct types
type Server struct {
    store Store
}

// 7. Constructor right after its type
func NewServer(store Store) *Server {
    return &Server{store: store}
}

// 8. Methods grouped by receiver
func (s *Server) Get(key string) (interface{}, error) {
    return s.store.Get(key)
}

func (s *Server) Set(key string, val interface{}) error {
    // Implementation
}

// 9. Package-level functions (exported first)
func ValidateKey(key string) error {
    // Implementation
}

// 10. Unexported functions
func isValidKey(key string) bool {
    // Implementation
}

// 11. Helper/utility functions at end
func formatKey(key string) string {
    // Implementation
}
```

## Package Naming Guidelines

```go
// ✅ Good package names
package http      // What it provides
package json      // What it handles
package sql       // Technology it wraps

// ❌ Bad package names
package util      // Too generic
package common    // Meaningless
package base      // Unclear purpose
package misc      // Grab bag
package helpers   // Too vague
package models    // Often unnecessary
package types     // Usually redundant

// Package names for commands
// cmd/server/main.go
package main      // Always main for executables
```

## Internal vs Public Packages

```go
// internal/ - Cannot be imported by external modules
// github.com/myorg/myproject/internal/auth/
package auth // Private to this module

// Can import from same module
import "github.com/myorg/myproject/internal/auth" // ✅ OK

// External modules cannot import
import "github.com/myorg/myproject/internal/auth" // ❌ Blocked

// pkg/ - Public API for external use
// github.com/myorg/myproject/pkg/client/
package client // Public API

// Anyone can import
import "github.com/myorg/myproject/pkg/client" // ✅ OK
```

## Package Documentation

```go
// Package server implements a high-performance server
// for handling HTTP requests with middleware support.
//
// Usage:
//
//	srv := server.New(config)
//	if err := srv.Start(); err != nil {
//	    log.Fatal(err)
//	}
package server

// Or in doc.go for longer documentation
// doc.go
/*
Package server provides comprehensive server functionality.

This package includes:
  - HTTP server with graceful shutdown
  - Middleware chain support
  - Request routing and handling
  - Metrics and monitoring

For more details, see the examples.
*/
package server
```

## Circular Dependencies Prevention

```go
// ❌ Circular dependency
// package a imports package b
// package b imports package a

// ✅ Solution 1: Interface in third package
// common/interfaces.go
package common

type DataStore interface {
    Get(string) (interface{}, error)
}

// ✅ Solution 2: Interface in consumer
// consumer/consumer.go
package consumer

type Provider interface {
    Provide() string
}

// ✅ Solution 3: Merge packages if highly coupled
// Combine into single package if they truly belong together
```
{{end}}
