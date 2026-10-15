# tokentoll

> Multi-tenant LLM gateway with semantic caching, token-based rate limiting, provider failover and AI-driven incident analysis.

![Java](https://img.shields.io/badge/Java-21-orange) ![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3-brightgreen) ![React](https://img.shields.io/badge/React-TypeScript-blue) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

**tokentoll** sits between your applications and LLM providers (OpenAI, Claude, Gemini, local models). It meters usage, controls cost, protects against abuse, keeps traffic flowing when a provider fails, and uses AI to explain incidents when something goes wrong.

> **Status:** 🚧 In active development. See the [Roadmap](#roadmap) for what is done and what is next.

---

## Table of Contents

- [Why tokentoll](#why-tokentoll)
- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [API Overview](#api-overview)
- [Admin Console](#admin-console)
- [Design Decisions and Trade-offs](#design-decisions-and-trade-offs)
- [Load Test Results](#load-test-results)
- [Testing](#testing)
- [Deployment](#deployment)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)
- [License](#license)
- [Author](#author)

---

## Why tokentoll

Teams that ship LLM features quickly run into the same problems:

| Problem | How tokentoll solves it |
|---|---|
| One provider outage breaks the product | Automatic failover with circuit breakers |
| LLM bills grow without warning | Semantic caching, per-tenant token budgets |
| One tenant can starve everyone else | Token-aware rate limiting |
| Sensitive data leaves the company | PII redaction and prompt-injection checks |
| Nobody knows why costs or errors spiked | Kafka-based anomaly detection with AI root-cause summaries |

---

## Features

**Gateway core**
- Unified REST API for multiple LLM providers, with streaming through Server-Sent Events (SSE)
- Smart routing and automatic failover using Resilience4j circuit breakers
- Semantic cache using PostgreSQL + pgvector: similar prompts return cached answers

**Cost and abuse control**
- Token-based rate limiting (not just requests per second) backed by Redis and atomic Lua scripts
- Monthly token and cost budgets per tenant, enforced at request time
- PII redaction (emails, phone numbers, card numbers) before data leaves your system
- Basic prompt-injection detection

**Incident intelligence**
- Usage events published to Kafka
- Real-time anomaly detection: cost spikes, latency jumps, error bursts, abusive tenants
- AI incident analyzer (Spring AI + RAG on pgvector): retrieves similar past incidents and generates a root-cause summary with a suggested fix
- Real-time alerts delivered to the UI over SSE

**Platform**
- Multi-tenancy with API keys, JWT authentication and role-based access control
- Audit log of administrative actions
- Evaluation module: replay a saved prompt set on two models and compare quality, cost and latency
- Metrics with Micrometer, Prometheus and Grafana

---

## Architecture

```
                         +---------------------+
   Client apps  ───────► |   tokentoll-api     | ───────►  LLM providers
 (API key / JWT)         |  (Spring Boot)      |           (OpenAI, Claude,
                         |                     |            Gemini, local)
                         |  auth → rate limit  |
                         |  → guardrails →     |
                         |  cache → router     |
                         +----+-------+--------+
                              |       |
                  +-----------+       +-------------+
                  v                                 v
            +-----------+                    +-------------+
            |   Redis   |                    |  PostgreSQL |
            | rate/limit|                    |  + pgvector |
            | budgets   |                    |  cache, RAG |
            +-----------+                    +------+------+
                                                    ^
                              usage events          |
   tokentoll-api ──────────► Kafka ──► tokentoll-analyzer
                                       (anomaly detection +
                                        AI incident analysis)
                                                    |
                                                    v  SSE alerts
                                          tokentoll-ui (React + TS)
```

**Request flow**
1. Client sends a request with an API key or JWT.
2. Auth and tenant resolution, then token-aware rate limit and budget check (Redis).
3. Guardrails: PII redaction and prompt-injection checks.
4. Semantic cache lookup (pgvector). On a hit, return immediately.
5. On a miss, route to a provider, with circuit breaker and failover.
6. Stream the response back over SSE and publish a usage event to Kafka.
7. The analyzer consumes events, detects anomalies, and generates AI incident summaries.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Java 21, Spring Boot 3, Spring Cloud Gateway / WebFlux, Spring Security, Spring AI |
| Resilience | Resilience4j |
| Messaging | Apache Kafka |
| Data | PostgreSQL, pgvector, Redis |
| Frontend | React 18, TypeScript, Vite, TanStack Query, Recharts, Tailwind CSS |
| Observability | Micrometer, Prometheus, Grafana |
| Testing | JUnit 5, Testcontainers, Vitest, k6 |
| Delivery | Docker, Docker Compose, Kubernetes, GitHub Actions |

---

## Getting Started

### Prerequisites
- Java 21
- Node.js 20+
- Docker and Docker Compose
- At least one LLM provider API key (or a local model endpoint)

### Run with Docker Compose

```bash
git clone <your-repo-url>
cd tokentoll

cp .env.example .env
# edit .env and add your provider API keys

docker compose up --build
```

Services:

| Service | URL |
|---|---|
| API | http://localhost:8080 |
| Admin console | http://localhost:3000 |
| Swagger / OpenAPI | http://localhost:8080/swagger-ui.html |
| Grafana | http://localhost:3001 |
| Prometheus | http://localhost:9090 |

### Run locally for development

```bash
# 1. start dependencies (Postgres, Redis, Kafka)
docker compose up -d postgres redis kafka

# 2. backend
cd backend
./mvnw spring-boot:run

# 3. frontend
cd frontend
npm install
npm run dev
```

---

## Configuration

Set these in `.env` (see `.env.example`):

| Variable | Description |
|---|---|
| `OPENAI_API_KEY` | OpenAI provider key (optional) |
| `ANTHROPIC_API_KEY` | Anthropic provider key (optional) |
| `GEMINI_API_KEY` | Google Gemini provider key (optional) |
| `JWT_SECRET` | Secret used to sign JWTs |
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | PostgreSQL connection |
| `REDIS_URL` | Redis connection |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers |
| `CACHE_SIMILARITY_THRESHOLD` | Minimum similarity for a semantic cache hit (for example `0.95`) |

Never commit real keys. `.env` is listed in `.gitignore`.

---

## API Overview

Full interactive docs are available at `/swagger-ui.html`.

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/login` | Obtain a JWT |
| POST | `/v1/chat/completions` | Send a prompt (supports `stream=true` via SSE) |
| GET | `/v1/usage` | Token and cost usage for the current tenant |
| POST | `/admin/keys` | Create an API key |
| PUT | `/admin/tenants/{id}/budget` | Set token limits and monthly budget |
| GET | `/admin/incidents` | List incidents with AI summaries |
| GET | `/admin/alerts/stream` | Live anomaly alerts (SSE) |
| POST | `/admin/evals` | Run a prompt set against two models |
| GET | `/admin/audit` | Audit log |

**Example request**

```bash
curl -N http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
        "model": "auto",
        "stream": true,
        "messages": [{"role": "user", "content": "Explain circuit breakers in one paragraph."}]
      }'
```

Response headers include `X-Cache: HIT|MISS`, `X-Provider`, and `X-Cost-USD`.

---

## Admin Console

React + TypeScript app with these pages:

| Page | Purpose |
|---|---|
| Login | JWT login with role-based UI |
| Dashboard | Cost per tenant, tokens, p95 latency, cache hit rate, error rate |
| Playground | Streamed replies, with cache hit or miss, cost, latency, and provider shown |
| Live Alerts | Real-time anomaly feed over SSE |
| Incidents | AI root-cause summary, suggested fix, similar past incidents |
| Keys and Budgets | Manage API keys, token limits and budgets |
| Eval Compare | Compare two models on the same prompts |
| Audit Log | Who did what, and when |

Types are generated from the backend OpenAPI spec to keep frontend and backend in sync:

```bash
npm run generate:api
```

---

## Design Decisions and Trade-offs

- **Why Kafka for usage events?** Decouples the request path from analytics, so a slow analyzer never slows user requests, and events can be replayed.
- **Why a sliding window or token bucket in Redis, and why Lua?** The check and the update must be atomic, otherwise concurrent requests can exceed the limit.
- **Semantic cache correctness.** A wrong cache hit returns a wrong answer. Hits require a similarity threshold, are scoped per tenant and per model, and expire with a TTL.
- **Reactive vs blocking.** Streaming responses hold connections open, so the gateway uses WebFlux (or virtual threads) to avoid exhausting threads.
- **JWT with SSE.** The browser's native `EventSource` cannot send an `Authorization` header, so the UI uses `fetch-event-source` (or a short-lived stream token).
- **False-positive control for alerts.** Anomaly rules use sliding windows and cooldowns to avoid alert storms.
- **Why pgvector instead of a dedicated vector DB?** One less system to run, and transactional consistency with the rest of the data.

---

## Load Test Results

Benchmarks will be published here after the core gateway is complete.

| Metric | Result |
|---|---|
| Sustained throughput | Coming soon |
| p95 latency (cache miss) | Coming soon |
| p95 latency (cache hit) | Coming soon |
| Cache hit rate | Coming soon |
| Simulated LLM cost reduction | Coming soon |
| Failover time after provider failure | Coming soon |
| Anomaly detection latency | Coming soon |

Load tests are written with k6 and will live in `loadtest/`:

```bash
k6 run loadtest/gateway.js
```

---

## Testing

```bash
# backend unit and integration tests (Testcontainers starts Postgres, Redis, Kafka)
cd backend && ./mvnw verify

# frontend tests
cd frontend && npm test
```

---

## Deployment

**Docker Compose** is the quickest way to run everything locally.

**Kubernetes** manifests will live in `k8s/`:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/
```

Includes Deployments, Services, ConfigMaps, Secrets, and a HorizontalPodAutoscaler for the API.

---

## Project Structure

```
tokentoll/
├── backend/
│   ├── tokentoll-api/        # gateway, auth, rate limiting, cache, routing
│   └── tokentoll-analyzer/   # Kafka consumer, anomaly detection, AI incidents
├── frontend/                 # React + TypeScript admin console
├── k8s/                      # Kubernetes manifests
├── loadtest/                 # k6 scripts
├── docs/                     # architecture diagram, screenshots
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Roadmap

- [ ] Gateway core with request logging
- [ ] Provider routing and Resilience4j failover
- [ ] Redis token-based rate limiting and budgets
- [ ] Semantic cache (pgvector)
- [ ] Kafka usage events and anomaly detection
- [ ] AI incident analyzer (Spring AI + RAG)
- [ ] SSE live alerts
- [ ] React admin console
- [ ] Docker Compose, Kubernetes, CI
- [ ] Load test results and demo video

Future ideas: agent orchestration layer, per-tenant model allow-lists, prompt versioning, OpenTelemetry tracing.

---

## License

Released under the MIT License. See the `LICENSE` file for details.

---

## Author

**Ashutosh**
