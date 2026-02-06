# Architecture

## The Three-Layer Model

The Hive protocol defines three coordination layers. Each layer is independent — an operator can use just the local layer without connecting to any hub. The layers compose upward when operators want to collaborate.

### Local Layer

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

### Spoke Layer

The operator's projection to the network. A lightweight contract that exposes selected local state to a hub without coupling local internals to the network.

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

### Hub Layer

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

## Ecosystem Map

| Component | Role | Maintainer | Status |
|-----------|------|-----------|--------|
| [the-hive](https://github.com/mellanon/the-hive) | Protocol specification | @mellanon | In progress |
| [pai-collab](https://github.com/mellanon/pai-collab) | Hub implementation ("Hive Zero") | @mellanon | Operational |
| [ivy-blackboard](https://github.com/jcfischer/ivy-blackboard) | Local layer (state) | @jcfischer | Shipped |
| [ivy-heartbeat](https://github.com/jcfischer/ivy-heartbeat) | Local layer (time/dispatch) | @jcfischer | Shipped |
| Spoke contract | Spoke layer | TBD | Designed ([#80](https://github.com/mellanon/pai-collab/issues/80)) |

## What's Specified vs What's Built

| Protocol | Specified | Built | Notes |
|----------|-----------|-------|-------|
| Local coordination | Yes (ivy-blackboard) | Yes | Shipped. SQLite-based agent coordination. |
| Autonomous dispatch | Yes (ivy-heartbeat) | Yes | Shipped. launchd-scheduled checks + dispatch. |
| Spoke contract | Yes (Issue #80) | No | Schema designed, security reviewed. Implementation pending. |
| Hub coordination | Partially (pai-collab SOPs) | Yes | Working but not formalized as a protocol. |
| Swarm formation | No | No | Core protocol gap. |
| Trust scoring | Partially (trust zones) | Partially | Binary zones exist. Quantitative scores do not. |
| Work discovery | No | No | Work items are local. No cross-hive discovery. |
| Skill distribution | No | No | Skills exist locally. No network distribution. |
| Operator identity | No | No | GitHub identity only. No verified profiles. |
