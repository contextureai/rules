---
title: Go Advanced Patterns and Practices
description: CLI design, JSON handling, functional options, and other advanced Go patterns
tags: [go]
trigger:
  type: model
  description: When designing CLIs, APIs with options, JSON handling, or time-based code
languages: [go]
variables:
    extended: false
---

# Go Advanced Patterns and Practices

{{if not .extended}}
## Functional Options
For constructors that take a complex configuration, the functional options pattern provides a flexible and clean API. It's more scalable than a large config struct or a long list of constructor arguments.

```go
// Define an Option as a function that modifies a Server.
type Option func(*Server)

// Implement specific options.
func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(timeout time.Duration) Option {
    return func(s *Server) { s.timeout = timeout }
}

// The constructor applies the provided options.
func NewServer(opts ...Option) *Server {
    s := &Server{ /* default values */ }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage is clean and self-documenting.
server := NewServer(WithPort(9000), WithTimeout(time.Minute))
```

## JSON Handling
- **Struct Tags**: Use `json` struct tags to control marshaling and unmarshaling. Common options include renaming fields (`"fieldName"`), omitting empty values (`",omitempty"`), and skipping fields (`"-"`).
- **Streaming**: For large JSON datasets, use `json.Encoder` and `json.Decoder` to work with streams (`io.Reader`/`io.Writer`) and avoid loading the entire dataset into memory.
- **Unknown Fields**: To reject requests with fields not defined in your struct, use `json.Decoder.DisallowUnknownFields()`.

## Validation
A common pattern is to add a `Validate()` method to structs that need validation. This method can be called in constructors or HTTP handlers to ensure the object is in a valid state.

```go
type User struct {
    Name string
    Age  int
}

func (u User) Validate() error {
    if u.Name == "" {
        return errors.New("name is required")
    }
    if u.Age < 0 {
        return errors.New("age cannot be negative")
    }
    return nil
}
```

## Embedding
- **Use embedding to compose behavior**, typically with interfaces (e.g., `io.ReadWriter`).
- **Do not embed types with state that you don't want to expose**, like `sync.Mutex`. Use a named field instead to control access.

## CLI Design
- **Use the `flag` package** for simple command-line tools. Flags should be defined in `main` packages, not libraries.
- For complex applications with subcommands, consider a library like `cobra` or `urfave/cli`, or implement a basic dispatcher using `flag.NewFlagSet`.

## Structured Logging
Use a structured logging library (e.g., `slog`, `zap`, `zerolog`) to produce machine-readable logs. Add contextual information (like a request ID) to logs to make tracing easier.

{{else}}
## Command-Line Interface Design

### Flag Naming
```go
// ✅ Good: Underscores in flag names
var (
    pollInterval = flag.Duration("poll_interval", time.Minute, "Polling interval")
    maxRetries   = flag.Int("max_retries", 3, "Maximum retry attempts")
    debugMode    = flag.Bool("debug_mode", false, "Enable debug logging")
)

// ❌ Bad: Mixed conventions
var (
    poll_interval = flag.Int("pollInterval", 60, "...") // Inconsistent
)
```

### Flag Scope
```go
// ✅ Good: Flags only in main package
package main

var (
    port = flag.Int("port", 8080, "Server port")
)

func main() {
    flag.Parse()
    // Use flags
}

// ❌ Bad: Flags in library packages
package mylib

var debugFlag = flag.Bool("mylib_debug", false, "...") // Don't do this!

// ✅ Good: Configure libraries via API
package mylib

type Config struct {
    Debug bool
}

func New(cfg Config) *Client {
    // Use config
}
```

### Complex CLIs with Subcommands
```go
// Using standard library
flag.Usage = func() {
    fmt.Fprintf(os.Stderr, "Usage: %s <command> [options]\n", os.Args[0])
    fmt.Fprintf(os.Stderr, "\nCommands:\n")
    fmt.Fprintf(os.Stderr, "  serve    Start the server\n")
    fmt.Fprintf(os.Stderr, "  migrate  Run database migrations\n")
}

// Subcommand pattern
switch os.Args[1] {
case "serve":
    serveCmd := flag.NewFlagSet("serve", flag.ExitOnError)
    port := serveCmd.Int("port", 8080, "Port to listen on")
    serveCmd.Parse(os.Args[2:])
    runServer(*port)
    
case "migrate":
    migrateCmd := flag.NewFlagSet("migrate", flag.ExitOnError)
    dir := migrateCmd.String("dir", "./migrations", "Migration directory")
    migrateCmd.Parse(os.Args[2:])
    runMigrations(*dir)
}
```

## Functional Options Pattern

```go
// Option type
type Option func(*Server)

// Concrete options
func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func WithLogger(logger *log.Logger) Option {
    return func(s *Server) {
        s.logger = logger
    }
}

// Constructor with options
func NewServer(opts ...Option) *Server {
    s := &Server{
        // Defaults
        port:    8080,
        timeout: 30 * time.Second,
        logger:  log.Default(),
    }
    
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// Usage
server := NewServer(
    WithPort(9000),
    WithTimeout(60 * time.Second),
)
```

## JSON Handling

### Struct Tags
```go
type User struct {
    ID        int64     `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email,omitempty"`      // Omit if empty
    Password  string    `json:"-"`                     // Never marshal
    CreatedAt time.Time `json:"created_at"`
    IsAdmin   bool      `json:"is_admin,omitempty"`   // Omit if false
    Metadata  *Metadata `json:"metadata,omitempty"`   // Omit if nil
}

// Custom marshaling
func (u User) MarshalJSON() ([]byte, error) {
    type Alias User // Prevent recursion
    return json.Marshal(&struct {
        CreatedAt string `json:"created_at"`
        *Alias
    }{
        CreatedAt: u.CreatedAt.Format(time.RFC3339),
        Alias:     (*Alias)(&u),
    })
}
```

### JSON Streaming
```go
// Encoding stream of objects
func writeUsers(w io.Writer, users []User) error {
    enc := json.NewEncoder(w)
    for _, user := range users {
        if err := enc.Encode(user); err != nil {
            return fmt.Errorf("encode user: %w", err)
        }
    }
    return nil
}

// Decoding stream
func readUsers(r io.Reader) ([]User, error) {
    var users []User
    dec := json.NewDecoder(r)
    
    for {
        var user User
        err := dec.Decode(&user)
        if err == io.EOF {
            break
        }
        if err != nil {
            return nil, fmt.Errorf("decode user: %w", err)
        }
        users = append(users, user)
    }
    
    return users, nil
}
```

### Unknown Fields
```go
// Strict parsing (reject unknown fields)
func parseStrict(data []byte) (*Config, error) {
    dec := json.NewDecoder(bytes.NewReader(data))
    dec.DisallowUnknownFields()
    
    var cfg Config
    if err := dec.Decode(&cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    
    return &cfg, nil
}

// Preserve unknown fields
type Config struct {
    Name  string                 `json:"name"`
    Value int                    `json:"value"`
    Extra map[string]interface{} `json:"-"`
}

func (c *Config) UnmarshalJSON(data []byte) error {
    type Alias Config
    aux := &struct {
        *Alias
    }{
        Alias: (*Alias)(c),
    }
    
    if err := json.Unmarshal(data, &aux); err != nil {
        return err
    }
    
    // Capture unknown fields
    if err := json.Unmarshal(data, &c.Extra); err != nil {
        return err
    }
    
    // Remove known fields
    delete(c.Extra, "name")
    delete(c.Extra, "value")
    
    return nil
}
```

## Time Handling

### Time in APIs
```go
// ✅ Good: Use time.Time in structs
type Event struct {
    ID        string
    Timestamp time.Time
}

// ✅ Good: Accept various formats
func ParseTime(s string) (time.Time, error) {
    formats := []string{
        time.RFC3339,
        time.RFC3339Nano,
        "2006-01-02T15:04:05Z",
        "2006-01-02 15:04:05",
        "2006-01-02",
    }
    
    for _, format := range formats {
        if t, err := time.Parse(format, s); err == nil {
            return t, nil
        }
    }
    
    return time.Time{}, fmt.Errorf("unrecognized time format: %q", s)
}

// ✅ Good: Time zones in context
func (s *Server) ScheduleAt(t time.Time) {
    // Convert to server timezone
    serverTime := t.In(s.location)
    // Schedule...
}
```

### Duration in Config
```go
type Config struct {
    Timeout         Duration `json:"timeout"`
    PollInterval    Duration `json:"poll_interval"`
    BackoffDuration Duration `json:"backoff_duration"`
}

// Custom Duration type for JSON
type Duration time.Duration

func (d Duration) MarshalJSON() ([]byte, error) {
    return json.Marshal(time.Duration(d).String())
}

func (d *Duration) UnmarshalJSON(b []byte) error {
    var s string
    if err := json.Unmarshal(b, &s); err != nil {
        return err
    }
    
    dur, err := time.ParseDuration(s)
    if err != nil {
        return err
    }
    
    *d = Duration(dur)
    return nil
}
```

## Embedding Best Practices

### When to Embed
```go
// ✅ Good: Embedding interfaces for extension
type ReadWriter struct {
    io.Reader
    io.Writer
}

// ✅ Good: Promoting methods intentionally
type MyBuffer struct {
    bytes.Buffer // Promote Buffer methods
}

// ❌ Bad: Embedding concrete types with state
type Server struct {
    sync.Mutex // Exposes Lock/Unlock - bad!
    config     Config
}

// ✅ Good: Named field for mutex
type Server struct {
    mu     sync.Mutex // Private, controlled access
    config Config
}
```

### Method Promotion
```go
type Logger interface {
    Log(string)
}

type Service struct {
    Logger // Promotes Log method
}

// Can override promoted methods
func (s *Service) Log(msg string) {
    // Custom implementation
    s.Logger.Log(fmt.Sprintf("[Service] %s", msg))
}
```

## Validation Patterns

```go
// Validation method pattern
type User struct {
    Name  string
    Email string
    Age   int
}

func (u User) Validate() error {
    var errs []string
    
    if u.Name == "" {
        errs = append(errs, "name is required")
    }
    
    if u.Email == "" {
        errs = append(errs, "email is required")
    } else if !strings.Contains(u.Email, "@") {
        errs = append(errs, "invalid email format")
    }
    
    if u.Age < 0 || u.Age > 150 {
        errs = append(errs, "age must be between 0 and 150")
    }
    
    if len(errs) > 0 {
        return fmt.Errorf("validation failed: %s", strings.Join(errs, "; "))
    }
    
    return nil
}

// Builder pattern with validation
type UserBuilder struct {
    user User
    err  error
}

func (b *UserBuilder) Name(name string) *UserBuilder {
    if b.err != nil {
        return b
    }
    if name == "" {
        b.err = errors.New("name cannot be empty")
        return b
    }
    b.user.Name = name
    return b
}

func (b *UserBuilder) Build() (User, error) {
    if b.err != nil {
        return User{}, b.err
    }
    return b.user, b.user.Validate()
}
```

## SQL and Database Patterns

```go
// SQL placeholder safety
func GetUser(db *sql.DB, id int64) (*User, error) {
    // ✅ Good: Parameterized query
    query := `SELECT id, name, email FROM users WHERE id = $1`
    
    var u User
    err := db.QueryRow(query, id).Scan(&u.ID, &u.Name, &u.Email)
    if err == sql.ErrNoRows {
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("query user: %w", err)
    }
    
    return &u, nil
}

// ❌ Never: String concatenation
query := fmt.Sprintf("SELECT * FROM users WHERE id = %d", id) // SQL injection!
```

## Logging Patterns

```go
// Structured logging with context
type Logger interface {
    Info(msg string, fields ...Field)
    Error(msg string, err error, fields ...Field)
}

func (s *Server) HandleRequest(ctx context.Context, req Request) error {
    logger := s.logger.With(
        Field("request_id", req.ID),
        Field("user_id", req.UserID),
    )
    
    logger.Info("handling request")
    
    if err := s.process(req); err != nil {
        logger.Error("request failed", err)
        return fmt.Errorf("handle request %s: %w", req.ID, err)
    }
    
    logger.Info("request completed",
        Field("duration", time.Since(start)),
    )
    
    return nil
}
```
{{end}}