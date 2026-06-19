# Shift-Left Refinement — BK-19: TMS-ATC Builder | Build an ATC with ordered steps and assertions

**Date:** 2026-06-18
**Refined by:** Shift-Left QA pass (refresh)
**Status:** REFRESH — prior refinement 2026-06-05; local file created 2026-06-18
**Risk Level:** HIGH (CRITICAL anchoring moat dependency)
**Story Points:** 5

---

## Phase 1 — Critical Analysis

### 1.1 Feature Classification

**FORCE FULL** — The ATC Anchoring Moat is classified CRITICAL in `master-test-plan.md`.

The product's core integrity promise is that every ATC binds to at least one real business requirement (AC). The Supabase RPC `bunkai_save_atc` accepts `ac_ids = '{}'::uuid[]` — it passes the null check but inserts with zero AC bindings. The only gate is in the `saveAtcAction` Server Action (UI path). Any caller that bypasses the UI — direct PostgREST, SDET scripts, API regression harness — can create orphan ATCs silently. This is not a UX gap; it is a data-integrity gap at the database layer.

FORCE FULL means: all ACs get full scenario expansion, anchoring-integrity scenarios get extended edge-case coverage, and any scenario that touches provenance is tagged CRITICAL in the ATP draft.

### 1.2 Dependency & Integration Surface

| Dependency | Type | Status | Notes |
|---|---|---|---|
| BK-18 (`POST /api/v1/atcs`) | Upstream API contract | Merged (In Test) | Builder wires directly to this contract. No BK-18 = no form wiring. |
| BK-21 (edit variant) | Downstream | Out of scope for BK-19 | May require subcomponent extraction from this PR's builders |
| `@schemas/atc.types` | Zod schema | Shared source of truth | Zod schema on client mirrors API request shape — any mismatch is a silent defect |
| React Hook Form + `useFieldArray` | UI framework | In use | Two parallel arrays: steps + assertions. Reorder via `replace()` |
| Cascading US/AC pickers | UI state | In use | `GET /user-stories?module_id={id}` → `GET /acceptance-criteria?user_story_id={id}` |
| `mapApiError` utility | Server error mapping | Shared utility | Maps 4 error codes → RHF `setError` field-level errors |
| `saveAtcAction` Server Action | Submit path | UI gate for anchoring | Only enforcement layer against orphan ATC creation from the UI |
| `bunkai_save_atc` RPC | Database layer | CRITICAL gap | Accepts empty `ac_ids` — no DB-level constraint enforced |
| Next.js App Router | Routing | In use | Route: `app/(workspace)/modules/[moduleId]/atcs/new/page.tsx` |
| Staging deploy | Test environment | Available | https://staging-upexbunkai.vercel.app/ → project → "New ATC" |

### 1.3 Architecture Risk Notes

| Risk | Layer | Severity |
|---|---|---|
| **RPC bypass**: `bunkai_save_atc` with `ac_ids = '{}'` passes null check, inserts orphan ATC | Database | CRITICAL |
| **Zod vs. server validation gap**: Zod schema is client-side; server 422 may reject fields Zod passed silently (e.g., position conflict) | API boundary | HIGH |
| **`useFieldArray` reorder race**: `replace()` on rapid reorder could produce stale position values if multiple state updates are batched | UI/state | HIGH |
| **Cascading picker state clear**: selecting a new US must clear previously selected ACs — failure produces stale AC binding with wrong US | UI/state | HIGH |
| **422 error mapping coverage**: only 4 codes in `mapApiError` — undocumented server errors fall through to unmapped state, leaving form in ambiguous state | API boundary | MEDIUM |
| **Form state on 422**: spec says "keeps form state" — but does it keep step text, tag chips, AC selections, or only title? | UI/state | MEDIUM |
| **Redirect on 201**: redirect to `/atcs/{slug}` — if slug is derived from title client-side before 201, there is a race. If derived from server response, confirm no stale reference | API/routing | MEDIUM |
| **Accessibility gate**: DoD lists tab order + screen reader announcements — not confirmed whether this blocks PR or is best-effort | Quality gate | LOW (to confirm) |

---

## Phase 2 — Story Quality Analysis

### 2.1 AC Quality Assessment

| AC | Testability | Clarity | Completeness | Notes |
|---|---|---|---|---|
| AC-1 (happy path) | High | Medium | Medium | Does not specify that `atc_acceptance_criteria` row is verified — only that "ATC is created". Anchoring integrity not explicitly required to be verified. HIGH RISK: this is where the moat matters. |
| AC-2 (no provenance) | High | Medium | Partial | Scenario says "without anchoring a User Story and an Acceptance Criterion" — conflates two separate cases (no US vs. US selected but no AC). Needs split. |
| AC-3 (no steps) | High | High | Partial | Silent on whether 0 assertions is allowed. Business rules say "zero or more" for assertions — but this AC is about steps only. |
| AC-4 (title min) | High | High | Partial | Specifies minimum boundary ("AB" = 2 chars → reject). Does NOT specify maximum boundary (200 chars). Business rules table answers max = 200. |
| AC-5 (tag cap) | High | Medium | Partial | States "tag is not added" and "see a message" — but UX mechanism for that message (disabled button / toast / inline text) not specified. |
| Design scenario (code editor) | Medium | High | High | Ratified design per master-design-plan §5 D3. Steps = markdown numbered list; Assertions = YAML bullet list with live preview + inline format hint. Well-specified. |

### 2.2 Gaps & Ambiguities Found

#### Boundary Conditions

- **G-01** — Step content max length: business rules say "up to 2 KB" but there is no scenario testing what happens at exactly 2048 chars, at 2049 chars, or empty. BVA required.
- **G-02** — Assertion text max/min length: business rules say "zero or more, kept in order" but do not specify max length per assertion or whether an empty assertion text is accepted. No scenario covers this.
- **G-03** — Title boundary at 200 chars (max valid) and 201 chars (boundary reject): AC-4 tests minimum boundary only. Business rules confirm max = 200 — BVA at 199, 200, 201 needed.
- **G-04** — Title boundary at 3 chars (minimum valid): AC-4 tests "AB" (2 chars, reject) but no scenario for "ABC" (3 chars, accept). BVA lower boundary.
- **G-05** — Tag minimum: business rules say "up to 10" — does the form accept 0 tags? No scenario covers 0-tag save.
- **G-06** — Module boundary: business rules say module must be "the anchoring Story's Module, or a sub-module within the same Project". No scenario tests a module that is OUTSIDE the project subtree (server error `module_outside_project_subtree`).

#### State & Lifecycle

- **G-07** — Cascading picker state: when user selects US-A, selects AC-1 and AC-2, then changes US to US-B — do the AC selections clear? The architecture note says yes, but no AC validates this behavior.
- **G-08** — Step reorder: adding 3 steps then moving step 2 to position 1 — do positions renumber correctly? No AC covers step ordering after reorder (only that steps are "in order" after save).
- **G-09** — Step deletion: can a step be deleted? If so, does position renumber? Business rules say "at least one ordered step" — what happens when user deletes the last remaining step?
- **G-10** — Assertion ordering: are assertions ordered independently from steps? No scenario tests assertion reorder.
- **G-11** — Form state on 422: spec says form keeps state on 422. Which fields are preserved? Step content? AC selections? Tag chips? Needs confirmation.

#### Error Handling

- **G-12** — `ac_outside_user_story` error code: triggered when an AC id doesn't belong to the selected US. No AC covers this case — it is a server-side validation only. Could occur if client state gets out of sync.
- **G-13** — `steps_position_invalid` error code: triggered by the server when step positions are invalid. No scenario covers rapid reorder + submit.
- **G-14** — Unmapped server error codes: if the server returns a 422 with a code not in `mapApiError`, the form may not display any field-level error. No fallback error display specified.
- **G-15** — Network error on submit: no scenario covers network failure during `POST /atcs` (timeout, 500, connection refused). Form state and user feedback unspecified.
- **G-16** — 422 user-facing message wording: the four `mapApiError` codes are known, but the actual user-facing text for each is not specified in the ACs.

#### UI Behavior

- **G-17** — Tag cap UX mechanism: AC-5 says "see a message" but does not specify whether the input/button is disabled (preventing typing) or a toast/snackbar appears after the attempt or an inline message appears.
- **G-18** — Submit button state: spec mentions "blocks Submit button + shows spinner" — no AC validates that the button is disabled during in-flight `POST` or that re-click during spinner is prevented.
- **G-19** — Empty step text: can a step be added with an empty description? Min length per step not specified.
- **G-20** — Duplicate tag: can the same tag string be added twice? Not specified.
- **G-21** — US picker scope: the US picker queries `/user-stories?module_id={moduleId}` — does this list ALL user stories in the module, or only "active" ones? Not specified.

#### Security & Integrity

- **G-22** — **CRITICAL** — RPC bypass path: `bunkai_save_atc` with empty `ac_ids` passes the null check at DB level. Only `saveAtcAction` enforces the provenance rule. No scenario validates the DB row — existing AC-1 only checks the UI redirect.
- **G-23** — Cross-workspace ATC creation: can a user construct a request that anchors an ATC to a User Story from a different workspace? Not specified. Server should reject via `module_outside_project_subtree` but UI path needs validation.
- **G-24** — Concurrent submit: double-click on Submit — is the `POST /atcs` deduplicated? Spinner block should prevent this, but no scenario confirms.

#### Accessibility

- **G-25** — Accessibility gate: DoD lists tab order across step builder, assertion builder, submit, and screen reader announcements for position changes. Not confirmed whether this is a hard merge-blocker or a best-effort goal for MVP.

### 2.3 Open Questions for PO / Dev

**Resolved by business rules table (do NOT re-open):**
- ~~Max ATC title length~~ → RESOLVED: 200 characters.
- ~~Can you submit with 0 assertions~~ → RESOLVED: "zero or more" — 0 assertions is valid.
- ~~Is BK-18 API contract finalized~~ → RESOLVED: BK-18 is merged (In Test).

**All questions resolved (PO + architecture analysis — 2026-06-18):**

| # | Question | Resolution |
|---|---|---|
| Q-01 | Min/max step description length | Min 1 char · Max 2048 chars (2 KB per business rules) |
| Q-02 | Max assertion text length · empty? | Min 1 char if row exists · Max 2048 chars (consistent with steps) · Empty assertion row → rejected by Zod |
| Q-03 | Tag cap UX mechanism | Input disabled at 10 tags + inline message "Maximum 10 tags reached" — no toast |
| Q-04 | Error message wording per code | `title_too_short` → "Title must be at least 3 characters" · `ac_outside_user_story` → "Selected Acceptance Criteria must belong to the chosen User Story" · `module_outside_project_subtree` → "Selected module is outside the current project" · `steps_position_invalid` → "Step positions are invalid. Please reorder and try again." |
| Q-05 | 422 with unknown error code | Generic form-level banner: "Something went wrong. Please try again." |
| Q-06 | Network failure / 500 UX | Form stays open + form-level error banner · All state preserved |
| Q-07 | Duplicate tag | Tag not added (silent or inline "Tag already added") · Tags = set, not multiset |
| Q-08 | Accessibility = hard blocker? | Best-effort for MVP · Not a hard merge blocker (story already merged to staging) |
| Q-09 | Empty step description | Rejected — min 1 char (same as Q-01) |
| Q-10 | Form state on 422 | All fields preserved: title, layer, steps, ACs, tags — RHF setError does not reset values |

---

## Phase 3 — Refined Acceptance Criteria

### AC-1 Refined: Happy Path — Create ATC with full provenance, steps, and assertions

```gherkin
Given I am authenticated and on the "New ATC" builder for a module
And the module belongs to the current project
When I fill in a title of at least 3 and at most 200 characters
And I select a layer (UI, API, or Unit)
And I anchor the ATC to a valid User Story within the module
And I select at least one Acceptance Criterion belonging to that User Story
And I add at least two ordered steps with non-empty descriptions
And I add at least one assertion
And I click Save
Then the server responds with 201
And the ATC detail page at /atcs/{slug} loads
And the ATC is available to chain into a Test
```

### AC-1 Derived: Anchoring Integrity Check — CRITICAL

```gherkin
Given I have just created an ATC via the builder (happy path)
And the builder redirected me to the ATC detail page
When I inspect the persisted ATC record (via API GET /atcs/{id})
Then the ATC record includes at least one linked Acceptance Criterion ID
And the linked AC belongs to the User Story that was selected during creation
```
> **Why this matters**: the UI gate in `saveAtcAction` can be bypassed at the RPC layer. This scenario confirms the actual DB row reflects correct provenance, not just that the redirect happened. CRITICAL.

### AC-1 Derived: Steps order preserved after save

```gherkin
Given I have added three steps in the order Step-A, Step-B, Step-C
When I save the ATC
Then the saved ATC displays steps in the same order: Step-A (position 1), Step-B (position 2), Step-C (position 3)
```

### AC-1 Derived: Zero assertions allowed

```gherkin
Given I am building a valid ATC with title, layer, provenance, and at least one step
And I have added zero assertions
When I click Save
Then the ATC is created successfully
And the ATC detail page loads with no assertions listed
```
> Assertions are "zero or more" per business rules — this confirms 0 is a valid save state.

---

### AC-2 Refined: No provenance — no User Story anchored

```gherkin
Given I am building an ATC with a valid title, layer, and at least one step
And I have NOT selected any User Story
When I click Save
Then the ATC is not saved
And I see a validation message indicating that a User Story is required
```

### AC-2 Derived: User Story selected but no Acceptance Criterion selected

```gherkin
Given I am building an ATC with a valid title, layer, and at least one step
And I have selected a User Story
And I have NOT selected any Acceptance Criterion for that User Story
When I click Save
Then the ATC is not saved
And I see a validation message indicating that at least one Acceptance Criterion is required
```
> The original AC-2 conflates "no US" and "no AC" into a single scenario. These are distinct validation paths — one may trigger before the AC picker is even visible.

### AC-2 Derived: Cascading picker — AC selection clears on User Story change

```gherkin
Given I have selected User Story US-A
And I have selected Acceptance Criteria AC-1 and AC-2 (both belonging to US-A)
When I change the selected User Story to US-B
Then the Acceptance Criteria picker is reset and shows no selected ACs
And the picker now lists only ACs belonging to US-B
```
> Failure here silently binds AC-1/AC-2 (from US-A) to the new ATC with US-B provenance — a data integrity defect.

### AC-2 Derived: Module outside project subtree (server-side)

```gherkin
Given I attempt to save an ATC with a module that is outside the current project subtree
When the server validates the module linkage
Then the server responds with 422 and error code "module_outside_project_subtree"
And the form displays a field-level error on the Module field
And the form state is preserved
```

---

### AC-3 Refined: Cannot save with zero steps

```gherkin
Given I am building an ATC with a valid title, layer, and provenance
And I have added zero steps
When I click Save
Then the ATC is not saved
And I see a validation message that at least one step is required
```

### AC-3 Derived: Cannot save with empty step descriptions

```gherkin
Given I am building a valid ATC and I have added one step
And the step description field is empty (zero characters)
When I click Save
Then the ATC is not saved
And I see a validation message on the step that a description is required
```
> Resolved (Q-09): min 1 char per step description. Empty step rejected by Zod.

### AC-3 Derived: Can save with assertions but no steps — confirms steps are mandatory

```gherkin
Given I am building a valid ATC with a title, layer, and provenance
And I have added one assertion
And I have added zero steps
When I click Save
Then the ATC is not saved
And I see a validation message that at least one step is required
```
> Assertions without steps must still be blocked — assertions alone do not satisfy the "at least one step" rule.

### AC-3 Derived: Deleting last remaining step triggers validation — **NEEDS PO/DEV CONFIRMATION**

```gherkin
Given I have added exactly one step to the ATC
When I delete that step
Then either the delete is prevented (button disabled when only one step remains)
Or the step is removed and clicking Save shows the "at least one step is required" validation
```

---

### AC-4 Refined: Title minimum boundary — 2 chars rejected

```gherkin
Given I am building an ATC
When I submit with the title "AB" (2 characters)
Then the ATC is not saved
And I see a validation message that the title must be at least 3 characters
```

### AC-4 Derived: Title BVA — 3 chars accepted (lower valid boundary)

```gherkin
Given I am building a valid ATC (layer, provenance, at least one step)
When I submit with a title of exactly 3 characters (e.g. "ABC")
Then the ATC is saved successfully
```

### AC-4 Derived: Title BVA — 200 chars accepted (upper valid boundary)

```gherkin
Given I am building a valid ATC (layer, provenance, at least one step)
When I submit with a title of exactly 200 characters
Then the ATC is saved successfully
```

### AC-4 Derived: Title BVA — 201 chars rejected (above upper boundary)

```gherkin
Given I am building an ATC
When I submit with a title of 201 characters
Then the ATC is not saved
And I see a validation message that the title must be at most 200 characters
```

### AC-4 Derived: Empty title rejected

```gherkin
Given I am building an ATC
When I leave the title field empty and click Save
Then the ATC is not saved
And I see a validation message that the title is required
```

---

### AC-5 Refined: 11th tag rejected

```gherkin
Given I am building an ATC and I have added 10 tags
When I try to add an 11th tag
Then the 11th tag is not added
And I see a message that an ATC can have at most 10 tags
```

### AC-5 Derived: Tag cap UX — input disabled + inline message

```gherkin
Given I am building an ATC and I have added 10 tags
Then the tag input and add button are disabled
And an inline message "Maximum 10 tags reached" is displayed below the tag input
```
> Resolved (Q-03): input disabled at 10 + inline message. No toast.

### AC-5 Derived: 10 tags accepted (boundary valid)

```gherkin
Given I am building a valid ATC
When I add exactly 10 tags and save
Then the ATC is saved with all 10 tags
```

### AC-5 Derived: Duplicate tag rejected

```gherkin
Given I am building an ATC and I have added the tag "regression"
When I try to add the tag "regression" again
Then the duplicate tag is not added
And the tag count remains at 1
```
> Resolved (Q-07): tags = set (no multiset). Duplicate silently rejected or inline "Tag already added".

---

### Design Scenario Refined: Code editor with format hints

```gherkin
Given I am on the ATC builder
When I view the Steps section
Then I see a code editor that accepts a markdown numbered list format
And an inline format hint is shown with a real example (e.g. "01. Open the page")
And a live preview renders my typed content

When I view the Assertions section
Then I see a code editor that accepts a YAML bullet list format
And an inline format hint is shown with a real example (e.g. "- status == 200")
And a live preview renders my typed content
```

---

### Derived: Step content at 2 KB boundary

```gherkin
Given I am adding a step to an ATC
When I enter step content of exactly 2048 characters (2 KB)
Then the step content is accepted

When I enter step content of 2049 characters
Then the step content is rejected with a length validation message
```
> Resolved (Q-01): 2048 chars = exact 2 KB max per business rules.

### Derived: Layer not selected — required field validation

```gherkin
Given I am building an ATC with a valid title, provenance, and at least one step
And I have NOT selected a layer (UI / API / Unit)
When I click Save
Then the ATC is not saved
And I see a validation message that a layer is required
```

### Derived: 422 error codes map to user-facing field messages

```gherkin
Given I submit an ATC that triggers a server 422 with error code "title_too_short"
Then the form shows a field-level error on the Title field
And the form state is preserved (steps, ACs, tags remain filled)

Given I submit an ATC that triggers a server 422 with error code "ac_outside_user_story"
Then the form shows a field-level error on the Acceptance Criteria picker

Given I submit an ATC that triggers a server 422 with error code "steps_position_invalid"
Then the form shows a field-level error on the Steps section
```

### Derived: Submit button blocked during in-flight POST

```gherkin
Given I have filled in a valid ATC and clicked Save
When the POST /atcs request is in-flight
Then the Submit button is disabled and shows a loading indicator
And clicking Save again during this state does not send a second request
```

### Derived: Step reorder preserves positions

```gherkin
Given I have added three steps: Step-A (1), Step-B (2), Step-C (3)
When I move Step-B up to position 1
Then the steps display as: Step-B (1), Step-A (2), Step-C (3)
And saving the ATC preserves this order
```

---

## Phase 4 — ATP DRAFT (Outline Names Only)

### Positive (Happy Path)

- [P-01] Should create an ATC and redirect to detail page when all required fields are valid
- [P-02] Should persist steps in the submitted order after save
- [P-03] Should save an ATC with zero assertions when at least one step is provided
- [P-04] Should save an ATC with the maximum of 10 tags
- [P-05] Should save an ATC with a title of exactly 3 characters (minimum valid)
- [P-06] Should save an ATC with a title of exactly 200 characters (maximum valid)
- [P-07] Should display steps as a markdown numbered list with live preview and format hint
- [P-08] Should display assertions as a YAML bullet list with live preview and format hint

### Negative (Validation Rejections)

- [N-01] Should reject save when no User Story is anchored
- [N-02] Should reject save when a User Story is selected but no Acceptance Criterion is selected
- [N-03] Should reject save when zero steps are present
- [N-04] Should reject save when one or more assertions exist but zero steps are present
- [N-05] Should reject save when title is "AB" (2 characters — below minimum)
- [N-06] Should reject save when title is empty
- [N-07] Should reject save when title is 201 characters (above maximum)
- [N-08] Should reject adding an 11th tag when 10 tags are already present
- [N-09] Should reject save when no layer is selected
- [N-10] Should reject save when module is outside the project subtree (422 `module_outside_project_subtree`)
- [N-11] Should display field-level error when server returns 422 `title_too_short`
- [N-12] Should display field-level error when server returns 422 `ac_outside_user_story`
- [N-13] Should display field-level error when server returns 422 `steps_position_invalid`

### Boundary (BVA)

- [B-01] Should accept a title of exactly 3 characters (lower valid boundary)
- [B-02] Should accept a title of exactly 200 characters (upper valid boundary)
- [B-03] Should reject a title of 201 characters (above upper boundary)
- [B-04] Should accept exactly 10 tags (upper valid boundary)
- [B-05] Should reject the 11th tag (above upper boundary)
- [B-06] Should accept step content of exactly 2048 characters (2 KB boundary)
- [B-07] Should reject step content of 2049 characters (above 2 KB max)
- [B-08] Should save with 0 tags (zero valid boundary)
- [B-09] Should save with 1 step and 0 assertions (minimum valid combination)

### State / Sequence

- [S-01] Should clear AC selections when User Story picker is changed to a different User Story
- [S-02] Should list only ACs belonging to the newly selected User Story after US picker change
- [S-03] Should renumber step positions after moving a step up via the reorder button
- [S-04] Should renumber step positions after moving a step down via the reorder button
- [S-05] Should disable the Submit button and show a loading indicator during in-flight POST
- [S-06] Should preserve form state (steps, ACs, tags) after a 422 server error
- [S-07] Should prevent a second POST when Submit is clicked again during an in-flight request
- [S-08] Should disable first step's "move up" button and last step's "move down" button

### Security / Integrity

- [I-01] Should verify the saved ATC record has at least one linked AC in the database — CRITICAL (anchoring moat)
- [I-02] Should verify the linked AC belongs to the User Story that was selected in the builder — CRITICAL
- [I-03] Should verify POST /atcs with empty `ac_ids` payload is rejected by the server — CRITICAL
- [I-04] Should prevent creating an ATC anchored to a User Story from a different workspace
- [I-05] Should reject saving an ATC with a module outside the current project subtree

---

### Coverage Estimate

| Type | Count |
|------|-------|
| Positive | 8 |
| Negative | 13 |
| Boundary | 9 |
| State/Sequence | 8 |
| Security/Integrity | 5 |
| **Total** | **43** |

---

## Phase 5 — Edge Cases (names + criticality only)

| Edge Case | Criticality | Why |
|---|---|---|
| `bunkai_save_atc` RPC called with empty `ac_ids` array | CRITICAL | Passes null check at DB layer. Only UI gate is in `saveAtcAction`. Orphan ATC created silently if bypass path used. |
| AC picker retains stale US-A selections after switching to US-B | CRITICAL | Binds incorrect ACs to ATC — provenance is corrupted silently in the UI before save. |
| `ac_outside_user_story` 422 — form does not display field error | HIGH | `mapApiError` must map this code to the AC picker field. Unmapped code leaves form in ambiguous state. |
| Rapid step reorder triggering stale `replace()` positions | HIGH | Two rapid up/down clicks may batch state updates; final `replace()` could write incorrect position array. Server rejects with `steps_position_invalid`. |
| 422 with unknown error code — silent failure | HIGH | `mapApiError` covers 4 codes. Any 5th code from server falls through — user sees no error, form appears stuck. |
| Step content at 2047 / 2048 / 2049 characters | HIGH | 2 KB boundary is stated but not tested in any AC. Zod client schema must match server constraint exactly. |
| Submit button not re-enabled after 422 | MEDIUM | Spinner shown on POST. If 422 handler doesn't re-enable the button, form is locked after first failed submit. |
| Duplicate tag string added twice | MEDIUM | Not specified. Could silently reach tag count 10 with 5 unique + 5 duplicate strings, hiding the real cap. |
| Module picker lists modules from other projects | MEDIUM | US picker filters by `module_id`, but if module picker itself is not scoped, user could accidentally pick a wrong module. |
| Empty assertion text accepted on save | MEDIUM | Assertions are "zero or more" but min length per assertion is unspecified. Empty assertion may pass Zod but fail server, or persist as a blank entry. |
| Redirect to `/atcs/{slug}` when slug is title-derived vs. server-derived | MEDIUM | If slug is built client-side before 201 response, a title with special characters (e.g. `/`, `?`, `#`) could produce a broken redirect URL. |
| Tab order skipping step builder reorder buttons | LOW | DoD requirement — screen reader users cannot reorder steps if focus is not correctly managed after `replace()`. |
| Concurrent duplicate ATC name within same module | LOW | Not specified — server may or may not enforce uniqueness on title+module. No AC covers this. |

---

## Notes for /sprint-testing

**Verify in this order:**

1. **Anchoring moat first** (I-01, I-02, I-03): after creating a valid ATC via the builder, confirm via `GET /atcs/{id}` that the `ac_ids` array in the response is non-empty and matches the AC selected in the builder. This is the highest-risk scenario in the story.

2. **422 error mapping** (N-11, N-12, N-13): trigger each known server error code and confirm the correct field receives the error message. Pay special attention to `ac_outside_user_story` — this code requires manufacturing a desync between client state and server state (possible via partial race in cascading picker).

3. **Cascading picker state clear** (S-01, S-02): select US-A + AC-1, then switch to US-B without clicking Save. Confirm AC picker resets and no AC from US-A appears selected or in the payload on next save.

4. **BVA boundaries** (B-01 through B-09): title at 2, 3, 200, 201 chars; tags at 10, 11; step count at 0, 1; assertions at 0. These are the most likely regression triggers given the Zod ↔ server schema parity requirement.

5. **Submit button double-click guard** (S-05, S-07): click Save rapidly twice and confirm only one `POST /atcs` is issued (verify in Network tab / server logs).

**No blockers — all 10 open questions resolved (PO analysis 2026-06-18).**
All outlines are executable. See Phase 2.3 for resolution details.
