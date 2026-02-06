# Hive Protocol

> **Status:** Draft
> **Resolved:** 2026-02-06

![Hives — managed collaboration spaces](../assets/concepts/hivemind-sketch-1-hives.png)

## Purpose

The hive protocol defines how collaboration spaces form, govern themselves, and coordinate work across operators. A hive is the hub blackboard — the shared surface where operators discover work, form swarms, and build trust.

## What is a Hive?

A managed collaboration space where work happens. Each hive has:

- **Identity** — name, description, governance model (`hive.yaml`)
- **Membership** — who can join, roles, trust tiers
- **Work** — what needs doing, who's doing it, what's done
- **Governance** — SOPs, review standards, contribution protocols
- **Trust** — how reputation is earned and tracked within this hive

## Hive Types

| Type | Membership | Lifespan | Identity | Example |
|------|-----------|----------|----------|---------|
| **Open** | Anyone | Permanent | GitHub, Google, social | Community tools hive |
| **Closed** | Invite-only | Permanent | GitHub + application | Private team workspace |
| **Enterprise** | Corporate | Permanent | SSO (Okta, Azure AD) | Company-internal hive |
| **Ephemeral** | Scoped | Task duration | Inherited from parent | A bounty or sprint |
| **Federated** | Cross-hive | Varies | Any (per-hive policy) | Multi-community initiative |

## hive.yaml (Hive Manifest)

```yaml
schemaVersion: "1.0"
name: <hive-name>
description: <what this hive is for>
type: open | closed | ephemeral
platform: github          # current platform (github, discord, etc.)
governance:
  model: maintainer | consensus | delegated
  maintainers:
    - <github-handle>
  sops:
    - <path to SOP>
membership:
  join: open | application | invite
  identity:                             # see Operator Identity protocol
    required:                           # operator must have at least one of these
      - github
    accepted:                           # all providers this hive recognizes
      - github
      - google
      - facebook
      - apple
    enterprise:                         # for closed/enterprise hives (optional)
      provider: <saml | oidc>
      tenant: <tenant-id>
      domain: <corporate-domain>
      sso_url: <SSO endpoint>
  roles:
    - contributor
    - reviewer
    - maintainer
trust:
  model: <reference to trust protocol>
  zones:
    - untrusted
    - trusted
    - maintainer
federation:
  registry: <org/repo>    # where this hive is listed (optional)
  peers: []               # other hives this one recognizes (optional)
```

## Hive Lifecycle

```
CREATE → CONFIGURE → OPERATE → (ARCHIVE | DISSOLVE)

CREATE:     Maintainer creates hive manifest + infrastructure
CONFIGURE:  Set governance, membership rules, initial SOPs
OPERATE:    Work posted, operators join, swarms form, trust builds
ARCHIVE:    Hive preserved as read-only record
DISSOLVE:   Ephemeral hive removed after task completion
```

## Minimum Viable Hive

A hive is a git repository with a `hive.yaml` at its root. That's it.

Everything else — REGISTRY.md, TRUST-MODEL.md, SOPs, project directories — is optional and additive. The `hive.yaml` manifest declares the hive's identity, governance model, and membership rules. A git repo with `hive.yaml` is a valid hive.

**To create a hive:**
```bash
blackboard init --level hub
# Creates: hive.yaml, REGISTRY.md (empty), .github/ scaffolding
```

This follows the local-first principle: the simplest hive is a repo. Complexity is added only when needed.

## Design Decisions

### 1. How are hives discovered?

**Decision:** Hub-level registry file (`REGISTRY.yaml`) in a known location, plus git-native search.

**Mechanism:** Each hive can optionally list itself in a parent registry (another git repo). Discovery works in three tiers:
1. **Direct link** — someone shares the repo URL (how pai-collab works today)
2. **Registry lookup** — a known registry repo contains a `hives/REGISTRY.yaml` listing all registered hives with name, URL, type, and topic tags
3. **Federation** — hives list each other as `peers` in their `hive.yaml`, enabling cross-hive discovery

**Rationale:** No central index is required. Discovery is decentralized and git-native. A registry is just another repo.

### 2. Can a hive migrate between platforms?

**Decision:** Yes. The `hive.yaml` `platform` field declares the current platform. The protocol is platform-agnostic — the manifest, spoke contracts, and trust model are all YAML/git artifacts that can move.

**Mechanism:** Migration is a three-step process:
1. Export: `blackboard export --level hub` produces a portable archive (hive.yaml + REGISTRY + trust data + work items)
2. Create new platform instance (e.g., Discord server, new git host)
3. Import: `blackboard import --level hub` restores state on the new platform

**Rationale:** Platform lock-in is a risk (pai-collab's own roadmap includes Discord migration). The protocol must survive platform changes. Git history is the durable layer — platforms are interfaces.

### 3. How does cross-hive trust work?

**Decision:** Trust is hive-scoped by default, portable by reference.

**Mechanism:**
- Each hive maintains its own trust zones and scores (sovereign)
- An operator's profile includes a list of hive memberships with their trust level in each
- When joining a new hive, existing trust in other hives is visible but NOT automatically inherited
- The new hive's maintainer can use cross-hive trust as evidence for faster promotion, but the decision is human

**Rationale:** Automatic trust transfer is dangerous — a trusted operator in a coding hive isn't necessarily trusted for security work. Cross-hive trust is *evidence*, not *authority*. Maintainers decide. See [Trust Protocol](trust-protocol.md) for scoring details.

### 4. What's the minimum viable hive?

**Decision:** A git repository with `hive.yaml` at the root.

See "Minimum Viable Hive" section above.

## Reference Implementation

[pai-collab](https://github.com/mellanon/pai-collab) is the first hive ("Hive Zero"). It implements the hub blackboard using GitHub as the platform — fork-and-PR for coordination, REGISTRY.md for project tracking, TRUST-MODEL.md for governance.
