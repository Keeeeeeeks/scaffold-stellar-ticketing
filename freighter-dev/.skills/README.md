# Freighter Skills

Reference documentation for the Freighter wallet ecosystem. Load these as context when working on features, PRDs, or technical decisions.

## Skills Index

| Skill | File | Load When... |
|-------|------|-------------|
| **System Architecture** | `freighter-system-architecture.md` | Planning any feature, scoping work, understanding how the 6 codebases connect |
| **Data Models** | `freighter-data-models.md` | Designing data, writing migrations, understanding balance/token/transaction schemas |
| **API Reference** | `freighter-api-reference.md` | Designing endpoints, understanding client→backend routing, planning v1→v2 migration |
| **PRD Technical Addendum** | `prd-technical-addendum.md` | Writing or reviewing a PRD — includes reusable template + Discover Tab v2 worked example |

## Quick Reference

**6 codebases**: wallet-backend (Go/GraphQL/PG), freighter-backend-v1 (Node/Fastify), freighter-backend-v2 (Go/net-http), freighter-ext (TS/React/Chrome), freighter-mobile (TS/React Native)

**Data flows**: Clients → v1 or v2 backend → wallet-backend (via wbclient+JWT) → PostgreSQL (14 tables) + Stellar RPC/Horizon

**State management**: Extension uses Redux Toolkit; Mobile uses Zustand (23 stores)

**Feature flags**: Served by `GET /api/v1/feature-flags` from freighter-backend-v2, evaluated per-platform and per-version

## Freshness

Generated: 2026-02-10 from direct codebase analysis of all 4 local repos + cloned wallet-backend. Update these files when significant architectural changes ship.
