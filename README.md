# Covara — Real-Time Drone Intelligence Platform

> Case study: a three-component platform for AI-assisted drone operations — edge object detection, real-time telemetry ingestion, and a live operations dashboard.
> **Status: in alpha.** Staging and production environments are live; load-tested to 100 concurrent authenticated connections on the 2 vCPU staging VM. Production-scale load testing is still pending.
> **Code is proprietary; this repo documents the build** — architecture, technical decisions, and known limitations.

---

## Context

| | |
|---|---|
| **Client** | Covara (Covara Technologies) |
| **Role** | Backend + Edge AI developer — backend architecture and API design, edge inference pipeline, and delivery of the operations dashboard frontend |
| **Timeline** | Mar 2025 — present |
| **Stage** | Alpha — deployed to staging and production |

## Problem

Covara operates drones for intelligence and surveillance missions and needed the full software stack behind them: a way to collect telemetry from drones in the field, run object-detection AI on captured imagery in real time, turn detections into operator-facing events and alerts, and present everything — positions, detections, missions, media — on a live dashboard.

The hard parts:

- **Heterogeneous pipeline.** Telemetry, video, AI detections, and device health heartbeats all flow differently and all need to land in one coherent operational picture.
- **Untrusted edge.** Drones connect from the field over unreliable links — the backend has to treat edge devices as authenticated, isolated tenants, not trusted LAN peers.
- **Real-time operations.** Operators act on live events; the dashboard is fed by WebSocket pushes, not polling.
- **Small staging budget.** Staging runs on a 2 vCPU / 4 GB VM, which forces honest resource discipline.

## Architecture

Three components, one data flow:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Edge       │────▶│   Backend    │◀───▶│   Frontend   │
│   (Jetson)   │     │   (GCP VM)   │     │   (Vercel)   │
│              │     │              │     │              │
│  YOLOv8n AI  │     │  PostgreSQL  │     │  Dashboard   │
│  Telemetry   │     │  TimescaleDB │     │  Live Map    │
│  Heartbeat   │     │  Redis Cache │     │  Mission Ctl │
└──────────────┘     │  MinIO Store │     └──────────────┘
                     └──────────────┘
```

**Data flow:**

1. Drone sends telemetry + video → RC Pro controller (WildBridge app) → edge device (NVIDIA Jetson Orin X)
2. Edge runs on-device AI inference (YOLOv8n) and forwards detections, telemetry, and heartbeats to the backend API over authenticated edge routes
3. Backend persists data, generates events/alerts from detections matching the active mission's selected classes, and broadcasts over authenticated, organisation-scoped WebSockets
4. Operators monitor on the dashboard, acknowledge events, and manage missions

### Stack

| Layer | Technology | Why |
|---|---|---|
| Backend API | Node.js 20, Express, TypeScript | Typed end-to-end, async-first fit for streaming workloads |
| Data | PostgreSQL 16 + TimescaleDB + PostGIS | Time-series telemetry, geospatial positions and queries in one engine |
| ORM | Prisma | Schema-as-code, migration discipline |
| Cache / tokens | Redis 7 | Token-backed dual-auth sessions, rate limiting |
| Object storage | MinIO (S3-compatible) | Imagery/video assets, self-hosted |
| Edge AI | Python, YOLOv8n, NVIDIA Jetson Orin X | On-device inference — no cloud round-trip per frame |
| Frontend | Next.js 14, React, TailwindCSS | Dashboard, live map, mission control, detection review |
| Deployment | Docker Compose on GCP Compute Engine (backend), Vercel (frontend) | Simple, operable by a small team |

### Edge AI pipeline

- **Custom 6-class YOLOv8n model:** boat, motorcycle, person, vehicle, vehicle_military, weapon_visible
- **Mission-scoped filtering:** incoming detection frames are filtered against the active mission's `selectedDetectionClasses` before events are generated — operators only get alerts for classes relevant to the current mission
- **Heartbeat monitoring:** every edge device reports health, so a dropped link is visible as an event, not a silent gap

### API surface

Thirteen documented API categories: authentication, drone registry, missions, organisations, telemetry, heartbeat, detections, events, media, WebSocket, last-known drone state, playback, and map (partial — see limitations). Error responses, pagination, filtering, and validation follow a written API standard, with RBAC and audit logging as cross-cutting concerns.

### Edge integration contract

The edge-to-backend contract is versioned and documented per payload type — telemetry, heartbeat, and detections each have a written spec (see the platform's API documentation). Edge devices authenticate against assigned drones, so a device cannot impersonate another unit in the fleet, and detection frames carry the model's class, confidence, and geospatial context so the backend can generate mission-scoped events without re-inference.

### Access control

Role-based access control spans viewer, operator, and administrator levels (with a platform-admin tier above organisation admins), enforced on REST routes and WebSocket subscriptions alike. Mission mutations are deliberately restricted: editing requires operator or higher; deletion is admin-only. Every organisation's data is scoped end-to-end — a user cannot see another organisation's telemetry, missions, or media.

### Audit logging

Security-relevant actions are written to an audit log (documented schema, queryable), so operational reviews don't depend on reconstructing events from application logs.

### Security posture (hardened during alpha)

The original ingestion routes (`POST /telemetry`, `/heartbeat`, `/media/*`) were unauthenticated. They were removed and replaced with authenticated edge routes, all dashboard reads were placed behind authentication and organisation scoping, and WebSocket connections now require JWT with resource-specific, organisation-scoped subscriptions. Edge devices can only report for their assigned drone.

## Environments

| Environment | Backend | Frontend |
|---|---|---|
| Staging | `https://api-staging.covara.tech` | `https://app-staging.covara.tech` |
| Production | `https://api.covara.tech` | `https://app.covara.tech` |

## Key Technical Decisions

| Decision | Rationale | Trade-off |
|---|---|---|
| Authenticated edge routes (legacy unauthenticated ingestion removed) | Edge devices are field-deployed and untrusted; org-scoped auth is the only safe default | Every edge payload carries auth overhead; device provisioning becomes a workflow |
| YOLOv8n on-device (Jetson) | Real-time inference without bandwidth/cloud cost per frame | Model updates must be distributed to devices; no live model switching (reload requires SIGHUP or restart) |
| PostgreSQL + TimescaleDB + PostGIS in one engine | Telemetry, geospatial, and relational data share transactions and one ops story | PostgreSQL-specific; not portable to other databases |
| MinIO over managed S3 | Self-hosted, cost-controlled object storage | Operational overhead — disk pressure on a small staging VM is real |
| WebSocket push to the dashboard | Operators need detections as they happen, not on refresh | Connection lifecycle management (JWT auth, scoped subscriptions, reconnects) |
| Docker Compose on a single GCP VM | A small team's operability beats orchestration ceremony at this stage | Vertical ceiling — the 2 vCPU / 4 GB staging VM is already the constraint |

## Known Limitations (alpha — documented, not hidden)

Honest limitations from the platform's own documentation:

- **501 (not yet implemented) endpoints:** detection verification workflow, flight paths / geofences / nearby queries, and report creation/listing/download all return `501 NOT_IMPLEMENTED`. Dashboard stats and summaries are implemented.
- **No live model switching.** The active inference model is chosen at startup (`MODEL_TYPE` env / config); changing it requires a `SIGHUP` or service restart. Backend-driven hot-reload and RGB↔thermal auto-switching are deferred.
- **Single-drone-per-device, single-mission-per-drone.** An edge device serves exactly one drone; a drone has exactly one active mission at a time.
- **Config surface is narrow.** The edge-device config endpoint updates `confidenceThreshold` only — not model type, video source, or nested fields.
- **Staging is small — and it shows.** The staging VM is `e2-medium` (2 vCPU, 4 GB RAM, 50 GB SSD). It holds 100 concurrent authenticated WebSocket connections cleanly, but a connection cap surfaces under reconnect churn (~90+ concurrent), and Docker images and logs consume disk — periodic cleanup is required. This is a real constraint, not a footnote.

## Load Testing

> **Load-tested on a 2 vCPU / 4 GB (e2-medium) staging VM: sustained 100 concurrent authenticated WebSocket connections for 90s with 100% connect success, zero session drops, and ~20–50ms server-side round-trip.**

- k6 ramp test (10 → 50 → 100 concurrent authenticated JWT connections), staging only: `api-staging.covara.tech`
- Connect→ack p99 under 3s end-to-end, dominated by client↔GCP network latency (~1.1s from test client in Brazil); server-side processing ~20–50ms
- Observed ceiling under reconnect churn: at ~90+ concurrent with continuous connect churn, the server initiates ~0.25 force-closes/sec — all clients reconnect successfully; suspected connection cap on the 2 vCPU instance
- Zero HTTP 5xx across all runs

## What I'd Do Differently

- **Load-test earlier.** Ran load testing late — a connection cap surfaced at ~90+ concurrent under churn that earlier stress testing would have caught at a fraction of the cost.
- **Structured logging from day one.** Debugging distributed WebSocket and edge-connectivity issues required retroactive log enrichment; structured logs should have been in the first milestone.
- **Design the config API for growth.** The narrow config-update surface (`confidenceThreshold` only) reflects early assumptions; a versioned device-config schema would have made thermal switching and hot-reload cheaper to add.
- **Allocate staging disk headroom.** 50 GB with Docker images, logs, and MinIO objects fills faster than expected — image pruning and log rotation should have been part of the deploy scripts from the start.

## Delivery Notes

- **Three live environments** behind one codebase: local (Docker Compose), staging, and production, promoted along `main` / `staging` / `feature/*` branches with PRs into staging.
- **Documentation as a deliverable:** a full documentation index — API specs per category, deployment guides, an edge integration contract, RBAC and audit-log references, and a non-technical product-owner setup guide — ships with the platform, because an ops platform that only its author can run isn't finished.
- Backend runs as a Docker Compose stack on GCP Compute Engine; the dashboard deploys to Vercel.

## Evidence

- **Environments:** available on request.
- **API documentation:** available on request.
- **Live demo:** available on request.

---

*Case study by David Aloba. [code.davidaloba.com](https://code.davidaloba.com) · [LinkedIn](https://linkedin.com/in/mistertaavetti)*
