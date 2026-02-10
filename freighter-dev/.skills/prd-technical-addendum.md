# PRD Technical Addendum Guide

> **Skill**: Load this when writing or reviewing a PRD for any Freighter feature. Use the template to draft a Technical Addendum that eliminates engineering clarification cycles.
>
> **When to use**: After a PRD's product requirements are defined, before engineering review. Attach as a companion document or append to the PRD.

---

## Why This Exists

The gap between a product PRD and an engineering-ready spec is where review cycles live. A Technical Addendum bridges that gap by answering the questions engineers will ask:

- Which repos are affected?
- What data needs to exist? Where?
- What API endpoints are needed?
- What changes on the client?
- How do we roll this out safely?
- What decisions need eng input?

## Template

Copy the template below. Fill in every section. Delete nothing — mark sections `N/A` if truly not applicable.

---

```markdown
# Technical Addendum: [Feature Name]

_Companion to: [Link to PRD]_
_Last updated: [Date]_
_Author: [Name]_

## 1. Affected Systems

| System | Impact Level | Changes Required |
|--------|-------------|-----------------|
| wallet-backend | None / Read / Write / New Tables / New Processor | |
| freighter-backend v1 | None / Modified / New Endpoint / Deprecated | |
| freighter-backend v2 | None / Modified / New Endpoint | |
| freighter-extension | None / New View / Modified State / New Component | |
| freighter-mobile | None / New Screen / Modified State / New Component | |
| Infrastructure | None / New Service / Config Change / New DB | |

## 2. Data Model Impact

### New Database Tables
<!-- Provide CREATE TABLE SQL or describe the schema -->

### Modified Tables
| Table | Column | Change Type | Migration Notes |
|-------|--------|------------|----------------|

### New Client Types
<!-- TypeScript interfaces or Go structs that need to exist -->

## 3. API Contract

### New Endpoints
| Method | Path | Request Shape | Response Shape | Auth | Backend |
|--------|------|--------------|----------------|------|---------|

### Modified Endpoints
| Method | Path | What Changes | Breaking? |
|--------|------|-------------|-----------|

### Example Payloads
<!-- One request/response example per new endpoint -->

## 4. Client State Changes

### Extension (Redux Toolkit)
| Slice (duck) | New State Fields | New Actions | New Selectors |
|-------------|-----------------|-------------|---------------|

### Mobile (Zustand)
| Store | New State Fields | New Actions |
|-------|-----------------|-------------|

### Persistence
| Key | Storage Mechanism | Platform | Purpose |
|-----|------------------|----------|---------|
<!-- chrome.storage.local, AsyncStorage, SecureStorage, Redis -->

## 5. Feature Flag Strategy

| Flag Name | Where Evaluated | Default | Rollback Behavior |
|-----------|----------------|---------|-------------------|
<!-- Feature flags live in freighter-backend-v2 FeatureFlagsHandler -->

## 6. Integration Dependencies

| Dependency | Type | Current Status | Owner | Blockers |
|-----------|------|---------------|-------|----------|
<!-- External services, prerequisite PRDs, infra requirements -->

## 7. Extension Points Used

Which existing architectural hooks does this feature use?

- [ ] wallet-backend ingestion processor (new or modified)
- [ ] wallet-backend GraphQL schema (new types/queries)
- [ ] wallet-backend database table (new or modified)
- [ ] freighter-backend-v2 handler (new endpoint)
- [ ] freighter-backend-v2 feature flag (new flag)
- [ ] Extension Redux slice (new or modified duck)
- [ ] Extension view (new or modified in popup/views/)
- [ ] Extension @shared types (new types)
- [ ] Mobile Zustand store (new or modified duck)
- [ ] Mobile screen (new or modified in components/screens/)
- [ ] Mobile service (new or modified in services/)

## 8. Rollout Plan

| Phase | Scope | Feature Flag | Rollback |
|-------|-------|-------------|----------|

## 9. Open Questions for Engineering

<!-- Numbered. Specific. Binary or multiple-choice where possible. -->
<!-- Bad: "How should we handle caching?" -->
<!-- Good: "Should bookmark data live in (a) chrome.storage.local, (b) a new backend table, or (c) synced via the user's Stellar account?" -->
```

---

## Worked Example: Discover Tab v2

Below is a filled-in Technical Addendum for the Discover Tab v2 PRD.

---

# Technical Addendum: Discover Tab v2

_Companion to: Discover Tab v2 PRD_

## 1. Affected Systems

| System | Impact Level | Changes Required |
|--------|-------------|-----------------|
| wallet-backend | None (Phase 1). Future: tx count queries for rankings | — |
| freighter-backend v1 | None | — |
| freighter-backend v2 | New Endpoints + New DB | dApp CRUD, rankings, click analytics |
| freighter-extension | New View + New State | Discover v2 UI, bookmarks, dApp details page |
| freighter-mobile | Modified Screens + New State | Enhanced DiscoveryHomepage, bookmarks store |
| Infrastructure | New DB for FBv2 | PostgreSQL (FBv2 currently has Redis only) |

## 2. Data Model Impact

### New Database Tables (Freighter Backend v2)

**Note**: FBv2 is currently stateless (Redis only). This feature requires adding PostgreSQL, or routing dApp data through wallet-backend. This is the highest-risk architectural decision.

```sql
CREATE TABLE dapps (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name          TEXT NOT NULL,
    description   TEXT,
    icon_url      TEXT,
    website_url   TEXT NOT NULL,
    contract_address TEXT,
    chains        TEXT[] DEFAULT '{"stellar"}',
    category      TEXT,
    is_audited    BOOLEAN DEFAULT false,
    is_blacklisted BOOLEAN DEFAULT false,
    is_wc_not_supported BOOLEAN DEFAULT false,
    whitepaper_url TEXT,
    sort_order    INTEGER DEFAULT 0,
    created_at    TIMESTAMPTZ DEFAULT NOW(),
    updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE dapp_tags (
    dapp_id UUID REFERENCES dapps(id) ON DELETE CASCADE,
    tag     TEXT NOT NULL,
    PRIMARY KEY (dapp_id, tag)
);

CREATE TABLE dapp_clicks (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dapp_id    UUID REFERENCES dapps(id),
    user_hash  TEXT NOT NULL,   -- privacy-preserving device hash
    clicked_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_dapp_clicks_dapp_id ON dapp_clicks(dapp_id);
CREATE INDEX idx_dapp_clicks_clicked_at ON dapp_clicks(clicked_at);
```

### New Client Types

```typescript
interface DappDetail {
    id: string;
    name: string;
    description: string;
    iconUrl: string;
    websiteUrl: string;
    contractAddress?: string;
    chains: string[];
    category: string;
    isAudited: boolean;
    tags: string[];
    whitepaperUrl?: string;
    metrics?: { txCount: number; activeUsers: number; ranking: number };
}

interface DappBookmark {
    dappId: string;
    url: string;
    savedAt: number;
}
```

## 3. API Contract

### New Endpoints (Freighter Backend v2)

| Method | Path | Request | Response | Auth |
|--------|------|---------|----------|------|
| GET | `/api/v1/dapps` | `?category=&audited=` | `{data: {dapps: DappDetail[]}}` | None |
| GET | `/api/v1/dapps/:id` | — | `{data: DappDetail}` | None |
| GET | `/api/v1/dapps/rankings` | `?period=1w&limit=10` | `{data: {rankings: DappDetail[]}}` | None |
| GET | `/api/v1/dapps/popular` | `?limit=10` | `{data: {dapps: DappDetail[]}}` | None |
| POST | `/api/v1/dapps/click` | `{dapp_id, user_hash}` | `{success: true}` | None |
| POST | `/api/v1/admin/dapps` | `DappDetail` | `{data: DappDetail}` | Admin |
| PUT | `/api/v1/admin/dapps/:id` | `Partial<DappDetail>` | `{data: DappDetail}` | Admin |
| DELETE | `/api/v1/admin/dapps/:id` | — | `{success: true}` | Admin |

### Modified Endpoints

| Method | Path | Change | Breaking? |
|--------|------|--------|-----------|
| GET | `/api/v1/protocols` | Soft-deprecated; keep working, but clients migrate to `/dapps` | No |
| GET | `/api/v1/feature-flags` | Add `discover_v2_enabled` field to `FeatureFlagsResponse` | No |

### Example: GET /api/v1/dapps

```json
{
    "data": {
        "dapps": [
            {
                "id": "550e8400-e29b-41d4-a716-446655440000",
                "name": "Blend",
                "description": "Decentralized lending protocol on Stellar",
                "iconUrl": "https://blend.capital/icon.png",
                "websiteUrl": "https://blend.capital",
                "contractAddress": "CBLEN...",
                "chains": ["stellar"],
                "category": "DeFi",
                "isAudited": true,
                "tags": ["DeFi", "Lending"],
                "whitepaperUrl": "https://blend.capital/whitepaper",
                "metrics": { "txCount": 145230, "activeUsers": 3200, "ranking": 1 }
            }
        ]
    }
}
```

## 4. Client State Changes

### Extension (new Redux slice)

| Slice | New State | New Actions | New Selectors |
|-------|-----------|-------------|---------------|
| `discover` | `{dapps, rankings, bookmarks, recentDapps, isLoading, selectedCategory}` | `fetchDapps, fetchRankings, addBookmark, removeBookmark, trackClick` | `selectFilteredDapps, selectBookmarkedDapps, selectRecentDapps` |

### Mobile (modified + new Zustand stores)

| Store | New State | New Actions |
|-------|-----------|-------------|
| `protocols` (existing) | Add `rankings`, `categories` | `fetchRankings()`, `fetchByCategory()` |
| `bookmarks` (NEW) | `{bookmarks: DappBookmark[]}` | `add()`, `remove()`, `getAll()` |

### Persistence

| Key | Storage | Platform | Purpose |
|-----|---------|----------|---------|
| `discover_bookmarks` | chrome.storage.local | Extension | Saved dApps |
| `discover_recent` | chrome.storage.local | Extension | Recently visited |
| `@freighter/bookmarks` | AsyncStorage | Mobile | Saved dApps |
| (Recent already tracked in `browserTabs` store) | Zustand | Mobile | — |

## 5. Feature Flag Strategy

| Flag | Where | Default | Rollback |
|------|-------|---------|----------|
| `discover_v2_enabled` | FBv2 `/feature-flags` | `false` | Shows current static protocol list |
| `discover_rankings_enabled` | FBv2 `/feature-flags` | `false` | Hides rankings section |

## 6. Integration Dependencies

| Dependency | Type | Status | Owner | Blockers |
|-----------|------|--------|-------|----------|
| PostgreSQL for FBv2 | New infra | Not started | Ops | FBv2 is currently stateless |
| Banners PRD | Prerequisite feature | In progress | Product | Carousel needs Banners |
| On-chain metrics source | Design decision | Open | Eng | Horizon? WB processor? Third-party? |
| Admin auth | New system | Not started | Eng | JWT, SSO, or shared secret? |

## 7. Extension Points Used

- [x] freighter-backend-v2 handler (new `/dapps` endpoints)
- [x] freighter-backend-v2 feature flag (new `discover_v2_enabled`)
- [x] Extension view (rebuild `popup/views/Discover/`)
- [x] Extension @shared types (new `DappDetail`, `DappBookmark`)
- [x] Mobile Zustand store (modify `protocols`, new `bookmarks`)
- [x] Mobile screen (modify `DiscoveryHomepage` component)
- [x] Mobile service (new `fetchDapps()`, `fetchRankings()` in `backend.ts`)
- [ ] wallet-backend ingestion processor (future P2: on-chain metrics)
- [ ] wallet-backend GraphQL schema (future P2)

## 8. Rollout Plan

| Phase | Scope | Flag | Rollback |
|-------|-------|------|----------|
| P0a | Backend: dApp CRUD + admin endpoints | `discover_admin_enabled` | Disable flag |
| P0b | Backend: Banners (separate PRD) | — | — |
| P0c | Clients: Discover v2 with static dApp list | `discover_v2_enabled=false` | Flag → v1 list |
| P1 | Backend: "Popular on Freighter" analytics | Gated by `discover_v2_enabled` | Section hidden if no data |
| P2 | Backend: On-chain rankings + categories | `discover_rankings_enabled` | Section hidden |

## 9. Open Questions for Engineering

1. **Where does dApp data live?** (a) New PostgreSQL in FBv2, (b) wallet-backend's existing PostgreSQL, or (c) external CMS? FBv2 is currently stateless — adding PG is a meaningful infra change.
2. **Rankings data source?** (a) Query Horizon per-dApp contract, (b) new wallet-backend ingestion processor, or (c) third-party (StellarExpert)? Each has different latency/freshness tradeoffs.
3. **Admin auth model?** (a) Internal SSO, (b) Stellar keypair signing (like WB's JWT), or (c) shared API key?
4. **Bookmark sync?** (a) Local-only (both platforms), or (b) server-synced (requires user identity, which doesn't exist today)?
5. **Click tracking privacy?** Is a hashed device ID sufficient, or do we need a formal privacy review before shipping "Popular on Freighter"?
