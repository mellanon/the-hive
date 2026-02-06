# Work Protocol

> **Status:** Planned

## Purpose

The work protocol defines how work is posted to a hive, how operators discover and claim it, how completion is verified, and how attribution is recorded.

## Work Item Lifecycle

```
CREATED → AVAILABLE → CLAIMED → IN PROGRESS → REVIEW → COMPLETED
                                     ↓                      ↓
                                  BLOCKED              TRUST UPDATED
                                     ↓
                                  RELEASED (back to AVAILABLE)
```

## Open Questions

1. What's the schema for a work item in a hive? (Extends ivy-blackboard's work items?)
2. How do operators discover work across hives? (Feed? Search? Notification?)
3. How are competing proposals handled? (Multiple claims? Bidding?)
4. How is completion verified? (Maintainer review? Automated checks? Peer review?)
5. How does the local blackboard work item relate to the hive-level work item?

## Prior Art

- ivy-blackboard work items — create, claim, release, complete, block (single-operator)
- pai-collab GitHub issues — current work tracking mechanism (multi-operator)
- pai-collab [competing-proposals SOP](https://github.com/mellanon/pai-collab) — multiple approaches to the same work
