# Non-Functional Specs — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 2 — Architecture
> Sources: DESIGN.md §8, lib/api/handler.ts, lib/api/error-envelope.ts, middleware.ts, migrations/0001–0012, app/qa/qa-config.ts, package.json

---

## Performance

### Inferred from Design Intent

| Concern | Spec | Source |
|---------|------|--------|
| Page navigation | Must feel "instantaneous" — no fade transitions > 80ms on route changes | DESIGN.md §8 "No page transitions" |
| Route-level loading | Next.js App Router SSR — data fetched server-side before first render | app/(app)/projects/[projectSlug]/page.tsx (parallel Promise.all) |
| ATC search | Full-text search via GIN-indexed `tsvector` column on `atcs.tsv` | migrations/0004_atcs.sql |
| PAT lookup | O(1) by `token_prefix` (indexed) + constant-time SHA-256 compare — no full-table scan | lib/api/pat.ts, lib/api/middleware/bearer.ts |
| Data load for project view | Modules, stories, ATCs, ACs loaded in parallel (Promise.all) on project page render | app/(app)/projects/[projectSlug]/page.tsx |
| Atomic ATC saves | `bunkai_save_atc` RPC = full-replace in a single DB transaction — no partial saves | migrations/0007_save_atc.sql |
| Status animation | "Running" dot pulse: 1.6s ease-in-out (subtle) | DESIGN.md §8 |

### Discovery Gaps
- No explicit SLA, p95 latency target, or Vercel Edge config found in codebase.
- Database connection pooling: Session Pooler port 5432 used (not Transaction Pooler port 6543 — comment in config says "no prepared statements").
- No caching layer (Redis, Vercel KV) configured in current routes or environment variables.

Source: DESIGN.md §8, app/(app)/projects/[projectSlug]/page.tsx, lib/api/middleware/bearer.ts

---

## Security

### Authentication

| Mechanism | Implementation | Notes |
|-----------|---------------|-------|
| Browser session | Supabase SSR cookie (`sb-<project-ref>-auth-token`) | `createServerClient` validates on every request via middleware |
| Headless Bearer PAT | `bk_pat_<prefix>.<secret>` — SHA-256 stored, constant-time compare | 256-bit entropy; family prefix enables secret-scanning |
| Magic-link OTP | Supabase `signInWithOtp` — one-time token | Best-effort audit in `magic_link_tokens` |

### Authorization

| Layer | Mechanism |
|-------|----------|
| Route-level | `middleware.ts` redirects unauthenticated requests to `/login` for protected paths |
| API-level | `requireAuth()` → `requireBearerToken()` or Supabase session validation |
| Scope-level | Bearer PAT scopes: `atc:read`, `atc:write`, `run:execute`, `workspace:admin`. Cookie sessions are unscoped (RLS is the constraint). |
| DB-level | Postgres RLS on all tables — `bunkai_is_workspace_member`, `bunkai_can_write_workspace`, `bunkai_is_workspace_admin`, `bunkai_is_workspace_owner` SECURITY DEFINER helpers |

### Secret Material Handling

- PAT secrets, invite tokens, and magic-link token hashes live in isolated sibling tables (`access_token_secrets`, `workspace_invite_secrets`, `magic_link_token_secrets`).
- `qa_inspector_ro` and `qa_inspector_rw` DB roles explicitly revoked from reading sibling secret tables.
- Admin client (`service_role`) used only after app-layer auth is verified.
- Raw tokens shown exactly once in POST responses. No retrieval endpoint.

### Input Validation

- All API route bodies validated with Zod. Zod failures → `validation_failed` (422) with issue details.
- Email: max 254 chars (RFC 5321 §4.5.3.1.3 pragmatic ceiling).
- Workspace slug: `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$` + reserved-slug check.
- Open-redirect guard: `next` parameter in magic-link must be root-relative (not `//` prefix).
- Idempotency key table: 24h TTL on `(user_id, endpoint, key)` — replay protection for POST endpoints.

### Secrets in Code

- Credentials read from env vars only (`.env`). No hardcoded values found in routes or lib files.
- `.env.example` contains only placeholder values.

Source: lib/api/handler.ts, lib/api/middleware/bearer.ts, lib/api/auth.ts, middleware.ts, migrations/0005_rls_helpers.sql, migrations/0008–0011

---

## Reliability (RTO/RPO)

### What is Confirmed in Code

| Concern | Finding |
|---------|---------|
| Atomic operations | `bunkai_bootstrap_workspace` and `bunkai_save_atc` are single-transaction RPCs — no partial state if request fails mid-way |
| Soft-delete semantics | PAT revocation, invite revocation use `revoked_at` timestamps — no physical DELETE. Audit trail preserved. |
| Idempotency | `idempotency_keys` table (24h TTL) — `pending → succeeded | failed` state machine. POST replay returns cached response. |
| Magic-link issuance | Best-effort audit; failure in `recordIssuance` is swallowed — does not break the auth flow |
| Request tracing | `x-request-id` on every response — users can quote the ID in bug reports |

### Discovery Gaps

- No explicit RTO/RPO targets defined in codebase.
- No database backup schedule or Supabase PITR configuration found.
- No retry policy for Supabase upstream errors (current: propagate `upstream_error` 502 to caller).
- No health-check monitoring interval defined — `GET /api/v1/health` exists as a liveness probe but no external checker configuration found.

Source: migrations/0007_save_atc.sql, migrations/0009_cross_cutting.sql, app/api/v1/health/route.ts

---

## Scalability

### Architecture Constraints

| Concern | Current State |
|---------|--------------|
| Multi-tenancy | Workspace-scoped RLS — all tenants share a single Postgres instance. Isolation is logical (policy), not physical. |
| Serverless compute | Vercel serverless — stateless, scales horizontally per request |
| Connection pooling | Session Pooler (port 5432) used. Transaction Pooler (6543) explicitly excluded (no prepared statements). |
| Module tree depth | Hard limit: max 6 path segments enforced in DB schema. |
| ATC full-text index | GIN index on `tsv` — scales well for read-heavy full-text queries |

### Discovery Gaps

- No load test results or capacity planning data in codebase.
- Connection pool size not configured (default Supabase Session Pooler limits apply).
- No horizontal sharding or workspace-level DB isolation for enterprise tier — this would need explicit implementation if enterprise tier demands physical isolation.
- Feature flag system exists for gradual rollout but no flags are actively read yet.

Source: app/qa/qa-config.ts (dbhub pooler config), migrations/0002_projects_modules.sql (depth constraint)

---

## Observability

### Currently Implemented

| Signal | Mechanism |
|--------|----------|
| Access logs | Structured JSON stdout via `logRequest()` in `lib/api/logging.ts` — includes `request_id`, method, path, status, duration_ms, error_code |
| Request tracing | `x-request-id` header injected on every API response |
| Magic-link audit | `magic_link_tokens` table — email, user_agent per issuance |
| Activity log | `activity_log` table — append-only workspace audit trail |
| Invite issuance | `console.log` with accept_url (MVP — no transactional email yet) |
| Liveness probe | `GET /api/v1/health` — returns 200 |

### Discovery Gaps

- No distributed tracing (OpenTelemetry, Datadog, Sentry) found in package.json or routes.
- No metrics collection (Prometheus, Vercel Analytics) configured.
- No alerting rules or on-call rotation defined.
- No readiness probe or deep-health endpoint (database connectivity check).
- Activity log has no API read endpoint — observable only via `qa_inspector_ro` DB role directly.

Source: lib/api/logging.ts, lib/api/handler.ts, lib/api/request-id.ts, app/api/v1/health/route.ts, migrations/0009_cross_cutting.sql

---

## Compliance

### Currently Implemented

| Requirement | Implementation |
|-------------|---------------|
| Token secrecy | SHA-256 stored only; raw token shown once with explicit `warning` field |
| Email privacy | `magic_link_token_secrets.ip_hash` hashes the IP before storage |
| Audit trail | Append-only `activity_log` per workspace |
| Soft-delete tokens | Revoked tokens retain their row for compliance audit |
| Open-source license | Apache-2.0 (stated in login page footer and `app/(auth)/login/page.tsx`) |

### Discovery Gaps

- No GDPR/CCPA data deletion workflow found — `revoked_at` soft-deletes preserve data indefinitely.
- No data retention policy documented.
- No GDPR "right to be forgotten" mechanism for `activity_log` or `magic_link_tokens`.
- No SOC 2 / ISO 27001 controls documentation found.
- Enterprise plan features (SSO, audit exports) are mentioned as future but no implementation exists.

Source: app/(auth)/login/page.tsx, migrations/0011_split_token_secrets.sql, migrations/0009_cross_cutting.sql

---

## Accessibility

### Confirmed from Design System

| Standard | Spec | Source |
|----------|------|--------|
| Contrast | `--fg-1` on `--bg-0` ≈ 12.5:1; `--fg-2` on `--bg-0` ≈ 7:1; accent `#d9543f` ≈ 5.1:1 | DESIGN.md §10 — meets WCAG AA |
| Focus | Every interactive element: 1px solid `--accent` outline with 1px offset on `:focus-visible`. Never remove without replacement. | DESIGN.md §10 |
| Keyboard | Every primary action has a keyboard path via command palette | DESIGN.md §10 |
| Focus traps | Modal/dialog focus traps required | DESIGN.md §10 |
| Color independence | Status dots paired with text and icons — color is never the sole signal | DESIGN.md §10 |
| Reduced motion | `prefers-reduced-motion` respected — disables caret blink and status dot pulse | DESIGN.md §8, §10 |

### Discovery Gaps

- Accessibility not verified against actual rendered components — DESIGN.md specifies intent, not verified implementation.
- No automated accessibility test suite (e.g., axe-core) found in current codebase.

Source: DESIGN.md §10

---

## Browser / Platform Support

### Confirmed

- Default theme: **dark mode** — light mode is Phase 2 (DESIGN.md §2).
- No explicit browser compatibility matrix found in codebase.
- The app uses Next.js 15 + React 19 — requires modern browsers with ES2022 support.
- Responsive breakpoints: `lg` (1024px) used in login page grid layout. Mobile-specific treatment not confirmed.

### Discovery Gaps

- No `browserslist` config found in `package.json` or `.browserslistrc`.
- Mobile layout not verified — login page has an `lg:grid-cols-[1fr_460px]` split suggesting a single-column mobile view.

Source: app/(auth)/login/page.tsx, DESIGN.md §2, package.json
