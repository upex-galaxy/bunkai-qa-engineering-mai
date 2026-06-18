# Test Session Memory — BK-15

> Shared payload across the 4 sub-agent dispatches. DO NOT Jira-mirror.

---

## Environment

| Key | Value |
|-----|-------|
| Active env | staging |
| WEB_URL | https://staging-upexbunkai.vercel.app |
| API_URL | https://staging-upexbunkai.vercel.app/api |
| DB_MCP | staging-dbhub |
| API_MCP | staging-openapi |

## TMS Modality

**Jira-native** (no Xray). ATP/ATR written to Story custom fields or structured comment fallback.

## Story Identity

| Field | Value |
|-------|-------|
| Key | BK-15 |
| Title | TMS-AC \| Manage criteria under a user story |
| Epic | BK-12 (User Stories & Acceptance Criteria) |
| Status | Ready For QA |
| Priority | Medium |
| Story Points | 3 |
| Shift-Left | Reviewed (2026-06-09) — SHORT-CIRCUIT applies |

## Endpoints (as shipped — from Dev comment 6/5)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/v1/user-stories/{id}/acceptance-criteria` | Create AC (optional position, else tail) |
| GET | `/api/v1/user-stories/{id}/acceptance-criteria` | List active ACs ordered by position |
| GET | `/api/v1/acceptance-criteria/{id}` | Read one active AC |
| PATCH | `/api/v1/acceptance-criteria/{id}` | Edit title/detail or reorder (position) |
| DELETE | `/api/v1/acceptance-criteria/{id}` | Soft-archive |
| PATCH | `/api/v1/user-stories/{id}` | Ready-to-test gate (409 when zero ACs) |

## Critical Behaviors (confirmed by Dev 6/5)

- Auto-revert story to draft on last AC removal: **CONFIRMED**
- Reorder mechanism: **up/down arrows** (no drag-drop)
- Auth: workspace outsider → **404**; in-workspace read-only → **403** on writes
- Soft-archive: `archived_at` column (WHERE archived_at IS NULL = active)
- 409 code: `ac_required_for_ready_to_test`
- 422 code: `validation_failed` with `title_too_short` reason
- XSS: Markdown detail sanitized on save (BK-16 path)
- Ready-to-test gate: race-safe (serialized at DB level)

## Stage State

| Stage | Status | Artifacts |
|-------|--------|-----------|
| Session Start | ✅ COMPLETE | plan.md, progress.md, context.md, test-session-memory.md |
| Stage 1 — Planning | ✅ COMPLETE | acceptance-test-plan.md (pointer stub); full ATP in Jira comments 11650 (Part 1) + 11651 (Part 2) |
| Stage 2 — Execution | ✅ COMPLETE | Stage 2 Execution Report (see below); evidence/ dir populated |
| Stage 3 — Reporting | ✅ COMPLETE | ATR → customfield_10147 (REST PUT HTTP 204); acceptance-test-results.md materialized; QA sign-off comment posted on BK-15; ticket transitioned Ready For QA → In Test → QA Approved |

## ATP Notes

- **Field**: `customfield_10067` — contains pointer stub (field `CONTENT_LIMIT_EXCEEDED` for full 36-TC ADF body)
- **Full ATP**: 2 Jira comments on BK-15 — comment 11650 (Part 1: TC-01 to TC-18) + comment 11651 (Part 2: TC-19 to TC-36)
- **Label added**: `shift-left-reviewed` on BK-15
- **36 TCs total**: 9 Positive, 10 Negative, 7 Boundary, 5 Integration, 5 API
- **Corrections applied**: E5/E6 outsider auth changed 403 → 404 per Dev comment 2026-06-05
- **NEEDS_CONFIRMATION**: TC-07, TC-09, TC-14, TC-15, TC-19, TC-26, TC-30 (blocked BK-18), TC-34

## Bugs Found

### BUG-1: Description byte limit is 50,000 bytes (decimal KB), not 51,200 bytes (binary KiB)
- **Severity**: Medium | **Blocking**: NO
- **TC**: TC-25 (boundary confirmation)
- **Expected**: 51,200 bytes (50 KiB = 50 × 1024) should be accepted as "50 KB"
- **Actual**: 50,001 bytes is rejected; 50,000 bytes is the actual max accepted
- **Root cause**: Implementation uses `50 * 1000 = 50,000` (decimal KB) instead of `50 * 1024 = 51,200` (binary KiB)
- **Error message shown**: "Detail must be at most 50 KB." (message says KB but enforces 50,000 bytes)
- **Impact**: Files/content between 50,001–51,200 bytes (1,200-byte gap) are incorrectly rejected

### BUG-2: TC-19 re-archive returns 409 instead of expected 404
- **Severity**: Low | **Blocking**: NO  
- **TC**: TC-19 (NEEDS_CONFIRMATION scenario)
- **Expected** (from shift-left refinement): 404 not_found when archiving already-archived AC
- **Actual**: 409 conflict with `{"reason":"already_archived"}` (distinct from P0002/not_found path)
- **Note**: This may be intentional — the implementation explicitly detects the already_archived state and returns a descriptive 409 rather than a generic 404. NEEDS PO/Dev confirmation on which is the intended behavior.

## Stage 2 TC Execution Summary

| Result | Count |
|--------|-------|
| PASSED | 28 |
| FAILED | 1 |
| OBSERVATION | 4 |
| NEEDS_CONFIRMATION | 3 |
| SKIPPED | 1 |
| **Total** | **37** |

*Note: TC count is 36 per ATP but TC-19 splits into two distinct observations (one OBSERVATION + one NEEDS_CONFIRMATION), effective 37 result rows*

## Auth Note (Stage 2)

- `STAGING_USER_PASSWORD` in `.env` is blank — staging app uses magic-link only for normal users
- Used `POST /api/v1/auth/signup` (headless QA endpoint, email_confirm: true) to create `bk15-qa-test-1781812465@bunkai-qa.dev`
- Created workspace, project, module, and all test data programmatically via API
- Created outsider user `bk15-outsider-test@bunkai-qa.dev` for isolation tests
- Invited outsider as `viewer` role to workspace for TC-18 test
- PAT used: `bk_pat_KHT3xemP6DEl.VRGluI32c4eSWpfcfEb4PMThq4EsIpA` (QA user)

## Evidence Paths

`evidence/` — all screenshots/HAR go here

### Screenshots captured (Stage 2)
- `BK-15-login-page.png` — Login page showing magic-link auth method
- `BK-15-magic-link-sent.png` — Magic link sent confirmation
- `BK-15-smoke-panel-open.png` — SMOKE: AC management panel open with numbered list, up/down arrows, edit/remove buttons, add form
- `BK-15-TC10-gate-blocked-zero-acs-pass.png` — Gate block message "Add at least one acceptance criterion before marking the story ready to test."
- `BK-15-TC06-gate-allowed-with-acs.png` — Gate allowed: Mark ready to test with ACs present
- `BK-15-TC21-edge-arrows-disabled-pass.png` — Move up disabled for pos 1, Move down disabled for last pos
- `BK-15-TC27-ready-to-test-6acs-state.png` — Precondition: ready_to_test story with 6 ACs
- `BK-15-TC27-one-ac-before-archive.png` — Precondition: 1 active AC remaining before archive
- `BK-15-TC27-auto-revert-to-draft-pass.png` — Auto-revert: story badge removed, panel shows Draft toggle
