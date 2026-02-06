# Swarm Protocol

> **Status:** Planned

## Purpose

The swarm protocol defines how operators and their agents form dynamic teams around work, execute together, attribute contributions, and dissolve when done.

## What is a Swarm?

A temporary formation of operator+agent pairs that converges on a task within a hive. Swarms are:

- **Dynamic** — form and dissolve around work, not org charts
- **Attributed** — every contribution is tracked to its operator+agent pair
- **Trust-building** — successful swarm participation compounds reputation
- **Self-organizing** — operators join based on capability and interest, not assignment

## Swarm Lifecycle

```
WORK POSTED → OPERATORS DISCOVER → SWARM FORMS → EXECUTES → DELIVERS → DISSOLVES
                                                                    ↓
                                                              TRUST UPDATED
```

## Open Questions

1. How do operators signal interest in joining a swarm?
2. How is work divided within a swarm? (Self-selection? Assignment? Bidding?)
3. How is attribution handled when multiple operators contribute?
4. What happens when a swarm member goes silent? (Timeout? Reallocation?)
5. Can swarms span multiple hives?
6. How do competing proposals work? (Multiple swarms on the same task?)

## Prior Art

- pai-collab's [competing-proposals SOP](https://github.com/mellanon/pai-collab) — multiple operators propose solutions to the same problem
- pai-collab's [parallel-reviews SOP](https://github.com/mellanon/pai-collab) — multiple independent reviewers on the same work
- ivy-blackboard's work claiming — atomic claim prevents double-assignment
