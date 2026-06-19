# BK-147 — Acceptance Criteria

> Jira field: `customfield_10063` · [View in Jira](https://jira.upexgalaxy.com/browse/BK-147)

```gherkin
Scenario: Explorer stays visible when opening an item
  Given Elena is in a project with modules, ATCs and Tests
  When she clicks an ATC or a Test in the explorer
  Then it opens as a tab in the workbench
    And the explorer tree remains visible beside it
    And the opened item is highlighted in the tree
```

```gherkin
Scenario: Multiple tabs open at once
  Given Elena has one item open
  When she opens a second and a third item
  Then each opens in its own tab
    And she can switch between tabs without losing the others
```

```gherkin
Scenario: Re-opening a focused item does not duplicate the tab
  Given an ATC is already open in a tab
  When Elena clicks that same ATC in the explorer
  Then its existing tab is focused instead of opening a duplicate
```

```gherkin
Scenario: Closing a tab
  Given Elena has several tabs open
  When she closes the active tab
  Then it is removed and an adjacent tab becomes active
    And the explorer stays visible
```

```gherkin
Scenario: Deep link opens directly as a tab
  Given Elena pastes a direct link to a Test
  When the page loads
  Then the Test opens as a tab within the workbench with the explorer visible
```

---
_Synced from Jira by sync-jira-issues_
