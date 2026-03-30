# Rust: High-Performance Backend Reference

## Table of Contents
1. [Framework Selection](#framework-selection)
2. [Project Structure](#project-structure)
3. [Async Runtime & Concurrency](#async-runtime)
4. [Axum Patterns](#axum-patterns)
5. [Database Access (SQLx)](#database-access)
6. [Error Handling](#error-handling)
7. [Caching](#caching)
8. [Observability](#observability)
9. [Testing](#testing)
10. [Performance Tuning](#performance-tuning)
11. [Containerization](#containerization)

---

## Framework Selection {#framework-selection}

| Framework | Style | Best For |
|---|---|---|
| **Axum** | Tower-based, modular, ergonomic | Most new projects (recommended default) |
| **Actix-web** | Actor model, mature | CPU-heavy workloads, existing projects |
| **Tonic** | gRPC-first | Internal microservice communication |

**Recommendation**: Start with Axum unless you have a specific reason for Actix-web. Axum's
Tower middleware ecosystem is more composable, and it integrates naturally with the Tokio stack.

---

## Project Structure {#project-structure}

### Workspace Layout (Multi-Service)

```
my-platform/
├── Cargo.toml                  # Workspace root
├── crates/
│   ├── common/                 # Shared types, error handling, utils
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── error.rs        # Common error types
│   │       ├── config.rs       # Config loading (env + file)
│   │       └── telemetry.rs    # Tracing/metrics setup
│   ├── user-service/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── lib.rs          # For integration test access
│   │       ├── routes/         # HTTP handlers
│   │       │   ├── mod.rs
│   │       │   └── users.rs
│   │       ├── domain/         # Business logic & types
│   │       │   ├── mod.rs
│   │       │   ├── model.rs
│   │       │   └── service.rs
│   │       ├── repo/           # Database layer
│   │       │   ├── mod.rs
│   │       │   └── postgres.rs
│   │       └── config.rs
│   └── order-service/
├── migrations/                 # SQLx migrations
├── proto/                      # Protobuf definitions (if using gRPC)
└── deploy/
    ├── Dockerfile
    └── k8s/
```

### Key Cargo.toml Dependencies

```toml
[dependencies]
# Web framework
axum = { version = "0.7", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["trace", "cors", "compression-gzip", "timeout", "limit"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Database
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "chrono", "uuid", "migrate"] }

# Configuration
config = "0.14"
dotenvy = "0.15"

# Observability
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-opentelemetry = "0.24"
opentelemetry = { version = "0.23", features = ["trace"] }
opentelemetry-otlp = "0.16"
metrics = "0.23"
metrics-exporter-prometheus = "0.15"

# Error handling
thiserror = "1"
anyhow = "1"

# Validation
validator = { version = "0.18", features = ["derive"] }

# Async utilities
futures = "0.3"
tokio-util = "0.7"
```

---

## Async Runtime & Concurrency {#async-runtime}

### Tokio Runtime Configuration

```rust
#[tokio::main]
async fn main() {
    // Default: multi-thread runtime with worker threads = CPU cores
    // For fine-tuning:
    let runtime = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(num_cpus::get())     // I/O-bound: match core count
        .max_blocking_threads(512)           // For blocking file I/O, etc.
        .enable_all()
        .build()
        .expect("Failed to build Tokio runtime");

    runtime.block_on(start_server());
}
```

### Concurrency Patterns

```rust
use tokio::sync::{Semaphore, RwLock};
use std::sync::Arc;

// Rate-limit concurrent operations with a semaphore
struct AppState {
    db: PgPool,
    http_client: reqwest::Client,
    external_api_semaphore: Arc<Semaphore>,  // Limit concurrent external calls
    cache: Arc<RwLock<HashMap<String, CachedValue>>>,
}

async fn call_external_api(state: &AppState, request: ApiRequest) -> Result<ApiResponse> {
    let _permit = state.external_api_semaphore
        .acquire()
        .await
        .map_err(|_| AppError::ServiceUnavailable)?;

    // Only N concurrent calls can execute here
    state.http_client
        .post("https://external-api.com/endpoint")
        .json(&request)
        .timeout(Duration::from_secs(5))
        .send()
        .await?
        .json()
        .await
        .map_err(Into::into)
}
```

### CPU-Bound Work: Use Rayon or spawn_blocking

```rust
// Never do CPU-heavy work on async threads — it blocks the executor
async fn process_image(data: Bytes) -> Result<Bytes> {
    // Move CPU work to a blocking thread
    tokio::task::spawn_blocking(move || {
        // Rayon for parallel computation
        let result = rayon::scope(|s| {
            // parallel image processing
        });
        Ok(result)
    })
    .await?
}
```

---

## Axum Patterns {#axum-patterns}

### Application Bootstrap

```rust
use axum::{Router, middleware};
use tower_http::{
    trace::TraceLayer,
    cors::CorsLayer,
    compression::CompressionLayer,
    timeout::TimeoutLayer,
    limit::RequestBodyLimitLayer,
};
use std::time::Duration;

pub fn create_router(state: AppState) -> Router {
    let api_routes = Router::new()
        .nest("/users", user_routes())
        .nest("/orders", order_routes())
        .layer(middleware::from_fn_with_state(state.clone(), auth_middleware));

    Router::new()
        .nest("/api/v1", api_routes)
        .route("/health/live", get(|| async { StatusCode::OK }))
        .route("/health/ready", get(health_check))
        .route("/metrics", get(metrics_handler))
        .layer(TraceLayer::new_for_http())                    // Distributed tracing
        .layer(CompressionLayer::new())                       // Gzip responses
        .layer(TimeoutLayer::new(Duration::from_secs(30)))    // Global timeout
        .layer(RequestBodyLimitLayer::new(10 * 1024 * 1024))  // 10MB body limit
        .layer(CorsLayer::permissive())                       // Configure per-environment
        .with_state(state)
}

async fn start_server(router: Router) {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    // Graceful shutdown
    axum::serve(listener, router)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    let ctrl_c = tokio::signal::ctrl_c();
    let mut sigterm = tokio::signal::unix::signal(
        tokio::signal::unix::SignalKind::terminate()
    ).unwrap();

    tokio::select! {
        _ = ctrl_c => {},
        _ = sigterm.recv() => {},
    }
    tracing::info!("Shutdown signal received, draining connections...");
}
```

### Extractors & Handlers

```rust
use axum::{extract::{Path, Query, State, Json}, http::StatusCode};
use validator::Validate;

#[derive(Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(email)]
    pub email: String,
    #[validate(length(min = 2, max = 100))]
    pub name: String,
}

#[derive(Deserialize)]
pub struct PaginationParams {
    pub cursor: Option<String>,
    pub limit: Option<i64>,
}

async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<UserResponse>), AppError> {
    payload.validate().map_err(|e| AppError::Validation(e.to_string()))?;

    let user = state.user_service.create_user(payload).await?;
    Ok((StatusCode::CREATED, Json(user.into())))
}

async fn list_users(
    State(state): State<AppState>,
    Query(params): Query<PaginationParams>,
) -> Result<Json<PaginatedResponse<UserResponse>>, AppError> {
    let limit = params.limit.unwrap_or(20).min(100);
    let users = state.user_service.list_users(params.cursor, limit).await?;
    Ok(Json(users))
}
```

### Middleware Pattern

```rust
use axum::{extract::Request, middleware::Next, response::Response};

async fn auth_middleware(
    State(state): State<AppState>,
    mut request: Request,
    next: Next,
) -> Result<Response, AppError> {
    let token = request.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(AppError::Unauthorized)?;

    let claims = state.jwt_validator.validate(token).await?;
    request.extensions_mut().insert(claims);

    Ok(next.run(request).await)
}
```

---

## Database Access (SQLx) {#database-access}

### Connection Pool Setup

```rust
use sqlx::postgres::{PgPoolOptions, PgConnectOptions};

pub async fn create_pool(config: &DatabaseConfig) -> PgPool {
    let connect_options = PgConnectOptions::new()
        .host(&config.host)
        .port(config.port)
        .database(&config.database)
        .username(&config.username)
        .password(&config.password);

    PgPoolOptions::new()
        .max_connections(config.max_connections)  // Default: 10, tune based on load
        .min_connections(config.min_connections)   // Keep some warm
        .acquire_timeout(Duration::from_secs(5))   // Fail fast
        .idle_timeout(Duration::from_secs(300))
        .max_lifetime(Duration::from_secs(1800))
        .connect_with(connect_options)
        .await
        .expect("Failed to create database pool")
}
```

### Compile-Time Checked Queries

```rust
// Single record
pub async fn find_by_id(pool: &PgPool, id: Uuid) -> Result<Option<User>> {
    sqlx::query_as!(User, "SELECT id, email, name, created_at FROM users WHERE id = $1", id)
        .fetch_optional(pool)
        .await
        .map_err(Into::into)
}

// Cursor-based pagination
pub async fn list_users(pool: &PgPool, cursor: Option<Uuid>, limit: i64) -> Result<Vec<User>> {
    match cursor {
        Some(cursor_id) => {
            sqlx::query_as!(User,
                r#"SELECT id, email, name, created_at FROM users
                   WHERE id > $1 ORDER BY id ASC LIMIT $2"#,
                cursor_id, limit
            )
            .fetch_all(pool)
            .await
            .map_err(Into::into)
        }
        None => {
            sqlx::query_as!(User,
                "SELECT id, email, name, created_at FROM users ORDER BY id ASC LIMIT $1",
                limit
            )
            .fetch_all(pool)
            .await
            .map_err(Into::into)
        }
    }
}

// Batch insert with UNNEST (PostgreSQL)
pub async fn bulk_insert(pool: &PgPool, users: &[NewUser]) -> Result<()> {
    let ids: Vec<Uuid> = users.iter().map(|u| u.id).collect();
    let emails: Vec<&str> = users.iter().map(|u| u.email.as_str()).collect();
    let names: Vec<&str> = users.iter().map(|u| u.name.as_str()).collect();

    sqlx::query!(
        r#"INSERT INTO users (id, email, name)
           SELECT * FROM UNNEST($1::uuid[], $2::text[], $3::text[])"#,
        &ids, &emails as &[&str], &names as &[&str]
    )
    .execute(pool)
    .await?;
    Ok(())
}
```

---

## Error Handling {#error-handling}

### Unified Error Type

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Resource not found: {0}")]
    NotFound(String),

    #[error("Validation error: {0}")]
    Validation(String),

    #[error("Unauthorized")]
    Unauthorized,

    #[error("Forbidden")]
    Forbidden,

    #[error("Conflict: {0}")]
    Conflict(String),

    #[error("Service unavailable")]
    ServiceUnavailable,

    #[error(transparent)]
    Database(#[from] sqlx::Error),

    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::NotFound(_) => (StatusCode::NOT_FOUND, "RESOURCE_NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            AppError::Forbidden => (StatusCode::FORBIDDEN, "FORBIDDEN"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
            AppError::ServiceUnavailable => (StatusCode::SERVICE_UNAVAILABLE, "SERVICE_UNAVAILABLE"),
            AppError::Database(_) | AppError::Internal(_) => {
                tracing::error!(error = %self, "Internal server error");
                (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR")
            }
        };

        let body = serde_json::json!({
            "error": {
                "code": code,
                "message": self.to_string(),
            }
        });

        (status, Json(body)).into_response()
    }
}
```

---

## Caching {#caching}

### In-Process Cache with moka

```rust
use moka::future::Cache;

pub struct CacheLayer {
    user_cache: Cache<Uuid, UserDto>,
}

impl CacheLayer {
    pub fn new() -> Self {
        Self {
            user_cache: Cache::builder()
                .max_capacity(10_000)
                .time_to_live(Duration::from_secs(300))
                .time_to_idle(Duration::from_secs(60))
                .build(),
        }
    }

    pub async fn get_or_fetch_user<F, Fut>(&self, id: Uuid, fetch: F) -> Result<UserDto>
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<UserDto>>,
    {
        if let Some(cached) = self.user_cache.get(&id).await {
            return Ok(cached);
        }
        let user = fetch().await?;
        self.user_cache.insert(id, user.clone()).await;
        Ok(user)
    }
}
```

---

## Observability {#observability}

### Tracing Setup

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

pub fn init_telemetry(service_name: &str) {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info,sqlx=warn,tower_http=debug"));

    let fmt_layer = tracing_subscriber::fmt::layer()
        .json()
        .with_target(true)
        .with_thread_ids(true);

    // Optional: OpenTelemetry export
    let otel_layer = if let Ok(endpoint) = std::env::var("OTEL_EXPORTER_OTLP_ENDPOINT") {
        let tracer = opentelemetry_otlp::new_pipeline()
            .tracing()
            .with_exporter(opentelemetry_otlp::new_exporter().tonic().with_endpoint(endpoint))
            .install_batch(opentelemetry_sdk::runtime::Tokio)
            .expect("Failed to init OTel tracer");
        Some(tracing_opentelemetry::layer().with_tracer(tracer))
    } else {
        None
    };

    tracing_subscriber::registry()
        .with(env_filter)
        .with(fmt_layer)
        .with(otel_layer)
        .init();
}
```

---

## Testing {#testing}

```rust
// Integration test with a real database (using sqlx test fixtures)
#[cfg(test)]
mod tests {
    use super::*;
    use axum::http::{Request, StatusCode};
    use tower::ServiceExt; // for oneshot

    async fn setup_app() -> (Router, PgPool) {
        let pool = PgPoolOptions::new()
            .connect(&std::env::var("TEST_DATABASE_URL").unwrap())
            .await.unwrap();
        sqlx::migrate!().run(&pool).await.unwrap();
        let state = AppState::new(pool.clone());
        (create_router(state), pool)
    }

    #[tokio::test]
    async fn test_create_user() {
        let (app, _pool) = setup_app().await;

        let response = app
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/api/v1/users")
                    .header("Content-Type", "application/json")
                    .body(r#"{"email":"test@example.com","name":"Test"}"#.into())
                    .unwrap()
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::CREATED);
    }
}
```

---

## Performance Tuning {#performance-tuning}

### Compile-Time Optimizations

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = "fat"            # Link-time optimization — slower build, faster binary
codegen-units = 1      # Better optimization at cost of compile time
strip = true           # Strip debug symbols
panic = "abort"        # Smaller binary, no unwinding overhead

[profile.release.build-override]
opt-level = 3
```

### Runtime Tips
- Use `jemalloc` instead of system allocator for better multi-threaded performance:
  ```rust
  #[global_allocator]
  static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
  ```
- Prefer `Bytes` over `Vec<u8>` for zero-copy buffer sharing
- Use `Arc<str>` instead of `String` for frequently cloned immutable strings
- Profile with `cargo flamegraph` and `tokio-console` for async bottlenecks

---

## Containerization {#containerization}

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM rust:1.78-bookworm AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y musl-tools
RUN rustup target add x86_64-unknown-linux-musl

COPY Cargo.toml Cargo.lock ./
COPY crates/ crates/
RUN cargo build --release --target x86_64-unknown-linux-musl

# Runtime stage — scratch or distroless for minimal attack surface
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/my-service /
EXPOSE 8080
ENTRYPOINT ["/my-service"]
```

Final image is typically 5-15MB with statically linked musl binary — far smaller than JVM or Go images.
