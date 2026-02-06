# Swarm Protocol

> **Status:** Draft
> **Resolved:** 2026-02-06

![Swarms — dynamic operator+agent formations](../assets/concepts/hivemind-sketch-2-swarms.jpg)

## Purpose

The swarm protocol defines how operators and their agents form dynamic teams around work, execute together, attribute contributions, and dissolve when done.

## What is a Swarm?

A temporary formation of operator+agent pairs that converges on a task within a hive. Swarms are:

- **Dynamic** -- form and dissolve around work, not org charts
- **Attributed** -- every contribution is tracked to its operator+agent pair
- **Trust-building** -- successful swarm participation compounds reputation
- **Self-organizing** -- operators join based on capability and interest, not assignment
- **Role-specialized** -- each operator takes a distinct role, not a duplicate of others

## Fundamental Constraint

The fundamental constraint of swarm coordination is **visibility without coupling**. Each operator works locally (ivy-blackboard), projects status outward (spoke), and the hub aggregates without reaching into local state. This means swarm coordination must work with **eventual consistency** -- operators see each other's status through periodic snapshots, not real-time streams.

This constraint drives every design decision below. Where real-time coordination is needed, it happens through git (PRs, issues, comments) not through custom infrastructure.

## Swarm Lifecycle

```
WORK POSTED -> INTEREST SIGNALED -> ROLES CLAIMED -> SWARM SEALED
                                                         |
                                          PARALLEL EXECUTION
                                                         |
                                          CONVERGENCE (PRs)
                                                         |
                                          REVIEW + MERGE
                                                         |
                                          TRUST UPDATED + DISSOLVED
```

### Phase 1: Work Posted

A maintainer or trusted operator creates a hub-level work item (GitHub issue with `hive/work` label and structured YAML front matter). See [Work Protocol](work-protocol.md) for the work item schema.

### Phase 2: Interest Signaled

**Decision: Comment-based intent on the work item's GitHub issue.**

Operators signal interest by posting a structured comment on the hub work item (GitHub issue):

```markdown
## Swarm Interest

**Operator:** @github-handle
**Role:** <one of: architect | builder | reviewer | documenter | tester | specialist>
**Capability:** <brief description of what you bring>
**Availability:** <estimated hours/week or "full-time until done">
**Spoke:** <org/repo> (optional -- link to your spoke project)
```

Why comment-based, not a separate registration system:
- Git-native. Comments are already part of GitHub's coordination model.
- Human-readable. Anyone can see who's interested by reading the issue.
- Audit trail. Every signal of interest is timestamped and attributed by GitHub.
- No infrastructure. No custom API, no database, no webhook setup.

Why not automatic agent discovery:
- Humans decide to join swarms. Agents execute the work.
- Automatic enrollment creates spam and diffuse accountability.
- The operator's judgment about fit is the first quality gate.

### Phase 3: Roles Claimed

**Decision: Self-selection with maintainer approval. Roles are specialized, not duplicated.**

Once enough interest exists, the work item maintainer (or the operator who posted the work) seals the swarm by:

1. Reviewing interest comments
2. Assigning roles (updating the issue with a swarm roster)
3. Creating a `swarm.yaml` file in the hub (see schema below)

**Role model** (inspired by Anthropic's agent teams research -- specialization beats duplication):

| Role | Responsibility | Concurrency |
|------|---------------|-------------|
| **architect** | Decomposes work, defines interfaces, splits tasks | 1 per swarm |
| **builder** | Implements code against the architect's decomposition | 1+ per swarm |
| **reviewer** | Independent review of builder output | 1+ per swarm |
| **documenter** | Documentation, SOPs, user guides | 0-1 per swarm |
| **tester** | Test strategy, test implementation, CI validation | 0-1 per swarm |
| **specialist** | Domain expert (security, performance, etc.) | 0+ per swarm |

Key principles:
- **One architect per swarm.** The architect decomposes the work into independent tasks. This is critical -- without architectural decomposition, parallel builders will collide. (Anthropic research: "monolithic tasks need architectural splitting.")
- **Builders take distinct sub-tasks.** Like Anthropic's agents each picking a different failing test, builders claim distinct pieces of the decomposition. No two builders work on the same piece.
- **Reviewers are independent.** Following pai-collab's parallel-reviews SOP, reviewers work independently and submit findings separately. No group-think.
- **Roles are not ranks.** An architect in one swarm can be a builder in another. Roles describe function, not authority.

**Assignment mechanism:**
1. Self-selection: operators state preferred role in their interest comment
2. Maintainer confirms: the work item maintainer accepts, adjusts, or negotiates
3. Conflict resolution: if two operators want the same singleton role (architect), the maintainer decides based on trust score and past performance

### Phase 4: Swarm Sealed

The maintainer commits a `swarm.yaml` to the hub that formalizes the swarm:

```yaml
schemaVersion: "1.0"
swarm_id: "swarm-<work-item-id>-<short-hash>"
work_item: "<work-item-id>"
status: active
created_at: "<ISO 8601>"
sealed_by: "<github-handle>"  # maintainer who approved

roster:
  - operator: "@alice"
    role: architect
    spoke: "alice-org/project-x"
  - operator: "@bob"
    role: builder
    spoke: "bob-org/project-x"
    sub_task: "implement-parser"
  - operator: "@carol"
    role: builder
    spoke: "carol-org/project-x"
    sub_task: "implement-validator"
  - operator: "@dave"
    role: reviewer

decomposition:
  # Created by the architect after sealing
  - id: "task-1"
    title: "Implement parser"
    assigned_to: "@bob"
    depends_on: []
    status: pending
  - id: "task-2"
    title: "Implement validator"
    assigned_to: "@carol"
    depends_on: ["task-1"]
    status: pending
  - id: "task-3"
    title: "Integration tests"
    assigned_to: null  # tester or builder picks up
    depends_on: ["task-1", "task-2"]
    status: pending

timeout:
  heartbeat_interval: "7d"  # max time between spoke status updates
  stale_threshold: "14d"    # time before operator is considered silent
  work_deadline: null        # optional hard deadline
```

### Phase 5: Parallel Execution

Each operator works locally:
- ivy-blackboard manages their agent sessions and local work items
- ivy-heartbeat provides autonomous dispatch
- The spoke projects status to the hub periodically

**The architect's decomposition is the coordination mechanism.** Each builder claims a distinct sub-task. There is no shared mutable state between builders -- only the decomposition in `swarm.yaml` and the interfaces the architect defined.

**Progress visibility:** Each operator's spoke `status.yaml` is aggregated by the hub. The hub can show a swarm dashboard:

```
Swarm: swarm-content-filter-a1b2
Work: Build and harden a content filtering system
Status: active (3/4 tasks in progress)

@alice (architect)   -- phase: shipped    -- last update: 2h ago
@bob   (builder)     -- phase: build      -- last update: 30m ago  -- task: implement-parser
@carol (builder)     -- phase: harden     -- last update: 1h ago   -- task: implement-validator
@dave  (reviewer)    -- phase: waiting    -- last update: 4h ago
```

### Phase 6: Convergence

Work products converge through PRs to the hub repository (or to a designated integration branch):

1. Each builder submits a PR for their sub-task
2. The architect reviews for interface compliance
3. Reviewers perform independent review (parallel-reviews SOP)
4. The maintainer merges

**Convergence order follows the dependency graph** in the decomposition. If task-2 depends on task-1, task-1's PR merges first.

### Phase 7: Trust Updated and Dissolved

On completion:
1. The swarm.yaml `status` is set to `completed`
2. Each operator's contribution is recorded in the attribution ledger (see below)
3. Trust scores are updated per the [Trust Protocol](trust-protocol.md)
4. The swarm is archived (not deleted -- it's part of the audit trail)

---

## Resolved Questions

### 1. How do operators signal interest in joining a swarm?

**Decision:** Structured comment on the hub work item's GitHub issue.

Operators post a structured interest comment (see Phase 2 above). This is visible, auditable, and requires no custom infrastructure. The maintainer reviews interest and seals the swarm.

No automated matching. No bidding system. The operator decides to volunteer. The maintainer decides to accept. Both are human decisions.

### 2. How is work divided within a swarm?

**Decision:** Architect decomposes, builders self-select sub-tasks, maintainer confirms.

The work division follows a three-step process:

1. **Architect decomposes** the work item into independent, parallelizable sub-tasks with defined interfaces. This is the critical step -- bad decomposition means collision, good decomposition means parallel velocity.
2. **Builders self-select** sub-tasks from the decomposition based on their capability.
3. **Maintainer confirms** assignments and resolves conflicts.

This mirrors Anthropic's finding that task decomposition into independent units is what makes parallel agent work effective. The architect role exists specifically to prevent the "monolithic task" failure mode.

For small work items that don't need decomposition (single-operator work), no swarm forms. The operator simply claims and completes the work item directly.

### 3. How is attribution handled when multiple operators contribute?

**Decision:** Per-operator, per-role, per-sub-task attribution. Granular, not collective.**

Attribution is recorded in the `swarm.yaml` upon completion:

```yaml
attribution:
  - operator: "@alice"
    role: architect
    contribution: "Threat model, interface design, decomposition"
    artifacts:
      - type: document
        ref: "threat-model.md"
      - type: decomposition
        ref: "swarm.yaml#decomposition"
    trust_events:
      - type: swarm_architect
        weight: 1.0

  - operator: "@bob"
    role: builder
    contribution: "Parser implementation"
    artifacts:
      - type: pull_request
        ref: "org/repo#42"
      - type: code
        ref: "src/parser/"
    trust_events:
      - type: swarm_builder
        weight: 1.0

  - operator: "@carol"
    role: builder
    contribution: "Validator implementation"
    artifacts:
      - type: pull_request
        ref: "org/repo#43"
    trust_events:
      - type: swarm_builder
        weight: 1.0

  - operator: "@dave"
    role: reviewer
    contribution: "Security review of parser and validator"
    artifacts:
      - type: review
        ref: "org/repo#42#review-1"
      - type: review
        ref: "org/repo#43#review-1"
    trust_events:
      - type: swarm_reviewer
        weight: 0.5  # reviewers get partial credit (reviewed, didn't build)
```

**Key principles:**
- Attribution names the operator, not the agent. The human is accountable.
- Artifacts link to verifiable evidence (PRs, commits, documents).
- Trust events have weights -- building earns more than reviewing, but both earn trust.
- No "team credit." Every contribution is individually attributed.
- The `swarm.yaml` is the single source of truth for who did what.

### 4. What happens when a swarm member goes silent?

**Decision:** Tiered timeout with automatic reallocation.**

The timeout mechanism uses spoke status updates as the heartbeat signal:

| Threshold | Action | Who acts |
|-----------|--------|----------|
| `heartbeat_interval` exceeded (default 7d) | Warning issued as comment on the work item issue | Automated (hub aggregator) |
| `stale_threshold` exceeded (default 14d) | Operator marked stale in swarm.yaml; sub-task released | Maintainer (human decision) |
| Operator self-reports unavailability | Sub-task released immediately | Operator + Maintainer |

**Reallocation process:**
1. The stale operator's sub-task status is set to `released` in `swarm.yaml`
2. The released sub-task is posted as available for other swarm members or new operators
3. Any partial work (branches, draft PRs) is documented in the sub-task description
4. A new operator can pick up from the partial state or restart

**The maintainer always decides.** Automated detection surfaces staleness. The human decides whether to wait, reallocate, or dissolve the swarm entirely.

Why spoke-based detection:
- No new infrastructure needed. The spoke already projects status.yaml with `lastCommit` timestamps.
- Aligned with the blackboard model -- the hub reads projections, it doesn't poll operators.
- Works offline -- an operator who goes offline simply stops updating their spoke.

### 5. Can swarms span multiple hives?

**Decision:** Yes, but with a single home hive. Cross-hive swarms are federated, not merged.**

A swarm always has a **home hive** where the `swarm.yaml` lives and where the work item was posted. Operators from other hives can join, but:

1. They must have at least `untrusted` membership in the home hive (they fork and follow the home hive's SOPs)
2. Their trust score in the home hive starts from their cross-hive portable trust (see Trust Protocol)
3. Attribution is recorded in the home hive's swarm.yaml
4. Trust earned flows back to both the home hive and the operator's origin hive

**Why not merge hives:**
- Each hive has its own governance, SOPs, and trust model
- Merging creates authority conflicts ("whose rules apply?")
- Federation preserves autonomy while enabling collaboration

**Mechanism:** The operator's spoke can project to multiple hubs simultaneously (spoke-protocol.md already supports this). A cross-hive operator simply registers their spoke with the home hive's hub.

### 6. How do competing proposals work?

**Decision:** Multiple swarms can form on the same work item. The maintainer selects which proposal to merge. Unselected proposals are archived, not deleted.**

This directly extends pai-collab's competing-proposals SOP:

1. A work item is posted with `mode: competing` (see Work Protocol)
2. Multiple operators (or swarms) submit independent proposals
3. Each proposal is a PR to the hub
4. The maintainer reviews all proposals and selects one (or synthesizes from multiple)
5. Unselected proposals are closed with attribution recorded -- the work was done, it just wasn't merged

**Competing proposals vs. collaborative swarms are declared upfront** in the work item:

| Work Mode | Description | When to use |
|-----------|-------------|-------------|
| `solo` | Single operator claims and completes | Simple, well-defined tasks |
| `collaborative` | One swarm forms with role specialization | Complex, multi-skill tasks |
| `competing` | Multiple independent proposals | Design problems, exploratory work |

**Attribution for competing proposals:**
- All proposals earn trust, not just the winner
- The weight differs: merged proposals get full trust weight, unselected proposals get partial weight (the work was real, it was just an alternative path)
- This incentivizes participation even when you might not "win"

---

## Swarm Coordination Anti-Patterns

Drawn from Anthropic's agent teams research and pai-collab operational experience:

| Anti-Pattern | Why it fails | What to do instead |
|-------------|-------------|-------------------|
| **No decomposition** | Builders collide on the same code | Require an architect role to split work before builders start |
| **Duplicate roles** | Two builders solving the same sub-problem | Each builder claims a distinct sub-task from the decomposition |
| **Silent failure** | Operator disappears, blocking the swarm | Spoke-based staleness detection + maintainer reallocation |
| **Assignment without consent** | Operator assigned work they didn't volunteer for | Interest-first model: operators volunteer, maintainer confirms |
| **Swarm without a deadline** | Work drags indefinitely with no accountability | Optional but recommended `work_deadline` in swarm.yaml |
| **Real-time coordination dependency** | Swarm breaks when operators are in different time zones | Async-first: spoke projections + PR-based convergence |

---

## Integration with Other Protocols

| Protocol | Integration Point |
|----------|------------------|
| [Work Protocol](work-protocol.md) | Work items trigger swarm formation. Work mode determines solo/collaborative/competing. |
| [Trust Protocol](trust-protocol.md) | Swarm attribution feeds trust scoring. Role weights determine trust earned. |
| [Spoke Protocol](spoke-protocol.md) | Spoke status is the swarm's visibility mechanism. Staleness detection reads spoke timestamps. |
| [Hive Protocol](hive-protocol.md) | Hive governance determines who can seal swarms and which SOPs apply. |
| [Operator Identity](operator-identity.md) | Operator profiles inform role selection and cross-hive eligibility. |

## Prior Art

- pai-collab's [competing-proposals SOP](https://github.com/mellanon/pai-collab) -- multiple operators propose solutions to the same problem
- pai-collab's [parallel-reviews SOP](https://github.com/mellanon/pai-collab) -- multiple independent reviewers on the same work
- ivy-blackboard's work claiming -- atomic claim prevents double-assignment
- Anthropic's agent teams research (C compiler project) -- specialization, task decomposition, file-based locking, test-driven feedback
