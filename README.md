<div align="center">

<img src="./assets/banner.svg" width="100%"/>

<br/>

<img src="https://skillicons.dev/icons?i=react,ts,spring,java,kafka,postgres,python,docker,githubactions" />

<br/><br/>

<img src="./assets/architecture-flow.svg" width="100%"/>

<br/>

<p>
  <img src="https://img.shields.io/badge/status-in%20development-orange?style=flat-square"/>
  <img src="https://img.shields.io/badge/architecture-modular%20monolith-3457D5?style=flat-square"/>
  <img src="https://img.shields.io/badge/RBAC-project--scoped-6DB33F?style=flat-square"/>
  <img src="https://img.shields.io/badge/AI-permission--scoped%20RAG-22D3EE?style=flat-square"/>
  <img src="https://img.shields.io/badge/SaaS-multi--tenant%20(in%20progress)-8A2BE2?style=flat-square"/>
  <img src="https://img.shields.io/badge/license-all%20rights%20reserved-B22222?style=flat-square"/>
</p>

> 🔒 **This is the public showcase of OPERIVA OS.** The product source code is private while it is prepared for market — live demo and code walkthrough available on request via [LinkedIn](https://www.linkedin.com/in/aditya-patil-279807251).

**[Overview](#-overview)** · **[Architecture](#-architecture)** · **[AI Assistant](#-ai-assistant-architecture)** · **[Tech Stack](#-tech-stack)** · **[Roadmap](#-development-roadmap)** · **[Run Locally](#-running-locally)**

</div>

---

## 🚀 Overview

[#-overview](#-overview)

**OPERIVA OS** replaces scattered email chains, verbal approvals, and disconnected spreadsheets with one auditable digital record of a project's entire lifecycle — intake, requirement capture, budgeting, multi-level approval, execution, and closure.

OPERIVA OS is being built as a **multi-tenant SaaS**: one shared installation serving many companies, each fully isolated and configured for its own country, currency and workflows.

The engine is **domain-agnostic**: the same workflow core that runs a corporate project pipeline can run a law firm's matter-approval process or a government permit workflow — only the labels change.

**Built for:** 🏢 Corporates · ⚖️ Law firms · 🏛️ Government bodies · 🤝 NGOs & SMBs

---

## ✨ Features

[#-features](#-features)

- Auto-coded project containers — no manual numbering
- CEO/Board feasibility gate with auto-generated client confirmation
- Chronological, permanent meeting/progress log per project
- Requirements & budget as **versioned, append-only** revision chains — never overwritten
- Generic, configuration-driven **Approval Chain engine** (Manager → Finance → CEO/Board)
- Three decision outcomes per level: approve, reject, request clarification (with loop-back)
- **SLA-based auto-escalation** — no silently-ignored approvals
- Strict **project-scoped RBAC** — a Manager sees only their assigned projects
- Cross-department request routing (budget / asset / HR tickets)
- Embedded, **role-aware AI assistant** scoped to exactly what the asker is permitted to see
- Full **append-only audit trail** on every state transition
- Event-driven core (Kafka) — async fan-out to notifications, audit log, and AI ingestion
- **Transactional outbox** + **idempotent consumers** + **dead-letter queue** — no lost or duplicated events
- Teams-style **real-time chat** over WebSocket — presence, typing, read receipts, reactions, attachments, task clarification threads
- Notifications for every workflow step, with sound
- Security: JWT with token revocation, **TOTP two-factor login**, CAPTCHA, OTP password reset, rate limiting
- Role dashboards for CEO, VP, Manager, HR, Finance, IT and Employee
- 🚧 **Multi-tenant SaaS** (company isolation + platform-owner console) and **PostgreSQL + pgvector** hybrid AI search — in progress

---

## 🏗 Architecture

[#-architecture](#-architecture)

```mermaid
flowchart TB
    UI["🖥️ React + TypeScript SPA<br/>role-based dashboards · Teams-style chat · AI widget"]
    GW["🔐 Security Filter Chain · JWT + TOTP 2FA · RBAC Resolver<br/>(tenant, user, project, role) permission context"]

    subgraph Services["Layer 3 — Application Services (Spring Boot)"]
        direction LR
        LC["Project Lifecycle Svc"]
        WF["Workflow / Approval Engine"]
        RB["Requirements & Budget Svc"]
        MT["Meeting / Progress Log Svc"]
        TK["Task Svc"]
        NT["Notification Svc"]
        AL["Audit Log Svc"]
        CH["Chat · Presence Svc<br/>(WebSocket)"]
        OB["Outbox Relay<br/>+ SLA Escalation Worker"]
    end

    BUS{{"🔁 Event Bus — Kafka<br/>project.state.changed · approval.granted/rejected · sla.breached<br/>idempotent consumers · dead-letter topics"}}

    AI["🧠 AI Assistant Service (FastAPI)<br/>Permission-Scoped RAG · self-hosted LLM (Ollama)"]

    subgraph Data["Layer 6 — Data Layer"]
        direction LR
        PG[("PostgreSQL 16<br/>system of record · Flyway · outbox table")]
        RD[("Redis<br/>permission / session cache (planned)")]
        VD[("Vector store<br/>Chroma → PostgreSQL + pgvector")]
    end

    UI -->|REST| GW
    GW --> LC & WF & RB & MT & TK
    LC & WF & RB & MT & TK -->|write event in same tx| PG
    OB -->|relay outbox| BUS
    UI <-->|WebSocket| CH
    BUS -->|consume| NT
    BUS -->|consume| AL
    BUS -->|consume| AI
    AI <-->|filtered retrieval| VD
    LC & WF & RB & MT & TK <--> PG
    GW <--> RD
    CH <--> PG
    AL --> PG

    style UI fill:#3457D5,color:#fff
    style GW fill:#10192E,color:#fff
    style BUS fill:#231F20,color:#fff
    style AI fill:#6DB33F,color:#fff
```

> Synchronous calls (REST) flow top-to-bottom through the Gateway and Services. State changes and their events are written **in the same database transaction** (transactional outbox); the relay publishes them to Kafka, which fans out to Notifications, Audit Log, and the AI ingestion pipeline. Consumers record processed event IDs (idempotent) and failing messages go to a dead-letter topic that admins can inspect and replay. Real-time chat and presence run over WebSocket with short-lived ticket auth.

### Reliability & event design

| Problem | How OPERIVA OS handles it |
|---|---|
| DB commit succeeds but Kafka publish fails (dual write) | **Transactional outbox** + relay with backoff; failed rows can be re-queued |
| Kafka redelivers a message | **Idempotent consumers** (`processed_events`) skip duplicates |
| A message keeps failing | **Dead-letter topics** with admin inspect / replay endpoints |
| Approvals silently ignored | **SLA escalation worker** escalates overdue steps and emits `sla.breached` |
| Schema drift | **Flyway**-versioned migrations as the single source of truth |

### 🏢 Multi-tenant SaaS (in progress)

```mermaid
flowchart LR
    OWN["👑 Operiva Administrator<br/>platform-owner console"] --> T1["🏢 Company A<br/>own users · roles · config"]
    OWN --> T2["🏢 Company B<br/>own users · roles · config"]
    T1 & T2 --> APP["Shared OPERIVA OS installation<br/>tenant resolved on every request"]
    APP --> DB[("PostgreSQL<br/>company-scoped rows on every table")]
    APP --> AIX["AI retrieval + caches<br/>keyed per tenant"]

    style OWN fill:#3457D5,color:#fff
    style APP fill:#10192E,color:#fff
```

- Every table and query is **company-scoped**; no cross-company reads, cached answers or notifications
- Country, currency, tax, limits and workflows are **configured per company** — nothing hard-coded
- The platform owner creates, configures and controls customer companies from a dedicated console

### Full end-to-end workflow — intake to closure

```mermaid
flowchart TD
    A(["👤 Client Inquiry"]) --> B["Create Project Container<br/>auto-code · state = INQUIRY"]
    B --> C{"CEO Reviews<br/>Slot / Feasibility"}
    C -->|Not feasible| D["Notify Client: Not Feasible"]
    C -->|Feasible| E["Auto-generate & Send Confirmation"]
    E --> F["Meetings Logged<br/>(Progress Section)"]
    F -.->|repeats per meeting| F
    F --> G["Requirements & Budget Captured<br/>versioned · appended"]
    G --> H{"Approval Chain:<br/>CEO / Board"}
    H -->|Rejected| I(["Project Closed"])
    H -->|Clarification requested| G
    H -->|No decision in SLA| H
    H -->|Approved| J["Manager Workspace Opens<br/>Assigns Tasks"]
    J --> K["Employees Execute Tasks"]
    K --> L["Budget / Asset / HR<br/>Ticket Routed"]
    L -.->|fulfilled / response| K
    K --> M["Submit Proposal / Deliverable"]
    M --> N{"Manager Reviews<br/>Proposal"}
    N -->|Rework / clarification| M
    N -->|Approved| O["Final Budget Sanction<br/>(if required)"]
    O --> P(["Project State:<br/>IN_PROGRESS → REVIEW → COMPLETED"])

    style A fill:#3457D5,color:#fff
    style D fill:#C0392B,color:#fff
    style I fill:#C0392B,color:#fff
    style E fill:#2E8B57,color:#fff
    style J fill:#2E8B57,color:#fff
    style O fill:#2E8B57,color:#fff
    style P fill:#10192E,color:#fff
```

> Every step across the whole flow publishes to Kafka and is written to `AuditLog` with actor + timestamp — queryable by the role-scoped AI Assistant, subject to the same RBAC boundary as the UI. `ON_HOLD` is reachable from any state.

---

## ⚙ Components

[#-components](#-components)

| Service | Responsibility |
|---|---|
| **Project Lifecycle Svc** | Owns the `Project` entity and its state machine; auto-generated project codes; CEO feasibility review |
| **Workflow / Approval Engine** | A **generic** `ApprovalChain` + `ApprovalStep` model — configurable levels, per-level approver role, per-chain SLA. Reused as-is for budget, asset, and HR requests |
| **Requirements & Budget Svc** | Append-only revision chains (self-referencing `supersedes_id`) with a materialized "current" pointer — full history, no overwrites |
| **Meeting / Progress Log Svc** | Chronological, searchable, permanent log of every meeting tied to a project |
| **AI Assistant Svc** | Permission-scoped RAG — retrieval happens inside the user's permission boundary before the model ever sees the data |
| **Notification / Audit Log Svc** | Consumes Kafka events to dispatch notifications and write an immutable audit record of every transition |
| **Chat · Presence Svc** | Teams-style real-time chat over WebSocket: direct/group chats, presence, typing, read receipts, reactions, pinning, attachments, task clarification → resolve cycle |
| **Outbox Relay / SLA Worker** | Publishes outbox rows to Kafka with backoff; escalates approval steps that pass their SLA |
| **Security** | JWT with token versioning, TOTP two-factor login, CAPTCHA, OTP password reset, login and AI rate limiting, WebSocket ticket auth |
| **Dashboards** | Role-specific dashboards (CEO, VP, Manager, HR, Finance, IT, Employee) with drill-down down the reporting line |

---

## 🧠 AI Assistant Architecture

[#-ai-assistant-architecture](#-ai-assistant-architecture)

```mermaid
flowchart LR
    Q(["💬 User Query<br/>'Status of Project X?'"]) --> P["Permission Context<br/>(user_id, project_ids, role)<br/>from RBAC resolver"]
    P --> V["Filtered Vector Search<br/>query embedding + metadata filter:<br/>project_id IN allowed_ids · category IN allowed<br/>(OPERATIONAL · FINANCIAL · REQUIREMENTS · TASKS)"]
    V --> G["AI Model Generation<br/>only permitted chunks<br/>reach the prompt"]
    G --> R(["✅ Scoped Answer"])

    style Q fill:#3457D5,color:#fff
    style V fill:#6DB33F,color:#fff
    style R fill:#10192E,color:#fff
```

**Guardrails**

1. The retrieval filter is enforced at the **vector-store query level**, never via prompt instructions.
2. The AI model is never given a tool that can bypass the filter (e.g. raw SQL access).
3. Every AI answer logs which chunks were retrieved, for audit purposes.

**Current implementation:** a Python **FastAPI** service with **ChromaDB** retrieval and a self-hosted **Ollama** model (temperature 0). The backend builds the scope from RBAC and sends it with every question; the index re-syncs from PostgreSQL through a read-only role; answers are cached per question + full scope + asker. **In progress:** moving retrieval to **PostgreSQL + pgvector with full-text hybrid search** and a local reranker, with permissions enforced in SQL and a fixed retrieval test set run before every release.

**Ingestion pipeline** runs continuously via Kafka consumers — meeting notes, requirement revisions, task updates, and approval decisions are chunked and embedded as they're created, each tagged with `project_id`, `allowed_roles`, `entity_type`, `created_at`.

| Component | Free / Open-Source | Paid Alternative |
|---|---|---|
| Orchestration | LangChain / LlamaIndex | Managed RAG-as-a-service |
| Embeddings | Sentence-Transformers, BGE (local) | OpenAI / Cohere embedding APIs |
| Vector DB | FAISS, Chroma, Qdrant (self-hosted) | Pinecone, Weaviate Cloud |
| Generation model | Llama 3 / Mistral / Phi-3 via Ollama | Claude API / GPT API |
| Reranking (optional) | Open-source cross-encoders | Cohere Rerank API |

> Recommended path: validate the RBAC-scoped retrieval architecture against the **free, self-hosted stack** first (LangChain + Sentence-Transformers + Chroma + Ollama); swap in a hosted API only once answer quality needs to improve for production.

---

## 🔁 Budget Request — Sequence Diagram

[#-budget-request--sequence-diagram](#-budget-request--sequence-diagram)

```mermaid
sequenceDiagram
    actor Employee
    participant WF as Workflow Engine
    participant Manager
    participant Finance as Finance/Sales
    actor CEO

    Employee->>WF: raise BudgetRequest
    WF->>Manager: create ApprovalStep L1 (notify)
    Manager-->>WF: decision: approve
    WF->>Finance: create ApprovalStep L2 (notify)
    Finance-->>WF: decision: approve, budget available
    WF->>CEO: create ApprovalStep L3 — final sanction (notify)
    CEO-->>WF: decision: approved
    WF->>Employee: publish budget.revised event → notify

    Note over WF,CEO: Each decision is written with an outbox event in the same transaction, then relayed to Kafka - AuditLog records it, Notification dispatches in-app/email, SLA worker auto-escalates on timeout
```

---

## 🔐 Role-Based Access Model

[#-role-based-access-model](#-role-based-access-model)

| Role | Visibility Scope | Cannot See |
|---|---|---|
| 👑 **CEO / Board** | All projects, budgets, meetings, approvals, org-wide analytics | — |
| 🧭 **VP** | Projects they own or oversee, their managers' teams and workloads down the reporting line | Projects outside their line |
| 🧑‍💼 **Manager** | Only assigned projects — requirements, tasks, progress | Other managers' projects, board notes, org-wide budgets |
| 🧑‍💻 **Employee** | Only their own assigned tasks + relevant requirements | Project budget, other employees' tasks, meeting/approval history |
| 💰 **Finance / Sales / HR / IT** | Only the specific request ticket routed to them; HR also manages reporting lines | Full project context, other departments' tickets |

Enforced at two points (and, with multi-tenancy, always within the caller's own company): **(1)** the API Gateway resolves and attaches the permission context to every request, and **(2)** the AI Assistant's retrieval layer applies the same filter before returning any content to the LLM.

---

## 🧬 Core Schema & State Machines

[#-core-schema--state-machines](#-core-schema--state-machines)

```mermaid
erDiagram
    USER ||--o{ PROJECTROLE : has
    PROJECT ||--o{ PROJECTROLE : "scoped by"
    PROJECT ||--o{ MEETING : logs
    PROJECT ||--o{ REQREVISION : "revision chain"
    PROJECT ||--o{ BUDGETREVISION : "revision chain"
    PROJECT ||--o{ TASK : contains
    PROJECT ||--o{ APPROVALCHAIN : triggers
    APPROVALCHAIN ||--o{ APPROVALSTEP : "ordered levels"
    PROJECT ||--o{ REQUESTTICKET : routes
    REQUESTTICKET }o--|| APPROVALCHAIN : uses
    USER ||--o{ NOTIFICATION : receives
    PROJECT ||--o{ AUDITLOG : "logs to"

    USER {
        uuid id PK
        string name
        string email
        string designation
    }
    PROJECT {
        uuid id PK
        string code
        string title
        enum state
        uuid manager_id FK
    }
    APPROVALCHAIN {
        uuid id PK
        enum request_type
        enum status
        int sla_hours
    }
    REQREVISION {
        uuid id PK
        int version_no
        uuid supersedes_id FK
    }
    BUDGETREVISION {
        uuid id PK
        decimal amount
        uuid supersedes_id FK
    }
```

**Project lifecycle**

```mermaid
stateDiagram-v2
    [*] --> INQUIRY
    INQUIRY --> DISCUSSION
    DISCUSSION --> REQUIREMENTS_DEFINED
    REQUIREMENTS_DEFINED --> BUDGET_PROPOSED
    BUDGET_PROPOSED --> UNDER_APPROVAL
    UNDER_APPROVAL --> APPROVED
    UNDER_APPROVAL --> REJECTED
    APPROVED --> IN_PROGRESS
    IN_PROGRESS --> REVIEW
    REVIEW --> COMPLETED
    COMPLETED --> [*]
    REJECTED --> [*]

    state "Any State" as ANY
    ANY --> ON_HOLD
    ON_HOLD --> ANY
```

**Generic approval chain** (reused for Budget / Asset / Clarification requests)

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> PENDING_LEVEL_1
    PENDING_LEVEL_1 --> PENDING_LEVEL_2N : approve
    PENDING_LEVEL_2N --> APPROVED : approve
    PENDING_LEVEL_1 --> REJECTED : reject
    PENDING_LEVEL_2N --> REJECTED : reject
    PENDING_LEVEL_1 --> CLARIFICATION_REQUESTED : clarify
    PENDING_LEVEL_2N --> CLARIFICATION_REQUESTED : clarify
    CLARIFICATION_REQUESTED --> PENDING_LEVEL_1 : loops back
    PENDING_LEVEL_1 --> PENDING_LEVEL_1 : SLA timeout → escalate
    APPROVED --> [*]
    REJECTED --> [*]
```

- `ReqRevision` and `BudgetRevision` are **append-only, self-referencing** (`supersedes_id`) — history is never lost.
- `ApprovalChain` + `ApprovalStep` is the **one generic engine** reused by `RequestTicket` for every request type.

---

## 🛠 Tech Stack

[#-tech-stack](#-tech-stack)

<div align="center">
<img src="https://skillicons.dev/icons?i=react,ts,spring,java,kafka,postgres,python,docker,githubactions" />
</div>
<br/>

| Layer | Technology | Notes |
|---|---|---|
| Frontend framework | React 18 + Vite | First Angular client archived in `frontend-angular/` |
| Frontend language | TypeScript 5.x | Strict mode enabled |
| Frontend styling | Tailwind CSS + design tokens | See [Design System](#-frontend-design-system) |
| Backend framework | Spring Boot 4.x (Security, Data JPA, WebSocket, Mail) | Modular monolith |
| Backend language | Java 21 (LTS) | |
| API style | REST (OpenAPI 3 via springdoc) + WebSocket | Sync client calls + real-time chat/presence |
| Event streaming | Apache Kafka 3.8 | Transactional outbox, idempotent consumers, dead-letter topics |
| Relational database | PostgreSQL 16 + Flyway | System of record, append-only revisions, versioned migrations |
| Cache | Redis 7.x (planned) | Permission-context & session caching |
| Vector database | ChromaDB today → PostgreSQL + pgvector (in progress) | Hybrid vector + full-text search |
| AI service | Python FastAPI | Scoped retrieval, prompt assembly, answer cache |
| AI model | Llama 3 via Ollama (self-hosted) | No project data leaves the company |
| Containerization | Docker Compose | Postgres, Kafka, Mailpit, Ollama |
| Security | JWT · TOTP 2FA · CAPTCHA · OTP reset | Token versioning for revocation |
| Orchestration (prod) | Kubernetes (optional at scale) | Not required for MVP |
| CI/CD | GitHub Actions | Test → build → containerize → deploy |
| Cloud hosting | Azure App Service / AKS, or AWS/GCP equivalent | Vendor-agnostic |

---

## 🗂 Project Structure

[#-project-structure](#-project-structure)

```
operiva-os
│
├── frontend                     # React + TypeScript + Vite SPA
│   ├── pages                    # CEO / VP / Manager / HR / Finance / IT / Employee dashboards,
│   │                            # Projects, Approval Center, Kanban, Org Chart, Chat, Tickets
│   ├── teams*                   # Teams-style chat components
│   └── api / context / hooks    # API client, auth context, realtime hooks
│
├── frontend-angular             # Archived first Angular client
│
├── backend                      # Spring Boot modular monolith
│   ├── controller               # Projects, approvals, tasks, chat, HR, dashboards, AI, admin
│   ├── service                  # Lifecycle, ApprovalChain engine, outbox relay, SLA worker, ...
│   ├── security                 # JWT, TOTP 2FA, rate limiters, WebSocket tickets
│   ├── realtime                 # WebSocket hub, presence
│   └── resources/db/migration   # Flyway migrations
│
├── ai-assistant                 # Python FastAPI, permission-scoped RAG (Chroma → pgvector)
│
├── infra
│   ├── docker-compose.yml       # Postgres, Kafka, Mailpit, Ollama
│   └── docker-compose.gpu.yml   # GPU variant for the LLM
│
├── .github/workflows            # Backend CI
│
└── docs                           # ER diagram, state machines, RBAC model
```

---

## 🎨 Frontend Design System

[#-frontend-design-system](#-frontend-design-system)

- **Palette:** deep navy ink `#10192E` (sidebar/header), soft canvas `#F5F7FB` (background), royal-blue `#3457D5` (primary actions)
- **State colors:** blue (early stage) · amber (under approval) · green (approved/complete) · red (rejected) · purple (on-hold/clarification)
- **Typography:** Fraunces (serif) for titles/wordmark, Inter (sans) for functional UI
- **Signature element — the Lifecycle Rail:** a segmented progress bar showing a project's exact state-machine position, used on dashboard cards, table rows, and the detail header
- **Responsive:** sidebar → slide-in drawer below 1024px; tables → stacked cards below 860px; stat grids reflow 4 → 1 column
- **Accessibility:** visible keyboard focus rings, `prefers-reduced-motion` respected

| Screen | What it shows |
|---|---|
| Login | Minimal, enterprise-styled sign-in |
| Dashboard | Role-aware stat cards, "projects in motion" feed, activity panel |
| Project List | Filterable by lifecycle state; table (desktop) / cards (mobile) |
| Project Detail | Overview · Progress · Requirements & Budget (RBAC-restricted) · Tasks · Approvals |

---

## 🐳 Running Locally

[#-running-locally](#-running-locally)

```bash
# Source code is private — commands shown for reference
cd OPERIVAOS/infra && docker compose up -d      # Postgres, Kafka, Mailpit, Ollama
cd ../backend && ./mvnw spring-boot:run        # Windows: .\mvnw.cmd spring-boot:run
cd ../frontend && npm install && npm run dev
cd ../ai-assistant && uvicorn main:app --host 127.0.0.1 --port 8000
```

`docker-compose.yml` brings up the infrastructure — **PostgreSQL, Kafka, Mailpit (dev mail) and Ollama** — and the app processes run alongside it:

```mermaid
flowchart LR
    FE["frontend<br/>react + vite"]
    BE["backend<br/>spring boot (modular monolith)"]
    AI["ai-assistant<br/>python fastapi"]
    PG[("postgres 16")]
    KF{{"kafka 3.8 (KRaft)"}}
    MP["mailpit<br/>dev mail"]
    OL["ollama<br/>self-hosted LLM"]
    VD[("vector store<br/>chroma → pgvector")]

    FE -->|REST + WebSocket| BE
    BE --> PG
    BE --> KF
    BE --> MP
    BE --> AI
    AI --> VD
    AI --> OL
    AI -->|read-only role| PG
```

Each backend Dockerfile is **multi-stage**: a build stage with the full SDK, and a slim `eclipse-temurin:21-jre-alpine` runtime — no build tools, no source code, minimal attack surface in the final image.

---

## 📊 Observability

[#-observability](#-observability)

Every `AuditLog` write and every AI retrieval is queryable alongside infrastructure metrics from day one.

```mermaid
flowchart LR
    A["OPERIVA services"] --> B["Centralized Logging<br/>+ AuditLog stream"]
    B --> C["Metrics<br/>Azure Monitor / Prometheus"]
    C --> D["Dashboards<br/>Grafana"]

    style A fill:#3457D5,color:#fff
    style D fill:#10192E,color:#fff
```

- **Secrets:** managed secret store (Azure Key Vault or equivalent) — never committed to source control, never baked into images.
- **Database:** managed PostgreSQL with automated backups and point-in-time recovery.
- Architecture is intentionally **cloud-vendor-agnostic** — every component has a direct equivalent on AWS or GCP.

---

## 🔁 CI/CD Pipeline

[#-cicd-pipeline](#-cicd-pipeline)

```mermaid
flowchart LR
    A["Push to main / PR"] --> B["Lint + Unit Tests"]
    B --> C["Build<br/>Maven / ng build"]
    C --> D["Docker Build<br/>tag: git SHA"]
    D --> E["Push to Registry"]
    E --> F["Deploy: staging"]
    F -->|smoke tests pass| G["Deploy: production"]

    style A fill:#3457D5,color:#fff
    style G fill:#2E8B57,color:#fff
```

- **Today:** GitHub Actions builds and runs the backend test suite on every push and pull request to `develop` and `main`; feature branches → PR → `develop` → release merge to `main`.
- A failing test blocks the merge.
- Images are tagged with the **Git commit SHA**, never overwriting `latest`.
- Production deploys require staging smoke tests to pass first; DB migrations run as a separate, reviewed step — never bundled silently into deploy.

---

## 🗺 Development Roadmap

[#-development-roadmap](#-development-roadmap)

10 weeks to a launch-ready MVP with a small, focused team (2–4 engineers).

```mermaid
gantt
    title OPERIVA OS — 10 Week MVP Roadmap
    dateFormat  YYYY-MM-DD
    axisFormat  W%W
    section Backend
    P0 Discovery & Design        :p0, 2026-01-05, 7d
    P1 Core Lifecycle Engine     :p1, after p0, 14d
    P2 RBAC & Auth Layer         :p2, after p1, 7d
    P3 Workflow / Approval Engine:p3, after p2, 14d
    P4 Events & Notifications    :p4, after p3, 7d
    P5 AI Assistant (RAG)        :p5, after p4, 7d
    section Frontend
    P6 Frontend Build            :p6, 2026-01-19, 49d
    section Release
    P7 Docker / CI/CD            :p7, after p5, 7d
    P8 Testing & Launch          :p8, after p7, 7d
```

| Phase | Name | Duration | Primary Deliverable |
|---|---|---|---|
| 0 | Discovery & System Design | Week 1 | Finalized schema, API contracts, RBAC model |
| 1 | Core Lifecycle Engine | Week 2–3 | Project entity, state machine, meetings, versioned requirements/budget |
| 2 | RBAC & Auth Layer | Week 4 | JWT auth, project-scoped permission resolver |
| 3 | Workflow / Approval Engine | Week 5–6 | Generic ApprovalChain engine |
| 4 | Event Layer & Notifications | Week 6–7 | Kafka topics, SLA-escalation worker, dispatch |
| 5 | AI Assistant (RAG) | Week 7–8 | Role-scoped retrieval pipeline, chat UI |
| 6 | Frontend Build | Week 3–9 (parallel) | All role-based screens, fully responsive |
| 7 | Docker, CI/CD & Deployment | Week 9 | Containerized services, automated pipeline |
| 8 | Testing, Hardening & Launch | Week 10 | End-to-end pass, production launch |

**Progress so far**

| Status | Area |
|---|---|
| ✅ Done | Real auth, project-scoped RBAC, approval chains with history, SLA escalation |
| ✅ Done | Kafka transactional outbox, idempotent consumers, dead-letter topics |
| ✅ Done | Real-time chat, presence, notifications, attachments |
| ✅ Done | 2FA, CAPTCHA, password reset, rate limiting |
| ✅ Done | Permission-scoped AI assistant; role dashboards for all 7 roles |
| 🚧 In progress | **Multi-tenant SaaS** — company isolation + platform-owner console |
| 🚧 In progress | **PostgreSQL + pgvector** hybrid AI search; project documents |
| 🔜 Next | Voice / video calls, meeting AI notes, mobile app |

---

## ⚠️ Risks & Mitigations

[#️-risks--mitigations](#️-risks--mitigations)

| Risk | Mitigation |
|---|---|
| RBAC modeled as simple global roles instead of project-scoped | Freeze the `(user, project, role)` model in Phase 0 before dependent code exists |
| Budget/requirements stored as mutable fields | Append-only revision tables from day one, never retrofitted |
| AI retrieval not scoped at the vector-store level | Enforce the permission filter in the vector query itself, never via prompts alone |
| All modules attempted simultaneously | Strict phase boundaries with defined exit criteria |
| Notification spam from naive SLA reminders | Escalate to the next level on timeout rather than re-pinging the same approver |
| Premature microservice split | Modular monolith first; extract only services whose load genuinely diverges |

---

## 💡 Design Principles

[#-design-principles](#-design-principles)

- Single source of truth for the entire project lifecycle
- Full, append-only audit trail — strong fit for regulated sectors
- One generic workflow engine reused across every approval type
- Project-scoped RBAC by design, not by convention
- Permission-scoped AI retrieval, never prompt-only guardrails
- Domain-agnostic core — usable well beyond one industry
- Ship in bounded phases — a working system exists early
- Reliable events by design: outbox, idempotency and dead-letter handling
- Multi-tenant from the data model up — no company data ever crosses a tenant boundary

---

## 📌 Applicable Domains

[#-applicable-domains](#-applicable-domains)

<div align="center">
<table>
<tr>
<td align="center" width="140"><span style="font-size:32px">🏢</span><br/><sub><b>Corporates</b></sub></td>
<td align="center" width="140"><span style="font-size:32px">🏛️</span><br/><sub><b>Government</b></sub></td>
<td align="center" width="140"><span style="font-size:32px">⚖️</span><br/><sub><b>Law Firms</b></sub></td>
<td align="center" width="140"><span style="font-size:32px">🤝</span><br/><sub><b>NGOs</b></sub></td>
<td align="center" width="140"><span style="font-size:32px">🏪</span><br/><sub><b>SMBs</b></sub></td>
</tr>
</table>
</div>

---

## 📄 License

[#-license](#-license)

**All rights reserved** — see [`LICENSE`](./LICENSE). This repository is for display only; no part of OPERIVA OS may be copied, modified or distributed without written permission.

<div align="center">
<img src="./assets/footer.svg" width="100%"/>
</div>
