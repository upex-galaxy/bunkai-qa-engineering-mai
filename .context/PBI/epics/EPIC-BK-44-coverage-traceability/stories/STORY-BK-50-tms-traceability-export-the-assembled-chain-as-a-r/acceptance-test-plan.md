# BK-50 — Acceptance Test Plan (QA)

> Jira field: `customfield_10067` · [View in Jira](https://jira.upexgalaxy.com/browse/BK-50)

# Shift-Left Refinement: BK-50 — TMS-Traceability | Export the assembled chain as a read-only snapshot

***Status***: Refined — Awaiting PO Estimation
***Mode***: Shift-Left (pre-sprint, batch grooming)
***Refined on***: 2026-06-16
***Refined by***: QA — Shift-Left batch session
***Modality***: Jira-native

---

## Phase 1 — Critical Analysis

### Business context

- ***Primary persona affected***: QA Lead — needs a fixed, shareable record of a user story's evidence chain to hand to auditors and stakeholders without granting them system access.
- ***Secondary personas (if any)***: External auditors / stakeholders — the actual consumers of the exported artifact, but they never touch the live system; their only contact with Bunkai is the snapshot itself, which raises the bar on the snapshot's standalone clarity (no UI chrome, no login, to interpret it correctly).
- ***Business value proposition****: Removes the need to grant temporary system access (or take ad-hoc screenshots) to satisfy an audit or stakeholder review — directly supports the Epic's "one-minute, data-backed answer to coverage and audit questions" value statement, extended to **external* parties who cannot log in at all.
- ***KPI(s) influenced***: Audit response time; reduction in ad-hoc access grants for compliance reviews.
- ***User journey position***: A terminal, read-side action on TOP OF the BK-45 evidence-chain view. The QA Lead must already be looking at (or have access to) an assembled chain before exporting it — this Story has no value in isolation, same dependency shape as its sibling BK-48.

### Technical context

- ***Frontend***: No traceability view, chain renderer, or export/download UI exists in the codebase today (`business-feature-map.md` §1 Inventory Summary lists "Test Execution (Runs)", "Defect Management", "Reporting / ROI" all as "Not yet implemented"). A light grep of the target repo (`upex-bunkai-tms`) for export/snapshot/PDF-related terms found zero feature-level matches — only incidental hits on the `export` keyword in module imports (e.g. `lib/tree.ts`, `lib/types.ts`) and one unrelated UI string. No download button, no "Export" action, no print/PDF view pattern exists anywhere in the UI inventory.
- ***Backend***: No chain-assembly endpoint exists (same finding as BK-45's and BK-48's refinements: no `GET /api/v1/user-stories/{id}/traceability` endpoint, no `tests`, `test*runs`, `run*results`, or `defects`/`bugs` tables in any reviewed migration). A `package.json` check for PDF/export libraries (`jspdf`, `puppeteer`, `exceljs`, `docx`, etc.) found none installed — there is no export/rendering toolchain present at all, not even an unused one.
- ***External services***: None identified for this Story specifically (no email-delivery or file-storage integration mentioned in the AC text).
- ***Integration points specific to this Story****: Direct, hard dependency on BK-45 (chain assembly) as the data source being exported, and transitively on BK-24 (Tests), BK-30 (Manual Execution & Runs), BK-31 (Bugs & Defect Heatmap) — none of which exist yet (all "Planificación" per `epic.md`). Additionally, unlike BK-48 (which only reads/filters live data), this Story introduces a NEW integration point that no other Story in the Epic needs yet: ****persisted storage for the exported artifact itself*** (a snapshot must be stored somewhere — file storage, a DB blob/JSON column, or a generated static file — to be re-opened later per AC2).

### Story complexity

| Axis | Rating | Why |
| --- | --- | --- |
| Business logic | Medium-High | Point-in-time snapshot semantics (AC2) require either a deep-copy-at-export strategy or a rendered static artifact — genuinely new logic, not a simple read/filter like BK-48. |
| Integration | High | Depends on a 5-entity chain that doesn't exist yet (inherited), PLUS introduces a brand-new persistence/storage integration point this Epic has not needed before. |
| Data validation | Medium | Must validate "read-only" semantics (no mutation path back into the live chain) and define the empty-chain export's exact content/format. |
| UI | Medium | New export/download action, a snapshot viewer (or file download), and an empty-coverage-stated variant — none of these patterns exist in the current component inventory. |

***Estimated test effort***: Cannot be reliably estimated yet — full ATP (in-sprint, parametrized) is blocked until BK-45 ships a stable, queryable chain data contract AND the export/persistence mechanism (file vs DB snapshot vs rendered document) is decided. Pre-sprint estimate: Medium-High once unblocked, driven mainly by the point-in-time immutability verification (AC2) and the choice of snapshot format.

### Epic-level inheritance (if applicable)

- Risks restated at Story level: the Epic (`epic.md`) explicitly states this is **"a read-side capstone over the test-execution layer... scheduled to be implemented once those land."** BK-50 inherits this sequencing constraint directly, same as BK-48.
- Integration points inherited: US → AC → ATC → Test → Run → Defect chain (defined in BK-45's refinement) is the dataset BK-50 exports. BK-50 does not introduce new chain entities, but DOES introduce a new artifact entity (the "snapshot" itself) that has no analog elsewhere in the Epic.
- PO/Dev answers already given at epic level: none yet — BK-45's refinement documents 7 open PO questions and 4 open Dev questions still unanswered, several of which (chain response shape, "latest run" definition) are prerequisites for knowing exactly **what** BK-50 must serialize into a snapshot.
- Test strategy inherited: same caution as BK-45/BK-48 — do not assign SP until upstream dependencies are confirmed for the same sprint window.
- Unique considerations not covered at epic level: snapshot persistence format (file vs DB record vs generated document), the meaning of "read-only" (UI-only restriction vs an actual access-control/immutability mechanism), and the empty-chain export's exact wording/format are specific to BK-50 and not addressed by BK-45 or BK-48's refinements.

---

## Phase 2 — Story Quality Analysis

### Ambiguities

| # | Location in Story | Question for PO/Dev | Impact on testing | Suggested clarification |
| --- | --- | --- | --- | --- |
| 1 | AC1: "a read-only snapshot is produced" | What format is the snapshot — a downloadable file (PDF/JSON/HTML), a new read-only in-app view/URL, or both? "Read-only" could mean a UI restriction (no edit controls shown) or a genuinely separate artifact disconnected from the live system. | Cannot design the export action's test (file download assertion vs UI route navigation vs API response shape) without knowing the artifact type. | Specify: is the deliverable a downloadable file the QA Lead can hand to an external auditor (per the user story's "without giving them system access"), or an in-app URL still gated by login? The user story text ("hand auditors... without giving them system access") strongly implies a standalone file, not a logged-in view — but the AC text alone doesn't say so explicitly. |
| 2 | AC1: "contains the same chain shown on screen" | Does the snapshot need to visually replicate the on-screen tree/table layout, or just contain the same underlying data (which could be rendered differently in export form, e.g. a flat report instead of a tree)? | Determines whether visual-parity assertions (screenshot diff / layout match) are in scope, or only data-completeness assertions. | Clarify: snapshot must contain the same DATA (all chain entities/fields), not necessarily an identical visual layout. Confirm with PO. |
| 3 | AC2: "the snapshot still shows the evidence as it was at export time" | What is the persistence mechanism that guarantees this — a deep copy of all chain data stored at export time, or a generated static file (PDF/HTML) that is inherently frozen? These have very different implementation AND testing implications (a DB-copy approach needs an explicit "this row belongs to snapshot X" versioning model; a static-file approach is automatically immutable by nature). | Cannot design the "chain changes after export, snapshot is unaffected" test without knowing which mechanism is used — a DB-copy approach needs to verify which fields were copied vs referenced live; a static-file approach only needs to verify the file isn't regenerated. | This is the single most important open question in this Story — recommend PO/Dev settle on the mechanism before estimation, since it changes both the data model and the QA test design. |
| 4 | AC1 + Story title: "read-only" | Is "read-only" purely descriptive (the snapshot has no edit UI) or does it imply an actual access-control / immutability guarantee (e.g. cannot be deleted, cannot be tampered with, has its own permission model separate from the live chain)? | Determines whether access-control test cases (can a non-QA-Lead role view/delete a snapshot? can the snapshot be edited via a direct API call even if the UI hides edit controls?) are in scope. | Confirm whether "read-only" is a UI-only concept or a backend-enforced immutability/access-control guarantee. |
| 5 | AC3: "the snapshot states the story had no coverage at export time" | What is the exact wording/format for this empty-chain snapshot? Is it a distinct document/file at all, or just the same export format with an empty-state message inside it (consistent with BK-45's "no coverage" empty state, mentioned in BK-48's refinement)? | Assertion text/format cannot be written without knowing the exact empty-state copy and whether it's delivered via the same artifact type as a populated export. | Provide exact copy (e.g. "No coverage existed for this story as of {date}") or confirm it reuses BK-45's existing empty-state copy/component, now baked into the export artifact. |

### Gaps (missing info)

| # | Type | Why critical | What to add | Risk if omitted |
| --- | --- | --- | --- | --- |
| 1 | AC | No AC defines snapshot storage/retention — where are exported snapshots kept, for how long, and can the QA Lead retrieve a past export later (a list of prior exports), or is each export a one-time, fire-and-forget download? | Add AC specifying whether snapshots are listed/retrievable in-app (a "Past Exports" view) or are pure point-in-time downloads with no server-side retention. | QA cannot design retrieval tests, retention/expiry tests, or storage-quota tests without this; also affects whether AC2's "opens a previously exported snapshot" implies an in-app open action or simply re-opening a previously downloaded file on the user's own machine. |
| 2 | AC | No AC defines who can export — is exporting restricted to the QA Lead role, or can any authenticated user with read access to the user story trigger an export? | Add AC specifying the role/permission gate on the export action (Universal Question U2 / Archetype Permissions P1). | Without a defined gate, QA cannot write a negative authorization test, and an unintended role could export sensitive evidence chains. |
| 3 | AC | No AC defines what happens to an in-flight export if the underlying chain is very large (the Epic's risk map flags chain-assembly N+1/performance risk at BK-45) — does export run synchronously (blocking the UI) or asynchronously (background job + notification when ready)? | Add AC or note specifying export is synchronous for v1, with an explicit size/row-count ceiling, OR async with a "your export is ready" notification. | QA cannot design a "large chain export" boundary/performance test, and a sync, blocking implementation could appear to hang for QA Leads with large stories. |
| 4 | Technical detail | No AC addresses tenant/workspace scoping of the export artifact itself — once exported, does the file/record carry any workspace-identifying metadata, and is a downloaded file subject to the same RLS/tenant-isolation rules the master test plan flags as CRITICAL (`master-test-plan.md` §Silent Killers)? | Add a note or AC confirming exported files cannot be used to infer or access data from a different workspace, and that the export action itself respects RLS (a user cannot export a chain for a story outside their workspace via a crafted ID). | A downloaded artifact is, by definition, outside the system's normal RLS enforcement once it leaves the app — if the export endpoint itself doesn't enforce workspace scoping at generation time, this is a direct data-leak vector, consistent with the master test plan's CRITICAL Tenancy risk. |

### Edge cases not in Story

| # | Scenario | Expected behavior (best guess) | Criticality | Action |
| --- | --- | --- | --- | --- |
| 1 | QA Lead exports the same user story's chain twice in quick succession (no changes in between) | Two independent snapshots are created (each is its own point-in-time record), not deduplicated | Low | NEEDS PO/DEV CONFIRMATION |
| 2 | The underlying user story itself is deleted or archived after a snapshot was exported | The snapshot remains viewable/downloadable on its own (it is a standalone artifact, decoupled from the live story's lifecycle) | High | NEEDS PO/DEV CONFIRMATION — if the snapshot is implemented as a live reference rather than a deep copy, this could silently break AC2's immutability promise |
| 3 | Export is requested for a user story the requesting user does not have read access to (crafted ID / direct API call) | Export is rejected (403/404), consistent with the master test plan's CRITICAL tenant-isolation concern | High | NEEDS PO/DEV CONFIRMATION — security-relevant |
| 4 | Chain is exported, then a defect linked to one of the chain's tests is later marked as a duplicate/closed and merged into another defect | Snapshot continues to show the original defect link as it existed at export time, unaffected by the later merge | Medium | NEEDS PO/DEV CONFIRMATION — direct test of AC2's immutability promise against a realistic real-world mutation, not just a generic "data changed" case |
| 5 | Snapshot file/record format becomes incompatible after an app upgrade changes the chain's underlying schema (e.g. a future Story adds a new chain layer) | Old snapshots remain readable in their original format (versioned schema) rather than breaking or silently dropping fields | Low | Test only — don't add AC (forward-looking, not blocking for v1) |

### Contradictions

No contradictions found between the Story description, the 3 ACs, and the (empty) comments. The user story's framing ("hand auditors and stakeholders a fixed record without giving them system access") and AC1's "read-only snapshot" are consistent in intent; the ambiguities above are gaps in specificity (format, mechanism, access model) rather than actual disagreements between sections.

### Testability validation

***Verdict***: No

Issues:

- Vague AC: AC1's "read-only snapshot" has no defined artifact format (file vs in-app view) or exact content shape.
- Missing error messages: no AC defines the response/copy when export is attempted on an unauthorized or nonexistent story.
- No test-data examples: none of the 3 ACs specify a concrete chain shape, a concrete "change after export" scenario, or exact empty-state copy.
- Missing performance criteria: no AC addresses export time or size limits for a large chain (inherits BK-45's N+1 assembly risk, now compounded by whatever serialization the export step performs).
- ***Cannot isolate***: dominant testability blocker — this Story exports a chain (BK-45) that has zero implementation, over entity types (BK-24 Tests, BK-30 Runs, BK-31 Defects) with no schema, AND additionally requires a persistence/storage mechanism that does not exist anywhere in the current codebase (confirmed by the light repo scan — no PDF/export library installed, no snapshot/export route present). See `## Data feasibility flags` below.

---

## Phase 3 — Refined Acceptance Criteria

### Original AC1 — Export an evidence chain

#### Scenario 1.1: Should produce a read-only snapshot containing the full assembled chain when the QA Lead exports a populated user story (Type: Positive, Priority: High)

- ***NEEDS PO/DEV CONFIRMATION***: artifact format inferred — confirm before sprint planning
- ***Given***: a user story with an assembled evidence chain (AC → ATC → Test → Run → Defect, mixed pass/fail)
- ***When***: the QA Lead triggers the export action
- ***Then***: a snapshot artifact is produced containing every chain entity and field visible on screen

#### Scenario 1.2: Should reject export of a user story the requesting user has no read access to (Type: Negative, Priority: High)

- ***NEEDS PO/DEV CONFIRMATION***: behavior inferred — confirm before sprint planning
- ***Given***: a user story belonging to a different workspace than the requesting user's
- ***When***: the user attempts to export it (e.g. via a crafted story ID)
- ***Then***: the export is rejected with 403/404, no snapshot is created, no chain data is disclosed

### Original AC2 — Snapshot reflects the moment of export

#### Scenario 2.1: Should preserve the chain's state at export time when the live chain changes afterward (Type: Positive, Priority: Critical)

- ***NEEDS PO/DEV CONFIRMATION***: mechanism inferred — confirm before sprint planning
- ***Given***: a user story exported at time T0 with a specific chain state (e.g. AC-1 → Test-A → Run "pass")
- ***When***: after export, the live chain changes (e.g. a new Run is added with verdict "fail", or an existing Defect is closed) and the QA Lead later opens the previously exported snapshot
- ***Then***: the snapshot still displays the chain exactly as it was at T0, unaffected by the later changes

### Original AC3 — Export an empty chain

#### Scenario 3.1: Should produce a snapshot stating the story had no coverage when exporting a story with zero chain entities (Type: Negative/Edge, Priority: High)

- ***NEEDS PO/DEV CONFIRMATION***: exact copy inferred — confirm before sprint planning
- ***Given***: a user story with no ACs, ATCs, Tests, Runs, or Defects (zero coverage, consistent with BK-45's "no coverage" empty state)
- ***When***: the QA Lead exports it
- ***Then***: a snapshot artifact is produced stating the story had no coverage as of the export timestamp (exact copy TBD)

### New scenarios surfaced from Phase 2 edge cases — NEEDS PO/DEV CONFIRMATION

#### Scenario E1: Should keep a previously exported snapshot accessible and unaffected after its source user story is deleted or archived (Type: Edge, Priority: High)

- ***NEEDS PO/DEV CONFIRMATION***: behavior inferred — confirm before sprint planning
- ***Given***: a snapshot exported from user story X, which is later deleted/archived
- ***When***: the QA Lead (or an auditor holding the snapshot) opens the previously exported snapshot
- ***Then***: TBD — either the snapshot remains fully viewable as a standalone artifact, or it becomes inaccessible (would break AC2's immutability promise if so)

#### Scenario E2: Should reject an unauthenticated/unauthorized attempt to re-open or download a snapshot via direct link (Type: Negative, Priority: High)

- ***NEEDS PO/DEV CONFIRMATION***: behavior inferred — confirm before sprint planning
- ***Given***: a snapshot exists, and its retrieval URL/ID is known or guessed by a user without access to the original story's workspace
- ***When***: that user attempts to retrieve the snapshot
- ***Then***: TBD — access is rejected (403/404), consistent with the master test plan's CRITICAL tenant-isolation concern; this is especially important given the Story's explicit goal of handing snapshots to EXTERNAL auditors, which implies some access path exists outside normal in-app authentication

---

## Phase 4 — Test Outlines (DRAFT — outline names only)

### Coverage estimate

| Type | Count | Notes |
| --- | --- | --- |
| Positive | 3 | Export populated chain, export empty chain, re-open snapshot after live chain changed |
| Negative | 3 | Export of unauthorized story, unauthorized snapshot retrieval, export attempt with no read access |
| Boundary | 2 | Export of a very large chain (performance/size ceiling), snapshot retrieval after source story deleted/archived |
| Integration | 2 | Snapshot persistence mechanism (DB-copy vs static file) verified independent of live chain; export action respects RLS/tenant scoping |
| API | 0 | No export endpoint contract exists yet to enumerate against — deferred until BK-45's chain endpoint AND the snapshot persistence mechanism are designed |
| ***Total**** | ****10*** |  |

***Rationale***: The count is deliberately modest — this Story has the same unbuilt-chain blocker as BK-45/BK-48 PLUS an entirely new, undecided persistence mechanism. The 10 outlines capture what is testable in principle today: the export action's shape, the immutability guarantee (the Story's most distinctive and risk-bearing AC), the empty-chain variant, and the access-control implications of handing artifacts to external parties. Deeper outlines (exact file format assertions, retention/expiry, large-chain performance thresholds) cannot be enumerated until the persistence mechanism is chosen.

### Outline list (NAMES ONLY — preconditions in 1 line, expected in 1 line)

#### Positive

- ***Should produce a read-only snapshot containing the full chain when exporting a populated user story*** — Pre: story with an assembled chain. Expected: snapshot artifact produced with all chain entities present.
- ***Should produce a snapshot stating no coverage when exporting an empty-chain user story*** — Pre: story with zero ACs/ATCs/Tests/Runs/Defects. Expected: snapshot states no coverage existed at export time, not an error.
- ***Should preserve the original chain state in a previously exported snapshot after the live chain changes*** — Pre: story exported at T0, then a new Run/Defect added afterward. Expected: re-opening the snapshot shows the T0 state, unaffected by the later change.

#### Negative

- ***Should reject export of a user story outside the requester's workspace*** — Pre: crafted story ID belonging to another workspace. Expected: 403/404, no snapshot created, no data disclosed.
- ***Should reject retrieval of a snapshot by a user without access to its source workspace*** — Pre: known/guessed snapshot URL or ID, requester lacks workspace access. Expected: 403/404, no chain data disclosed.
- ***Should reject the export action for a role without export permission, if such a role exists*** — Pre: authenticated user with read-only chain visibility but no export permission (PENDING Phase 2 Gap #2 — role model for export undefined). Expected: 403, export action unavailable or blocked server-side.

#### Boundary

- ***Should complete (or queue) export for a very large chain without timing out or truncating data*** — Pre: story with a high ATC/Run/Defect count. Expected: full chain present in the snapshot; no silent truncation (NEEDS PO/DEV CONFIRMATION on sync vs async export — Phase 2 Gap #3).
- ***Should keep a previously exported snapshot retrievable after its source user story is deleted or archived*** — Pre: snapshot exported, then source story deleted/archived. Expected: snapshot remains a standalone, viewable artifact (NEEDS PO/DEV CONFIRMATION — Scenario E1).

#### Integration

- ***Should verify the snapshot's persisted data is structurally independent of the live chain (no shared foreign keys that would let live mutations leak into the snapshot)*** — Pre: persistence mechanism implemented (DB-copy or static file). Expected: mutating the live chain after export never alters the snapshot's stored payload.
- ***Should verify the export action enforces the same RLS/tenant-scoping rules as the live chain view*** — Pre: export endpoint implemented. Expected: export respects workspace boundaries identically to the read path it's exporting from (inherits the master test plan's CRITICAL Tenancy risk).

> ***NOT included here*** (deferred to in-sprint planning by `/sprint-testing` Stage 1): parametrization tables, per-outline test-data JSON, numbered test steps, Faker generation strategies. Coverage estimate IS included because PO uses it for estimation.

---

## Phase 5 — Edge Cases (DRAFT)

| # | Edge case | In original Story? | Criticality | Action |
| --- | --- | --- | --- | --- |
| 1 | Exporting the same story twice in quick succession produces two independent snapshots (not deduplicated) | No | Low | Test only — don't add AC |
| 2 | Snapshot remains accessible after its source story is deleted/archived | No | High | Add to AC (PO confirm) |
| 3 | Export attempted on a story outside the requester's workspace (crafted ID) | No | High | Add to AC (PO confirm) — security-relevant |
| 4 | A defect linked in the snapshot is later merged/closed in the live system; snapshot must still show the original link | No | Medium | Add to AC (PO confirm) — direct test of the immutability promise |
| 5 | Snapshot format compatibility across future schema changes to the chain (forward-looking) | No | Low | Test only — don't add AC |
| 6 | Snapshot retrieval via a direct/guessed link by a user with no workspace access | No | High | Add to AC (PO confirm) — security-relevant, especially since the Story's stated goal is sharing with external parties |

> Test-data generation strategy + Faker recipes are NOT defined here. They land in `/sprint-testing` Stage 1 when the feature exists.

---
_Synced from Jira by sync-jira-issues_
