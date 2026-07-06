# Comments for BK-47

[View in Jira](https://jira.upexgalaxy.com/browse/BK-47)

---

### Nahuel Gomez - 6/29/2026, 11:29:21 PM

## Shift-Left QA Refinement — 2026-06-29

### Quality Gaps Found

| Gap | Severity |
| --- | --- |
| No ACs exist | HIGH |
| No DoD | HIGH |
| "Failing" undefined (per-step/per-ATC/run verdict?) | HIGH |
| "Passing" undefined | HIGH |
| Bug impact on metric unclear | HIGH |
| Calendar vs business hours | MEDIUM |
| Multi-Test US consolidation | MEDIUM |
| Never-passing scenario | MEDIUM |
| Only-passing scenario | MEDIUM |

### Open Questions for PO

1. ***Failing boundary:*** Run status=failed? Or any Run where at least one ATC failed?
2. ***Bug reopening:*** Does re-opening a bug after a "passing" Run reset the time-to-green clock?
3. ***Multi-Test consolidation:*** Union (earliest fail → latest pass across all Tests) or per-Test rollup?
4. ***Blocked steps:*** Do blocked `run_steps` count as "failing" for the metric?
5. ***Calendar or business hours*** for duration computation?

### Open Questions for Dev

1. ***Query path:*** US→ATC→test*steps→runs, or add a direct FK (`run.user*story_id`)?
2. ***Materialized view*** or live computation?
3. ***Aborted runs:*** Excluded or counted as neutral?

### ATP DRAFT — 11 outlines

1. TTC01 — Time-to-green first fail to first pass
2. TTC02 — Earliest failing run selected
3. TTC03 — Only passing runs shows N/A
4. TTC04 — No passing run shows "Still failing"
5. TTC05 — Aborted run skipped
6. TTC06 — Multi-Test consolidation
7. TTC07 — Bug fix window in breakdown
8. TTC08 — Re-opened bug resets time-to-green
9. TTC09 — New run triggers data refresh
10. TTC10 — Workspace isolation
11. TTC11 — Single passing run N/A

Full refinement: `shift-left-bk47.md` in QA repo.

---


_Synced from Jira by sync-jira-issues_
