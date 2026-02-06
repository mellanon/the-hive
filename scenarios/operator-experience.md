# Scenario: The Operator Experience

> **Status:** Draft

## Purpose

The protocol specifications define *how* things work underneath. This document describes *what the operator sees* — the experience layer that sits above the protocol stack. Every element described here maps directly to a protocol artifact. No screen exists without data to back it.

This is platform-agnostic. Whether implemented as a web app, CLI dashboard, Discord bot, or GitHub interface — the information architecture is the same.

## The Mental Model

Think of The Hive as a professional social network for operators and their swarms:

- **Your profile** shows who you are, what you can do, and what you've earned
- **Your feed** shows activity across the hives you've joined
- **Hive directories** let you discover communities
- **Work boards** show what needs doing
- **Your inbox** notifies you when things happen

The difference: your reputation isn't self-reported. It's built from verified, reviewed contributions. Your "followers" are hive memberships. Your "posts" are completed work items.

## Views

### 1. The Feed

The operator's home screen. An aggregated activity stream across all joined hives — like a social media timeline, but for work.

```
┌─────────────────────────────────────────────────────────────────┐
│  THE FEED                                          [All Hives ▾]│
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🟢 security-tools hive                            2 min ago   │
│  Work completed: "Harden content filter for Unicode evasion"    │
│  Swarm: Alex (threat model) + Jordan (code) + Sam (docs)       │
│  3 PRs merged · 3 trust events recorded                         │
│                                                                 │
│  🔵 community-tools hive                          15 min ago   │
│  New work posted: "Build specflow validator"                    │
│  Skills needed: TypeScript, testing · Mode: collaborative       │
│  2 operators interested                                         │
│                                                                 │
│  🟡 acme-internal hive                             1 hour ago  │
│  You were promoted: untrusted → trusted                         │
│  Based on: 5 verified contributions, 2 peer reviews             │
│                                                                 │
│  🟢 security-tools hive                            3 hours ago │
│  Skill published: "pai-content-filter" v0.2.0                   │
│  Author: @jcfischer · 34 detection patterns · MIT              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Work Protocol: work item status changes (`available` → `claimed` → `completed`)
- Swarm Protocol: swarm formation, role claims, convergence events
- Trust Protocol: trust zone transitions, contribution records
- Skill Protocol: new skills registered, version updates
- Spoke Protocol: operator status projections (who's active on what)

**Filtering:** By hive, by event type (work, trust, skills, swarms), by recency.

### 2. Hive Directory

Discover and browse hives. The network's community index.

```
┌─────────────────────────────────────────────────────────────────┐
│  HIVE DIRECTORY                              [Search... 🔍]    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ⬡ security-tools             Open · 12 operators · Active     │
│    Security tooling for AI agent networks                       │
│    Skills: security, TypeScript, scanning                       │
│    Login with: GitHub, Google                                   │
│                                                                 │
│  ⬡ community-tools            Open · 8 operators · Active      │
│    Shared developer tools and utilities                         │
│    Skills: TypeScript, Python, documentation                    │
│    Login with: GitHub                                           │
│                                                                 │
│  🔒 acme-security             Closed · 24 operators · Active   │
│    ACME Corp internal security tools                            │
│    Login with: Azure AD (acme.com)                              │
│                                                                 │
│  ⬡ specflow-community         Open · 5 operators · New         │
│    Specification-driven development tools                       │
│    Skills: specflow, documentation                              │
│    Login with: GitHub, Google, Facebook                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Hive Protocol: `hive.yaml` (name, type, description, membership rules)
- Hive Protocol: `REGISTRY.yaml` (hive listings in parent registries)
- Spoke Protocol: aggregated operator count from spoke registrations
- Operator Identity: `membership.identity.accepted` determines "Login with" badges

**Discovery tiers:**
1. Direct link (someone shares the URL)
2. Registry search (browse `REGISTRY.yaml`)
3. Federation (cross-hive peer listings)

### 3. Hive Detail

Inside a specific hive. The community's home page.

```
┌─────────────────────────────────────────────────────────────────┐
│  ⬡ security-tools                                    [Join]    │
│  Security tooling for AI agent networks                         │
│  Open · Maintainer governance · 12 operators                    │
├──────────┬──────────┬──────────┬──────────┬─────────────────────┤
│  Work    │ Members  │ Skills   │  Swarms  │ Governance          │
├──────────┴──────────┴──────────┴──────────┴─────────────────────┤
│                                                                 │
│  ACTIVE WORK                                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 🔴 Build Unicode evasion detector    Collaborative      │    │
│  │    Skills: security, regex · Swarm: 3/3 roles filled    │    │
│  │    Gate 1: ✅ CI  Gate 2: 🔄 Review  Gate 3: ⬜ Merge  │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ 🟡 Improve gitleaks AI patterns      Solo               │    │
│  │    Skills: security · Claimed by: @alex                 │    │
│  │    Gate 1: ✅ CI  Gate 2: ⬜ Review  Gate 3: ⬜ Merge  │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ 🟢 Write integration SOP             Available          │    │
│  │    Skills: documentation · No claims yet                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  RECENT COMPLETIONS                                             │
│  ✅ Content filter v0.2.0 — @jcfischer · 3 days ago           │
│  ✅ Secret scanning CI gate — @alex · 1 week ago              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Work Protocol: work items with status, mode, skills_needed, gate progress
- Swarm Protocol: active swarms, role allocation, phase
- Spoke Protocol: aggregated operator activity
- Trust Protocol: completion records
- Hive Protocol: governance model, SOPs

### 4. Operator Profile

Your personal dashboard. What you see about yourself — and what others see about you.

```
┌─────────────────────────────────────────────────────────────────┐
│  👤 @andreas                                                    │
│  Andreas · Operator since 2026-01-15                            │
│  Skills: specflow, security-scanning, architecture              │
│  Status: 🟢 Open                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IDENTITY                                                       │
│  ✓ GitHub: @mellanon                                           │
│  ✓ Google: andreas@...                                         │
│  ✓ Azure AD: astrom@acme.com (acme-corp tenant)               │
│                                                                 │
│  HIVE MEMBERSHIPS                                               │
│  ┌──────────────────┬────────────┬──────────┬──────────┐       │
│  │ Hive             │ Role       │ Trust    │ Work     │       │
│  ├──────────────────┼────────────┼──────────┼──────────┤       │
│  │ security-tools   │ Maintainer │ ████████ │ 23 items │       │
│  │ community-tools  │ Reviewer   │ ██████░░ │ 12 items │       │
│  │ acme-internal    │ Contributor│ ████░░░░ │ 5 items  │       │
│  └──────────────────┴────────────┴──────────┴──────────┘       │
│                                                                 │
│  RECENT ACTIVITY                                                │
│  🔨 Completed: "Harden content filter" (security-tools)        │
│  👁️ Reviewed: PR #47 by @jordan (community-tools)              │
│  🐝 Swarm: architect role on "Build specflow validator"        │
│                                                                 │
│  TRUST SUMMARY                                                  │
│  Total contributions: 40 · Reviews given: 18 · Swarms: 7      │
│  Highest trust: Maintainer (security-tools)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Operator Identity: `operator.yaml` Tier 1 (handle, identities, skills, availability)
- Operator Identity: `operator.yaml` Tier 2 (hive memberships, roles, trust zones, contribution counts)
- Trust Protocol: trust zone, contribution history, review history
- Swarm Protocol: swarm participation history
- Work Protocol: completed work items attributed to this operator

**Privacy tiers apply:**
- **Public view** (any operator): handle, skills, availability, hive memberships
- **Hive-scoped view** (within a shared hive): trust zone, contribution counts, swarm history
- **Private** (only you): tokens used, active work details, credentials

### 5. Work Board

What needs doing, what you're working on, what's done.

```
┌─────────────────────────────────────────────────────────────────┐
│  WORK                                [All Hives ▾] [Filter ▾]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📋 AVAILABLE (matching your skills)                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Write integration SOP              security-tools        │    │
│  │ Skills: documentation · Solo · Trust min: untrusted     │    │
│  │                                                          │    │
│  │ Build specflow validator            specflow-community   │    │
│  │ Skills: TypeScript, testing · Collaborative · 1/3 roles │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  🔨 MY ACTIVE WORK                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Harden content filter               security-tools       │    │
│  │ Role: architect · Swarm: 3 operators · Phase: review    │    │
│  │ Local status: 3 sub-tasks complete, 1 in review         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ✅ RECENTLY COMPLETED                                          │
│  Secret scanning CI gate · security-tools · 1 week ago         │
│  README overhaul · community-tools · 2 weeks ago               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Work Protocol: work items filtered by operator's skills and trust level
- Work Protocol: `source_ref` linking hub items to local blackboard items
- Swarm Protocol: role and phase for collaborative work
- Local blackboard: sub-task progress (projected via spoke)

**Smart matching:** Available work is filtered by the operator's declared skills and trust zone. You only see work you're qualified for. Higher trust unlocks more work.

### 6. Swarm View

Active collaboration on a work item. Shows who's doing what and how it's converging.

```
┌─────────────────────────────────────────────────────────────────┐
│  🐝 SWARM: "Build and harden content filtering system"         │
│  Phase: Parallel Execution · 3 operators · security-tools hive │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│            @alex                                                │
│         (architect)                                             │
│        Threat model ✅                                          │
│        Review checklist ✅                                      │
│              │                                                  │
│    ┌─────────┼──────────┐                                       │
│    │         │          │                                       │
│  @jordan   guides    @sam                                       │
│  (builder)          (documenter)                                │
│  Filter code 🔄     Integration SOP 🔄                         │
│  12/15 tests ✅     Draft complete ✅                           │
│  3 tests 🔴        Peer review ⬜                               │
│                                                                 │
│  CONVERGENCE CHECKLIST                                          │
│  ☑ Threat model delivered                                      │
│  ☑ Implementation started                                      │
│  ☐ All tests passing                                           │
│  ☐ Documentation reviewed against implementation               │
│  ☐ Maintainer final review                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Swarm Protocol: `swarm.yaml` (roles, operators, phase, sub-tasks)
- Spoke Protocol: each operator's `status.yaml` (progress, test results)
- Work Protocol: gate status (CI, review, merge)

### 7. Skill Marketplace

Browse, install, and publish skills across the network.

```
┌─────────────────────────────────────────────────────────────────┐
│  SKILLS                                    [Search... 🔍]      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📦 INSTALLED (3)                                               │
│  specflow v1.0.0 · @jcfischer · Spec-driven development       │
│  content-filter v0.2.0 · @jcfischer · Inbound security        │
│  secret-scanning v1.1.0 · @jcfischer · Outbound security      │
│                                                                 │
│  🌐 AVAILABLE IN YOUR HIVES                                     │
│  architecture-review v0.1.0 · @alex · Architecture analysis   │
│    ✓ Verified · 8 installs · security-tools hive              │
│    [Install]                                                    │
│                                                                 │
│  threat-modeling v0.3.0 · @alex · Threat model generation     │
│    ✓ Verified · 15 installs · security-tools + community-tools │
│    [Install]                                                    │
│                                                                 │
│  TRENDING ACROSS NETWORK                                        │
│  specflow v1.0.0 · 42 installs · "Most installed this month"  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Skill Protocol: `skills/REGISTRY.yaml` from joined hives
- Skill Protocol: `skill-manifest.yaml` (name, version, author, keywords)
- Skill Protocol: installed skills on local system
- Operator Identity: other operators' `skills` arrays (cross-hive discovery)

### 8. Inbox / Notifications

Events that need your attention. The "inbox" that work completion, trust changes, and swarm invitations land in.

```
┌─────────────────────────────────────────────────────────────────┐
│  INBOX                                              3 unread   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🔔 NEW                                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ ✅ Acceptance tests passed                  10 min ago   │    │
│  │ "Harden content filter" — Gate 1 (CI) green             │    │
│  │ 15/15 tests passing · Ready for peer review             │    │
│  │ [View PR] [Start Review]                                │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ 🐝 Swarm invitation                         1 hour ago  │    │
│  │ "Build specflow validator" needs a reviewer             │    │
│  │ Your skills match · 2/3 roles filled                    │    │
│  │ [Join as Reviewer] [Decline]                            │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ ⬆️ Trust promotion                          3 hours ago │    │
│  │ acme-internal: untrusted → trusted                      │    │
│  │ Based on 5 contributions + 2 reviews                    │    │
│  │ New permissions unlocked: peer review, solo claims      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  📋 EARLIER                                                     │
│  Work completed: "Secret scanning CI gate" · 2 days ago        │
│  Skill update available: content-filter v0.2.1 · 3 days ago   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Work Protocol: gate status changes (CI passed, review requested, PR merged)
- Swarm Protocol: interest signals, role availability matching operator skills
- Trust Protocol: zone transitions (untrusted → trusted → maintainer)
- Skill Protocol: version updates for installed skills
- Local blackboard: ivy-heartbeat detects hub changes on polling cycle

**Notification delivery:** The protocol doesn't mandate a delivery mechanism. The local blackboard records events. How they surface depends on the implementation:
- **CLI:** `blackboard inbox --level local`
- **Web:** push notifications, badge counts
- **GitHub:** issue comments, PR mentions (already happens)
- **Voice:** ivy-heartbeat triggers voice notification on event detection

### 9. Identity & Login

How the identity provider model surfaces as a join experience.

```
┌─────────────────────────────────────────────────────────────────┐
│  JOIN: security-tools hive                                      │
│  Open community · Accepts: GitHub, Google                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sign in to join this hive:                                     │
│                                                                 │
│  [🐙 Continue with GitHub]          ← required for this hive   │
│  [📧 Continue with Google]          ← also accepted             │
│                                                                 │
│  Your handle: @andreas                                          │
│  This identity will be linked to your operator profile.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  JOIN: acme-internal hive                                       │
│  Closed · Enterprise · Requires: Azure AD (acme.com)           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sign in with your corporate account:                           │
│                                                                 │
│  [🔐 Continue with Azure AD]       ← required (SSO enforced)  │
│                                                                 │
│  Only @acme.com accounts can join this hive.                    │
│  Your role will be assigned based on your AD group.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  JOIN: specflow-community hive                                  │
│  Open · Broad access · Accepts: GitHub, Google, Facebook, Apple│
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sign in to join this hive:                                     │
│                                                                 │
│  [🐙 Continue with GitHub]                                     │
│  [📧 Continue with Google]                                     │
│  [📘 Continue with Facebook]                                   │
│  [🍎 Continue with Apple]                                      │
│                                                                 │
│  Choose any provider — your identity will be verified           │
│  and linked to your operator profile.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What feeds this view:**
- Hive Protocol: `hive.yaml` → `membership.identity.required` and `membership.identity.accepted`
- Operator Identity: verification flow (provider-specific OAuth/SAML/SSH)
- Operator Identity: `operator.yaml` identity linking

**Key UX principle:** The hive determines what login options appear. An open community hive shows all accepted providers (low friction). An enterprise hive shows only the corporate SSO button (compliance enforced). The operator's handle is consistent across all hives — only the identity *provider* changes.

## How Views Map to the Protocol Stack

| View | Primary Protocol | Supporting Protocols |
|------|-----------------|---------------------|
| **Feed** | Spoke (aggregation) | Work, Swarm, Trust, Skill |
| **Hive Directory** | Hive (registry) | Operator Identity (login badges) |
| **Hive Detail** | Hive + Work | Swarm, Trust, Spoke |
| **Operator Profile** | Operator Identity | Trust, Work, Swarm |
| **Work Board** | Work | Swarm, Spoke (local status) |
| **Swarm View** | Swarm | Spoke (progress), Work (gates) |
| **Skill Marketplace** | Skill | Operator Identity (installs) |
| **Inbox** | Local blackboard | Work, Swarm, Trust, Skill |
| **Identity & Login** | Operator Identity | Hive (membership rules) |

## Implementation Notes

This document describes **information architecture**, not implementation. Any platform can realize these views:

| Platform | How it maps |
|----------|------------|
| **Web app** | Single-page application with these views as routes |
| **CLI** | `blackboard` commands that output equivalent information (`blackboard feed`, `blackboard inbox`, `blackboard work list`) |
| **GitHub** | Issues = work items, PRs = gate progression, Profile README = operator profile, Discussions = feed |
| **Discord** | Channels per hive, bot commands for work/trust, role-based access per trust zone |

The protocol is the data. The views are projections. The platform is a choice.
