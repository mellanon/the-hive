# Architecture

## The Blackboard Model

The Hive protocol is built on the **blackboard pattern** (Hayes-Roth, 1985) — a shared coordination surface where independent agents read and write without direct coupling. This pattern appears at every level of the architecture, qualified by scope:

| Blackboard | Scope | Storage | Access | Implementation |
|------------|-------|---------|--------|----------------|
| **Local Blackboard** | One operator, one machine | SQLite | Real-time, multi-agent | ivy-blackboard |
| **Spoke Blackboard** | One operator → network | YAML files | Snapshot, published | `.collab/` contract |
| **Hub Blackboard** | Community, multi-operator | Git repository | Async, human-reviewed | pai-collab |

**The pattern is the same at every level** — a surface where participants coordinate without direct coupling. What changes is scope, storage, and access model. A local blackboard is private and real-time. A spoke blackboard is a curated projection. A hub blackboard is shared and governed.

```
LOCAL BLACKBOARD          Operator's private surface.
(ivy-blackboard)          SQLite, real-time, multi-agent.
        │
        │  spoke projects selected state upward
        ▼
SPOKE BLACKBOARD          Operator's network projection.
(.collab/)                manifest.yaml + status.yaml, curated snapshot.
        │
        │  hub aggregates spoke blackboards
        ▼
HUB BLACKBOARD            Community's shared surface.
(pai-collab)              Git-based, multi-operator, human-reviewed.
```

## The Three-Layer Model

The Hive protocol defines three coordination layers. Each layer is independent — an operator can use just the local blackboard without connecting to any hub. The layers compose upward when operators want to collaborate.

### Local Layer (Local Blackboard)

The operator's private coordination surface. Manages agent sessions, work items, heartbeats, and events on a single machine.

**Responsibilities:**
- Agent lifecycle (register, heartbeat, deregister, crash recovery)
- Work item management (create, claim, release, complete)
- Autonomous dispatch (periodic checks, condition evaluation, agent wake)
- Event audit trail (append-only, queryable)
- Observability (CLI, web dashboard, SSE streams)

**Properties:**
- Fully standalone — works without network
- SQLite-based — no infrastructure dependencies
- Multi-agent safe — atomic claims, stale detection, auto-sweep

**Reference implementation:** [ivy-blackboard](https://github.com/jcfischer/ivy-blackboard) (state) + [ivy-heartbeat](https://github.com/jcfischer/ivy-heartbeat) (time/dispatch)

### Spoke Layer (Spoke Blackboard)

The operator's projection to the network. A lightweight contract that exposes selected local blackboard state to a hub without coupling local internals to the network.

**Responsibilities:**
- Manifest: who you are (identity, project, license, capabilities)
- Status: what's happening (phase, progress, test results, git state)
- Publishing: push status to hub on change or schedule

**Properties:**
- Two files in `.collab/` at the spoke repo root
- Human-maintained manifest + auto-generated status
- Hub never sees local internals — only the projection
- Status is a snapshot, not a live feed

**Schema:** See [spoke-protocol.md](protocols/spoke-protocol.md)

**Design source:** [pai-collab Issue #80](https://github.com/mellanon/pai-collab/issues/80)

### Hub Layer (Hub Blackboard)

The shared coordination surface for a community. Where operators find each other, form swarms, and build trust through verified contributions.

**Responsibilities:**
- Project registry (what work exists, who maintains it)
- Trust model (zones, verification, reputation)
- Governance (SOPs, contribution protocols, review standards)
- Spoke aggregation (pull status from registered spokes)
- Work coordination (posting, discovery, swarm formation)

**Properties:**
- GitHub-native (fork-and-PR as coordination protocol)
- Human-in-the-loop (all changes reviewed before merge)
- Trust zones (untrusted → trusted → maintainer)
- Extensible via SOPs

**Reference implementation:** [pai-collab](https://github.com/mellanon/pai-collab) ("Hive Zero")

## How the Layers Connect

```
Operator A                           Operator B
┌──────────────┐                     ┌──────────────┐
│ LOCAL        │                     │ LOCAL        │
│ blackboard   │                     │ blackboard   │
│ heartbeat    │                     │ heartbeat    │
└──────┬───────┘                     └──────┬───────┘
       │ .collab/                           │ .collab/
       │ manifest + status                  │ manifest + status
┌──────┴───────┐                     ┌──────┴───────┐
│ SPOKE A      │                     │ SPOKE B      │
└──────┬───────┘                     └──────┴───────┘
       │                                    │
       └────────────┬───────────────────────┘
                    │
             ┌──────┴───────┐
             │ HUB          │
             │ (a hive)     │
             │ registry     │
             │ trust model  │
             │ governance   │
             │ work items   │
             └──────────────┘
```

### Key Design Principles

1. **Local-first.** Everything works offline. Network is additive, not required.
2. **Projection, not exposure.** Spokes publish what they choose. Hubs never reach into local state.
3. **Git-native.** The coordination protocol is fork-and-PR. No custom wire protocols.
4. **Human-in-the-loop.** Agents do work. Humans review, approve, and govern.
5. **Trust through work.** Reputation is earned from verified contributions, not claimed.
6. **Open protocol.** The protocol is MIT. Implementations can vary.

## The Blackboard CLI

`blackboard` is a single CLI that operates at all three blackboard levels. The level is always explicit via the `--level` flag — no auto-detection, no ambiguity.

```
blackboard <command> --level local|spoke|hub
```

### Design Decision

One CLI, explicit level. This was chosen over:
- ~~Auto-detect by directory context~~ — too magical, ambiguous in repos that have both `.collab/` and local blackboard
- ~~Separate CLIs per level~~ — more to install, harder to discover, splits documentation

### Commands by Level

| Level | Commands | What they operate on |
|-------|----------|---------------------|
| `--level local` | `init`, `status`, `work`, `agent`, `observe`, `sweep`, `serve` | Local blackboard (`~/.pai/blackboard/local.db`) |
| `--level spoke` | `init`, `status`, `validate` | Spoke blackboard (`.collab/manifest.yaml` + `status.yaml`) |
| `--level hub` | `init`, `pull`, `registry`, `status` | Hub blackboard (REGISTRY.md, spoke aggregation) |

### Analogy: RabbitMQ Federation

The blackboard model shares structural similarities with RabbitMQ's messaging architecture:

| RabbitMQ | The Hive | Parallel |
|----------|----------|----------|
| **Queue** | Local blackboard | Where work sits, agents consume from it |
| **Exchange/Binding** | Spoke blackboard | Routes selected state from local → hub |
| **Broker** | Hub blackboard | Aggregates across multiple spokes |
| **Federation** | Multi-hub (future) | Hubs sync with each other |

Like RabbitMQ, each level can operate independently. A local blackboard doesn't need a hub. A hub doesn't need to federate. But the protocol supports composition when you need it.

## Security Boundary

Security is not a layer — it's a cross-cutting concern that spans all three layers. The Hive inherits a defense-in-depth model from pai-collab's trust architecture.

### The Six Defense Layers

```
OUTBOUND DEFENSE (secrets leaving)
──────────────────────────────────────────────────
Layer 1: Pre-commit scanning          ← Operator's machine
         Catches secrets before they enter git history.
         Tool: gitleaks with custom PAI rules (11 AI provider patterns).
         Blocks commits containing API keys, credentials, .env values.

Layer 2: CI gate                      ← Repository level
         Catches anything Layer 1 missed (bypassed hooks, new patterns).
         Runs on every push and PR via GitHub Actions.

Layer 3: Fork and pull request        ← Structural boundary
         Human review before merge. Git's native isolation model.

INBOUND DEFENSE (threats entering)
──────────────────────────────────────────────────
Layer 4: Content trust boundary       ← Agent context loading
         Scans inbound markdown for prompt injection before LLM context.
         34 deterministic patterns (instruction overrides, PII, encoding evasion).
         No LLM classification — auditable regex + schema validation.

Layer 5: Tool restrictions            ← Agent sandbox
         Untrusted content = restricted agent.
         No Bash, no Write, no WebFetch. Read-only MCP access.
         Quarantine sandbox for acquired external content.

Layer 6: Audit trail                  ← Observability
         Append-only JSONL log of every content loading decision.
         Human override with recorded reasoning.
         Enables detection of slow-burn manipulation patterns.
```

### How Security Maps to Layers

| Defense | Protects | Where it runs | Implementation |
|---------|----------|---------------|----------------|
| **Outbound** (Layers 1-2) | Prevents secrets leaking into shared repos | Local + Hub | [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) |
| **Structural** (Layer 3) | Isolates contributions for review | Hub | Git fork-and-PR (built-in) |
| **Inbound** (Layers 4-5) | Prevents prompt injection from compromising agents | Local + Hub | [pai-content-filter](https://github.com/jcfischer/pai-content-filter) |
| **Observability** (Layer 6) | Detects and records security events | All layers | pai-content-filter audit trail + ivy-blackboard event log |

### Outbound: Secret Scanning

**Problem:** Operators work with private infrastructure — API keys, voice credentials, personal paths. Contributing to a hive requires separating public code from private secrets. Without automated scanning, every contribution is a potential exposure event.

**Solution:** [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) provides two automated gates:
- **Pre-commit hook** — blocks commits containing secrets on the operator's machine
- **CI gate** — scans PRs at the repository level before merge

**Coverage:** 11 custom AI provider patterns (Anthropic, OpenAI, ElevenLabs, Telegram, Replicate, HuggingFace, Groq) + ~150 built-in gitleaks patterns (AWS, GCP, Azure, SSH keys, JWTs).

**Position in workflow:** Runs during the "Contrib Prep" phase — after code is built and hardened, before it's submitted for review.

### Inbound: Content Filtering

**Problem:** When agents load content from shared repositories (the blackboard pattern), malicious instructions can be hidden in markdown — prompt injection attacks that cause the reviewing agent to execute unintended actions.

**Solution:** [pai-content-filter](https://github.com/jcfischer/pai-content-filter) provides four-layer defense:
1. **Pattern matching** — 34 deterministic regex patterns detecting instruction overrides, PII, encoding evasion
2. **Architectural isolation** — flagged content triggers tool-restricted agent mode (read-only, no shell, no network)
3. **Audit trail** — every content loading decision logged with source, patterns matched, and reasoning
4. **Sandbox enforcer** — PreToolUse hook intercepts acquisition commands (`git clone`, `curl`) and redirects to sandbox for scanning before agent access

**Design choice:** Deterministic regex, not LLM classification. Auditable, reproducible, no recursive AI calls.

**Coverage:** Direct prompt injection, encoded payloads (base64, Unicode, hex), format marker exploits (Llama/Mistral delimiters), authority impersonation, incremental drift detection.

### Security as Prerequisite

These aren't optional add-ons. They're prerequisites for collaboration:
- **Operators must install secret scanning** before their first contribution
- **Content filtering activates automatically** when agents load shared content
- **All trust zone transitions** (untrusted → trusted → maintainer) pass through these gates
- **Maintainers are not exempt** — scanning applies to all contributors regardless of trust zone

## Ecosystem Map

| Component | Role | Maintainer | Status |
|-----------|------|-----------|--------|
| [the-hive](https://github.com/mellanon/the-hive) | Protocol specification | @mellanon | In progress |
| [pai-collab](https://github.com/mellanon/pai-collab) | Hub implementation ("Hive Zero") | @mellanon | Operational |
| [ivy-blackboard](https://github.com/jcfischer/ivy-blackboard) | Local layer (state) | @jcfischer | Shipped |
| [ivy-heartbeat](https://github.com/jcfischer/ivy-heartbeat) | Local layer (time/dispatch) | @jcfischer | Shipped |
| [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | Security (outbound) | @jcfischer | Shipped |
| [pai-content-filter](https://github.com/jcfischer/pai-content-filter) | Security (inbound) | @jcfischer | Shipped |
| Spoke contract | Spoke layer | TBD | Designed ([#80](https://github.com/mellanon/pai-collab/issues/80)) |

## What's Specified vs What's Built

| Protocol | Specified | Built | Notes |
|----------|-----------|-------|-------|
| Local coordination | Yes (ivy-blackboard) | Yes | Shipped. SQLite-based agent coordination. |
| Autonomous dispatch | Yes (ivy-heartbeat) | Yes | Shipped. launchd-scheduled checks + dispatch. |
| Spoke contract | Yes (Issue #80) | No | Schema designed, security reviewed. Implementation pending. |
| Hub coordination | Partially (pai-collab SOPs) | Yes | Working but not formalized as a protocol. |
| Swarm formation | No | No | Core protocol gap. |
| Outbound security | Yes (pai-secret-scanning) | Yes | Shipped. Pre-commit + CI gate. 11 custom AI patterns. |
| Inbound security | Yes (pai-content-filter) | Yes | Shipped. 34 patterns + sandbox + audit trail. 389 tests. |
| Trust scoring | Partially (trust zones) | Partially | Binary zones exist. Quantitative scores do not. |
| Work discovery | No | No | Work items are local. No cross-hive discovery. |
| Skill distribution | No | No | Skills exist locally. No network distribution. |
| Operator identity | No | No | GitHub identity only. No verified profiles. |
