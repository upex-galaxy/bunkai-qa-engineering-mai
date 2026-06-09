# Backend Infrastructure — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 3 — Infrastructure
> Sources: next.config.ts, middleware.ts, lib/env.ts, lib/api/*, app/api/v1/**, .env.example, lib/urls.ts, supabase/migrations/0001–0012

---

## Runtime

- **Language**: TypeScript (strict mode, `^5.9.3`)
- **Framework**: Next.js 15 App Router — API routes at `app/api/`
- **Runtime environment**: Bun `>= 1.0.0` (package manager + script runner; Node-compatible via Bun)
- **React version**: 19

---

## Build Configuration

```bash
# Install
bun install

# Development server
bun run dev       # next dev

# Production build
bun run build     # next build

# Production start
bun run start     # next start

# Type check
bun run typecheck         # tsc --noEmit
bun run types:check       # alias

# Generate Supabase TypeScript types
bun run types:gen         # scripts/gen-supabase-types.ts → lib/types/supabase.ts

# Generate OpenAPI spec
bun run openapi:gen       # scripts/openapi-gen.ts → public/openapi.json

# Lint
bun run lint:check        # eslint .
bun run lint:fix          # eslint --fix .

# Format
bun run format:check
bun run format:fix

# Full repo health check
bun run repo:check        # format + lint + types + vars + skills
```

Source: package.json (project-config.md §Run Commands)

---

## Database

- **Engine**: PostgreSQL 17 via Supabase
- **ORM**: None — raw Supabase JS client (`@supabase/supabase-js ^2.106.0`) + PostgREST + typed RPCs
- **Migrations**: `supabase/migrations/` — 12 files (`0001_tenancy.sql` → `0012_drop_legacy_token_hashes.sql`)
- **Type generation**: `bun run types:gen` → `lib/types/supabase.ts`
- **RLS**: Enabled on ALL 14 tables

### Supabase client pattern

| Client | File | Auth bypass | Usage |
|--------|------|-------------|-------|
| Server (cookie) | `lib/supabase/server.ts` | No — RLS active | Browser routes, middleware |
| Admin (service_role) | `lib/supabase/admin.ts` | Yes — bypasses RLS | Secret tables, cross-user ops |
| Browser | `lib/supabase/client.ts` | No | Client components |

### QA Database Roles

| Role | Permissions | Secret table access |
|------|-------------|---------------------|
| `qa_inspector_ro` | SELECT on `public.*`, BYPASSRLS | Explicitly REVOKED from `access_token_secrets`, `magic_link_token_secrets`, `workspace_invite_secrets` |
| `qa_inspector_rw` | SELECT/INSERT/UPDATE/DELETE on `public.*`, BYPASSRLS | Same revocations as above |

> **Note**: CREATE ROLE DDL for `qa_inspector_ro` / `qa_inspector_rw` is NOT in any migration file. Provisioned out-of-band via Supabase Studio or a separate infra script.

Source: `supabase/migrations/0011_split_token_secrets.sql`

### Database Connection Config

| Variable | Port | Purpose |
|----------|------|---------|
| `POSTGRES_URL` | 6543 | Pooled (Supabase pooler) — do NOT use for prepared statements |
| `POSTGRES_URL_NON_POOLING` | 5432 | Direct — use for migrations and type generation |
| `POSTGRES_PRISMA_URL` | 6543 + pgbouncer=true | Prisma-specific (pooled) |

---

## Auth Flow

> This section is the critical contract for KATA fixtures.

### Flow 1 — Magic Link (Browser / Cookie Session)

**Step-by-step:**

```
1. User submits email at /login (or POSTs to /api/v1/auth/magic-link)

2. POST /api/v1/auth/magic-link
   Content-Type: application/json
   Body: { "email": "user@example.com", "next": "/projects" }

   Server: supabase.auth.signInWithOtp(email, { emailRedirectTo: "<APP_URL>/auth/callback?next=/projects" })
   Response: { "ok": true }
   Side-effect: best-effort audit row in magic_link_tokens + magic_link_token_secrets

3. User clicks email link → GET /auth/callback?code=<otp_code>&next=/projects
   Server: supabase.auth.exchangeCodeForSession(code)
   Sets cookie: sb-<project-ref>-auth-token (httpOnly, managed by Supabase SSR)
   Redirects to: /projects (or the next param)

4. Subsequent requests to protected routes:
   Cookie: sb-<project-ref>-auth-token=<session>
   middleware.ts: createServerClient → supabase.auth.getUser() on every request
   If !user && isProtected(pathname) → redirect /login?next={pathname}
```

**Protected prefixes**: `/projects`, `/onboarding`
**Public prefixes**: `/login`, `/auth`, `/api/auth`

Source: `middleware.ts`, `app/api/v1/auth/magic-link/route.ts`, `app/auth/callback/route.ts`

---

### Flow 2 — Bearer PAT (Headless / CLI / AI Agent)

**Step 1 — Provision credentials (one time)**

Option A — New QA user:
```
POST /api/v1/auth/signup
Content-Type: application/json
Body: {
  "email": "qabot@example.com",
  "password": "secure-password",
  "pat_name": "ci-runner",
  "pat_scopes": ["atc:read", "atc:write", "run:execute", "workspace:admin"],
  "pat_expires_in_days": 365
}
Status: 201
Response: {
  "user": { "id": "<uuid>", "email": "..." },
  "session": { "access_token": "...", "refresh_token": "...", "expires_at": ..., "token_type": "bearer" },
  "pat": { "token": "bk_pat_<prefix>.<secret>", "id": "<uuid>", "name": "ci-runner", "scopes": [...], "expires_at": null },
  "warning": "Store the PAT token now — it cannot be retrieved later."
}
```

Option B — Existing user:
```
POST /api/v1/auth/signin
Content-Type: application/json
Body: {
  "email": "qabot@example.com",
  "password": "secure-password",
  "pat_scopes": ["atc:read", "atc:write"]
}
Status: 200
Response: { same shape as signup, with "user", "session", "pat" }
```

Option C — Session-authenticated (browser user mints a PAT):
```
POST /api/v1/tokens
Cookie: sb-<project-ref>-auth-token=<session>
Content-Type: application/json
Body: { "scopes": ["atc:read", "atc:write"], "expires_in_days": 30, "name": "my-token" }
Status: 201
Response: { "id": "...", "token": "bk_pat_...", "scopes": [...], "expires_at": "...", "warning": "..." }
```

**Step 2 — Use the PAT on every API request**

```
GET /api/v1/me
Authorization: Bearer bk_pat_<12-char-prefix>.<secret-remainder>

Server auth pipeline (bearer.ts):
  1. Parse Authorization header — must start with "Bearer "
  2. Strip "bk_pat_" family prefix
  3. Split at first "." → prefix (12 chars) + remainder
  4. Reconstruct fullSecret = prefix + remainder
  5. SELECT from access_tokens WHERE token_prefix = prefix (O(1) index)
  6. Fetch hash from access_token_secrets for candidate ids
  7. Compare SHA-256(fullSecret) === stored hash (constant-time)
  8. Check: revoked_at IS NULL AND (expires_at IS NULL OR expires_at > now())
  9. Fire-and-forget UPDATE access_tokens SET last_used_at = now()
  → Returns: { userId, workspaceId, scopes, tokenId }
```

**PAT token format**: `bk_pat_<12-char-base64url-prefix>.<base64url-remainder>`
Full secret = 32 random bytes (256-bit entropy), base64url-encoded, split at position 12.

**PAT scopes**:

| Scope | Purpose |
|-------|---------|
| `atc:read` | Read test cases (ATCs) |
| `atc:write` | Create/update ATCs |
| `run:execute` | Trigger test runs |
| `workspace:admin` | Workspace management ops |

**Revocation**: `DELETE /api/v1/tokens/{id}` → soft-delete (`revoked_at = now()`). No physical DELETE. Row stays for audit.

Source: `lib/api/middleware/bearer.ts`, `lib/api/pat.ts`, `lib/api/auth.ts`, `app/api/v1/auth/signup/route.ts`, `app/api/v1/auth/signin/route.ts`, `app/api/v1/tokens/route.ts`

---

### Flow 3 — Active Workspace Cookie (Soft Preference)

After sign-in, the browser workspace switcher POSTs to set a preference cookie:

```
POST /api/v1/me/active-workspace
Cookie: sb-<project-ref>-auth-token=<session>
Content-Type: application/json
Body: { "workspace_id": "<uuid>" }

Response sets: bk_active_ws=<workspace-uuid> (httpOnly, SameSite=Lax, maxAge=90 days)
```

This cookie is a UI preference — it does NOT affect JWT or RLS. PAT callers use `workspaceId` from the token itself.

Source: `app/api/v1/me/active-workspace/route.ts`, `lib/api/workspace-cookie.ts`

---

## API Structure

> Complete catalog of all 19 handler registrations across `app/api/v1/` + OpenAPI + docs endpoints.

| Method | Path | Auth required | Body shape | Status codes | Description |
|--------|------|---------------|------------|--------------|-------------|
| GET | `/api/v1` | None | — | 200 | API discovery banner (version, openapi, docs) |
| OPTIONS | `/api/v1` | None | — | 204 | CORS preflight |
| GET | `/api/v1/health` | None | — | 200 | Liveness probe — returns env + timestamp |
| GET | `/api/v1/me` | Cookie or Bearer | — | 200, 401 | Identity + workspaces + active_workspace_id |
| POST | `/api/v1/me/active-workspace` | Cookie | `{ workspace_id: uuid }` | 200, 400, 401, 403 | Switch active workspace (sets `bk_active_ws` cookie) |
| GET | `/api/v1/workspaces` | Cookie or Bearer | — | 200, 401 | List caller's workspaces (RLS/membership filtered) |
| POST | `/api/v1/workspaces` | Cookie | `{ name, slug }` | 201, 400, 401, 409, 422 | Create workspace (calls `bunkai_bootstrap_workspace` RPC) |
| GET | `/api/v1/workspaces/{id}` | Cookie | — | 200, 400, 401, 404 | Get single workspace |
| PATCH | `/api/v1/workspaces/{id}` | Cookie (owner) | `{ name? }` | 200, 400, 401, 403 | Update workspace name (RLS: owner only) |
| GET | `/api/v1/workspaces/{id}/invites` | Cookie (admin+) | — | 200, 400, 401 | List workspace invites with derived status |
| POST | `/api/v1/workspaces/{id}/invites` | Cookie (admin+) | `{ email, role }` | 201, 400, 401, 403, 422 | Issue invite — returns raw token + accept_url (once) |
| POST | `/api/v1/workspaces/{id}/invites/{inviteId}` | Cookie (admin+) | — | 200, 400, 401, 403, 404 | Rotate invite token (extend expiry, new secret) |
| DELETE | `/api/v1/workspaces/{id}/invites/{inviteId}` | Cookie (admin+) | — | 200, 400, 401, 403, 404 | Revoke invite (sets `revoked_at`) |
| POST | `/api/v1/invites/accept` | Cookie | `{ token: string }` | 200, 400, 401, 403, 404, 409 | Accept invite (email match required) |
| GET | `/api/v1/tokens` | Cookie | — | 200, 401 | List caller's PATs (no secrets) |
| POST | `/api/v1/tokens` | Cookie | `{ scopes[], name?, workspace_id?, expires_in_days? }` | 201, 400, 401, 422 | Mint PAT (session only — no PAT-to-PAT issuance) |
| DELETE | `/api/v1/tokens/{id}` | Cookie | — | 204, 400, 401, 404 | Revoke PAT (soft-delete) |
| POST | `/api/v1/auth/magic-link` | None | `{ email, next? }` | 200, 400, 422, 429, 502 | Trigger magic-link OTP email |
| POST | `/api/v1/auth/signup` | None | `{ email, password, pat_name?, pat_scopes?, pat_expires_in_days? }` | 201, 400, 409, 422, 502 | Provision QA user (email_confirm: true) + mint PAT |
| POST | `/api/v1/auth/signin` | None | `{ email, password, pat_name?, pat_scopes?, pat_expires_in_days? }` | 200, 400, 401, 422 | Sign in + mint PAT in one round trip |
| GET | `/api/openapi` | None | — | 200 | OpenAPI JSON spec (static, force-cached 300s) |
| GET | `/api/docs` | None | — | 200 | Interactive Scalar docs UI |

**Non-v1 routes (browser-facing)**:

| Route | Method | Handler file | Auth |
|-------|--------|--------------|------|
| `/auth/callback` | GET | `app/auth/callback/route.ts` | None (OTP exchange) |

Source: `app/api/v1/**` route.ts files, `app/auth/callback/route.ts`

---

## Request / Response Conventions

| Aspect | Detail |
|--------|--------|
| Content-Type | `application/json` for all v1 routes |
| Request tracing | `x-request-id` header injected on ALL responses (inbound propagated or minted) |
| Error envelope | `{ error: { code, message, details?, request_id? } }` |
| Error codes | `bad_request` \| `validation_failed` \| `unauthorized` \| `forbidden` \| `not_found` \| `conflict` \| `rate_limited` \| `internal_error` \| `upstream_error` |
| Idempotency | `Idempotency-Key: <8–128 chars, [a-zA-Z0-9_-]>` header on POST endpoints; keyed on `(user_id, endpoint, key)` with 24h TTL |
| CORS preflight | `OPTIONS /api/v1` → 204 with `access-control-allow-headers: authorization, content-type, idempotency-key, x-request-id` |

Source: `lib/api/handler.ts`, `lib/api/error-envelope.ts`, `lib/api/idempotency.ts`, `app/api/v1/route.ts`

---

## Environment Variables

> All required unless marked optional. Source: `.env.example`, `lib/env.ts`, `middleware.ts`.

### Supabase (Required)

| Variable | Notes |
|----------|-------|
| `NEXT_PUBLIC_SUPABASE_URL` | `https://<project-ref>.supabase.co` — used by middleware + client |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | **Runtime name used by `middleware.ts` and `lib/env.ts`**. `.env.example` lists `SUPABASE_PUBLISHABLE_KEY` as the display name — these are the same key; set BOTH or alias one to the other |
| `SUPABASE_PUBLISHABLE_KEY` | Browser-safe publishable key (alias / legacy name in `.env.example`) |
| `SUPABASE_SECRET_KEY` | Service-role key (`.env.example` name). `lib/env.ts` reads it as `SUPABASE_SERVICE_ROLE_KEY` — **set BOTH variable names or the env validation will fail** |
| `SUPABASE_SERVICE_ROLE_KEY` | Runtime name read by `lib/env.ts` — required for admin client |
| `SUPABASE_JWT_SECRET` | Optional in MVP; required for custom JWT claim signing |
| `SUPABASE_ACCESS_TOKEN` | Supabase MCP / management API personal access token |

### Postgres Direct Connection

| Variable | Notes |
|----------|-------|
| `POSTGRES_HOST` | `db.<project-ref>.supabase.co` |
| `POSTGRES_USER` | `postgres` (default) |
| `POSTGRES_PASSWORD` | Supabase database password |
| `POSTGRES_DATABASE` | `postgres` (default) |
| `POSTGRES_URL` | Pooled connection string (port 6543) |
| `POSTGRES_URL_NON_POOLING` | Direct connection string (port 5432) — use for migrations |
| `POSTGRES_PRISMA_URL` | Pooled + pgbouncer=true (Prisma-specific) |

### App Config

| Variable | Notes |
|----------|-------|
| `NEXT_PUBLIC_APP_URL` | Base URL for auth redirects, invite links, OTP callbacks. Local: `http://localhost:3000` |

### External Integrations

| Variable | Purpose |
|----------|---------|
| `ATLASSIAN_URL` | Jira Cloud base URL |
| `ATLASSIAN_EMAIL` | Jira account email |
| `ATLASSIAN_API_TOKEN` | Jira API token |
| `TAVILY_API_KEY` | Web search MCP |
| `N8N_API_URL` | n8n workflow automation MCP |
| `N8N_API_KEY` | n8n API key |
| `RESEND_API_KEY` | Transactional email |

Source: `.env.example`, `lib/env.ts`

---

## External Services

| Service | Purpose | Integration |
|---------|---------|-------------|
| Supabase (Postgres 17) | Primary DB + Auth + SSR | `@supabase/supabase-js` + `@supabase/ssr` |
| Supabase Auth | Magic-link OTP + user lifecycle | `supabase.auth.*` SDK methods |
| Vercel | Serverless hosting + CDN | Next.js build target; env detection via `VERCEL_ENV` |
| Email provider | OTP delivery | Managed by Supabase Auth (not configured in app code) |
| Jira Cloud | External issue tracker (reference only) | `user_stories.external_id / external_url` — no live API calls from app |

Source: `lib/urls.ts`, `.env.example`, `app/qa/qa-config.ts`

---

## Observability

| Mechanism | Detail |
|-----------|--------|
| Request logging | `lib/api/logging.ts` — single-line JSON to stdout; all `withApiHandler` wrapped routes. Fields: `level`, `ts`, `component=api`, `request_id`, `method`, `path`, `status`, `duration_ms`, `error_code` |
| `x-request-id` | Injected on every API response. Inbound header propagated if present; else fresh UUID. Clients quote this in bug reports |
| `activity_log` table | Append-only workspace-scoped audit trail. Written by service_role via middleware. Fields: `entity_type`, `entity_id`, `action`, `payload` |
| `magic_link_tokens` | Best-effort OTP issuance audit for rate-limiting signals |
| Vercel logs | `console.log/warn/error` captured and indexed by Vercel (structured JSON) |

Source: `lib/api/logging.ts`, `lib/api/handler.ts`, `supabase/migrations/0009_cross_cutting.sql`

---

## Discovery Gaps

- [ ] `NEXT_PUBLIC_SUPABASE_ANON_KEY` vs `SUPABASE_PUBLISHABLE_KEY` mismatch — `middleware.ts` + `lib/env.ts` read `NEXT_PUBLIC_SUPABASE_ANON_KEY`; `.env.example` only defines `SUPABASE_PUBLISHABLE_KEY`. Set both variable names to the same value (or confirm Vercel integration injects `NEXT_PUBLIC_SUPABASE_ANON_KEY` directly).
- [ ] `SUPABASE_SECRET_KEY` vs `SUPABASE_SERVICE_ROLE_KEY` — `.env.example` uses the former; `lib/env.ts` validates the latter. Must set `SUPABASE_SERVICE_ROLE_KEY` for the admin client to work.
- [ ] `qa_inspector_ro` / `qa_inspector_rw` CREATE ROLE DDL — not in any migration file; provisioned out-of-band.
- [ ] Supabase project reference (`<project-ref>`) — needed to construct the session cookie name (`sb-<project-ref>-auth-token`) for test fixture configuration.
- [ ] ATC slug generation strategy — `atcs.slug` is UNIQUE per project but no server-side auto-generation was found in API routes or migrations.
- [ ] Idempotency adoption — `lib/api/idempotency.ts` exists but no current `app/api/v1/` route calls `beginIdempotentRequest`. Feature is available but not yet wired.
- [ ] `GET /api/v1/workspaces/{id}` — discovered to accept Cookie only in code (no `requireAuth` call, uses `createClient()` directly); does NOT support Bearer. Not reflected in prior-phase catalog.
