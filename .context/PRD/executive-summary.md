# Executive Summary — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 2 — Architecture
> Sources: DESIGN.md, app/(auth)/login/page.tsx, app/(app)/*, app/qa/qa-config.ts, supabase/migrations/0001–0012

---

## Problem Statement

Quality engineering teams managing large test suites lack a purpose-built tool that treats test authoring as first-class software engineering. Generic spreadsheets and bolt-on Jira plugins force QA engineers to manage hundreds of Acceptance Test Cases (ATCs) in tools designed for different workflows. Orphaned tests — cases with no traceable link to a business requirement — are structurally impossible to prevent in these tools, leading to test debt and poor coverage mapping.

Existing TMS tools fail on three axes:
1. **No anchoring moat**: test cases can float free of any user story or acceptance criterion.
2. **Not API-first**: CI pipelines and AI agents cannot read or write test artifacts headlessly.
3. **Not agentic-ready**: no purpose-built testability guide, no PAT-based auth layer for non-browser consumers.

Source: DESIGN.md §1–§2, app/(auth)/login/page.tsx (FEATURE_TICKS array), app/qa/qa-config.ts

---

## Solution

Bunkai (分解 — Japanese: "to decompose/analyze") is a developer-first, multi-tenant Test Management System built on Next.js 15 + Supabase. Its core value proposition:

1. **Structural anchoring**: an ATC MUST be bound to at least one Acceptance Criterion via the `atc_acceptance_criteria` M:N table. The DB enforces this — orphan tests are architecturally impossible.
2. **Versioned test cases**: every ATC save increments `atcs.version`, providing an optimistic-lock handle and a change-history signal.
3. **Full-text search**: `atcs.tsv` (GIN index over title + tags) supports fast workspace-wide ATC discovery.
4. **API-first design**: full REST API at `/api/v1` with Bearer PAT auth (`bk_pat_*` format) for CI pipelines and AI agents.
5. **Agentic-ready**: `/qa` page ships Playwright fixture snippets, MCP config blocks, and PAT bootstrap flows. `qa_inspector_ro`/`qa_inspector_rw` DB roles exist for direct data access.
6. **Multi-tenant isolation**: workspace-scoped tenancy backed by Postgres RLS — one SaaS instance, many isolated teams.
7. **IQL methodology**: Integrated Quality Lifecycle — the product promotes an explicit Story → AC → ATC → Run → Bug chain.

Source: DESIGN.md §1–§2, migrations/0004_atcs.sql, migrations/0011_split_token_secrets.sql, app/qa/qa-config.ts

---

## Success Metrics

> Note: No formal KPI definitions were found in the codebase. The following are inferred from the product design intent.

| Metric | Inference Basis |
|--------|----------------|
| % ATCs with ≥ 1 bound AC | "Anchoring moat" is the primary structural guarantee; measure adherence |
| API request volume (PAT-authenticated) | Reflects CI/agent adoption — first-class consumers |
| Active workspaces per plan tier | Revenue signal; community → cloud → enterprise conversion |
| ATC authoring sessions / day | Core engagement metric for QA engineers |
| Magic-link auth success rate | Auth funnel health |

Source: DESIGN.md §2, migrations/0001_tenancy.sql (plan constraint), app/qa/qa-config.ts

---

## Scope (in / out)

### In Scope (implemented in migrations 0001–0012 + current routes)

- Multi-tenant workspace management with RBAC (viewer/member/admin/owner)
- Project + module tree (self-referential, max depth 6)
- User story authoring with optional Jira external link
- Acceptance criteria CRUD under user stories
- ATC authoring with steps, assertions, and AC bindings (Monaco editor)
- PAT issuance, listing, and revocation
- Workspace invite flow (admin issues, invitee redeems via token)
- REST API v1 with OpenAPI spec at `GET /api/openapi` + Scalar docs at `GET /api/docs`
- Magic-link authentication + headless email+password auth for QA bots
- Activity log (append-only audit trail)
- Feature flags (global + workspace-scoped)
- In-app testability guide (`/qa`) with Playwright fixtures and MCP config

### Out of Scope (confirmed absent from current codebase)

- Test run execution engine — `atcs.status` tracks last known result, but no `runs` table exists yet (planned Sprint 2)
- Automated email sending for invites — MVP logs the accept URL to stdout only
- OAuth (GitHub/Google) — UI shows disabled buttons with "soon" label
- SSO — mentioned as future in login page
- Light theme — DESIGN.md §2 marks light mode as Phase 2
- Project creation UI — current placeholder says "ships in Phase E"
- Multi-workspace switcher UI — partial (WorkspaceSwitcher component exists, full picker Phase E)
- ATC slug auto-generation logic — field exists in DB, generation not found in migrations/lib

Source: app/(app)/projects/page.tsx, app/(auth)/login/page.tsx, DESIGN.md §12, migrations/0001–0012

---

## Key Stakeholders

| Stakeholder | Role in Product |
|-------------|----------------|
| QA Engineer | Primary daily user — creates/edits ATCs in the tree editor, links to AC |
| SDET / Automation Engineer | Headless CI consumer — mints PATs, queries ATCs via API, posts run results |
| Engineering Manager / Tech Lead | Workspace admin — invites team, manages membership, reviews activity log |
| AI Agent (Claude Code, OpenCode) | Agentic consumer — browser via Playwright MCP or REST API via Bearer PAT |
| Product Owner / Team Lead | Reads test coverage reports, reviews ATC-to-story traceability |

Source: DESIGN.md §1, app/qa/qa-config.ts (playwright + api sections)

---

## Discovery Gaps

- [ ] Formal success KPIs — no explicit metric definitions in codebase; all metrics above are inferred.
- [ ] Pricing/feature matrix per plan — `community`/`cloud`/`enterprise` plans exist in DB but feature gating logic is not in any migration or route file.
- [ ] Staging and production URLs — not in `.env.example`; must be retrieved from Vercel project settings.
- [ ] ATC run/execution model — `atcs.status` persists last known result, but no `runs`, `run_results`, or `test_execution` table exists in migrations 0001–0012 (noted as Sprint 2 in `qa-config.ts`).
- [ ] Jira integration depth — `user_stories.external_id`/`external_url` fields suggest Jira link, but no sync script or webhook is implemented.
