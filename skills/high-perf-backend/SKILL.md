---
name: high-perf-backend
user_invocable: true
description: >
  High-performance backend development covering Java (Spring Boot 3, Microservices), Rust (Axum/Actix-web/Tokio),
  and Go (Gin, gRPC, stdlib). Use whenever the user builds, optimizes, or architects backend services or APIs in
  these languages. Triggers: "backend", "API server", "microservice", "high concurrency", "low latency",
  "Spring Boot", "Axum", "Actix", "Tokio", "Gin", "gRPC", "circuit breaker", "rate limiting", "connection pool",
  "cache strategy", "distributed system", "observability", "profiling", "benchmarking", or any backend
  design/review/optimization in Java, Rust, or Go. Also trigger for code review, DB schema design for high-load,
  CI/CD for microservices, or general "help me write a REST API" in these stacks.
---

# High-Performance Backend Service Development

This skill provides battle-tested patterns, architecture guidance, and implementation best practices
for building production-grade, high-performance backend services in **Java**, **Rust**, and **Go**.

It includes a **技术选型决策矩阵** (tech stack decision matrix) covering 40+ real-world business scenarios,
and an interactive workflow to help users land on the right technology combination for their specific needs.

---

## Workflow: How to Use This Skill

### Phase 1: Gather Context (Ask the User)

Before making any technology recommendation, collect the following information from the user.
Use clear, structured questions. If the user has already provided some of this info in their message,
skip those questions and confirm what you understood.

**Required questions** (ask all that are not already answered):

1. **业务场景 (Business Scenario)**
   - What kind of service are you building? (e.g., e-commerce order service, chat system, API gateway, IoT device gateway, trading engine)
   - What is the core business problem this service solves?

2. **规模与性能要求 (Scale & Performance)**
   - Expected QPS / concurrent connections? (low: <1K, medium: 1K-10K, high: 10K-100K, very-high: >100K)
   - Latency requirements? (high-tolerance: >500ms, medium: 100-500ms, low: 10-100ms, ultra-low: <10ms)
   - Data consistency model? (strong consistency / eventual consistency / mix)

3. **技术偏好与约束 (Tech Preferences & Constraints)**
   - Do you have a language preference? (Java / Rust / Go / open to suggestions)
   - Any existing infrastructure or tech stack constraints? (e.g., already using Kafka, must deploy on K8s, team only knows Java)
   - Is this a greenfield project or adding to an existing system?

4. **特殊需求 (Special Requirements)**
   - Real-time communication needs? (WebSocket, SSE, gRPC streaming)
   - Multi-tenancy requirements?
   - Compliance / audit trail needs?
   - Edge deployment or resource-constrained environments?

### Phase 2: Match Against the Tech Stack Matrix

After gathering user input, search the tech stack decision matrix CSV for matching scenarios.

**Step 1: Read the CSV**
Read `references/tech-stack-matrix.csv` and parse it.

**Step 2: Score and rank rows**
For each row in the CSV, calculate a relevance score based on how many user-provided keywords
appear in these columns: `scenario`, `tags`, `notes`, `scale`, `latency_req`, `consistency_req`, `language`.

Scoring rules:
- Exact tag match in `tags` column: +3 points
- Keyword found in `scenario` or `notes`: +2 points
- Match on `scale` level: +2 points
- Match on `latency_req`: +2 points
- Match on `consistency_req`: +1 point
- Match on `language` (if user specified): +3 points
- Partial keyword overlap: +1 point

**Step 3: Select top matches**
Take the top 1-3 rows by score. These are the **reference architectures** to present to the user.

### Phase 3: Generate the Recommendation

Based on the matched rows, synthesize a technology recommendation structured as follows:

```
## 推荐技术方案 (Recommended Tech Stack)

### 方案概述 (Overview)
- 参考场景: [matched scenario name(s)]
- 推荐语言/框架: [language + framework]
- 核心理由: [why this combination fits]

### 技术选型详情 (Stack Details)
| 层级 | 选择 | 理由 |
|------|------|------|
| 语言/框架 | ... | ... |
| 通信协议 | ... | ... |
| 主数据库 | ... | ... |
| 辅助存储 | ... | ... |
| 缓存方案 | ... | ... |
| 消息中间件 | ... | ... |
| 搜索引擎 | ... | ... |
| API风格 | ... | ... |
| 认证方案 | ... | ... |
| 可观测性 | ... | ... |
| 部署方案 | ... | ... |

### 架构模式 (Architecture Pattern)
[Describe the recommended architecture pattern and why]

### 弹性设计 (Resilience)
[Circuit breaker, retry, bulkhead, etc. specific to this scenario]

### 关键库/依赖 (Key Libraries)
[List with brief explanation of each]

### 潜在风险与权衡 (Risks & Trade-offs)
[Be honest about what you're trading off with this choice]

### 如果规模增长... (Scaling Path)
[How to evolve this architecture as load grows 10x]
```

### Phase 4: Deep Dive with Reference Files

After presenting the recommendation, offer to go deeper. Read the relevant reference file
based on the recommended language:

- Java + Spring Boot 3 + Microservices → read `references/java-springboot.md`
- Rust (Axum, Actix-web, Tokio) → read `references/rust.md`
- Go (Gin, stdlib, gRPC) → read `references/go.md`

Then provide:
- Production-ready code scaffolding for the recommended architecture
- Specific configuration templates (application.yml, Cargo.toml, etc.)
- Testing strategy tailored to the scenario
- Dockerfile and deployment manifests

---

## Handling Ambiguous or Multi-Language Requests

If the user's scenario maps to multiple languages (e.g., a polyglot microservice system), present
multiple matched rows and explain the trade-offs. For example, a trading platform might use:
- Rust for the matching engine (ultra-low latency)
- Go for the API gateway (high throughput, fast development)
- Java for the clearing/settlement service (complex business logic, strong framework support)

Read all relevant reference files and provide guidance for each component.

---

## Handling "I Just Want to Code" Requests

If the user skips the Q&A and says something like "help me write a REST API in Go" or
"set up a Spring Boot microservice", respect that:

1. Skip Phase 1-2, go directly to the reference file for the specified language
2. Read the relevant `references/*.md` file
3. Provide production-quality code following the patterns in that reference
4. Still apply the universal principles below

---

## Universal High-Performance Principles

These apply across all three languages. Language-specific implementation details are in the reference files.

### Architecture Patterns

**Service Decomposition**
- Decompose by business capability, not by technical layer
- Each service owns its data (Database-per-Service pattern)
- Prefer asynchronous communication (event-driven) over synchronous RPC for cross-service calls
  when eventual consistency is acceptable
- Use API Gateway as the single entry point for external clients

**Communication Patterns**
- Synchronous: REST (OpenAPI 3.x) for external APIs, gRPC (protobuf) for internal service-to-service
- Asynchronous: Event-driven via message brokers (Kafka, RabbitMQ, NATS)
- Use the Outbox Pattern to guarantee at-least-once delivery without distributed transactions
- Implement idempotency keys for all write operations

**Resilience**
- Circuit Breaker: prevent cascade failures (fail-fast when downstream is unhealthy)
- Bulkhead: isolate failure domains (separate thread pools / connection pools per dependency)
- Retry with exponential backoff + jitter for transient failures
- Timeout every external call — no unbounded waits
- Graceful degradation: serve partial/cached results when non-critical dependencies fail

### Data Layer

**Database Selection**
- OLTP with strong consistency → PostgreSQL (preferred), MySQL
- High-throughput key-value → Redis, DragonflyDB
- Document store → MongoDB (when schema flexibility outweighs join needs)
- Time-series → TimescaleDB, ClickHouse
- Search → Elasticsearch, Meilisearch

**Connection Pooling**
- Always use connection pools; never create connections per-request
- Size the pool based on: `pool_size ≈ (core_count * 2) + effective_spindle_count`
- Monitor pool saturation; alert when wait time exceeds P95 threshold

**Caching Strategy**
- L1 (in-process): small, hot data with short TTL — Caffeine (Java), moka (Rust), freecache (Go)
- L2 (distributed): Redis/Valkey with consistent hashing
- Cache invalidation: prefer event-driven invalidation over TTL-only
- Protect against cache stampede with request coalescing (singleflight in Go, similar elsewhere)

### Observability (The Three Pillars)

- **Logging**: JSON-formatted, correlation IDs, never log PII
- **Metrics**: RED (Rate/Errors/Duration) for services, USE (Utilization/Saturation/Errors) for resources
- **Tracing**: OpenTelemetry SDK, W3C Trace Context propagation, strategic sampling

### Performance Engineering

**Common Bottlenecks (check in this order)**
1. N+1 queries / missing indexes → check query plans
2. Serialization overhead → consider binary formats (protobuf, MessagePack)
3. Connection pool exhaustion → monitor wait times
4. GC pressure (Java/Go) → tune GC, reduce allocations
5. Lock contention → use lock-free data structures or sharding
6. Inefficient I/O → batch operations, use async I/O

**Concurrency Models**
- Java: Virtual Threads (Project Loom, JDK 21+) for I/O-bound; ForkJoinPool for CPU-bound
- Rust: Tokio async runtime for I/O-bound; Rayon for CPU-bound parallelism
- Go: Goroutines + channels; errgroup for structured concurrency

### API Design

- OpenAPI 3.1 spec as the contract for REST APIs
- Cursor-based pagination (not offset) for large datasets
- Token bucket or sliding window rate limiting; return `429` with `Retry-After`
- Consistent error response format with error code, message, details, and request_id

### Security Baseline

- AuthN: JWT (short-lived) or OAuth2/OIDC; AuthZ: RBAC or ABAC at middleware
- TLS everywhere, mTLS for internal service-to-service
- Input validation on every boundary; dependency scanning in CI

### Deployment

- Container-first: multi-stage Dockerfiles for minimal images
- Health endpoints: `/health/live` + `/health/ready`
- Graceful shutdown: drain in-flight requests before termination
- 12-Factor App configuration principles

### Language Selection Quick Reference

| Criteria | Java + Spring Boot 3 | Rust | Go |
|---|---|---|---|
| Best for | Complex business logic, enterprise | Ultra-low-latency, safety-critical | Infra, cloud-native, network services |
| Dev speed | Fast (rich frameworks) | Slower (borrow checker) | Fast (simple language) |
| Memory | Higher (JVM) | Lowest | Low |
| Latency | Good (Virtual Threads) | Best | Very good |
| Ecosystem | Massive | Growing fast | Strong for infra/cloud |
