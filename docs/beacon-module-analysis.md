# Beacon Module — Detailed Analysis

**Module:** Business Capability Model ("Beacon Command")  
**Analysis Date:** August 2026  
**Purpose:** Stakeholder deep-dive for rebuild decision-making

---

## 1. Executive Summary

Beacon is the **Business Capability Model** feature within Nexus. It provides a structured, visual framework for organizations to:

1. **Document** what business capabilities they have (hierarchy: Domain → Capability → Sub-capability)
2. **Assess** the maturity, pain, and strategic importance of each capability
3. **Identify gaps** using an automated priority scoring algorithm
4. **Map applications** to the capabilities they support (for rationalization decisions)
5. **Plan investments** through initiatives linked to capabilities they advance

### Target Audiences

| Audience | Primary Use |
|----------|-------------|
| **Executive Leadership Team** | Long-range strategic planning, investment prioritization |
| **Domain VPs** | Annual portfolio planning within their business domain |
| **Enterprise Architecture** | Application rationalization (which apps to invest/tolerate/retire) |

### Business Value Delivered

- Single source of truth for "what the organization does"
- Quantified gap analysis (where to invest next)
- Application landscape mapped to business capabilities
- Investment portfolio visibility (what's funded, what's not)
- Automated priority scoring removes subjective bias from planning

---

## 2. Data Model

### 2.1 Entity Relationship Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         BEACON DATA MODEL                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────┐       ┌──────────────────┐       ┌─────────────────────┐  │
│  │ BeaconDomain │ 1───N │ BeaconCapability  │ 1───N │ BeaconSubCapability │  │
│  │              │       │                  │       │                     │  │
│  │ • name       │       │ • name           │       │ • name              │  │
│  │ • description│       │ • purpose        │       │ • purpose           │  │
│  │ • owner      │       │ • maturity (1-5) │       │ • maturity (1-5)    │  │
│  │ • color      │       │ • pain (1-5)     │       │ • pain (1-5)        │  │
│  │ • sortOrder  │       │ • importance(1-5)│       │ • importance (1-5)  │  │
│  └──────────────┘       │ • targetMaturity │       │ • targetMaturity    │  │
│                          │ • tier           │       └─────────────────────┘  │
│                          │ • spend ($M)     │                                │
│                          │ • owner          │                                │
│                          └────────┬─────────┘                                │
│                                   │                                          │
│                          M:N      │      M:N                                 │
│                    ┌──────────────┼──────────────┐                           │
│                    │              │              │                            │
│                    ▼              │              ▼                            │
│  ┌─────────────────────┐         │   ┌────────────────────┐                 │
│  │ BeaconApplication    │         │   │ BeaconInitiative    │                │
│  │                      │         │   │                     │                │
│  │ • name               │         │   │ • name              │                │
│  │ • vendor             │         │   │ • horizon           │                │
│  │ • type               │         │   │ • cost ($M)         │                │
│  │ • cost ($M/yr)       │         │   │ • status            │                │
│  │ • status (TIME)      │         │   │ • owner             │                │
│  │ • lifecycle          │         │   │ • outcome           │                │
│  └─────────────────────┘         │   │ • confidence        │                │
│                                   │   └────────────────────┘                 │
│                                   │                                          │
│  Junction Tables:                 │                                          │
│  • BeaconCapabilityApplication ───┘                                          │
│  • BeaconInitiativeCapability                                                │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Entity Details

#### BeaconDomain (Top-level grouping)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| name | String | Business domain name | "Customer Operations" |
| description | Text | What this domain covers | "All patient-facing..." |
| owner | String | Domain VP / executive | "Maya Chen — SVP Patient Growth" |
| color | String | Hex color for UI | "#4292c6" |
| sortOrder | Int | Display ordering | 0, 1, 2... |

#### BeaconCapability (Core entity)

| Field | Type | Description | Scale/Values |
|-------|------|-------------|--------------|
| name | String | Capability name | "Order Management" |
| purpose | Text | What this capability does | Free text |
| maturity | Int | Current maturity level | 1 (Ad hoc) → 5 (Optimized) |
| pain | Int | Current pain level | 1 (None) → 5 (Severe) |
| importance | Int | Strategic importance | 1 (Hygiene) → 5 (Mission-critical) |
| targetMaturity | Int | Desired maturity | Auto-derived or manually set |
| tier | String | Differentiation classification | Differentiating / Core / Commodity |
| spend | Float | Annual spend in $M | 0.0 → N |
| owner | String | Capability owner | "Name — Title" format |

**Derived scores (computed at runtime, never stored):**
| Score | Formula | Range |
|-------|---------|-------|
| Target | `current + 2` if importance ≥ 4 AND maturity ≤ 2, else `current + 1` (cap at 5) | 1-5 |
| Gap | `max(0, target - current)` | 0-4 |
| Priority Score (PS) | `importance × pain × (5 - maturity)` | 0-100 |

#### BeaconSubCapability (Granular breakdown)

Same scoring fields as Capability (maturity, pain, importance, target), nested under a specific capability.

#### BeaconApplication (Software systems)

| Field | Type | Description | Values |
|-------|------|-------------|--------|
| name | String | Application name | "Salesforce", "Epic EHR" |
| vendor | String | Vendor/provider | "Salesforce Inc." |
| type | String | Category | "SaaS", "Custom", "EHR" |
| cost | Float | Annual cost ($M) | 0.0 → N |
| status | String | TIME classification | **S**trategic / **I**nvest / **T**olerate / **R**etire |
| lifecycle | String | Current phase | "Production", "Sunset" |

#### BeaconInitiative (Investment plans)

| Field | Type | Description | Values |
|-------|------|-------------|--------|
| name | String | Initiative name | "Billing platform replatform" |
| horizon | String | Planning horizon | Now / Next / Later |
| cost | Float | Investment amount ($M) | 0.0 → N |
| status | String | Current state | Proposed / Approved / In flight / Scaling |
| owner | String | Initiative owner | "Name — Title" |
| outcome | Text | Expected outcome | Free text |
| confidence | String | Delivery confidence | High / Medium / Low |

### 2.3 Relationships

| Relationship | Type | Meaning |
|-------------|------|---------|
| Domain → Capability | One-to-Many | A domain contains multiple capabilities |
| Capability → SubCapability | One-to-Many | A capability has granular sub-capabilities |
| Capability ↔ Application | Many-to-Many | Apps support capabilities (via junction table) |
| Capability ↔ Initiative | Many-to-Many | Initiatives advance capabilities (via junction table) |

---

## 3. Scoring Model (Algorithm)

### 3.1 Input Dimensions

| Dimension | Scale | Meaning |
|-----------|-------|---------|
| **Maturity (M)** | 1–5 | 1=Ad hoc, 2=Repeatable, 3=Defined, 4=Managed, 5=Optimized |
| **Pain (P)** | 1–5 | 1=No pain, 2=Minor, 3=Moderate, 4=Significant, 5=Severe |
| **Strategic Importance (S)** | 1–5 | 1=Hygiene, 2=Support, 3=Important, 4=Critical, 5=Mission-defining |

### 3.2 Derived Scores

```
Target Maturity:
  IF importance >= 4 AND maturity <= 2:
    target = min(5, maturity + 2)    ← aggressive improvement for high-importance, low-maturity
  ELSE:
    target = min(5, maturity + 1)    ← standard +1 improvement
  
  Can be manually overridden (always capped at 5, never below current)

Gap:
  gap = max(0, target - current)

Priority Score (PS):
  PS = importance × pain × (5 - maturity)
  
  Range: 0 to 100 (max: 5 × 5 × 4 = 100)
  Interpretation: Higher PS = more urgent investment need
```

### 3.3 Score Interpretation

| PS Range | Bucket | Action |
|----------|--------|--------|
| 0 | 0 | No action needed |
| 1–20 | 1 | Monitor |
| 21–40 | 2 | Plan improvement |
| 41–60 | 3 | Prioritize investment |
| 61–80 | 4 | Urgent attention |
| 81–100 | 5 | Critical — immediate action |

### 3.4 Recommendation Logic (Detail Panel)

When viewing a capability's detail, the system generates an actionable recommendation:

| Condition | Recommendation |
|-----------|---------------|
| Tier = Differentiating AND gap ≥ 2 | **INVEST** — significant competitive advantage at risk |
| Tier = Core AND pain ≥ 4 | **Fix the pain** — operational pain in core business function |
| Tier = Commodity AND gap ≤ 1 | **Outsource or standardize** — low differentiation, near target |
| Maturity ≥ 4 AND at target | **Defend & extend** — already strong, maintain position |
| Default | **Maintain** — monitor pain & importance for changes |

---

## 4. API Surface

### 4.1 Single Read Endpoint

```
GET /beacon
Authorization: Bearer <JWT>
RBAC: canAccessPage('beacon')

Response: {
  domains: [
    {
      id, name, description, color, owner,
      capabilities: [
        {
          id, name, purpose, maturity, pain, importance,
          tier, spend, owner,
          target (derived), gap (derived), ps (derived),
          applicationIds: [...],
          initiativeIds: [...],
          subCapabilities: [
            { id, name, purpose, maturity, pain, importance, target, gap, ps }
          ]
        }
      ]
    }
  ],
  applications: [
    { id, name, vendor, type, cost, status, lifecycle, capabilityIds: [...] }
  ],
  initiatives: [
    { id, name, horizon, cost, status, owner, outcome, confidence, capabilityIds: [...] }
  ]
}
```

**Key design choice:** A single API call returns the ENTIRE model. All 8 frontend views render from this one response. This avoids multiple round-trips and keeps the UI snappy.

### 4.2 CRUD Endpoints (16 total)

| Entity | Create | Update | Delete |
|--------|--------|--------|--------|
| Domains | `POST /beacon/domains` | `PUT /beacon/domains/:id` | `DELETE /beacon/domains/:id` |
| Capabilities | `POST /beacon/capabilities` | `PUT /beacon/capabilities/:id` | `DELETE /beacon/capabilities/:id` |
| Sub-capabilities | `POST /beacon/sub-capabilities` | `PUT /beacon/sub-capabilities/:id` | `DELETE /beacon/sub-capabilities/:id` |
| Applications | `POST /beacon/applications` | `PUT /beacon/applications/:id` | `DELETE /beacon/applications/:id` |
| Initiatives | `POST /beacon/initiatives` | `PUT /beacon/initiatives/:id` | `DELETE /beacon/initiatives/:id` |

All write operations require `canEditPage('beacon')` permission.

### 4.3 Many-to-Many Link Management

When creating/updating capabilities, applications, or initiatives, the payload includes relationship arrays:

```json
// Creating a capability with app & initiative links:
POST /beacon/capabilities
{
  "domainId": "...",
  "name": "Order Management",
  "maturity": 2,
  "pain": 4,
  "importance": 5,
  "applicationIds": ["app-1", "app-2"],
  "initiativeIds": ["init-1"]
}
```

The service uses **delete-and-recreate** for junction tables (simple, atomic, avoids stale links).

---

## 5. Frontend Views (8 Tabs)

### 5.1 Capability Map (Default View)

**Purpose:** Visual overview of all capabilities organized by domain.

- Color-coded grid: each domain is a row, each capability is a card
- **Lens selector** colors cards by: Maturity / Pain / Strategic Importance / Priority Score / Annual Spend / Gap to Target / Differentiation
- Summary tiles: Total Spend, High-Importance Gaps, Avg Maturity, Commodity Spend
- "Key Transformation Areas" overlay: top 12 sub-capabilities by Priority Score
- Click any card → opens Detail Panel (slide-out)

### 5.2 Gap Analysis

**Purpose:** Prioritized list of where investment is needed most.

- Ranked list of capabilities sorted by gap distance
- Filters: tier (Differentiating/Core/Commodity), minimum importance threshold
- Sort options: Priority Score / Gap / Spend
- Visual gap bars showing current vs target maturity
- Indicates whether initiatives are already funding the gap

### 5.3 Current vs Target

**Purpose:** Side-by-side maturity comparison.

- Grouped by domain
- Each capability shows current M → target M with visual step indicator
- Quickly identifies largest gaps per domain

### 5.4 Value Matrix (Bubble Chart)

**Purpose:** Strategic positioning quadrant view.

```
        High Importance
             │
  INVEST     │   DEFEND &
  (top-left) │   EXTEND
             │   (top-right)
─────────────┼──────────────── Maturity →
  TOLERATE   │   SUNSET /
  (bot-left) │   OUTSOURCE
             │   (bot-right)
             │
        Low Importance
```

- X-axis: Current Maturity (1→5)
- Y-axis: Strategic Importance (1→5)
- Bubble size: Annual Spend
- Bubble color: Differentiation Tier

### 5.5 Investment Portfolio

**Purpose:** View all initiatives and what they're advancing.

- Cards grouped by horizon: **Now** / **Next** / **Later**
- Each card shows: status, confidence, owner, cost ($M), outcome, linked capabilities
- Helps answer: "What are we investing in and why?"

### 5.6 Applications

**Purpose:** Application landscape mapped to capabilities.

- Table sorted by cost (highest first)
- TIME classification summary (counts + total cost per category)
- Shows which capabilities each application supports
- Enables "tolerate vs retire" rationalization decisions

### 5.7 Owners

**Purpose:** Accountability view by domain VP.

- Grouped by capability owner
- Shows total spend, gap count, differentiation mix, initiative count per owner
- Answers: "Who is responsible for which capabilities and what's their investment posture?"

### 5.8 About

**Purpose:** Self-documenting scoring model explanation.

- Explains M/P/S scales
- Shows target formula
- Describes gap and PS computation
- Explains differentiation tiers and their investment implications

---

## 6. User Experience Features

### 6.1 Personas

Three pre-configured view modes that set default lens, view, and density:

| Persona | Default View | Default Lens | Density |
|---------|-------------|-------------|---------|
| Executive | Capability Map | Priority Score | Comfy |
| Domain VP | Gap Analysis | Gap to Target | Regular |
| Architect | Applications | Maturity | Compact |

### 6.2 Lenses (Color Coding)

| Lens | What it Shows | Color Scale |
|------|---------------|-------------|
| Default | Neutral | Domain colors |
| Maturity | Current performance | Red(1) → Green(5) |
| Pain | Current pain level | Green(1) → Red(5) |
| Strategic Importance | Business criticality | Gray(1) → Purple(5) |
| Priority Score | Investment urgency | Green(low) → Red(high) |
| Annual Spend | Cost allocation | Light(low) → Dark(high) |
| Gap to Target | Distance to goal | Green(0) → Red(4) |
| Differentiation | Investment type | Tier-based colors |

### 6.3 Editor (Beacon Editor Panel)

- Slide-out panel with tab-based entity switching
- Full CRUD for all 5 entity types
- **Live score preview**: as you adjust M/P/S sliders, Target/Gap/PS update in real-time
- Multi-chip selectors for many-to-many relationships
- Validation: prevents save without required fields

### 6.4 Detail Panel

- Slide-out on capability card click
- Shows: all scores, gap visualization, recommendation, sub-capabilities list, supporting apps, active initiatives
- Jump links to other views (e.g., "View in Gap Analysis")

---

## 7. Technical Implementation Summary

### 7.1 Backend

| Aspect | Implementation |
|--------|---------------|
| Framework | NestJS (Controller + Service + Module) |
| Database | PostgreSQL via Prisma ORM |
| Tenant isolation | Prisma middleware auto-injects `tenantId` |
| Auth | JWT guard + page-level RBAC |
| Scoring | Runtime-computed (never stored) |
| API pattern | Single GET for full model, individual CRUD for mutations |

### 7.2 Frontend

| Aspect | Implementation |
|--------|---------------|
| Framework | React (single component ~1500 lines) |
| State | Local state (useState) — model fetched once, re-fetched on mutation |
| Styling | Custom CSS with CSS variables |
| Editor | Separate BeaconEditor component (~500 lines) |
| Scoring | Mirrored from backend for live preview |
| Responsive | Basic — works best on desktop/tablet |

---

## 8. Strengths & Weaknesses

### Strengths

| # | Strength |
|---|----------|
| 1 | **Powerful scoring algorithm** — PS formula effectively surfaces investment priorities |
| 2 | **Single-call API** — Entire model in one fetch, enables rich multi-view UI |
| 3 | **Live score preview** — Users see impact of M/P/S changes before saving |
| 4 | **Multiple views** — Same data, 8 different perspectives for different audiences |
| 5 | **Recommendation engine** — Actionable guidance (invest/fix/outsource/maintain) |
| 6 | **Persona-based UX** — Quick switching between executive/VP/architect viewpoints |
| 7 | **Many-to-many relationships** — Apps and Initiatives properly linked to capabilities |
| 8 | **Self-documenting** — About tab explains the model to new users |

### Weaknesses

| # | Weakness | Impact |
|---|----------|--------|
| 1 | **Monolithic frontend component** (~1500 lines) | Hard to maintain, test, or extend |
| 2 | **No versioning/history** | Can't track how maturity changed over time |
| 3 | **No bulk import/export** | Must enter capabilities one-by-one (tedious for large orgs) |
| 4 | **Single-tenant model load** | May become slow with hundreds of capabilities |
| 5 | **No collaboration features** | No comments, change tracking, or approval workflow |
| 6 | **No offline scoring changes** | Can't simulate "what if maturity improves to 4?" |
| 7 | **Zero tests** | No backend or frontend tests for scoring or CRUD |
| 8 | **Flat permission model** | Can't restrict by domain (e.g., VP only edits their domain) |

---

## 9. Rebuild Recommendations

### 9.1 Keep As-Is (Proven Value)

- ✅ Scoring algorithm (PS = S × P × (5-M))
- ✅ Target maturity formula (auto + manual override)
- ✅ 8-view UI concept (map, gap, compare, matrix, portfolio, apps, owners, about)
- ✅ Persona switching
- ✅ Lens-based color coding
- ✅ Recommendation engine logic
- ✅ Single-API-call model (GET /beacon returns everything)
- ✅ Entity hierarchy (Domain → Capability → SubCapability)
- ✅ TIME classification for applications

### 9.2 Improve in Rebuild

| # | Improvement | Rationale |
|---|-------------|-----------|
| 1 | **Add maturity history tracking** | Track score changes over time (quarterly snapshots) |
| 2 | **Bulk CSV import/export** | Large orgs need to seed from spreadsheets |
| 3 | **What-if simulation** | Allow users to preview score impact without saving |
| 4 | **Domain-level RBAC** | VPs should only edit capabilities in their domain |
| 5 | **Modular Flutter components** | Split monolithic view into small, testable widgets |
| 6 | **Realtime collaboration** | Multiple users editing simultaneously (Supabase Realtime) |
| 7 | **Comments/annotations** | Allow stakeholders to discuss specific capabilities |
| 8 | **Audit trail** | Who changed what and when (for governance) |
| 9 | **Dashboard integration** | Beacon summary cards on the main dashboard |
| 10 | **API pagination** | For orgs with 200+ capabilities, paginate sub-resources |

### 9.3 Schema Enhancements for v2

```sql
-- New: History tracking
CREATE TABLE beacon_capability_snapshots (
  id UUID PRIMARY KEY,
  capability_id UUID REFERENCES beacon_capabilities(id),
  maturity INT,
  pain INT,
  importance INT,
  snapshot_date DATE,
  created_by UUID REFERENCES auth.users(id)
);

-- New: Comments
CREATE TABLE beacon_comments (
  id UUID PRIMARY KEY,
  entity_type TEXT,  -- 'capability' | 'application' | 'initiative'
  entity_id UUID,
  user_id UUID REFERENCES auth.users(id),
  content TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- New: Audit log
CREATE TABLE beacon_audit_log (
  id UUID PRIMARY KEY,
  entity_type TEXT,
  entity_id UUID,
  action TEXT,  -- 'create' | 'update' | 'delete'
  changes JSONB,
  user_id UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 10. Stakeholder Decision Points

Questions for stakeholder discussion:

1. **Keep Beacon in MVP?** It's a sophisticated module — worth the rebuild investment for v2.0?
2. **History tracking priority?** Is quarterly maturity tracking important for your org?
3. **Collaboration features?** Do multiple people need to edit simultaneously?
4. **Integration with Projects?** Should capabilities link to Nexus projects (not just apps/initiatives)?
5. **Import from existing data?** Do you have an existing capability model to migrate?
6. **Domain-level permissions?** Should VPs only see/edit their own domain?
7. **Mobile support?** Is Beacon needed on mobile or desktop-only is fine?
8. **Reporting/exports?** Need PDF/PowerPoint export for board presentations?

---

*This analysis was produced by complete source code review of the Beacon module across both backend and frontend repositories.*
