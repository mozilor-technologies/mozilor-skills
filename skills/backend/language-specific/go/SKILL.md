---
name: go
description: Go backend best practices, concurrency patterns, SDK integrations, code review standards, and test generation.
triggers:
  - "golang"
  - "go backend"
  - "go patterns"
  - "go concurrency"
  - "go testing"
  - "go sdk"
  - "go code review"
  - "go routines"
---

# Go — Best Practices, Concurrency Patterns & Testing

## 1) Project Structure (Standard Layout)

```
cmd/
├── server/
│   └── main.go             # Entry point — thin, just wires dependencies
└── worker/
    └── main.go

internal/                   # Private application code
├── auth/
│   ├── handler.go          # HTTP handlers
│   ├── service.go          # Business logic
│   ├── repository.go       # Data access
│   ├── models.go           # Domain models
│   └── service_test.go
├── users/
├── config/
│   └── config.go
└── shared/
    ├── errors/
    ├── middleware/
    └── logger/

pkg/                        # Public reusable packages
├── httpclient/
└── validator/

api/
└── openapi.yaml

scripts/
Makefile
go.mod
go.sum
```

## 2) Context-First Pattern

Every function that does I/O or long computation must accept a context:

```go
// BAD: no context propagation
func (s *UserService) GetUser(id string) (*User, error) {
    return s.repo.FindByID(id)
}

// GOOD: context first, always
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    return s.repo.FindByID(ctx, id)
}

// Context carries: cancellation, deadlines, request-scoped values, tracing
func handleRequest(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    user, err := userService.GetUser(ctx, r.PathValue("id"))
    // ...
}
```

## 3) Concurrency Patterns

### Bounded Worker Pool

```go
func NewWorkerPool(size int) *WorkerPool {
    return &WorkerPool{
        sem: make(chan struct{}, size), // Semaphore limits concurrency
    }
}

type WorkerPool struct {
    sem chan struct{}
    wg  sync.WaitGroup
}

func (p *WorkerPool) Submit(ctx context.Context, fn func()) error {
    select {
    case p.sem <- struct{}{}: // Acquire slot
    case <-ctx.Done():
        return ctx.Err() // Respect cancellation
    }

    p.wg.Add(1)
    go func() {
        defer func() {
            <-p.sem  // Release slot
            p.wg.Done()
        }()
        fn()
    }()
    return nil
}

func (p *WorkerPool) Wait() {
    p.wg.Wait()
}
```

### Fan-Out / Fan-In with Context

```go
func fanOut(ctx context.Context, work <-chan Job, workers int) <-chan Result {
    results := make(chan Result, workers)
    var wg sync.WaitGroup

    for range workers {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case job, ok := <-work:
                    if !ok {
                        return
                    }
                    result, err := process(ctx, job)
                    select {
                    case results <- Result{Data: result, Err: err}:
                    case <-ctx.Done():
                        return
                    }
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

### Goroutine Leak Prevention

```go
// ALWAYS ensure goroutines have an exit path
go func() {
    for {
        select {
        case msg := <-ch:
            handle(msg)
        case <-ctx.Done(): // Required: exit when context cancelled
            return
        case <-stop:       // Or a stop channel
            return
        }
    }
}()

// Run tests with race detector to catch leaks early
// go test -race ./...
```

## 4) Error Handling

```go
// Sentinel errors for expected conditions
var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrDuplicateKey  = errors.New("duplicate key")
)

// Wrap errors to preserve stack context
func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    user, err := r.db.QueryRowContext(ctx, query, id).Scan(...)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
    }
    if err != nil {
        return nil, fmt.Errorf("find user %s: %w", id, err)
    }
    return user, nil
}

// Caller unwraps
user, err := repo.FindByID(ctx, id)
if errors.Is(err, ErrNotFound) {
    http.Error(w, "user not found", http.StatusNotFound)
    return
}
if err != nil {
    log.Error("unexpected error", "err", err)
    http.Error(w, "internal error", http.StatusInternalServerError)
    return
}
```

## 5) Interface Design

```go
// Small, focused interfaces (Go proverb: the bigger the interface, the weaker the abstraction)
type UserStorer interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

type UserCreator interface {
    Create(ctx context.Context, user *User) error
}

// Compose when needed
type UserRepository interface {
    UserStorer
    UserCreator
}

// Accept interfaces, return concrete types
func NewUserService(repo UserRepository, cache UserStorer) *UserService {
    return &UserService{repo: repo, cache: cache}
}
```

## 6) SDK Integration Patterns

```go
// Wrap third-party SDKs behind interfaces for testability
type StorageClient interface {
    Upload(ctx context.Context, key string, data io.Reader) error
    Download(ctx context.Context, key string) (io.ReadCloser, error)
    Delete(ctx context.Context, key string) error
}

// Real implementation
type S3Client struct {
    client *s3.Client
    bucket string
}

func (c *S3Client) Upload(ctx context.Context, key string, data io.Reader) error {
    _, err := c.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket: &c.bucket,
        Key:    &key,
        Body:   data,
    })
    return fmt.Errorf("s3 upload %s: %w", key, err)
}

// Test mock — no AWS needed
type MockStorageClient struct {
    uploads map[string][]byte
}

func (m *MockStorageClient) Upload(_ context.Context, key string, data io.Reader) error {
    b, _ := io.ReadAll(data)
    m.uploads[key] = b
    return nil
}
```

### Retry with Exponential Backoff

```go
func withRetry(ctx context.Context, maxAttempts int, fn func() error) error {
    var err error
    for attempt := range maxAttempts {
        if attempt > 0 {
            wait := time.Duration(math.Pow(2, float64(attempt-1))) * 100 * time.Millisecond
            select {
            case <-time.After(wait):
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        if err = fn(); err == nil {
            return nil
        }
    }
    return fmt.Errorf("after %d attempts: %w", maxAttempts, err)
}
```

## 7) Testing Patterns

### Table-Driven Tests

```go
func TestUserService_GetUser(t *testing.T) {
    tests := []struct {
        name    string
        userID  string
        setup   func(*MockUserRepo)
        want    *User
        wantErr error
    }{
        {
            name:   "returns user when found",
            userID: "123",
            setup: func(r *MockUserRepo) {
                r.On("FindByID", mock.Anything, "123").Return(&User{ID: "123"}, nil)
            },
            want: &User{ID: "123"},
        },
        {
            name:   "wraps not found error",
            userID: "999",
            setup: func(r *MockUserRepo) {
                r.On("FindByID", mock.Anything, "999").Return(nil, ErrNotFound)
            },
            wantErr: ErrNotFound,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := &MockUserRepo{}
            tt.setup(repo)
            svc := NewUserService(repo)

            got, err := svc.GetUser(context.Background(), tt.userID)

            if tt.wantErr != nil {
                require.ErrorIs(t, err, tt.wantErr)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

### Testing Concurrent Code

```go
// Use -race flag always
// go test -race ./...

// Test for goroutine leaks with goleak
func TestWorkerPool_NoLeaks(t *testing.T) {
    defer goleak.VerifyNone(t)

    pool := NewWorkerPool(5)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    for range 100 {
        pool.Submit(ctx, func() { time.Sleep(10 * time.Millisecond) })
    }
    pool.Wait()
}

// Use testing/synctest for async time-based tests (Go 1.25+)
func TestDeadlineExpiry(t *testing.T) {
    synctest.Run(func() {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        done := make(chan struct{})
        go func() {
            <-ctx.Done()
            close(done)
        }()

        synctest.Wait() // Fast-forward virtual time
        select {
        case <-done: // Should have fired
        default:
            t.Fatal("context did not expire")
        }
    })
}
```

## 8) Code Review Checklist

- [ ] Every goroutine has a clear termination path
- [ ] Context is accepted and propagated through all I/O calls
- [ ] Errors are wrapped with `fmt.Errorf("context: %w", err)`, not swallowed
- [ ] No data races — run `go test -race ./...`
- [ ] Interfaces are small and focused
- [ ] Third-party SDKs are wrapped behind interfaces
- [ ] `defer` used correctly (not in tight loops)
- [ ] No global mutable state without synchronization
- [ ] `sync.WaitGroup.Add()` called before goroutine starts
- [ ] Channel directions specified in function signatures (`chan<-`, `<-chan`)
- [ ] `close()` only called by the producer, never the consumer

## 9) Observability

```go
// Structured logging with slog (Go 1.21+)
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

logger.InfoContext(ctx, "user created",
    slog.String("user_id", user.ID),
    slog.String("email", user.Email),
    slog.Duration("latency", time.Since(start)),
)

// Always pass logger via context or dependency injection
// Never use global logger in library code
```

## 10) Go Quality Commands

```bash
make lint          # golangci-lint run
make test          # go test ./...
make test-race     # go test -race ./...
make test-cover    # go test -coverprofile=coverage.out ./...
make vet           # go vet ./...
make build         # go build ./cmd/server
```

golangci-lint config (`.golangci.yml`):
```yaml
linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused
    - gofmt
    - goimports
    - gosec
    - exhaustive
    - noctx
```

## References

- [Go Concurrency Patterns 2026](https://reintech.io/blog/go-concurrency-patterns-2026-modern-approaches-parallel-programming)
- [testing/synctest (Go 1.25)](https://go.dev/blog/synctest)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Standard Project Layout](https://github.com/golang-standards/project-layout)
