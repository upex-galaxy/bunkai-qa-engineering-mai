# Comments for BK-43

[View in Jira](https://jira.upexgalaxy.com/browse/BK-43)

---

### Nahuel Gomez - 6/29/2026, 11:29:24 PM

## Shift-Left QA Refinement — 2026-06-29

### Quality Gaps Found

| Gap | Severity |
| --- | --- |
| Integration mechanism undefined (polling/event/webhook?) | HIGH |
| No Gherkin ACs | HIGH |
| Retry policy undefined | HIGH |
| Field mapping undefined | HIGH |
| Sync on update unaddressed | HIGH |
| Deletion semantics undefined | HIGH |
| Authentication mechanism undefined | MEDIUM |
| Duplicate detection | MEDIUM |

### Open Questions for PO

1. ***Sync on update:*** When a Bunkai bug is edited, should the change propagate to the external tracker?
2. ***Deletion semantics:*** If a Bunkai bug is deleted, should the external issue also be deleted?
3. ***External tracker:*** Confirm Jira Cloud?
4. ***Field mapping:*** severity→priority, module→component, evidence→attachment?

### Open Questions for Dev

1. ***Integration mechanism:*** DB event trigger, pg_cron poller, or event bus webhook?
2. ***Retry policy:*** max retries, backoff formula, permanent failure threshold
3. ***Deduplication key:*** external_id field, content hash, or idempotency key?
4. ***Rate limiting:*** Expected external API limits, 429 backoff strategy
5. ***Auth refresh:*** How does admin update expired credentials?

### ATP DRAFT — 13 outlines

1. TDS01 — New defect auto-syncs
2. TDS02 — Fire-and-forget on network failure
3. TDS03 — Failed sync auto-retried
4. TDS04 — Sync-failed badge + retry button
5. TDS05 — One-way: no reverse sync
6. TDS06 — Workspace without integration — no sync
7. TDS07 — Duplicate prevention
8. TDS08 — Permanent auth failure stops retries
9. TDS09 — Bug update propagates (if confirmed)
10. TDS10 — Deletion does not delete external
11. TDS11 — Rate limit backoff
12. TDS12 — Field mapping accuracy
13. TDS13 — Workspace isolation

Full refinement: `shift-left-bk43.md` in QA repo.

---

### Nahuel Gomez - 7/3/2026, 5:32:24 PM

## QA Refinements (Shift-Left Analysis)

### Quality Gaps Found

| Gap | Severity |
| --- | --- |
| Integration mechanism undefined (polling/event/webhook?) | HIGH |
| No Gherkin ACs | HIGH |
| Retry policy undefined | HIGH |
| Field mapping undefined | HIGH |
| Sync on update unaddressed | HIGH |
| Deletion semantics undefined | HIGH |
| Authentication mechanism undefined | MEDIUM |
| Duplicate detection | MEDIUM |

### Open Questions for PO

1. ***Sync on update:*** When a Bunkai bug is edited, should the change propagate to the external tracker?
2. ***Deletion semantics:*** If a Bunkai bug is deleted, should the external issue also be deleted?
3. ***External tracker:*** Confirm Jira Cloud?
4. ***Field mapping:*** severity→priority, module→component, evidence→attachment?

### Open Questions for Dev

1. ***Integration mechanism:*** DB event trigger, pg_cron poller, or event bus webhook?
2. ***Retry policy:*** max retries, backoff formula, permanent failure threshold
3. ***Deduplication key:*** external_id field, content hash, or idempotency key?
4. ***Rate limiting:*** Expected external API limits, 429 backoff strategy
5. ***Auth refresh:*** How does admin update expired credentials?

### ATP DRAFT — 13 outlines

ATP DRAFT lives in the 🧪 Acceptance Test Plan (ATP) field. Covers 13 outlines (7 positive, 4 negative/error, 2 boundary). Full detail in customfield_10067.

---


_Synced from Jira by sync-jira-issues_
