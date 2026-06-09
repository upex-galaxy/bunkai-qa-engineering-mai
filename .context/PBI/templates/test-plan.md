# Test Plan (ATP) — Format Reference

> **Read-only reference.** In jira-native mode the ATP lives in the Story's
> `{{jira.acceptance_test_plan}}` custom field (synced to `acceptance-test-plan.md`).
> In jira-xray mode it lives in a `Test Plan` issue linked to the Story.

---

## ATP: BK-XX — [Story title]

**Story**: BK-XX
**Epic**: BK-XX
**Test type**: [Manual | Automated | Mixed]
**Environment**: staging
**Prepared by**: [QA name]
**Date**: YYYY-MM-DD

---

## Scope

### In scope
- [What this ATP covers]

### Out of scope
- [What this ATP explicitly skips]

---

## Test Cases

### TC-001: [Test case name]

| Field | Value |
|-------|-------|
| Priority | High / Medium / Low |
| Type | Positive / Negative / Boundary / Integration |
| AC reference | AC-1 |
| Preconditions | [what must be true before running] |

**Steps**:
1. [Step 1]
2. [Step 2]
3. [Step N]

**Expected result**: [What passes]

---

### TC-002: [Test case name]

[Same structure]

---

## Coverage Summary

| AC | TCs covering it | Priority |
|----|-----------------|----------|
| AC-1 | TC-001, TC-002 | High |
| AC-N | TC-XXX | Medium |

---

## Risk Areas

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| [Risk description] | Low/Med/High | Low/Med/High | [How to reduce] |

---

## Entry Criteria

- [ ] Story in `Ready For QA`
- [ ] Build deployed to staging
- [ ] Test data available (credentials in `.env`)

## Exit Criteria

- [ ] All High priority TCs passed
- [ ] No open Critical/Blocker bugs
- [ ] Evidence captured
- [ ] ATR written and pushed to Jira
