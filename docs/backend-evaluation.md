# Nexus Backend — Technical Evaluation

**Repository:** `agni-eialarasu/nexus-backend`  
**Analysis Date:** August 2026  
**Status:** Reverse-engineered from source code

---

## 1. Executive Summary

Nexus Backend is a **NestJS 10** application serving as the API layer for a multi-tenant Project Portfolio Management (PPM) platform. It provides 28+ API controllers covering project management, OKRs, financial tracking, resource planning, RAID management, and business capability modeling. The system uses **Prisma 5** as the ORM with a **PostgreSQL** database hosted on **Supabase**, implementing a sophisticated dual-layer tenant isolation strategy (application-level middleware + database-level RLS).

### Key Strengths
- Robust multi-tenancy architecture with defense-in-depth
- Comprehensive domain coverage (PPM, OKRs, RAID, Financials, Capacity, Beacon)
- Clean NestJS modular architecture with consistent patterns
- Multiple deployment targets (Docker, Vercel serverless)
- Swagger/OpenAPI documentation built-in

### Key Risks
- **Zero test coverage** — no unit, integration, or E2E tests
- No linting or formatting configuration
- Mixed authorization patterns (some controllers unguarded)
- Sensitive defaults (hardcoded JWT secret in dev mode)
- No rate limiting or request throttling

---

## 2. Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Runtime | Node.js | 20 (Alpine) |
| Language | TypeScript | ^5.9.3 |
| Framework | NestJS | ^10.4.15 |
| HTTP Platform | Express | 4.21.2 |
| ORM | Prisma | 5.8.0 |
| Database | PostgreSQL | (Supabase-hosted) |
| Authentication | JWT (jsonwebtoken ^9.0.2) | — |
| Password Hashing | bcrypt | ^6.0.0 |
| Google OAuth | google-auth-library | ^10.6.2 |
| API Documentation | @nestjs/swagger | ^7.4.2 |
| Validation | class-validator / class-transformer | ^0.14.1 / ^0.5.1 |
| HTTP Client | Axios / @nestjs/axios | ^1.7.9 / ^3.1.3 |
| Configuration | @nestjs/config | ^3.3.0 |
| AWS SDK | @aws-sdk/client-cognito-identity-provider | ^3.0.0 |

---

## 3. Architecture

### 3.1 Design Pattern

**Modular Layered Architecture** following NestJS conventions:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Express / Vercel                          │
├─────────────────────────────────────────────────────────────────┤
│  Global Middleware (CORS, Tenant Resolution, JWT Extraction)    │
├─────────────────────────────────────────────────────────────────┤
│                     NestJS Router Layer                          │
├─────────────────────────────────────────────────────────────────┤
│  Controllers (28)  │  Guards (JwtAuthGuard)  │  RBAC Checks     │
├─────────────────────────────────────────────────────────────────┤
│                      Service Layer                               │
│  (Business Logic, Validation, Orchestration)                    │
├─────────────────────────────────────────────────────────────────┤
│  PrismaService (Tenant Middleware) + AsyncLocalStorage Context  │
├─────────────────────────────────────────────────────────────────┤
│              PostgreSQL (Supabase) + RLS Policies                │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Module Organization

The app is divided into **28 feature modules** registered in `AppModule`:

| Domain | Modules |
|--------|---------|
| **Core** | PrismaModule, AuthModule, UsersModule |
| **Project Management** | ProjectsModule, PpmProjectsModule, TasksModule, WorkModule |
| **Portfolio & Strategy** | PortfoliosModule, StrategiesModule, RoadmapsModule |
| **Planning** | OkrsModule, ResourcesModule, CapacityPlans |
| **Financial** | FinancialsModule, TimesheetsModule |
| **Risk & Governance** | RaidModule, RequestsModule |
| **Collaboration** | DiscussionsModule, NotificationsModule, TemplatesModule |
| **Analytics** | DashboardsModule, BeaconModule |
| **Integrations** | JiraModule, JellyfishModule, IntegrationsConfigModule |

### 3.3 Multi-Tenancy Implementation

The system uses a **shared-database, shared-schema** approach with `tenantId` on every table:

**Layer 1 — Application Middleware:**
1. Global Express middleware extracts JWT from `Authorization` header
2. Decodes user → looks up `user.tenantId` → resolves `Tenant` entity
3. Calls `runWithTenantContext({ tenantId, tenantSlug }, next)` using Node.js `AsyncLocalStorage`

**Layer 2 — Prisma Query Middleware:**
4. `PrismaService.$use()` middleware intercepts ALL Prisma operations
5. For reads: injects `WHERE tenantId = ?`
6. For creates: injects `tenantId` into data payload (including nested creates)
7. For updates/deletes: pre-checks record belongs to tenant before mutation

**Layer 3 — Database RLS:**
8. `withTenantTransaction()` sets `SET LOCAL ROLE app_backend` + `SET LOCAL app.current_tenant_id`
9. PostgreSQL RLS policies enforce isolation at the database level
10. Even if application middleware has a bug, RLS prevents cross-tenant data access

---

## 4. Data Model

### 4.1 Database

**PostgreSQL** hosted on Supabase. Schema defined in `prisma/schema.prisma` (1715 lines, 40+ models).

### 4.2 Entity Categories

```
┌─────────────────────────────────────────────────────┐
│                    IDENTITY & AUTH                    │
│  Tenant, User, TenantRole                           │
├─────────────────────────────────────────────────────┤
│                  PROJECT MANAGEMENT                   │
│  Project, ProjectMember, Portfolio, PortfolioMember  │
├─────────────────────────────────────────────────────┤
│                  STRATEGIC PLANNING                   │
│  Strategy, StrategicCategory, Objective, KeyResult   │
├─────────────────────────────────────────────────────┤
│                   WORK MANAGEMENT                     │
│  WorkItem, WorkItemAssignment, Board, KanbanCard    │
├─────────────────────────────────────────────────────┤
│                     FINANCIAL                         │
│  ProjectFinancial, CompanyBudget, TimesheetEntry     │
├─────────────────────────────────────────────────────┤
│                   RISK & GOVERNANCE                   │
│  Risk, Issue, Assumption, Dependency, Request       │
├─────────────────────────────────────────────────────┤
│                    RESOURCES                          │
│  ResourceAssignment, CapacityPlan, Milestone        │
├─────────────────────────────────────────────────────┤
│                   COLLABORATION                       │
│  Discussion, DiscussionPost, Comment, Notification  │
├─────────────────────────────────────────────────────┤
│              BEACON (CAPABILITY MODEL)               │
│  BeaconDomain, BeaconCapability, BeaconApplication  │
│  BeaconSubCapability, BeaconInitiative              │
└─────────────────────────────────────────────────────┘
```

### 4.3 Key Relationships

- **Tenant → All Entities** — Every model has a `tenantId` foreign key
- **User → Manager** — Self-referencing hierarchy for org chart
- **User → TenantRole** — Role-based permissions per tenant
- **Project → Portfolio → Strategy** — Strategic alignment chain
- **Objective → KeyResult → Contribution** — OKR cascade
- **BeaconDomain → Capability → SubCapability** — Capability tree
- **BeaconCapability ↔ Application** — Many-to-many junction table

### 4.4 User Model (Key Fields)

| Field | Type | Purpose |
|-------|------|---------|
| email | String | Primary identifier (unique per tenant) |
| password | String? | bcrypt hash (null for Google-only users) |
| googleId | String? | Google OAuth identifier |
| role | UserRole enum | ADMIN, USER, VIEWER |
| isOrgAdmin | Boolean | Full-access org administrator |
| tenantRoleId | String? | Custom tenant role assignment |
| permissions | Json? | Fine-grained page/feature access |
| managerId | String? | Self-referencing reporting hierarchy |
| capacity | Float? | Hours per period (capacity planning) |
| costRate | Float? | Hourly rate (financial calculations) |

### 4.5 Migration History

14 migrations covering schema evolution:
- `20260106` — Initial schema
- `20260107–20260113` — Company budget, KR direction, discussions, timesheets
- `20260114–20260120` — RAID enhancements, kanban history, portfolio strategy
- `20260121–20260129` — Multi-tenancy, user scoping, cleanup (removed Slack/Confluence tables)

---

## 5. API Surface

### 5.1 Authentication Endpoints (Public)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/auth/register` | Email/password registration with tenant join/create |
| POST | `/api/auth/login` | Email/password login → JWT |
| POST | `/api/auth/google` | Google OAuth credential verification |
| POST | `/api/auth/supabase-exchange` | Exchange Supabase session token for app JWT |
| GET | `/api/auth/tenant/lookup?code=` | Look up tenant by join code |
| GET | `/api/auth/me` | Current user profile (JWT required) |
| GET | `/api/auth/tenant` | Current tenant info (JWT required) |
| POST | `/api/auth/setup-tenant` | Configure tenant name/slug |
| GET | `/api/auth/my-tenants` | List user's tenants across orgs |
| POST | `/api/auth/switch-tenant` | Switch active tenant → new JWT |

### 5.2 Core Business Endpoints (JWT Protected)

| Domain | Prefix | Key Operations |
|--------|--------|----------------|
| Projects (PPM) | `/ppm/projects` | CRUD, members, milestones, budget |
| Portfolios | `/portfolios` | CRUD, members, strategic categories |
| Work Items | `/work-items` | CRUD, assignments, transitions, links |
| Kanban | `/kanban-cards` | CRUD, position management, assignment history |
| OKRs | `/objectives`, `/key-results` | CRUD, contributions, progress tracking |
| Financials | `/financials` | Entries, company budget, fiscal summaries |
| RAID | `/risks`, `/issues`, `/assumptions`, `/dependencies` | CRUD per entity |
| Resources | `/resource-assignments`, `/capacity-plans` | Allocation, capacity planning |
| Roadmaps | `/roadmaps` | Timeline items management |
| Strategies | `/strategies`, `/strategic-categories` | Strategic alignment |
| Timesheets | `/timesheets` | Time entry CRUD |
| Discussions | `/discussions` | Thread + post management |
| Templates | `/templates` | Project template CRUD |
| Notifications | `/notifications` | User notification management |
| Dashboards | `/dashboards` | Widget-based dashboard configuration |
| Beacon | `/beacon` | Business capability model (domains, capabilities, apps, initiatives) |
| Users | `/users` | User management, org chart, permissions, roles |

### 5.3 Integration Endpoints

| Integration | Prefix | Operations |
|-------------|--------|------------|
| Jira | `/api/jira` | Projects, boards, issues, search (JQL), transitions, comments |
| Confluence | `/api/confluence` | Spaces, pages, comments |
| Config | `/integrations-config` | CRUD for integration configurations |

### 5.4 System Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/` | Hello/version |
| GET | `/health` | Health check with environment diagnostics |
| GET | `/api-docs` | Swagger UI |

---

## 6. External Integrations

| Service | Purpose | Connection Method |
|---------|---------|-------------------|
| **Supabase** | PostgreSQL hosting, JWT verification (JWKS) | Direct connection string + HTTP |
| **Google** | OAuth sign-in | google-auth-library (ID token verification) |
| **Jira** | Project/issue synchronization | REST API (Basic Auth with email + token) |
| **Confluence** | Wiki/documentation access | REST API (Basic Auth) |
| **Jellyfish** | Engineering metrics | REST API (service layer only) |
| **AWS Cognito** | Legacy user pool (referenced but not primary) | AWS SDK |

---

## 7. Authentication & Authorization

### 7.1 Authentication Flow

```
┌──────────┐     ┌──────────────┐     ┌────────────────┐
│  Client  │────▶│  Auth API    │────▶│   JWT Signed   │
│          │◀────│  (register/  │◀────│   (sub, email) │
│          │     │   login/     │     │   HS256        │
│          │     │   google)    │     │                │
└──────────┘     └──────────────┘     └────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │ Supabase Session │ ───▶ /supabase-exchange
              │ (OAuth redirect) │        → app JWT
              └─────────────────┘
```

**JWT payload:** `{ sub: userId, email: userEmail }`  
**Algorithm:** HS256 with configurable secret  
**Token verification:** Supports HS256 (shared secret) and RS256/ES256 (Supabase JWKS endpoint)

### 7.2 Authorization (RBAC)

| Level | Mechanism |
|-------|-----------|
| **Org Admin** | `user.isOrgAdmin === true` → full access to all pages/features |
| **Tenant Role** | `user.tenantRole.permissions` → page-level access grants |
| **User Override** | `user.permissions` → per-user fine-grained overrides |
| **View Mode** | `single` (own data), `team` (direct reports), `global` (all tenant) |

Permission resolution order: isOrgAdmin > TenantRole > User permissions

### 7.3 Guard Implementation

- `JwtAuthGuard` — NestJS guard that verifies JWT, attaches `req.user` (payload) and `req.dbUser` (full User entity with role)
- RBAC checks via `canAccessPage(req, pageId)` and `canEditPage(req, pageId)` in controller methods

---

## 8. Configuration & Environment

### 8.1 Required Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `DATABASE_URL` | — | PostgreSQL connection string |
| `JWT_SECRET` | `'nexus-dev-jwt-secret'` | JWT signing secret |
| `PORT` | 3001 (dev) / 50051 (prod) | HTTP listen port |
| `SUPABASE_URL` | — | Supabase project URL |
| `SUPABASE_JWT_SECRET` | — | Supabase HS256 verification |
| `GOOGLE_CLIENT_ID` | — | Google OAuth client ID |
| `CORS_ORIGINS` | `localhost:3000,localhost:80,nexus.cetana.io` | Allowed origins |
| `SWAGGER_PATH` | `api-docs` | Swagger UI route |

### 8.2 Integration Variables

| Variable | Purpose |
|----------|---------|
| `JIRA_BASE_URL` | Jira instance URL |
| `JIRA_EMAIL` | Jira API authentication email |
| `JIRA_API_TOKEN` | Jira API token |

### 8.3 Environment Files

- `.env` — Default (local development)
- `.env.dev` — Development environment
- `.env.prod` — Production environment

---

## 9. Build & Deployment

### 9.1 Docker (Multi-Stage)

```dockerfile
# Stage 1: Builder
FROM node:20-alpine AS builder
# Install deps, generate Prisma client, compile TypeScript

# Stage 2: Development
FROM node:20-alpine AS development
# Hot-reload dev server, port 3000

# Stage 3: Production
FROM node:20-alpine AS production
# Minimal image, non-root user, port 50051
CMD ["node", "dist/src/main"]
```

### 9.2 Vercel Serverless

- `api/index.ts` wraps the NestJS app in an Express handler for Vercel's serverless functions
- `vercel-build` script: `prisma generate && nest build`
- Compatible with Vercel's Node.js 20 runtime

### 9.3 Scripts

| Script | Command | Purpose |
|--------|---------|---------|
| `start:dev` | `nest start --watch` | Local development |
| `start:prod` | `node dist/src/main` | Production start |
| `build` | `nest build` | Compile TypeScript |
| `vercel-build` | `prisma generate && nest build` | Vercel deployment |
| `prisma:migrate:dev` | `dotenv -e .env.dev -- prisma migrate dev` | Dev migrations |
| `prisma:migrate:prod` | `dotenv -e .env.prod -- prisma migrate deploy` | Prod migrations |
| `prisma:studio:dev` | `dotenv -e .env.dev -- prisma studio` | DB GUI |

---

## 10. Testing

### Current State: ❌ NO TESTS

```json
"test": "echo \"Error: no test specified\" && exit 1"
```

- No unit tests
- No integration tests
- No E2E tests
- No test framework configured
- No test coverage tooling

**Risk Assessment:** HIGH — A complex multi-tenant system with 40+ models and 28 controllers has zero automated verification. Any refactoring or feature addition has no safety net.

---

## 11. Code Quality

### 11.1 Positive Patterns

- Consistent NestJS module structure (Controller → Service → Module)
- TypeScript strict compilation
- Prisma schema with proper indexes and relations
- Defense-in-depth tenancy (middleware + RLS)
- NestJS exception filters for error responses
- Swagger decorators on auth controller

### 11.2 Concerns

| Area | Issue | Severity |
|------|-------|----------|
| **Testing** | Zero test coverage | 🔴 Critical |
| **Linting** | No ESLint/Prettier configuration | 🟡 Medium |
| **Auth Defaults** | Hardcoded `JWT_SECRET` default in code | 🟡 Medium |
| **Guard Coverage** | Some controllers lack `@UseGuards(JwtAuthGuard)` | 🟡 Medium |
| **Validation** | DTOs not consistently using class-validator decorators | 🟡 Medium |
| **Error Handling** | Inconsistent patterns across controllers | 🟡 Medium |
| **Logging** | Minimal structured logging | 🟡 Medium |
| **Rate Limiting** | No request throttling | 🟡 Medium |

### 11.3 Security Considerations

| Item | Status |
|------|--------|
| Password hashing | ✅ bcrypt with salt rounds 10 |
| JWT verification | ✅ HS256 + JWKS fallback |
| SQL injection | ✅ Prisma parameterized queries + tenantId validation regex |
| Tenant isolation | ✅ Dual-layer (middleware + RLS) |
| CORS | ✅ Configurable allowed origins |
| Secrets in code | ⚠️ Dev-mode defaults present |
| Rate limiting | ❌ Not implemented |
| Input validation | ⚠️ Inconsistent DTO validation |
| HTTPS enforcement | ⚠️ Depends on deployment platform |

---

## 12. Technical Debt & Risks

### 12.1 Critical

1. **No test suite** — Zero automated tests for a complex multi-tenant system
2. **Mixed auth guard coverage** — Integration controllers (Jira, Confluence) have no guards; could leak data if tenant middleware fails

### 12.2 Medium

3. **Legacy `ProjectsModule`** vs `PpmProjectsModule` — Two project systems coexist (likely migration in progress)
4. **AWS Cognito references** — Dead code/dependencies for unused auth provider
5. **No structured logging** — Difficult to trace issues in production
6. **No health check sophistication** — `/health` doesn't verify DB connectivity
7. **Package registry artifacts** — Dockerfile references private CodeArtifact registry (rewrites to public npm)

### 12.3 Low

8. **No API versioning** — Breaking changes will affect all clients
9. **Swagger incomplete** — Not all controllers have Swagger decorators
10. **No database connection pooling config** — Relies on Prisma/Supabase defaults

---

## 13. Recommendations

### Quick Wins (1–2 days each)

- [ ] Add ESLint + Prettier configuration
- [ ] Add `@nestjs/throttler` for rate limiting
- [ ] Remove hardcoded JWT_SECRET default; fail startup if not set in production
- [ ] Add `@UseGuards(JwtAuthGuard)` to all non-public controllers
- [ ] Add proper health check (DB ping, memory usage)

### Medium-Term (1–2 weeks)

- [ ] Implement test infrastructure (Jest + Supertest for E2E)
- [ ] Write integration tests for critical auth flows
- [ ] Add structured logging (Winston or Pino)
- [ ] Consolidate `ProjectsModule` and `PpmProjectsModule`
- [ ] Add DTO validation decorators across all controllers
- [ ] Remove unused AWS Cognito dependency

### Strategic (1–2 months)

- [ ] Achieve 60%+ test coverage on service layer
- [ ] Implement API versioning (URI or header-based)
- [ ] Add database connection health monitoring
- [ ] Implement audit trail for tenant admin actions
- [ ] Performance test tenant isolation under concurrent load
- [ ] Add OpenTelemetry for distributed tracing

---

## 14. Appendix: File Structure

```
nexus-backend/
├── api/
│   └── index.ts                    # Vercel serverless entry
├── prisma/
│   ├── schema.prisma               # Database schema (1715 lines)
│   ├── migrations/                 # 14 migrations
│   └── supabase-rls-policies.sql   # RLS policy definitions
├── src/
│   ├── main.ts                     # NestJS bootstrap
│   ├── app.module.ts               # Root module
│   ├── app.controller.ts           # Health/root endpoints
│   ├── auth/                       # Authentication module
│   ├── prisma/                     # PrismaService (tenant middleware)
│   ├── tenant/                     # AsyncLocalStorage context
│   ├── projects/                   # Legacy projects
│   ├── ppm-projects/               # PPM projects (full-featured)
│   ├── tasks/                      # Task management
│   ├── portfolios/                 # Portfolio management
│   ├── dashboards/                 # Dashboard widgets
│   ├── financials/                 # Financial tracking
│   ├── raid/                       # Risks, Issues, Assumptions, Dependencies
│   ├── requests/                   # Intake requests
│   ├── okrs/                       # Objectives & Key Results
│   ├── resources/                  # Resource assignments
│   ├── roadmaps/                   # Roadmap management
│   ├── strategies/                 # Strategic planning
│   ├── work/                       # Work items & Kanban
│   ├── timesheets/                 # Time tracking
│   ├── discussions/                # Collaboration
│   ├── templates/                  # Project templates
│   ├── notifications/              # Notification system
│   ├── users/                      # User management
│   ├── beacon/                     # Business capability model
│   ├── integrations/               # Jira, Jellyfish
│   └── integrations-config/        # Integration settings
├── Dockerfile                      # Multi-stage build
├── package.json                    # Dependencies
├── nest-cli.json                   # NestJS CLI config
└── tsconfig.json                   # TypeScript config
```
