# Nexus Platform — System Overview

**Analysis Date:** August 2026  
**Repositories:**
- Backend: `agni-eialarasu/nexus-backend`
- Frontend: `agni-eialarasu/nexus-frontend`

---

## 1. What is Nexus?

Nexus is a **multi-tenant Project Portfolio Management (PPM) platform** designed for enterprise teams. It provides a unified workspace for:

- **Project Management** — Track projects with Jira integration, work items, Kanban boards
- **Portfolio Management** — Group projects into portfolios with strategic alignment
- **OKRs** — Objectives & Key Results with cascade tracking
- **Financial Tracking** — Budgets, actuals, forecasts, cost rates
- **Resource & Capacity Planning** — Team allocation, utilization, capacity
- **RAID Management** — Risks, Assumptions, Issues, Dependencies
- **Business Capability Model (Beacon)** — Map capabilities to applications & initiatives
- **Strategic Planning** — Strategies, roadmaps, strategic categories
- **Timesheets** — Time entry and tracking
- **Collaboration** — Discussions, notifications, comments

The platform is hosted at **nexus.cetana.io** (production).

---

## 2. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           USERS / BROWSERS                            │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ HTTPS
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     FRONTEND (React SPA)                              │
│  • React 18 + TypeScript + MUI v5                                    │
│  • Deployed: Docker (nginx) or Vercel                                │
│  • Port: 80 (prod) / 3000 (dev)                                     │
│  • Tenant-scoped routing: /:tenantSlug/*                             │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ REST API (JSON)
                               │ Authorization: Bearer <JWT>
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     BACKEND (NestJS API)                              │
│  • NestJS 10 + TypeScript + Prisma 5                                 │
│  • Deployed: Docker or Vercel Serverless                             │
│  • Port: 3001 (dev) / 50051 (prod Docker)                           │
│  • 28 controllers, multi-tenant middleware                           │
└────────────┬────────────────────────────────┬────────────────────────┘
             │                                │
             ▼                                ▼
┌────────────────────────┐     ┌─────────────────────────────────────┐
│   PostgreSQL (Supabase) │     │       External Services              │
│  • 40+ tables           │     │  • Supabase Auth (OAuth/JWKS)       │
│  • RLS policies         │     │  • Google OAuth                      │
│  • Tenant isolation     │     │  • Jira (REST API)                  │
│  • 14 migrations        │     │  • Confluence (REST API)            │
└────────────────────────┘     │  • Jellyfish (metrics)              │
                                └─────────────────────────────────────┘
```

---

## 3. Technology Stack Comparison

| Layer | Frontend | Backend |
|-------|----------|---------|
| **Language** | TypeScript (strict) | TypeScript (strict) |
| **Runtime** | Browser (React 18) | Node.js 20 (NestJS 10) |
| **Build** | CRA (react-scripts 5.0.1) | NestJS CLI (nest build) |
| **Deployment** | Docker (nginx) / Vercel | Docker / Vercel Serverless |
| **State** | React Context + local state | — |
| **Data Access** | Axios HTTP client | Prisma 5 ORM |
| **Auth** | Supabase client + localStorage JWT | JWT (jsonwebtoken) + bcrypt |
| **UI** | MUI v5 + custom theme | Swagger UI |
| **Testing** | Jest/RTL (unused) | None |

---

## 4. Communication Pattern

### 4.1 API Contract

| Aspect | Detail |
|--------|--------|
| Protocol | REST over HTTPS |
| Format | JSON request/response bodies |
| Auth Header | `Authorization: Bearer <JWT>` |
| Base URL | Configurable (`REACT_APP_API_URL`) |
| CORS | Backend allows configured origins |
| Error Format | Standard HTTP status codes + JSON error body |

### 4.2 Request Flow

```
Browser → Frontend (React Router) → API Client (Axios)
    → [Request Interceptor: inject JWT]
    → Backend (Express + NestJS)
        → [Tenant Middleware: resolve tenant from JWT]
        → [JwtAuthGuard: verify token, attach user]
        → [RBAC check in controller]
        → Service Layer
        → PrismaService [auto-inject tenantId]
        → PostgreSQL [RLS enforcement]
    → Response
    → [Response Interceptor: handle 401 → silent refresh]
    → Component renders data
```

### 4.3 Token Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        TOKEN LIFECYCLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. LOGIN                                                        │
│     Frontend → POST /api/auth/login (or /google, /register)     │
│     Backend → verifies credentials → signs JWT {sub, email}     │
│     Frontend ← receives { accessToken, user }                    │
│     Frontend → stores in localStorage + in-memory                │
│                                                                  │
│  2. AUTHENTICATED REQUESTS                                       │
│     Axios interceptor → adds "Bearer <token>" header            │
│     Backend middleware → jwt.verify(token) → resolve tenant     │
│     JwtAuthGuard → attach req.user + req.dbUser                 │
│                                                                  │
│  3. TOKEN EXPIRY / INVALID                                       │
│     Backend returns 401/403                                      │
│     Axios response interceptor catches it                        │
│     → Gets Supabase session (if available)                       │
│     → POST /api/auth/supabase-exchange → new app JWT            │
│     → Retries original request with new token                    │
│     → If no Supabase session → redirect to login                 │
│                                                                  │
│  4. TENANT SWITCH                                                │
│     Frontend → POST /api/auth/switch-tenant { tenantId }        │
│     Backend → verifies user belongs to tenant → new JWT         │
│     Frontend → stores new token, navigates to /:newSlug/        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Multi-Tenancy Architecture

### 5.1 Strategy: Shared Database, Shared Schema

Every table has a `tenantId` column. Data isolation is enforced at three levels:

```
┌─────────────────────────────────────────────────────────────┐
│  LEVEL 1: Frontend (URL Routing)                            │
│  Routes: /:tenantSlug/*                                     │
│  TenantContext provides current tenant to all components    │
├─────────────────────────────────────────────────────────────┤
│  LEVEL 2: Backend (Application Middleware)                   │
│  Express middleware → JWT → User → tenantId                 │
│  AsyncLocalStorage propagates context                        │
│  Prisma middleware auto-injects WHERE tenantId = ?          │
├─────────────────────────────────────────────────────────────┤
│  LEVEL 3: Database (PostgreSQL RLS)                         │
│  SET LOCAL ROLE app_backend                                  │
│  SET LOCAL app.current_tenant_id = '<id>'                   │
│  RLS policies: tenant_id = current_setting(...)             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Tenant Provisioning

| Action | Flow |
|--------|------|
| **Create Org** | Register with `tenantName` → creates Tenant record + user marked `isOrgAdmin` + seeds default roles |
| **Join Org** | Register with `tenantCode` → adds user to existing tenant |
| **Invite** | Shareable `?invite=CODE` link pre-fills join flow |
| **Switch** | Multi-tenant users can switch via dropdown → new JWT issued |

---

## 6. Feature Module Mapping

| Feature | Frontend Component | Backend Module | API Prefix |
|---------|-------------------|----------------|------------|
| Dashboard | `Dashboard.tsx` | DashboardsModule | `/dashboards` |
| Executive View | `ExecutiveDashboard.tsx` | DashboardsModule | `/dashboards` |
| Projects | `ProjectList.tsx` | PpmProjectsModule | `/ppm/projects` |
| Portfolios | `Portfolios.tsx` | PortfoliosModule | `/portfolios` |
| OKRs | `OkrsPage.tsx` | OkrsModule | `/objectives`, `/key-results` |
| Financials | `FinancialsPage.tsx` | FinancialsModule | `/financials` |
| Capacity | `CapacityPlanningPage.tsx` | ResourcesModule | `/capacity-plans`, `/resource-assignments` |
| RAID | `Raid.tsx` | RaidModule | `/risks`, `/issues`, `/assumptions`, `/dependencies` |
| Roadmaps | `RoadmapsPage.tsx` | RoadmapsModule | `/roadmaps` |
| Strategy | `StrategicAlignmentPage.tsx` | StrategiesModule | `/strategies`, `/strategic-categories` |
| Beacon | `Beacon.tsx`, `BeaconEditor.tsx` | BeaconModule | `/beacon` |
| Timesheets | `TimesheetsPage.tsx` | TimesheetsModule | `/timesheets` |
| Team | `TeamManagement.tsx` | UsersModule | `/users` |
| Admin | `OrgAdminPage.tsx` | UsersModule + AuthModule | `/users`, `/api/auth` |
| Settings | `Settings.tsx` | IntegrationsConfigModule | `/integrations-config` |
| Discussions | `discussions/` | DiscussionsModule | `/discussions` |
| Notifications | — (header widget) | NotificationsModule | `/notifications` |
| Jira Sync | (within Projects) | JiraModule | `/api/jira` |

---

## 7. Authentication & Authorization (End-to-End)

### 7.1 Auth Providers

| Provider | Purpose | Frontend | Backend |
|----------|---------|----------|---------|
| Email/Password | Direct registration/login | `authApi.register/login` | bcrypt hash + JWT sign |
| Google OAuth | Social sign-in | Supabase OAuth redirect | google-auth-library verification |
| Supabase | Session management + OAuth broker | `@supabase/supabase-js` | JWKS/HS256 token verification |

### 7.2 RBAC System

```
Permission Resolution Order:
  1. isOrgAdmin === true  →  FULL ACCESS (all pages, all actions)
  2. TenantRole.permissions  →  Role-level page access
  3. User.permissions  →  Per-user overrides

Permission Structure:
  {
    pages: {
      "dashboard": { view: true, edit: true, viewMode: "global" },
      "projects": { view: true, edit: false, viewMode: "team" },
      ...
    }
  }

View Modes:
  - "single"  →  Only own data
  - "team"    →  Own + direct reports' data
  - "global"  →  All tenant data
```

### 7.3 Frontend Guard Implementation

- Sidebar navigation items: filtered by `hasPageAccess(dbUser, pageId)`
- Route rendering: wrapped in `guardPage(pageId)` → redirect if unauthorized
- Edit buttons: conditionally rendered via `canEditPage(dbUser, pageId)`

### 7.4 Backend Guard Implementation

- `@UseGuards(JwtAuthGuard)` on protected controllers
- Controller methods call `canAccessPage(req, pageId)` / `canEditPage(req, pageId)`
- Returns 403 if unauthorized

---

## 8. Integration Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     NEXUS BACKEND                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │  JiraModule  │    │ JellyfishMod │                   │
│  │              │    │              │                   │
│  │ • Projects   │    │ • Eng Metrics│                   │
│  │ • Boards     │    │              │                   │
│  │ • Issues     │    └──────┬───────┘                   │
│  │ • Search     │           │                           │
│  │ • Transitions│           │ REST API                  │
│  └──────┬───────┘           ▼                           │
│         │           ┌──────────────┐                    │
│         │ REST API  │  Jellyfish   │                    │
│         ▼           │  Service     │                    │
│  ┌──────────────┐   └──────────────┘                    │
│  │  Jira Cloud  │                                       │
│  │  (Atlassian) │   ┌──────────────┐                   │
│  └──────────────┘   │ Confluence   │                    │
│                      │ Module       │                    │
│  ┌──────────────┐   └──────┬───────┘                    │
│  │  Supabase    │           │ REST API                  │
│  │  • Auth      │           ▼                           │
│  │  • PostgreSQL│   ┌──────────────┐                    │
│  │  • JWKS      │   │ Confluence   │                    │
│  └──────────────┘   │ Cloud        │                    │
│                      └──────────────┘                    │
│  ┌──────────────┐                                       │
│  │ Google OAuth │                                       │
│  │ (ID Token)   │                                       │
│  └──────────────┘                                       │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Integration Configuration

Per-tenant integration settings stored in `Integration` table:
- `type`: jira, confluence, jellyfish, etc.
- `baseUrl`, `apiToken`, `apiKey`, `email`
- Managed via `/integrations-config` endpoint

---

## 9. Deployment Architecture

### 9.1 Docker Deployment

```
┌─────────────────────────┐     ┌─────────────────────────┐
│  Frontend Container      │     │  Backend Container       │
│  (nginx:alpine)          │     │  (node:20-alpine)        │
│                          │     │                          │
│  • Static SPA files      │     │  • NestJS app            │
│  • SPA fallback routing  │     │  • Prisma client         │
│  • Gzip compression      │     │  • Port 50051            │
│  • Security headers      │     │  • Non-root user         │
│  • Port 80               │     │                          │
└─────────────┬────────────┘     └─────────────┬────────────┘
              │                                 │
              │    REST API                     │
              └────────────────────────────────▶│
                                                │
                                                ▼
                                   ┌────────────────────┐
                                   │  PostgreSQL         │
                                   │  (Supabase Cloud)   │
                                   └────────────────────┘
```

### 9.2 Vercel Deployment

- **Frontend:** Static SPA deployed as Vercel project
- **Backend:** Serverless functions via `api/index.ts` (Express adapter wrapping NestJS)

### 9.3 Production URL

- **Application:** `https://nexus.cetana.io`
- **Allowed CORS origins:** `localhost:3000`, `localhost:80`, `nexus.cetana.io`

---

## 10. Data Flow Examples

### 10.1 Creating a Project

```
1. User fills form in ProjectList.tsx
2. Frontend calls ppmProjectsApi.create(projectData)
3. Axios interceptor adds Bearer token
4. Backend receives POST /ppm/projects
5. Tenant middleware resolves tenantId from JWT → AsyncLocalStorage
6. JwtAuthGuard verifies token, attaches req.dbUser
7. PpmProjectsController checks canEditPage(req, 'projects')
8. PpmProjectsService calls prisma.project.create(data)
9. Prisma middleware auto-injects tenantId into data
10. PostgreSQL RLS validates tenant context
11. Record created → response returned
12. Frontend updates local state, re-renders
```

### 10.2 Multi-Tenant Org Switch

```
1. User clicks org in dropdown (App.tsx)
2. Frontend calls authApi.switchTenant(targetTenantId)
3. Backend verifies user has membership in target tenant
4. Backend issues new JWT with updated tenant association
5. Frontend stores new token (localStorage + in-memory)
6. Frontend navigates to /:newTenantSlug/dashboard
7. TenantContext refreshes → fetches new tenant info
8. All subsequent API calls use new JWT → new tenant scope
```

---

## 11. Security Model

| Layer | Protection |
|-------|-----------|
| **Transport** | HTTPS (platform-enforced) |
| **Authentication** | JWT with HS256 signature |
| **Session Recovery** | Supabase session → transparent token refresh |
| **Authorization** | RBAC with page-level + action-level + view-mode checks |
| **Tenant Isolation** | 3-layer: URL routing + app middleware + database RLS |
| **Password Storage** | bcrypt (10 salt rounds) |
| **SQL Injection** | Prisma parameterized queries |
| **CORS** | Configurable allowed origins |
| **XSS** | React escaping + nginx security headers |
| **Input Validation** | Partial (class-validator on some endpoints) |

---

## 12. Quality Assessment Summary

| Aspect | Frontend | Backend | Overall |
|--------|----------|---------|---------|
| **Test Coverage** | ❌ None (placeholder only) | ❌ None | 🔴 Critical Risk |
| **Type Safety** | ✅ TS strict mode | ✅ TS strict mode | ✅ Good |
| **Architecture** | ✅ Clean routing + RBAC | ✅ NestJS modular | ✅ Good |
| **Multi-Tenancy** | ✅ URL-scoped | ✅ 3-layer isolation | ✅ Excellent |
| **Auth** | ✅ Multi-provider + recovery | ✅ JWT + OAuth + RBAC | ✅ Comprehensive |
| **Code Quality** | 🟡 No linting config | 🟡 No linting config | 🟡 Needs tooling |
| **Performance** | 🟡 No code splitting | ✅ Serverless-ready | 🟡 Frontend concern |
| **Deployment** | ✅ Docker + Vercel | ✅ Docker + Vercel | ✅ Good |
| **Documentation** | 🟡 No README/docs | 🟡 Swagger (partial) | 🟡 Minimal |
| **Error Handling** | 🟡 No error boundaries | 🟡 Inconsistent | 🟡 Needs improvement |

---

## 13. Risk Register

| # | Risk | Impact | Likelihood | Mitigation |
|---|------|--------|------------|------------|
| 1 | Zero test coverage across both repos | High | Certain | Implement tiered test strategy (unit → integration → E2E) |
| 2 | CRA build tool is unmaintained | Medium | High | Migrate to Vite (straightforward for CRA projects) |
| 3 | No structured logging on backend | Medium | High | Add Winston/Pino with correlation IDs |
| 4 | Frontend bundle size (no code splitting) | Medium | Medium | Add React.lazy + route-based splitting |
| 5 | Some API routes missing auth guards | High | Low | Audit all controllers, add JwtAuthGuard |
| 6 | No rate limiting | Medium | Medium | Add @nestjs/throttler |
| 7 | Single-region deployment | Medium | Low | Consider multi-region if user base grows |
| 8 | No database backup verification | High | Low | Add automated backup + restore testing |

---

## 14. Recommended Improvement Roadmap

### Phase 1: Foundation (Weeks 1–2)
- [ ] Add ESLint + Prettier to both repos
- [ ] Migrate frontend from CRA to Vite
- [ ] Add error boundaries to frontend
- [ ] Add rate limiting to backend
- [ ] Remove hardcoded secrets defaults
- [ ] Add proper health checks (DB connectivity)

### Phase 2: Quality (Weeks 3–6)
- [ ] Set up Jest + Supertest for backend integration tests
- [ ] Write tests for auth flows (registration, login, OAuth, tenant switch)
- [ ] Add React Testing Library tests for critical frontend flows
- [ ] Add structured logging (backend)
- [ ] Implement React Query for frontend data caching
- [ ] Add code splitting (React.lazy per route)

### Phase 3: Resilience (Weeks 7–12)
- [ ] Achieve 50%+ backend service test coverage
- [ ] Add E2E tests with Playwright
- [ ] Implement API versioning
- [ ] Add performance monitoring (Core Web Vitals + APM)
- [ ] Security audit (OWASP Top 10 checklist)
- [ ] Add database connection pooling optimization
- [ ] Implement audit trail for admin actions

### Phase 4: Scale (Months 3–6)
- [ ] Evaluate state management upgrade (Zustand/Jotai)
- [ ] Implement optimistic updates for frequent operations
- [ ] Add WebSocket/SSE for real-time notifications
- [ ] Multi-region deployment evaluation
- [ ] Performance load testing for multi-tenant isolation
- [ ] Accessibility audit (WCAG 2.1 AA)

---

## 15. Key Architectural Decisions (ADRs)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Multi-tenancy model | Shared DB + shared schema | Simpler ops, easier cross-tenant reporting, lower cost |
| Frontend framework | React + MUI | Enterprise UI library, large ecosystem, component density |
| Backend framework | NestJS | Modular architecture, TypeScript-first, DI container |
| ORM | Prisma | Type-safe queries, migration system, schema-as-code |
| Auth strategy | JWT + Supabase | Stateless tokens + managed OAuth provider |
| Deployment | Docker + Vercel | Flexibility (container or serverless) |
| Tenant isolation | 3-layer (app + middleware + RLS) | Defense-in-depth against data leakage |
| State management | React Context | Sufficient for current complexity; avoid over-engineering |
| Build tool (FE) | CRA | Legacy choice (should be migrated to Vite) |
