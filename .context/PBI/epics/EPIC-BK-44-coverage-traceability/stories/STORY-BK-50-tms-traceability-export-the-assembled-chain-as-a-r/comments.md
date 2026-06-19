# Comments for BK-50

[View in Jira](https://jira.upexgalaxy.com/browse/BK-50)

---

### Benjamin Segovia - 6/16/2026, 1:31:59 PM

# ATP DRAFT (continued) — truncated tail sections

> The ATP DRAFT custom field hit Jira's ~200KB ADF content-size limit. These sections did NOT fit in customfield_10067 and are posted here as a continuation. Read together with the field content for the full ATP DRAFT.

## Story Quality Assessment

***Verdict***: Significant Issues

***Key findings*** (1-3 bullets):

- The 3 ACs are clear in INTENT (export, immutability, empty-chain variant) but leave the single most consequential implementation decision completely open: HOW is "read-only snapshot" realized — a downloadable file, a frozen in-app view, or a DB-copy record? This decision changes the data model, the access-control model, and nearly every test outline in Phase 4, so it is a genuine blocker, not a nice-to-have clarification.
- Beyond the AC-level ambiguity, the dominant issue (same shape as sibling BK-48) is ***complete non-existence of the feature being exported***: BK-50 exports a chain (BK-45) that has no implementation, over entity types (BK-24 Tests, BK-30 Runs, BK-31 Defects) with no schema. BK-50 additionally has NO existing persistence/export tooling anywhere in the codebase to build on (confirmed by the repo scan — zero PDF/export libraries installed, zero export routes).
- The "read-only" framing and the Story's explicit goal of handing the artifact to EXTERNAL auditors (who have no system login) implies an access-control surface (a shareable, unauthenticated-or-differently-authenticated retrieval path) that none of the 3 ACs address — this is a security-relevant gap, not just a UX one.

---

## Critical Questions for PO

> These BLOCK sprint planning until answered.

1. ***What artifact format does "export" produce — a downloadable file (PDF/JSON/HTML) the QA Lead can hand to someone with no system login, or an in-app read-only view still gated by authentication?***

1. ***What mechanism guarantees the snapshot reflects the moment of export (AC2) — a deep copy of all chain data stored at export time, or a generated static document that is inherently frozen by nature?***

1. ***Who is allowed to trigger an export, and what access model governs who can later retrieve/view an already-exported snapshot (especially given that external auditors with no login are the Story's stated audience)?***

---

## Technical Questions for Dev

> These do not block PO but block implementation.

1. ***Will the export run synchronously (blocking the request/UI) or asynchronously (background job + "export ready" notification)?*** — Context: Phase 2 Gap #3; the Epic's risk map already flags an N+1/performance risk at the chain-assembly layer (inherited from BK-45), and export adds a serialization step on top. Testing impact: determines whether a large-chain export is tested as a simple synchronous-response assertion or requires polling/notification-flow test design.

1. ***If the snapshot is a DB-copy rather than a static file, what is the storage/retention policy — indefinite, time-limited, or subject to manual deletion by the QA Lead?*** — Context: Phase 2 Gap #1; no AC addresses retention. Testing impact: determines whether "list past exports" and expiry/cleanup test outlines are in scope at all for v1.

1. ***Does the export endpoint independently re-verify workspace/RLS scoping at generation time, or does it trust the caller's already-authenticated session context the same way the live chain view does?*** — Context: Phase 2 Gap #4; an artifact that leaves the system is a higher-stakes surface for a missed RLS check than an in-app read, since there's no second chance to catch a leak after the file is downloaded. Testing impact: determines whether the export endpoint needs its own dedicated tenant-isolation test, separate from BK-45's.

---

## Suggested Story Improvements

| # | Current state | Suggested change | Benefit |
| --- | --- | --- | --- |
| 1 | "a read-only snapshot is produced" (AC1) | "exporting produces a downloadable [file format TBD by PO] containing the full chain, retrievable without requiring the recipient to log into Bunkai" | Removes Critical Q1's format ambiguity and makes the "without giving them system access" promise from the user story testable. |
| 2 | "the snapshot still shows the evidence as it was at export time" (AC2) | "the snapshot is generated as a [static document / versioned copy — PO to choose] that is structurally independent of the live chain, such that no later mutation to the live chain (including deletion of the source story) can alter or invalidate it" | Removes Critical Q2's mechanism ambiguity and makes the immutability guarantee concrete enough to test against deletion/mutation edge cases. |
| 3 | No AC on who can export or who can later view an exported snapshot | Add AC: "Only users with read access to the user story can trigger export; [exported snapshots are retrievable via a scoped, time-limited share mechanism for external recipients — TBD]" | Closes Critical Q3 and Gap #2 — prevents an unintentionally open access surface on an artifact designed to leave the system. |
| 4 | "the snapshot states the story had no coverage at export time" (AC3) | "the snapshot states the story had no coverage at export time, using the copy: '[exact wording TBD by PO]', consistent with the empty-coverage state shown elsewhere in the chain view" | Makes the AC3 assertion deterministic instead of relying on prose interpretation. |

---

## Data feasibility flags

***DATA-FEASIBILITY-RISK******:****** confirmed and concrete — same root cause as BK-48, plus an additional, Story-specific gap.***

BK-50 exports "the assembled evidence chain" — the same chain that BK-45 ("Render full US to bug evidence chain in one read") is responsible for producing. BK-45 is still in status ***Estimation*** (per the orchestrator's known facts). Reusing BK-48's confirmed finding on this shared dependency, then adding what is unique to BK-50:

- ***Entity / fixture missing (shared with BK-48)***: There is no queryable data structure to export. BK-45's own refinement confirms zero implementation of `tests`, `test*runs`/`run*results`, or `defects`/`bugs` tables across all reviewed migrations, and no `GET /api/v1/user-stories/{id}/traceability` endpoint exists. BK-50 has nothing to export — not "limited data," literally no chain-assembly capability at all.
- ***API contract gap (shared with BK-45/BK-48)***: An export capability needs a stable response shape to serialize. That shape is still under PO/Dev negotiation in BK-45's own open questions. Designing an export format against an undefined response shape risks rework.
- ***NEW gap specific to BK-50 — no persistence/export tooling exists at all***: A direct, scoped repo check of `upex-bunkai-tms` found zero PDF/document-generation libraries installed (`jspdf`, `puppeteer`, `exceljs`, `docx`, etc. — none present in `package.json`), zero export/snapshot/download routes, and zero UI export affordances anywhere in the codebase. Unlike filtering (BK-48), which only needs query logic once the chain exists, export needs an entirely new capability class (file generation and/or a new persisted-snapshot data model) that this Epic — and this codebase — has never needed before. This is a second, independent blocker on top of the shared chain-assembly dependency.
- ***Required pre-work***: (1) BK-45 must reach at least a stable, documented chain-assembly contract before BK-50 can be implemented or meaningfully estimated — same as BK-48. (2) PO/Dev must decide the export mechanism (static file vs DB-copy vs hybrid) and the external-access model (file download vs scoped share link) BEFORE estimation, since this decision changes the Story's complexity rating from Medium to High depending on the answer.

***Sequencing risk***: BK-50 should NOT enter sprint planning or receive an SP estimate ahead of BK-45, for the same reason as BK-48. Additionally, recommend the PO/Dev settle the export-mechanism and external-access-model questions (Critical Q1-Q3) in a short design conversation BEFORE estimation — this Story carries a unique "new capability class" risk that filtering (BK-48) does not.

---

## Recommended testing strategy

### Pre-implementation

- Do not write parametrized test-data or numbered test steps yet — defer to in-sprint planning once BK-45's chain contract is stable AND the export mechanism (file vs DB-copy) is decided.
- Track BK-45's status and the PO's answer to Critical Question 2 (persistence mechanism) specifically — that single decision reshapes most of Phase 4's outlines.
- Resolve the 3 Critical PO Questions and the 3 Dev questions before any SP estimation session.

### During implementation

- Verify the export endpoint independently enforces RLS/tenant scoping (Tech Q3) early — an artifact that leaves the system is a higher-stakes surface for a missed isolation check than any in-app read.
- Verify the chosen immutability mechanism (static file vs DB-copy) actually holds under the realistic mutation scenarios in Edge Cases #2 and #4 (source story deleted; linked defect later merged) — these are the scenarios most likely to silently break the "moment of export" promise if the underlying implementation takes a shortcut (e.g. storing a live foreign key instead of a true copy).

### Post-implementation (in-sprint by /sprint-testing)

- Expand the 10 DRAFT outlines into full parametrized test cases with concrete chain shapes, mutation timelines, and exact snapshot copy once BK-45's response shape and the export mechanism are known.
- Add the deferred edge cases (Phase 5, especially #2, #3, #4, #6) as formal ACs or test-only cases per PO's confirmation.
- Design the external-access-model test suite (tokenized link expiry, revocation, no-login retrieval) once Critical Question 3 is answered — this is wholly new test surface for this Epic.

---

## Risks & mitigation

| # | Risk | Likelihood | Impact | Mitigated by which outlines |
| --- | --- | --- | --- | --- |
| 1 | BK-50 estimated/scheduled before BK-45 ships, committing sprint capacity to an unbuildable Story | High | High | N/A — mitigated by sequencing recommendation in `## Data feasibility flags`, not by a test outline |
| 2 | "Read-only snapshot" implemented as a live, re-fetched view rather than a true point-in-time copy, silently breaking AC2 the first time the live chain changes | Medium | Critical | Outline "Should preserve the original chain state in a previously exported snapshot after the live chain changes" |
| 3 | Exported artifact (file or share link) leaks data across workspace boundaries because the export endpoint trusts session context instead of independently re-verifying RLS | Low | Critical | Outline "Should verify the export action enforces the same RLS/tenant-scoping rules as the live chain view" + Critical Question 3 |
| 4 | Snapshot becomes inaccessible or corrupted once its source user story is deleted/archived, defeating the Story's "fixed record" promise | Medium | High | Outline "Should keep a previously exported snapshot retrievable after its source user story is deleted or archived" |
| 5 | No export permission gate implemented, allowing any authenticated user (not just QA Lead) to export and externally share sensitive evidence chains | Medium | Medium | Outline "Should reject the export action for a role without export permission, if such a role exists" + Critical Question 3 |

---

## Next steps

- [ ] PO answers Critical Questions before sprint planning
- [ ] Dev answers Technical Questions before estimation
- [ ] Story enters sprint at status Ready For Dev once estimated
- [ ] When Story reaches Ready For QA, `/sprint-testing` will short-circuit refinement (label `shift-left-reviewed` detected)
- [ ] ***BLOCKER***: Do not estimate or schedule BK-50 ahead of BK-45 reaching a stable chain-assembly contract
- [ ] ***BLOCKER***: Do not estimate BK-50 until PO/Dev decide the export-artifact format and the external-access model (Critical Questions 1-3) — this Story carries a "new capability class" risk that its sibling BK-48 does not

---

### Benjamin Segovia - 6/16/2026, 1:32:13 PM

## Acceptance Test Plan (ATP) — Shift-Left DRAFT ready for review

The ATP DRAFT lives in the Acceptance Test Plan field.

Action Required: review ambiguities, answer critical questions, confirm edge-case behavior, validate parametrization.
Refined on: 2026-06-16 — QA Shift-Left batch session
Local working copy: .context/PBI/epics/EPIC-BK-44-coverage-traceability/stories/STORY-BK-50-tms-traceability-export-the-assembled-chain-as-a-r/shift-left-refinement.md

---


_Synced from Jira by sync-jira-issues_
