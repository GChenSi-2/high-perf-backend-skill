# 🚀 High-Performance Backend Skill

A Claude Code skill for building production-grade, high-performance backend services in **Java**, **Rust**, and **Go**.

Includes a **40-scenario tech stack decision matrix** that maps real-world business scenarios to battle-tested technology combinations — so you get architecture recommendations, not just code snippets.

## ✨ Features

### Interactive Tech Stack Advisor
Instead of jumping straight to code, this skill first asks about your business scenario, scale requirements, and constraints, then searches a curated decision matrix to recommend the right technology combination.

### 40+ Real-World Scenario Templates
Each template is a complete architecture blueprint covering:

| Dimension | What's Included |
|-----------|----------------|
| Language & Framework | Java/Spring Boot 3, Rust/Axum, Go/Gin, and more |
| Communication | REST, gRPC, WebSocket, MQTT, Kafka, NATS |
| Databases | PostgreSQL, Redis, ClickHouse, TimescaleDB, Neo4j, etc. |
| Caching | L1 in-process + L2 distributed strategies |
| Architecture | Hexagonal, CQRS, Event Sourcing, Saga, Pipeline |
| Resilience | Circuit Breaker, Bulkhead, Backpressure, Retry |
| Observability | OpenTelemetry, Prometheus, Grafana |
| Deployment | Kubernetes, Edge, Bare-metal |

### Business Domains Covered

- 🛒 **E-commerce** — Product catalog, orders, payments, flash sales, recommendations, user accounts
- 💬 **Social** — Feed/timeline, instant messaging, user relationships
- 💰 **Fintech** — Market data, matching engine, risk control, clearing & settlement
- ☁️ **SaaS** — Multi-tenant API, billing & metering
- 📡 **IoT** — Device gateway, data pipeline, edge computing
- 🎮 **Gaming** — Lobby/matchmaking, battle server
- 🤖 **AI/ML** — Model serving gateway, feature engineering
- ⛓️ **Blockchain** — Node RPC gateway, on-chain indexer
- 🏢 **Enterprise** — Workflow engine, file storage, auth center
- 📊 **Monitoring** — Metrics collection, log aggregation
- 🔔 **Notification** — Multi-channel push service
- 🔍 **Search** — Full-text search platform
- 📋 **Compliance** — Audit logging

### Deep-Dive Reference Guides

Once the skill recommends a stack, it loads language-specific reference guides with production-ready patterns:

- **Java/Spring Boot 3** — Virtual Threads, Resilience4j, Spring Cloud, JPA optimization, GraalVM Native Image
- **Rust** — Axum patterns, Tokio runtime tuning, SQLx, moka cache, release profile optimization
- **Go** — Gin/stdlib, pgx connection pool, errgroup, singleflight, pprof profiling

## 📦 Installation

### Option A: As a Plugin (Claude Code CLI)

```bash
# Add as marketplace
/plugin marketplace add GChenSi-2/high-perf-backend-skill

# Install the skill
/plugin install high-perf-backend@silicon-backend-skills
```

### Option B: Manual Install (Claude Code CLI)

```bash
# Clone and copy to your skills directory
git clone https://github.com/GChenSi-2/high-perf-backend-skill.git
cp -r high-perf-backend-skill/skills/high-perf-backend ~/.claude/skills/
```

### Option C: Upload to Claude.ai

1. Download the `high-perf-backend.skill` file from [Releases](https://github.com/GChenSi-2/high-perf-backend-skill/releases)
2. Go to **Settings → Customize → Skills**
3. Click **"+"** and upload the `.skill` file

## 🎯 Usage

Just describe what you're building:

```
"I need to build a real-time market data feed service that handles 100K+ concurrent WebSocket connections with sub-10ms latency"
```

The skill will:
1. Ask clarifying questions about your constraints
2. Search the decision matrix for matching scenarios
3. Present a structured recommendation with trade-offs
4. Offer to generate production-ready scaffolding code

Or go straight to coding:

```
"Help me set up an Axum REST API with SQLx and PostgreSQL"
```

## 📁 Repository Structure

```
high-perf-backend-skill/
├── .claude-plugin/
│   └── plugin.json                 # Plugin manifest
├── skills/
│   └── high-perf-backend/
│       ├── SKILL.md                # Main skill (routing + universal principles)
│       └── references/
│           ├── tech-stack-matrix.csv   # 40-scenario decision matrix
│           ├── java-springboot.md      # Java/Spring Boot 3 deep-dive
│           ├── rust.md                 # Rust/Axum/Tokio deep-dive
│           └── go.md                   # Go/Gin/gRPC deep-dive
├── marketplace.json                # Marketplace catalog
├── LICENSE
└── README.md
```

## 🤝 Contributing

PRs welcome! You can contribute by:

- Adding new business scenarios to `tech-stack-matrix.csv`
- Improving language-specific reference guides
- Adding new language support (e.g., C#/.NET, Python/FastAPI, TypeScript/NestJS)
- Fixing inaccuracies or outdated library versions

## 📄 License

MIT — see [LICENSE](./LICENSE).
