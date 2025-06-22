---
layout: post
title:  "GO idiomatic patterns"
date:   2025-06-22 10:00:00 +0530
categories: engineering coding golang
---

# Go Idiomatic Patterns Notebook

## 1. Compile-Time Interface Compliance Check

**Purpose**: Ensure a type implements an interface at compile time

```go
// Basic interface compliance check
var _ http.Handler = (*MyHandler)(nil)
var _ io.Reader = (*MyReader)(nil)
var _ fmt.Stringer = MyType{}

// Real-world example from chi router
var _ Router = &Mux{}

// Multiple interface checks
var (
    _ io.Reader = (*Buffer)(nil)
    _ io.Writer = (*Buffer)(nil)
    _ io.Closer = (*Buffer)(nil)
)
```

## 2. Embedding for Interface Composition

**Purpose**: Compose interfaces and promote methods

```go
// Interface embedding
type ReadWriter interface {
    io.Reader
    io.Writer
}

// Struct embedding for method promotion
type Server struct {
    *http.Server  // Embedded - promotes all methods
    logger Logger
}

// Anonymous embedding in structs
type MyWriter struct {
    io.Writer  // Embedded interface
    prefix    string
}
```

## 3. Functional Options Pattern

**Purpose**: Flexible configuration with optional parameters

```go
type Server struct {
    host string
    port int
    timeout time.Duration
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func NewServer(opts ...Option) *Server {
    s := &Server{
        host: "localhost",
        port: 8080,
        timeout: 30 * time.Second,
    }

    for _, opt := range opts {
        opt(s)
    }

    return s
}

// Usage
server := NewServer(
    WithHost("0.0.0.0"),
    WithPort(9090),
)
```

## 4. Builder Pattern with Method Chaining

**Purpose**: Fluent API for object construction

```go
type QueryBuilder struct {
    query  strings.Builder
    params []interface{}
}

func NewQuery() *QueryBuilder {
    return &QueryBuilder{}
}

func (q *QueryBuilder) Select(fields ...string) *QueryBuilder {
    q.query.WriteString("SELECT ")
    q.query.WriteString(strings.Join(fields, ", "))
    return q
}

func (q *QueryBuilder) From(table string) *QueryBuilder {
    q.query.WriteString(" FROM ")
    q.query.WriteString(table)
    return q
}

func (q *QueryBuilder) Where(condition string, args ...interface{}) *QueryBuilder {
    q.query.WriteString(" WHERE ")
    q.query.WriteString(condition)
    q.params = append(q.params, args...)
    return q
}

func (q *QueryBuilder) Build() (string, []interface{}) {
    return q.query.String(), q.params
}

// Usage
query, params := NewQuery().
    Select("id", "name").
    From("users").
    Where("age > ?", 18).
    Build()
```

## 5. Context Pattern for Cancellation

**Purpose**: Propagate cancellation and timeouts

```go
func DoWork(ctx context.Context) error {
    select {
    case <-time.After(2 * time.Second):
        return nil // work completed
    case <-ctx.Done():
        return ctx.Err() // cancelled or timeout
    }
}

// Usage patterns
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

ctx = context.WithValue(ctx, "userID", 123)

// Pipeline pattern with context
func Pipeline(ctx context.Context, input <-chan int) <-chan int {
    output := make(chan int)
    go func() {
        defer close(output)
        for {
            select {
            case val, ok := <-input:
                if !ok {
                    return
                }
                // Process val
                select {
                case output <- val * 2:
                case <-ctx.Done():
                    return
                }
            case <-ctx.Done():
                return
            }
        }
    }()
    return output
}
```

## 6. Worker Pool Pattern

**Purpose**: Limit concurrent operations

```go
type Job struct {
    ID   int
    Data string
}

type Result struct {
    Job Job
    Err error
}

func Worker(id int, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        // Simulate work
        time.Sleep(time.Millisecond * 100)
        results <- Result{Job: job}
    }
}

func WorkerPool(numWorkers int, jobs []Job) []Result {
    jobChan := make(chan Job, len(jobs))
    resultChan := make(chan Result, len(jobs))

    // Start workers
    for i := 0; i < numWorkers; i++ {
        go Worker(i, jobChan, resultChan)
    }

    // Send jobs
    for _, job := range jobs {
        jobChan <- job
    }
    close(jobChan)

    // Collect results
    var results []Result
    for i := 0; i < len(jobs); i++ {
        results = append(results, <-resultChan)
    }

    return results
}
```

## 7. Singleton Pattern with sync.Once

**Purpose**: Ensure single instance creation

```go
type Database struct {
    connection string
}

var (
    dbInstance *Database
    dbOnce     sync.Once
)

func GetDatabase() *Database {
    dbOnce.Do(func() {
        dbInstance = &Database{
            connection: "db://localhost:5432",
        }
    })
    return dbInstance
}

// Alternative with sync.Once in struct
type Config struct {
    once sync.Once
    data map[string]string
}

func (c *Config) Load() {
    c.once.Do(func() {
        c.data = make(map[string]string)
        // Load configuration
    })
}
```

## 8. Error Wrapping Pattern

**Purpose**: Add context to errors while preserving the original

```go
import (
    "errors"
    "fmt"
)

// Custom error types
type ValidationError struct {
    Field string
    Value interface{}
    Err   error
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed for field %s with value %v: %v",
        e.Field, e.Value, e.Err)
}

func (e *ValidationError) Unwrap() error {
    return e.Err
}

// Error wrapping
func ProcessUser(user User) error {
    if err := validateUser(user); err != nil {
        return fmt.Errorf("failed to process user %s: %w", user.Name, err)
    }
    return nil
}

// Error checking
func HandleError(err error) {
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        // Handle validation error specifically
        log.Printf("Validation error: %s", validationErr.Field)
    }

    if errors.Is(err, ErrNotFound) {
        // Handle not found error
        log.Printf("Resource not found")
    }
}
```

## 9. Middleware Pattern

**Purpose**: Chain processing functions

```go
type Middleware func(http.Handler) http.Handler

func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Chain middlewares
func Chain(middlewares ...Middleware) Middleware {
    return func(next http.Handler) http.Handler {
        for i := len(middlewares) - 1; i >= 0; i-- {
            next = middlewares[i](next)
        }
        return next
    }
}

// Usage
handler := Chain(
    LoggingMiddleware,
    AuthMiddleware,
)(http.HandlerFunc(myHandler))
```

## 10. Table-Driven Tests Pattern

**Purpose**: Parameterized testing with multiple test cases

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"zero", 0, 5, 5},
        {"mixed", -1, 1, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d, want %d",
                    tt.a, tt.b, got, tt.want)
            }
        })
    }
}

// Benchmark table-driven tests
func BenchmarkAdd(b *testing.B) {
    tests := []struct {
        name string
        a, b int
    }{
        {"small", 1, 2},
        {"large", 1000000, 2000000},
    }

    for _, tt := range tests {
        b.Run(tt.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                Add(tt.a, tt.b)
            }
        })
    }
}
```

## 11. Resource Management with defer

**Purpose**: Ensure cleanup happens regardless of function exit path

```go
func ProcessFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close() // Always executed

    // Multiple defers execute in LIFO order
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Recovered from panic: %v", r)
        }
    }()

    defer log.Println("Processing completed")

    // Process file...
    return nil
}

// Resource pool pattern
type Pool struct {
    resources chan Resource
}

func (p *Pool) Get() Resource {
    return <-p.resources
}

func (p *Pool) Put(r Resource) {
    p.resources <- r
}

func UseResource(pool *Pool) error {
    resource := pool.Get()
    defer pool.Put(resource) // Always return to pool

    // Use resource...
    return nil
}
```

## 12. Type Assertion and Type Switch Patterns

**Purpose**: Handle different types dynamically

```go
// Type assertion with ok idiom
func ProcessValue(v interface{}) {
    if str, ok := v.(string); ok {
        fmt.Printf("String: %s", str)
        return
    }

    if num, ok := v.(int); ok {
        fmt.Printf("Number: %d", num)
        return
    }

    fmt.Printf("Unknown type: %T", v)
}

// Type switch pattern
func HandleValue(v interface{}) {
    switch val := v.(type) {
    case string:
        fmt.Printf("String of length %d: %s", len(val), val)
    case int:
        fmt.Printf("Integer: %d", val)
    case []string:
        fmt.Printf("String slice with %d elements", len(val))
    case nil:
        fmt.Println("Nil value")
    default:
        fmt.Printf("Unknown type: %T with value: %v", val, val)
    }
}
```

## 13. Non-Blocking Channel Operations

**Purpose**: Avoid blocking when sending to channels using select with default

```go
// Non-blocking send to avoid blocking goroutines
func NonBlockingSend(ch chan<- int, value int) error {
    select {
    case ch <- value:
        // Success - channel accepted the value
        return nil
    default:
        // Channel is full/blocked - return immediately
        return errors.New("would block")
    }
}

// Non-blocking job submission pattern
// Prefer to use a channel of size 1 to avoid lost events
func SubmitJob(workChan chan<- Job, job Job) bool {
    select {
    case workChan <- job:
        // Job submitted successfully
        return true
    default:
        // All workers busy, handle overflow
        log.Printf("Workers busy, dropping job %d", job.ID)
        return false
    }
}

// Non-blocking receive
func NonBlockingReceive(ch <-chan int) (int, bool) {
    select {
    case value := <-ch:
        // Successfully received value
        return value, true
    default:
        // No value available
        return 0, false
    }
}

// Producer with overflow handling
func ProducerWithOverflow(workChan chan<- Job, overflowChan chan<- Job) {
    job := Job{ID: 1, Data: "work"}
    
    select {
    case workChan <- job:
        // Normal processing
    case overflowChan <- job:
        // Overflow queue
    default:
        // Both channels full, log and drop
        log.Printf("System overloaded, dropping job %d", job.ID)
    }
}

// Timeout pattern with non-blocking operations
func ProcessWithTimeout(workChan chan<- Job, job Job, timeout time.Duration) error {
    select {
    case workChan <- job:
        return nil
    case <-time.After(timeout):
        return errors.New("timeout waiting to submit job")
    }
}
```

These patterns represent some of the most common and useful Go idioms that you'll encounter in well-written Go code. Each serves a specific purpose and helps make code more maintainable, readable, and robust.
