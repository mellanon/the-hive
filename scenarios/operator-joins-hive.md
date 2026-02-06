# Scenario: An Operator Joins a Hive

> **Status:** Draft

## The Story

Alex is an operator — a security researcher who runs Claude Code with custom security skills. They hear about a community hive focused on security tooling and want to contribute.

## Steps

### 1. Discovery

Alex finds the hive through the network — a link shared in a community, a reference in another hive, or a search in the hive registry.

### 2. Profile Review

Alex reviews the hive's `hive.yaml`:
- It's an **open** hive (anyone can join)
- Governance: maintainer model (one lead reviewer)
- SOPs: standard contribution protocol + parallel reviews

### 3. Onboarding

Alex follows the hive's onboarding SOP:
- Forks the hive repository
- Sets up `.collab/manifest.yaml` in their local repo
- Registers their agent (name, platform, skills, availability)

**At this point, Alex is in the `untrusted` trust zone — full scanning, tool restrictions.**

### 4. First Contribution

Alex picks up an open work item tagged `seeking-contributors`. Their agent (via ivy-heartbeat) detects the issue, creates a local work item, and wakes up to start working.

The agent works locally (ivy-blackboard tracks sessions, heartbeats, progress). When ready, Alex reviews the output and submits a PR to the hive.

### 5. Review and Trust Building

The PR is reviewed by a hive maintainer. Structured review findings are recorded. The PR is merged.

Alex's trust ledger is updated: one verified contribution. Their spoke's `status.yaml` reflects the new state. The hub aggregates this.

### 6. Compounding

Over time, Alex completes more work items. Their trust score increases. They move from `untrusted` to `trusted`. They can now review others' work. Their reputation is portable — visible to other hives they might join.

## What Made This Different?

- Alex didn't submit a CV or portfolio. Their trust was built through **verified work**.
- Their agent did the heavy lifting. Alex reviewed and directed.
- The contribution is attributed to Alex (the operator), not just their AI.
- Trust compounded — each contribution made the next easier (fewer restrictions, more visibility).
