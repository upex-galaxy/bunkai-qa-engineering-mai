# Project Config — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 1 — Constitution
> Sources: package.json, .env.example, next.config.ts, middleware.ts, lib/api/pat.ts, app/qa/qa-config.ts

---

## Repositories

| Repo | Absolute Path | Role |
|------|--------------|------|
| `upex-bunkai-tms` | `/Users/maibethvega/Documents/Maibeth/UpexGalaxy_QA+IA/Projects/upex-bunkai-tms` | Target app — monorepo: Next.js FE + API Routes + Supabase backend |
| `bunkai-qa-engineering` | `/Users/maibethvega/Documents/Maibeth/UpexGalaxy_QA+IA/Projects/bunkai-qa-engineering` | QA framework (this repo) |

Source: app/qa/qa-config.ts (`reposShape: 'mono'`, `backendRepo`, `frontendRepo`)

---

## Tech Stack

| Layer | Technology | Version / Notes |
|-------|-----------|-----------------|
| Runtime | Bun | >= 1.0.0 (package.json engine) |
| Framework | Next.js (App Router) | `^15` |
| Language | TypeScript | `^5.9.3` |
| React | React | `^19` |
| Styling | Tailwind CSS | `^3.4` |
| Component primitives | Radix UI (Dialog, DropdownMenu, Tabs, Tooltip) | Multiple `@radix-ui/*` |
| Component library | shadcn/ui | (wired via Radix + CVA + tailwind-merge) |
| Icons | Lucide React | `^1.16.0` |
| Code editor | Monaco Editor (React) | `@monaco-editor/react ^4.7.0` |
| Table | TanStack Table | `^8.21.3` |
| Toast | Sonner | `^2.0.7` |
| Command palette | cmdk | `^1.1.1` |
| Database | Supabase (PostgreSQL 17) | `@supabase/supabase-js ^2.106.0` |
| Auth / Session | Supabase SSR + cookie session | `@supabase/ssr ^0.10.3` |
| Auth (headless) | Bearer PAT (`bk_pat_*`) | Custom — see lib/api/pat.ts |
| API spec | OpenAPI (via zod-to-openapi) | `@asteasolutions/zod-to-openapi ^8.5.0` |
| API docs | Scalar UI | `@scalar/api-reference-react ^0.9.38` |
| Validation | Zod | `^4.4.3` |
| Linter | ESLint (antfu config) | `^9.28.0` |
| Formatter | Prettier | `^3.7.4` |
| Git hooks | Husky + lint-staged | `husky ^9.1.7` |
| DB browser (MCP) | DBHub | `dbhub.toml` config at root |

Source: package.json

---

## Environments

| Name | URL | Notes |
|------|-----|-------|
| local | `http://localhost:3000` | Default `NEXT_PUBLIC_APP_URL` in `.env.example` |
| staging | Discovery Gap — not in `.env.example` | Likely on Vercel; must be retrieved from Vercel project |
| production | Discovery Gap — not in `.env.example` | Likely on Vercel |

Source: .env.example (`NEXT_PUBLIC_APP_URL=http://localhost:3000`)

---

## Run Commands

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
bun run typecheck   # tsc --noEmit
# (also aliased as)
bun run types:check

# Lint
bun run lint:check  # eslint .
bun run lint:fix    # eslint --fix .

# Format
bun run format:check
bun run format:fix

# Full repo health check
bun run repo:check  # format + lint + types + vars + skills

# Generate Supabase types
bun run types:gen   # bun scripts/gen-supabase-types.ts

# Supabase migrations (via Supabase CLI)
# (no bun script — run supabase db push / supabase migration up directly)
```

Source: package.json

---

## Auth Flow

Bunkai supports two auth modes:

### 1. Cookie-based (browser / magic-link)
- User enters email at `/login`
- Server calls `POST /api/v1/auth/magic-link` → Supabase sends OTP email
- Supabase callback at `/auth/callback` exchanges token → sets `sb-<project-ref>-auth-token` cookie
- Middleware (`middleware.ts`) validates session via `supabase.auth.getUser()` on every request
- Protected routes: `/projects/*`, `/onboarding`
- Public routes: `/login`, `/auth/*`, `/api/auth/*`

### 2. Bearer PAT (headless / CLI / AI agent)
- Format: `bk_pat_<12-char-prefix>.<secret>`
- Issued via:
  - `POST /api/v1/tokens` (session-authenticated, cookie required)
  - `POST /api/v1/auth/signup` (one-time QA bot provisioning with email+password)
  - `POST /api/v1/auth/signin` (email+password → fresh PAT)
- PAT middleware: `lib/api/middleware/` looks up `token_prefix` in `access_tokens`, then constant-time SHA-256 compare against `access_token_secrets.hash`
- Scopes: `atc:read` | `atc:write` | `run:execute` | `workspace:admin`
- Revocation: `DELETE /api/v1/tokens/{id}` → sets `revoked_at` (soft-delete, no physical DELETE)

Source: middleware.ts, lib/api/pat.ts, app/qa/qa-config.ts

---

## API Contract

| Aspect | Detail |
|--------|--------|
| Base URL | `/api/v1` |
| Spec format | OpenAPI (JSON) |
| Spec endpoint | `GET /api/openapi` |
| Docs UI | Scalar — `GET /api/docs` |
| Spec generator | `app/api/openapi/` + `app/api/v1/route.openapi.ts` pattern (zod-to-openapi) |

### Key API Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/api/v1/health` | None | Liveness probe |
| GET | `/api/v1/me` | Cookie or Bearer | Identity + workspaces |
| GET | `/api/v1/workspaces` | Cookie or Bearer | Caller's workspace list |
| POST | `/api/v1/auth/signup` | None | Provision QA user + mint PAT |
| POST | `/api/v1/auth/signin` | None | email+password → session + fresh PAT |
| POST | `/api/v1/auth/magic-link` | None | Trigger magic-link email |
| POST | `/api/v1/tokens` | Cookie or Bearer | Mint a PAT |
| GET | `/api/v1/tokens` | Cookie or Bearer | List PATs (no secrets) |
| DELETE | `/api/v1/tokens/{id}` | Cookie or Bearer | Revoke a PAT |
| GET | `/api/v1/workspaces/{id}/invites` | Cookie or Bearer (workspace:admin) | List workspace invites |
| POST | `/api/v1/invites/accept` | None (token in body) | Accept an invite |
| PUT | `/api/v1/me/active-workspace` | Cookie or Bearer | Switch active workspace |
| GET | `/api/docs` | None | Interactive Scalar docs |
| GET | `/api/openapi` | None | OpenAPI JSON spec |

Source: app/qa/qa-config.ts (endpoints array), app/api/v1/ directory structure

---

## Database

| Aspect | Detail |
|--------|--------|
| Engine | PostgreSQL 17 (via Supabase) |
| Connection config | `dbhub.toml` (root) |
| Session Pooler port | 5432 |
| Transaction Pooler port | 6543 (NOT for use — no prepared statements) |
| Migrations | `supabase/migrations/0001–0012` |
| Type generation | `bun run types:gen` → `lib/types/supabase.ts` |
| RLS | Enabled on all 14 tables |

### QA Database Roles

| Role | Access |
|------|--------|
| `qa_inspector_ro` | SELECT on `public.*`, BYPASSRLS — cannot read secret tables |
| `qa_inspector_rw` | SELECT/INSERT/UPDATE/DELETE on `public.*`, BYPASSRLS — cannot read secret tables |

Secret tables explicitly revoked from QA roles:
- `access_token_secrets`
- `magic_link_token_secrets`
- `workspace_invite_secrets`

Source: migrations/0011_split_token_secrets.sql, app/qa/qa-config.ts

---

## Issue Tracker

| Aspect | Detail |
|--------|--------|
| Tool | Jira Cloud |
| Project key | `BK` |
| Base URL | `https://upexgalaxy69.atlassian.net/` |
| Credentials reference | Jira Epic BK-29 (per app/qa/qa-config.ts `credentialsSource`) |
| Known story | BK-15 — "TMS-AC | Manage criteria under a user story" (source spec: FR-008) |

---

## Required Environment Variables

```bash
# Supabase project (Required)
NEXT_PUBLIC_SUPABASE_URL=       # https://<project-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=  # (used by middleware — NOTE: .env.example uses SUPABASE_PUBLISHABLE_KEY;
                                #  middleware.ts reads NEXT_PUBLIC_SUPABASE_ANON_KEY — verify alignment)
SUPABASE_PUBLISHABLE_KEY=       # browser-safe publishable key
SUPABASE_SECRET_KEY=            # server-only secret key
SUPABASE_JWT_SECRET=            # sign/verify custom JWTs
SUPABASE_ACCESS_TOKEN=          # Supabase MCP / management API

# App
NEXT_PUBLIC_APP_URL=            # http://localhost:3000 (local) | https://<deploy-url> (prod)

# Postgres direct connection
POSTGRES_HOST=
POSTGRES_USER=postgres
POSTGRES_PASSWORD=
POSTGRES_DATABASE=postgres
POSTGRES_URL=                   # pooled (port 6543)
POSTGRES_URL_NON_POOLING=       # direct (port 5432)

# Atlassian (Jira)
ATLASSIAN_URL=
ATLASSIAN_EMAIL=
ATLASSIAN_API_TOKEN=

# Tavily (web search MCP)
TAVILY_API_KEY=

# n8n (workflow automation MCP)
N8N_API_URL=
N8N_API_KEY=

# Resend (email)
RESEND_API_KEY=

# QA-specific (from qa-config.ts)
DBHUB_TYPE=
DBHUB_HOST=
DBHUB_PORT=
DBHUB_DATABASE=
DBHUB_USER=
DBHUB_PASSWORD=
API_BASE_URL=
OPENAPI_SPEC_PATH=
API_TOKEN=
POSTMAN_API_KEY=
```

Source: .env.example, middleware.ts, app/qa/qa-config.ts

---

## MCP Servers (configured in `.mcp.json`)

| MCP | Purpose | Config |
|-----|---------|--------|
| DBHub | DB query access (PostgreSQL via `dbhub.toml`) | `bunx -y @bytebase/dbhub@latest --config dbhub.toml` |
| OpenAPI | API endpoint exploration (dynamic tools) | `bunx -y @ivotoby/openapi-mcp-server --tools dynamic` |
| Postman | Postman collection access | Remote HTTP — `https://mcp.postman.com/mcp` |
| Playwright | Browser automation | `bunx @playwright/mcp@latest --caps vision,pdf,testing,tracing,tabs` |
| Supabase | Project management / migrations | via `SUPABASE_ACCESS_TOKEN` |
| Atlassian | Jira / Confluence | via `ATLASSIAN_*` vars |
| Tavily | Web search | via `TAVILY_API_KEY` |
| n8n | Workflow automation | via `N8N_API_*` vars |

Source: app/qa/qa-config.ts (`mcp` block), .env.example

---

## Discovery Gaps

- [ ] Staging and production URLs — not in `.env.example`; must be retrieved from Vercel project settings.
- [ ] `NEXT_PUBLIC_SUPABASE_ANON_KEY` vs `SUPABASE_PUBLISHABLE_KEY` — `middleware.ts` reads `NEXT_PUBLIC_SUPABASE_ANON_KEY` but `.env.example` only defines `SUPABASE_PUBLISHABLE_KEY`; likely an alias or migration — verify which var name the deployed instance uses.
- [ ] Supabase project reference — `<project-ref>` appears in snippets but the actual ref is not in `.env.example`; needed for the DBHub pooler user (`DBHUB_USER`) and cookie name.
- [ ] ATC slug generation strategy — `atcs.slug` is UNIQUE per project but no auto-generation logic was found in migrations or lib/ (likely a server action or client-side logic in `app/(app)/projects/[projectSlug]/atcs/`).
- [ ] Vercel project name and org — needed for `vercel` CLI integration; not in any config file.
- [ ] `qa_inspector_ro` / `qa_inspector_rw` role creation SQL — referenced in migration 0011 REVOKE statements but CREATE ROLE DDL not in any migration file (likely provisioned via Supabase Studio or a separate infra script).
