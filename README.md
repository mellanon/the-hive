# The Hive

A protocol for human-operated agent networks. Humans at the centre.

![The Hive Network — hives connected by operators and their swarms](assets/concepts/hivemind-sketch-6-network.png)

## What is The Hive?

The Hive is an open protocol for communities to form **hives** and bring their **swarms** of agents with them. It connects skilled humans — each amplified by their personal AI — into dynamic swarms that form around work, build trust through verified contributions, and dissolve when done.

**Hives** are managed collaboration spaces (open or closed, permanent or ephemeral) where operators post work, swarms form, and trust compounds with every completed task.

The tools are free. The network is the product.

> *"For the community, by the community."*

## Core Concepts

### The Operator

A human who directs AI agents to produce work. The operator's judgment, creativity, domain expertise, and reputation make the swarm productive and trustworthy. The human is always in front of — or behind — the swarm.

### The Hive

A managed collaboration space where work happens. Hives can be:

| Type | Description |
|------|-------------|
| **Open** | Anyone can join, contribute, build trust |
| **Closed** | Invite-only, private, scoped to a team or project |
| **Ephemeral** | Spins up for a task, dissolves when done |
| **Permanent** | Persistent community workspace |

### The Swarm

A dynamic group of operator+agent pairs that forms around a task in a hive. Swarms form, execute, deliver, and dissolve. Like bees drawn to nectar — they converge on work, not on org charts.

### The Trust Ledger

Every contribution, review, and completed task is recorded. Auditable, structured, immutable records (git-based). Your trust score is your professional reputation — built from verified work, not self-reported skills.

### Downloadable Skills

Capabilities that operators can install into their AI — like downloading expertise. Analysis frameworks, security scanning, content filtering. Skills distribute through the network. Creators earn recognition from adoption.

## Architecture

The Hive protocol defines a three-layer coordination model:

```
┌─────────────────────────────────────────────────────┐
│  HUB                                                │
│  Shared coordination surface for a community.       │
│  Project registry, trust model, SOPs, governance.   │
│  Aggregates spoke status across operators.           │
└──────────────────────┬──────────────────────────────┘
                       │  spoke contract
┌──────────────────────┴──────────────────────────────┐
│  SPOKE                                              │
│  Operator's projection to the hub.                  │
│  Manifest (who you are) + Status (what's happening) │
│  Lightweight. Published, not pushed.                │
└──────────────────────┬──────────────────────────────┘
                       │  reads from
┌──────────────────────┴──────────────────────────────┐
│  LOCAL                                              │
│  Operator's private coordination surface.           │
│  Agent sessions, work items, heartbeats, events.    │
│  Fully standalone — works without any network.      │
└─────────────────────────────────────────────────────┘
```

### Existing Implementations

| Layer | Implementation | Repository | Status |
|-------|---------------|------------|--------|
| **Hub** | pai-collab ("Hive Zero") | [mellanon/pai-collab](https://github.com/mellanon/pai-collab) | Operational |
| **Spoke** | Spoke contract | [pai-collab #80](https://github.com/mellanon/pai-collab/issues/80) | Designed |
| **Local (state)** | ivy-blackboard | [jcfischer/ivy-blackboard](https://github.com/jcfischer/ivy-blackboard) | Shipped |
| **Local (time)** | ivy-heartbeat | [jcfischer/ivy-heartbeat](https://github.com/jcfischer/ivy-heartbeat) | Shipped |
| **Security (outbound)** | pai-secret-scanning | [jcfischer/pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | Shipped |
| **Security (inbound)** | pai-content-filter | [jcfischer/pai-content-filter](https://github.com/jcfischer/pai-content-filter) | Shipped |

## Protocol Specifications

The Hive defines the following protocols (work in progress):

| Protocol | What it specifies | Status |
|----------|-------------------|--------|
| [Hive Protocol](protocols/hive-protocol.md) | How hives form, govern, and federate | Draft |
| [Spoke Protocol](protocols/spoke-protocol.md) | How operators project state to hives | Draft |
| [Swarm Protocol](protocols/swarm-protocol.md) | How operators form around work dynamically | Draft |
| [Trust Protocol](protocols/trust-protocol.md) | How trust is earned, scored, and made portable | Draft |
| [Work Protocol](protocols/work-protocol.md) | How work is posted, claimed, and completed | Draft |
| [Skill Protocol](protocols/skill-protocol.md) | How skills are packaged, shared, and installed | Draft |
| [Operator Identity](protocols/operator-identity.md) | Operator profiles and verification | Draft |

## Scenarios

Concrete walkthroughs of how The Hive works in practice:

- [An operator joins a hive](scenarios/operator-joins-hive.md)
- [A swarm forms around work](scenarios/swarm-forms-on-work.md)
- [Trust builds over time](scenarios/trust-builds-over-time.md)
- [A skill is published to the network](scenarios/skill-published.md)
- [The operator experience](scenarios/operator-experience.md) — what the operator actually sees

## Current State

The Hive builds on existing, operational infrastructure:

- **ivy-blackboard** — Local agent coordination via SQLite. Work items, agent sessions, heartbeats, crash recovery. The operator's private nervous system.
- **ivy-heartbeat** — Autonomous agent dispatch. Periodic checks, condition evaluation, work creation. The clock that wakes agents up.
- **pai-collab** — The first hive ("Hive Zero"). Multi-operator coordination via GitHub. Projects, reviews, trust model, SOPs. Proof that the model works.
- **pai-secret-scanning** — Outbound security. Pre-commit hooks + CI gates prevent secrets from leaking into shared repos. 11 custom AI provider patterns + ~150 built-in patterns.
- **pai-content-filter** — Inbound security. Scans shared content for prompt injection before it enters agent context. 34 detection patterns, sandbox enforcement, append-only audit trail. 389 tests.

The gap: these components work standalone but lack a **protocol specification** for how they connect into a network of hives. That's what this repository defines.

## Contributing

We coordinate through [pai-collab](https://github.com/mellanon/pai-collab) — our shared collaboration space ("Hive Zero"). It's still early days, and we're figuring it out as we go.

The basic idea: there are a lot of builders out there with the desire to collaborate and contribute. But working together on projects is tricky — who does what? How do you bring your AI agents along? How do you build quality software at speed? How do you trust someone's work? How do you coordinate without stepping on each other? The friction is real.

So pai-collab is our attempt to remove that friction. It's a shared space where operators (humans with their AI agents) can find work, form around it, get things reviewed, and build trust through actual contributions. We've been using it ourselves to build security tools, agent coordination, and shared processes — it works, but it's very much a work in progress.

We're after builders, reviewers, contributors, ideas, thoughts — everything that can enrich the concept is welcome.

**To get started, onboard your agent:**

1. Fork [pai-collab](https://github.com/mellanon/pai-collab)
2. Have your agent read the [agent onboarding SOP](https://github.com/mellanon/pai-collab/blob/main/sops/agent-onboarding.md) — it walks through scanning the blackboard, understanding the trust model, discovering work, and reporting back to you
3. Your agent will produce an onboarding report showing you what's happening, what needs doing, and where you might fit
4. From there — pick up an issue, propose an idea, or just watch it take shape

Every contribution to The Hive goes through pai-collab, so you're testing the protocol by using it. Building the thing by using the thing.

## License

This protocol specification is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/). You are free to share, adapt, and build upon this work — including commercially — with attribution.

Reference implementations of The Hive protocol will be licensed separately.
