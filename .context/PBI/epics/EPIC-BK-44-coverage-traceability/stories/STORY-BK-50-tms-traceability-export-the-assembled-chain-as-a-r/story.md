# TMS-Traceability | Export the assembled chain as a read-only snapshot

**Jira Key:** [BK-50](https://jira.upexgalaxy.com/browse/BK-50)
**Epic:** [BK-44](https://jira.upexgalaxy.com/browse/BK-44) (Coverage & Traceability)
**Type:** Historia
**Status:** Estimation
**Priority:** Medium
**Story Points:** -
**Web Link:** https://staging-upexbunkai.vercel.app/

---

## Overview

## User story

As a QA Lead, I want to export a user story's assembled evidence chain as a shareable, read-only pack so that I can hand auditors and stakeholders a fixed record without giving them system access.

---

## QA Refinements (Shift-Left Analysis) — Added 2026-06-16

> Refined Acceptance Criteria live in the `acceptance_criteria` field (Step 1a).

### Edge Cases Identified

| # | Edge case | In original Story? | Criticality | Action |
| --- | --- | --- | --- | --- |
| 1 | Exporting the same story twice in quick succession produces two independent snapshots (not deduplicated) | No | Low | Test only — don't add AC |
| 2 | Snapshot remains accessible after its source story is deleted/archived | No | High | Add to AC (PO confirm) |
| 3 | Export attempted on a story outside the requester's workspace (crafted ID) | No | High | Add to AC (PO confirm) — security-relevant |
| 4 | A defect linked in the snapshot is later merged/closed in the live system; snapshot must still show the original link | No | Medium | Add to AC (PO confirm) — direct test of the immutability promise |
| 5 | Snapshot format compatibility across future schema changes to the chain (forward-looking) | No | Low | Test only — don't add AC |
| 6 | Snapshot retrieval via a direct/guessed link by a user with no workspace access | No | High | Add to AC (PO confirm) — security-relevant, especially since the Story's stated goal is sharing with external parties |

### Clarified Business Rules

- "Read-only snapshot" (AC1) needs a confirmed artifact format — a downloadable file (PDF/JSON/HTML), a frozen in-app view, or both — before it can be implemented or tested; the user story's "without giving them system access" framing suggests a standalone file or unauthenticated share link rather than a logged-in view.
- "Snapshot reflects the moment of export" (AC2) needs a confirmed persistence mechanism — a deep copy of chain data stored at export time, or a generated static document inherently frozen by nature — since these carry different data models and different test designs.
- No AC currently defines who may trigger an export or who may later retrieve an already-exported snapshot, despite the Story's explicit goal of sharing snapshots with external auditors who have no system login.
- No AC currently defines snapshot storage/retention (in-app retrievable list vs one-time download) or behavior for very large chains (synchronous vs asynchronous export).

### Critical Questions for PO

1. ***What artifact format does "export" produce — a downloadable file (PDF/JSON/HTML) the QA Lead can hand to someone with no system login, or an in-app read-only view still gated by authentication?***

1. ***What mechanism guarantees the snapshot reflects the moment of export (AC2) — a deep copy of all chain data stored at export time, or a generated static document that is inherently frozen by nature?***

1. ***Who is allowed to trigger an export, and what access model governs who can later retrieve/view an already-exported snapshot (especially given that external auditors with no login are the Story's stated audience)?***

### Technical Questions for Dev

1. ***Will the export run synchronously (blocking the request/UI) or asynchronously (background job + "export ready" notification)?*** — Context: Phase 2 Gap #3; the Epic's risk map already flags an N+1/performance risk at the chain-assembly layer (inherited from BK-45), and export adds a serialization step on top. Testing impact: determines whether a large-chain export is tested as a simple synchronous-response assertion or requires polling/notification-flow test design.

1. ***If the snapshot is a DB-copy rather than a static file, what is the storage/retention policy — indefinite, time-limited, or subject to manual deletion by the QA Lead?*** — Context: Phase 2 Gap #1; no AC addresses retention. Testing impact: determines whether "list past exports" and expiry/cleanup test outlines are in scope at all for v1.

1. ***Does the export endpoint independently re-verify workspace/RLS scoping at generation time, or does it trust the caller's already-authenticated session context the same way the live chain view does?*** — Context: Phase 2 Gap #4; an artifact that leaves the system is a higher-stakes surface for a missed RLS check than an in-app read, since there's no second chance to catch a leak after the file is downloaded. Testing impact: determines whether the export endpoint needs its own dedicated tenant-isolation test, separate from BK-45's.

> Full refinement (Phases 1-5, outline DRAFT, risk + data feasibility) lives in the ATP DRAFT custom field and the canonical comment below.

---

## Fields

> Each rich-text field is a separate file in this folder.

- [Acceptance Criteria](./acceptance-criteria.md)
- [Scope](./scope.md)
- [Out Of Scope](./out-of-scope.md)
- [Acceptance Test Plan (QA)](./acceptance-test-plan.md)

---

## Traceability

### Historia (1)

- [BK-45](https://jira.upexgalaxy.com/browse/BK-45): TMS-Traceability | Render full US to bug evidence chain in one read _(Estimation)_

---

## Metadata

- **Created:** 6/1/2026
- **Updated:** 6/16/2026
- **Reporter:** Ely
- **Assignee:** Benjamin Segovia
- **Labels:** new-feature, shift-left-2026-06-16, shift-left-reviewed

---

_Synced from Jira by sync-jira-issues_
