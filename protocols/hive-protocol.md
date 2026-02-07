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
  signing:
    required: true                        # require signed commits (recommended)
    allowed_signers: .hive/allowed-signers # path to trust anchor file
    key_types:                            # accepted key types
      - ssh-ed25519                       # recommended
      - ssh-rsa                           # accepted (legacy)
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
# Creates: hive.yaml, .hive/allowed-signers, REGISTRY.md (empty), .github/ scaffolding
```

This follows the local-first principle: the simplest hive is a repo. Complexity is added only when needed.

## Cryptographic Trust Anchor: allowed-signers

Every hive maintains a `.hive/allowed-signers` file that maps operators to their Ed25519 public keys. This is the **trust anchor** for the hive — the source of truth for "who is who."

```
# .hive/allowed-signers
# Format: <email> <key-type> <public-key>
# Maintained by hive maintainers. Changes require PR review.
andreas@example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIO6xYoY3NSCNkCSiS4GU+EhZ...
jcfischer@example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIx...
```

**Properties:**
- **Version-controlled** — changes to allowed-signers are tracked in git history (who added whom, when)
- **Maintainer-reviewed** — adding a new operator requires a reviewed PR (the PR itself proves key ownership via signed commit)
- **Git-native verification** — anyone can verify any commit against this file: `git config gpg.ssh.allowedSignersFile .hive/allowed-signers && git log --show-signature`
- **Revocation by removal** — removing a line revokes that operator's signing authority. Future commits from that key won't verify.

**Maintenance rules:**
1. Only maintainers can merge changes to `.hive/allowed-signers`
2. Adding a new operator: operator submits signed PR with their public key. The signed commit IS the proof of key ownership.
3. Removing an operator: maintainer removes the entry. Revocation is immediate for future verification.
4. Key rotation: operator submits signed PR (signed with OLD key) adding new key. Maintainer reviews and merges. Then a second PR (signed with NEW key) removes the old key.
5. CI should enforce: all commits in PRs are signed by a key listed in `.hive/allowed-signers`

## Enforcement Model: CI as State Machine

Hive security and onboarding are enforced through CI pipelines on the hub repository — not through contributor-side tooling. This means:

- **Contributors bring:** git (2.34+) and an SSH key — nothing else
- **The hive enforces:** signing verification, secret scanning, schema validation, governance checks

### How It Works

The hub CI pipeline runs ordered gates on every pull request:

```
PR submitted
    │
    ├─ Gate 1: IDENTITY
    │   ├─ All commits signed? (warn if not)
    │   └─ Signing key in allowed-signers? (notice if new contributor)
    │
    ├─ Gate 2: SECURITY
    │   ├─ Secret scanning passes? (error — blocks merge)
    │   └─ Content filter passes? (error — blocks merge)
    │
    ├─ Gate 3: SCHEMA
    │   ├─ PROJECT.yaml valid? (error — blocks merge)
    │   ├─ JOURNAL.md present? (error — blocks merge)
    │   └─ REGISTRY/STATUS aligned? (error — blocks merge)
    │
    └─ Gate 4: GOVERNANCE
        ├─ Issue referenced? (warning)
        ├─ Journal updated? (warning)
        └─ STATUS.md updated? (warning if PROJECT.yaml changed)
```

### Why CI, Not Custom Tooling

The enforcement lives on the **receiving end** (the hive), not the sending end (the contributor). This is critical for low-friction onboarding — a new operator can fork, write, sign commits, and PR without installing anything beyond git.

The pattern is: **SOPs describe the process. CI enforces the checkable parts. Humans review the rest.** Not every step can be machine-verified (e.g., "read the trust model"), but the enforceable steps (signing, scanning, schemas) are gated automatically.

### State Tracking

Operator state is tracked through git artifacts, not a custom database:

| State | Tracked In | How It Changes |
|-------|-----------|----------------|
| Registered | `.hive/allowed-signers` | Maintainer merges PR adding key |
| Trust zone | `CONTRIBUTORS.yaml` | Maintainer updates entry |
| Contributions | git log + JOURNAL.md | Each merged PR |
| Active projects | PROJECT.yaml contributors | Maintainer updates |

This is inspired by [Arbor](https://github.com/trust-arbor/arbor)'s principle that SOPs should be state machines rather than best-effort documentation. The adaptation for git-based protocols: the CI pipeline is the state machine runtime, and git artifacts are the state storage.

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
