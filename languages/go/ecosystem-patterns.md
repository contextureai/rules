---
title: Go Ecosystem and External Integration
description: Protocol buffers, gRPC, modules, vendoring, and third-party integration patterns
tags: [go, protobuf, grpc]
trigger:
  type: model
  description: When working with protobuf, gRPC, Go modules, or external dependencies
languages: [go]
variables:
  extended: false
---
{{if not .extended}}
## Protocol Buffers (Protobuf)
- **Pass messages by pointer** (`*userpb.User`) as they can be large.
- **Compare messages with `proto.Equal`**, not `==` or `reflect.DeepEqual`.
- **Clone messages with `proto.Clone`** before modification to avoid side effects.
- **Use `GetX()` accessors** (e.g., `user.GetName()`) for safe access to fields, which returns a zero value if the parent message is `nil`.
- **Validate incoming messages** to ensure required fields are set.

## gRPC
- **Embed `Unimplemented...Server`** in your server implementation for forward compatibility.
- **Use gRPC status codes** for errors (`status.Error(codes.InvalidArgument, "...")`). This provides structured error information to clients.
- **Propagate `context.Context`** through all gRPC calls for cancellation and timeouts.
- **Handle gRPC errors on the client side** by checking the status code (`status.FromError`) to differentiate between network errors and application errors.

## Go Modules
- `go.mod` defines the module's path and its dependencies.
- `go mod tidy` is your primary tool for keeping `go.mod` and `go.sum` clean.
- Use `replace` directives for local development or to pin a dependency to a specific fork or version.
- `go get -u ./...` updates all dependencies to their latest minor or patch versions.

## HTTP Clients
- **Create and reuse a single `http.Client`** for the lifetime of your application. Reusing clients enables connection pooling and avoids resource exhaustion.
- **Configure timeouts** on the `http.Client` to prevent requests from hanging indefinitely.
- **Always close the response body** (`defer resp.Body.Close()`) to avoid leaking connections.
- **Pass `context.Context`** to requests using `http.NewRequestWithContext` to enable cancellation.

## Database Access
- **Never format SQL queries with `fmt.Sprintf`**. This is a major security risk (SQL injection). Always use parameterized queries with `?` or `$1` placeholders.
- **Use transactions (`sql.Tx`)** to group multiple database operations that must succeed or fail together.
- **Pass `context.Context`** to all database operations (`db.QueryRowContext`, `tx.ExecContext`, etc.) to support cancellation.
{{else}}
## Protocol Buffers

### Import Naming
```go
// ✅ Always use 'pb' suffix for proto imports
import (
    userpb "myproject/gen/user_go_proto"
    orderpb "myproject/gen/order_go_proto"
    sharedpb "myproject/gen/shared_go_proto"
)

// ✅ Underscore in import path requires renaming
import (
    foopb "path/to/foo_service_go_proto"
)
```

### Proto Best Practices
```go
// ✅ Pass proto messages as pointers
func ProcessUser(user *userpb.User) error {
    // Protos can be large and grow over time
    return db.Save(user)
}

// ✅ Use proto.Equal for comparison
if !proto.Equal(msg1, msg2) {
    // Messages differ
}

// ✅ Clone before modifying shared messages
func ModifyUser(user *userpb.User) *userpb.User {
    clone := proto.Clone(user).(*userpb.User)
    clone.Name = "Modified"
    return clone
}

// ✅ Nil checks
if user.GetAddress() != nil {
    city := user.GetAddress().GetCity()
}

// Or using safe navigation
city := user.GetAddress().GetCity() // Returns "" if Address is nil
```

### Proto Validation
```go
// Validate required fields
func ValidateUser(user *userpb.User) error {
    if user.GetId() == "" {
        return errors.New("user ID is required")
    }
    
    if user.GetEmail() == "" {
        return errors.New("email is required")
    }
    
    // Validate nested messages
    if addr := user.GetAddress(); addr != nil {
        if addr.GetCountry() == "" {
            return errors.New("country is required in address")
        }
    }
    
    return nil
}
```

## gRPC Patterns

### Server Implementation
```go
type Server struct {
    UnimplementedServer // Forward compatibility
    
    db     *sql.DB
    logger *log.Logger
}

func NewServer(db *sql.DB) *Server {
    return &Server{
        db:     db,
        logger: log.Default(),
    }
}

func (s *Server) GetUser(
    ctx context.Context,
    req *GetUserRequest,
) (*GetUserResponse, error) {
    // Validate request
    if req.GetUserId() == "" {
        return nil, status.Error(codes.InvalidArgument, "user_id is required")
    }
    
    // Check context
    if err := ctx.Err(); err != nil {
        return nil, status.Error(codes.Canceled, "request canceled")
    }
    
    // Business logic
    user, err := s.db.GetUser(ctx, req.GetUserId())
    if err == sql.ErrNoRows {
        return nil, status.Error(codes.NotFound, "user not found")
    }
    if err != nil {
        s.logger.Printf("database error: %v", err)
        return nil, status.Error(codes.Internal, "internal error")
    }
    
    return &GetUserResponse{
        User: user,
    }, nil
}
```

### Client Usage
```go
func callUserService(ctx context.Context, client Client) error {
    // Add timeout
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()
    
    // Make request
    resp, err := client.GetUser(ctx, &GetUserRequest{
        UserId: "123",
    })
    
    // Handle gRPC errors
    if err != nil {
        if status, ok := status.FromError(err); ok {
            switch status.Code() {
            case codes.NotFound:
                return ErrUserNotFound
            case codes.InvalidArgument:
                return fmt.Errorf("invalid request: %s", status.Message())
            case codes.DeadlineExceeded:
                return ErrTimeout
            default:
                return fmt.Errorf("RPC failed: %w", err)
            }
        }
        return fmt.Errorf("connection error: %w", err)
    }
    
    // Use response
    log.Printf("User: %v", resp.GetUser())
    return nil
}
```

### Streaming RPCs
```go
// Server streaming
func (s *Server) ListUsers(
    req *ListUsersRequest,
    stream ListUsersServer,
) error {
    // Send multiple responses
    for _, user := range users {
        if err := stream.Send(&ListUsersResponse{
            User: user,
        }); err != nil {
            return err
        }
    }
    return nil
}

// Client streaming
func (s *Server) CreateUsers(
    stream CreateUsersServer,
) error {
    var count int32
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&CreateUsersResponse{
                Count: count,
            })
        }
        if err != nil {
            return err
        }
        
        // Process each request
        if err := s.createUser(req.GetUser()); err != nil {
            return err
        }
        count++
    }
}
```

## Go Modules

### Module Structure
```
myproject/
├── go.mod
├── go.sum
├── cmd/
│   └── server/
│       └── main.go
├── internal/        # Private to module
│   └── auth/
├── pkg/            # Public packages
│   └── client/
└── vendor/         # Optional, vendored deps
```

### go.mod Best Practices
```go
module github.com/myorg/myproject

go 1.21

require (
    github.com/google/go-cmp v0.5.9
    google.golang.org/grpc v1.50.0
    google.golang.org/protobuf v1.28.1
)

require (
    // Indirect dependencies listed separately
    golang.org/x/net v0.7.0 // indirect
)

// Version pinning
replace github.com/broken/package => github.com/fixed/package v1.2.3

// Local development
replace github.com/myorg/shared => ../shared

// Exclude broken versions
exclude github.com/bad/package v1.0.0
```

### Dependency Management
```bash
# Add dependency
go get github.com/pkg/errors@latest

# Update dependencies
go get -u ./...

# Tidy dependencies
go mod tidy

# Vendor dependencies
go mod vendor

# Verify dependencies
go mod verify

# Why is a dependency needed?
go mod why github.com/some/package
```

## Import Organization

### Complex Import Groups
```go
import (
    // Standard library
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    
    // External packages
    "github.com/google/go-cmp/cmp"
    "go.uber.org/zap"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    
    // Proto imports
    userpb "myproject/gen/user_go_proto"
    orderpb "myproject/gen/order_go_proto"
    
    // Internal packages
    "myproject/internal/auth"
    "myproject/internal/database"
    "myproject/pkg/utils"
    
    // Side-effect imports (always last)
    _ "github.com/lib/pq"       // PostgreSQL driver
    _ "myproject/internal/init" // Package initialization
)
```

## Testing External Dependencies

### Test Doubles for gRPC
```go
// Mock server for testing
type mockServer struct {
    UnimplementedServer
    
    getUserFunc func(context.Context, *GetUserRequest) (*GetUserResponse, error)
}

func (m *mockServer) GetUser(
    ctx context.Context,
    req *GetUserRequest,
) (*GetUserResponse, error) {
    if m.getUserFunc != nil {
        return m.getUserFunc(ctx, req)
    }
    return nil, status.Error(codes.Unimplemented, "not implemented")
}

// Test using mock
func TestClient(t *testing.T) {
    // Setup mock server
    mock := &mockServer{
        getUserFunc: func(ctx context.Context, req *GetUserRequest) (*GetUserResponse, error) {
            return &GetUserResponse{
                User: &User{
                    Id:   req.GetUserId(),
                    Name: "Test User",
                },
            }, nil
        },
    }
    
    // Create test server
    srv := grpc.NewServer()
    RegisterServer(srv, mock)
    
    // Test client against mock
    // ...
}
```

### Integration Testing
```go
// Real server for integration tests
func TestIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    // Start real dependencies
    ctx := context.Background()
    db := setupTestDB(t)
    defer db.Close()
    
    server := NewServer(db)
    
    // Test with real server
    resp, err := server.GetUser(ctx, &GetUserRequest{
        UserId: "test-user",
    })
    
    if err != nil {
        t.Fatalf("GetUser failed: %v", err)
    }
    
    // Assertions...
}
```

## Working with External APIs

### HTTP Client Best Practices
```go
// Reuse HTTP clients
var defaultClient = &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}

func callAPI(ctx context.Context, url string) (*Response, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }
    
    req.Header.Set("User-Agent", "myapp/1.0")
    
    resp, err := defaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("HTTP request: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, body)
    }
    
    var result Response
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("decode response: %w", err)
    }
    
    return &result, nil
}
```

## Common Third-Party Libraries

### Database/SQL
```go
// Always use prepared statements
stmt, err := db.Prepare("SELECT * FROM users WHERE id = $1")
if err != nil {
    return err
}
defer stmt.Close()

// Use transactions for multiple operations
tx, err := db.BeginTx(ctx, nil)
if err != nil {
    return err
}
defer tx.Rollback() // No-op if committed

// Operations...

return tx.Commit()
```

### Popular Library Patterns
```go
// Zap logging
logger, _ := zap.NewProduction()
defer logger.Sync()

logger.Info("processing request",
    zap.String("user_id", userID),
    zap.Int("items", len(items)),
)

// Cobra CLI
var rootCmd = &cobra.Command{
    Use:   "app",
    Short: "Application description",
    Run: func(cmd *cobra.Command, args []string) {
        // Main logic
    },
}

// Viper configuration
viper.SetConfigName("config")
viper.AddConfigPath("/etc/app/")
viper.AddConfigPath(".")
if err := viper.ReadInConfig(); err != nil {
    log.Fatal(err)
}
```
{{end}}