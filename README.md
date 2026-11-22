# FARO
### Fog Aggregation and Relay Orchestration

> A fault-tolerant, multi-objective task scheduling system for heterogeneous fog computing environments.

FARO accepts IoT task submissions via REST, scores fog nodes using a hybrid Fault Tolerance engine (M/M/1 queuing theory + fuzzy logic), filters and ranks candidate nodes, dispatches tasks via a multi-objective NSGA-II optimizer, and continuously improves through closed-loop execution feedback.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Pipeline](#pipeline)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Run Locally](#run-locally)
  - [Run with Docker](#run-with-docker)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Dashboard](#dashboard)
- [Project Structure](#project-structure)
- [Running Tests](#running-tests)
- [Documents](#documents)

---

## Overview

Fog computing nodes are unreliable by nature — variable CPU load, fluctuating queue depths, uncertain latency, unplanned failures. Most schedulers either ignore fault tolerance entirely (Round Robin, FCFS) or fall back to cloud, which introduces unacceptable latency for deadline-sensitive IoT tasks.

FARO solves this by computing a **Combined FT Score** for every candidate node before any assignment is made. Only nodes that can provably meet the task's deadline and resource requirements are considered. The assignment decision is then optimised simultaneously across three objectives: completion time, fault tolerance, and load balance.

**Key metrics FARO targets:**

| Metric | Target |
|--------|--------|
| Deadline Satisfaction Rate (DSR) | ≥ 90% at 200 tasks/s |
| Assignment decision latency | < 50 ms p95 |
| Failure recovery time | < 200 ms |
| Dashboard data freshness | ≤ 3 seconds |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        IoT Devices / Gateways                   │
│                     POST /api/v1/tasks (REST)                   │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                        FARO — Spring Boot                       │
│                                                                 │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────────┐  │
│  │  REST API    │   │  Core Engine   │   │   Thymeleaf UI   │  │
│  │  Controllers │──▶│  FT + NSGA-II  │   │   Dashboard      │  │
│  │  DTOs        │   │  Filter+Ranker │   │   (3s refresh)   │  │
│  └──────────────┘   └───────┬────────┘   └──────────────────┘  │
│                             │                                   │
│  ┌──────────────────────────▼──────────────────────────────┐   │
│  │              Services (Orchestration Layer)              │   │
│  │  TaskService │ SchedulingService │ ExecutionMonitor      │   │
│  │  NodeRegistryService │ FeedbackService                   │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│  ┌──────────────────────────▼──────────────────────────────┐   │
│  │              Infrastructure Layer                        │   │
│  │  Spring Data JPA (H2/PostgreSQL)                        │   │
│  │  InMemoryNodeRegistry (ConcurrentHashMap)               │   │
│  │  FogNodeClient (RestTemplate → fog node REST APIs)      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
            ┌─────────────────────────┐
            │      Fog Nodes          │
            │  node-01  node-02  ...  │
            └─────────────────────────┘
```

---

## Pipeline

FARO processes every task through an 8-step pipeline:

```
Step 1 — IoT Task Intake
         POST /api/v1/tasks → validate → persist (status: PENDING)
         ↓
Step 2 — Multi-Attribute Task Parsing
         Deadline, Priority, Criticality, CPU demand, Dependencies
         ↓
Step 3 — Node State Registry
         Live node state from InMemoryNodeRegistry
         Heartbeat timeout: 15s → DEGRADED, 30s → DOWN
         ↓
Step 4 — Hybrid Fault Tolerance Engine
         Analytical Model: M/M/1 queuing (ρ, Wq) + availability
         Fuzzy Engine: CPU load × latency variance × failure rate → node risk
         Combined FT Score = 0.6 × analytical + 0.4 × (1 − fuzzy risk)
         ↓
Step 5 — Hard Feasibility Filter
         Eliminates: insufficient CPU/memory, unreachable deadline, DOWN nodes
         ↓
Step 6 — SCCB Ranker
         Score + Criticality + Capacity + Balance → ranked candidate list
         ↓
Step 7 — FT-NSGA-II Optimizer
         3-objective Pareto optimisation:
           f1: min completion time
           f2: max FT score
           f3: min load imbalance
         → single best assignment (weighted Pareto selection)
         ↓
Step 8 — Assignment + Monitoring + Feedback
         Dispatch to fog node → poll execution → detect failure → re-assign
         → update node health (EMA) → update ranker/optimizer weights
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 21 |
| Framework | Spring Boot 3.x |
| Template Engine | Thymeleaf |
| Persistence | Spring Data JPA + H2 (dev) / PostgreSQL (prod) |
| Build | Maven |
| Charts | Chart.js 4.x |
| Animation | Anime.js 3.x + Motion.dev 10.x |
| Icons | Lucide |
| Containerisation | Docker + docker-compose |
| Testing | JUnit 5 + Mockito + Spring Boot Test |
| Coverage | JaCoCo |
| CI | GitHub Actions |

---

## Getting Started

### Prerequisites

- Java 21+
- Maven 3.9+
- Docker + Docker Compose (for containerised run)

### Run Locally

```bash
# Clone the repository
git clone https://github.com/<your-username>/faro.git
cd faro

# Copy environment variables
cp .env.example .env
# Edit .env and set FARO_API_KEY

# Run with dev profile (H2 in-memory database)
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

App starts at `http://localhost:8080`
Dashboard: `http://localhost:8080/dashboard`

### Run with Docker

```bash
# Build and start FARO + PostgreSQL
docker-compose up --build

# App starts at http://localhost:8080
# PostgreSQL on port 5432
```

To stop:
```bash
docker-compose down
```

To run with a clean database:
```bash
docker-compose down -v && docker-compose up --build
```

---

## Configuration

All configuration lives in `src/main/resources/application.yml`.
Sensitive values are loaded from environment variables.

Copy `.env.example` to `.env` and fill in the values:

```env
# Required
FARO_API_KEY=your-secret-api-key

# Database (prod profile only — dev uses H2)
DB_URL=jdbc:postgresql://localhost:5432/farodb
DB_USER=faro
DB_PASS=your-db-password
```

Key configuration options (`application.yml`):

```yaml
faro:
  scheduler:
    heartbeat-timeout-degraded-ms: 15000   # Node → DEGRADED after this
    heartbeat-timeout-down-ms: 30000       # Node → DOWN after this
    execution-poll-interval-ms: 2000       # How often to poll task status
    top-k-candidates: 5                    # Nodes passed to optimizer

  ft:
    analytical-weight: 0.6                 # Weight of M/M/1 model in FT Score
    fuzzy-weight: 0.4                      # Weight of fuzzy engine in FT Score
    safe-threshold: 0.7                    # FT Score ≥ 0.7 = safe node
    marginal-threshold: 0.4                # FT Score < 0.4 = excluded

  sccb:
    ft-weight: 0.4
    criticality-weight: 0.3
    capacity-weight: 0.2
    balance-weight: 0.1
```

---

## API Reference

All API endpoints require the header:
```
X-FARO-Key: <your-api-key>
```

### Submit a Task

```http
POST /api/v1/tasks
Content-Type: application/json

{
  "taskId": "550e8400-e29b-41d4-a716-446655440000",
  "deadline_ms": 1700000060000,
  "priority": 3,
  "criticality": "HIGH",
  "cpu_demand_mhz": 200,
  "memory_demand_mb": 128,
  "dependencies": []
}
```

Response `202 Accepted`:
```json
{
  "assignmentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "status": "PENDING",
  "message": "Task accepted. Assignment in progress."
}
```

### Register a Fog Node

```http
POST /api/v1/nodes/register
Content-Type: application/json

{
  "nodeId": "node-01",
  "displayName": "Fog Node Alpha",
  "cpu_total_mhz": 2000,
  "memory_total_mb": 4096,
  "queue_capacity": 32,
  "endpoint": "http://192.168.1.10:8080"
}
```

### Send Heartbeat

```http
POST /api/v1/nodes/node-01/heartbeat
Content-Type: application/json

{
  "cpu_available_mhz": 1400,
  "memory_available_mb": 2800,
  "queue_depth": 8,
  "avg_latency_ms": 94
}
```

Nodes must send a heartbeat every **5 seconds**.
Missing heartbeat for 15s → `DEGRADED`. Missing for 30s → `DOWN`.

### Query Assignments

```http
GET /api/v1/assignments?page=0&size=50
```

---

## Dashboard

The Thymeleaf dashboard is available at `/dashboard` after startup.

| Page | URL | Description |
|------|-----|-------------|
| Dashboard | `/dashboard` | Metric cards, node grid, assignment log, charts |
| Nodes | `/nodes` | Full node registry with FT score bars |
| Assignments | `/assignments` | Full assignment log, filterable |
| Metrics | `/metrics` | Expanded DSR and FT score charts |

**Features:**
- Auto-refreshes every 3 seconds
- Dark mode default, light mode toggle (persisted in `localStorage`)
- DSR vs arrival rate bar chart
- FT score over time per node (line chart)
- Node status indicators: `UP` (green) / `DEGRADED` (yellow) / `DOWN` (red)
- Cloud fallback tracking

---

## Project Structure

```
src/
├── main/
│   ├── java/dev/faro/
│   │   ├── api/              # REST Controllers + DTOs
│   │   ├── core/             # Pure domain logic (zero Spring dependencies)
│   │   │   ├── ft/           # FT Engine: Analytical + Fuzzy + Composer
│   │   │   ├── filter/       # Hard Feasibility Filter
│   │   │   ├── ranking/      # SCCB Ranker
│   │   │   ├── optimizer/    # FT-NSGA-II Optimizer
│   │   │   └── assignment/   # Assignment Engine
│   │   ├── service/          # Application services (orchestration)
│   │   ├── infrastructure/   # JPA, InMemoryRegistry, FogNodeClient
│   │   ├── web/              # Thymeleaf Dashboard Controllers
│   │   └── config/           # Spring configuration
│   └── resources/
│       ├── static/
│       │   ├── css/faro.css
│       │   ├── js/faro-dashboard.js
│       │   ├── js/faro-theme.js
│       │   └── fonts/        # Geist + Geist Mono woff2
│       └── templates/        # Thymeleaf HTML templates
└── test/
    └── java/dev/faro/
        ├── core/             # Unit tests for all core modules
        ├── service/          # Service layer tests
        └── api/              # Controller integration tests
```

---

## Running Tests

```bash
# Run all tests
mvn test

# Run with coverage report
mvn test jacoco:report
# Report at: target/site/jacoco/index.html

# Run a specific test class
mvn test -Dtest=FTScoreComposerTest

# Run only core module tests
mvn test -Dtest="dev.faro.core.**"
```

Coverage requirement: **≥ 80% line coverage on `core/`**. The build fails below this threshold.

---

## Documents

Full project documentation is in the `/docs` folder:

| Document | Description |
|----------|-------------|
| [`PRD.md`](docs/PRD.md) | Product Requirements — goals, functional requirements, data models |
| [`ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System design, pipeline flow, algorithms, DB schema, API contracts |
| [`DESIGN_SYSTEM.md`](docs/DESIGN_SYSTEM.md) | UI design system — colours, typography, components, animations |
| [`SPRINT_PLAN.md`](docs/SPRINT_PLAN.md) | 4-sprint delivery plan with task breakdown and done criteria |
| [`ENGINEERING_STANDARDS.md`](docs/ENGINEERING_STANDARDS.md) | Code style, naming conventions, testing rules, Git standards |

---

## Status

| Sprint | Scope | Status |
|--------|-------|--------|
| Sprint 0 | Project skeleton, CI | 🔲 Not started |
| Sprint 1 | REST API + Node Registry | 🔲 Not started |
| Sprint 2 | FT Engine + Filter + Ranker | 🔲 Not started |
| Sprint 3 | Optimizer + Assignment + Monitoring | 🔲 Not started |
| Sprint 4 | Dashboard + Docker + Release | 🔲 Not started |

---

## License

MIT License — see [`LICENSE`](LICENSE) for details.

---

*FARO v1.0.0 — Fog Aggregation and Relay Orchestration*  
*Minimal. Deterministic. Fault-tolerant.*
