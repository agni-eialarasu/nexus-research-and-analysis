# Nexus v2 — Architecture Decision Record

**Date:** August 2026  
**Status:** Approved  
**Context:** Complete rebuild of Nexus PPM platform from React/NestJS to Flutter/Supabase

---

## 1. Decision Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| UI Framework | Flutter (Web-first, responsive) | Multi-platform path (web now → iOS/Android later), single codebase |
| Backend | Supabase (BaaS) + Edge Functions | Eliminates custom backend maintenance, built-in auth/realtime/storage |
| Database | PostgreSQL (Supabase-hosted) | Same engine as v1, proven for the domain |
| Multi-Tenancy | Shared schema + RLS + JWT claims | Supabase-native, defense-in-depth, cost-effective |
| Authentication | Supabase Auth (email, Google OAuth) | Built-in org management, JWT with custom claims |
| Authorization | Role-Based Access Control (RBAC) via RLS + app-level | Page/feature/action gating with view modes |
| State Management | Riverpod 3 (code generation) | Stream-first, testable, Supabase Realtime fit |
| Routing | GoRouter | Web-URL aware, deep linking, declarative |
| Realtime | Supabase Realtime (PostgreSQL changes) | Native RLS-respecting streams |
| Deployment (Web) | Vercel | PR previews, CDN, proven workflow |
| Deployment (Backend) | Supabase CLI | Migrations, Edge Functions, config |
| Repository | Monorepo | Solo dev efficiency, atomic feature PRs |
| Testing | Unit + Widget + RLS tests from day one | Avoid v1's zero-test debt |

---

## 2. Platform & Target

### 2.1 Flutter Web-First

**Decision:** Build for Flutter Web with responsive design (small/medium/large breakpoints). iOS and Android will be added in a future phase using the same codebase.

**Breakpoints:**
| Size | Width | Layout |
|------|-------|--------|
| Small (Mobile) | < 600px | Single column, bottom nav |
| Medium (Tablet) | 600–1024px | Two-column, side nav collapsed |
| Large (Desktop) | > 1024px | Full layout, side nav expanded |

**Why Flutter over React (v1):**
- Single codebase for web + future mobile
- Strong type system (Dart)
- Rich widget ecosystem for enterprise UIs
- Excellent Supabase SDK support
- Avoids CRA deprecation issue from v1

### 2.2 Web Rendering Strategy

**Decision:** Use Flutter's **CanvasKit** renderer for web (better fidelity) with fallback to HTML renderer for SEO-sensitive pages if needed.

---

## 3. Backend Architecture

### 3.1 Supabase as Primary Backend

**Decision:** Use Supabase as the complete backend platform — no separate NestJS/Express server.

```
┌─────────────────────────────────────────────────────────────────┐
│                     SUPABASE PLATFORM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Auth         │  │  Database    │  │  Edge Functions       │  │
│  │              │  │  (PostgreSQL)│  │  (Deno/TypeScript)    │  │
│  │  • Email/Pwd │  │              │  │                       │  │
│  │  • Google    │  │  • Tables    │  │  • Tenant setup       │  │
│  │  • JWT +     │  │  • Views     │  │  • Complex queries    │  │
│  │    claims    │  │  • RLS       │  │  • Integrations       │  │
│  │  • Org mgmt  │  │  • Triggers  │  │  • Scheduled jobs     │  │
│  │              │  │  • Functions  │  │  • Webhook handlers   │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │  Realtime    │  │  Storage     │                             │
│  │              │  │              │                             │
│  │  • DB changes│  │  • File      │                             │
│  │  • Presence  │  │    uploads   │                             │
│  │  • Broadcast │  │  • Avatars   │                             │
│  └──────────────┘  └──────────────┘                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Edge Functions — When to Use

| Use Case | Why Edge Function (not direct DB) |
|----------|-----------------------------------|
| Tenant provisioning (create org) | Multi-step: create tenant → assign admin → seed defaults |
| Complex reports/aggregations | Cross-table joins that would be slow via client |
| External integrations (Jira, etc.) | Secrets management, API key handling |
| Scheduled jobs (notifications digest) | Cron-triggered server-side logic |
| Data validation beyond RLS | Business rules that span multiple entities |
| Webhook receivers | Inbound events from external systems |

### 3.3 Direct Client Access — When to Use

| Use Case | Why Direct (via Supabase SDK) |
|----------|-------------------------------|
| Standard CRUD operations | RLS handles authorization, no middleware needed |
| Realtime subscriptions | Native stream support |
| Auth operations | Built-in SDK methods |
| File uploads | Direct to Storage with RLS |
| Simple filtered queries | PostgREST is efficient |

---

## 4. Multi-Tenancy Architecture

### 4.1 Strategy: Shared Schema + RLS + JWT Claims

```
┌──────────────────────────────────────────────────────────────┐
│  TENANT ISOLATION — 3 LAYERS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Layer 1: Application (Flutter)                               │
│  • Tenant context from auth session                           │
│  • UI scoped to current org                                   │
│  • Route: /org/:slug/...                                      │
│                                                               │
│  Layer 2: API (Supabase PostgREST + RLS)                     │
│  • JWT contains org_id in app_metadata                        │
│  • RLS policy: tenant_id = auth.jwt()->'app_metadata'->>'org_id'│
│  • Every query auto-filtered by RLS                           │
│                                                               │
│  Layer 3: Database (PostgreSQL RLS)                           │
│  • Policies enforced at DB engine level                       │
│  • Even Edge Functions (service_role) bypass intentionally    │
│  • No code path can accidentally leak cross-tenant data       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 JWT Custom Claims

When a user logs in or switches tenant, their JWT includes:

```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "app_metadata": {
    "org_id": "tenant-cuid",
    "org_slug": "acme-corp",
    "role": "admin",
    "permissions": { ... }
  }
}
```

### 4.3 RLS Policy Pattern

```sql
-- Every table follows this pattern:
CREATE POLICY "tenant_isolation" ON projects
  USING (tenant_id = (auth.jwt()->'app_metadata'->>'org_id')::uuid);

-- Write policies add role check:
CREATE POLICY "tenant_write" ON projects
  FOR INSERT
  WITH CHECK (
    tenant_id = (auth.jwt()->'app_metadata'->>'org_id')::uuid
  );
```

---

## 5. Role-Based Access Control (RBAC)

### 5.1 RBAC Model

```
┌─────────────────────────────────────────────────────────────────┐
│                        RBAC HIERARCHY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Org Admin (isOrgAdmin = true)                                   │
│  └── Full access to all pages, all actions, all data             │
│                                                                  │
│  Tenant Roles (configurable per org)                             │
│  ├── "Project Manager"                                           │
│  │   └── pages: { projects: rw, portfolios: rw, okrs: rw, ... }│
│  ├── "Team Member"                                               │
│  │   └── pages: { projects: r, timesheets: rw, ... }           │
│  ├── "Viewer"                                                    │
│  │   └── pages: { dashboard: r, projects: r, ... }             │
│  └── Custom roles (org-defined)                                  │
│                                                                  │
│  User-Level Overrides                                            │
│  └── Fine-grained per-user permission adjustments                │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                       VIEW MODES                                  │
├─────────────────────────────────────────────────────────────────┤
│  • "own"    → User sees only their own data                      │
│  • "team"   → User sees own + direct reports' data               │
│  • "global" → User sees all tenant data                          │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Implementation Layers

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| **Database (RLS)** | Policies check `app_metadata.org_id` | Tenant isolation (hard boundary) |
| **Database (RLS + functions)** | `check_permission(page, action)` function | Page/action access |
| **Edge Function** | Claims management, role assignment | Admin operations |
| **Flutter App** | `ref.watch(permissionsProvider)` | UI gating (show/hide, enable/disable) |
| **Flutter Router** | GoRouter redirect guards | Route-level access control |

### 5.3 Permission Structure

```dart
// Permission model (Dart)
@freezed
class Permissions with _$Permissions {
  const factory Permissions({
    required Map<String, PagePermission> pages,
  }) = _Permissions;
}

@freezed
class PagePermission with _$PagePermission {
  const factory PagePermission({
    @Default(false) bool view,
    @Default(false) bool edit,
    @Default(ViewMode.own) ViewMode viewMode,
  }) = _PagePermission;
}

enum ViewMode { own, team, global }
```

### 5.4 RBAC in RLS (Advanced Policies)

```sql
-- Example: Projects visible based on view_mode
CREATE POLICY "projects_view_mode" ON projects FOR SELECT USING (
  tenant_id = current_tenant_id()
  AND (
    -- Global viewers see all
    get_view_mode('projects') = 'global'
    -- Team viewers see own + direct reports
    OR (get_view_mode('projects') = 'team' 
        AND (lead_user_id = auth.uid() 
             OR lead_user_id IN (SELECT id FROM users WHERE manager_id = auth.uid())))
    -- Own viewers see only assigned
    OR (get_view_mode('projects') = 'own' 
        AND lead_user_id = auth.uid())
  )
);
```

---

## 6. State Management Architecture

### 6.1 Riverpod 3 Provider Organization

```
lib/providers/
├── auth/
│   ├── auth_provider.dart          # Supabase auth state
│   ├── current_user_provider.dart  # Current user profile
│   └── permissions_provider.dart   # RBAC permissions
├── tenant/
│   ├── tenant_provider.dart        # Current tenant context
│   └── tenant_list_provider.dart   # User's tenant memberships
├── features/
│   ├── projects_provider.dart      # Project CRUD + realtime
│   ├── portfolios_provider.dart
│   ├── okrs_provider.dart
│   └── ...
└── core/
    ├── supabase_provider.dart      # Supabase client instance
    └── connectivity_provider.dart  # Network state
```

### 6.2 Provider Pattern

```dart
// Realtime stream provider (auto-updates via Supabase Realtime)
@riverpod
Stream<List<Project>> projects(ref) {
  final supabase = ref.watch(supabaseProvider);
  final tenantId = ref.watch(currentTenantProvider).id;
  
  return supabase
    .from('projects')
    .stream(primaryKey: ['id'])
    .eq('tenant_id', tenantId)
    .map((data) => data.map(Project.fromJson).toList());
}

// Mutation provider
@riverpod
class ProjectNotifier extends _$ProjectNotifier {
  @override
  FutureOr<void> build() {}
  
  Future<void> create(CreateProjectDto dto) async {
    final supabase = ref.read(supabaseProvider);
    await supabase.from('projects').insert(dto.toJson());
    // Realtime will auto-update the stream provider
  }
}
```

---

## 7. Realtime Architecture

### 7.1 Supabase Realtime Usage

| Feature | Realtime Channel | Purpose |
|---------|-----------------|---------|
| Projects | `projects` table changes | Live project status updates |
| Notifications | `notifications` table inserts | Instant notification delivery |
| Discussions | `discussion_posts` inserts | Live chat/comments |
| Kanban Board | `kanban_cards` updates | Real-time board changes |
| Presence | Custom channel | Who's online / viewing same page |

### 7.2 Realtime + RLS

Supabase Realtime respects RLS policies — users only receive events for rows they're authorized to see. No additional filtering needed on the client.

---

## 8. Development Process

### 8.1 Sprint Protocol

| Command | Action |
|---------|--------|
| `/sprint-start` | Create `sprint/N` branch, update tracker |
| `/sprint-finish` | Push, create PR, update tracker |
| `/sprint-update` | Post-merge docs (valuation, evaluation) |
| `/release-start` | Staging smoke test verification |
| `/release-finish` | Production promotion |

### 8.2 CI/CD Pipeline

```
PR → main:
  ├── flutter analyze
  ├── flutter test (unit + widget)
  ├── supabase db lint (migration check)
  └── flutter build web (compilation check)

Merge to main:
  ├── supabase db push (apply migrations)
  ├── supabase functions deploy (edge functions)
  └── vercel deploy (Flutter web)
```

### 8.3 Testing Requirements per PR

Every feature PR must include:
1. ✅ Unit tests for models and utilities
2. ✅ Widget tests for new UI components
3. ✅ RLS policy tests (SQL) for new/modified tables
4. ✅ `flutter analyze` clean (zero warnings)

### 8.4 Quality Gates

- `flutter analyze` — zero warnings
- Test coverage — minimum 60% for new code
- RLS coverage — 100% of tables must have isolation policies
- No hardcoded strings (i18n-ready from start)

---

## 9. Migration Path from v1

### 9.1 Database

| Approach | Action |
|----------|--------|
| Schema | Redesign from scratch (informed by v1 Prisma schema) |
| Data | Export from v1 PostgreSQL → transform → import to v2 |
| Users | Migrate to Supabase Auth (password hashes compatible if bcrypt) |
| RLS | Write from scratch (v1 had RLS but relied more on middleware) |

### 9.2 Features

Prioritization pending stakeholder discussion. Schema design will begin once MVP feature scope is confirmed (items 6, 7, 8).

---

## 10. Key Differences from v1

| Aspect | v1 (Current) | v2 (Rebuild) |
|--------|--------------|--------------|
| Frontend | React 18 + CRA + MUI | Flutter Web (responsive) |
| Backend | NestJS + Prisma + Express | Supabase (no custom server) |
| Auth | Custom JWT + bcrypt + Supabase bridge | Supabase Auth (native) |
| Multi-tenancy | 3-layer (middleware + Prisma + RLS) | 2-layer (JWT claims + RLS) |
| RBAC | Custom middleware checks | RLS policies + app-level guards |
| State | React Context (no caching) | Riverpod 3 (streams + cache) |
| Realtime | None (polling) | Supabase Realtime (WebSocket) |
| Testing | Zero tests | Mandatory from Sprint 1 |
| Build | CRA (unmaintained) | Flutter (actively maintained) |
| Mobile | None | Future phase (same codebase) |
| Integrations | Jira, Confluence, Jellyfish | TBD (Edge Functions) |
| Deployment | Docker + Vercel | Vercel (web) + Supabase CLI (backend) |

---

## 11. Open Items (Pending Stakeholder Discussion)

- [ ] MVP feature scope — which of the 12+ features ship first?
- [ ] Features to drop from v1
- [ ] New features to add
- [ ] Timeline expectations
- [ ] Data migration strategy (fresh start vs import from v1)
- [ ] Integration priorities (keep Jira? Add others?)

---

*This document will be updated as decisions are finalized.*
