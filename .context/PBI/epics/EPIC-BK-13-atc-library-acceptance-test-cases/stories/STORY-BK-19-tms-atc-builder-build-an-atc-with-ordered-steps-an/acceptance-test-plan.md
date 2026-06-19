# BK-19 — Acceptance Test Plan (QA)

> Jira field: `customfield_10067` · [View in Jira](https://jira.upexgalaxy.com/browse/BK-19)

# ATP DRAFT — BK-19: TMS-ATC Builder | Build an ATC with ordered steps and assertions

***Shift-Left QA Refresh · 2026-06-18 · Risk Level******:****** HIGH (CRITICAL anchoring moat)***

---

## Resolved Questions (PO analysis — no blockers for sprint-testing)

| Question | Resolution |
| --- | --- |
| Step description min/max | Min 1 char · Max 2048 chars (2 KB) |
| Assertion text min/max | Min 1 char if row exists · Max 2048 chars |
| Tag cap UX mechanism | Input disabled at 10 + inline message "Maximum 10 tags reached" |
| Error message wording | `title*too*short` → "Title must be at least 3 characters" · `ac*outside*user*story` → "Selected Acceptance Criteria must belong to the chosen User Story" · `module*outside*project*subtree` → "Selected module is outside the current project" · `steps*position*invalid` → "Step positions are invalid. Please reorder and try again." |
| 422 unknown error code | Generic form-level banner: "Something went wrong. Please try again." |
| Network failure / 500 | Form stays open + error banner · All state preserved |
| Duplicate tag | Rejected — tags = set (not multiset) |
| Accessibility gate | Best-effort for MVP · Not a hard merge blocker |
| Empty step text | Rejected — min 1 char |
| Form state on 422 | All fields preserved: title, layer, steps, ACs, tags |

---

## ATP Outlines (43 total)

### Positive — Happy Path (8)

- [P-01] Should create an ATC and redirect to detail page when all required fields are valid
- [P-02] Should persist steps in the submitted order after save
- [P-03] Should save an ATC with zero assertions when at least one step is provided
- [P-04] Should save an ATC with the maximum of 10 tags
- [P-05] Should save an ATC with a title of exactly 3 characters (minimum valid)
- [P-06] Should save an ATC with a title of exactly 200 characters (maximum valid)
- [P-07] Should display steps as a markdown numbered list with live preview and format hint
- [P-08] Should display assertions as a YAML bullet list with live preview and format hint

### Negative — Validation Rejections (13)

- [N-01] Should reject save when no User Story is anchored
- [N-02] Should reject save when a User Story is selected but no Acceptance Criterion is selected
- [N-03] Should reject save when zero steps are present
- [N-04] Should reject save when one or more assertions exist but zero steps are present
- [N-05] Should reject save when title is "AB" (2 characters — below minimum)
- [N-06] Should reject save when title is empty
- [N-07] Should reject save when title is 201 characters (above maximum)
- [N-08] Should reject adding an 11th tag when 10 tags are already present; input disabled + inline message shown
- [N-09] Should reject save when no layer is selected
- [N-10] Should reject save when module is outside the project subtree (422 `module*outside*project_subtree`)
- [N-11] Should display field-level error when server returns 422 `title*too*short`
- [N-12] Should display field-level error when server returns 422 `ac*outside*user_story`
- [N-13] Should display field-level error when server returns 422 `steps*position*invalid`

### Boundary — BVA (9)

- [B-01] Should accept a title of exactly 3 characters (lower valid boundary)
- [B-02] Should accept a title of exactly 200 characters (upper valid boundary)
- [B-03] Should reject a title of 201 characters (above upper boundary)
- [B-04] Should accept exactly 10 tags (upper valid boundary)
- [B-05] Should reject the 11th tag (above upper boundary)
- [B-06] Should accept step content of exactly 2048 characters (2 KB boundary)
- [B-07] Should reject step content of 2049 characters (above 2 KB max)
- [B-08] Should save with 0 tags (zero valid boundary)
- [B-09] Should save with 1 step and 0 assertions (minimum valid combination)

### State / Sequence (8)

- [S-01] Should clear AC selections when User Story picker is changed to a different User Story
- [S-02] Should list only ACs belonging to the newly selected User Story after US picker change
- [S-03] Should renumber step positions after moving a step up via the reorder button
- [S-04] Should renumber step positions after moving a step down via the reorder button
- [S-05] Should disable the Submit button and show a loading indicator during in-flight POST
- [S-06] Should preserve form state (steps, ACs, tags) after a 422 server error
- [S-07] Should prevent a second POST when Submit is clicked again during an in-flight request
- [S-08] Should disable first step's "move up" button and last step's "move down" button

### Security / Integrity — CRITICAL (5)

- [I-01] Should verify the saved ATC record has at least one linked AC in the database **(CRITICAL — anchoring moat)**
- [I-02] Should verify the linked AC belongs to the User Story selected in the builder **(CRITICAL)**
- [I-03] Should verify POST /atcs with empty `ac_ids` payload is rejected by the server **(CRITICAL)**
- [I-04] Should prevent creating an ATC anchored to a User Story from a different workspace
- [I-05] Should reject saving an ATC with a module outside the current project subtree

---

## Coverage Summary

| Type | Count |
| --- | --- |
| Positive | 8 |
| Negative | 13 |
| Boundary (BVA) | 9 |
| State / Sequence | 8 |
| Security / Integrity | 5 |
| ***Total**** | ****43*** |

---

## Sprint-Testing Priority Order

1. ***I-01, I-02, I-03*** — Anchoring moat (CRITICAL) — verify DB row after save
2. ***N-11, N-12, N-13*** — 422 error mapping — trigger each server error code
3. ***S-01, S-02*** — Cascading picker state clear on US change
4. ***B-01 through B-09*** — BVA boundary suite
5. ***S-05, S-07*** — Submit double-click guard

---
_Synced from Jira by sync-jira-issues_
