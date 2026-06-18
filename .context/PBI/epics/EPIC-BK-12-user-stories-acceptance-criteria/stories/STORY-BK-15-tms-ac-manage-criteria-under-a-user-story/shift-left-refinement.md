# Shift-Left Refinement: BK-15 — TMS-AC | Manage Acceptance Criteria under a User Story

**Status**: Refined — Awaiting PO Estimation
**Mode**: Shift-Left (retroactive grooming — story is Ready For QA; refinement written pre-execution)
**Refined on**: 2026-06-09
**Refined by**: QA — Shift-Left batch session
**Modality**: Jira-native (no Xray)

---

## Phase 1 — Critical Analysis

### Business context

- **Primary persona affected**: Senior QA Engineer (Elena) — the principal author of User Stories and their Acceptance Criteria inside the TMS. Writes ACs as testable conditions before sending a Story to QA.
- **Secondary personas**: SDET / AI Agent consuming ACs headlessly via PAT to bind ATCs; Engineering Manager reviewing Story readiness before sprint planning.
- **Business value proposition**: Enforces the "anchoring moat" one level upstream. ACs are the bridge between business requirements and test cases. Without structured AC management under a Story, ATCs cannot be meaningfully anchored — they lose their business rationale. The ready-to-test gate is the operational enforcement that ACs exist before QA begins.
- **KPI(s) influenced**: Story readiness quality (% of Stories with ≥1 AC before entering QA); ATC orphan rate (ATCs with zero AC bindings — this feature directly reduces that risk upstream).
- **User journey position**: Directly follows BK-14 (User Story creation). Elena creates a Story, immediately adds ACs, then marks it ready to test. ACs must exist before BK-18 (ATC authoring) can anchor test cases to them.

### Technical context

- **Frontend**: AC management panel under the selected User Story — add form (title + optional Markdown detail with 50 KB overCap gate), numbered list with per-AC edit / remove / up-down arrow reorder buttons. User-story status badge and Mark-ready-to-test / Back-to-draft toggle. Mirrors `create-module` and `user-story` form styling + Sidebar action patterns (Slice 5).
- **Backend**:
  - `POST /api/v1/user-stories/{id}/acceptance-criteria` — creates via `bunkai_insert_acceptance_criterion` RPC
  - `GET /api/v1/user-stories/{id}/acceptance-criteria` — lists active ACs ordered by position
  - `GET /api/v1/acceptance-criteria/{id}` — reads one active AC
  - `PATCH /api/v1/acceptance-criteria/{id}` — updates title/description (RLS direct) or position (via `bunkai_move_acceptance_criterion`)
  - `DELETE /api/v1/acceptance-criteria/{id}` — soft-archives via `bunkai_archive_acceptance_criterion`
  - `PATCH /api/v1/user-stories/{id}` — extended with `status` field; ready-to-test gate counts active ACs before transition
- **DB entities**: `acceptance_criteria` (ordered, soft-archiveable), `user_stories` (new `status` column: `draft | ready_to_test`)
- **DB primitives**: Three `SECURITY DEFINER` plpgsql RPCs (`bunkai_insert_acceptance_criterion`, `bunkai_move_acceptance_criterion`, `bunkai_archive_acceptance_criterion`); partial unique index on `(user_story_id, position) WHERE archived_at IS NULL`; negative-parking collision trick for rebalancing.
- **Validation**: `lib/acceptance-criteria/validation.ts` — `criterionTitleError` (min 3, max 200 chars), `MAX_AC_DESCRIPTION_BYTES` = 50 KB. Error helpers: `mapCriterionRpcError` (42501 → 403, P0002 → 404, default → 500).
- **External services**: None. All data is in Supabase Postgres.
- **Integration points specific to this Story**: Ready-to-test gate on the User Story PATCH route; `user_stories.status` new column drives UI toggle and gate behavior; `bunkai_archive_acceptance_criterion` triggers auto-revert of story status to `draft` when last active AC is removed.

### Story complexity

| Axis | Rating | Why |
|------|--------|-----|
| Business logic | High | Ordered-list state machine with partial unique index, negative-parking trick, soft-archive semantics, and a cross-entity status revert on archive |
| Integration | High | Three SECURITY DEFINER RPCs + RLS workspace gate + new `user_stories.status` column extended across a separate route |
| Data validation | Medium | Title bounds (3–200 chars), 50 KB byte cap on description, conditional RPC arg omission (nullable params) |
| UI | Medium | Up/down reorder (no drag-drop), inline edit form, status badge + toggle, 50 KB overCap submit-gate, no `window.prompt` |

**Estimated test effort**: High — 6 original ACs cover the happy path; Phase 2 surfaces 9+ gaps requiring additional scenarios. Expect 25–30 total outlines (including API-level negative and boundary cases). Recommend allocating ≥2 QA days in-sprint.

### Epic-level inheritance

- Parent epic BK-12 (User Stories & Acceptance Criteria) — no `feature-test-plan.md` found in the epic folder at time of refinement.
- Risk inherited from the master test plan: **CRITICAL — Multi-tenant RLS isolation**. Every AC belongs to a workspace via the chain `acceptance_criteria → user_stories → modules → projects → workspaces`. The three SECURITY DEFINER RPCs gate on `bunkai_can_write_workspace`; this chain must be tested for cross-workspace access attempts.
- Risk inherited: **HIGH — Anchoring moat**. ACs are the upstream link. If soft-archive cascades to `atc_acceptance_criteria` without warning (cascade DELETE or deferred constraint), previously anchored ATCs may become orphaned. This is partially in scope for BK-18 but the AC deletion path is authored here.

---

## Phase 2 — Story Quality Analysis

### Ambiguities

| # | Location in Story | Question for PO/Dev | Impact on testing | Suggested clarification |
|---|-------------------|---------------------|-------------------|------------------------|
| 1 | AC6 — "the Story can no longer stay ready to test" | Does "can no longer stay" mean the story status is **automatically reverted to `draft`** by the archive RPC, or is it merely **blocked from being re-marked** ready? Implementation plan §AC6 says "reverts that story to draft" — but AC6 uses weaker language. | Test must know whether to assert `user_stories.status = 'draft'` in the DB or only a UI block. If auto-revert: negative test for removing last AC must verify status change. | Confirm: archive of last active AC auto-reverts story status to `draft` (implementation plan says yes; AC should state it). |
| 2 | Workflow step 3 — "She can drag to reorder" | The implementation plan (Slice 5) explicitly states **up/down arrow reordering, no drag-drop**. The workflow description says "drag to reorder". | If drag-drop is not implemented, testing drag interactions would find a false defect. | Clarify workflow description — should read "up/down arrows" to match implementation. Flag as documentation inconsistency. |
| 3 | AC2 — "preserves the order of the others" | After insertion, are the sibling positions contiguous with no gaps? E.g., if A=1, B=2, C=3 and X is inserted between A and B, is the final state A=1, X=2, B=3, C=4? Or could there be gaps (A=1, X=2, B=4, C=5)? | Must know the exact expected position sequence to write a precise assertion. | Confirm: insertion always re-numbers the active list to contiguous 1..N with no gaps. |
| 4 | Business rule "Removal is soft archive" | After archive, is the AC visible anywhere to the end user (e.g., a "show archived" toggle), or is it hidden entirely from all UI views? | Determines scope: if archived ACs are visible, need tests for the archived list view. If hidden, test that they do not appear in any list/count. | Define post-archive visibility: hidden from UI and excluded from active count, or visible via opt-in toggle? **[NEEDS PO/DEV CONFIRMATION]** |

### Gaps (missing info)

| # | Type | Why critical | What to add | Risk if omitted |
|---|------|--------------|-------------|-----------------|
| 1 | AC | `MAX_AC_DESCRIPTION_BYTES = 50 KB` is implemented (Slice 2 + Slice 3) but zero ACs cover description length validation | Add AC: "A criterion description exceeding 50 KB is rejected with a clear error" | QA will never test this path; a user pasting a large Markdown block will hit a server-side 422 with no documented behavior |
| 2 | AC | **Edit AC** (title and/or description) is explicitly in scope ("Edit and remove individual Acceptance Criteria") and in the implementation (`PATCH /acceptance-criteria/{id}` via RLS update) but **zero ACs describe the edit flow** | Add AC: "An existing criterion title can be updated; edited title obeys the same 3–200 character rule" | The edit path is entirely untested. Title-too-short on edit, title-too-long on edit, and successful edit are all missing. |
| 3 | AC / BR | **Soft-archive semantics** — business rule states "Removal is soft archive" but no AC defines: (a) what the archived state looks like in the DB (`archived_at` stamped), (b) whether archived ACs are counted as "active", (c) whether they are restoreable | Add AC: "Removing a criterion sets `archived_at`; it no longer appears in the active list or active count" | Ambiguity about what "removed" means leads to untested soft-delete behavior; restore path (if any) is completely unknown |
| 4 | AC | **Title maximum** — business rule states 3–200 characters. AC5 tests the minimum (3 chars) but **no AC tests the maximum (200 chars) or exceeding it (201 chars)** | Add AC boundary scenario: "A title of exactly 200 characters is accepted; a title of 201 characters is rejected" | Off-by-one at the upper boundary (200 vs 201) is a classic validation bug; without an AC this boundary is untested |
| 5 | AC | **Default position on append** — AC1 only covers "no criteria yet" (first AC lands as position 1). When a second or third AC is added **without specifying a position**, it should land at the tail. No AC covers this "append to end" case for a Story that already has ACs. | Add AC: "Adding a criterion without specifying a position appends it as the last in the list" | The default-tail behavior is part of `bunkai_insert_acceptance_criterion` (implementation plan §AC1: "defaults position to tail") but is not verifiable without an AC |
| 6 | AC | **Boundary reorder — first AC up / last AC down** — no AC covers what happens when the user clicks "move up" on the first AC or "move down" on the last AC | Add AC: "The up arrow is disabled or no-op for the first criterion; the down arrow is disabled or no-op for the last criterion" **[NEEDS PO/DEV CONFIRMATION]** | Without defined behavior, UI might display an error, silently fail, or wrap around. Cannot test without a spec. |
| 7 | AC | **Multi-tenant workspace isolation** — the three SECURITY DEFINER RPCs gate on `bunkai_can_write_workspace`. No AC covers the 403 path when a user attempts to create/edit/archive an AC in a workspace they do not belong to | Add negative AC: "A user without workspace membership receives a 403 when attempting any AC operation" | Cross-workspace AC access is a CRITICAL risk per the master test plan; without an AC this path is never explicitly tested |
| 8 | AC | **API error codes** — implementation plan defines specific error codes: 409 (`ac_required_for_ready_to_test`), 422 (`title_too_short`, `validation_failed`), 403 (`not_a_member`). No AC specifies these codes or response shapes | Add negative ACs with explicit error code assertions | Without error code assertions, tests can pass on a 422 that happens to return a different `reason` or on a 403 vs 401 confusion |

### Edge cases not in Story

| # | Scenario | Expected behavior (best guess) | Criticality | Action |
|---|----------|-------------------------------|-------------|--------|
| 1 | Add AC to a Story that belongs to a different workspace via direct API call | 403 `not_a_member` from `bunkai_can_write_workspace` RLS gate | Critical | Add to AC (Gap #7 above) |
| 2 | Move AC position to its current position (no-op reorder) | 200 with unchanged list, or no-op with 200 | Medium | Test only — **[NEEDS PO/DEV CONFIRMATION]** |
| 3 | Concurrent reorder: two users move ACs simultaneously | Last-write-wins; negative-parking trick prevents collisions at DB level | High | Test only (concurrency scenario) — **[NEEDS PO/DEV CONFIRMATION]** |
| 4 | Archive AC that is still bound to an active ATC via `atc_acceptance_criteria` | Per business-data-map: `atc_acceptance_criteria` likely cascades on AC delete, potentially orphaning the ATC. The archive is soft (`archived_at`), but if the binding remains on archived ACs, ATC bindings point to invisible ACs. | Critical | **[NEEDS PO/DEV CONFIRMATION]** — define cascade behavior for soft-archive vs ATC binding |
| 5 | Create AC with a title consisting entirely of whitespace (e.g., three spaces) | Should be rejected by title validation (3-char minimum is character count, not visible-char count) — but behavior depends on whether Zod trims before validating | Medium | Test only — **[NEEDS PO/DEV CONFIRMATION]** |
| 6 | Create AC with Unicode multibyte characters in title (e.g., emoji or CJK) that is ≤200 chars by code-point but >200 bytes | Depends on whether the 200-char limit is a character count or byte count in Zod `criterionTitleError`. Implementation plan says "min 3, max 200" — unit of measure unspecified. | Medium | **[NEEDS PO/DEV CONFIRMATION]** |
| 7 | Description exactly at 50 KB boundary (51,200 bytes) | Accepted | Medium | Boundary — test only |
| 8 | Description at 50 KB + 1 byte | Rejected with 422 and a clear message | High | Add to AC (Gap #1 above) |
| 9 | Add AC to a Story marked `ready_to_test` — does the status change? | Story remains `ready_to_test` (adding ACs does not revert; only removing the last one reverts) | High | **[NEEDS PO/DEV CONFIRMATION]** — confirm adding an AC to a ready story does not change its status |
| 10 | Re-archive an already-archived AC | Should return 404 (AC not found in active set, P0002 → 404 per `mapCriterionRpcError`) | Medium | Test only |

### Contradictions

1. **Workflow vs. implementation plan — reorder mechanism**: The workflow description (step 3) says "She can drag to reorder" but the implementation plan (Slice 5) explicitly states "up/down arrow reordering, no drag-drop." These two documents contradict each other. The implementation plan is the binding technical spec; the workflow document appears to carry stale UX intent. This must be resolved before QA execution to avoid testing the wrong interaction model.

2. **AC6 language vs. implementation plan**: AC6 says the Story "can no longer stay ready to test" (passive, suggests a UI block), while the implementation plan says the archive RPC "reverts that story to draft and reports it" (active, automatic status change). These are functionally different: one requires the user to take action after the AC is removed; the other changes the status automatically. The implementation plan is authoritative, but AC6 must be updated to reflect this.

### Testability validation

**Verdict**: Partial

Issues:
- **Edit AC path is entirely unspecified**: No AC, no business rule, no workflow step describes editing an existing AC. The `PATCH /acceptance-criteria/{id}` endpoint exists in the implementation plan but has zero testable specification. This path cannot be tested without inventing expected behavior.
- **Soft-archive visibility is undefined**: "Removal is a soft archive" (business rule) but no document states where or whether archived ACs appear in any view. Without this, the "then" clause of any archive test is incomplete.
- **No error message literals defined**: AC5 says "I see a message that the title must be at least 3 characters" — but no exact copy. The implementation plan provides `title_too_short` as the error code but not the user-facing message. Precision tests require exact strings.
- **Boundary conditions not formally required**: The 200-char title cap and 50 KB description cap are in the implementation plan only, not in any AC. A developer could ship without these bounds and all 6 ACs would pass.

---

## Phase 3 — Refined Acceptance Criteria

### Original AC1 — Add an Acceptance Criterion (first on a Story)

#### Scenario 1.1: Should add first AC at position 1 when story has no criteria (Type: Positive, Priority: Critical)
- **Given**: A workspace member is viewing User Story "Refund a paid order" which has zero Acceptance Criteria and status `draft`
- **When**: They submit a new AC with title "Full refund within 30 days" and no position specified via `POST /api/v1/user-stories/{id}/acceptance-criteria`
- **Then**:
  - UI: the criterion appears as the only item in the ordered list, numbered "1"
  - API: 201 with body containing `{ id, title: "Full refund within 30 days", position: 1, archived_at: null }`
  - DB: one row in `acceptance_criteria` with `user_story_id = {id}`, `position = 1`, `archived_at = null`
  - System state: Story status remains `draft`; active AC count = 1

#### Scenario 1.2: Should append AC at tail when story already has criteria and no position specified (Type: Positive, Priority: High)
- **Given**: User Story has three ACs at positions 1, 2, 3
- **When**: A fourth AC is added via POST without specifying a position
- **Then**:
  - UI: new AC appears as item 4 at the bottom of the list
  - API: 201 with `{ position: 4 }`
  - DB: four rows with positions 1, 2, 3, 4 — no gaps
  - System state: Story status unchanged

---

### Original AC2 — Inserting a criterion preserves and re-numbers the order

#### Scenario 2.1: Should insert AC at specified position and shift existing ACs by +1 (Type: Positive, Priority: High)
- **Given**: A Story with ACs at positions: A=1, B=2, C=3
- **When**: A new AC "X" is inserted at position 2 via `POST /api/v1/user-stories/{id}/acceptance-criteria` with `{ title: "X", position: 2 }`
- **Then**:
  - UI: ordered list reads A(1), X(2), B(3), C(4)
  - API: 201 with `{ position: 2 }`
  - DB: A=1, X=2, B=3, C=4 — contiguous, no gaps, no duplicates in the partial unique index
  - System state: Story status unchanged; active count = 4

#### Scenario 2.2: Should handle insert at position 1 (prepend) correctly (Type: Boundary, Priority: Medium)
- **Given**: A Story with ACs at positions 1, 2, 3
- **When**: A new AC is inserted at position 1
- **Then**: New AC lands at position 1; all previous ACs shift to 2, 3, 4 — contiguous, no gaps

---

### Original AC3 — Reordering re-numbers with no gaps

#### Scenario 3.1: Should move AC to top and re-number remaining ACs contiguously (Type: Positive, Priority: High)
- **Given**: A Story with ACs ordered A(1), B(2), C(3)
- **When**: AC "C" is moved to position 1 via `PATCH /api/v1/acceptance-criteria/{C.id}` with `{ position: 1 }`
- **Then**:
  - UI: list reads C(1), A(2), B(3)
  - API: 200 with updated list or at minimum `{ position: 1 }`
  - DB: C=1, A=2, B=3 — contiguous with no gaps; the partial unique index has no conflicts
  - System state: Story status unchanged

#### Scenario 3.2: Should move AC to bottom and re-number remaining ACs contiguously (Type: Positive, Priority: Medium)
- **Given**: A Story with ACs ordered A(1), B(2), C(3)
- **When**: AC "A" is moved to position 3
- **Then**: UI list reads B(1), C(2), A(3) — contiguous, no gaps

#### Scenario 3.3: Should disable (or no-op) move-up on the first AC and move-down on the last AC (Type: Boundary, Priority: High) **[NEEDS PO/DEV CONFIRMATION]**
- **NEEDS PO/DEV CONFIRMATION**: behavior at list edges not specified. Best guess: up arrow is disabled/hidden for position 1; down arrow is disabled/hidden for the last position.
- **Given**: A Story with ACs at positions 1, 2, 3
- **When**: User clicks "move up" on AC at position 1 (or "move down" on AC at position 3)
- **Then**: No position change occurs; UI shows the button as disabled or absent; no API call is made (or if made, returns the unchanged list)

---

### Original AC4 — Zero ACs blocks ready-to-test transition

#### Scenario 4.1: Should block story status transition to ready_to_test when story has zero active ACs (Type: Negative, Priority: Critical)
- **Given**: A User Story with zero Acceptance Criteria and status `draft`
- **When**: A `PATCH /api/v1/user-stories/{id}` request is sent with `{ status: "ready_to_test" }`
- **Then**:
  - API: 409 with error body `{ error: { code: "ac_required_for_ready_to_test", message: "...", ... } }`
  - DB: `user_stories.status` remains `draft`; no mutation occurs
  - UI: toggle remains in `draft` state; toast displays "at least one Acceptance Criterion is required"

#### Scenario 4.2: Should allow story status transition to ready_to_test when story has at least one active AC (Type: Positive, Priority: Critical)
- **Given**: A User Story with one active Acceptance Criterion and status `draft`
- **When**: A `PATCH /api/v1/user-stories/{id}` is sent with `{ status: "ready_to_test" }`
- **Then**:
  - API: 200 with updated story including `{ status: "ready_to_test" }`
  - DB: `user_stories.status = 'ready_to_test'`
  - UI: toggle updates to reflect `ready_to_test` state

---

### Original AC5 — Title minimum validation

#### Scenario 5.1: Should reject an AC title shorter than 3 characters (Type: Negative, Priority: High)
- **Given**: A workspace member is adding an Acceptance Criterion
- **When**: They submit a title of exactly 2 characters (e.g., "OK") via POST
- **Then**:
  - API: 422 with `{ error: { code: "validation_failed", details: [{ reason: "title_too_short" }] } }`
  - DB: no row inserted in `acceptance_criteria`
  - UI: inline error message stating the title must be at least 3 characters

#### Scenario 5.2: Should reject an AC title of exactly 201 characters (Type: Boundary, Priority: High)
- **Given**: A workspace member is adding an Acceptance Criterion
- **When**: They submit a title of exactly 201 characters
- **Then**:
  - API: 422 with a title-too-long validation error
  - DB: no row inserted
  - UI: inline error indicating the title exceeds the maximum length

#### Scenario 5.3: Should accept an AC title of exactly 3 characters (Type: Boundary, Priority: Medium)
- **Given**: A workspace member is adding an Acceptance Criterion
- **When**: They submit a title of exactly 3 characters (e.g., "Buy")
- **Then**: 201 created; AC appears in the list with the 3-character title

#### Scenario 5.4: Should accept an AC title of exactly 200 characters (Type: Boundary, Priority: Medium)
- **Given**: A workspace member is adding an Acceptance Criterion
- **When**: They submit a title of exactly 200 characters
- **Then**: 201 created; AC appears in the list with the full 200-character title

---

### Original AC6 — Removing the last AC reverts story status

#### Scenario 6.1: Should auto-revert story status to draft when the last active AC is archived (Type: Positive, Priority: Critical)
- **Given**: A User Story with status `ready_to_test` and exactly one active Acceptance Criterion
- **When**: That AC is archived via `DELETE /api/v1/acceptance-criteria/{id}`
- **Then**:
  - API: 200 with response body including `{ criterion: {..., archived_at: <timestamp> }, user_story_reverted: true }`
  - DB: `acceptance_criteria.archived_at` is stamped; `user_stories.status` is reverted to `draft`; active AC count = 0
  - UI: Story status badge changes to `draft`; toast displays "Story has been moved back to draft — at least one Acceptance Criterion is required"

#### Scenario 6.2: Should NOT revert story status when a non-last AC is archived (Type: Positive, Priority: High)
- **Given**: A User Story with status `ready_to_test` and three active Acceptance Criteria
- **When**: One AC (not the last) is archived
- **Then**:
  - API: 200 with `{ user_story_reverted: false }` (or absence of the flag)
  - DB: `user_stories.status` remains `ready_to_test`; active AC count = 2
  - UI: Story status badge unchanged; no revert toast

---

### New scenarios surfaced from Phase 2 gaps — NEEDS PO/DEV CONFIRMATION

#### Scenario E1: Should update an existing AC title within bounds (Type: Positive, Priority: High) **[NEEDS PO/DEV CONFIRMATION]**
- **NEEDS PO/DEV CONFIRMATION**: edit flow not in any original AC; behavior inferred from scope ("Edit...individual Acceptance Criteria") and implementation plan (`PATCH /acceptance-criteria/{id}`).
- **Given**: An active Acceptance Criterion with title "Full refund within 30 days"
- **When**: A `PATCH /api/v1/acceptance-criteria/{id}` is sent with `{ title: "Full refund within 14 days" }`
- **Then**: 200; the criterion appears with the updated title; position and `archived_at` are unchanged; DB row updated

#### Scenario E2: Should reject an edited AC title that falls below 3 characters (Type: Negative, Priority: High) **[NEEDS PO/DEV CONFIRMATION]**
- **NEEDS PO/DEV CONFIRMATION**: edit validation behavior inferred from validation module applying to PATCH the same as POST.
- **Given**: An active Acceptance Criterion
- **When**: A PATCH is sent with `{ title: "Hi" }` (2 characters)
- **Then**: 422 with `title_too_short`; original title unchanged in DB

#### Scenario E3: Should reject an AC description exceeding 50 KB (Type: Negative, Priority: High)
- **Given**: A workspace member is creating an Acceptance Criterion
- **When**: They submit a description body of 51,201 bytes (50 KB + 1 byte)
- **Then**: 422 validation error; no AC row inserted; UI shows overCap gate message before submission (client-side guard) and server-side rejection as defense-in-depth

#### Scenario E4: Should accept an AC description of exactly 50 KB (Type: Boundary, Priority: Medium)
- **Given**: A workspace member is creating an Acceptance Criterion
- **When**: They submit a description body of exactly 51,200 bytes (50 KB)
- **Then**: 201 created; AC stored with full description

#### Scenario E5: Should return 403 when a non-member attempts to create an AC (Type: Negative, Priority: Critical)
- **Given**: An authenticated user who is not a member of the workspace that owns the User Story
- **When**: They call `POST /api/v1/user-stories/{id}/acceptance-criteria` with a valid request body
- **Then**: 403 `not_a_member`; no AC row inserted; DB state unchanged

#### Scenario E6: Should return 403 when a non-member attempts to archive an AC (Type: Negative, Priority: Critical)
- **Given**: An authenticated user who is not a member of the workspace that owns the AC
- **When**: They call `DELETE /api/v1/acceptance-criteria/{id}`
- **Then**: 403 `not_a_member`; `archived_at` remains null; story status unchanged

#### Scenario E7: Should return 404 when attempting to archive an already-archived AC (Type: Negative, Priority: Medium) **[NEEDS PO/DEV CONFIRMATION]**
- **NEEDS PO/DEV CONFIRMATION**: `mapCriterionRpcError` maps P0002 → 404; inferred that an archived AC is not found in the active set.
- **Given**: An Acceptance Criterion that has already been archived (`archived_at` is not null)
- **When**: A `DELETE /api/v1/acceptance-criteria/{id}` is called again
- **Then**: 404 `not_found`; no state change

#### Scenario E8: Should exclude archived ACs from the active count for the ready-to-test gate (Type: Integration, Priority: Critical)
- **Given**: A User Story with two ACs, one active and one archived
- **When**: The active count is evaluated (e.g., via attempted `PATCH` to `ready_to_test`, or via GET listing)
- **Then**: Active count = 1; the archived AC is not included; the ready-to-test gate is not blocked

---

## Phase 4 — Test Outlines (DRAFT — outline names only)

### Coverage estimate

| Type | Count | Notes |
|------|-------|-------|
| Positive | 9 | Happy path: add, append, insert, reorder, edit, archive (non-last), status transitions |
| Negative | 10 | Validation failures, workspace auth failures, gate rejections, 404 on re-archive, edit with invalid title |
| Boundary | 7 | Title min(3)/max(200)/201, description at 50 KB / 50 KB+1, insert at position 1, reorder at list edges |
| Integration | 5 | Status auto-revert on last-AC archive, ATC binding cascade on AC archive, cross-workspace RLS, concurrent reorder, GET active list excludes archived |
| API | 5 | Per endpoint: POST create, GET list, GET single, PATCH update/reorder, DELETE archive |
| **Total** | **36** | High coverage justified by High complexity on all 4 axes and 9 confirmed gaps |

**Rationale**: Business logic is High (ordered-list state machine with cross-entity status revert) and integration is High (3 SECURITY DEFINER RPCs + RLS workspace gate + new status column on a different table). The 36 outlines are driven by the combination of 6 original ACs (each needing multiple scenarios), 8 confirmed gaps each generating at least one new scenario, and 4 boundary conditions at validation edges. The high negative/boundary ratio (17 of 36) reflects the critical importance of the workspace isolation guarantee and the validation precision required for the 50 KB cap and title bounds.

### Outline list (NAMES ONLY — preconditions in 1 line, expected in 1 line)

#### Positive
- **Should add first AC at position 1 when story has no criteria** — Pre: story with 0 ACs. Expected: 201, position=1, active count=1.
- **Should append AC at tail position when story already has ACs and no position is specified** — Pre: story with 3 ACs. Expected: 201, new AC at position 4, list is 1-2-3-4.
- **Should insert AC at a specified position and shift existing ACs forward by 1** — Pre: story with ACs A(1), B(2), C(3). Expected: 201, inserted AC at requested position, list contiguous.
- **Should move AC from bottom to top and re-number remaining ACs** — Pre: story with 3 ACs. Expected: 200, moved AC at position 1, others at 2 and 3.
- **Should move AC from top to bottom and re-number remaining ACs** — Pre: story with 3 ACs. Expected: 200, moved AC at position 3, others at 1 and 2.
- **Should allow story status transition to ready_to_test with at least one active AC** — Pre: story with 1 active AC. Expected: 200, `status = ready_to_test`.
- **Should update an existing AC title within valid bounds** — Pre: active AC. Expected: 200, title updated, position unchanged. **[NEEDS PO/DEV CONFIRMATION]**
- **Should NOT revert story status when a non-last AC is archived** — Pre: story with 3 ACs and status ready_to_test. Expected: 200, status remains ready_to_test.
- **Should add AC to a story that is already ready_to_test without changing its status** — Pre: ready_to_test story with 1 AC. Expected: 201, status unchanged. **[NEEDS PO/DEV CONFIRMATION]**

#### Negative
- **Should block story status transition to ready_to_test when story has zero active ACs** — Pre: story with 0 ACs. Expected: 409 `ac_required_for_ready_to_test`.
- **Should reject AC creation when title is shorter than 3 characters** — Pre: authenticated member. Expected: 422 `title_too_short`, no DB row.
- **Should reject AC creation when title exceeds 200 characters** — Pre: authenticated member. Expected: 422 title-too-long validation error.
- **Should reject AC creation when description exceeds 50 KB** — Pre: authenticated member. Expected: 422 byte-cap error, no DB row.
- **Should reject AC edit when updated title is shorter than 3 characters** — Pre: active AC. Expected: 422 `title_too_short`, original title preserved. **[NEEDS PO/DEV CONFIRMATION]**
- **Should reject AC edit when updated title exceeds 200 characters** — Pre: active AC. Expected: 422 title-too-long, original title preserved. **[NEEDS PO/DEV CONFIRMATION]**
- **Should return 403 when a non-workspace-member attempts to create an AC** — Pre: user not in workspace. Expected: 403 `not_a_member`, no AC inserted.
- **Should return 403 when a non-workspace-member attempts to archive an AC** — Pre: user not in workspace. Expected: 403 `not_a_member`, `archived_at` remains null.
- **Should return 403 when a non-workspace-member attempts to reorder an AC** — Pre: user not in workspace. Expected: 403 `not_a_member`, positions unchanged.
- **Should return 404 when attempting to archive an already-archived AC** — Pre: AC with `archived_at` set. Expected: 404 `not_found`. **[NEEDS PO/DEV CONFIRMATION]**

#### Boundary
- **Should accept AC title of exactly 3 characters** — Pre: authenticated member. Expected: 201 created.
- **Should accept AC title of exactly 200 characters** — Pre: authenticated member. Expected: 201 created.
- **Should reject AC title of exactly 201 characters** — Pre: authenticated member. Expected: 422.
- **Should accept AC description of exactly 50 KB (51200 bytes)** — Pre: authenticated member. Expected: 201 created.
- **Should reject AC description of exactly 50 KB + 1 byte** — Pre: authenticated member. Expected: 422.
- **Should disable or no-op move-up button for the first AC in the list** — Pre: story with 3 ACs. Expected: no position change for AC at position 1. **[NEEDS PO/DEV CONFIRMATION]**
- **Should disable or no-op move-down button for the last AC in the list** — Pre: story with 3 ACs. Expected: no position change for AC at last position. **[NEEDS PO/DEV CONFIRMATION]**

#### Integration
- **Should auto-revert story status to draft when the last active AC is archived** — Pre: story with 1 active AC and status ready_to_test. Expected: 200, `user_story_reverted: true`, `user_stories.status = draft`.
- **Should exclude archived ACs from active count used in the ready-to-test gate** — Pre: story with 1 active and 1 archived AC. Expected: gate uses count=1, transition is not blocked.
- **Should maintain active-list position contiguity after a sequence of insert → move → archive operations** — Pre: story with 5 ACs. Expected: no gaps in positions after composite operations.
- **Should preserve ATC bindings (or surface a warning) when an AC that is bound to an ATC is archived** — Pre: AC with at least one ATC binding in `atc_acceptance_criteria`. Expected: defined cascade behavior. **[NEEDS PO/DEV CONFIRMATION]**
- **Should enforce workspace isolation: cannot read or mutate ACs from a different workspace via direct API** — Pre: two isolated workspaces, User A in WS-A, AC in WS-B. Expected: 403 or 404 on all operations.

#### API
- **Should return active ACs ordered by position ascending on GET /user-stories/{id}/acceptance-criteria** — Pre: story with 3 ACs at positions 1, 2, 3. Expected: list ordered 1, 2, 3; archived ACs excluded.
- **Should return a single active AC on GET /acceptance-criteria/{id}** — Pre: active AC. Expected: 200 with full AC object.
- **Should return 404 for GET /acceptance-criteria/{id} when AC is archived** — Pre: archived AC. Expected: 404. **[NEEDS PO/DEV CONFIRMATION]**
- **Should include `user_stories.status` in the GET /user-stories/{id} response after BK-15 ships** — Pre: story with known status. Expected: `status` field present in payload (Slice 4 adds it to STORY_COLUMNS).
- **Should return 409 with error code `ac_required_for_ready_to_test` on PATCH /user-stories/{id} when zero ACs exist** — Pre: story with 0 ACs. Expected: exact error code in response envelope.

> **NOT included here** (deferred to in-sprint planning by `/sprint-testing` Stage 1): parametrization tables, per-outline test-data JSON, numbered test steps, Faker generation strategies.

---

## Phase 5 — Edge Cases (DRAFT)

| # | Edge case | In original Story? | Criticality | Action |
|---|-----------|-------------------|-------------|--------|
| 1 | `MAX_AC_DESCRIPTION_BYTES` 50 KB cap | No | High | Add to AC (Gap #1) |
| 2 | Edit existing AC title/description | No | High | Add to AC (Gap #2) — **[NEEDS PO/DEV CONFIRMATION]** |
| 3 | Soft-archive visibility (archived AC in UI or API) | No | Medium | Add to AC (Gap #3) — **[NEEDS PO/DEV CONFIRMATION]** |
| 4 | Story status auto-revert to `draft` on last-AC archive | Partially (AC6 ambiguous) | Critical | Clarify AC6 (Ambiguity #1) |
| 5 | Title maximum 200 chars (AC5 only tests minimum) | No | High | Add to AC (Gap #4) |
| 6 | Default position = tail on append | No | Medium | Add to AC (Gap #5) |
| 7 | Move-up on first AC / move-down on last AC | No | High | Add to AC (Gap #6) — **[NEEDS PO/DEV CONFIRMATION]** |
| 8 | Cross-workspace AC operation (403 path) | No | Critical | Add to AC (Gap #7) |
| 9 | API error code assertions (409 / 422 / 403) | No | High | Add to AC (Gap #8) |
| 10 | AC edit with boundary-invalid title (too short/long) | No | High | Add to AC (Gap #2 extension) — **[NEEDS PO/DEV CONFIRMATION]** |
| 11 | Title with only whitespace characters | No | Medium | Test only — **[NEEDS PO/DEV CONFIRMATION]** |
| 12 | Re-archive an already-archived AC | No | Medium | Test only — **[NEEDS PO/DEV CONFIRMATION]** |
| 13 | ATC binding behavior when bound AC is archived | No | Critical | **[NEEDS PO/DEV CONFIRMATION]** — cross-story concern with BK-18 |
| 14 | Insert at position 1 (prepend all existing) | No | Medium | Covered in Scenario 2.2 |
| 15 | Add AC to a `ready_to_test` story — status unchanged? | No | High | **[NEEDS PO/DEV CONFIRMATION]** |
| 16 | Concurrent reorder by two users simultaneously | No | High | Test only — **[NEEDS PO/DEV CONFIRMATION]** |
| 17 | Description at exactly 50 KB boundary | No | Medium | Test only (Boundary outline) |
| 18 | Move AC to its current position (no-op reorder) | No | Low | Test only — **[NEEDS PO/DEV CONFIRMATION]** |

> Test-data generation strategy and Faker recipes are NOT defined here. They land in `/sprint-testing` Stage 1 when the feature exists.

---

## Story Quality Assessment

**Verdict**: Needs Improvement

**Key findings**:
- The 6 original ACs cover the happy-path ordering scenarios well but leave entire implemented slices untested: the edit path (PATCH title/description) has zero AC coverage despite being explicitly in scope; the 50 KB description cap has zero AC coverage despite being a server-side constraint; and the soft-archive semantics (what happens after "removal") are defined only at the business-rule level with no testable behavior specified.
- Two of the six original ACs contain language that contradicts or underspecifies the implementation: AC6 uses passive language that obscures the automatic status-revert behavior (a critical data mutation), and the workflow description references drag-to-reorder while the implementation ships up/down arrows only.
- Cross-workspace isolation (the most critical risk per the master test plan) has no AC coverage at all. The three SECURITY DEFINER RPCs gate on workspace membership — this path must be explicitly tested and an AC must require it.

---

## Critical Questions for PO

> These BLOCK sprint planning until answered.

1. **Does removing the last active AC automatically revert story status to `draft`, or does it only block future re-marking as ready-to-test?**
   - **Context**: AC6 says "the Story can no longer stay ready to test" (implies a future block), but the implementation plan says the archive RPC "reverts that story to draft and reports it" (immediate mutation). These produce different observable behaviors and require different test assertions.
   - **Impact if unanswered**: QA cannot write the "Then" clause for any archive scenario involving a `ready_to_test` story. If the implementation is wrong relative to the PO's intent, a critical data-state bug ships undetected.
   - **Suggested answer**: Confirm the implementation plan is correct — archive of last active AC auto-reverts story to `draft`. Then update AC6 to state this explicitly.

2. **What are the post-archive visibility rules for an archived AC?**
   - **Context**: The business rule states "Removal is soft archive" but does not say whether archived ACs are (a) hidden from all views and excluded from all counts, (b) visible in a separate "archived" section, or (c) restorable by any user action. This affects the active-count calculation and the ready-to-test gate behavior.
   - **Impact if unanswered**: QA cannot test the "archived AC is excluded from active count" path. If archived ACs are accidentally counted, the gate logic breaks silently.
   - **Suggested answer**: Define: archived ACs are excluded from all active counts, not visible in the active list, and not restorable via any current UI action (soft-delete for audit trail only). Confirm if a "restore" path is planned.

3. **What happens when an Acceptance Criterion that is bound to one or more ATCs is archived?**
   - **Context**: `atc_acceptance_criteria` links ATCs to ACs. If a bound AC is archived (soft-delete), the binding row either remains (pointing to an invisible AC) or cascades to delete. Either way has consequences: orphaned bindings silently weaken the anchoring moat; cascade deletion could remove the last binding on an ATC and violate the "every ATC must bind ≥1 AC" invariant.
   - **Impact if unanswered**: This is a CRITICAL data integrity risk that spans BK-15 and BK-18. If not defined now, both stories may ship with an untested interaction that corrupts the anchoring moat.
   - **Suggested answer**: Define the cascade behavior explicitly. Recommended: soft-archive of an AC should either (a) block if it has active ATC bindings, with a user-facing error, or (b) warn the user and let them proceed (with the binding pointing to the archived AC). Confirm which approach ships in this story vs. BK-18.

---

## Technical Questions for Dev

> These do not block PO but block implementation or accurate test assertion.

1. **Does `PATCH /acceptance-criteria/{id}` for title/description update use direct RLS or call the insert RPC?** — The implementation plan (Slice 3) says "updates title/description through an RLS update and/or position through the move function" — the word "and/or" is ambiguous. Needed to confirm whether a single PATCH can update both title and position atomically, or if they must be separate calls. Testing impact: a combined title+position PATCH test may hit different code paths.

2. **What is the exact format of the `bunkai_archive_acceptance_criterion` return value when `user_story_reverted = true`?** — Implementation plan says "returns jsonb (criterion plus `user_story_reverted` flag)" but does not specify the exact JSON key names or whether the full updated story object is returned. Needed to write precise API response assertions.

3. **Does the Zod `criterionTitleError` measure the 200-character limit by JavaScript string length (UTF-16 code units), Unicode code points, or bytes?** — For CJK/emoji titles this matters. A 200-char Zod `.max(200)` uses code-unit length by default, which means a 100-emoji title (200 code units but 100 code points) would pass while a 101-emoji title fails. Test data depends on the answer.

4. **What is the response shape for `GET /acceptance-criteria/{id}` when the AC is archived — 404 or 410 Gone?** — `mapCriterionRpcError` maps P0002 → 404 for "no row found", which an archived AC would trigger (since the query filters `WHERE archived_at IS NULL`). Confirm 404 is the correct status for archived ACs and not a dedicated 410 or 422 code.

5. **Is the `user_stories.status` column added in migration 0017 protected from direct PostgREST PATCH by RLS?** — The gate logic lives in the application-layer `PATCH /user-stories/{id}` route (Slice 4). If a member can call PostgREST directly and set `status = 'ready_to_test'` on a story with zero ACs, the gate is bypassed entirely. Confirm whether the status transition is enforced at the DB level (trigger or RPC) or only at the application route level.

---

## Suggested Story Improvements

| # | Current state | Suggested change | Benefit |
|---|---------------|------------------|---------|
| 1 | AC6: "the Story can no longer stay ready to test" | "The Story status is automatically reverted to `draft` and the user is informed that at least one Acceptance Criterion is required" | Unambiguously specifies the state mutation; removes the block/revert ambiguity for both Dev and QA |
| 2 | Workflow step 3: "She can drag to reorder" | "She can use up/down arrows to reorder" | Aligns workflow description with implementation design decision; eliminates the reorder-mechanism contradiction |
| 3 | AC5: only tests title minimum (3 chars) | Split into two ACs or add a sub-scenario: "A title of 200 characters is accepted; a title of 201 characters is rejected" | Ensures both bounds of the `criterionTitleError` are formally required, not just tested by implementation detail |
| 4 | No AC for the edit path | Add: "An existing criterion title can be updated; the same title validation rules apply" | Closes the largest single-path coverage gap; the edit endpoint exists but is entirely un-specced |
| 5 | No AC for workspace isolation | Add: "A user without membership in the story's workspace receives a 403 on all AC operations" | Elevates the most critical security requirement (per master test plan) from an implicit implementation assumption to a formal testable requirement |

---

## Data feasibility flags

No pre-existing data is required before this feature exists — all test entities can be generated freshly (workspace → project → module → user story → ACs). No DATA-FEASIBILITY-RISK detected.

One conditional risk: **BK-18 (ATC authoring) is BLOCKED** at time of refinement. The ATC-binding cascade test (Edge case #13) requires a functional ATC with an AC binding. That outline is deferred until BK-18 unblocks. All other outlines can execute independently of BK-18.

---

## Recommended testing strategy

### Pre-implementation
- Resolve the three Critical Questions for PO above before Dev starts. AC6 ambiguity (auto-revert vs. block) and the ATC cascade behavior will drive DB-level design decisions that are expensive to change post-implementation.
- Run Supabase migration 0017 in the test environment and verify: (1) `user_stories.status` column exists with the `draft | ready_to_test` CHECK constraint, (2) the partial unique index on `(user_story_id, position) WHERE archived_at IS NULL` is created, (3) the three SECURITY DEFINER RPCs exist with correct `REVOKE EXECUTE FROM public, anon`.

### During implementation
- Unit test `lib/acceptance-criteria/validation.ts` immediately: title bounds (2 chars, 3 chars, 200 chars, 201 chars), byte cap (exactly 50 KB, 50 KB + 1 byte). These are pure functions — test without any API or DB dependency (Slice 6 explicitly mandates this).
- Verify negative-parking collision safety manually: insert at position 1 in a list of 10 ACs and confirm the `acceptance_criteria` table has no negative-position rows after the operation and all positions are 1..11 contiguous.
- Gate test via `PATCH /user-stories/{id}` with zero ACs immediately after the route extension lands (Slice 4) — before any UI exists.

### Post-implementation (in-sprint by /sprint-testing)
- Execute all 36 outlined test scenarios against staging. Priority order: Critical scenarios first (AC4 gate, AC6 revert, E5/E6 workspace isolation), then High (validation bounds, edit path, archive semantics), then Medium/Boundary.
- Validate the OpenAPI spec update (`bun run openapi:gen`) reflects the new routes and the `status` field on the user story GET response.
- Smoke test the UI with the "manual smoke" plan from Slice 6: add three ACs, reorder via arrows, archive one, verify gate in both directions.
- If BK-18 unblocks, add the ATC-binding cascade test (Edge case #13) as an integration scenario.

---

## Risks & mitigation

| # | Risk | Likelihood | Impact | Mitigated by which outlines |
|---|------|-----------|--------|-----------------------------|
| 1 | Cross-workspace AC access bypasses RLS gate | Medium | Critical | Outlines API-5, E5, E6 |
| 2 | Story status auto-revert does not fire (archive RPC bug) | Medium | High | Scenarios 6.1, Integration-1 |
| 3 | Edit path ships without title validation (no AC coverage in original) | High | Medium | Scenarios E1, E2 (require PO confirmation first) |
| 4 | ATC bindings silently orphaned on AC archive | Medium | Critical | Integration-4 (blocked on BK-18 + PO answer to Critical Question #3) |
| 5 | Position gaps after sequence of operations | Low | Medium | Integration-3 |
| 6 | 50 KB byte cap not enforced server-side (no original AC) | Medium | Medium | Scenarios E3, E4 |
| 7 | Ready-to-test status writable via direct PostgREST bypassing application gate | Medium | High | Integration-5, API-5 (requires Dev answer to Tech Question #5) |

---

## Next steps

- [ ] PO answers the 3 Critical Questions before sprint planning (particularly: auto-revert semantics, archived AC visibility, ATC cascade on archive)
- [ ] Dev answers Technical Questions 1–5 before implementation estimation
- [ ] Workflow document updated to reflect "up/down arrows" (not drag-to-reorder)
- [ ] AC6 updated to explicitly state "auto-reverts story to `draft`"
- [ ] New ACs added: edit path, 50 KB cap, title maximum bound, workspace isolation, position edge behavior
- [ ] Story enters sprint at status `Ready For Dev` once estimated
- [ ] When Story reaches `Ready For QA`, `/sprint-testing` will short-circuit Phases 1-3 (label `shift-left-reviewed` expected)
