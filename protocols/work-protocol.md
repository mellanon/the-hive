# Work Protocol

> **Status:** Draft
> **Resolved:** 2026-02-06

## Purpose

The work protocol defines how work is posted to a hive, how operators discover and claim it, how completion is verified, and how attribution is recorded.

## Fundamental Constraint

Work coordination in The Hive operates across three independent layers with different consistency models:

| Layer | Consistency | Latency | Who sees it |
|-------|------------|---------|-------------|
| **Local** (ivy-blackboard) | Strong (SQLite transactions) | Milliseconds | One operator's agents |
| **Spoke** (.collab/) | Eventual (snapshot on push) | Minutes to hours | The operator + hub |
| **Hub** (git repo) | Eventual (PR-based) | Hours to days | All hive members |

This means: **local work items are real-time, hub work items are async.** The protocol must bridge these without pretending they're the same thing.

## Work Item Schema

### Hub-Level Work Item (GitHub Issue with YAML Front Matter)

Hub work items are GitHub issues on the hive repository. They use a structured YAML block in the issue body to enable machine parsing while remaining human-readable:

```markdown
---
hive_work:
  schemaVersion: "1.0"
  work_id: "work-<short-hash>"
  type: feature | bugfix | security | documentation | skill | research
  mode: solo | collaborative | competing
  priority: P1 | P2 | P3
  skills_needed:
    - <skill-tag>
  trust_minimum: untrusted | trusted | maintainer
  deadline: <ISO 8601 or null>
  decomposable: true | false
  bounty: null  # reserved for future incentive model
---

## Description

<Human-readable description of the work>

## Acceptance Criteria

- [ ] <Criterion 1>
- [ ] <Criterion 2>

## Context

<Links to related issues, prior art, architectural decisions>
```

**Why GitHub issues, not a custom format:**
- Git-native. Issues are already part of the coordination model.
- Discoverable. GitHub search, labels, and milestones work out of the box.
- Attributable. Every state change (open, close, label, assign) is tracked by GitHub.
- Accessible. Any operator with a GitHub account can participate. Zero setup.

**Why YAML front matter in the issue body:**
- Machine-parseable without a database. The `blackboard` CLI can extract structured data.
- Human-readable. The issue still reads naturally in GitHub's web UI.
- Extensible. New fields can be added without breaking existing issues.
- Versionable. `schemaVersion` enables forward compatibility.

### Hub-Level Work Item Labels

Standard labels for hub work items (applied as GitHub labels):

| Label | Meaning |
|-------|---------|
| `hive/work` | This issue is a hive work item (required) |
| `hive/mode:solo` | Single operator, claim-and-complete |
| `hive/mode:collaborative` | Swarm formation expected |
| `hive/mode:competing` | Multiple proposals welcome |
| `hive/priority:P1` | Critical / urgent |
| `hive/priority:P2` | Standard |
| `hive/priority:P3` | Low priority / nice-to-have |
| `hive/status:available` | Open for claiming |
| `hive/status:claimed` | An operator or swarm is working on it |
| `hive/status:review` | Work submitted, under review |
| `hive/status:completed` | Done and merged |
| `hive/skill:<tag>` | Skill domain needed (e.g., `hive/skill:security`) |

### Local Work Item (ivy-blackboard)

The local work item schema is defined by ivy-blackboard and is unchanged:

```typescript
interface BlackboardWorkItem {
  item_id: string;
  project_id: string | null;
  title: string;
  description: string | null;
  source: string;          // "hive", "github", "local", "operator"
  source_ref: string | null;  // link to hub work item
  status: "available" | "claimed" | "completed" | "blocked" | "waiting_for_response";
  priority: "P1" | "P2" | "P3";
  claimed_by: string | null;
  claimed_at: string | null;
  completed_at: string | null;
  blocked_by: string | null;
  created_at: string;
  metadata: string | null;  // JSON blob
}
```

**The relationship between hub and local work items is one-to-many.** A single hub work item may spawn multiple local work items on an operator's blackboard (one per sub-task in a swarm decomposition, or one representing the whole item for solo work).

---

## Resolved Questions

### 1. What's the schema for a work item in a hive?

**Decision:** GitHub issues with YAML front matter (see schema above). The hub work item extends, not replaces, ivy-blackboard's local schema.

The design separates concerns cleanly:

| Concern | Where it lives | Why |
|---------|---------------|-----|
| **Work definition** (what needs doing) | Hub: GitHub issue with YAML front matter | Shared, discoverable, human-readable |
| **Work execution** (who's doing it, what's the status) | Local: ivy-blackboard work item | Real-time, multi-agent safe, private |
| **Work projection** (what the hub can see) | Spoke: .collab/status.yaml | Curated snapshot, operator-controlled |

The hub work item does NOT try to replicate ivy-blackboard's real-time state. It captures:
- What the work is (definition, acceptance criteria, skills needed)
- What mode it's in (solo, collaborative, competing)
- Who's working on it (swarm roster, via Swarm Protocol)
- Whether it's done (status labels, maintainer merge)

The local work item handles:
- Agent session management
- Atomic claiming
- Heartbeats and progress tracking
- Crash recovery

**Linking mechanism:** The local work item's `source` field is set to `"hive"` and `source_ref` contains the hub issue URL (e.g., `"org/hive-repo#42"`). This is a soft link -- the local blackboard doesn't import hub state, it references it.

```
Hub Issue #42                    Local Blackboard (Operator A)
"Build content filter"    <---   work item { source: "hive", source_ref: "org/repo#42" }
  mode: collaborative            status: claimed
  status: claimed                 claimed_by: agent-session-abc

                           <---   work item { source: "hive", source_ref: "org/repo#42:task-1" }
                                  status: completed (sub-task)

                           <---   work item { source: "hive", source_ref: "org/repo#42:task-2" }
                                  status: claimed (sub-task)
```

### 2. How do operators discover work across hives?

**Decision:** Three complementary mechanisms, from simplest to most sophisticated. Start with the first, add the others as the network grows.**

**Mechanism 1: GitHub-native discovery (available now)**

Work items are GitHub issues. Operators discover them using:
- GitHub issue search across repositories they follow
- GitHub notifications when issues are created in hives they're members of
- `hive/work` label filtering
- The `blackboard` CLI can query across registered hives:

```bash
blackboard work list --level hub               # list work items in the hive
blackboard work list --level hub --skill security  # filter by skill
blackboard work list --level hub --mode competing  # competing proposals only
blackboard work list --level hub --available       # unclaimed only
```

**Mechanism 2: Spoke-based notification (requires spoke implementation)**

An operator's ivy-heartbeat can periodically check registered hives for new work items matching the operator's skills:

```yaml
# ivy-heartbeat check configuration
checks:
  - name: hive-work-scan
    schedule: "0 */4 * * *"  # every 4 hours
    command: "blackboard work list --level hub --skill {{operator.skills}} --available --json"
    condition: "result.count > 0"
    action:
      create_work_item:
        source: "hive"
        title: "New hive work: {{result.items[0].title}}"
```

This creates a local work item when matching hub work appears -- bridging hub discovery into the local blackboard where the operator's agents can see it.

**Mechanism 3: Hub aggregation feed (future)**

A hub-level feed that aggregates work items across federated hives:

```bash
blackboard work feed --level hub  # aggregated work from all connected hives
```

This requires hub federation (Hive Protocol open question) and is deferred to a later protocol version.

**What we explicitly do NOT build:**
- A centralized work marketplace. Hives are the discovery surfaces, not a global index.
- Push notifications to operators. Pull-based (operator checks) or spoke-based (heartbeat checks) only.
- AI-powered matching ("this work matches your skills"). The operator decides what to work on.

### 3. How are competing proposals handled?

**Decision:** Competing proposals are a work mode, not a special case. The work item declares `mode: competing` and the protocol follows pai-collab's competing-proposals SOP.**

**Workflow:**

1. **Posting:** The work item is created with `mode: competing` and labeled `hive/mode:competing`

2. **Proposals:** Each operator (or swarm) forks the hub repo and works independently. Proposals are submitted as PRs.

   ```markdown
   ## Proposal: [Title]

   **Operator:** @github-handle
   **Approach:** <brief description of the approach taken>
   **Trade-offs:** <what this approach optimizes for, what it sacrifices>

   <detailed proposal content>
   ```

3. **Evaluation:** The maintainer reviews all proposals against the acceptance criteria. Review comments are public.

4. **Selection:** The maintainer selects one proposal (or synthesizes from multiple):
   - Selected PR is merged
   - Unselected PRs are closed with a structured comment:

   ```markdown
   ## Proposal Outcome

   **Status:** Not selected
   **Reason:** <why this approach wasn't chosen>
   **Attribution:** Recorded in swarm.yaml. Trust credit at partial weight.
   ```

5. **Attribution:** All proposals earn trust credit:
   - Selected proposal: full weight
   - Unselected proposals: partial weight (0.3x default -- the work was real, it was an alternative path)
   - Reviews of proposals: standard review weight

**Why unselected proposals still earn trust:**
- The work was real. Code was written, designs were produced.
- Penalizing "losing" discourages participation in competing proposals.
- The trust system rewards effort and quality, not just luck of selection.
- The maintainer may synthesize from multiple proposals -- all inputs have value.

**What happens to unselected work:**
- PRs are closed but not deleted. They remain in git history.
- Operators retain their branches. The code is theirs.
- If the selected proposal later proves problematic, an unselected proposal can be revived.

### 4. How is completion verified?

**Decision:** Three-gate verification. All three gates must pass for completion.**

| Gate | Who | What they check | Mechanism |
|------|-----|----------------|-----------|
| **Gate 1: Automated** | CI pipeline | Tests pass, security scanning clean, schema valid | GitHub Actions on PR |
| **Gate 2: Peer review** | Reviewer role in swarm (or hive reviewers) | Code quality, correctness, alignment with acceptance criteria | PR review comments |
| **Gate 3: Maintainer merge** | Hive maintainer | Final approval, governance compliance, trust zone check | GitHub merge |

**Gate 1: Automated verification**

Runs on every PR to the hub:
- Tests pass (if the work item includes code)
- pai-secret-scanning clean (no secrets in the contribution)
- pai-content-filter scan (no prompt injection in markdown/documentation)
- Schema validation (spoke manifest, swarm.yaml if present)
- Acceptance criteria checklist (GitHub issue checkboxes)

**Gate 2: Peer review**

Required for all work items in `collaborative` and `competing` modes. Optional but recommended for `solo` mode.

Follows pai-collab's parallel-reviews SOP:
- Reviewers work independently (no discussion before individual review)
- Each reviewer submits structured findings:
  ```markdown
  ## Review: [Work Item Title]

  **Reviewer:** @github-handle
  **Verdict:** approve | request-changes | block
  **Summary:** <one paragraph>

  ### Findings
  - [ ] <finding 1>
  - [ ] <finding 2>
  ```
- Structured findings are recorded in the trust ledger

**Gate 3: Maintainer merge**

The maintainer has final authority. They verify:
- All automated checks pass
- Peer review is complete (at least one approval for collaborative mode)
- The contribution fits the hive's governance model
- The operator's trust zone permits this type of contribution

**Completion event chain:**

```
PR merged (Gate 3)
    -> Hub work item labeled `hive/status:completed`
    -> GitHub issue closed
    -> Attribution recorded in swarm.yaml (if swarm) or issue comment (if solo)
    -> Trust events emitted (see Trust Protocol)
    -> Operators' local blackboard items are updated via source_ref back-link
       (operator or ivy-heartbeat detects the closure and completes local items)
```

### 5. How does the local blackboard work item relate to the hive-level work item?

**Decision:** Soft reference via `source` + `source_ref`. No synchronization. The local blackboard is sovereign.**

The relationship is deliberately loose:

```
Hub Work Item (GitHub Issue)
  |
  | source_ref back-link (not enforced, not synchronized)
  |
  +-- Local Work Item (Operator A's blackboard)
  |     source: "hive"
  |     source_ref: "org/repo#42"
  |     status: claimed (local state, not hub state)
  |
  +-- Local Work Item (Operator B's blackboard)
  |     source: "hive"
  |     source_ref: "org/repo#42:task-2"
  |     status: completed (local state)
  |
  +-- (Operator C has no local item -- they haven't claimed anything)
```

**Design principles:**

1. **No synchronization.** The local blackboard does not poll the hub. The hub does not write to local blackboards. They are independent systems connected by a reference string.

2. **Local is authoritative for execution state.** If the local blackboard says the agent is working on a task, that's the truth. The hub's view (via spoke projection) is a snapshot -- it may be stale.

3. **Hub is authoritative for definition and completion.** The hub defines what the work is (acceptance criteria, mode, priority). The hub records when work is done (PR merged, issue closed).

4. **The spoke bridges the gap.** The spoke's `status.yaml` projects local execution state to the hub. This is the only sanctioned data flow from local to hub -- and it's a curated snapshot, not a live feed.

5. **One-to-many mapping.** A hub work item can spawn multiple local items (sub-tasks in a swarm decomposition). A local work item references at most one hub work item.

6. **Creation is manual or automated.** An operator can manually create a local work item referencing a hub item. Or ivy-heartbeat can detect new hub work and auto-create a local item. Either way, the operator decides.

**What this means in practice:**

- An operator can be "working on" a hub work item without the hub knowing (their spoke hasn't updated yet)
- Two operators can both have local items for the same hub work item (normal for swarms -- each has their sub-task)
- An operator can create local work items that don't reference any hub item (purely local work)
- If the hub work item is closed/cancelled, local items are NOT automatically closed -- the operator decides

---

## Work Mode Details

### Solo Mode

The simplest flow. One operator, one work item, no swarm.

```
1. Operator discovers work item (mode: solo)
2. Operator comments on issue to claim
3. Maintainer assigns (GitHub assign)
4. Hub label: hive/status:claimed
5. Operator creates local work item (source: "hive", source_ref: issue URL)
6. Operator's agents execute work locally
7. Operator submits PR
8. Three-gate verification
9. PR merged, issue closed, trust updated
```

**Claiming mechanism for solo mode:** GitHub issue assignment. First operator to express interest and get assigned by the maintainer claims the work. This is intentionally human-gated -- the maintainer approves, not a race condition.

If the assigned operator goes stale (no PR within a reasonable time), the maintainer can unassign and the work returns to `available`.

### Collaborative Mode

Multiple operators form a swarm. See [Swarm Protocol](swarm-protocol.md) for full details.

```
1. Work item posted (mode: collaborative)
2. Operators signal interest (structured comments)
3. Maintainer seals swarm (swarm.yaml committed)
4. Architect decomposes work into sub-tasks
5. Builders claim sub-tasks
6. Parallel execution (each operator works locally)
7. Convergence via PRs (dependency order)
8. Three-gate verification per PR
9. Final integration PR reviewed and merged
10. Swarm attribution recorded, trust updated, swarm dissolved
```

### Competing Mode

Multiple independent proposals. See Resolved Question 3 above.

```
1. Work item posted (mode: competing)
2. Operators work independently (no swarm formation)
3. Each submits a proposal PR
4. Maintainer evaluates all proposals
5. One selected and merged (or synthesis from multiple)
6. All proposals attributed (selected: full weight, others: partial weight)
7. Trust updated for all participants
```

---

## Work Item Lifecycle

```
                    +-----------+
                    | CREATED   |  maintainer posts issue with hive/work label
                    +-----+-----+
                          |
                          v
                    +-----------+
                    | AVAILABLE |  hive/status:available label
                    +-----+-----+
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
         +--------+  +---------+  +----------+
         | SOLO   |  | COLLAB  |  | COMPETING|
         | CLAIMED|  | SWARM   |  | OPEN     |
         +---+----+  +----+----+  +----+-----+
             |             |           |
             v             v           v
        +---------+   +---------+  +---------+
        | REVIEW  |   | REVIEW  |  | REVIEW  |   three-gate verification
        +----+----+   +----+----+  +----+-----+
             |             |           |
             v             v           v
        +----------+  +----------+  +----------+
        | COMPLETED|  | COMPLETED|  | COMPLETED|   PR merged, issue closed
        +----------+  +----------+  +----------+
                                         |
                                    TRUST UPDATED
```

Additional state transitions:

| From | To | Trigger |
|------|----|---------|
| CLAIMED | AVAILABLE | Operator abandons or goes stale; maintainer unassigns |
| CLAIMED | BLOCKED | External dependency identified |
| BLOCKED | CLAIMED | Dependency resolved |
| Any (except COMPLETED) | CANCELLED | Maintainer closes issue as not-planned |

---

## Integration with Other Protocols

| Protocol | Integration Point |
|----------|------------------|
| [Swarm Protocol](swarm-protocol.md) | Collaborative and competing work modes trigger swarm formation |
| [Trust Protocol](trust-protocol.md) | Work completion and review events feed trust scoring |
| [Spoke Protocol](spoke-protocol.md) | Local work status projected to hub via spoke status.yaml |
| [Hive Protocol](hive-protocol.md) | Hive governance determines who can post work and review |
| [Operator Identity](operator-identity.md) | Operator skills inform work discovery and role selection |

## Prior Art

- ivy-blackboard work items -- create, claim, release, complete, block (single-operator)
- pai-collab GitHub issues -- current work tracking mechanism (multi-operator)
- pai-collab [competing-proposals SOP](https://github.com/mellanon/pai-collab) -- multiple approaches to the same work
- Anthropic's agent teams research -- task decomposition, test-driven feedback, comparative oracle
