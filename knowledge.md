# OpenFoundry — Agent Knowledge Base

> Living document. Updated as the session progresses.
> Last updated: 2026-05-16

---

## 1. What This Repo Is

Open-source, cloud-native **operational data platform** inspired by Palantir Foundry.
Lets teams: connect data sources → version datasets → build object ontologies → expose APIs → automate workflows → govern access → run analytical/AI workloads — all with end-to-end auditability and self-hosted control.

Single Go module (`github.com/openfoundry/openfoundry-go`, Go 1.25) + React 19 frontend.
Originated as a port from a Rust workspace; Rust is gone but vocabulary lingers in older docs.

---

## 2. Top-Level Directory Map

```
apps/web/         React 19 + Vite + TypeScript frontend
services/         42 Go microservices (copy new ones from services/template/)
libs/             33 shared Go packages
proto/            Source-of-truth .proto files (26 packages)
libs/proto-gen/   Generated Go — DO NOT EDIT by hand, run `make gen`
sdks/             Generated TS/Python/Java client SDKs
infra/            Helm charts, ArgoCD, Terraform, runbooks
docs/             VitePress docs site
docs/archive/     Historical migration logs — DO NOT READ unless asked
tools/            CLIs: of-cli, route-audit, lint helpers, demos
```

Per-service shape (all services follow this exactly):
```
services/<svc>/
  cmd/<svc>/main.go
  internal/server/          chi router (/healthz /metrics /api)
  internal/handlers/
  internal/domain/          pure logic
  internal/repo/            sqlc-generated data access
  internal/repo/migrations/ goose SQL migrations
  internal/models/          wire types
  internal/config/          koanf-backed config
```

---

## 3. Canonical Commands

```sh
# Run from repo root — Makefile is authoritative, ignore justfile
make tools             # Install buf, golangci-lint, sqlc, gofumpt → ./bin
make ci                # tidy + vet + lint + contracts-check + test
make test              # Unit tests, -race + coverage (no Docker)
make test-integration  # Integration tests (testcontainers, needs Docker)
make gen               # Regen proto Go + sqlc + OpenAPI + SDKs
make contracts-check   # Verify OpenAPI + TS/Python/Java SDK drift
make build-services    # Compile all binaries → ./bin/
make lint              # golangci-lint (new-from-rev: HEAD baseline)
make fmt               # gofumpt + gci

# Frontend (apps/web/)
pnpm --filter @open-foundry/web dev    # Vite dev server
pnpm --filter @open-foundry/web check  # tsc -b --noEmit
pnpm --filter @open-foundry/web test   # Vitest
```

---

## 4. Technology Stack

| Layer            | Technology                                  | Notes |
|------------------|---------------------------------------------|-------|
| Backend          | Go 1.25, single root go.mod                 | 42 services + 33 libs |
| Frontend         | React 19 + Vite + TypeScript                | React Router 7, TanStack Query 5 |
| Contracts        | Protobuf 3 (buf)                            | 26 proto packages → OpenAPI → TS/Python/Java SDKs |
| HTTP routing     | chi                                         | Stateless reverse proxy at edge |
| Auth             | JWT (HS256/RS256) + Cedar ABAC/RBAC         | WebAuthn, OIDC, SAML support |
| Postgres         | CNPG (4 clusters)                           | Schema, policies, outbox, per-service data |
| Cassandra        | Hot operational state                       | Sub-ms writes, high-rate (ADR-0020) |
| Iceberg          | Analytical lakehouse                        | Audit events, AI events, dataset backing |
| Vespa            | Search + RAG                                | Hybrid BM25 + HNSW + phased ranking |
| Kafka            | Data-plane streaming                        | High-volume, durable events |
| NATS JetStream   | Control-plane signalling                    | Low-latency RPC-style signals (ADR-0011) |
| UI components    | TailwindCSS 4, ECharts, Cytoscape, MapLibre, Monaco | |
| Testing          | go test -race, testcontainers, Vitest, Playwright | |
| Infra            | Helm (5 releases), ArgoCD, Terraform        | |

---

## 5. The 42 Services (by Helm Release)

### Release 1 — `of-platform` (Identity & Control Plane)
| Service | Purpose |
|---------|---------|
| `edge-gateway-service` | HTTP reverse proxy, auth headers, rate limiting, audit fan-out |
| `identity-federation-service` | OIDC/SAML/MFA/WebAuthn login, JWT issuance, refresh rotation |
| `authorization-policy-service` | Cedar policies, RBAC roles/permissions/groups, restricted views |
| `tenancy-organizations-service` | Tenant resolution, orgs, enrollments, projects, spaces |

### Release 2 — `of-data-engine` (Data Integration & Ingestion)
| Service | Purpose |
|---------|---------|
| `connector-management-service` | Data connection sources, webhooks, egress policies, REST API sources |
| `ingestion-replication-service` | Ingest-job materialization, CDC metadata, replication control plane |
| `dataset-versioning-service` | Dataset CRUD, branches, transactions, versions, files, Iceberg backing |
| `media-sets-service` | Media item references, storage APIs |
| `lineage-service` | Dataset and column lineage |

### Release 3 — `of-ontology` (Object Model & Querying)
| Service | Purpose |
|---------|---------|
| `ontology-definition-service` | Object types, properties, interfaces, link types, action definitions |
| `object-database-service` | Object instances, link instances, revision history, transactional outbox |
| `ontology-query-service` | Search, graph traversal, object-set queries, KNN, read models |
| `ontology-actions-service` | Action execution, validation, funnel/functions/rules, policy-aware filters |

### Release 4 — `of-ml-aip` (ML & AI)
| Service | Purpose |
|---------|---------|
| `llm-catalog-service` | AI provider catalog (OpenAI, Anthropic, custom) |
| `model-catalog-service` | ML experiments, model versions, metadata |
| `model-deployment-service` | Deployments, predictions, drift detection, batch inference |
| `agent-runtime-service` | Agent/AI runtime, tool execution, prompt workflows, chat completions |
| `ai-evaluation-service` | AI guardrails and evaluations |
| `retrieval-context-service` | Knowledge-base retrieval, RAG context |

### Release 5 — `of-apps-ops` (Application Layer & Operations)
| Service | Purpose |
|---------|---------|
| `application-composition-service` | Workshop (app composition), templates, widget catalog, publishing |
| `workflow-automation-service` | Workflow orchestration, FASE, approvals |
| `notebook-runtime-service` | Notebook kernels, cells, sessions, reporting |
| `ontology-exploratory-analysis-service` | Exploratory analysis, geospatial (Map widget, layers) |
| `audit-compliance-service` | Audit collection, retention, SDS, GDPR compliance |
| `telemetry-governance-service` | Monitoring rules, telemetry governance |
| `federation-product-exchange-service` | Federation, marketplace, Nexus collaboration |
| `notification-alerting-service` | Notifications, delivery channels, WebSocket fan-out |
| `code-repository-review-service` | Code repositories, developer-platform flows |
| `entity-resolution-service` | Entity resolution, fusion APIs |
| `solution-design-service` | Solution/template authoring |
| `sql-bi-gateway-service` | Arrow Flight SQL, BI gateway, warehousing |
| `sdk-generation-service` | OpenAPI/TS/Python/Java SDK generation |
| `pipeline-build-service` | Pipeline authoring |
| `iceberg-catalog-service` | Iceberg REST catalog |

### Data-Plane Workers (Kafka consumers, no gateway routes)
| Service | Purpose |
|---------|---------|
| `audit-sink` | Kafka → Iceberg audit events |
| `ai-sink` | Kafka → Iceberg AI runtime events |
| `ontology-indexer` | Cassandra → Vespa read model indexing |
| `pipeline-runner` | Pipeline execution (DataFusion) |
| `pipeline-runner-spark` | Pipeline execution (Spark Operator) |
| `reindex-coordinator-service` | Foundry-pattern reindex orchestration |
| `compute-module-service` | User-supplied compute module hosting |
| `media-transform-runtime-service` | Media transform worker |

---

## 6. The 33 Shared Libraries

| Library | Purpose |
|---------|---------|
| `auth-middleware` | **SECURITY-CRITICAL** JWT validation, claims, RBAC, RLS, tenant resolution |
| `authz-cedar-go` | Cedar policy engine Rust bindings + validation |
| `audit-trail` | Audit event types, envelope format, publisher, middleware |
| `core-models` | Canonical wire types: IDs, errors, health, pagination, dataset RID, markings |
| `db-pool` | Postgres writer + reader pool pair (CNPG + PgBouncer) |
| `event-bus-control` | NATS JetStream control-plane publisher/subscriber |
| `event-bus-data` | Kafka data-plane consumer/producer |
| `event-scheduler` | Cron scheduler tick loop |
| `observability` | slog structured logging + OTel tracing + Prometheus metrics |
| `ontology-kernel` | Largest lib (~35k LOC): object types, properties, actions, functions, rules, search, handlers |
| `pipeline-expression` | Pipeline transform expressions, DataFusion runtime, schema inference |
| `plugin-sdk` | WASM connector SDK (placeholder) |
| `proto-gen` | Generated Protobuf Go code — DO NOT EDIT |
| `python-sidecar` | Python function execution, pyo3 bridge |
| `query-engine` | Query evaluation, literal SELECT, BI aggregations |
| `saga` | LIFO compensation saga runner with typed events |
| `scheduling-cron` | Cron job definition and parsing |
| `scheduling-linter` | Cron expression validation |
| `state-machine` | Postgres-backed state machine with optimistic concurrency |
| `storage-abstraction` | Filesystem/S3 presigning, multipart upload helpers |
| `testing` | Testcontainers helpers, fixtures, test runners |
| `vector-store` | Vector backend abstraction (Vespa + pgvector) |
| `ai-kernel-go` | ~7.7k LOC: multi-provider LLM gateway, ReAct agent loop, chat runtime |
| `ml-kernel-go` | ML model utilities |
| `geospatial-core` | GeoPoint, GeometricShape, GPX parser, CRS policy, Haversine |
| `geospatial-tiles` | Vector tile generation, TMS service |
| `cassandra-kernel` | Cassandra connection pooling, keyspace helpers |
| `idempotency` | Postgres idempotency store (record-before-process pattern) |
| `outbox` | Transactional outbox pattern + Debezium EventRouter SMT |
| `media-scanner` | Media file scanning, metadata extraction |
| `analytical-logic` | Analytical expressions, aggregation functions |
| `search-abstraction` | Search backend abstraction |
| `capabilities` | Platform capability snapshot/registry |

---

## 7. Proto Packages (26 total under `proto/`)

`ai/`, `app_builder/`, `audit/`, `auth/`, `code_repo/`, `common/`, `data_integration/`, `dataset/`, `fusion/`, `geospatial/`, `marketplace/`, `media_set/`, `ml/`, `nexus/`, `notebook/`, `notification/`, `ontology/`, `pipeline/`, `query/`, `report/`, `runtime/`, `streaming/`, `workflow/`

---

## 8. Frontend Routes (`apps/web/`)

| Route | Surface |
|-------|---------|
| `/dashboard` | Main platform home |
| `/datasets`, `/pipelines`, `/workflows` | Data platform |
| `/ontology` | Object model browsing/editing |
| `/apps`, `/apps/runtime/:slug` | Workshop app composer + runtime |
| `/audit`, `/alerts`, `/federation` | Operations |
| `/geospatial` | Map view (MapLibre GL) |
| `/ml`, `/ai` | Model/agent catalog |

---

## 9. Key Architectural Patterns

| Pattern | Implementation |
|---------|---------------|
| **Contracts-first** | `.proto` → OpenAPI → TS/Python/Java SDKs; all generated, drift-checked in CI |
| **Transactional outbox** | Postgres `outbox` table + Debezium CDC → Kafka; atomic domain events |
| **Foundry-pattern orchestration** | Kafka + state machines replaces Temporal; fully async saga execution (FASE) |
| **Cedar authorization** | Declarative ABAC/RBAC; embedded Rust library; auditable rules |
| **Marking-based access** | Security labels propagate through ontology graph; Cedar enforces at eval |
| **DI via structs** | All state on `*AppState`/`*Handlers`; no package-level globals; only 3 `init()` in entire tree |
| **Split event buses** | NATS (control, low-latency) + Kafka (data, high-volume) |
| **Golden tests** | JSON fixture files pin wire format; regression detection for SDK/frontend |
| **DDD bounded contexts** | One handler module per domain subdirectory; pure domain logic separate from HTTP |

---

## 10. Coding Conventions

- **Errors**: `errors.Is`-style sentinels at package scope (`ErrNotFound`, `ErrPreconditionFailed`, …); HTTP layer maps them
- **Auth claims**: always from `r.Context()` via lib helpers — never parse JWT in handlers
- **Wire types**: `models.Page[T]` and `models.ListResponse[T]`; cursor-pagination uses `next_cursor`
- **Observability**: `libs/observability` only; each service exposes `/metrics`; do not re-register globals
- **Testing**: unit tests next to source as `*_test.go`; anything needing infra uses `//go:build integration` + testcontainers helpers
- **Single module**: root `go.mod` only — never create per-service modules

---

## 11. Security-Critical Zones (Require Human Review)

- `services/identity-federation-service/` — OIDC, SAML, MFA, WebAuthn, SCIM, JWKS rotation
- `services/authorization-policy-service/` — Cedar engine, ABAC/RBAC evaluation
- `libs/auth-middleware/` — JWT validation chain
- `services/*/internal/repo/migrations/` — destructive DDL
- `proto/auth/`, `proto/audit/` — wire-format breakage hits every consumer

---

## 12. ADRs (Key Decisions)

| ADR | Decision |
|-----|---------|
| ADR-0011 | Control plane (NATS) vs. data plane (Kafka) split |
| ADR-0020 | Cassandra for operational state (not Iceberg) |
| ADR-0022 | Transactional outbox (Postgres) + Debezium |
| ADR-0024 | Postgres consolidation (4 CNPG clusters) |
| ADR-0027 | Cedar policy engine (embedded Rust) |
| ADR-0030 | Service consolidation to 33 boundaries + 5 Helm releases |
| ADR-0031 | Helm chart split into 5 releases |
| ADR-0037 | Foundry-pattern orchestration (Kafka + state machines, replaces Temporal) |

---

## 13. CI/CD Gates

| Workflow | Trigger | Jobs |
|----------|---------|------|
| `openfoundry-go.yml` | Push to `main`, PRs | lint, vet, tidy, proto, sqlc, test, integration, contracts-check |
| `proto-check.yml` | Proto changes | buf lint, OpenAPI/SDK validation |
| `security-audit.yml` | Schedule + go.mod/sum changes | govulncheck |
| `chaos-smoke.yml` | Nightly + manual dispatch | chaos-mesh SPOF tests |

**Removed gates (not yet reimplemented in Go):**
- `bus-contract` lint
- `data-residency` registry check
- Per-service Iceberg llvm-cov ≥72% threshold
- Pyiceberg / Playwright Iceberg E2E suites

---

## 14. Tools Directory

| Tool | Purpose |
|------|---------|
| `of-cli` | OpenAPI validation, SDK generation, smoke execution, benchmarks, mock providers |
| `route-audit` | HTTP route smoke testing (detects orphaned endpoints) |
| `capabilities-snapshot` | Platform capability registry |
| `kafka-lint` | Kafka configuration linter |
| `ceph-lint` | Ceph configuration linter |
| `demo/` | Trail Running demo (fixtures, pipelines, functions, Workshop app) |
| `online-retail/` | Online retail demo scenario |

---

## 15. Session Notes (updated as we work)

- **2026-05-16**: Created `knowledge.md` (this file) as the agent knowledge base
- **2026-05-16**: Created `IDEAS.md` — 8 sections, 30+ detailed use cases covering: work (legionella testing company), home networking, music collection, self-hosting, personal finance, home automation, media library, open data. Priority matrix included.
