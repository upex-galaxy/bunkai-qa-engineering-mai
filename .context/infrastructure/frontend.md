# Frontend Infrastructure — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 3 — Infrastructure
> Sources: app/layout.tsx, app/globals.css, tailwind.config.ts, components.json, tsconfig.json, next.config.ts, app/(app)/**, app/(auth)/**, components/**, lib/urls.ts

---

## Framework

- **Framework**: Next.js 15 App Router
- **React**: 19 (Server + Client components)
- **Language**: TypeScript `^5.9.3` (strict mode)
- **Rendering**: Server Components by default; Client Components explicitly marked `'use client'`

---

## UI Stack

| Library | Version | Purpose |
|---------|---------|---------|
| Tailwind CSS | `^3.4` | Utility-first styling with custom design tokens |
| shadcn/ui | new-york style | Component primitives wired to Radix + Tailwind |
| Radix UI | Multiple | Headless primitives: Dialog, DropdownMenu, Tabs, Tooltip |
| Lucide React | `^1.16.0` | Icon library (shadcn iconLibrary setting) |
| Monaco Editor | `@monaco-editor/react ^4.7.0` | In-app code editing for ATC step/assertion authoring |
| TanStack Table | `^8.21.3` | ATC list view data grids |
| Sonner | `^2.0.7` | Toast notifications (dark theme, top-right, richColors) |
| cmdk | `^1.1.1` | `Cmd/Ctrl+K` command palette for global navigation |
| Scalar | `@scalar/api-reference-react ^0.9.38` | Interactive API docs viewer at `/api/docs` |

---

## Routing

All routes use the Next.js App Router convention. Route groups in parentheses are layout-only (not in URL).

| Route (URL) | File | Auth | Feature |
|-------------|------|------|---------|
| `/` | `app/page.tsx` | None | Redirect → `/login` |
| `/login` | `app/(auth)/login/page.tsx` | None (public) | Magic-link email form |
| `/auth/callback` | `app/auth/callback/route.ts` | None (OTP exchange) | Supabase OTP → session cookie, redirect to `next` |
| `/onboarding` | `app/(app)/onboarding/page.tsx` | Cookie session | Create first workspace flow |
| `/projects` | `app/(app)/projects/page.tsx` | Cookie session | Workspace/project index — auto-redirects to first project |
| `/projects/[projectSlug]` | `app/(app)/projects/[projectSlug]/page.tsx` | Cookie session | Project detail — ATC list |
| `/projects/[projectSlug]/atcs/[atcId]` | `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx` | Cookie session | ATC editor — steps, assertions, AC anchoring |
| `/workspaces/[id]/members` | `app/(app)/workspaces/[id]/members/page.tsx` | Cookie session | Workspace member + invite management |
| `/invites/accept` | `app/invites/accept/page.tsx` | Cookie session | Accept workspace invite via token |
| `/api/docs` | `app/api/docs/page.tsx` | None | Scalar API docs UI (RSC) |
| `/design-tokens` | `app/design-tokens/page.tsx` | None | Design system token reference page |
| `/qa` | `app/qa/page.tsx` | None | QA integration guide / onboarding docs |

**Middleware matcher**: all paths EXCEPT `_next/static`, `_next/image`, `favicon.ico`, static assets.

Source: `app/` directory scan, `middleware.ts`

---

## Component Architecture

```
components/
  atcs/
    AnchoringPanel.tsx       — AC binding UI (ATC ↔ AcceptanceCriteria M:N)
    AtcEditor.tsx            — Full ATC edit form (header + steps + assertions)
    AtcTable.tsx             — TanStack Table-based ATC list view
    StepEditor.tsx           — Monaco-backed step/assertion editor
  layout/
    CommandPalette.tsx       — cmdk-powered Cmd+K palette
    Sidebar.tsx              — Module tree navigation
    Topbar.tsx               — App header bar
    Wordmark.tsx             — Brand wordmark
    WorkspaceSwitcher.tsx    — Workspace selection dropdown
  providers/
    auth-context.tsx         — Client-side AuthContext (React 19 Context API)
                               Exposes: user, session, loading, signInWithMagicLink, signOut
  ui/
    badge.tsx                — shadcn Badge
    button.tsx               — shadcn Button
    card.tsx                 — shadcn Card
    input.tsx                — shadcn Input
    label.tsx                — shadcn Label
    tabs.tsx                 — shadcn Tabs
```

**Naming conventions**:
- `PascalCase` for all components
- Client components with `'use client'` directive at top
- shadcn components aliased at `@components/ui` (see `tsconfig.json` paths)

Source: `components/` directory scan, `components.json`

---

## State Management

| Mechanism | Scope | Usage |
|-----------|-------|-------|
| React Server Components (RSC) | Server-only | All data-fetching pages (`app/(app)/**`) use `async` RSC with direct Supabase calls |
| React Context (`AuthContext`) | Client global | `components/providers/auth-context.tsx` — wraps auth state for client components |
| Supabase `onAuthStateChange` | Client global | Real-time session refresh listening in `AuthProvider` |
| `bk_active_ws` cookie | Server-persisted | Active workspace preference — set via `POST /api/v1/me/active-workspace`, read in `GET /api/v1/me` |
| `user_view_state` DB table | DB-persisted | Per-user UI state (view layout, column config) — `(user_id, project_id, view_kind)` |
| React `useState` / `useCallback` | Local component | `AtcEditor`, `StepEditor`, form state |

No global client state library (Redux, Zustand, Jotai) — server-first architecture.

Source: `components/providers/auth-context.tsx`, `app/(app)/projects/page.tsx`, `lib/supabase/with-workspace.ts`

---

## Design System

### Theme: Dark-only

The app ships dark-mode only (`<html class="dark">`). No light theme. CSS custom properties in `app/globals.css`:

```
Surfaces (bg):   --bg-0 (#0a0b0d) → --bg-5 (#2d333c)    6-stop dark scale
Foreground (fg): --fg-0 (#f1f3f5) → --fg-4 (#4a5057)    5-stop text scale
Strokes:         --stroke-1 → --stroke-strong             4 alpha values
Accent:          --accent (#d9543f) vermillion             + hi / glow / soft variants
```

### Signal Colors (ATC status)

| State | Color | Token |
|-------|-------|-------|
| pass | `#2fb673` (green) | `--pass` / `--pass-bg` |
| fail | `#e5484d` (red) | `--fail` / `--fail-bg` |
| blocked | `#e8a838` (amber) | `--blocked` / `--blocked-bg` |
| skipped | `#8a91a0` (gray) | `--skipped` / `--skipped-bg` |
| running | `#4f8cf7` (blue) | `--running` / `--running-bg` |

### Layer Chip Colors

| Layer | Color |
|-------|-------|
| UI | `#8b6df0` (purple) |
| API | `#4f8cf7` (blue) |
| Unit | `#2fb673` (green) |

### Typography

- **Sans**: Inter (Google Fonts, `--font-inter`) — base 13px, line-height 1.45
- **Mono**: JetBrains Mono (Google Fonts, `--font-jetbrains-mono`) — code, steps, slugs
- **JP**: Noto Serif JP (Google Fonts, weights 600/700) — wordmark / branding

### Border Radii

`--r-1: 3px` / `--r-2: 5px` / `--r-3: 7px` / `--r-4: 10px` — mapped to Tailwind `rounded-{1..4}`

### shadcn Config

```json
{
  "style": "new-york",
  "rsc": true,
  "baseColor": "neutral",
  "cssVariables": true,
  "prefix": ""
}
```

Source: `app/globals.css`, `tailwind.config.ts`, `components.json`, `app/layout.tsx`

---

## Test IDs / Selector Strategy

**No `data-testid` attributes found** in any app/ or components/ file. The app does not currently have an explicit test selector convention.

Playwright test selectors must rely on:
1. **Semantic HTML roles** — `button`, `input`, `nav`, `main`, headings
2. **Aria labels** — not yet systematically applied (no `aria-label` found in component scan)
3. **Text content** — button labels, headings, status text (stable English strings)
4. **CSS classes** — Tailwind classes are generated at build time; prefer semantic selectors
5. **URL patterns** — `/projects/[slug]`, `/projects/[slug]/atcs/[id]`

> Discovery gap: no `data-testid` or systematic `aria-label` strategy exists. Recommend establishing a `data-testid` convention when test-automation work begins.

Source: `components/**/*.tsx` grep scan

---

## Build / Bundle

### `next.config.ts` settings

| Setting | Value | Notes |
|---------|-------|-------|
| `reactStrictMode` | `true` | Double-invocation in dev |
| `typedRoutes` | `true` | Compile-time route type checking |
| `outputFileTracingRoot` | `path.resolve(import.meta.dirname)` | Vercel file tracing root |
| `images.remotePatterns` | `[]` | No external image domains configured |

### TypeScript paths (`tsconfig.json`)

| Alias | Resolves to |
|-------|-------------|
| `@/*` | `./*` (root) |
| `@app/*` | `./app/*` |
| `@components/*` | `./components/*` |
| `@lib/*` | `./lib/*` |

**Module resolution**: `bundler` (Next.js 15 default)
**Target**: ES2022
**Types**: `bun-types`, `node`

Source: `next.config.ts`, `tsconfig.json`

---

## Fonts

All fonts served via `next/font/google` (self-hosted by Next.js at build time, no external requests):
- `Inter` — variable: `--font-inter`
- `JetBrains_Mono` — variable: `--font-jetbrains-mono`
- `Noto_Serif_JP` weights 600/700 — variable: `--font-noto-serif-jp`

Source: `app/layout.tsx`

---

## Discovery Gaps

- [ ] No `data-testid` or systematic `aria-label` convention — selector strategy for Playwright automation must be established before `test-automation` work.
- [ ] `app/(app)/projects/[projectSlug]/page.tsx` — the project detail page (ATC list view) was not deeply read; TanStack Table column definitions and filter state details are not documented.
- [ ] `app/(app)/workspaces/[id]/members/page.tsx` — members management UI internals not read.
- [ ] `app/(app)/onboarding/page.tsx` — onboarding flow steps (workspace creation form) not read.
- [ ] No Client-Side routing guard found — auth protection at app level is handled exclusively by server middleware (`middleware.ts`). Client components that fetch data assume session exists.
- [ ] `app/design-tokens/page.tsx` — design token reference page may be a useful visual aid for QA but was not read.
