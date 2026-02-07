# Architecture

## The Protocol Stack

Like the Internet's TCP/IP model, The Hive defines a layered protocol stack where each layer has a clear responsibility, communicates through defined interfaces, and can evolve independently. Lower layers provide services to upper layers. Each layer can be implemented differently as long as it honours the contract.

```
┌─────────────────────────────────────────────────────────────────┐
│  APPLICATION    Swarm Protocol, Work Protocol, Skill Protocol   │
│                 How operators collaborate, find work, share     │
│                 capabilities. The "what people do" layer.       │
├─────────────────────────────────────────────────────────────────┤
│  IDENTITY       Operator Identity, Trust Protocol               │
│                 Who you are, what you've earned, what you can   │
│                 do. Portable across hives. The "who" layer.     │
├─────────────────────────────────────────────────────────────────┤
│  COORDINATION   Hive Protocol, Spoke Protocol                   │
│                 How blackboards connect — local to spoke to     │
│                 hub. The "how things talk" layer.               │
├─────────────────────────────────────────────────────────────────┤
│  SECURITY       Secret Scanning, Content Filtering, Audit       │
│                 Cross-cutting. Spans all layers. The "safety"   │
│                 layer. Not optional.                            │
├─────────────────────────────────────────────────────────────────┤
│  TRANSPORT      Git (commits, PRs, branches, forks)             │
│                 The reliable delivery mechanism. Immutable      │
│                 history, cryptographic integrity, distributed.  │
├─────────────────────────────────────────────────────────────────┤
│  STORAGE        Local Blackboard (SQLite), Spoke (YAML), Hub    │
│                 (Git repo). The persistence layer.              │
└─────────────────────────────────────────────────────────────────┘
```

### OSI/TCP-IP Parallels

| TCP/IP Layer | The Hive Layer | Parallel |
|-------------|---------------|----------|
| Application (HTTP, SMTP) | Swarm, Work, Skill protocols | What users actually do |
| Presentation/Session | Operator Identity, Trust | Who you are, session context |
| Transport (TCP/UDP) | Git (commits, PRs) | Reliable, ordered delivery |
| Network (IP) | Hive + Spoke protocols | Addressing and routing between nodes |
| Data Link/Physical | SQLite, YAML, Git repos | How data is stored and accessed |

**Key insight from TCP/IP:** The Internet succeeded because each layer has a simple, well-defined contract. HTTP doesn't care if you're on WiFi or Ethernet. Similarly, the Swarm Protocol doesn't care if the hub blackboard is GitHub or Discord — it talks to the Hive Protocol interface, which talks to the Transport layer (git).

**Key insight from OSI:** Security is cross-cutting (like TLS — it can operate at multiple layers). In The Hive, secret scanning operates at the transport layer (pre-commit), content filtering operates at the coordination layer (context loading), and audit operates at all layers.

## Agent Team Patterns

Research from Anthropic's agent team experiments (building a C compiler with autonomous agent swarms) validates several architectural choices in The Hive:

| Pattern | Anthropic Finding | The Hive Application |
|---------|------------------|---------------------|
| **File-based locking** | Agents claim work by writing lock files; git conflicts prevent duplicates | ivy-blackboard's atomic `work claim` (SQLite transactions) serves the same purpose at the local layer |
| **Role specialization** | Different agents assume distinct roles (optimizer, critic, documenter) rather than all solving the same problem | Swarm Protocol's role model: architect, builder, reviewer, documenter, tester |
| **Task decomposition** | Independent failing tests let each agent pick a different one; monolithic tasks need architectural splitting | Swarm Protocol requires an architect to decompose work before builders parallelize |
| **Test-driven feedback** | High-quality tests are the agent's primary guidance mechanism | Work Protocol's three-gate verification: CI tests, peer review, maintainer merge |
| **Comparative oracle** | A known-good reference enables concurrent debugging | Competing proposals: multiple independent approaches to the same problem, maintainer selects |
| **Eventual consistency** | Agents pull, merge, push — git's conflict detection handles coordination | Hub blackboard is eventually consistent (git-based, PR-mediated). Local blackboard is strongly consistent (SQLite). Spoke bridges them. |

**The critical difference:** Anthropic's experiment used unsupervised agents in a loop (`while true; do claude...; done`). The Hive puts humans in the loop — agents do work, humans review and govern. This is the architectural choice that makes trust possible.

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
6. **Signed by default.** All commits signed with operator's Ed25519 SSH key. Unsigned = untrusted.
7. **Open protocol.** The protocol is CC-BY-4.0. Implementations can vary.

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

## Security Architecture

Security is not a layer — it's a cross-cutting concern that spans all three blackboard layers. The Hive uses a **unified reflex pipeline** where security operations fire automatically at boundary crossings between layers.

### Commit Signing as Trust Foundation

All commits in The Hive are signed with the operator's Ed25519 SSH key. Git's native SSH signing (2.34+) provides the cryptographic identity layer — no custom PKI, no certificate authorities. Three git config commands + an `allowed-signers` file = complete identity.

The **allowed-signers file** (`.hive/allowed-signers`) is the trust anchor:
- Version-controlled in git (tracks who added whom, when)
- Maintainer-reviewed (adding an operator requires a reviewed PR)
- Git-native verification: `git log --show-signature`
- Revocation by removal (immediate for future commits)

**Four signature states:**

| State | Meaning | Action |
|-------|---------|--------|
| Signed + key in allowed-signers | Verified operator | Accept |
| Signed + key NOT in allowed-signers | Unknown operator | Requires onboarding |
| Unsigned commit | Untrusted | Reject at CI |
| Revoked key | Former operator | Reject |

### The Unified Reflex Pipeline

Four reflexes fire at boundary crossings between the three blackboard layers. Each reflex is a security checkpoint that operates at a specific transition point:

```
LOCAL BLACKBOARD                    SPOKE BLACKBOARD                    HUB BLACKBOARD
(operator's machine)                (.collab/ contract)                 (shared repository)

   commit ──┤ Reflex A ├──→ push ──┤ Reflex B ├──→ merge
             (outbound)              (outbound)
             Pre-commit              CI Gate

   load ←──┤ Reflex D ├──── pull ←──┤ Reflex C ├──── clone/fetch
             (inbound)               (inbound)
             Context Filter          Sandbox Enforcer
```

| Reflex | Boundary | Direction | What It Does | Implementation |
|--------|----------|-----------|-------------|----------------|
| **A: Pre-commit** | Local → Spoke | Outbound | Secret scanning + signing enforcement | gitleaks pre-commit hook |
| **B: CI Gate** | Spoke → Hub | Outbound | Signature verification + secret scanning + schema validation | GitHub Actions (4 ordered gates) |
| **C: Acquisition** | Hub → Spoke | Inbound | Quarantines external content in sandbox before access | [pai-content-filter](https://github.com/jcfischer/pai-content-filter) SandboxEnforcer hook |
| **D: Context Load** | Spoke → Local | Inbound | Scans content for prompt injection before LLM context | [pai-content-filter](https://github.com/jcfischer/pai-content-filter) ContentFilter hook |

**Per-operation reflex map — which reflexes fire for each git operation:**

| Operation | Reflexes | Why |
|-----------|----------|-----|
| `git commit` | A | Block secrets, enforce signing |
| `git push` | B | Verify identity, scan again, validate schema |
| `git clone` / `curl` | C | Quarantine to sandbox |
| Read file from sandbox | D | Scan before agent loads content |
| `gh pr create` | B | Full CI pipeline on PR |
| Skill install | C → D | Acquire to sandbox, then scan before loading |

### CI as State Machine

The hub enforces security through four ordered gates on every PR. This is **enforcement on the receiving end** (the hub), not the sending end (the contributor). Contributors need only git + SSH key — zero custom tooling.

```
Gate 1: IDENTITY    → All commits signed? Key in allowed-signers?
Gate 2: SECURITY    → Secret scanning passes? Content filter passes?
Gate 3: SCHEMA      → PROJECT.yaml valid? JOURNAL.md present? REGISTRY aligned?
Gate 4: GOVERNANCE  → Issue referenced? Journal updated? STATUS.md updated?
```

Gates 1-2 block on failure. Gates 3-4 warn. State is tracked entirely through git artifacts — no databases, no external services.

### Observable Setup Signals

Whether an operator has completed security onboarding is **binary and verifiable**, not self-reported:

| Signal | Evidence | How Verified |
|--------|----------|-------------|
| Signing setup complete | First commit signed with operator's key | CI Gate 1 checks against `allowed-signers` |
| Secret scanning active | Pre-commit hook catches secrets before CI | Observable through absence — if Layer 2 catches what Layer 1 should have, the signal is negative |
| Content filter active | Audit entries in operator's local blackboard | Self-attested via signed spoke manifest |

These signals feed into trust scoring as automatic positive feedback. An operator with verified setup + positive review history may qualify for zone promotion sooner — but promotion is always explicit and human-approved.

### Verification Asymmetry

Not all reflexes can be verified the same way. The hub's verification capability depends on whether evidence flows through git:

| Reflex | Verification | Method |
|--------|-------------|--------|
| **A (signing)** | Cryptographic proof | CI Gate 1 checks signatures against `allowed-signers` |
| **A (secret scanning)** | Negative signal | If CI catches secrets pre-commit should have blocked, Reflex A isn't running |
| **B (CI gate)** | Self-evident | The CI pipeline IS Reflex B — it's running by definition |
| **C (sandbox)** | Self-attested | Operator declares in signed `manifest.yaml` security section |
| **D (content filter)** | Self-attested | Operator declares in signed `manifest.yaml` security section |

**Design choice:** Self-attestation in a signed manifest is weaker than cryptographic proof, but stronger than a checkbox. The signature makes claims attributable — if an operator declares "content filter active" but a prompt injection incident occurs, the signed manifest is evidence of the gap. This trades verification strength for onboarding simplicity, consistent with "enforcement on the receiving end."

### Implementation Detail

**Outbound defense (Reflexes A + B):**
- [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) provides the scanning engine
- gitleaks with 11 custom AI provider patterns (Anthropic, OpenAI, ElevenLabs, etc.) + ~150 built-in patterns
- Pre-commit hook (Reflex A) blocks locally; CI gate (Reflex B) catches anything that slips through

**Inbound defense (Reflexes C + D):**
- [pai-content-filter](https://github.com/jcfischer/pai-content-filter) provides both hooks
- **SandboxEnforcer** (Reflex C): PreToolUse hook intercepts `git clone`, `curl`, `wget` and redirects to sandbox directory
- **ContentFilter** (Reflex D): PreToolUse hook scans Read/Glob/Grep targeting sandbox. 34 deterministic regex patterns, no LLM classification. Decisions: ALLOWED, HUMAN_REVIEW, BLOCKED
- Fail-closed design: any hook error → block (exit code 2)

### Security as Prerequisite

These aren't optional add-ons. They're prerequisites for collaboration:
- **Operators must enable signing and install secret scanning** before their first contribution
- **Content filtering fires automatically** when agents load shared content
- **All trust zone transitions** (untrusted → trusted → maintainer) pass through these gates
- **Maintainers are not exempt** — scanning applies to all contributors regardless of trust zone
- **Earned, not claimed** — a signed first PR is stronger evidence than a checked checkbox

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
| Spoke contract | Yes (spoke-protocol.md) | No | Schema designed, identity section specified. Implementation pending. |
| Hub coordination | Partially (pai-collab SOPs) | Yes | Working but not formalized as a protocol. |
| Swarm formation | No | No | Core protocol gap. |
| Outbound security | Yes (pai-secret-scanning) | Yes | Shipped. Reflex A (pre-commit) + Reflex B (CI gate). 11 custom AI patterns. |
| Inbound security | Yes (pai-content-filter) | Yes | Shipped. Reflex C (sandbox enforcer) + Reflex D (content filter). 492 tests. |
| Commit signing | Yes (operator-identity.md) | Yes | Ed25519 SSH signing, allowed-signers trust anchor, CI Gate 1 verification. |
| CI enforcement | Yes (hive-protocol.md) | Partially | Gate 1 (Identity) operational on pai-collab. Gates 2-4 specified. |
| Trust scoring | Yes (trust-protocol.md) | Partially | Binary zones exist. Quantitative scoring + observable setup signals specified. |
| Operator identity | Yes (operator-identity.md) | Partially | SSH signing identity operational. Portable profiles and three-tier schema specified. |
| Work discovery | No | No | Work items are local. No cross-hive discovery. |
| Skill distribution | No | No | Skills exist locally. No network distribution. |
