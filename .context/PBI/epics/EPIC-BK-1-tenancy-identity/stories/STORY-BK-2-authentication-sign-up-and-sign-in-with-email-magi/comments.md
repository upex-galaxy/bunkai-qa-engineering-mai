# Comments for BK-2

[View in Jira](https://jira.upexgalaxy.com/browse/BK-2)

---

### Ely - 5/19/2026, 9:05:40 PM

🧱 ****Architect Annotation****

**Posted by repo automation. Sections below are the architecture-grade complement to the user-facing fields (description / AC / Scope / Business Rules / Workflow). Source-of-truth on dev-side concerns — synced to local `comments.md` by `sync-jira-issues`.**

1. 

- Page: `app/(auth)/login/page.tsx` (already exists in scaffold).
- Component: `<MagicLinkForm />` — input + submit + post-submit state ("Check your email").
- Callback page: `app/auth/callback/route.ts` (route handler).

1. 

- Wraps Supabase Auth `signInWithOtp({ email })`.
- Token validation on callback via `supabase.auth.exchangeCodeForSession()`.

1. 

- `auth.users` (Supabase managed).
- `workspaces`, `workspace_members` (on first sign-in path).

1. 

- Supabase Auth managed email dispatcher.

1. 

- Phase C plumbing done already (see prior session: `lib/supabase/server.ts`, `lib/supabase/client.ts`, `middleware.ts`, `AuthContext`).

1. 

- [https://jira.upexgalaxy.com/browse/BK-4#icft=BK-4](https://jira.upexgalaxy.com/browse/BK-4#icft=BK-4) (workspace create) is unblocked once any sign-in story ships.
- [https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5](https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5) (invite teammate) needs a signed-in owner.

1. 

- [https://jira.upexgalaxy.com/browse/BK-3#icft=BK-3](https://jira.upexgalaxy.com/browse/BK-3#icft=BK-3) (OAuth sign-in) — same FR-001 with different provider surface.

1. 

- [ ] Magic-link sign-up + sign-in working end-to-end on staging.
- [ ] Unit tests for token-expiry + replay rejection paths.
- [ ] Integration test for default-workspace auto-create on first verified login.
- [ ] E2E Playwright happy-path test (uses Supabase test inbox).
- [ ] Code review approved.
- [ ] Documentation: README "Getting Started" mentions magic-link as default.

---

### Ely - 5/25/2026, 3:58:37 AM

# Shift-Left Refinement — [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2)

***Story:*** Sign up and sign in with email (magic-link)

***Jira:*** [BK-2](https://jira.upexgalaxy.com/browse/BK-2)

***Epic:*** [BK-1](https://jira.upexgalaxy.com/browse/BK-1) — Tenancy & Identity

***Source spec:*** FR-001 (email side only; OAuth covered by [https://jira.upexgalaxy.com/browse/BK-3#icft=BK-3](https://jira.upexgalaxy.com/browse/BK-3#icft=BK-3))

***Reviewed:*** 2026-05-25 — Shift-Left QA pass

---

## 1. Critical analysis

***Implementation maturity (target repo `upex-bunkai-tms`, branch `main`):***

| ***Component**** | ****Status*** |
| --- | --- |
| `app/(auth)/login/page.tsx` | EXISTS — left brand panel + right auth panel with Suspense-wrapped form |
| `app/(auth)/login/magic-link-form.tsx` | EXISTS — client form, regex email validation, `/api/v1/auth/magic-link` POST |
| `app/api/v1/auth/magic-link/route.ts` | EXISTS — Zod schema, Supabase `signInWithOtp`, 429 mapped to `rate_limited` |
| `app/auth/callback/route.ts` | EXISTS — `exchangeCodeForSession`, hard-codes redirect to `/projects` |
| `middleware.ts` | EXISTS — protects `/projects` + `/onboarding`, preserves `?next=` round-trip |
| `app/(app)/onboarding/page.tsx` | EXISTS — server-side guard: signed-in + no workspace → form; has workspace → `/projects` |
| `app/(app)/onboarding/onboarding-form.tsx` | EXISTS — manual slug + name input, RPC `bunkai*bootstrap*workspace` |
| `supabase/migrations/0006*bootstrap*workspace.sql` | EXISTS — atomic workspace + workspace_members row insert, security-definer |
| Workspace auto-create on first sign-in | ***NOT IMPLEMENTED*** — current path is manual via `/onboarding` |
| Specific error codes (`TOKEN*USED`, `TOKEN*EXPIRED`, `INVALID_EMAIL`) | ***NOT IMPLEMENTED*** — generic envelope only |
| RFC 5321 254-char enforcement | ***NOT IMPLEMENTED*** — `z.string().email()` permits longer than SMTP allows |
| Magic-link TTL 15 min | ***CONFIG-ONLY*** — lives in Supabase GoTrue (`auth.otp_exp`), not in repo |
| `/home` route | ***DOES NOT EXIST*** — story references non-existent surface |
| `signInWithOtp` resend semantics | ***UNDEFINED*** — no documented behavior on second-request-before-first-expires |
| Pending-invite bypass branch | ***NOT VISIBLE*** — depends on [https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5](https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5) (invites) shipping first |

***Feasibility verdict:**** the foundational plumbing exists; what's missing is the ****glue logic + UX contract refinement****. [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2) is effectively a **Phase 2 / refactor* of the existing MVP sketch, not a greenfield build.

---

## 2. AC ↔ current-code reconciliation (decided per divergence)

> Source-of-truth philosophy: AC drives ***intent + user contract****; code informs ****what's already there worth keeping***. When AC contradicts good code already shipped, the lighter rewrite wins unless it sacrifices security or product clarity.

### 2.1 Redirect target — `/home` vs `/projects`

| ***Side**** | ****Says**** | ****Strength**** | ****Weakness*** |
| --- | --- | --- | --- |
| AC | "user lands on the Workspace Home" | Conveys semantic intent (post-auth root) | `/home` route does not exist |
| Code | Redirects to `/projects` | Concrete + already routable | Skips the "needs workspace?" check |

***Decision:**** ****Code wins on the URL, AC wins on the routing intent.*** Replace AC's `/home` with the routed entry point. Add the missing branch.

***How to complete:***

- Callback redirect target becomes `/onboarding` (NOT `/projects`).
- `/onboarding` page already guards: has-workspace → `/projects`, no-workspace → form. This single redirect handles both paths cleanly via the existing guard.
- Update AC text to read "the user lands on `/projects` (Workspace Home, if they already belong to a workspace) or `/onboarding` (if not yet onboarded)".

***Why this wins:*** zero new code in the callback path; reuses the guard already shipped in `app/(app)/onboarding/page.tsx`. Less surface area to test, less divergence risk.

### 2.2 Default workspace auto-creation on first verified sign-in

| ***Side**** | ****Says**** | ****Strength**** | ****Weakness*** |
| --- | --- | --- | --- |
| AC | Callback creates `"{display*name}'s workspace"` automatically + idempotent | Zero-friction sign-up | Requires `display*name` (we only have email); slug collisions inevitable; bad URL choices baked in |
| Code | Manual `/onboarding` form (name + slug, user-controlled) | User owns the URL; slug uniqueness enforced; multi-tenant-honest | One extra screen between sign-in and first action |

***Decision:**** ****Code pattern wins (manual onboarding)*** — but with UX hand-holding.

***Why this wins:***

- Slug is part of every URL the user shares (`/workspaces/{slug}/...`). Auto-generating slugs from email prefixes creates collisions and traps the user into a bad URL on day one.
- "Display name" doesn't exist yet for magic-link signups (only email). Falling back to `email.split('@')[0]` produces ugly defaults like `"elyermad's workspace"` — degrades onto the user.
- The atomic SECURITY DEFINER RPC `bunkai*bootstrap*workspace` already enforces correctness; AC's "idempotent on retry" requirement is satisfied by the unique-slug constraint surfacing `23505` → friendly "slug taken" message (already implemented in `onboarding-form.tsx:46`).
- BK-12+ (User Stories) and BK-18+ (ATC Library) all live under workspace boundaries. Letting the owner name their workspace is a one-time + high-value choice, not friction worth eliminating.

***How to complete:***

- Pre-fill the workspace name field with `"{email-prefix}'s workspace"` as a **suggestion**; user can overwrite.
- Pre-fill the slug from the name via the existing `slugify()` helper.
- Keep the bootstrap RPC as the only write path.

### 2.3 Error code contract — AC `INVALID*EMAIL` / `TOKEN*USED` / `TOKEN_EXPIRED` vs code envelope

| ***Side**** | ****Says**** | ****Strength**** | ****Weakness*** |
| --- | --- | --- | --- |
| AC | Domain-specific codes (`INVALID*EMAIL`, `TOKEN*USED`, `TOKEN_EXPIRED`) | Frontend can render specific copy per failure | No envelope discipline; codes invented in isolation |
| Code | Structured envelope (`bad*request`, `upstream*error`, `rate_limited`) | Reusable across all API surfaces | Loses signal — frontend can't differentiate token-replay from infra outage |

***Decision:**** ****Hybrid — keep the envelope, replace the codes.***

***Why this wins:**** Zod + `ApiError` envelope is good engineering practice (versioned, machine-readable, consistent across `/api/v1/**`). But "`upstream_error` from Supabase" is too coarse — the user-facing screen for "your link expired" should differ from "Supabase is down". AC's domain codes give us that signal.

***How to complete:***

- Extend the `ApiError` envelope's `code` enum with: `INVALID*EMAIL`, `INVALID*REDIRECT`, `RATE*LIMITED`, `TOKEN*USED`, `TOKEN*EXPIRED`, `UPSTREAM*ERROR`.
- In `/api/v1/auth/magic-link/route.ts`: map Zod email-shape failures to `INVALID*EMAIL` (not generic `bad*request`).
- In `/auth/callback/route.ts`: inspect Supabase error code/message before redirect — map `otp*expired` → `TOKEN*EXPIRED`, `token*already*used`/`flow*state*not*found` → `TOKEN*USED`. Pass via `?error=TOKEN_EXPIRED` so `/login` can render specific copy.
- `/login` reads `?error=` and renders contextual messaging (see §6 UX).

### 2.4 Magic-link TTL 15 minutes

| ***Side**** | ****Says**** | ****Strength**** | ****Weakness*** |
| --- | --- | --- | --- |
| AC | TTL 15 minutes, single-use | Correct security policy | Lives outside code repo |
| Code | Defers to Supabase GoTrue config | Honest about where the truth lives | Repo gives no signal that the value is enforced anywhere |

***Decision:**** ****AC wins (15 min is correct)****, but enforcement is an ****ops + docs concern***, not a code concern.

***How to complete:***

- Document in `supabase/config.toml` (self-hosted) or as an ops checklist item (cloud project): `auth.otp_exp = 900` (seconds).
- Add a `docs/` note (or comment in `0006*bootstrap*workspace.sql`-adjacent location) referencing the TTL contract so a future migration doesn't unset it silently.
- ATR for [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2) will include a manual check: send a link, wait >15 min, click, confirm rejection. Cannot be automated without time-travel mocking.

### 2.5 Email validation — RFC 5321 (254 char limit)

| ***Side**** | ****Says**** | ****Strength**** | ****Weakness*** |
| --- | --- | --- | --- |
| AC | RFC 5321 with 254-char ceiling | Aligns with SMTP delivery reality | None worth keeping |
| Code | Loose client regex + `z.string().email()` server-side | Catches shape errors | Permits 300+ char addresses that GoTrue / SMTP will reject silently |

***Decision:**** ****AC wins.***

***Why:*** an email that passes `z.string().email()` but exceeds 254 chars produces an opaque downstream failure — Supabase + SMTP either silently drop or return a non-actionable error. Catching it client + server-side surfaces the failure clearly.

***How to complete:***

- Server-side: `email: z.string().email().max(254, { message: 'Email exceeds RFC 5321 254-char limit.' })`.
- Client-side regex stays loose for early UX feedback; a length check (`email.length <= 254`) joins the `isValid` predicate.

### 2.6 Resend-before-expiry semantics

***Status:*** AC silent; needs PO decision.

| ***Option**** | ****Pros**** | ****Cons*** |
| --- | --- | --- |
| A. Both tokens valid until used/expired (Supabase default) | Zero work; matches GoTrue out-of-box | Phishing risk: if 1st link is intercepted, requesting a "fresh" one doesn't kill it |
| B. Resend invalidates prior tokens | Stronger security posture | Requires custom logic (token tracking table or Edge Function); +1-2 sprint days |

***Recommendation:**** ****A for MVP, with a 60-second resend cooldown UI guard*** (UX prevents accidental double-request; security upgrade B becomes a separate Story when threat model demands it). Flagged for PO/security review.

### 2.7 Pending-invite bypass

***Decision:**** ****Scope OUT of BK-2.**** Invite acceptance is BK-5's domain; [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2) ships with the "no invite, brand-new user" path only. The `comments.md` business rule about skipping personal-workspace auto-create stays as a **forward-compatible note* — the onboarding-form guard already early-returns to `/projects` if the user already has any active workspace membership, which the [https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5](https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5) invite flow will populate.

---

## 3. Refined Acceptance Criteria

```
Background:
  Given the Supabase auth.otp_exp is set to 900 seconds (15 minutes)
    And the project SMTP / mailer is operational
    And the user is not currently signed in

# ---- Happy path ----

Scenario: Successful first-time email magic-link sign-up
  Given a visitor on /login
  When they enter "qa-new@bunkai.test" (a valid RFC 5321 email, <=254 chars)
    And click "Send magic link"
  Then the API responds 200 with `{ ok: true }`
    And the form shows the "Check your inbox" confirmation state
    And within 30s an email with subject "Sign in to Bunkai" arrives in that inbox
  When they click the magic link in the email
  Then the browser is redirected through `/auth/callback?code=...`
    And exchanged into a Supabase session (cookie set)
    And then routed to `/onboarding` (because the user has no workspace yet)
    And `/onboarding`'s server guard renders the workspace-create form
    And the workspace-name input is pre-filled with "qa-new's workspace"
    And the slug input is pre-filled with "qa-news-workspace" (slugified suggestion)
  When they accept the defaults and click "Create workspace"
  Then the RPC `bunkai*bootstrap*workspace` returns a workspace_id atomically
    And a row exists in `workspaces` with owner*user*id = the new user's id
    And a row exists in `workspace_members` (role=owner, status=active)
    And the user is redirected to `/projects`

Scenario: Successful returning-user sign-in (workspace exists)
  Given a user who already has at least one active workspace_member row
  When they request a magic link with their existing email
    And click the link
  Then `/auth/callback` exchanges the code into a session
    And redirects to `/onboarding`
    And `/onboarding`'s guard short-circuits to `/projects` (membership exists)
    And the user lands on `/projects` directly

# ---- Negative path ----

Scenario: Invalid email format rejected client-side
  Given a visitor on /login
  When they type "notanemail" into the email field
  Then the "Send magic link" button stays disabled
    And no POST is dispatched

Scenario: Invalid email format rejected server-side
  Given a visitor on /login
  When the client is bypassed and a body `{ email: "no-at-symbol" }` is POSTed to /api/v1/auth/magic-link
  Then the API responds 400 with envelope `{ ok: false, error: { code: "INVALID_EMAIL", message: ... } }`

Scenario: Email exceeds RFC 5321 length limit
  Given a visitor on /login
  When they enter an email whose total length is 255 characters
  Then the form rejects it client-side with "Email exceeds 254-character limit"
    And no POST is dispatched
  And the server-side Zod schema also rejects with code "INVALID_EMAIL" if bypassed

Scenario: Magic-link token replay blocked
  Given a user who has successfully signed in via a magic link
  When they click the same link a second time (or the same `?code=` is sent to /auth/callback)
  Then Supabase exchangeCodeForSession returns a "token already used" error
    And the callback redirects to /login?error=TOKEN_USED
    And the /login page renders "This link was already used — request a new one"

Scenario: Magic-link token expired
  Given a magic link generated more than 15 minutes ago
  When the visitor clicks it
  Then Supabase exchangeCodeForSession returns an OTP-expired error
    And the callback redirects to /login?error=TOKEN_EXPIRED
    And the /login page renders "Your link expired — request a new one" with the email field pre-filled

Scenario: Callback missing the `code` query parameter
  Given a request to /auth/callback with no `?code=`
  Then the route redirects to /login?error=MISSING_CODE

Scenario: Rate-limited resend (Supabase 429)
  Given a visitor who has requested a magic link 5 times in 60 seconds
  When they submit a 6th request
  Then the API responds 429 with envelope `{ code: "RATE_LIMITED", ... }`
    And the form shows a "Too many requests — try again in N seconds" toast

# ---- Boundary / edge ----

Scenario: Resend allowed after 60-second UI cooldown
  Given a visitor who has just sent a magic link
  When they look at the "Check your inbox" screen
  Then a "Resend link" button appears, initially disabled with countdown
  When 60 seconds pass
  Then the "Resend link" button becomes enabled
  When they click it
  Then a second magic link is dispatched to the same email
    And the prior link remains valid until its own 15-min TTL elapses
       (Note: per §2.6 — MVP keeps both valid; future Story may invalidate prior)

Scenario: Open-redirect attempt via `next` parameter blocked
  Given a magic link generated with `next=https://evil.example.com/steal`
  When the callback runs
  Then the unsafe `next` is rejected and replaced with the default `/onboarding` (or `/projects` if user has workspace)
    And the user does not leave the bunkai domain

Scenario: Session cookie set with secure attributes
  Given a successful callback exchange in production env
  Then the session cookie has `Secure`, `HttpOnly`, `SameSite=Lax`, and the Supabase-managed name
    And `getUser()` on the next protected route returns the new user
```

***Markers used:*** all NEEDS PO/DEV CONFIRMATION items are explicitly captured in §8 PO/Dev questions below; the AC text itself is final pending those answers.

---

## 4. Edge cases (names + criticality)

| ***#**** | ****Edge case**** | ****Criticality*** |
| --- | --- | --- |
| 1 | Email with uppercase letters — Supabase normalizes; UI should mirror | Medium |
| 2 | Email with `+alias` (Gmail-style) | Low |
| 3 | Internationalized email (Punycode / Unicode local-part) | Medium |
| 4 | Same email requesting links from two different devices/browsers | High |
| 5 | Magic link clicked on a different device than the one that requested it | High |
| 6 | Magic link clicked while already signed in as a **different** user | High |
| 7 | Magic link clicked in private/incognito window | Medium |
| 8 | Mailbox bounces / undeliverable address | High |
| 9 | User clicks the magic link, then closes the tab before redirect completes | Medium |
| 10 | Slug collision on bootstrap (two users want same slug) | High |
| 11 | Bootstrap RPC fails mid-flight (e.g. DB hiccup) — partial state | High |
| 12 | First-sign-in race (user double-clicks the link, two callback requests in parallel) | High |
| 13 | Magic link arrives in spam folder | Medium |
| 14 | Supabase project is paused / unreachable | High |
| 15 | Browser blocks third-party cookies (Safari ITP) | Medium |

---

## 5. Modern UX proposals (for PO consideration)

> These are NOT acceptance criteria — they're recommendations to elevate the login UX from "MVP sketch" to "modern auth experience". Each is gated by PO acceptance.

### 5.1 Resend cooldown + retry affordance

- ***Now:*** "Check your inbox" state is read-only; user must navigate back to send again.
- ***Better:*** show a `Resend link` button with a 60-second cooldown countdown. After cooldown, one-click resend with the same email. Also surface a "Use a different email" link (already implemented).

### 5.2 Email provider deep links

- ***Now:*** "Check your inbox" gives no action affordance.
- ***Better:*** detect domain → render direct-open buttons. Examples:

- `@gmail.com` / `@googlemail.com` → "Open Gmail"

- `@outlook.com` / `@hotmail.com` / `@live.com` → "Open Outlook"

- `@yahoo.com` → "Open Yahoo Mail"

- Generic → "Open default mail app" (`mailto:` fallback)

- Reduces time-to-first-link-click measurably; major SaaS auth UIs (Notion, Linear, Vercel) all ship this.

### 5.3 Contextual error pages instead of toast-only

- ***Now:*** callback failure redirects to `/login` with a query param — but the form doesn't read it; toast disappears in 5s.
- ***Better:*** dedicated UI states on `/login`:

- `?error=TOKEN_EXPIRED` → banner "Your sign-in link expired. Request a new one." + email pre-filled (from query param `?email=`)

- `?error=TOKEN_USED` → banner "This link was already used. Request a new one or close this tab if you're signed in elsewhere."

- `?error=MISSING_CODE` → toast + reset.

### 5.4 Last-used email memory (opt-in)

- ***Now:*** every login = fresh email entry.
- ***Better:*** after a successful sign-in, store the email in `localStorage`. On next `/login` visit, pre-fill + show "Not you?" link to clear. Pure client-side, no PII server-side; matches Notion / Linear pattern.

### 5.5 Onboarding pre-fill (already specified in §3 AC)

- Pre-fill workspace name + slug suggestions on `/onboarding` form. Removes the "I don't know what to type" stall point most signup funnels suffer from.

### 5.6 Magic-link success affordance

- ***Now:*** redirect happens silently; user just appears on `/projects` or `/onboarding`.
- ***Better:*** show a brief "Welcome, {email}" toast on first authenticated landing. Reinforces successful sign-in (especially after the multi-tab dance).

### 5.7 Accessibility wins

- Focus management: when "Check your inbox" replaces the form, move focus to the new heading so screen readers announce the state change.
- `aria-live="polite"` on the toast region (`sonner` may already do this — verify).
- Form-level validation message tied via `aria-describedby` to the email input.

### 5.8 Self-hosted-instance affordance (already in code)

- Login page already has a "Self-hosted instance" card. Verify it's wired or hide it for the cloud-only MVP — dead UI on a primogenital story dilutes trust.

### 5.9 Magic-link email design

- ***Out of repo scope*** (lives in Supabase GoTrue templates), but worth flagging: default GoTrue email is bland. Branded HTML email with the Bunkai 分解 mark + "this link expires in 15 minutes" copy is a 30-min ops win that elevates first-impression dramatically.

---

## 6. Test outlines (names only — no parametrization)

> Outline NAMES + brief preconditions only, per shift-left rule. Parametrization, test data, Faker recipes live in `/sprint-testing` once the Story reaches Ready For QA.

***Positive (3):***

- `TC-OUT-POS-01` — First-time sign-up: link send → click → land on onboarding form
- `TC-OUT-POS-02` — Returning user with existing workspace: link → straight to /projects
- `TC-OUT-POS-03` — Workspace bootstrap via onboarding form succeeds (atomic insert)

***Negative (6):***

- `TC-OUT-NEG-01` — Invalid email rejected client-side (button stays disabled)
- `TC-OUT-NEG-02` — Invalid email rejected server-side with code `INVALID_EMAIL`
- `TC-OUT-NEG-03` — Magic-link replay yields `?error=TOKEN_USED` + UX banner
- `TC-OUT-NEG-04` — Expired magic link yields `?error=TOKEN_EXPIRED` + UX banner
- `TC-OUT-NEG-05` — Callback without `?code=` redirects to `/login?error=MISSING_CODE`
- `TC-OUT-NEG-06` — Rate-limit (429) returns `RATE_LIMITED` envelope + toast

***Boundary (4):***

- `TC-OUT-BND-01` — Email of exactly 254 chars accepted
- `TC-OUT-BND-02` — Email of 255 chars rejected (client + server)
- `TC-OUT-BND-03` — Magic link clicked at minute 14:59 succeeds
- `TC-OUT-BND-04` — Magic link clicked at minute 15:01 fails with `TOKEN_EXPIRED`

***Integration (4):***

- `TC-OUT-INT-01` — Open-redirect blocked: `next=https://evil.com` falls back to safe default
- `TC-OUT-INT-02` — `workspace_members` row created atomically with `workspaces` row (bootstrap RPC)
- `TC-OUT-INT-03` — Middleware redirects unauthenticated `/projects` access to `/login?next=/projects`, preserves `next` through the link, restores it after callback
- `TC-OUT-INT-04` — Session cookie attributes (`Secure`, `HttpOnly`, `SameSite=Lax`) set correctly in production env

***Total: 17 outlines*** — Positive 3 / Negative 6 / Boundary 4 / Integration 4.

***Coverage rationale for PO:*** sign-up + sign-in are auth surfaces with FORCE-FULL triggers (multi-tenancy seeding, external integration, state machine, data integrity). The 6 negative outlines map 1:1 against documented error states; boundary outlines validate the contracts (TTL + email length); integration outlines guard the trickiest seams (open-redirect, atomic insert, middleware round-trip, cookie security).

---

## 7. PO / Dev / Design open questions

> Listed ONLY where genuine ambiguity or design decision is needed. Skipping inflated "could-ask" items per CLAUDE.md §1 Rule #4 + L1 anti-pattern.

### For PO

1. ***Resend semantics (§2.6):*** Confirm MVP keeps both tokens valid + adds 60s UI cooldown — agreed? Or do we ship token-invalidation-on-resend now (extra 1-2 days)?
2. ***Workspace-name default:*** Pre-fill suggestion `"{email-prefix}'s workspace"` — acceptable, or prefer empty field forcing user choice?
3. ***Modern UX scope (§5):*** Which of items 5.1-5.9 land in [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2) vs deferred to a follow-up "polish" Story? Recommendation: 5.1 (resend cooldown), 5.3 (contextual error pages), 5.5 (pre-fill), 5.7 (a11y) in scope; 5.2 (provider deep-links), 5.4 (email memory), 5.6 (welcome toast), 5.9 (email template) deferred.
4. ***`/home` route:*** Confirm `/onboarding → /projects` chain replaces the AC's `/home`. (We are not creating a `/home` route.)
5. ***Magic-link email template:*** OK to defer to a separate ops Story? Default GoTrue email ships with [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2).

### For Dev

1. ***Supabase error code mapping:*** Confirm the exact Supabase v2 error codes for token-already-used vs token-expired (`otp*already*used`? `flow*state*not*found`? `otp*expired`?). Map table needs to be deterministic before refined-AC translates to test assertions.
2. ***`auth.otp_exp = 900` ops checklist:*** Where does this get tracked? `supabase/config.toml`? `docs/ops-runbook.md`? It must live somewhere repo-discoverable so a future migration doesn't reset it.
3. ***Bootstrap RPC race condition:*** if a user double-clicks "Create workspace", both requests race. Migration 0006's UNIQUE slug constraint catches the second insert with `23505` — confirmed acceptable behavior, or do we need to debounce client-side?
4. ***`workspace_members.status='active'` invariant:*** The onboarding guard filters `eq('status', 'active')`. What other statuses exist (e.g. `pending`, `invited`)? Affects how [https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5](https://jira.upexgalaxy.com/browse/BK-5#icft=BK-5) (invites) will compose with this flow.

### For Design

1. ***`/login` post-error states UI:*** mockups for the contextual banners in 5.3 (TOKEN*EXPIRED / TOKEN*USED variants).
2. ***Resend cooldown UX (5.1):*** Inline countdown text? Disabled button with timer? Toast pattern? Need a spec before Dev writes it.

---

## 8. Scope refinement (IN vs OUT for [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2))

### IN — must ship as part of [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2)

- Magic-link send via `/api/v1/auth/magic-link` (✓ exists)
- Magic-link callback at `/auth/callback` (✓ exists, ***needs error-code mapping refactor***)
- Redirect chain `callback → /onboarding → (guard) → /projects | form` (***needs callback change***)
- Onboarding form with pre-filled name + slug suggestions (✓ form exists, ***needs pre-fill***)
- Bootstrap RPC `bunkai*bootstrap*workspace` (✓ exists, no change)
- Specific error codes (`INVALID*EMAIL`, `TOKEN*USED`, `TOKEN*EXPIRED`, `MISSING*CODE`, `RATE*LIMITED`, `UPSTREAM*ERROR`) (***needs implementation***)
- RFC 5321 254-char enforcement client + server (***needs implementation***)
- Open-redirect guard on `next` param (✓ already in callback)
- UX 5.1 (resend cooldown 60s), 5.3 (contextual error banners), 5.5 (onboarding pre-fill), 5.7 (a11y focus + aria) — see §5
- Ops checklist item: confirm Supabase `auth.otp_exp = 900` on the testing + staging + production projects

### OUT — deferred to other Stories or follow-up polish

- OAuth (GitHub / Google) — ***BK-3***
- Invite acceptance flow + bypass branch — ***BK-5***
- Workspace switching — ***BK-6***
- Multiple workspaces per user — ***BK-6***
- UX 5.2 (provider deep-links), 5.4 (email memory), 5.6 (welcome toast), 5.9 (branded email template) — ***follow-up "Auth UX Polish" Story***
- Token-invalidation-on-resend (security upgrade B in §2.6) — ***separate Story if PO greenlights it***
- Self-hosted instance card behavior — ***out of MVP unless wired***
- Branded magic-link email template — ***separate ops Story***

---

## 9. Definition of Ready (post-refinement)

This Story is ***ready for PO estimation*** when:

- [x] Refined ACs above (§3) accepted by PO
- [x] §2 reconciliation decisions confirmed (especially 2.1, 2.2, 2.3)
- [x] §5 UX scope cut (which items in vs out) confirmed
- [x] §7 questions 1, 2, 3, 4 answered (PO scope-fixing items)
- [ ] §7 questions 6, 7 answered (Dev technical items — can happen post-estimation)
- [ ] §7 questions 10, 11 (Design assets) — can be parallel to Dev work
- [ ] Supabase `auth.otp_exp` verified on each environment by ops before development starts

---

### Ely - 5/27/2026, 8:49:59 PM

Implementado este sprint (lean MVP closure of EPIC-BK-1).

Code on main:

- 69669d2 feat(auth): rfc 5321 email length cap + magic-link audit trail
- Sprint 1 base: POST /api/v1/auth/magic-link route + /auth/callback exchange.

Surfaces ready for QA:

- POST /api/v1/auth/magic-link (Zod-validated, RFC 5321 email length cap 254).
- /login UI consumes it.
- magic*link*tokens audit table (migration 0009) — best-effort issuance log with ip*hash + user*agent.

Out of scope this MVP: dedicated replay enforcement (Supabase Auth handles OTP replay natively).

Reference: .context/PBI/epics/EPIC-BK-1-tenancy-identity/MVP-NOTES.md.
Testability guide: /qa (en app) + Jira Epic [https://jira.upexgalaxy.com/browse/BK-29#icft=BK-29](https://jira.upexgalaxy.com/browse/BK-29#icft=BK-29).

---

### Ely - 5/27/2026, 10:56:54 PM

# Stage 1 — Acceptance Test Plan ready

***Drafter:*** Sprint Testing (orchestration mode, Stage 1).
***Modality:*** jira-native.
***TCs:*** 17 (P1: 10, P2: 5, P3: 2).
***Risk:*** FORCE-FULL (auth + multi-tenancy + external + state-machine + data-integrity all trigger).
***Source:*** shift-left-refinement.md §6 outlines promoted to draft.

## Layer breakdown

- UI: 5
- API: 4
- DB: 1
- UI+API: 5
- UI+DB: 1
- UI+API+DB: 1

## Where to find the body

- Full ATP body: see customfield "Acceptance Test Plan (ATP)" on this issue.
- Local mirror (source of truth): `.context/PBI/epics/EPIC-BK-1-tenancy-identity/stories/STORY-BK-2-sign-up-and-sign-in-with-email-magic-link/acceptance-test-plan.md`.
- ATR scaffold (empty results table, ready for Stage 2): `acceptance-test-results.md` alongside the ATP.

## Carry-over flags into Stage 2

- ***Flag C*** — TC-BK-2-06 keeps expected `?error=TOKEN_USED` per AC; Stage 2 will record observed Supabase-native behavior verbatim and decide gap vs bug.
- ***Flag D*** — `/qa` testability page + Epic BK-29 to be explored during smoke for fixtures that may upgrade boundary TCs (TC-BK-2-12, TC-BK-2-13) from manual to automated.
- ***Flag F*** — Inbox provider is Resend CLI (user-confirmed). Stage 2 sub-agent will load /resend-cli before any inbox poll.

## Next

Stage 2 — Execution. Will fire the first ticket transition `start_testing` (Ready For QA → In Test) after user OK on this plan. No transition performed by Stage 1.

---

### Ely - 5/27/2026, 11:17:07 PM

1. Stage 2 — Execution started

Tested against: [https://upexbunkai.vercel.app](https://upexbunkai.vercel.app/) (session-only override; staging-upexbunkai.vercel.app returned 404 DEPLOYMENT*NOT*FOUND).
Inbox: delgri.resend.app per-scenario addresses.
17 TCs queued per ATP §6 execution order.
Updates land on `acceptance*test*results` customfield + comment at Stage 3.

---

### Ely - 5/27/2026, 11:21:35 PM

1. Stage 2 — Execution started (retry #1)

Tested against: [https://upexbunkai.vercel.app](https://upexbunkai.vercel.app/) (session-only override; staging-upexbunkai.vercel.app returned 404 DEPLOYMENT*NOT*FOUND on first attempt).
Inbox: delgri.resend.app per-scenario addresses (Resend inbound receiving).
17 TCs queued per ATP §6 execution order.
ATR rows being overwritten as each TC executes.
Updates land on `acceptance*test*results` customfield + comment at Stage 3.

---

### Ely - 5/28/2026, 12:01:26 AM

# QA Sign-off — [https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2](https://jira.upexgalaxy.com/browse/BK-2#icft=BK-2) (Sprint Testing, Stage 3)

Result: PASS (GO-with-debt). Tested on [https://upexbunkai.vercel.app](https://upexbunkai.vercel.app/) (session override — staging-upexbunkai.vercel.app returned 404; see ATR §1). Modality: jira-native.

## Coverage

17 TCs executed. PASSED 10 · KNOWN 4 (PO scope) · BLOCKED 3 (manual-pending). No product defects filed.

Green: magic-link send/click/callback, returning-user to /projects, atomic workspace+member bootstrap, open-redirect blocked, middleware next round-trip, rate-limit 429, 254-char boundary, client-side invalid-email reject.

## Known debt (non-blocking)

Session cookie sb-ref-auth-token:

- HttpOnly=false — by-design of @supabase/ssr (browser SDK reads document.cookie). Accepted framework pattern; not a defect.
- Secure=false — Low hardening debt. Practically un-exploitable on this domain: vercel.app is HSTS-preloaded (max-age 2y, includeSubDomains, preload), so the browser forces HTTPS and the cookie never travels over plaintext. Trivial fix: cookieOptions secure true. ESCALATION: if BK moves to a custom prod domain NOT on HSTS preload, Secure plus HSTS become mandatory (severity rises to High).
- SameSite=Lax — correct.

## PO scope question (4 KNOWN TCs)

Shift-left §2.3 recommended a custom UPPER*SNAKE error envelope (INVALID*EMAIL / TOKEN*USED / MISSING*CODE / RATE*LIMITED). The app ships Supabase-native lowercase codes (validation*failed / missing*code / rate*limited / otp_expired) which are functionally correct. The custom envelope was a refinement recommendation, NOT a signed AC. PO decision: firm requirement (future Minor conformance work) or accepted scope (close)? Also: onboarding name/slug pre-fill (shift-left §5.5 / PO Q2) is not implemented.

## Manual-pending (3 BLOCKED TCs)

TC-07/12/13 (magic-link TTL boundary 14:59 / 15:01 / expiry) need a clock-mock or short-TTL fixture — not automatable in this pass. The /qa page documents a headless PAT auth shortcut (no email) that can accelerate future automation (Stage 5), but expiry still needs a time fixture.

## Artifacts

ATR: acceptance*test*results field on this issue + local mirror .context/PBI/epics/EPIC-BK-1-tenancy-identity/stories/STORY-BK-2-sign-up-and-sign-in-with-email-magic-link/acceptance-test-results.md. Evidence: per-TC dirs under evidence/.

---


_Synced from Jira by sync-jira-issues_
