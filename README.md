# FlowCore — Workflow Engine for Enterprise Approval Flows

A code-first workflow engine for orchestrating multi-stage approval processes — built with FastAPI, Next.js 15, and PostgreSQL. First-class support for parallel branches, decision gates, ERP integration, and immutable audit trails.

> **Note:** This repository documents the architecture and engineering patterns of a production system. The application source is kept in a separate private repository.

---

## Why this exists

Off-the-shelf workflow tools fall into two camps: heavy enterprise BPM suites (expensive, slow to change, vendor lock-in) or no-code form builders (limited integration, weak audit, no real extensibility). FlowCore demonstrates a third path — a small, fast, code-first engine that:

- Plugs cleanly into existing ERP and identity systems
- Treats every state transition as an immutable, auditable event
- Supports parallel branches and approval gates as first-class primitives
- Adds new workflow modules in days, not months
- Costs nothing per seat

In production, the engine runs multiple modules including non-conformance management (RNC) and HR approval flows. A reference implementation — a New Product Development pipeline with 8 stages, 14 specialized forms, and a 4-decision-maker approval gate — exercises the full feature set end-to-end (43 e2e assertions passing).

---

<img width="1905" height="917" alt="2026-05-04 13_46_50-Configurações" src="https://github.com/user-attachments/assets/4e67f873-a271-43cb-951d-d7652e7166cf" />
<img width="1900" height="911" alt="2026-05-04 13_47_31-Configurações" src="https://github.com/user-attachments/assets/49abc689-e9d3-41ad-a5e0-db5726abf358" />



## Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI · SQLAlchemy 2.0 (async) · asyncpg |
| Database | PostgreSQL 15 |
| Auth | JWT (python-jose) · argon2id (argon2-cffi) |
| Object storage | MinIO (S3-compatible) with presigned URLs |
| PDF generation | Jinja2 templates → xhtml2pdf |
| Email | aiosmtplib (SMTP, async) |
| Frontend | Next.js 15 (App Router) · TypeScript · ShadCN UI · Tailwind |
| Forms | react-hook-form + Zod |
| Containers | Docker Compose — non-root, dropped capabilities, read-only rootfs |

---

## Architecture

```
┌─────────────────┐      HTTP       ┌──────────────────┐
│  Next.js 15     │ ──────────────► │  FastAPI         │
│  App Router     │                 │  async workers   │
└─────────────────┘                 └────────┬─────────┘
                                             │
                          ┌──────────────────┼──────────────────┐
                          ▼                  ▼                  ▼
                   ┌─────────────┐   ┌──────────────┐   ┌──────────────┐
                   │ PostgreSQL  │   │    MinIO     │   │   Identity   │
                   │  workflow   │   │  attachments │   │    service   │
                   │             │   │              │   │  (external)  │
                   └─────────────┘   └──────────────┘   └──────────────┘
```

The engine is **stateless at the API layer**. All state lives in Postgres; all binary content lives in MinIO; identity is delegated to an external service. This means horizontal scaling is just spinning up more API workers behind a load balancer — no sticky sessions, no session store.

---

## Engine concepts

### State machine

A workflow is a sequence of stages. Each stage is one of three types:

- **Sequential** — one task, one owner, completes before the next stage opens
- **Parallel group** — multiple tasks running concurrently; the stage advances only when *every* task in the group is complete
- **Decision gate** — multiple decision-makers; advances based on configurable consensus rules (`all_approve`, `any_reject_terminates`, `any_standby_pauses`, etc.)

Stage configuration is data, not code. New workflow modules are added without changing the engine.

### Immutable history

Every state transition writes a row to an append-only history table. This row records: who did it, when, the previous state, the new state, and the form data submitted. Rows are never updated or deleted.

The result: the audit trail is a first-class artifact. Compliance reviews, debugging "why is this stuck?", and reconstructing the exact state of any task at any point in time are all the same query.

### Form data as JSONB

Each task carries a typed `form_data` JSONB payload. The shape is determined by `form_type` and validated by a Pydantic schema at submission. Two consequences:

- **Wizards are free.** Partial saves skip validation; submission triggers it. No special "draft" entity, no schema split.
- **New forms don't need migrations.** Adding a `form_type` is a Pydantic schema and a frontend component — zero database changes.

### External identity

Authentication delegates to an external identity service (any PostgreSQL-backed user store with argon2id-hashed passwords). The JWT carries `user_id`, `area`, `role`, plus identity claims — so authorization happens with **zero database roundtrip per request**. Token expiration: 8h, with refresh-token rotation. Startup fails fast on a misconfigured secret rather than degrading silently in production, and legacy credential checks use constant-time comparison.

### Visibility models

Four visibility patterns, configurable per stage:

- **Department/group routing** (default) — task is visible to anyone in the matching area + role combination
- **Specific user** — task assigned to a single named user
- **Open** — visible to anyone with platform access
- **Time-limited public link** — a signed, expiring token grants a named external collaborator access to one specific step with no account required; every action taken through the link is written to a separate, admin-only audit log

Completed tasks always render read-only. Cross-area access returns `403`.

### Attachments via presigned URLs

The frontend never proxies binary data through the API. MinIO presigned URLs go direct from the browser. The backend's only role is authorizing the URL grant. This keeps the API tier lean and stateless even when users are uploading large files.

### Deadline monitoring and notifications

A background sweep checks open tasks against their configured deadline on an interval, flips a visual "expired" indicator in the UI, and raises an alert through the same event channel used for stage transitions — deadline tracking is a consumer of the engine's state changes, not a bolted-on cron job. Stage-assignment and completion emails render from pure, escaping-safe builder functions and dispatch asynchronously with per-recipient deduplication, so a step with several assignees or overlapping watchers never double-sends.

---

## Reference modules in production

| Module | Description |
|---|---|
| **RNC** (non-conformance) | 6-step quality management flow with department-based routing |
| **HR approvals** | Multi-level HR request flow with management-tier decision gate |
| **NPD pipeline** | 8-stage product development flow exercising the full feature set: parallel groups, 4-person decision gate, 14 specialized forms, ERP integration, PDF export |

---

## Engineering patterns worth highlighting

### Auth claims, not auth queries

Every authorization check reads from JWT claims — `user_id`, `area`, `role`. Zero database roundtrip per request. Identity service is hit once at login, then JWT is the source of truth until expiration.

### Parallel groups by `(stage, group_id)`

Parallel branches share a `group_id`. The engine advances when `count(completed) == count(total)` for the group — independent of order. Adding a new branch to an existing parallel stage is a config change, not a code change.

### Decision gates as configurable predicates

Approval rules (`all_approve_advances`, `any_reject_terminates`, `any_standby_pauses`) are configuration, not hardcoded `if/else` chains. New gate behaviors are added without touching engine code.

### Append-only history as state recovery

Because history is append-only, replaying it from `t=0` reconstructs any task's state at any point in time. Useful for compliance, "why did this happen?" debugging, and time-travel UIs.

### Pydantic at the schema boundary, JSONB at the storage boundary

Form payloads validate strictly at API ingress (Pydantic), then store as JSONB (flexible). This combines the safety of typed forms with the schema flexibility of NoSQL — without giving up SQL's transactional guarantees.

### Read-only ERP mirror for live lookups

Autocomplete widgets (supplier, product, batch/lot) query a dedicated async, read-only connection pool against a mirrored ERP database — never the system of record directly. Lookups stay fast without any risk of a reporting query touching transactional writes.

---

## Status

Reference implementation. Production deployment is in active daily use across multiple modules.

## License

Proprietary — All rights reserved.
