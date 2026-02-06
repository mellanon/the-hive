# Implementation Plan

> **Status:** Draft
> **Updated:** 2026-02-06

## PAI Principles Alignment

The Hive is built on PAI infrastructure. Every design decision should align with the [PAI Founding Principles](https://github.com/danielmiessler/PAI). This review maps each principle to The Hive's architecture.

### Alignment Review

| # | Principle | Alignment | Notes |
|---|-----------|-----------|-------|
| 1 | **User Centricity** | **Aligned** | Operator is always at centre. Agents invisible to protocol. Human-in-the-loop at every gate. |
| 2 | **Foundational Algorithm** | N/A | The Algorithm is PAI's execution loop. The Hive is infrastructure the Algorithm operates on. |
| 3 | **Clear Thinking First** | **Aligned** | Protocol specs are design-first. All open questions resolved before building. |
| 4 | **Scaffolding > Model** | **Aligned** | Protocol stack is model-agnostic. No AI model mentioned anywhere in the architecture. |
| 5 | **Deterministic Infrastructure** | **Aligned** | YAML schemas, git-based audit trails, regex-based security patterns — all deterministic. |
| 6 | **Code Before Prompts** | **Aligned** | CLI commands, gitleaks rules, schema validators. Code solutions before AI solutions. |
| 7 | **Spec / Test / Evals First** | **Gap** | Three-gate verification exists for work. But no conformance tests for the protocols themselves. Need protocol test suites. |
| 8 | **UNIX Philosophy** | **Aligned** | Each component does one thing: blackboard=state, heartbeat=time, scanning=secrets, filter=content. Text interfaces (YAML, JSONL). Composable via pipes and CLI. |
| 9 | **ENG / SRE Principles** | **Aligned** | Git-versioned, CI gates, audit trails, monitoring (`observe` command), crash recovery (`sweep`). |
| 10 | **CLI as Interface** | **Aligned** | `blackboard` CLI with `--level` flag is the primary interface. Web is secondary. |
| 11 | **Goal → Code → CLI → Prompts → Agents** | **Aligned** | The hierarchy: protocol spec (goal) → TypeScript implementation (code) → blackboard CLI → agent interaction. |
| 12 | **Skill Management** | **Aligned** | Skill Protocol follows PAI's skill structure exactly (SKILL.md, workflows, components). Adds `skill-manifest.yaml` for network distribution. |
| 13 | **Memory System** | **Partial** | Local blackboard is memory (event log, work history). Spoke projects memory upward. But no explicit learning/improvement loop at the protocol level. |
| 14 | **Agent Personalities** | N/A | Agents are invisible to the protocol by design. Operators bring their own agents (specialized or general). |
| 15 | **Science as Meta-Loop** | **Partial** | Trust scoring iterates. But no explicit hypothesis-test cycle in the collaboration protocols. |
| 16 | **Permission to Fail** | **Aligned** | Untrusted zone is a low-risk sandbox. New operators can experiment without high-stakes consequences. Abandoned work is handled gracefully (staleness protocol). |

### Gaps to Address

1. **Protocol conformance tests** (Principle 7) — need test suites that validate a hive implementation actually follows the protocol
2. **Learning loop** (Principle 13) — the protocol should specify how hives improve over time (retrospectives, governance evolution, SOP iteration)

## The Interface Question: AX vs Web UI

There are two ways operators interact with The Hive:

### Option A: Agent as the Interface (AX)

The operator's AI agent IS the interface. You don't open a web browser — you talk to your agent, and it interacts with the hive on your behalf.

```
Operator: "What's happening in the security-tools hive?"
Agent:    → reads spoke status from hub
          → reads work items from GitHub
          → reads trust zone from CONTRIBUTORS.yaml
          → presents: "3 active work items, 1 swarm forming,
             you were promoted to trusted yesterday"
```

**Pros:**
- Perfectly aligned with PAI Principles 10-11 (CLI/agent first)
- No new infrastructure to build — the agent already has the tools (git, gh, Read, Grep)
- The protocol data (YAML, git, GitHub) IS the interface — the agent reads it natively
- Every operator already has an agent — it's the whole point of The Hive

**Cons:**
- Less "discoverable" for new operators
- No ambient awareness (you have to ask)
- Harder to browse visually

### Option B: Web Dashboard

A social-network-like web interface (as described in `scenarios/operator-experience.md`).

**Pros:**
- Visual, browsable, ambient — see everything at a glance
- Discoverable — new operators can explore without an agent
- Shareable — links, screenshots, embeds

**Cons:**
- Requires new infrastructure to build and host
- Contradicts PAI Principle 10 (CLI > GUI)
- Another surface to maintain and secure

### Decision: Agent-First, Web-Optional

**The agent is the primary interface.** This follows PAI's architecture exactly:

```
Goal → Code → CLI → Prompts → Agents
                ↑                  ↑
           blackboard CLI    operator's agent
           (deterministic)   (reads hive data)
```

The `blackboard` CLI is the deterministic tool layer. The agent uses it. The operator talks to the agent.

A web dashboard is an **optional observability layer** — like ivy-blackboard's `blackboard serve` command, which spins up a web dashboard for local state. The same pattern can extend to spoke and hub levels:

```bash
blackboard serve --level local    # Local dashboard (exists today in ivy-blackboard)
blackboard serve --level spoke    # Spoke status dashboard
blackboard serve --level hub      # Hub dashboard (feed, work, members)
```

**Build order:** Agent interaction first (it works today with existing tools). Web dashboard later (when there's enough multi-operator activity to make it valuable).

## Current State: What Exists

| Component | Repo | Tech | CLI | Distribution | Status |
|-----------|------|------|-----|-------------|--------|
| **Local blackboard** | [ivy-blackboard](https://github.com/jcfischer/ivy-blackboard) | TypeScript + Bun | `blackboard` (compiled binary) | `~/bin/blackboard` via build.sh | Shipped |
| **Heartbeat/dispatch** | [ivy-heartbeat](https://github.com/jcfischer/ivy-heartbeat) | TypeScript + Bun | `bun src/cli.ts` | Source + bun | Shipped |
| **Hub (Hive Zero)** | [pai-collab](https://github.com/mellanon/pai-collab) | Markdown + YAML + GitHub Actions | `gh` (GitHub CLI) | Git repo | Operational |
| **Secret scanning** | [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | TOML config + gitleaks | `gitleaks` | One-command installer | Shipped |
| **Content filter** | [pai-content-filter](https://github.com/jcfischer/pai-content-filter) | TypeScript + Bun + Zod | `content-filter` | Hook symlinks | Shipped |
| **Protocol spec** | [the-hive](https://github.com/mellanon/the-hive) | Markdown | — | Git repo | In progress |

### Shared Tech Stack

All implementation repos share:
- **Language:** TypeScript
- **Runtime:** Bun
- **Schema validation:** Zod
- **CLI framework:** Commander.js
- **Data format:** YAML + JSON + SQLite
- **Distribution:** Git-based (clone + install)

This consistency is a strength — the unified CLI can share the same toolchain.

## Implementation Gaps

| Protocol | Specified | Built | Gap |
|----------|-----------|-------|-----|
| **Local blackboard** | Yes | Yes (ivy-blackboard) | None — shipped |
| **Heartbeat/dispatch** | Yes | Yes (ivy-heartbeat) | None — shipped |
| **Spoke contract** | Yes (Issue #80) | No | Need `blackboard --level spoke` commands: `init`, `status`, `validate` |
| **Hub coordination** | Partially | Yes (pai-collab) | Need `blackboard --level hub` commands: `init`, `pull`, `registry`, `status` |
| **Unified CLI** | Designed | Partially (local only) | Need `--level spoke` and `--level hub` in the blackboard binary |
| **Swarm formation** | Yes (protocol spec) | No | Need swarm.yaml generation, role claiming, lifecycle management |
| **Work discovery** | Yes (protocol spec) | No | Need cross-hive work feed aggregation |
| **Skill distribution** | Yes (protocol spec) | No | Need `blackboard skills install/list/search` commands |
| **Operator identity** | Yes (protocol spec) | No | Need `operator.yaml` generation, identity linking, verification |
| **Trust scoring** | Partially | Partially (zones only) | Need quantitative scoring, vouching mechanism |
| **Vouching** | Yes (protocol spec) | No | Need vouch CLI commands, accountability tracking |
| **Secret scanning** | Yes | Yes (pai-secret-scanning) | None — shipped |
| **Content filter** | Yes | Yes (pai-content-filter) | None — shipped |
| **Web dashboard** | Conceptual | Partially (local only) | `blackboard serve --level local` exists. Spoke/hub dashboards don't. |

## Bundling Model: PAI Packs for The Hive

Following PAI's distribution model (Packs + Bundles), The Hive would be distributed as:

### Infrastructure Packs

| Pack | Contents | Dependencies |
|------|----------|-------------|
| **hive-blackboard** | Unified `blackboard` CLI with `--level local\|spoke\|hub` | Bun |
| **hive-heartbeat** | Autonomous dispatch + scheduling | hive-blackboard |
| **hive-security-outbound** | Pre-commit hooks + CI gate config + gitleaks rules | gitleaks |
| **hive-security-inbound** | Content filter + sandbox enforcer hooks | hive-blackboard |

### Operator Packs

| Pack | Contents | Dependencies |
|------|----------|-------------|
| **hive-spoke** | Spoke contract: `.collab/` scaffolding, manifest + status generation, validation | hive-blackboard |
| **hive-identity** | `operator.yaml` generation, identity linking, verification commands | hive-spoke |
| **hive-skills** | Skill installation, search, publishing via `blackboard skills` commands | hive-blackboard |

### Hub Packs

| Pack | Contents | Dependencies |
|------|----------|-------------|
| **hive-hub** | Hub scaffolding: `hive.yaml`, REGISTRY, trust model, GitHub Actions workflows | hive-blackboard |
| **hive-swarm** | Swarm formation: `swarm.yaml` generation, role claiming, lifecycle | hive-hub, hive-spoke |
| **hive-trust** | Trust scoring, vouching, zone management | hive-hub |

### The Hive Bundle

A meta-bundle that installs everything an operator needs to join a hive:

```
The Hive Bundle (operator starter kit)
├── hive-blackboard       ← unified CLI
├── hive-heartbeat        ← autonomous dispatch
├── hive-security-outbound ← secret scanning
├── hive-security-inbound  ← content filter
├── hive-spoke            ← spoke contract
├── hive-identity         ← operator profile
└── hive-skills           ← skill management
```

Installation:
```bash
# Option 1: Full bundle (recommended)
git clone https://github.com/mellanon/the-hive.git
cd the-hive && bun run install.ts

# Option 2: Individual packs
blackboard pack install hive-spoke
blackboard pack install hive-identity
```

### How Packs Relate to Existing Repos

| Pack | Built from | How |
|------|-----------|-----|
| **hive-blackboard** | ivy-blackboard + new spoke/hub commands | Extend existing CLI with `--level` support |
| **hive-heartbeat** | ivy-heartbeat | Repackage as installable pack |
| **hive-security-outbound** | pai-secret-scanning | Repackage installer + rules as pack |
| **hive-security-inbound** | pai-content-filter | Repackage hooks + patterns as pack |
| **hive-spoke** | New (from spoke-protocol.md spec) | Build spoke contract commands |
| **hive-identity** | New (from operator-identity.md spec) | Build identity management |
| **hive-hub** | Extracted from pai-collab patterns | Formalize as reproducible scaffolding |
| **hive-swarm** | New (from swarm-protocol.md spec) | Build swarm lifecycle |
| **hive-trust** | New (from trust-protocol.md spec) | Build scoring + vouching |
| **hive-skills** | New (from skill-protocol.md spec) | Build skill distribution |

## Interaction Model

### The `blackboard` CLI (Unified)

The operator interacts with The Hive through the `blackboard` CLI — one tool, three levels, explicit `--level` flag:

```bash
# LOCAL LEVEL — operator's private coordination
blackboard status --level local          # Show local blackboard health
blackboard work list --level local       # List local work items
blackboard agent list --level local      # List active agents
blackboard observe --level local         # Show event log
blackboard serve --level local           # Web dashboard (localhost:3141)

# SPOKE LEVEL — operator's network projection
blackboard init --level spoke            # Create .collab/ with manifest + status
blackboard status --level spoke          # Show spoke projection status
blackboard validate --level spoke        # Validate manifest + status schemas
blackboard publish --level spoke         # Push status to hub

# HUB LEVEL — community coordination
blackboard init --level hub              # Create hive.yaml + scaffolding
blackboard status --level hub            # Show hive health (aggregated spokes)
blackboard registry --level hub          # List registered projects/operators
blackboard work list --level hub         # List hub work items
blackboard work feed --level hub         # Aggregated work feed
blackboard serve --level hub             # Web dashboard (hive view)

# CROSS-LEVEL COMMANDS
blackboard skills install <org/repo>     # Install skill from network
blackboard skills list --level local     # Show installed skills
blackboard skills search --level hub     # Search hive skill registry

blackboard identity init                 # Create operator.yaml
blackboard identity link --provider github  # Link identity provider
blackboard identity verify               # Verify linked identities

blackboard swarm list --level hub        # List active swarms
blackboard swarm join <work-id>          # Express interest in work
blackboard swarm status <swarm-id>       # Show swarm progress

blackboard trust status                  # Show your trust across hives
blackboard trust vouch <handle>          # Vouch for another operator

blackboard inbox                         # Show notifications (local inbox)
```

### The Agent Interface (AX)

The operator's AI agent uses these same CLI commands — plus direct git/GitHub access — to interact with hives on the operator's behalf. The agent IS the interface:

```
Operator talks to agent
    ↓
Agent reads protocol data (YAML, git, GitHub API)
Agent runs blackboard CLI commands
Agent presents results in natural language
    ↓
Operator sees their hive, their work, their trust
```

This means The Hive works out of the box for any operator running PAI — their agent already has access to git, GitHub CLI, and Bun. Install the blackboard CLI, and the agent can interact with the full protocol stack.

### What the Agent Can Do Today (No New Code)

Even before building the unified CLI, an operator's agent can already:

| Action | How (existing tools) |
|--------|---------------------|
| Browse hive | Read `hive.yaml` from any hive repo |
| See work items | `gh issue list --repo <hive>` |
| Claim work | `gh issue edit --add-assignee` |
| Check spoke status | Read `.collab/status.yaml` |
| Run local blackboard | `blackboard status --level local` |
| Check trust | Read `CONTRIBUTORS.yaml` from hive |
| Install skill | `git clone` + symlink |
| Scan for secrets | `gitleaks protect --staged` |
| Submit work | Fork + PR via `gh pr create` |

**The protocol data IS the interface.** The CLI makes it ergonomic. The web dashboard makes it visual. But the data is always git + YAML + GitHub — readable by any agent.

## Build Order

### Phase 1: Spoke Contract (highest leverage)

The spoke is the missing link between local (shipped) and hub (operational). Building it connects the layers.

**Deliverables:**
- `blackboard init --level spoke` — scaffolds `.collab/manifest.yaml` + `status.yaml`
- `blackboard status --level spoke` — generates status from local blackboard
- `blackboard validate --level spoke` — validates schemas (Zod)
- `blackboard publish --level spoke` — pushes status to hub (git commit + push)

**Built from:** Spoke protocol spec + ivy-blackboard's existing TypeScript/Bun/Commander stack.

### Phase 2: Hub Commands

Formalize pai-collab's patterns as reproducible CLI commands.

**Deliverables:**
- `blackboard init --level hub` — creates `hive.yaml` + scaffolding
- `blackboard registry --level hub` — list/manage registered projects and operators
- `blackboard status --level hub` — aggregate spoke statuses

### Phase 3: Operator Identity

Enable multi-hive identity.

**Deliverables:**
- `blackboard identity init` — create `operator.yaml`
- `blackboard identity link` — link identity providers
- `blackboard identity verify` — run verification flow

### Phase 4: Skill Distribution

Network skill sharing.

**Deliverables:**
- `blackboard skills install/list/search/remove`
- Security gates (outbound scanning on publish, inbound filtering on install)

### Phase 5: Swarm + Trust + Vouching

The collaboration and reputation layer.

**Deliverables:**
- Swarm lifecycle commands
- Trust scoring
- Vouching mechanism

### Phase 6: Observability

Extend PAI Signal's observability framework to hive-level events.

**Deliverables:**
- Hive event types (join, leave, work lifecycle, swarm lifecycle, trust feedback)
- JSONL event storage per level (local/spoke/hub)
- Pre-built query scripts (`trust-trends.sh`, `swarm-activity.sh`, `operator-feedback.sh`)
- `blackboard observe --level hub` command
- PII scrubbing on all events (from Signal's 17-pattern sanitizer)

**Built from:** PAI Signal's event schema design (TypeScript types, factory functions, JSONL logging, PII scrubbing). Signal is designed and merged into PAI but not yet deployed — use the event schema and patterns as reference, not a running dependency. Same three-layer model: CLI queries (Layer 0) → scripts (Layer 1) → Grafana dashboard (Layer 2).

### Phase 7: Web Dashboard (Optional)

Extend `blackboard serve` to spoke and hub levels.

## Tech Stack Decision

All new code follows the established stack:

| Choice | Rationale |
|--------|-----------|
| **TypeScript** | All existing components use it. PAI ecosystem standard. |
| **Bun** | Runtime for ivy-blackboard and ivy-heartbeat. Compiled binaries via `bun build --compile`. |
| **Commander.js** | CLI framework used by both existing tools. |
| **Zod** | Schema validation used by both existing tools. |
| **SQLite** | Local storage (bun:sqlite). |
| **YAML** | Protocol data format (spoke, hub). |
| **Git** | Transport layer. No custom wire protocols. |
