# Infrastructure — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 3 — Infrastructure
> Sources: lib/urls.ts, .env.example, next.config.ts, .github/ (absent), scripts/**, supabase/migrations/**, lib/api/logging.ts, app/qa/qa-config.ts

---

## Environments

| Name | URL | Detection | Notes |
|------|-----|-----------|-------|
| Local | `http://localhost:3000` | `VERCEL_ENV` absent | `bun run dev` |
| Staging | `https://staging-upexbunkai.vercel.app` | `VERCEL_ENV === 'preview'` | Vercel preview deployments |
| Production | `https://upexbunkai.vercel.app` | `VERCEL_ENV === 'production'` | Vercel production deployment |

Environment detection via `lib/urls.ts` (`getEnvironment()`): reads `VERCEL_ENV` system env var injected automatically by Vercel — no manual configuration needed in deployed environments.

**Active workspace default**: `staging` per `.agents/project.yaml` `testing.default_env`.

Source: `lib/urls.ts` (`APP_URLS` constant), `.env.example`

---

## CI/CD

**No `.github/workflows/` directory found** — the repository does not have GitHub Actions configured.

| Aspect | Status |
|--------|--------|
| GitHub Actions | Not configured |
| Automated test runs on PR | Not set up |
| Automated deploy trigger | Vercel Git integration (auto-detected: every push to main → production, every PR branch → preview URL) |
| Branch protection rules | Unknown — not discoverable from repo files |

> **Discovery Gap**: CI pipeline is absent. Recommend setting up a GitHub Actions workflow for type-check + lint on every PR before the regression-testing phase.

Source: Filesystem scan — no `.github/` directory found

---

## Deployment

- **Platform**: Vercel (serverless Next.js)
- **Build command**: `next build` (standard Next.js; `bun run build`)
- **Deploy triggers**: Vercel Git integration — push to `main` → production; PR branches → preview deployments
- **Config file**: No `vercel.json` found in repo root — using Vercel defaults for Next.js
- **Environment variables**: Injected via Vercel project settings (Vercel Storage → Supabase integration auto-generates `NEXT_PUBLIC_SUPABASE_URL`; others set manually)
- **Edge runtime**: Not used — all API routes use Node-compatible runtime (Bun/Next.js serverless functions)
- **Output tracing root**: Set in `next.config.ts` via `outputFileTracingRoot`

### Vercel + Supabase integration note

The Vercel Supabase integration auto-injects `NEXT_PUBLIC_SUPABASE_URL` into deployed environments. All other Supabase vars must be set manually in Vercel project settings. Copy snippet available from Vercel project → Storage → Supabase store → Quickstart.

Source: `.env.example` comments, `lib/urls.ts`, `next.config.ts`

---

## Database Operations

```bash
# Generate TypeScript types from live Supabase schema
bun run types:gen
# → Outputs: lib/types/supabase.ts

# Run migrations (Supabase CLI — no bun alias)
supabase db push
# or
supabase migration up

# Reset local DB
supabase db reset

# Connect to local Supabase
supabase start       # starts local Supabase stack
supabase stop

# View migration status
supabase migration list

# Generate new migration
supabase migration new <name>
```

**Migration files** (12 total, `supabase/migrations/`):

| File | Summary |
|------|---------|
| `0001_tenancy.sql` | `workspaces`, `workspace_members` + RLS scaffold |
| `0002_projects_modules.sql` | `projects`, `modules` (self-referential tree) |
| `0003_authoring.sql` | `user_stories`, `acceptance_criteria` |
| `0004_atcs.sql` | `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`, FTS trigger |
| `0005_rls_helpers.sql` | SECURITY DEFINER helpers (`bunkai_is_workspace_member`, etc.) |
| `0006_bootstrap_workspace.sql` | `bunkai_bootstrap_workspace` RPC (atomic workspace + owner) |
| `0007_save_atc.sql` | `bunkai_save_atc` RPC (atomic ATC full-replace) |
| `0008_access_tokens.sql` | `access_tokens` + token validation functions |
| `0009_cross_cutting.sql` | `idempotency_keys`, `activity_log`, `feature_flags`, `user_view_state`, `magic_link_tokens` |
| `0010_workspace_invites.sql` | `workspace_invites` |
| `0011_split_token_secrets.sql` | `access_token_secrets`, `magic_link_token_secrets`, `workspace_invite_secrets` + QA role revocations |
| `0012_drop_legacy_token_hashes.sql` | Drops legacy inline hash columns |

Source: `supabase/migrations/` directory listing

---

## Monitoring / Observability

| Mechanism | Detail |
|-----------|--------|
| **Structured JSON logs** | Every `withApiHandler`-wrapped route writes single-line JSON to stdout: `{ level, ts, component: "api", request_id, method, path, status, duration_ms, error_code? }` — Vercel captures and indexes these |
| **`x-request-id` header** | Injected on every API response. Clients quote it in bug reports. |
| **`activity_log` table** | Append-only workspace audit trail: `entity_type`, `entity_id`, `action`, `payload`, `workspace_id`, `actor_user_id`, `created_at`. QA/analytics roles can read their workspace's log. |
| **`magic_link_tokens` table** | Best-effort OTP issuance audit for replay detection / rate-limiting |
| **`idempotency_keys` table** | POST replay protection — status: `pending` → `succeeded` \| `failed`. 24h TTL. |
| **Vercel dashboard** | Runtime logs, function invocation metrics, error rates |

No external APM (Datadog, Sentry, New Relic) configured in the app code.

Source: `lib/api/logging.ts`, `lib/api/handler.ts`, `lib/api/idempotency.ts`, `supabase/migrations/0009_cross_cutting.sql`

---

## Security Posture

### Defense-in-depth layers

```
+--------------------------------------------------+
| Layer 1: Vercel / HTTPS                           |
|   - TLS enforced by Vercel (all envs)             |
|   - No HTTP fallback in production                |
+--------------------------------------------------+
           |
+--------------------------------------------------+
| Layer 2: Next.js Middleware (middleware.ts)       |
|   - Session validation on every request           |
|   - Protected prefixes: /projects, /onboarding   |
|   - Unauthenticated → redirect /login?next={path}|
+--------------------------------------------------+
           |
+--------------------------------------------------+
| Layer 3: API Route Auth (lib/api/auth.ts)        |
|   - requireAuth() → Bearer first, then cookie    |
|   - Bearer: O(1) prefix lookup + SHA-256 compare |
|   - Constant-time comparison (no timing oracle)  |
|   - requireScopeOrCookie() for write ops         |
+--------------------------------------------------+
           |
+--------------------------------------------------+
| Layer 4: Postgres RLS                            |
|   - ALL 14 tables have RLS enabled               |
|   - auth.uid() == workspace_members.user_id      |
|   - SECURITY DEFINER helpers prevent recursion   |
+--------------------------------------------------+
           |
+--------------------------------------------------+
| Layer 5: Admin Client Isolation                  |
|   - service_role ONLY after app-layer auth done  |
|   - Secret tables (access_token_secrets,         |
|     workspace_invite_secrets,                    |
|     magic_link_token_secrets) never exposed      |
|     to qa_inspector roles                        |
+--------------------------------------------------+
```

### Token Security

| Aspect | Implementation |
|--------|---------------|
| PAT secret entropy | 256-bit random (32 bytes), base64url-encoded |
| Storage | SHA-256 hash only — raw secret NEVER stored |
| Prefix | First 12 chars of secret — O(1) index lookup |
| Family prefix | `bk_pat_` — enables GitHub/GitGuardian secret scanning |
| Revocation | Soft-delete via `revoked_at` timestamp (no physical DELETE) |
| Comparison | Constant-time SHA-256 comparison |
| Invite tokens | Same pattern — hash in sibling table, raw token returned once |
| Active workspace | `bk_active_ws` cookie: httpOnly, SameSite=Lax, secure in production, 90-day maxAge |

### RBAC

| Role | Permissions |
|------|-------------|
| `viewer` | Read-only workspace data |
| `member` | Read + write project data |
| `admin` | Member permissions + invite/revoke members |
| `owner` | Admin + workspace rename + workspace delete |

**PAT scopes** (headless path):

| Scope | Grants |
|-------|--------|
| `atc:read` | Read ATCs |
| `atc:write` | Create/update ATCs |
| `run:execute` | Trigger test runs |
| `workspace:admin` | Workspace management operations |

### Open-Redirect Guard

`/auth/callback` validates `next` parameter: must start with `/` and not `//` — prevents open redirects from OTP links.

Source: `lib/api/middleware/bearer.ts`, `lib/api/pat.ts`, `app/auth/callback/route.ts`, `lib/api/workspace-cookie.ts`, `supabase/migrations/0005_rls_helpers.sql`

---

## Rollback Procedure

No formal rollback procedure documented. Inferred from platform:

1. **Application code rollback**: Vercel dashboard → Deployments → select previous deployment → "Redeploy"
2. **Database migration rollback**: Supabase does not support automatic down-migrations. Manual rollback requires:
   - Write a new migration that reverses the schema change
   - Run `supabase migration up` / `supabase db push`
   - Coordinate with any deployed code that depends on the old schema
3. **Feature flags**: `feature_flags` table supports gradual rollout via `enabled: false` per workspace — disable without redeploying code

> **Note**: No `supabase/migrations/*_rollback.sql` files exist. All migrations are forward-only.

Source: Inferred from Vercel + Supabase patterns; no rollback docs found in repo

---

## Supabase Connection Details (DBHub / QA)

The `dbhub.toml` at repo root configures the DBHub MCP for direct Postgres access:

| Variable | Notes |
|----------|-------|
| `DBHUB_TYPE` | Database type (postgres) |
| `DBHUB_HOST` | Supabase session pooler host |
| `DBHUB_PORT` | 5432 (session pooler) |
| `DBHUB_DATABASE` | `postgres` |
| `DBHUB_USER` | `qa_inspector_ro` or `qa_inspector_rw` |
| `DBHUB_PASSWORD` | Role password (provisioned out-of-band) |

> Use session pooler (port 5432) for QA DB connections — NOT the transaction pooler (port 6543), which does not support prepared statements.

Source: `.env.example`, `app/qa/qa-config.ts`

---

## Available Scripts

| Script | Command | Purpose |
|--------|---------|---------|
| `bun run dev` | `next dev` | Local development server |
| `bun run build` | `next build` | Production build |
| `bun run start` | `next start` | Serve production build |
| `bun run typecheck` | `tsc --noEmit` | TypeScript validation |
| `bun run types:gen` | `bun scripts/gen-supabase-types.ts` | Refresh `lib/types/supabase.ts` |
| `bun run openapi:gen` | `bun scripts/openapi-gen.ts` | Generate `public/openapi.json` |
| `bun run lint:check` | `eslint .` | Lint without fix |
| `bun run lint:fix` | `eslint --fix .` | Lint with auto-fix |
| `bun run repo:check` | format + lint + types + vars + skills | Full repo health |

Source: `package.json` (via project-config.md §Run Commands), `scripts/` directory listing

---

## Discovery Gaps

- [ ] **CI/CD absent** — no `.github/workflows/` found. Zero automated checks on PRs. Critical gap before regression testing setup.
- [ ] **Staging / production Supabase project ref** — needed for session cookie name `sb-<project-ref>-auth-token`. Not in any config file; retrieve from Supabase dashboard.
- [ ] **`vercel.json`** — not found. Vercel deployment uses all defaults. Redirects, headers, and function config are unverified.
- [ ] **`qa_inspector_ro` / `qa_inspector_rw` credentials** — CREATE ROLE DDL absent from migrations. Password must be retrieved from Supabase Studio or out-of-band provisioning script.
- [ ] **Supabase project reference** — `NEXT_PUBLIC_SUPABASE_URL` contains it (`https://<project-ref>.supabase.co`) but the ref itself is not extracted/documented anywhere in repo files.
- [ ] **Error tracking** — no Sentry, Datadog, or equivalent APM configured. Errors only visible via Vercel function logs.
- [ ] **Rate limiting** — `POST /api/v1/auth/magic-link` comment mentions "Phase F adds a real rate-limit middleware" — not yet implemented. Only upstream Supabase 429 is surfaced.
- [ ] **Email provider** — OTP delivery is entirely managed by Supabase Auth. The SMTP/email provider used by the Supabase project is not documented in the repo.
