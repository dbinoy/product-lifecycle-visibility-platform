# CompassX — Solution Proposal

**Enterprise Product Lifecycle Visibility Platform**

---

## 1. Business Outcomes

CompassX addresses the operational and strategic costs of fragmented lifecycle data across enterprise manufacturing and delivery. The platform is designed to deliver the following outcomes:

**Customer-Facing**
- **Improved transparency**: Clients gain real-time, self-service visibility into the status of their orders across every lifecycle stage — eliminating the need to contact account managers for routine updates.
- **Proactive issue communication**: When delays, quality failures, or installation blockers arise, clients are notified automatically with revised timelines rather than discovering problems at the last moment.
- **Increased trust and satisfaction**: Transparent communication during long delivery cycles builds client confidence and reduces perceived risk.

**Operational**
- **Earlier delay detection**: Integrating signals from manufacturing, quality, and logistics allows teams to identify risks weeks earlier than current manual tracking allows.
- **Reduced coordination overhead**: A shared lifecycle view eliminates duplicate status queries across teams and reduces the time spent in cross-functional status meetings.
- **Faster response to quality or production issues**: Anomaly detection surfaces issues at the moment they emerge, enabling immediate escalation and mitigation.

**Executive**
- **Delivery performance visibility**: Leadership gains a real-time view of on-time delivery rates, manufacturing health, and quality trends — previously unavailable without manual reporting cycles.
- **Revenue risk identification**: Orders at risk of SLA breach or customer dissatisfaction are surfaced proactively, enabling executive intervention before impact.

---

## 2. Product Vision

CompassX is a **lifecycle intelligence hub** — an overlay platform that sits above existing systems of record and presents a unified, role-appropriate view of every order's journey from placement through ongoing maintenance.

Rather than replacing existing systems, CompassX integrates with them to provide what none of them can offer individually: a complete, cross-functional picture of lifecycle status for every stakeholder.

### Lifecycle Stages

Every order in CompassX progresses through the following stages, with status surfaced from the appropriate source system at each step:

```
Order Placement → BOM & Configuration → Manufacturing → Quality Assurance
      → Shipping & Logistics → Installation & Setup → Maintenance & Service
```

### Experience by Role

**Client Portal**

Clients see their order as a journey. The portal provides:
- A vertical timeline showing each lifecycle stage with status (completed / in progress / pending / at risk)
- Estimated completion dates at each stage with confidence indicators
- Plain-language alerts when issues arise ("Manufacturing rework required — revised delivery estimate: June 3")
- Key documents: technical specifications, certificates of conformance, delivery confirmations
- A single contact point for escalation when alerts are raised

The goal is to make clients feel informed and in control without requiring any interaction with internal teams for routine updates.

**Operations Dashboard**

Operations teams see all active orders in a portfolio view. The dashboard provides:
- A filterable order table by stage, risk level, customer, product line, and assigned team
- A bottleneck panel highlighting which lifecycle stages have the most stalled orders
- A dependency view showing orders blocked by upstream issues (BOM shortages, quality holds, logistics delays)
- A live alert feed with severity tiers (critical / warning / informational)
- Drill-down into any order's full lifecycle detail

The goal is to give each team the operational context they need to prioritize work and resolve blockers without waiting for a status meeting.

**Executive Dashboard**

Executives see a high-level command center. The dashboard provides:
- KPI cards: On-Time Delivery %, average order cycle time, quality pass rate, count of orders at risk
- Trend charts (30 / 90 / 180-day windows) for delivery performance and manufacturing throughput
- A revenue-at-risk table linking delayed orders to contract value
- An AI-generated risk narrative: a daily natural-language summary of the most significant risks across the portfolio

The goal is to give leadership the ability to spot systemic issues, track performance trends, and intervene before risks become customer-impacting events.

---

## 3. Data and Systems Strategy

### Source Systems

CompassX integrates with the following categories of systems. In the initial build, all integrations use mock connectors that simulate realistic data; the adapter pattern ensures each mock can be swapped for a real connector without changing core platform logic.

| System Category | Example Products | Data Provided |
|---|---|---|
| CRM | Salesforce, HubSpot | Client profiles, order records, account contacts |
| ERP | SAP S/4HANA, Oracle | Order management, fulfillment status, inventory |
| PLM / BOM | Windchill, Teamcenter | Product configurations, bill of materials, revisions |
| MES | Siemens Opcenter, Plex | Production work orders, progress, machine events |
| QMS | ETQ, MasterControl | Inspection records, test results, compliance status |
| TMS / Logistics | Oracle TMS, SAP TM | Shipment records, carrier tracking, delivery ETAs |
| FSM | ServiceMax, Salesforce FSM | Installation records, site readiness, technician assignments |
| ITSM | ServiceNow, Jira Service Mgmt | Service requests, maintenance history, SLA status |

### Key Data Entities

The platform's internal data model is organized around the `Order` as the primary lifecycle container, with all downstream entities linked by `order_id`:

```
Order
 ├── ProductConfiguration
 │    └── BillOfMaterials (line items)
 ├── ProductionWorkOrder (one or many per order)
 ├── QualityInspectionRecord (one or many per work order)
 ├── Shipment
 │    └── DeliveryEvent (tracking milestones)
 ├── InstallationRecord
 └── ServiceTicket (recurring, post-installation)
```

Every entity carries:
- `source_system`: which integration provided this record
- `external_id`: the ID in the source system
- `last_synced_at`: timestamp of the most recent data pull
- `data_confidence`: enum (fresh / stale / missing) based on sync age thresholds

### Integration Patterns

| Pattern | Used For | Rationale |
|---|---|---|
| REST polling | CRM, ERP, PLM, QMS, FSM, ITSM | Most enterprise systems expose REST APIs; polling on configurable intervals (1–15 min) is sufficient for these stages |
| Webhooks / event push | MES, TMS / Logistics | Production events and shipment tracking require near-real-time updates |
| Batch ETL | Legacy ERP / on-premise systems | Some older systems have no API; scheduled batch exports (CSV, DB views) are processed nightly |
| Change Data Capture (CDC) | Database-direct integrations | Where direct DB access is granted, CDC via Debezium captures row-level changes with minimal latency |

All integration events are routed through **Amazon SQS** before processing. This decouples ingestion rate from processing speed and provides a durable buffer during high-volume periods or downstream outages.

### Handling Fragmented and Inconsistent Data

| Challenge | Strategy |
|---|---|
| Missing stage data | Display "Awaiting data from [System]" with last-synced timestamp; never block lifecycle progress display |
| Conflicting records | Apply a reconciliation priority order per entity type (e.g., ERP overrides CRM for order status) |
| Stale data | Surface staleness indicator in UI when `last_synced_at` exceeds threshold (configurable per connector) |
| Schema variations | Each connector adapter normalizes source data into the canonical CompassX entity model before storage |
| Duplicate events | Idempotent ingestion workers: records are upserted by `(source_system, external_id)` — no duplicate rows |

---

## 4. Solution Architecture

### 4.1 System Overview

```
  INTERNET
     │
     ▼
┌────────────────────────────────────────────────────────────────────────┐
│  AWS ACCOUNT  (us-east-1, multi-AZ: us-east-1a / us-east-1b)           │
│                                                                        │
│  ┌─────────────────────────── PUBLIC SUBNETS ──────────────────────┐   │
│  │   Route 53 (compassx.io)                                        │   │
│  │        │                            │                           │   │
│  │   CloudFront ◄── S3 (React SPA)    ALB (HTTPS :443)             │   │
│  │   (static CDN)                      │                           │   │
│  └─────────────────────────────────────┼───────────────────────────┘   │
│                                        │  (VPC-internal, port 8000)    │
│  ┌─────────────────────────── PRIVATE SUBNETS ─────────────────────┐   │
│  │                                                                 │   │
│  │   ECS Fargate Cluster                                           │   │
│  │   ┌──────────────────┐  ┌─────────────────────┐                 │   │
│  │   │  api-service     │  │  worker-service     │                 │   │
│  │   │  (FastAPI)       │  │  (Ingestion Workers)│                 │   │
│  │   │  2–8 tasks, auto │  │  1–4 tasks, auto    │                 │   │
│  │   │  scale on CPU    │  │  scale on SQS depth │                 │   │
│  │   └────────┬─────────┘  └─────────┬───────────┘                 │   │
│  │            │                      │                             │   │
│  │   ┌────────▼──────────┐  ┌────────▼────────────┐                │   │
│  │   │  RDS PostgreSQL   │  │  SQS FIFO Queues    │                │   │
│  │   │  (Multi-AZ)       │  │  (per source system)│                │   │
│  │   │  + RDS Proxy      │  │  + Dead Letter Queue│                │   │
│  │   └───────────────────┘  └─────────────────────┘                │   │
│  │                                                                 │   │
│  │   ┌───────────────────┐  ┌─────────────────────┐                │   │
│  │   │  ElastiCache Redis│  │  EventBridge        │                │   │
│  │   │  (cluster mode)   │  │  (Scheduler + Rules)│                │   │
│  │   └───────────────────┘  └─────────────────────┘                │   │
│  │                                                                 │   │
│  │   ┌─────────────────────────────────────────────────────────┐   │   │
│  │   │  S3 Buckets                                             │   │   │
│  │   │  compassx-events (raw payloads)  │  compassx-ml-models  │   │   │
│  │   │  compassx-documents              │  compassx-exports    │   │   │
│  │   └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  CROSS-ACCOUNT / EXTERNAL                                              │
│  Secrets Manager  │  Cognito User Pool  │  API Gateway (HTTP API)      │
│  CloudWatch Logs  │  ECR (container registry)  │  X-Ray (tracing)      │
└────────────────────────────────────────────────────────────────────────┘
         │                           │
    SOURCE SYSTEMS              Anthropic Claude API
 (CRM, ERP, MES, etc.)         (external HTTPS)
```

---

### 4.2 Compute: ECS Fargate Services

All application code runs as Docker containers on ECS Fargate — no servers to manage, scales independently per service.

**`api-service`** — FastAPI application
- Task definition: 1 vCPU, 2 GB RAM per task
- Desired count: 2 tasks (HA); auto-scales 2–8 based on ALB request count
- Exposes port 8000; ALB target group health check: `GET /health`
- Environment variables injected from AWS Secrets Manager at task startup (DB URL, Redis URL, Cognito pool ID, Claude API key)
- Runs with `uvicorn app.main:app --workers 1` (single worker per container; ECS handles horizontal scaling)

**`worker-service`** — Ingestion workers
- Task definition: 0.5 vCPU, 1 GB RAM per task
- Auto-scales 1–4 tasks based on SQS approximate message count (target: < 500 messages per task)
- Each worker process subscribes to all SQS queues in a round-robin polling loop
- No inbound network exposure; only outbound to RDS, Redis, S3, and source system APIs
- Polling schedules managed by **EventBridge Scheduler** (one rule per source system, configurable interval)

**Task role (IAM):** Both services share a task execution role with least-privilege policies:
- `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:SendMessage` on the ingestion queues
- `s3:PutObject` on `compassx-events`, `s3:GetObject` on `compassx-ml-models`
- `secretsmanager:GetSecretValue` on scoped secret ARNs
- `rds-db:connect` via IAM database authentication (no static password in env)

---

### 4.3 Networking and Routing

**VPC layout:**
- 2 public subnets (one per AZ): ALB, NAT Gateways
- 4 private subnets (two per AZ): ECS tasks (app), data resources (RDS, Redis, SQS VPC endpoints)
- All ECS tasks have no public IPs; outbound traffic routes through NAT Gateway

**Application Load Balancer (ALB):**
- HTTPS listener (ACM certificate for `api.compassx.io`)
- Listener rules:
  - `POST /webhooks/*` → `api-service` target group (webhook ingestion endpoint)
  - `/api/*` → `api-service` target group (REST + WebSocket)
  - All other paths return 404 (frontend is served from CloudFront, not ALB)

**CloudFront + S3 (frontend):**
- React SPA built by Vite → uploaded to `compassx-frontend` S3 bucket on each deployment
- CloudFront distribution: `compassx.io` → S3 origin for static assets; `/api/*` forwarded to ALB origin
- Single origin policy: frontend and API share the same domain — no CORS required
- CloudFront caches static assets with long TTLs; API paths are uncached (pass-through)

**API Gateway (HTTP API):**
- Used for external webhook ingestion only (source systems that cannot reach the ALB directly)
- Routes `POST /ingest/{source}` → Lambda authorizer → SQS `SendMessage` directly
- Decouples webhook receipt from ECS availability; SQS provides the buffer

**Security groups:**
| Resource | Inbound | Outbound |
|---|---|---|
| ALB | 443 from 0.0.0.0/0 | 8000 to api-service SG |
| api-service ECS tasks | 8000 from ALB SG | 5432 to RDS SG, 6379 to Redis SG, 443 to internet (Secrets, SQS, Claude API) |
| worker-service ECS tasks | none | 5432 to RDS SG, 6379 to Redis SG, 443 to internet |
| RDS | 5432 from ECS task SGs | none |
| ElastiCache | 6379 from ECS task SGs | none |

---

### 4.4 Authentication and Authorization

**AWS Cognito User Pool (`compassx-users`)**
- 3 groups: `clients`, `ops`, `executives`
- Supports standard username/password and SAML federation (enterprise SSO via Okta/Azure AD)
- Access token lifetime: 1 hour; refresh token: 30 days
- Custom attributes: `custom:org_id` (tenant isolation), `custom:role`

**Token flow:**
1. User authenticates via Cognito Hosted UI or PKCE flow from the React app
2. Cognito issues JWT access token (signed with RS256)
3. React app stores tokens in memory (not localStorage); refresh token in httpOnly cookie
4. Every API request includes `Authorization: Bearer <access_token>`
5. FastAPI `verify_token` dependency validates JWT signature against Cognito JWKS endpoint, extracts `custom:role` and `custom:org_id`, attaches to request context
6. Route-level decorators enforce role: `@require_role("ops")`, `@require_role("executive")`

**Tenant isolation:** All database queries are scoped by `org_id` extracted from the JWT. ORM base class automatically applies `WHERE org_id = :org_id` — no query can cross tenant boundaries.

---

### 4.5 Integration and Ingestion Layer

**ConnectorAdapter interface (Python abstract base class):**
```python
class ConnectorAdapter(ABC):
    async def fetch_updates(self, since: datetime) -> list[RawEvent]
    async def handle_webhook(self, payload: dict) -> RawEvent
    def get_source_system(self) -> SourceSystem
```

Mock adapters (`MockCRMAdapter`, `MockERPAdapter`, etc.) implement this interface using Faker-generated data seeded by `order_id` for determinism. Real adapters implement identical method signatures — no downstream code changes when swapping.

**Polling flow (REST-based source systems):**
1. EventBridge Scheduler fires a rule every N minutes (per-connector, configurable: 1–15 min)
2. Rule invokes ECS Run Task on `worker-service` with task override `CONNECTOR=crm`
3. Worker instantiates the correct adapter, calls `fetch_updates(since=last_run_timestamp)`
4. Raw events are written to S3 (`compassx-events/{source}/{date}/`) and published to the source's SQS FIFO queue
5. `last_run_timestamp` is updated in a `connector_sync_state` table in PostgreSQL

**Webhook flow (near-real-time source systems: MES, TMS):**
1. Source system POSTs to `POST /webhooks/{source}` on the ALB
2. FastAPI webhook handler validates HMAC signature (per-connector secret stored in Secrets Manager)
3. Payload is immediately enqueued to SQS — handler returns 200 within ~5ms
4. SQS message is processed asynchronously by `worker-service`

**SQS queue design:**
- One FIFO queue per source system (8 queues total): `compassx-crm.fifo`, `compassx-mes.fifo`, etc.
- Message group ID = `order_id` — guarantees ordered processing per order across concurrent workers
- Visibility timeout: 60 seconds; max receive count: 3
- Dead Letter Queue (`compassx-dlq.fifo`): messages that fail 3 times are moved here and trigger a CloudWatch alarm

**Ingestion worker processing (per SQS message):**
1. Deserialize `RawEvent` from SQS message body
2. Call `DataNormalizer.normalize(raw_event)` → produces canonical entity (e.g., `ProductionWorkOrder`)
3. Call `Reconciler.upsert(entity)` → `INSERT ... ON CONFLICT (source_system, external_id) DO UPDATE`
4. Invalidate relevant Redis cache keys for the affected `order_id`
5. If alert threshold crossed: publish alert to Redis Pub/Sub channel `alerts:{org_id}`
6. Delete SQS message (acknowledge)

---

### 4.6 Data Layer

**PostgreSQL (RDS `db.t3.large`, Multi-AZ)**

Connection management via **RDS Proxy** — pools connections from potentially many ECS tasks (up to 50 concurrent) without exhausting PostgreSQL's connection limit. ECS tasks connect to the Proxy endpoint, not RDS directly.

Key tables (simplified):
```sql
orders              (id, org_id, external_id, source_system, customer_id, status, ...)
product_configs     (id, order_id, ...)
bom_items           (id, config_id, part_number, quantity, availability_status, ...)
production_work_orders (id, order_id, work_center, progress_pct, status, ...)
quality_inspections (id, work_order_id, result, hold_reason, ...)
shipments           (id, order_id, carrier, tracking_number, eta, ...)
delivery_events     (id, shipment_id, event_type, timestamp, location, ...)
installation_records (id, order_id, technician_id, scheduled_date, status, ...)
service_tickets     (id, order_id, type, severity, status, created_at, ...)
-- All tables include: source_system, external_id, last_synced_at, org_id
```

**pgvector extension** is enabled on the same PostgreSQL instance. A `lifecycle_embeddings` table stores vector embeddings (1536-dimensional) of lifecycle summaries for RAG retrieval. This avoids a separate vector database for the initial build.

**Read replica:** A single read replica serves the Executive Dashboard's aggregate queries (KPI calculations, trend data) to avoid contention with the write path.

**Redis (ElastiCache, `cache.r7g.large`, cluster mode with 2 shards)**

Cache key taxonomy:
| Key Pattern | Content | TTL |
|---|---|---|
| `orders:portfolio:{org_id}:{role}:{filter_hash}` | Paginated order list | 60s |
| `orders:{order_id}:lifecycle` | Full lifecycle detail for one order | 30s |
| `kpis:{org_id}:executive:{period}` | KPI aggregates (OTD %, etc.) | 5 min |
| `alerts:{org_id}:recent` | Last 50 unread alerts | 10s |

Cache invalidation: the ingestion worker explicitly deletes the `orders:{order_id}:lifecycle` key and any portfolio keys for the affected `org_id` after each upsert. KPI keys expire naturally (TTL-based).

Redis Pub/Sub (separate from cache): channel `alerts:{org_id}` is used for real-time WebSocket push. All `api-service` tasks subscribe to this channel; when a message arrives, each task pushes to connected WebSocket clients for that org.

**S3 Buckets**
| Bucket | Purpose | Lifecycle Policy |
|---|---|---|
| `compassx-events` | Raw event payloads (audit trail, reprocessing) | Glacier after 90 days |
| `compassx-documents` | Client-facing documents (specs, certs) | No expiry |
| `compassx-ml-models` | Serialized XGBoost model files | Versioned |
| `compassx-exports` | CSV/PDF report exports | Expire after 7 days |

---

### 4.7 Application / API Layer

**FastAPI application structure:**
```
app/
  routers/
    orders.py          # GET /api/orders, GET /api/orders/{id}/lifecycle
    manufacturing.py   # GET /api/orders/{id}/production
    logistics.py       # GET /api/orders/{id}/shipments
    alerts.py          # GET /api/alerts, WS /api/ws/alerts
    webhooks.py        # POST /webhooks/{source}
    ai.py              # GET /api/orders/{id}/summary, POST /api/ai/ask
  services/
    lifecycle.py       # Assembles full lifecycle view from DB + cache
    ai_service.py      # summarize_order(), score_risk(), predict_delay()
    alert_service.py   # Alert generation and Pub/Sub publish
  middleware/
    auth.py            # JWT validation + RBAC
    tenant.py          # org_id scoping
  models/              # SQLAlchemy ORM models
  schemas/             # Pydantic request/response schemas
```

**WebSocket (real-time alerts):**
- Endpoint: `GET /api/ws/alerts` (upgraded to WebSocket after JWT validation)
- On connect: client authenticates via `?token=<jwt>` query param (WebSocket doesn't support headers)
- API task subscribes to Redis Pub/Sub channel for `org_id`; forwards messages to the WebSocket connection
- On disconnect: unsubscribes from Redis channel
- Heartbeat: ping/pong every 30s to detect stale connections

**AI Service module:**
- `summarize_order(order_id, audience)` — fetches full lifecycle context from DB, calls `claude-opus-4-6` with a role-specific system prompt, caches result in Redis for 10 min
- `score_risk(order_id)` — loads XGBoost model from S3 (cached in memory after first load), runs inference on current order features, calls `claude-haiku-4-5-20251001` to generate a 1-sentence explanation
- `predict_delay(order_id)` — same XGBoost model; returns probability and predicted days-at-risk
- `ask(question, org_id)` — embeddings search in pgvector (top-5 relevant records), assembles context, calls `claude-opus-4-6` with RAG prompt

---

### 4.8 Frontend Deployment

**Build and delivery:**
1. `vite build` produces a `dist/` directory of static assets (HTML, JS bundles, CSS, images)
2. GitHub Actions uploads `dist/` to S3 bucket `compassx-frontend` using `aws s3 sync`
3. CloudFront cache invalidation (`/*`) is triggered after the S3 sync completes
4. Users receive the new version on their next page load (or immediately after invalidation propagates, ~30s)

**React app architecture:**
- Single SPA served at `compassx.io`
- React Router v6: route `/portal/*` for clients, `/ops/*` for operations, `/exec/*` for executives
- Role-based routing: after authentication, JWT claims determine which route the user lands on
- React Query manages all server state: queries for order lists, lifecycle details, KPIs; mutations for alert acknowledgment
- Vite proxy in development (`/api` → `localhost:8000`) mirrors production CloudFront behavior

---

### 4.9 CI/CD Pipeline

Two independent GitHub Actions workflows, triggered on merge to `main`:

**Backend workflow:**
```
1. Run pytest (unit + integration tests against a local PostgreSQL container)
2. Run ruff (linter) + mypy (type checker)
3. Build Docker image: docker build -t compassx-api .
4. Push to Amazon ECR: compassx-api:{git-sha}
5. Register new ECS task definition revision (update image URI)
6. ECS rolling deployment: aws ecs update-service --force-new-deployment
   (replaces tasks one at a time; ALB drains connections before terminating old tasks)
7. Same steps repeated for worker-service image
```

**Frontend workflow:**
```
1. Run vitest (unit tests)
2. Run eslint + tsc --noEmit
3. vite build (production)
4. aws s3 sync dist/ s3://compassx-frontend --delete
5. aws cloudfront create-invalidation --paths "/*"
```

**Infrastructure changes (Terraform):**
- Terraform state stored in S3 backend with DynamoDB lock table
- `terraform plan` runs on every PR; output posted as PR comment
- `terraform apply` runs manually (or via a separate "infra" workflow with a required approval step) — never automatically on merge

---

### 4.10 End-to-End Data Flow Example

Tracing a MES webhook event (production work order updated) through the full stack:

```
MES System
  │  POST /webhooks/mes  (HMAC-signed JSON payload)
  ▼
ALB → api-service (FastAPI /webhooks/mes handler)
  │  Validates HMAC signature
  │  Enqueues to SQS: compassx-mes.fifo (MessageGroupId = order_id)
  │  Returns HTTP 200 in < 5ms
  ▼
SQS: compassx-mes.fifo
  ▼
worker-service (polling SQS)
  │  Deserializes RawEvent
  │  MockMESAdapter.normalize() → ProductionWorkOrder entity
  │  Reconciler.upsert() → PostgreSQL UPDATE production_work_orders
  │  Computes risk score (XGBoost inference)
  │  If risk score changed: generates alert
  │  Invalidates Redis keys: orders:{order_id}:lifecycle, orders:portfolio:*
  │  Publishes to Redis Pub/Sub: alerts:{org_id}
  │  Writes raw payload to S3: compassx-events/mes/{date}/{uuid}.json
  ▼
api-service (Redis Pub/Sub subscriber, background task)
  │  Receives alert from Redis channel
  │  Pushes JSON alert to all WebSocket connections for org_id
  ▼
React frontend (Operations Dashboard)
  │  WebSocket message received
  │  React Query cache invalidated for affected order
  │  Alert appears in live feed without page refresh
  │  Order risk badge updates in portfolio table
  ▼
User sees alert in < 2 seconds of MES event
```

---

### 4.11 Observability

**Logging:** All services use `structlog` for structured JSON logs. Every log entry includes `service`, `trace_id`, `org_id`, `order_id` (where applicable). Logs ship to CloudWatch Logs (`/ecs/api-service`, `/ecs/worker-service`).

**Metrics:** Custom CloudWatch metrics published by workers:
- `IngestLag` — time between event timestamp and DB write
- `QueueDepth` — SQS approximate message count per queue
- `CacheHitRate` — Redis hit/miss ratio
- `AIResponseLatency` — Claude API call duration per capability

**Alarms:** CloudWatch Alarms on:
- SQS DLQ message count > 0 → SNS → on-call notification
- API 5xx rate > 1% over 5 min
- RDS connection count > 80% of RDS Proxy limit
- Worker IngestLag > 5 minutes (stale data risk)

**Secrets:** All credentials (DB password, Redis auth token, Claude API key, Cognito client secret, connector HMAC secrets) stored in AWS Secrets Manager. ECS tasks reference secrets by ARN in task definitions — secrets are injected as environment variables at container startup, never baked into images.

---

## 5. AI Capabilities

AI is embedded at multiple points in the platform to move from reactive reporting to proactive intelligence. Five capabilities are planned, each with a distinct implementation approach suited to the nature of the problem.

| Capability | Approach | Input Data | Output |
|---|---|---|---|
| **Delay Prediction** | Gradient boosting model (scikit-learn / XGBoost) trained on historical order milestones and lead times | Order age, BOM completion %, production progress %, historical cycle times | Probability of on-time delivery; predicted delay in days |
| **Anomaly Detection** | Statistical process control (Z-score / CUSUM) on quality and production metrics | QMS inspection pass rates, MES throughput per work center | Anomaly flag with affected metric and deviation magnitude |
| **Lifecycle Summaries** | LLM generation via Claude API | All lifecycle entities for a given order | 2–3 sentence plain-language status update, tailored per audience (client vs. executive) |
| **Risk Scoring** | Composite score combining rule-based signals + ML model output | Delay prediction, anomaly flags, BOM shortage indicators, logistics ETAs | 0–100 risk score with breakdown of contributing factors |
| **AI Assistant** | RAG pipeline: embeddings stored in pgvector, retrieval + generation via Claude API | Lifecycle DB records, historical orders, knowledge base documents | Natural-language answers to lifecycle questions ("What is blocking order #4821?") |

---

### 5.1 Delay Prediction

**Goal:** Predict whether a specific order will be delivered on time, and if not, estimate how many days late it will be.

**Approach:** Supervised ML using gradient boosting (XGBoost). This is appropriate because the input is structured tabular data, training labels (actual vs. promised delivery date) are available from historical order records, and the model needs to run as fast inference on every ingestion event — not just on demand.

**Training:**
- Training set: all completed historical orders (sourced from ERP and lifecycle DB)
- Label: `days_late = actual_delivery_date - promised_delivery_date` (positive = late, negative = early)
- Features engineered from the lifecycle DB at each training record's point-in-time snapshot:
  - Order age in days at time of prediction
  - BOM completion % (items confirmed vs. total)
  - Count of open BOM shortage flags
  - Manufacturing progress % (work orders completed / total)
  - Days in current manufacturing stage (stall indicator)
  - QA pass rate for this order's product family (rolling 90-day)
  - Count of QA holds on this order
  - Carrier on-time performance % (rolling 90-day, from TMS history)
  - Historical avg cycle time for this product category
  - Customer region (installation distance proxy)
- Model: XGBoost regressor (`n_estimators=300, max_depth=6, learning_rate=0.05`)
- Training cadence: retrained monthly via an EventBridge-scheduled ECS task; new model artifact uploaded to `compassx-ml-models/delay-prediction/` and versioned in S3
- Evaluation: RMSE on held-out 20% test set; a new model only replaces the live model if RMSE improves

**Inference:**
- Triggered by the ingestion worker after each DB upsert for an active order
- Feature vector is computed in Python from the current DB state and passed to the loaded XGBoost model
- Model is loaded once from S3 into process memory on worker startup; reloaded automatically when a new version is uploaded (S3 event → SQS signal to workers)
- Outputs: `p_on_time` (probability 0–1), `predicted_delay_days` (float)
- Both values are written to an `order_predictions` table and used downstream by the risk scorer and the client-facing confidence indicator

**Cold start (no historical data):** For the initial deployment with mock data only, the model is trained on 6 months of synthetically generated historical orders. These are generated to reflect realistic distributions (e.g., MES delays correlate with BOM shortages). The model is replaced with a real-data-trained version as soon as production history accumulates.

---

### 5.2 Anomaly Detection

**Goal:** Detect when a manufacturing work center, a product line's quality pass rate, or a supplier's BOM delivery rate deviates meaningfully from its own historical baseline — and surface the anomaly before it impacts specific orders.

**Approach:** Statistical process control (SPC) rather than ML, because the signals being monitored are time-series metrics with well-defined historical baselines. SPC is interpretable, requires no training data, and responds within a single observation window.

**Metrics monitored (per work center / product family / supplier):**
- QA inspection pass rate (daily)
- MES throughput (units completed per shift)
- BOM item on-time delivery rate (per supplier, weekly)
- Average work order completion time (rolling 7-day)

**Detection method:**
- **CUSUM (Cumulative Sum):** Detects sustained directional drift — e.g., a work center whose throughput has been declining for 5 consecutive shifts. Sensitivity parameter `k` is set to 0.5σ; alert threshold `h` is set to 5σ accumulation.
- **Z-score spike detection:** Detects single-observation outliers — e.g., a QA pass rate that drops from 94% to 61% in one day. Alert fires when `|z| > 3`.
- Both methods run after each batch of daily metrics is ingested. Implemented in Python using `numpy`; no external ML library required.

**Anomaly record:** When an anomaly is detected, an `AnomalyRecord` is written to PostgreSQL:
```
anomaly_type         (CUSUM_DRIFT | Z_SCORE_SPIKE)
metric               (qa_pass_rate | throughput | bom_delivery_rate | ...)
entity_type          (work_center | product_family | supplier)
entity_id
deviation_magnitude  (z-score or CUSUM statistic)
baseline_value
observed_value
detected_at
affected_order_ids   (list of orders currently at this work center / using this supplier)
```

**Surfacing in the product:**
- Operations Dashboard alert feed: "Work center WC-07 throughput has declined 18% over 6 shifts (CUSUM alert). 4 active orders are assigned to this work center."
- Anomalies are an input signal to the risk scorer (Section 5.4) — any order whose current work center has an active anomaly gets a risk score boost
- `claude-haiku-4-5-20251001` generates a 1-sentence description of each anomaly for the alert feed (converts numeric deviation into plain English)

---

### 5.3 Lifecycle Summaries

**Goal:** Generate a plain-language summary of an order's current status that is appropriate for the audience — avoiding jargon for clients and including operational detail for executives.

**Approach:** Structured prompting via Claude API (`claude-opus-4-6`). The summary is not a free-form generation — it is tightly controlled by a structured prompt that provides the full current lifecycle state as context and constrains the output format.

**Context assembly:** Before calling the Claude API, `ai_service.summarize_order()` fetches the full lifecycle context from PostgreSQL and assembles a structured JSON context block:
```json
{
  "order": { "id": "ORD-4821", "product": "Industrial Compressor", "customer": "Acme Corp", "promised_date": "2025-06-03" },
  "current_stage": "manufacturing",
  "stage_statuses": [
    { "stage": "order_placement", "status": "complete", "completed_at": "2025-03-12" },
    { "stage": "bom_configuration", "status": "complete", "completed_at": "2025-03-19" },
    { "stage": "manufacturing", "status": "in_progress", "progress_pct": 68, "active_alerts": ["rework_required on sub-assembly A3"] },
    { "stage": "quality_assurance", "status": "pending" }
  ],
  "delay_prediction": { "p_on_time": 0.71, "predicted_delay_days": 3 },
  "risk_score": 42,
  "open_alerts": [{ "type": "manufacturing_hold", "description": "Rework required on sub-assembly A3", "raised_at": "..." }]
}
```

**System prompt (client audience):**
```
You are CompassX, an order status assistant. Write a 2–3 sentence update
for a client about their order. Use plain language — no internal codes,
system names, or manufacturing jargon. Focus on what the client cares
about: where their order is, when it will arrive, and whether there are
any issues they should know about. If there is a delay risk, state it
clearly and give a revised estimate. Do not speculate beyond the data provided.
```

**System prompt (executive audience):**
```
You are CompassX, an operational intelligence assistant. Write a 2–3 sentence
executive briefing on this order. Include the current stage, any active risks
with business impact, the delay probability, and the key operational factor
driving any risk. Use precise operational language. Reference specific metrics
where available.
```

**Caching and freshness:**
- Generated summaries are cached in Redis with a 10-minute TTL
- Cache is invalidated whenever the ingestion worker processes a new event for that `order_id`
- On cache miss, the summary is regenerated synchronously (typical Claude API latency: 1–3s)
- For the executive daily digest, summaries for all at-risk orders are pre-generated at 07:00 UTC by an EventBridge-scheduled task, so the dashboard loads instantly

**Output storage:** Generated summaries are stored in PostgreSQL (`order_summaries` table) with `audience`, `generated_at`, and `model_version` columns — enabling audit of what was communicated to clients and when.

---

### 5.4 Risk Scoring

**Goal:** Assign every active order a single 0–100 risk score that reflects its combined exposure across all lifecycle signals — and decompose that score into contributing factors so teams know where to focus.

**Approach:** A composite score combining rule-based signals with the ML delay prediction output. Rules are explicit and auditable; the ML component adds predictive power beyond what rules alone can capture.

**Score computation** (runs after every ingestion event for an order):

```
risk_score = weighted_sum(
  delay_prediction_component,    # weight: 0.35
  bom_risk_component,            # weight: 0.20
  quality_risk_component,        # weight: 0.20
  anomaly_component,             # weight: 0.15
  logistics_component,           # weight: 0.10
) × 100, clamped to [0, 100]
```

**Component calculation:**

| Component | Logic |
|---|---|
| `delay_prediction_component` | `1 - p_on_time` from XGBoost model (Section 5.1) |
| `bom_risk_component` | `(shortage_count / total_bom_items) + (unconfirmed_pct × 0.5)` |
| `quality_risk_component` | `(open_qa_holds / total_inspections) + (0.3 if rework_required)` |
| `anomaly_component` | `0.0` (no active anomalies) → `0.5` (anomaly at current stage) → `1.0` (critical anomaly affecting this order) |
| `logistics_component` | `0.0` (carrier OTP > 95%) → `1.0` (carrier OTP < 70% or shipment currently delayed) |

**Risk tiers:**
- 0–30: Low (green) — on track
- 31–60: Medium (amber) — monitor closely
- 61–100: High (red) — intervention required

**Score decomposition:** Each component value is stored alongside the composite score in `order_risk_scores`. This drives the "risk breakdown" tooltip in the Operations Dashboard: hovering a risk badge shows which factors are contributing most.

**LLM explanation:** After computing the score, `claude-haiku-4-5-20251001` is called with the component breakdown to generate a 1-sentence explanation:
> "Risk score of 72: primarily driven by a 29% delay probability and 3 open BOM shortages on critical path components."

This explanation is stored with the score and shown in tooltips and alert bodies. Haiku is used here (not Opus) because this runs on every ingestion event across all active orders — latency and cost must be minimized.

**Score history:** Risk scores are append-only (never updated, always inserted) in `order_risk_scores` with a timestamp. This produces a risk trajectory chart for each order — visible in the order detail view.

---

### 5.5 AI Assistant (RAG)

**Goal:** Allow operations teams to ask natural-language questions about lifecycle state without navigating dashboards — e.g., "Which orders are most at risk this week?", "What is blocking ORD-4821?", "How many orders are in QA holds right now?"

**Approach:** Retrieval-Augmented Generation (RAG) using pgvector for semantic search and `claude-opus-4-6` for generation. The assistant is grounded in live DB data — it does not rely on the LLM's parametric knowledge for facts about orders.

**Embedding generation:**
- After each lifecycle DB write, the ingestion worker generates a text representation of the updated order's lifecycle state (same JSON context used for summaries in Section 5.3)
- This text is embedded using the Anthropic `voyage-3` embedding model (1024-dimensional) via the Anthropic API
- The embedding is upserted into `lifecycle_embeddings(order_id, embedding vector(1024), text_snapshot, updated_at)`
- pgvector `ivfflat` index on the embedding column enables fast approximate nearest-neighbor search

**Query flow (per user question):**
```
User question
  │
  ▼
Embed question using voyage-3  →  1024-dimensional query vector
  │
  ▼
pgvector similarity search:
  SELECT order_id, text_snapshot
  FROM lifecycle_embeddings
  WHERE org_id = :org_id                        ← tenant isolation enforced
  ORDER BY embedding <=> :query_vector          ← cosine distance
  LIMIT 5
  │
  ▼
Fetch full live context for top-5 orders from PostgreSQL
(not just the embedding snapshot — ensures data is current)
  │
  ▼
Assemble RAG prompt:
  [System]: You are CompassX assistant. Answer only from the provided
            order context. If the answer is not in the context, say so.
            Never speculate. Cite order IDs when referencing specific orders.
  [Context]: {top-5 order lifecycle snapshots as structured text}
  [Question]: {user's question}
  │
  ▼
claude-opus-4-6 generates response
  │
  ▼
Response + cited order IDs returned to frontend
Frontend renders cited order IDs as clickable links to order detail
```

**Query routing:** Before retrieval, the question is classified by a lightweight rule-based router:
- **Aggregate question** ("How many orders are in QA?") → answered directly from a pre-computed DB query, no LLM or embedding needed
- **Specific order question** ("What is blocking ORD-4821?") → retrieval targets that order specifically (exact match by order ID, bypasses vector search)
- **Semantic/exploratory question** ("Which orders have supply chain risk?") → full vector search flow above

This routing avoids unnecessary Claude API calls for questions that can be answered deterministically.

**Scope and guardrails:**
- The assistant only has access to data within the user's `org_id` — enforced at the pgvector query level
- The system prompt explicitly prohibits speculating beyond the retrieved context
- The assistant does not have write access to any data — it is read-only
- Responses include a "data as of" timestamp drawn from the most recent `last_synced_at` across the retrieved records, so users know if the answer reflects potentially stale data

**Conversation history:** The last 6 turns of conversation are included in the prompt (as user/assistant message pairs) to support follow-up questions. History is stored in the client browser session — not persisted server-side.

---

### 5.6 Claude API Usage Summary

| Capability | Model | When Called | Volume |
|---|---|---|---|
| Lifecycle Summaries | `claude-opus-4-6` | On cache miss (user views order) + daily pre-generation | Low (per-order, cached 10 min) |
| AI Assistant responses | `claude-opus-4-6` | On each user question | Low–medium (interactive) |
| Risk score explanations | `claude-haiku-4-5-20251001` | After every ingestion event for active orders | High (automated, frequent) |
| Anomaly descriptions | `claude-haiku-4-5-20251001` | When anomaly is detected | Low (event-driven) |

Costs are managed by: caching summary outputs aggressively, routing aggregate questions away from the LLM entirely, and using Haiku for all high-volume automated calls.

---

## 6. Application Experience

### Client Portal

```
┌─────────────────────────────────────────────────────┐
│  Order #ORD-4821   Acme Corp   Industrial Compressor │
│  ─────────────────────────────────────────────────── │
│  Estimated Delivery: June 3, 2025  [High Confidence] │
│                                                       │
│  ● Order Placed          Mar 12  ✓ Complete          │
│  ● BOM & Configuration   Mar 19  ✓ Complete          │
│  ● Manufacturing         ████░░  In Progress (68%)  │
│    ⚠ Rework required on sub-assembly A3              │
│      Revised stage completion: May 14                │
│  ○ Quality Assurance     Pending                     │
│  ○ Shipping & Logistics  Pending                     │
│  ○ Installation          Scheduled: May 28           │
│  ○ Maintenance           —                           │
│                                                       │
│  [View Documents]  [Contact Account Manager]         │
└─────────────────────────────────────────────────────┘
```

Key elements:
- Progress bars for in-progress stages
- Inline alerts on the affected stage (not a separate alert panel)
- Confidence indicator on delivery date (driven by delay prediction model)
- Document access without contacting support

### Operations Dashboard

```
┌──────────────────────────────────────────────────────────────────┐
│  Active Orders (47)        Filter: [Stage ▼] [Risk ▼] [Team ▼]  │
│  ──────────────────────────────────────────────────────────────  │
│  ORDER      CUSTOMER     STAGE          RISK    ETA       TEAM   │
│  ORD-4821   Acme Corp    Manufacturing  ⚠ Med   Jun 3    MFG-2   │
│  ORD-4790   Globex       QA             🔴 High  May 22   QA-1   │
│  ORD-4756   Initech      Logistics      ✅ Low   May 18   LOG-3  │
│  ...                                                              │
│                                                                   │
│  BOTTLENECKS                   ALERTS                            │
│  Manufacturing  ████ 14 orders  🔴 ORD-4790: QA hold day 8      │
│  QA             ██   6 orders   ⚠ ORD-4831: BOM shortage        │
│  Installation   █    3 orders   ℹ ORD-4756: Delivery confirmed  │
└──────────────────────────────────────────────────────────────────┘
```

Key elements:
- Single-screen portfolio view with inline risk indicators
- Bottleneck panel updated in real-time via WebSocket
- Alert feed with one-click drill-down to order detail

### Executive Dashboard

```
┌──────────────────────────────────────────────────────────────────┐
│  DELIVERY PERFORMANCE          MANUFACTURING HEALTH              │
│  OTD:  84%  ▲ +3% (90d)       Throughput: 23 units/wk ▼ -5%    │
│  Avg Cycle Time: 47 days       Quality Pass Rate: 91.2%  ▲      │
│  Orders At Risk: 6             Open QA Holds: 3                  │
│                                                                   │
│  ────────────────────────────────────────────────────────────── │
│  REVENUE AT RISK                                                  │
│  ORD-4790  Globex      $2.1M   QA hold — potential 2wk delay    │
│  ORD-4831  Umbrella    $1.4M   BOM shortage — MFG start blocked │
│  ORD-4799  Initech     $890K   Installation rescheduled          │
│                                                                   │
│  ────────────────────────────────────────────────────────────── │
│  AI RISK SUMMARY  (generated Apr 8, 08:00)                       │
│  "2 high-value orders (Globex, Umbrella Corp) face supply chain  │
│   and quality risks that could affect $3.5M in Q2 revenue.      │
│   Manufacturing throughput declined 5% WoW — recommend review   │
│   of work center WC-07 capacity allocation."                     │
└──────────────────────────────────────────────────────────────────┘
```

Key elements:
- KPI cards with trend direction and period comparison
- Revenue-at-risk table linking operational issues to financial exposure
- Daily AI narrative summary with concrete, actionable language

---

## 7. MVP and Delivery Plan

### Guiding Principle
Deliver value in vertical slices — each phase ships a complete, usable capability for at least one user group rather than building horizontal infrastructure layers that aren't visible until late in the project.

### Phase 1 — Foundation (Weeks 1–6)

**Scope:**
- Core lifecycle data model and PostgreSQL schema
- Mock CRM + ERP connector adapters (realistic order and fulfillment data)
- FastAPI backend: order listing, lifecycle status, basic alerts
- Client Portal: order timeline with milestone status
- Operations Dashboard: portfolio view with order table and basic risk flags
- Authentication: AWS Cognito with RBAC (3 roles: `client`, `ops`, `executive`)
- Infrastructure: ECS Fargate + RDS + Redis deployed to AWS dev environment via Terraform

**Value delivered:** Clients can log in and see their order status. Ops teams have a unified order list replacing manual spreadsheet tracking.

### Phase 2 — Manufacturing & Quality (Weeks 7–10)

**Scope:**
- Mock MES + QMS connector adapters
- Manufacturing progress panel in Operations Dashboard (work order status, % complete)
- Quality inspection panel (inspection records, hold status, reopen history)
- Executive Dashboard with static KPI cards (OTD %, quality pass rate)

**Value delivered:** Operations teams have manufacturing and quality visibility in a single view. Executives have a first version of the KPI dashboard.

### Phase 3 — Logistics, Installation & Intelligence (Weeks 11–16)

**Scope:**
- Mock TMS + FSM connector adapters
- Logistics and installation stages in client timeline
- AI delay prediction model (trained on Phase 1–2 mock data)
- Claude API integration: lifecycle summaries generated for Client Portal
- Risk scoring engine surfaced in Ops and Executive dashboards
- Revenue-at-risk table in Executive Dashboard

**Value delivered:** Full lifecycle visibility end-to-end. AI features make the platform proactive rather than just informational.

### Phase 4 — Production Readiness (Weeks 17+)

**Scope:**
- Replace mock connectors with real system integrations (prioritized by data availability and business value)
- AI Assistant (RAG): natural-language lifecycle Q&A for operations teams
- Performance optimization: query tuning, cache strategy hardening
- Mobile-responsive design for client portal
- Security hardening: pen test, SOC 2 gap assessment, audit logging

**Value delivered:** Production-grade platform with real data, ready for enterprise rollout.

### Integration Sequencing Rationale

Real integrations are prioritized in this order in Phase 4:

1. **CRM + ERP** — highest data quality, most mature APIs, foundational to all other stages
2. **MES + QMS** — highest operational value for manufacturing-heavy organizations
3. **TMS** — logistics data is critical for client ETA accuracy
4. **FSM + ITSM** — post-delivery stages, lower urgency but needed for full lifecycle

### Technology Summary

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, shadcn/ui, React Query, Vite |
| Backend API | Python 3.12, FastAPI, SQLAlchemy (async), Pydantic |
| Database | PostgreSQL 16 (AWS RDS), Redis (AWS ElastiCache) |
| Message Broker | Amazon SQS + EventBridge |
| AI / ML | scikit-learn / XGBoost, Anthropic Claude API, pgvector |
| Auth | AWS Cognito (OAuth 2.0 / OIDC) |
| Infrastructure | AWS (ECS Fargate, RDS, ElastiCache, S3, SQS, API Gateway) |
| IaC | Terraform |
| CI/CD | GitHub Actions → Amazon ECR → ECS Fargate |
| Observability | AWS CloudWatch, structlog |
