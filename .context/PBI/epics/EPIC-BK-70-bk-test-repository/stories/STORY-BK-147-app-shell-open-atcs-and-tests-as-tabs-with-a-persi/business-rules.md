# BK-147 — Business Rules

> Jira field: `customfield_10116` · [View in Jira](https://jira.upexgalaxy.com/browse/BK-147)

- Read access is unchanged: a user only sees, and can only open, items in projects they already have access to.
- Opening an item never mutates it — the workbench tabs are read/navigate surfaces; editing remains gated by its own stories.
- An item that no longer exists (deleted or not visible) shows a safe not-found state inside the workbench rather than a separate broken page.
- The same item opened twice focuses the existing tab instead of creating a duplicate.

---
_Synced from Jira by sync-jira-issues_
