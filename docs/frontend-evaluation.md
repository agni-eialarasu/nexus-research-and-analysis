# Nexus Frontend — Technical Evaluation

**Repository:** `agni-eialarasu/nexus-frontend`  
**Analysis Date:** August 2026  
**Status:** Reverse-engineered from source code

---

## 1. Executive Summary

Nexus Frontend is a **React 18** single-page application built with **TypeScript** and **Material UI (MUI) v5**. It serves as the user interface for a multi-tenant Project Portfolio Management platform, providing dashboards, project views, financial tracking, OKR management, capacity planning, and a business capability model (Beacon). The app implements tenant-scoped routing, role-based access control, and multi-provider authentication (email/password, Google OAuth via Supabase).

### Key Strengths
- Comprehensive feature coverage matching all backend capabilities
- Solid multi-tenant architecture with tenant-scoped routing
- Silent session recovery (transparent token refresh)
- Clean design system with well-defined tokens
- TypeScript strict mode throughout
- RBAC-gated navigation and routes

### Key Risks
- **No meaningful test coverage** (single placeholder test)
- Using legacy Create React App (CRA) — no longer maintained
- No external state management library (Context + local state only)
- Large `App.tsx` component with too many responsibilities
- No error boundary implementation
- No code splitting / lazy loading

---

## 2. Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | React | 18.x |
| Language | TypeScript | (strict mode) |
| Build Tool | Create React App (react-scripts) | 5.0.1 |
| UI Library | Material UI (MUI) | ^5.15 |
| Routing | react-router-dom | ^6.26 |
| HTTP Client | Axios | ^1.13 |
| Auth Provider | @supabase/supabase-js | ^2.95 |
| Google OAuth | @react-oauth/google | — |
| Charts | chart.js + react-chartjs-2, recharts | — |
| Tables | @tanstack/react-table v8, @mui/x-data-grid | — |
| Drag & Drop | @dnd-kit/core + @dnd-kit/sortable | — |
| Calendar | react-big-calendar | — |
| Rich Text | react-quill | — |
| PDF Export | jspdf + jspdf-autotable | — |
| Spreadsheets | xlsx | — |
| Grid Layout | react-grid-layout | — |
| Date Utils | date-fns | ^3 |
| Styling | @emotion/react, @emotion/styled, sass | — |

---

## 3. Architecture

### 3.1 Component Architecture

**Pattern:** Page-level flat components with feature sub-directories. No atomic design.

```
src/
├── index.tsx              # Entry point (ThemeProvider + CssBaseline)
├── App.tsx                # Root orchestrator (auth, routing, layout)
├── api/                   # API layer
│   ├── client.ts          # Axios instance + all API namespaces
│   ├── ppm.ts             # PPM-specific APIs
│   └── supabase-api.ts    # Supabase helpers
├── components/            # Page-level components
│   ├── Login.tsx          # Multi-flow authentication
│   ├── Dashboard.tsx      # Main dashboard
│   ├── ExecutiveDashboard.tsx
│   ├── ProjectList.tsx
│   ├── Portfolios.tsx / PortfolioDetail.tsx
│   ├── Beacon.tsx / BeaconEditor.tsx
│   ├── OkrsPage.tsx
│   ├── RoadmapsPage.tsx
│   ├── Raid.tsx
│   ├── FinancialsPage.tsx
│   ├── TimesheetsPage.tsx
│   ├── CapacityPlanningPage.tsx
│   ├── StrategicAlignmentPage.tsx
│   ├── TeamManagement.tsx
│   ├── OrgAdminPage.tsx / OrgChart.tsx
│   ├── Settings.tsx / UserSettings.tsx
│   ├── layout/            # Reusable layout (PageHeader, StatCard, TableCard)
│   ├── discussions/       # Inline discussion panel
│   ├── charts/            # Chart components
│   ├── grid/              # Dashboard grid components
│   └── capacity/          # Capacity planning sub-components
├── contexts/              # React Contexts
│   ├── TenantContext.tsx  # Multi-tenant context provider
│   └── DiscussionContext.tsx
├── hooks/                 # Custom hooks
│   ├── useCurrentUser.ts
│   └── useTenantNav.ts
├── lib/
│   └── supabase.ts        # Supabase client initialization
├── theme/
│   ├── nexusTheme.ts      # MUI theme overrides
│   └── tokens.ts          # Design tokens
├── types/
│   ├── index.ts           # Domain type definitions
│   └── beacon.ts          # Beacon-specific types
├── utils/
│   ├── rbac.ts            # Permission resolution
│   ├── beaconScoring.ts   # Capability scoring
│   ├── finance.ts         # Financial calculations
│   └── format.ts          # Formatting utilities
└── data/
    └── defaultData.ts     # Default/seed data
```

### 3.2 Routing Structure

All routes are **tenant-scoped** under `/:tenantSlug/`:

| Route | Component | Access |
|-------|-----------|--------|
| `/:slug/overview` | ExecutiveDashboard | RBAC-gated |
| `/:slug/dashboard` | Dashboard | Default landing |
| `/:slug/strategic-alignment` | StrategicAlignmentPage | RBAC-gated |
| `/:slug/beacon` | Beacon | RBAC-gated |
| `/:slug/portfolios` | Portfolios | RBAC-gated |
| `/:slug/portfolios/:id` | PortfolioDetail | RBAC-gated |
| `/:slug/requests` | Requests | RBAC-gated |
| `/:slug/projects` | ProjectList | RBAC-gated |
| `/:slug/projects/:key` | ProjectList (detail) | RBAC-gated |
| `/:slug/projects/internal/:id` | InternalProjectDetail | RBAC-gated |
| `/:slug/financials` | FinancialsPage | RBAC-gated |
| `/:slug/capacity` | CapacityPlanningPage | RBAC-gated |
| `/:slug/okrs` | OkrsPage | RBAC-gated |
| `/:slug/roadmaps` | RoadmapsPage | RBAC-gated |
| `/:slug/raid` | Raid | RBAC-gated |
| `/:slug/templates` | TemplatesPage | RBAC-gated |
| `/:slug/timesheets` | TimesheetsPage | RBAC-gated |
| `/:slug/settings` | Settings | RBAC-gated |
| `/:slug/user-settings` | UserSettings | RBAC-gated |
| `/:slug/team` | TeamManagement | RBAC-gated |
| `/:slug/management` | OrgAdminPage | Admin only |

**Route Guard:** `guardPage(pageId)` function checks `hasPageAccess(dbUser, pageId)` — unauthorized users are redirected to dashboard.

### 3.3 Application Shell

```
┌─────────────────────────────────────────────────────┐
│  Header (Profile Dropdown, Org Switcher, Invite)     │
├────────┬────────────────────────────────────────────┤
│        │                                            │
│  Side  │           Route Content                    │
│  bar   │         (Page Component)                   │
│        │                                            │
│ (nav   │                                            │
│  items │                                            │
│  RBAC  │                                            │
│  gated)│                                            │
│        │                                            │
├────────┴────────────────────────────────────────────┤
│  Inline Discussion Panel (slide-in overlay)          │
└─────────────────────────────────────────────────────┘
```

---

## 4. State Management

### 4.1 Pattern

**React Context + Local Component State** — No Redux, Zustand, or other state library.

### 4.2 Contexts

| Context | Purpose | Key Values |
|---------|---------|------------|
| `TenantContext` | Multi-tenant identity | `currentTenant`, `isLoading`, `refreshTenant()` |
| `DiscussionContext` | Discussion panel state | `override`, `openDiscussion()`, `clearOverride()` |

### 4.3 App-Level State (in `AppContent`)

| State | Type | Purpose |
|-------|------|---------|
| `token` | string | JWT access token |
| `dbUser` | User object | Current user profile from API |
| `myTenants` | Tenant[] | User's tenant memberships |
| `sidebarCollapsed` | boolean | Sidebar UI toggle |
| `selectedProjectId` | string | Currently viewed project |
| Various modals | boolean | Add org, profile, etc. |

### 4.4 Assessment

The Context approach works for current complexity, but risks include:
- **Prop drilling** for deeply nested components
- **Re-render cascading** when top-level state changes (token, dbUser)
- **No caching layer** for API responses (every mount re-fetches)
- **No optimistic updates** pattern

---

## 5. API Integration

### 5.1 Client Setup (`src/api/client.ts`)

```
Base URL: REACT_APP_API_URL || 'http://localhost:3001'
```

**Request Interceptor:**
- Injects `Authorization: Bearer <token>` from in-memory variable or localStorage

**Response Interceptor (Silent Session Recovery):**
1. On 401/403 response → gets current Supabase session
2. Calls `/api/auth/supabase-exchange` with Supabase token
3. Updates stored app JWT
4. Retries the original failed request
5. If recovery fails → clears all tokens, redirects to login

### 5.2 API Namespaces

| Namespace | Endpoints |
|-----------|-----------|
| `authApi` | login, register, google, supabaseExchange, tenantLookup, switchTenant |
| `usersApi` | me, ensure, list, update, permissions, roles, orgChart |
| `projectsApi` | CRUD for projects |
| `tasksApi` | CRUD for tasks |
| `beaconApi` | domains, capabilities, subCapabilities, applications, initiatives |
| `jiraApi` | projects, boards, issues, search, transitions, comments |
| `confluenceApi` | spaces, pages, comments |
| `dashboardsApi` | executive dashboard data |
| `ppmProjectsApi` | PPM projects with portfolios, budgets, members |

---

## 6. UI & Design System

### 6.1 Theme Philosophy

**"Minimal / Flat"** — White surfaces, hairline borders, single accent color, generous whitespace.

### 6.2 Design Tokens (`theme/tokens.ts`)

| Token Category | Values |
|----------------|--------|
| **Accent** | #2563EB (Blue) |
| **Neutrals** | Gray scale (50–900) |
| **Semantic** | Success (green), Danger (red), Warning (amber), Info (blue) |
| **Border Radius** | sm(6), md(8), lg(12), pill(999) |
| **Shadows** | Single `pop` shadow for floating elements |
| **Typography** | Inter font family |
| **Chart Palette** | 8 distinct colors |

### 6.3 MUI Theme Overrides

- All Cards: `outlined` variant, no elevation
- Buttons: `disableElevation: true`
- Inputs: Outlined style, 8px border radius
- Paper: No elevation, 1px border
- Light mode only

### 6.4 Additional UI Libraries

- **@mui/x-data-grid** — Advanced data tables
- **@mui/x-date-pickers** — Date/time selection
- **chart.js + recharts** — Data visualization
- **react-grid-layout** — Drag-and-drop dashboard widgets
- **@dnd-kit** — General drag-and-drop
- **react-big-calendar** — Calendar views
- **react-quill** — Rich text editing

---

## 7. Authentication Flow

### 7.1 Login Methods

```
┌───────────────────────────────────────────────────────────┐
│                      Login Screen                          │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────┐  ┌─────────────────────────────┐   │
│  │ Email/Password  │  │ Google OAuth (via Supabase)  │   │
│  │ → authApi.login │  │ → supabase.auth.signInWith   │   │
│  │ → JWT returned  │  │   OAuth({provider:'google'}) │   │
│  └────────┬────────┘  └──────────────┬──────────────┘   │
│           │                           │                   │
│           ▼                           ▼                   │
│    Store token              Supabase session obtained     │
│    Set dbUser               → authApi.supabaseExchange    │
│    Navigate to              → App JWT returned            │
│    /:slug/dashboard         → Store token, set dbUser     │
│                                                           │
├───────────────────────────────────────────────────────────┤
│  Multi-Tenant Handling:                                   │
│  • No tenant → Show create/join org form                  │
│  • Single tenant → Auto-login                             │
│  • Multiple tenants → Tenant picker                       │
├───────────────────────────────────────────────────────────┤
│  Invite Flow:                                             │
│  • ?invite=CODE → Pre-fills tenant join                   │
│  • Preserved across OAuth redirect via localStorage       │
└───────────────────────────────────────────────────────────┘
```

### 7.2 Session Lifecycle

| Event | Action |
|-------|--------|
| Page load | Check localStorage for token → validate via `/users/me` |
| OAuth redirect | Detect `?code=` or `#access_token` → process Supabase session |
| 401/403 on API call | Interceptor: refresh via Supabase → retry |
| Recovery fails | Clear all tokens, redirect to login |
| Logout | Clear localStorage, `supabase.auth.signOut()`, redirect to `/` |
| Tenant switch | `authApi.switchTenant()` → new JWT → navigate to new slug |

---

## 8. Configuration & Environment

### 8.1 Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `REACT_APP_API_URL` | `http://localhost:3001` | Backend API base URL |
| `REACT_APP_SUPABASE_URL` | — | Supabase project URL |
| `REACT_APP_SUPABASE_ANON_KEY` | — | Supabase anonymous key |

### 8.2 Docker Build Args

| Arg | Purpose |
|-----|---------|
| `CODEARTIFACT_AUTH_TOKEN` | Private npm registry token (AWS CodeArtifact) |

### 8.3 TypeScript Configuration

- Target: ES5
- Module: ESNext
- Strict mode: enabled
- JSX: react-jsx (automatic runtime)
- `noEmit: true` (CRA handles compilation)

---

## 9. Build & Deployment

### 9.1 Development

```bash
npm start  # CRA dev server, port 3000
```

Docker development stage: `node:20-alpine`, hot-reload, port 3000.

### 9.2 Production

```bash
npm run build  # CRA production build → /app/build
```

Docker production stage:
- **nginx:alpine** serves static files
- SPA fallback: `try_files $uri $uri/ /index.html`
- Gzip compression enabled
- 1-year cache headers for static assets with `immutable`
- Security headers: X-Frame-Options, X-Content-Type-Options, X-XSS-Protection
- Health check: `/health` endpoint

### 9.3 Deployment Targets

- **Docker** (multi-stage: dev + prod/nginx)
- **Vercel** (indicated by `VERCEL_ENV_SETUP.md`)

---

## 10. Testing

### Current State: ❌ MINIMAL (Placeholder Only)

- Single test file: `src/App.test.tsx`
- Content: `renders learn react link` — **doesn't match actual app** (looks for text that doesn't exist)
- No component tests
- No integration tests
- No E2E tests

**Testing Libraries Available (unused):**
- `@testing-library/react` v16
- `@testing-library/jest-dom` v6
- `@testing-library/user-event` v13
- Jest (via CRA)

**Risk Assessment:** HIGH — Complex multi-page app with auth flows, RBAC, tenant switching, and 20+ page components has zero working tests.

---

## 11. Code Quality

### 11.1 Positive Patterns

- TypeScript strict mode with good type coverage
- Well-defined design tokens and consistent theme
- RBAC utility with clear permission resolution
- Silent session recovery prevents UX disruption
- Typed API client with organized namespaces
- Proper interfaces for domain objects

### 11.2 Concerns

| Area | Issue | Severity |
|------|-------|----------|
| **Testing** | Zero working tests | 🔴 Critical |
| **Build Tool** | CRA is unmaintained (should migrate to Vite) | 🟡 Medium |
| **App.tsx** | Single 600+ line component doing too much | 🟡 Medium |
| **State** | No caching layer for API data (stale data risk) | 🟡 Medium |
| **Error Boundaries** | No React error boundaries | 🟡 Medium |
| **Code Splitting** | No lazy loading (entire app in one bundle) | 🟡 Medium |
| **Accessibility** | No explicit a11y testing or patterns | 🟡 Medium |
| **i18n** | Hardcoded English strings (no internationalization) | 🟢 Low |
| **Console logs** | Some debug logs left in production code | 🟢 Low |

### 11.3 Bundle Size Concerns

Heavy dependency list without code splitting:
- chart.js + recharts (two charting libraries)
- xlsx (spreadsheet parsing — large)
- jspdf (PDF generation — large)
- react-quill (rich text editor)
- @dnd-kit (drag-and-drop)
- react-big-calendar
- @tanstack/react-table + @mui/x-data-grid (two table solutions)

Estimated bundle size: **2–3 MB+ uncompressed** (without code splitting)

---

## 12. Technical Debt & Risks

### 12.1 Critical

1. **No test coverage** — 20+ page components, complex auth flows, and RBAC entirely untested
2. **CRA deprecation** — Create React App is no longer maintained; security vulnerabilities will accumulate

### 12.2 Medium

3. **Monolithic App.tsx** — Authentication, routing, layout, tenant switching all in one component
4. **Duplicate libraries** — Two charting libs (chart.js + recharts), two table libs (@tanstack + MUI DataGrid)
5. **No error boundaries** — Unhandled errors crash the entire app
6. **No code splitting** — All pages loaded upfront regardless of what user accesses
7. **No data caching** — Every component mount triggers API calls (no SWR/React Query)
8. **`any` type usage** — Some API call sites lose type safety

### 12.3 Low

9. **No PWA/offline support** — Purely online application
10. **No performance monitoring** — No Core Web Vitals tracking
11. **Hardcoded strings** — No i18n infrastructure if needed later

---

## 13. Recommendations

### Quick Wins (1–2 days each)

- [ ] Add React Error Boundary at app root + per-route
- [ ] Migrate from CRA to **Vite** (well-documented migration path)
- [ ] Add `React.lazy()` + `Suspense` for route-level code splitting
- [ ] Remove duplicate libraries (pick one charting lib, one table lib)
- [ ] Extract App.tsx into smaller components (AuthShell, AppLayout, RouteConfig)

### Medium-Term (1–2 weeks)

- [ ] Add **React Query** or **SWR** for API data caching + deduplication
- [ ] Write tests for critical auth flows (login, session recovery, RBAC)
- [ ] Implement proper error boundaries with fallback UI per page
- [ ] Add bundle analysis (`vite-plugin-visualizer` or `source-map-explorer`)
- [ ] Set up Storybook for component documentation

### Strategic (1–2 months)

- [ ] Achieve 40%+ test coverage (focus on auth, RBAC, API client)
- [ ] Implement Zustand or Jotai for shared application state
- [ ] Add Core Web Vitals monitoring (performance budget)
- [ ] Accessibility audit (WCAG 2.1 AA compliance)
- [ ] Implement optimistic updates for frequent operations
- [ ] Add E2E tests with Playwright for critical user flows

---

## 14. Appendix: Dependency List

### Production Dependencies (notable)

| Package | Purpose |
|---------|---------|
| `react` / `react-dom` | Core framework |
| `react-router-dom` | Client-side routing |
| `axios` | HTTP client |
| `@mui/material` + icons + x-data-grid + x-date-pickers | UI components |
| `@supabase/supabase-js` | Auth provider |
| `@react-oauth/google` | Google sign-in |
| `chart.js` + `react-chartjs-2` | Charts |
| `recharts` | Additional charts |
| `@tanstack/react-table` | Data tables |
| `@dnd-kit/core` + `@dnd-kit/sortable` | Drag & drop |
| `react-big-calendar` | Calendar views |
| `react-quill` | Rich text editor |
| `react-grid-layout` | Dashboard layouts |
| `jspdf` + `jspdf-autotable` | PDF export |
| `xlsx` | Excel export/import |
| `date-fns` | Date manipulation |
| `sass` | SCSS support |

### Dev Dependencies

| Package | Purpose |
|---------|---------|
| `typescript` | Type system |
| `@testing-library/*` | Testing utilities (unused) |
| `@types/*` | Type definitions |
