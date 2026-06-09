# Business Model — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 1 — Constitution
> Sources: DESIGN.md, lib/types.ts, lib/api/pat.ts, app/qa/qa-config.ts, middleware.ts, supabase/migrations/0001–0012

---

## Problem Statement

Quality engineering teams that manage large test suites lack a purpose-built tool that treats test authoring as first-class software engineering. Generic spreadsheets or bolt-on Jira plugins force QAs to manage hundreds of Acceptance Test Cases (ATCs) in tools designed for different workflows. Bunkai (分解 — Japanese: "to decompose/analyze") is a developer-first Test Management System that structures ATCs with traceable links to User Stories and Acceptance Criteria, provides an in-browser editor with Monaco, and exposes a REST API + PAT auth layer so AI agents and CI pipelines can read and write test artifacts headlessly.

Source: DESIGN.md §1, app/qa/qa-config.ts

---

## Target Users

| User | Role | Primary Workflow |
|------|------|-----------------|
| QA Engineer | Daily authoring | Create/edit ATCs in the tree editor, link to AC, run execution |
| SDET / Automation Engineer | Headless CI | Mint PATs, query ATCs via API, post run results |
| Tech Lead / Engineering Manager | Oversight | Invite teammates, manage workspace, view activity log |
| AI Agent (Claude, OpenCode) | Agentic QA | Browser via Playwright MCP or REST API via Bearer PAT |

Source: DESIGN.md §1–§2, app/qa/qa-config.ts §playwright + §api

---

## Value Proposition

1. **Anchoring moat**: an ATC MUST be linked to at least one Acceptance Criterion — orphan tests are structurally impossible.
2. **Information density**: designed for QAs managing hundreds of ATCs daily; 13px base font, 4-px grid, no decorative whitespace.
3. **API-first**: full REST API with Bearer PAT auth means CI pipelines and AI agents are first-class consumers, not afterthoughts.
4. **Multi-tenant isolation**: workspace-scoped tenancy backed by Postgres RLS — one SaaS instance serves many teams without data leakage.
5. **Agentic-ready**: `/qa` Testability Guide page shipped in-app with Playwright fixture snippets, MCP config blocks, and PAT bootstrap flows.

Source: DESIGN.md §2, migrations/0003_authoring.sql, migrations/0004_atcs.sql, app/qa/qa-config.ts

---

## Business Model Canvas (summary)

| Block | Details |
|-------|---------|
| **Customer Segments** | QA/SDET teams at product companies; AI-augmented QA teams using agentic workflows; teams running Playwright-based automation |
| **Value Propositions** | Traceable ATC authoring; structured AC anchoring; API-first test management; multi-tenant SaaS with RBAC; agentic-ready testability guide |
| **Channels** | GitHub (open-source / community tier); direct SaaS deployment (cloud/enterprise); CLI tooling (`bk_pat_*` tokens) |
| **Revenue Streams** | Community (free, self-hosted/limited); Cloud (managed SaaS); Enterprise (enhanced RBAC, SSO, SLAs) — plans encoded in DB |
| **Key Resources** | Supabase Postgres (multi-tenant data store); Next.js 15 App Router (FE + API); Monaco editor integration; OpenAPI spec |
| **Key Activities** | ATC authoring and versioning; workspace/team management; test run execution; API access token management; AI agent integration |
| **Key Partnerships** | Supabase (backend-as-a-service); Vercel (hosting); Atlassian Jira (upstream issue tracker via `external_id`/`external_url` fields) |
| **Cost Structure** | Supabase usage (DB + auth + storage); Vercel compute; development tooling |

Source: migrations/0001_tenancy.sql (plan check constraint), DESIGN.md §1, app/qa/qa-config.ts

---

## Plans / Pricing Tiers

| Plan | Code Value | Notes |
|------|------------|-------|
| `community` | `'community'` | Default tier for new workspaces (`bunkai_bootstrap_workspace` inserts with `'community'`). Likely self-hosted or free SaaS tier. |
| `cloud` | `'cloud'` | Managed SaaS — specific feature gating not yet in migrations (Phase 2 via `feature_flags`). |
| `enterprise` | `'enterprise'` | Enhanced RBAC, SSO, audit logs — specific capabilities not yet implemented in current migrations. |

Source: migrations/0001_tenancy.sql line 33 (CHECK constraint), migrations/0006_bootstrap_workspace.sql line 45

---

## Testing Maturity Assessment

| Dimension | Finding | Verdict |
|-----------|---------|---------|
| Test framework | No Playwright, Vitest, Jest, or Cypress in `package.json` (no `devDependencies` test runner) | **ABSENT** |
| CI/CD | No `.github/workflows/` directory | **ABSENT** |
| Test config | No `playwright.config.ts`, `vitest.config.ts`, `jest.config.*` in root | **ABSENT** |
| In-app QA guide | `/qa` route with `qa-config.ts` ships Playwright fixture snippets and MCP config | Present as **documentation/guide only** |
| DB test roles | `qa_inspector_ro` and `qa_inspector_rw` roles mentioned in migration 0011 | **Schema-ready** for QA data access |
| OpenAPI spec | `/api/openapi` endpoint + Scalar UI at `/api/docs` | **API contract exists** — no automation yet consuming it |

**Verdict**: The app is **zero-maturity** from an automated test perspective — no test runner, no CI. It is, however, **testability-mature**: QA roles, OpenAPI spec, PAT auth, and `/qa` guidance page are all in place. The QA repo (`bunkai-qa-engineering`) is the designated home for all automated tests.

Source: package.json (no test deps), ls output (no .github), app/qa/qa-config.ts

---

## Discovery Gaps

- [ ] Pricing/feature matrix per plan (`community` vs `cloud` vs `enterprise`) — plans exist in DB but feature gating logic is not in migrations 0001–0012.
- [ ] Staging/production URL — not in `.env.example` (only `http://localhost:3000` for `NEXT_PUBLIC_APP_URL`); must be obtained from Vercel project or team.
- [ ] ATC run / execution model — `atcs.status` tracks last known status but no `runs` or `run_results` table exists yet (noted as "Sprint 2" in `qa-config.ts`).
- [ ] Jira integration depth — `user_stories.external_id` / `external_url` fields suggest Jira link, but no sync script or webhook is implemented in current migrations.
- [ ] `qa_inspector_ro` / `qa_inspector_rw` DB role creation SQL — referenced in migration 0011 but the `CREATE ROLE` DDL is not in any migration file (likely provisioned out-of-band in Supabase).
