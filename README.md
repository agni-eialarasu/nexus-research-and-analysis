# Nexus — Research & Analysis

This repository contains reverse-engineering evaluation documents for the **Nexus** platform — a multi-tenant Project Portfolio Management (PPM) system.

## Documents

| Document | Description |
|----------|-------------|
| [Backend Evaluation](./docs/backend-evaluation.md) | Complete technical evaluation of `nexus-backend` |
| [Frontend Evaluation](./docs/frontend-evaluation.md) | Complete technical evaluation of `nexus-frontend` |
| [System Overview](./docs/system-overview.md) | Combined architecture showing how frontend & backend integrate |
| [Architecture Decisions](./docs/architecture-decisions.md) | Nexus v2 rebuild decisions (Flutter + Supabase stack, RBAC, multi-tenancy) |
| [Beacon Module Analysis](./docs/beacon-module-analysis.md) | Deep-dive into the Business Capability Model module (data model, scoring, views, rebuild recommendations) |

## Repositories Analyzed

| Repository | Stack | Purpose |
|------------|-------|---------|
| `agni-eialarasu/nexus-backend` | NestJS + Prisma + PostgreSQL | API server, business logic, data layer |
| `agni-eialarasu/nexus-frontend` | React + TypeScript + MUI | Single-page application UI |

## Analysis Date

**August 2026** — Generated via reverse engineering of source code.

---

*This analysis was produced by systematic code review and architectural tracing of both repositories.*
