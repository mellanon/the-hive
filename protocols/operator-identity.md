# Operator Identity

> **Status:** Draft
> **Resolved:** 2026-02-06

![The Operator — human directing AI agents](../assets/concepts/hivemind-sketch-3-operator.png)

## Purpose

Defines how operators are identified, verified, and represented across the network.

## What is an Operator?

A human who directs AI agents to produce work. The operator is always at the centre — their judgment, creativity, domain expertise, and reputation make the swarm productive and trustworthy.

An operator's identity includes:

- **Who they are** — verified identity (not anonymous)
- **What they can do** — skills, domain expertise, agent capabilities
- **What they've done** — trust score from verified contributions
- **Who they've worked with** — swarm history, hive membership

## Operator Profile Schema

The operator profile is a YAML file (`operator.yaml`) that lives in the operator's spoke (`.collab/operator.yaml`). It has three tiers of information:

### Tier 1: Public Identity (visible to all hives)

```yaml
schemaVersion: "1.0"
handle: <github-handle>
name: <display-name>                    # optional
verified: true
verification:
  method: github                        # github | pgp | dns
  evidence: <github-handle>             # verifiable reference
  verified_at: <ISO 8601>
skills:                                 # operator's capabilities (what they bring to a hive)
  - specflow
  - security-scanning
  - content-filtering
availability: open | busy | offline
```

### Tier 2: Hive-Scoped (visible within joined hives)

```yaml
hives:
  - hive: <org/repo>
    role: contributor | reviewer | maintainer
    trust_zone: untrusted | trusted | maintainer
    joined: <ISO 8601>
    contributions: <int>                # count of merged contributions
    reviews: <int>                      # count of completed reviews
    swarms: <int>                       # count of completed swarms
```

### Tier 3: Private (local blackboard only, never published)

```yaml
# NOT in operator.yaml — lives in local blackboard only
tokens_used:
  session: <int>
  7d: <int>
active_work:
  - item: <work-id>
    hive: <org/repo>
    status: in_progress
```

## Design Decisions

### 1. How is identity verified?

**Decision:** GitHub identity is the primary verification method. PGP and DNS TXT records as alternatives.

**Mechanism:**
- **GitHub (default):** Operator's GitHub handle IS their identity. Verified by the fact that they can push to their spoke repo (proof of account control). No additional ceremony needed.
- **PGP (optional):** Operator signs their `operator.yaml` with a PGP key. The key fingerprint is recorded. For operators who want cryptographic identity beyond GitHub.
- **DNS TXT (optional):** Operator adds a TXT record to a domain they control, referencing their operator handle. For operators with domain-backed identity.

**Rationale:** GitHub is already the coordination platform. Every operator already has an account. Adding verification ceremony on top of GitHub would add friction without adding security for the common case. PGP and DNS are opt-in for operators who need stronger identity guarantees.

### 2. How is the operator profile portable across hives?

**Decision:** The `operator.yaml` lives in the spoke (`.collab/`), published alongside the spoke contract. Any hive can read it.

**Mechanism:**
- Tier 1 (public identity) is always visible — any hive can verify who you are
- Tier 2 (hive-scoped) lists all hive memberships — a new hive can see your history elsewhere
- Trust doesn't transfer automatically — it's evidence for the new hive's maintainer to consider
- The spoke `status.yaml` references the operator profile, so hub aggregation picks it up

**Rationale:** The spoke blackboard is already the projection mechanism. Adding operator identity to it is natural — no new infrastructure needed.

### 3. How do capabilities relate to the operator profile?

**Decision:** Skills are listed at the operator level, not per-agent. An operator may run multiple agents under different names — that's an implementation detail. The network cares about what the operator can do, not how many agents they run.

**Mechanism:**
- `skills` array lists the operator's capabilities (references [Skill Protocol](skill-protocol.md) names)
- `availability` is a single status for the operator (open/busy/offline)
- Agents are invisible to the protocol — they're local infrastructure, not network identity

**Rationale:** An operator might bring one agent or ten. They might rename them, swap platforms, or run different agents for different tasks. The hive doesn't need to track that. It needs to know: what can this operator do, and are they available?

### 4. What's public vs what's shared only within a hive?

**Decision:** Three-tier privacy model (see schema above).

| Tier | Visibility | Contents |
|------|-----------|----------|
| **Public** | Any hive, any operator | Handle, verification, agent capabilities |
| **Hive-scoped** | Within joined hives | Trust zone, contribution counts, swarm history |
| **Private** | Local blackboard only | Token usage, active work details, personal config |

**Rationale:** Operators should control what's visible. Public identity enables discovery. Hive-scoped data enables trust assessment within a community. Private data stays on the operator's machine — the local blackboard is sovereign.

## Prior Art

- pai-collab REGISTRY.md agent entries — name, operator, platform, skills, availability
- pai-collab CONTRIBUTORS.yaml — trust zone assignments per operator
- PAI `settings.json` — local identity configuration (name, voice, preferences)
