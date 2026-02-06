# Hive Protocol

> **Status:** Draft

## Purpose

The hive protocol defines how collaboration spaces form, govern themselves, and coordinate work across operators. A hive is the hub layer — the shared surface where operators discover work, form swarms, and build trust.

## What is a Hive?

A managed collaboration space where work happens. Each hive has:

- **Identity** — name, description, governance model
- **Membership** — who can join, roles, trust tiers
- **Work** — what needs doing, who's doing it, what's done
- **Governance** — SOPs, review standards, contribution protocols
- **Trust** — how reputation is earned and tracked within this hive

## Hive Types

| Type | Membership | Lifespan | Example |
|------|-----------|----------|---------|
| **Open** | Anyone | Permanent | Community tools hive |
| **Closed** | Invite-only | Permanent | Private team workspace |
| **Ephemeral** | Scoped | Task duration | A bounty or sprint |
| **Federated** | Cross-hive | Varies | Multi-community initiative |

## hive.yaml (Hive Manifest)

```yaml
schemaVersion: "1.0"
name: <hive-name>
description: <what this hive is for>
type: open | closed | ephemeral
governance:
  model: maintainer | consensus | delegated
  maintainers:
    - <github-handle>
  sops:
    - <path to SOP>
membership:
  join: open | application | invite
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

## Open Questions

1. How are hives discovered? (Registry? Search? Federation?)
2. Can a hive migrate between platforms? (GitHub today, Discord tomorrow)
3. How does cross-hive trust work? (Portable reputation vs hive-scoped)
4. What's the minimum viable hive? (Just a repo with hive.yaml?)

## Reference Implementation

[pai-collab](https://github.com/mellanon/pai-collab) is the first hive ("Hive Zero"). It implements the hub layer using GitHub as the platform — fork-and-PR for coordination, REGISTRY.md for project tracking, TRUST-MODEL.md for governance.
