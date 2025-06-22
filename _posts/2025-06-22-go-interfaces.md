---
layout: post
title:  "Go Standard Library Interfaces Reference"
date:   2025-06-22 10:00:00 +0530
categories: engineering coding golang
---

## What Makes Go Interfaces Special?

Go interfaces are fundamentally different from interfaces in languages like Java or C#. Here are the key distinctions:

**Implicit Implementation**: In Go, you don't explicitly declare that a type implements an interface. If a type has the methods that an interface requires, it automatically satisfies that interface. This is called "structural typing" and enables incredible flexibility.

**Small and Focused**: Go interfaces tend to be very small - often just one or two methods. The most powerful interfaces in Go's standard library have only a single method (`io.Reader`, `io.Writer`, `error`). This follows the principle: "The bigger the interface, the weaker the abstraction."

**Composition Over Inheritance**: Go doesn't have inheritance, but interfaces can embed other interfaces. This allows you to build complex behaviors by composing simple, focused interfaces.

**Duck Typing with Compile-Time Safety**: Go gives you the flexibility of duck typing ("if it walks like a duck and quacks like a duck, it's a duck") but with compile-time verification that methods actually exist.

The interfaces below represent the core abstractions that make Go's standard library so composable and powerful. Understanding these interfaces is key to writing idiomatic Go code.

## Core I/O Interfaces

### io.Reader
**Package**: `io`  
**Purpose**: Represents the read end of a stream of data

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

**Key Concept**: Universal abstraction for reading data from any source  
**Common Implementations**: `os.File`, `strings.Reader`, `bytes.Buffer`, `http.Request.Body`, `gzip.Reader`

**Example Usage**:
```go
func processData(r io.Reader) error {
    data, err := io.ReadAll(r)
    if err != nil {
        return err
    }
    // Process data...
    return nil
}

// Works with files, HTTP bodies, strings, etc.
processData(strings.NewReader("hello world"))
processData(resp.Body) // HTTP response
```

### io.Writer
**Package**: `io`  
**Purpose**: Represents the write end of a stream of data

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

**Key Concept**: Universal abstraction for writing data to any destination  
**Common Implementations**: `os.File`, `os.Stdout`, `bytes.Buffer`, `http.ResponseWriter`, `gzip.Writer`

**Example Usage**:
```go
func writeLog(w io.Writer, message string) {
    fmt.Fprintf(w, "[%s] %s\n", time.Now().Format(time.RFC3339), message)
}

// Works with files, HTTP responses, buffers, etc.
writeLog(os.Stdout, "Application started")
writeLog(logFile, "User logged in")
```

### io.Closer
**Package**: `io`  
**Purpose**: Represents resources that can be closed

```go
type Closer interface {
    Close() error
}
```

**Key Concept**: Ensures proper resource cleanup  
**Common Implementations**: `os.File`, `http.Response.Body`, database connections

### 4. Composed I/O Interfaces

```go
// Combines reading and writing
type ReadWriter interface {
    Reader
    Writer
}

// For readable resources that need cleanup
type ReadCloser interface {
    Reader
    Closer
}

// For writable resources that need cleanup
type WriteCloser interface {
    Writer
    Closer
}

// Full I/O with cleanup
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

## Error Handling

### error (Built-in)
**Package**: Built-in  
**Purpose**: Standard way to represent error conditions

```go
type error interface {
    Error() string
}
```

**Key Concept**: Any type implementing `Error() string` can be used as an error  
**Usage Pattern**: Functions return `(result, error)` where error is nil on success

**Example Implementation**:
```go
type ValidationError struct {
    Field string
    Value interface{}
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("invalid value %v for field %s", e.Value, e.Field)
}

// Usage
func validateAge(age int) error {
    if age < 0 {
        return ValidationError{Field: "age", Value: age}
    }
    return nil
}
```

## String Representation

### fmt.Stringer
**Package**: `fmt`  
**Purpose**: Provides string representation for custom types

```go
type Stringer interface {
    String() string
}
```

**Key Concept**: Automatically used by `fmt` package for printing  
**Similar to**: `toString()` in Java, `__str__()` in Python

**Example**:
```go
type User struct {
    ID   int
    Name string
}

func (u User) String() string {
    return fmt.Sprintf("User{ID: %d, Name: %s}", u.ID, u.Name)
}

// Automatically uses String() method
user := User{ID: 1, Name: "Alice"}
fmt.Println(user) // Output: User{ID: 1, Name: Alice}
```

## HTTP Handling

### http.Handler
**Package**: `net/http`  
**Purpose**: Core interface for HTTP request handling

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

**Key Concept**: Foundation of all HTTP servers in Go  
**Usage**: Middleware, routers, and handlers all implement this interface

**Example**:
```go
type HelloHandler struct{}

func (h HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}

// Usage
http.Handle("/hello/", HelloHandler{})
```

### http.ResponseWriter
**Package**: `net/http`  
**Purpose**: Interface for constructing HTTP responses

```go
type ResponseWriter interface {
    Header() Header
    Write([]byte) (int, error)
    WriteHeader(statusCode int)
}
```

**Note**: Also implements `io.Writer`

## Sorting and Comparison

### sort.Interface
**Package**: `sort`  
**Purpose**: Enables sorting of custom collections

```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

**Key Concept**: Decouples sorting algorithm from data structure

**Example**:
```go
type People []Person

func (p People) Len() int           { return len(p) }
func (p People) Less(i, j int) bool { return p[i].Age < p[j].Age }
func (p People) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

// Usage
sort.Sort(people) // Sorts by age
```

## Context and Cancellation

### context.Context
**Package**: `context`  
**Purpose**: Carries deadlines, cancellation signals, and request-scoped values

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

**Key Methods**:
- `Done()`: Channel closed when context is canceled
- `Err()`: Reason for cancellation
- `Value()`: Request-scoped values

**Example**:
```go
func doWork(ctx context.Context) error {
    select {
    case <-time.After(2 * time.Second):
        return nil // Work completed
    case <-ctx.Done():
        return ctx.Err() // Canceled or timed out
    }
}

// Usage with timeout
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel()
err := doWork(ctx)
```

## JSON Handling

### json.Marshaler and json.Unmarshaler
**Package**: `encoding/json`  
**Purpose**: Custom JSON serialization/deserialization

```go
type Marshaler interface {
    MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
    UnmarshalJSON([]byte) error
}
```

**Example**:
```go
type Timestamp time.Time

func (t Timestamp) MarshalJSON() ([]byte, error) {
    return json.Marshal(time.Time(t).Unix())
}

func (t *Timestamp) UnmarshalJSON(data []byte) error {
    var unix int64
    if err := json.Unmarshal(data, &unix); err != nil {
        return err
    }
    *t = Timestamp(time.Unix(unix, 0))
    return nil
}
```



## Key Patterns and Best Practices

### Interface Composition
Go interfaces are often composed of smaller interfaces:
```go
type ReadWriteCloser interface {
    Reader    // Embedded interface
    Writer    // Embedded interface
    Closer    // Embedded interface
}
```

### Accept Interfaces, Return Structs
```go
// Good: Accept interface (flexible)
func ProcessData(r io.Reader) (*Result, error) { ... }

// Good: Return concrete type (clear)
func NewProcessor() *DataProcessor { ... }
```

### Empty Interface
```go
interface{} // Can hold any value (similar to Object in Java)
any         // Alias for interface{} (Go 1.18+)
```

### Interface Satisfaction
Interfaces are satisfied implicitly - no explicit "implements" keyword needed:
```go
type MyWriter struct{}

func (w MyWriter) Write(p []byte) (int, error) {
    // Implementation
    return len(p), nil
}

// MyWriter automatically satisfies io.Writer interface
var w io.Writer = MyWriter{}
```

These interfaces form the foundation of Go's standard library and enable the composable, flexible design patterns that make Go code so powerful and reusable. 