# Scenario: A Swarm Forms Around Work

> **Status:** Draft

## The Story

A hive maintainer posts a complex work item that needs multiple skills — security analysis, code implementation, and documentation. No single operator has all three. A swarm forms.

## Steps

### 1. Work Posted

The maintainer creates a work item in the hive:
- Title: "Build and harden a content filtering system"
- Skills needed: security analysis, TypeScript, technical writing
- Type: open for competing proposals or collaborative swarm

### 2. Operators Discover

Three operators see the work item:
- **Alex** — security researcher, strong on threat modeling
- **Jordan** — builder, strong on TypeScript implementation
- **Sam** — technical writer, strong on documentation and SOPs

Each operator's ivy-heartbeat detects the work item (via spoke status aggregation or direct notification). Local work items are created on each operator's blackboard.

### 3. Swarm Forms

The operators signal interest. The swarm forms around the work item:
- Alex takes threat modeling and security review
- Jordan takes implementation
- Sam takes documentation and SOP drafting

Each works locally (ivy-blackboard managing their agent sessions), projecting status through their spokes.

### 4. Parallel Execution

The three operators work in parallel:
- Alex's agent produces a threat model and review checklist
- Jordan's agent implements the content filter, guided by Alex's threat model
- Sam's agent writes user documentation and an integration SOP

Progress is visible through spoke status updates. The hive can see three spokes active on the same work item.

### 5. Convergence

Work products converge in the hive via PRs:
- Alex reviews Jordan's implementation against the threat model
- Sam reviews documentation accuracy against the implementation
- The maintainer reviews the full package

### 6. Completion and Trust

The work item is marked complete. Each operator's trust ledger is updated:
- Attribution records who did what (threat model, code, docs)
- The swarm's collective output is greater than any single operator could produce
- All three operators earned trust from verified, reviewed contributions

### 7. Dissolution

The swarm dissolves. Operators return to their individual work. The trust earned persists.

## What Made This Different?

- No project manager assigned roles. Operators self-selected based on capability.
- Three agents worked in parallel, each amplifying their operator's expertise.
- Attribution was granular — not "the team did it" but "Alex did threat modeling, Jordan did code, Sam did docs."
- The swarm was temporary. No ongoing team to manage. Form, execute, dissolve.
