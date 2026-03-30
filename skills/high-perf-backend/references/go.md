# Go: High-Performance Backend Reference

## Table of Contents
1. [Framework Selection](#framework-selection)
2. [Project Structure](#project-structure)
3. [HTTP Server Patterns](#http-server)
4. [gRPC Services](#grpc)
5. [Database Access](#database-access)
6. [Concurrency Patterns](#concurrency)
7. [Caching](#caching)
8. [Error Handling](#error-handling)
9. [Observability](#observability)
10. [Testing](#testing)
11. [Performance Tuning](#performance-tuning)
12. [Containerization](#containerization)

---

## Framework Selection {#framework-selection}

| Option | Style | Best For |
|---|---|---|
| **stdlib net/http** | Minimal, Go 1.22+ routing | Simple services, max control |
| **Gin** | Express-like, middleware chain | REST APIs, rapid development |
| **Echo** | Similar to Gin, slightly different API | REST APIs |
| **Chi** | stdlib-compatible router | When you want stdlib + better routing |
| **Connect / gRPC-Go** | RPC-first | Internal service-to-service |

**Recommendation**: For REST APIs, use **Gin** or **stdlib** (Go 1.22+ has built-in method routing).
For internal microservice communication, use **gRPC** with **Connect** for browser compatibility.

---

## Project Structure {#project-structure}

### Standard Layout

```
my-service/
├── cmd/
│   └── server/
│       └── main.go              # Entry point, wire dependencies
├── internal/                    # Private application code
│   ├── config/
│   │   └── config.go            # Configuration loading
│   ├── domain/                  # Business types and interfaces
│   │   ├── user.go              # Domain model
│   │   └── repository.go        # Repository interfaces (ports)
│   ├── handler/                 # HTTP/gRPC handlers (adapters)
│   │   ├── user_handler.go
│   │   └── middleware.go
│   ├── service/                 # Business logic (use cases)
│   │   └── user_service.go
│   ├── repo/                    # Database implementations (adapters)
│   │   └── postgres/
│   │       └── user_repo.go
│   └── platform/                # Infrastructure concerns
│       ├── database.go          # DB connection setup
│       ├── cache.go             # Redis/cache setup
│       └── telemetry.go         # Observability setup
├── api/                         # API definitions
│   ├── openapi/                 # OpenAPI specs
│   └── proto/                   # Protobuf definitions
├── migrations/                  # SQL migrations
├── deploy/
│   ├── Dockerfile
│   └── k8s/
├── go.mod
├── go.sum
└── Makefile
```

Key conventions:
- `internal/` makes packages truly private to this module — other Go modules cannot import them
- `domain/` contains interfaces, `repo/` and `handler/` implement them (Dependency Inversion)
- `cmd/` wires everything together, `internal/` has no global state

### Dependency Injection (Manual, No Framework)

```go
// cmd/server/main.go
func main() {
    cfg := config.Load()

    // Infrastructure
    db := platform.NewPostgresPool(cfg.Database)
    defer db.Close()
    cache := platform.NewRedisClient(cfg.Redis)
    defer cache.Close()

    // Repositories
    userRepo := postgres.NewUserRepo(db)

    // Services
    userService := service.NewUserService(userRepo, cache)

    // Handlers
    userHandler := handler.NewUserHandler(userService)

    // Router
    router := handler.NewRouter(userHandler, cfg)

    // Server
    srv := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      router,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Graceful shutdown
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal("Server failed", "error", err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}
```

---

## HTTP Server Patterns {#http-server}

### Gin-Based Server

```go
package handler

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

func NewRouter(userHandler *UserHandler, cfg *config.Config) *gin.Engine {
    if cfg.Env == "production" {
        gin.SetMode(gin.ReleaseMode)
    }

    r := gin.New()

    // Global middleware
    r.Use(gin.Recovery())
    r.Use(RequestIDMiddleware())
    r.Use(LoggingMiddleware())
    r.Use(CORSMiddleware())
    r.Use(TimeoutMiddleware(30 * time.Second))

    // Health checks (no auth)
    r.GET("/health/live", func(c *gin.Context) { c.Status(http.StatusOK) })
    r.GET("/health/ready", readinessCheck(db))

    // API routes
    v1 := r.Group("/api/v1")
    v1.Use(AuthMiddleware())
    {
        users := v1.Group("/users")
        {
            users.GET("", userHandler.List)
            users.POST("", userHandler.Create)
            users.GET("/:id", userHandler.GetByID)
            users.PUT("/:id", userHandler.Update)
            users.DELETE("/:id", userHandler.Delete)
        }
    }

    return r
}
```

### Handler Pattern

```go
type UserHandler struct {
    service *service.UserService
}

func NewUserHandler(s *service.UserService) *UserHandler {
    return &UserHandler{service: s}
}

type CreateUserRequest struct {
    Email string `json:"email" binding:"required,email"`
    Name  string `json:"name" binding:"required,min=2,max=100"`
}

type PaginationParams struct {
    Cursor string `form:"cursor"`
    Limit  int    `form:"limit,default=20" binding:"max=100"`
}

func (h *UserHandler) Create(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, ErrorResponse{
            Code:    "VALIDATION_ERROR",
            Message: err.Error(),
        })
        return
    }

    user, err := h.service.CreateUser(c.Request.Context(), req.Email, req.Name)
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, user)
}

func (h *UserHandler) List(c *gin.Context) {
    var params PaginationParams
    if err := c.ShouldBindQuery(&params); err != nil {
        c.JSON(http.StatusBadRequest, ErrorResponse{Code: "VALIDATION_ERROR", Message: err.Error()})
        return
    }

    users, nextCursor, err := h.service.ListUsers(c.Request.Context(), params.Cursor, params.Limit)
    if err != nil {
        handleError(c, err)
        return
    }

    c.JSON(http.StatusOK, PaginatedResponse{
        Data:       users,
        NextCursor: nextCursor,
    })
}
```

### Middleware Examples

```go
// Request ID middleware
func RequestIDMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }
        c.Set("request_id", requestID)
        c.Header("X-Request-ID", requestID)
        c.Next()
    }
}

// Timeout middleware
func TimeoutMiddleware(timeout time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), timeout)
        defer cancel()
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}

// Rate limiter middleware (token bucket with golang.org/x/time/rate)
func RateLimitMiddleware(rps int, burst int) gin.HandlerFunc {
    limiter := rate.NewLimiter(rate.Limit(rps), burst)
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, ErrorResponse{
                Code:    "RATE_LIMITED",
                Message: "Too many requests",
            })
            return
        }
        c.Next()
    }
}
```

---

## gRPC Services {#grpc}

### Protobuf Definition

```protobuf
// api/proto/user/v1/user.proto
syntax = "proto3";
package user.v1;
option go_package = "myservice/gen/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}

message GetUserRequest { string id = 1; }
message GetUserResponse { User user = 1; }
message User {
  string id = 1;
  string email = 2;
  string name = 3;
  google.protobuf.Timestamp created_at = 4;
}
```

### gRPC Server with Interceptors

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/keepalive"
)

func newGRPCServer(userSvc *service.UserService) *grpc.Server {
    srv := grpc.NewServer(
        grpc.KeepaliveParams(keepalive.ServerParameters{
            MaxConnectionIdle: 5 * time.Minute,
            Time:              2 * time.Hour,
            Timeout:           20 * time.Second,
        }),
        grpc.ChainUnaryInterceptor(
            grpc_recovery.UnaryServerInterceptor(),
            otelgrpc.UnaryServerInterceptor(),      // OpenTelemetry tracing
            grpc_zap.UnaryServerInterceptor(logger), // Structured logging
            rateLimitInterceptor(),
        ),
        grpc.MaxRecvMsgSize(10 * 1024 * 1024), // 10MB max message
    )

    userv1.RegisterUserServiceServer(srv, NewUserGRPCHandler(userSvc))
    return srv
}
```

---

## Database Access {#database-access}

### Connection Pool with pgx

```go
import "github.com/jackc/pgx/v5/pgxpool"

func NewPostgresPool(cfg DatabaseConfig) *pgxpool.Pool {
    config, err := pgxpool.ParseConfig(cfg.URL)
    if err != nil {
        log.Fatal("Failed to parse database URL", "error", err)
    }

    config.MaxConns = int32(cfg.MaxConns)       // Default: 4 * num_cpu
    config.MinConns = int32(cfg.MinConns)        // Keep some connections warm
    config.MaxConnLifetime = 30 * time.Minute
    config.MaxConnIdleTime = 5 * time.Minute
    config.HealthCheckPeriod = 1 * time.Minute

    pool, err := pgxpool.NewWithConfig(context.Background(), config)
    if err != nil {
        log.Fatal("Failed to create pool", "error", err)
    }
    return pool
}
```

### Repository Pattern

```go
type UserRepo struct {
    pool *pgxpool.Pool
}

func NewUserRepo(pool *pgxpool.Pool) *UserRepo {
    return &UserRepo{pool: pool}
}

func (r *UserRepo) FindByID(ctx context.Context, id uuid.UUID) (*domain.User, error) {
    var user domain.User
    err := r.pool.QueryRow(ctx,
        `SELECT id, email, name, created_at FROM users WHERE id = $1`, id,
    ).Scan(&user.ID, &user.Email, &user.Name, &user.CreatedAt)

    if errors.Is(err, pgx.ErrNoRows) {
        return nil, domain.ErrNotFound
    }
    return &user, err
}

// Cursor-based pagination
func (r *UserRepo) List(ctx context.Context, cursor *uuid.UUID, limit int) ([]domain.User, error) {
    var rows pgx.Rows
    var err error

    if cursor != nil {
        rows, err = r.pool.Query(ctx,
            `SELECT id, email, name, created_at FROM users
             WHERE id > $1 ORDER BY id ASC LIMIT $2`, *cursor, limit)
    } else {
        rows, err = r.pool.Query(ctx,
            `SELECT id, email, name, created_at FROM users ORDER BY id ASC LIMIT $1`, limit)
    }
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    users := make([]domain.User, 0, limit)
    for rows.Next() {
        var u domain.User
        if err := rows.Scan(&u.ID, &u.Email, &u.Name, &u.CreatedAt); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()
}

// Batch insert with COPY protocol (fastest for bulk data)
func (r *UserRepo) BulkInsert(ctx context.Context, users []domain.NewUser) (int64, error) {
    copyCount, err := r.pool.CopyFrom(
        ctx,
        pgx.Identifier{"users"},
        []string{"id", "email", "name"},
        pgx.CopyFromSlice(len(users), func(i int) ([]any, error) {
            return []any{users[i].ID, users[i].Email, users[i].Name}, nil
        }),
    )
    return copyCount, err
}
```

---

## Concurrency Patterns {#concurrency}

### Structured Concurrency with errgroup

```go
import "golang.org/x/sync/errgroup"

func (s *DashboardService) GetDashboard(ctx context.Context, userID uuid.UUID) (*Dashboard, error) {
    g, ctx := errgroup.WithContext(ctx)

    var user *User
    var orders []Order
    var notifications []Notification

    g.Go(func() error {
        var err error
        user, err = s.userRepo.FindByID(ctx, userID)
        return err
    })
    g.Go(func() error {
        var err error
        orders, err = s.orderRepo.FindByUserID(ctx, userID)
        return err
    })
    g.Go(func() error {
        var err error
        notifications, err = s.notifRepo.FindByUserID(ctx, userID)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }

    return &Dashboard{User: user, Orders: orders, Notifications: notifications}, nil
}
```

### Singleflight (Cache Stampede Prevention)

```go
import "golang.org/x/sync/singleflight"

type UserService struct {
    repo   domain.UserRepository
    cache  *redis.Client
    group  singleflight.Group
}

func (s *UserService) GetUser(ctx context.Context, id uuid.UUID) (*domain.User, error) {
    key := "user:" + id.String()

    // Try cache first
    cached, err := s.cache.Get(ctx, key).Result()
    if err == nil {
        var user domain.User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // Singleflight: only one goroutine fetches from DB, others wait for its result
    result, err, _ := s.group.Do(key, func() (any, error) {
        user, err := s.repo.FindByID(ctx, id)
        if err != nil {
            return nil, err
        }
        // Warm cache
        data, _ := json.Marshal(user)
        s.cache.Set(ctx, key, data, 5*time.Minute)
        return user, nil
    })
    if err != nil {
        return nil, err
    }
    return result.(*domain.User), nil
}
```

### Worker Pool

```go
func ProcessBatch(ctx context.Context, items []Item, concurrency int) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(concurrency) // Limit concurrent goroutines

    for _, item := range items {
        item := item // capture loop variable (not needed in Go 1.22+)
        g.Go(func() error {
            return processItem(ctx, item)
        })
    }

    return g.Wait()
}
```

---

## Error Handling {#error-handling}

### Domain Errors + HTTP Mapping

```go
// internal/domain/errors.go
var (
    ErrNotFound     = errors.New("resource not found")
    ErrConflict     = errors.New("resource already exists")
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
)

type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

// internal/handler/errors.go
type ErrorResponse struct {
    Code      string `json:"code"`
    Message   string `json:"message"`
    RequestID string `json:"request_id,omitempty"`
}

func handleError(c *gin.Context, err error) {
    requestID, _ := c.Get("request_id")

    switch {
    case errors.Is(err, domain.ErrNotFound):
        c.JSON(http.StatusNotFound, ErrorResponse{
            Code: "NOT_FOUND", Message: err.Error(), RequestID: requestID.(string),
        })
    case errors.Is(err, domain.ErrConflict):
        c.JSON(http.StatusConflict, ErrorResponse{
            Code: "CONFLICT", Message: err.Error(), RequestID: requestID.(string),
        })
    case errors.Is(err, domain.ErrUnauthorized):
        c.JSON(http.StatusUnauthorized, ErrorResponse{
            Code: "UNAUTHORIZED", Message: err.Error(), RequestID: requestID.(string),
        })
    default:
        slog.Error("Internal error", "error", err, "request_id", requestID)
        c.JSON(http.StatusInternalServerError, ErrorResponse{
            Code: "INTERNAL_ERROR", Message: "An internal error occurred", RequestID: requestID.(string),
        })
    }
}
```

---

## Observability {#observability}

### Structured Logging with slog (Go 1.21+)

```go
func SetupLogger(env string) *slog.Logger {
    var handler slog.Handler
    if env == "production" {
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelInfo,
        })
    } else {
        handler = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelDebug,
        })
    }
    logger := slog.New(handler)
    slog.SetDefault(logger)
    return logger
}

// Usage: inject request context into logs
func LoggingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        slog.Info("Request completed",
            "method", c.Request.Method,
            "path", c.Request.URL.Path,
            "status", c.Writer.Status(),
            "duration_ms", time.Since(start).Milliseconds(),
            "request_id", c.GetString("request_id"),
        )
    }
}
```

### Prometheus Metrics

```go
import "github.com/prometheus/client_golang/prometheus"

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration)
}

func MetricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start).Seconds()

        httpRequestsTotal.WithLabelValues(
            c.Request.Method, c.FullPath(), strconv.Itoa(c.Writer.Status()),
        ).Inc()
        httpRequestDuration.WithLabelValues(
            c.Request.Method, c.FullPath(),
        ).Observe(duration)
    }
}
```

---

## Testing {#testing}

### Table-Driven Tests

```go
func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        name_   string
        wantErr error
    }{
        {"valid user", "test@example.com", "Test User", nil},
        {"duplicate email", "existing@example.com", "Duplicate", domain.ErrConflict},
        {"invalid email", "not-an-email", "Bad Email", domain.ErrValidation},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := setupTestService(t)
            _, err := svc.CreateUser(context.Background(), tt.email, tt.name_)
            if !errors.Is(err, tt.wantErr) {
                t.Errorf("CreateUser() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### Integration Tests with testcontainers-go

```go
import "github.com/testcontainers/testcontainers-go/modules/postgres"

func setupTestDB(t *testing.T) *pgxpool.Pool {
    t.Helper()
    ctx := context.Background()

    pgContainer, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    require.NoError(t, err)
    t.Cleanup(func() { pgContainer.Terminate(ctx) })

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    pool, err := pgxpool.New(ctx, connStr)
    require.NoError(t, err)

    // Run migrations
    runMigrations(t, pool)
    return pool
}
```

---

## Performance Tuning {#performance-tuning}

### GOGC and Memory Tuning

```bash
# Reduce GC frequency for throughput (default 100)
GOGC=200 ./my-service

# Set soft memory limit (Go 1.19+) — better than GOGC for containers
GOMEMLIMIT=512MiB ./my-service
```

### Build Flags

```bash
# Production build
CGO_ENABLED=0 go build -ldflags="-s -w" -o my-service ./cmd/server/

# -s: strip symbol table
# -w: strip DWARF debugging info
# CGO_ENABLED=0: fully static binary
```

### Profiling with pprof

```go
import _ "net/http/pprof"

// In main(), expose on a separate port (not the public API port)
go func() {
    http.ListenAndServe("localhost:6060", nil)
}()
```

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine dump
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

### Common Go Performance Pitfalls
- Pre-allocate slices when size is known: `make([]T, 0, expectedLen)`
- Use `sync.Pool` for frequently allocated/freed objects (e.g., buffers)
- Avoid string concatenation in loops — use `strings.Builder`
- Use `[]byte` instead of `string` for hot-path processing
- Profile before optimizing — `go test -bench` + `benchstat` for comparison

---

## Containerization {#containerization}

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache git
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server/

# Runtime stage
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

Final image: ~10-20MB. Using `scratch` as the base provides the smallest possible attack surface.
For services that need TLS, DNS resolution, or a shell for debugging, use `alpine` or `distroless` instead.
