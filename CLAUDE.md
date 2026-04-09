# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Overview

An organization manufactures complex enterprise products for clients. Today, the process from order placement through manufacturing, delivery, installation, and maintenance spans multiple systems and teams. 


Because these systems are fragmented, there is no unified view of the lifecycle of a customer order. Different teams rely on separate tools and manual coordination to understand the status of an order or product. 


Leadership would like to create a centralized lifecycle visibility platform that overlays existing systems of record and provides clear insights for multiple stakeholders.  The goal of the platform is to provide transparency and operational intelligence across the full product lifecycle.


The solution should provide visibility across the following stages: 
1. Client order placement 
2. Product configuration and bill of materials processing 
3. Manufacturing and production progress 
4. Quality assurance and inspection 
5. Shipping and delivery logistics 
6. Product installation and setup 
7. Ongoing maintenance and service lifecycle 


## Stakeholders
The application should support multiple user groups. 

- Clients: Clients should be able to see the status of their orders and products across the lifecycle. They should have visibility into milestones, delivery timelines, and any issues that may affect delivery or installation. 

- Operations Teams: Internal teams responsible for manufacturing, supply chain, logistics, installation, and support need operational visibility into order progress, dependencies, delays, and risks. 

- Executive Leadership: Executives should have high-level visibility into overall delivery health, manufacturing performance, quality trends, and potential risks to revenue or customer satisfaction. 

## Technical Challenges
The lifecycle data currently lives across multiple systems of record, such as:
- CRM systems for client and order information
- ERP systems for order management and fulfillment
- Product lifecycle or engineering systems for bill of materials
- Manufacturing execution systems for production tracking
- Quality management systems for inspection and compliance
- Logistics and shipping systems
- Field service platforms for installation and maintenance
- Support or ticketing platforms for service requests
  
The new platform must pull data from these systems and present a unified view of the lifecycle.


---

# Solution Proposal

See [SOLUTION_PROPOSAL.md](SOLUTION_PROPOSAL.md) for the complete proposal covering business outcomes, product vision, data strategy, architecture, AI capabilities, UX designs, and MVP delivery plan.

---

# Technology Stack

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

---

# Architecture

**Key decisions:**
- All integration events route through Amazon SQS before processing (decouples ingestion from processing)
- `Order` is the primary lifecycle container — all entities link by `order_id`
- Every entity carries `source_system`, `external_id`, `last_synced_at`, and `data_confidence` fields
- Source system integrations use mock adapters initially; each implements a common `ConnectorAdapter` interface so mocks swap for real connectors without changing core logic
- Three role-scoped frontends: Client Portal, Operations Dashboard, Executive Dashboard
- AI module exposes internal interfaces: `summarize_order()`, `score_risk()`, `predict_delay()`
- Claude API: `claude-opus-4-6` for summaries/assistant; `claude-haiku-4-5-20251001` for batch scoring