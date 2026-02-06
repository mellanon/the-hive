# Trust Protocol

> **Status:** Draft

![Trust Ledger — history, provenance, approval](../assets/concepts/hivemind-sketch-4-trust.png)

## Purpose

The trust protocol defines how trust is established, enforced, and built within and across hives. Trust has two dimensions: **security infrastructure** (automated, enforced) and **reputation** (earned, scored).

## Principles

1. **Earned, not claimed.** Trust comes from verified work, not self-reported skills.
2. **Enforced, not assumed.** Automated security gates apply to all contributors regardless of trust zone.
3. **Auditable.** Every trust-relevant event is recorded in immutable, queryable logs.
4. **Portable.** Trust earned in one hive should be recognizable in another.
5. **Decaying.** Inactive trust decays over time — reputation must be maintained.
6. **Git-based.** The trust ledger is auditable through git history, not blockchain.

## Dimension 1: Security Infrastructure (Built)

Trust starts with automated security. These are prerequisites for collaboration, not optional add-ons.

### The Six Defense Layers

| Layer | Direction | What it does | Implementation | Status |
|-------|-----------|-------------|----------------|--------|
| 1 | Outbound | Pre-commit secret scanning | [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | Shipped |
| 2 | Outbound | CI gate scanning on push/PR | [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | Shipped |
| 3 | Boundary | Fork-and-PR review isolation | Git (structural) | Built-in |
| 4 | Inbound | Content scanning before LLM context | [pai-content-filter](https://github.com/jcfischer/pai-content-filter) | Shipped |
| 5 | Inbound | Tool restrictions for untrusted content | [pai-content-filter](https://github.com/jcfischer/pai-content-filter) | Shipped |
| 6 | All | Append-only audit trail | pai-content-filter + ivy-blackboard | Shipped |

### Outbound Defense: Secret Scanning

Prevents secrets from leaking into shared repositories.

**Threat:** Operators work with private infrastructure (API keys, voice credentials, personal paths). Every contribution to a hive is a potential exposure event.

**Mechanism:**
- **Layer 1 (pre-commit):** gitleaks with 11 custom AI provider patterns + ~150 built-in patterns. Runs locally, blocks commits containing secrets.
- **Layer 2 (CI gate):** Same scanning on GitHub Actions. Runs on every push to main and every PR. Catches bypassed hooks.

**Coverage:** Anthropic, OpenAI, ElevenLabs, Telegram, Replicate, HuggingFace, Groq keys. AWS/GCP/Azure credentials. SSH keys, JWTs, .env values.

**Rule:** All operators must install secret scanning before their first contribution. Maintainers are not exempt.

### Inbound Defense: Content Filtering

Prevents prompt injection from compromising agents reading shared content.

**Threat:** Malicious instructions hidden in markdown (HTML comments, encoded payloads, format markers) that cause reviewing agents to execute unintended actions.

**Mechanism:**
- **Layer 4 (content boundary):** 34 deterministic regex patterns detect instruction overrides, PII exposure, and encoding evasion. Scans content before it enters LLM context. No LLM classification — fully auditable.
- **Layer 5 (tool restrictions):** Flagged content triggers quarantine mode — agent gets read-only access, no shell, no writes, no network. Sandbox enforcer intercepts acquisition commands and redirects to quarantine directory.
- **Layer 6 (audit trail):** Append-only JSONL log of every content loading decision. Human override requires recorded reasoning.

**Coverage:** Direct injection, base64/hex/Unicode encoded payloads, model-specific format markers (Llama/Mistral), authority impersonation, incremental drift.

**Design choice:** Deterministic regex over LLM classification. Auditable, reproducible, no latency from recursive AI calls.

## Dimension 2: Trust Zones (Built)

The existing three-zone model from pai-collab:

| Zone | Description | Permissions |
|------|-------------|------------|
| **Untrusted** | Default for all new operators | Full inbound scanning, tool restrictions, detailed logging |
| **Trusted** | Demonstrated track record, promoted explicitly | Passes security layers but with reduced friction |
| **Maintainer** | Governance authority | Can promote/demote, merge PRs, modify SOPs |

**Key properties:**
- **All zones pass through security layers.** Maintainers are scanned too.
- **Promotion is explicit.** A human maintainer promotes an operator based on verified contribution history.
- **Two-level scoping:** Trust can be scoped to a hive (overall) or to a specific project within a hive.

## Dimension 3: Vouching (Designed)

A trusted operator can vouch for another operator — staking their own reputation on the referral. Like the Direct Connect hub model: you get in because someone trusted put their name on you.

### How Vouching Works

```
Operator A (trusted in Hive X)
    │
    │  vouches for
    ▼
Operator B (wants to join Hive X)
    │
    │  result
    ▼
Operator B joins at "untrusted" zone, but with:
  - Accelerated review priority (vouched PRs reviewed faster)
  - Voucher visible on profile (social proof)
  - Voucher's reputation linked (skin in the game)
```

**Key principle:** Vouching is NOT automatic trust promotion. Operator B still starts as `untrusted` and still passes through all security layers. The vouch provides:

1. **Access** — for closed/invite-only hives, a vouch from a trusted member is the entry ticket
2. **Social proof** — other operators and maintainers can see who vouched for you
3. **Accelerated trust building** — maintainers may promote vouched operators faster based on the voucher's reputation
4. **Accountability chain** — the voucher's reputation is partially linked to the vouched operator's behavior

### Vouch Schema

```yaml
# In the hive's trust ledger (CONTRIBUTORS.yaml or equivalent)
vouches:
  - voucher: <handle>                    # who is vouching
    voucher_trust_zone: trusted          # voucher must be trusted or maintainer
    vouchee: <handle>                    # who is being vouched for
    reason: "Worked together on content-filter in security-tools hive"
    timestamp: <ISO 8601>
    cross_hive_evidence:                 # optional — reference to other hive history
      hive: <org/repo>
      trust_zone: trusted
      contributions: 12
```

### Vouch Rules

| Rule | Description |
|------|-------------|
| **Minimum voucher trust** | Only `trusted` or `maintainer` operators can vouch |
| **Vouch limit** | Each operator can vouch for at most N operators per time period (prevents vouching spam) |
| **Accountability** | If a vouched operator's contribution is rejected for security reasons, the voucher's trust score is noted (not automatically penalized, but the event is logged for maintainer review) |
| **Cross-hive vouching** | An operator trusted in Hive A can vouch for someone joining Hive B — the vouch carries the evidence but Hive B's maintainer decides |
| **Vouch revocation** | A voucher can revoke their vouch if circumstances change |
| **No transitive vouching** | B vouched by A cannot vouch for C using A's trust — vouching is direct, not chained |

### Hive Configuration

Hives can configure their vouching policy in `hive.yaml`:

```yaml
trust:
  vouching:
    enabled: true
    required_for_join: false       # true = invite-only via vouch (DC hub model)
    min_voucher_zone: trusted      # minimum trust zone to vouch
    max_vouches_per_month: 5       # prevents vouching spam
    cross_hive: true               # accept vouches from operators in other hives
```

**Hive type implications:**
- **Open hive:** vouching optional, accelerates trust but not required to join
- **Closed hive:** vouching required (`required_for_join: true`) — the Direct Connect model
- **Enterprise hive:** vouching typically disabled (SSO group membership replaces it)

## Dimension 4: Trust Scoring (Designed)

Quantitative trust that compounds over time and is portable across hives. Modeled after marketplace seller ratings (TradeMe, eBay) and professional network endorsements (LinkedIn) — but backed by verified work, not self-reported skills.

### The Feedback Model

Every completed interaction generates a feedback event: **positive**, **neutral**, or **negative**. The breakdown is visible on the operator's profile — anyone can see their track record.

```
┌─────────────────────────────────────────────────────┐
│  @andreas · security-tools hive                      │
│                                                      │
│  ✅ Positive: 37    ⬜ Neutral: 4    ❌ Negative: 1 │
│  ████████████████████████████░░░░░                    │
│  Trust zone: Trusted · Member since 2026-01            │
│                                                      │
│  Recent feedback:                                    │
│  ✅ "Clean implementation, thorough tests" — @jordan │
│  ✅ "Excellent threat model" — @sam                  │
│  ⬜ "Good work, needed revision on edge cases" — @kim│
└─────────────────────────────────────────────────────┘
```

### What Generates Feedback

| Event | Rating | Who Rates | Mechanism |
|-------|--------|-----------|-----------|
| **PR merged (first attempt)** | Positive | Reviewer + maintainer | PR merged with approval, no revision cycles |
| **PR merged (after revisions)** | Neutral | Reviewer | PR required changes but ultimately merged |
| **PR rejected** | Negative | Reviewer + maintainer | PR closed without merge (quality/security/relevance) |
| **Review given (helpful)** | Positive | PR author | Author marks review as helpful |
| **Review given (unhelpful)** | Neutral | PR author | Author marks review as unhelpful or not actionable |
| **Swarm completed (on time)** | Positive | Swarm peers | Swarm delivered, operator met their role commitments |
| **Swarm completed (operator lagged)** | Neutral | Swarm peers | Swarm delivered but this operator caused delays |
| **Work abandoned** | Negative | Maintainer | Operator claimed work and didn't deliver within deadline |
| **Security violation** | Negative | Automated | Secret leak detected, content filter triggered on submitted code |
| **Vouch successful** | Positive | System | Vouched operator earns 5+ positive ratings (voucher rewarded) |
| **Vouch failed** | Negative | System | Vouched operator earns 3+ negative ratings (voucher penalized) |

### Scoring Formula

```
trust_score = (positive - negative) / total_feedback
trust_percentage = positive / total_feedback * 100

Example:
  37 positive, 4 neutral, 1 negative = 42 total
  trust_score = (37 - 1) / 42 = 0.857
  trust_percentage = 37/42 = 88.1% positive
```

**Display:** Like TradeMe — show the raw counts AND the percentage. `88% positive (42 ratings)`. Operators can see the breakdown. Maintainers can read individual feedback comments.

### Trust Score Schema

```yaml
# In operator's hive-scoped profile (Tier 2 of operator.yaml)
hives:
  - hive: mellanon/security-tools
    trust_zone: trusted
    feedback:
      positive: 37
      neutral: 4
      negative: 1
      total: 42
      percentage: 88.1
    feedback_log:                        # last N entries, append-only
      - type: positive
        from: jordan
        context: "PR #47 merged"
        comment: "Clean implementation, thorough tests"
        timestamp: 2026-02-05T14:30:00Z
      - type: neutral
        from: kim
        context: "PR #52 merged after revision"
        comment: "Good work, needed revision on edge cases"
        timestamp: 2026-02-04T09:15:00Z
```

### Zone Promotion Thresholds

Maintainers can set thresholds for trust zone promotions in `hive.yaml`:

```yaml
trust:
  scoring:
    promotion_thresholds:
      trusted:
        min_positive: 5                # at least 5 positive ratings
        min_percentage: 80             # at least 80% positive
        min_age_days: 14               # member for at least 2 weeks
      maintainer:
        min_positive: 20               # at least 20 positive ratings
        min_percentage: 90             # at least 90% positive
        min_age_days: 90               # member for at least 3 months
        vouches_from_maintainers: 1    # at least 1 maintainer vouch
```

**Promotion is still human-gated.** Thresholds are eligibility criteria, not automatic promotion. The maintainer reviews the profile and decides.

### Cross-Hive Portability

An operator's feedback history is visible across hives (Tier 1 of operator profile — public). When joining a new hive:

- New hive sees: `@andreas — 88% positive (42 ratings) in security-tools, 92% positive (24 ratings) in community-tools`
- This is **evidence**, not authority — the new hive's maintainer decides if this warrants faster promotion
- Feedback earned in one hive does NOT transfer as ratings in another — each hive maintains its own score
- Cross-hive summary is read-only social proof, like a LinkedIn recommendation from another company

### Decay

Inactive trust decays:
- No contributions in 90 days → feedback score grayed out on profile ("Inactive")
- No contributions in 180 days → feedback score archived (still visible but marked historical)
- Returning operators restart their active feedback counter but keep their historical record

### Design Principles

- **Feedback is from verified work, not votes.** You can't "like" someone's profile. Feedback comes from code reviews, PR merges, swarm completions — auditable events.
- **Negative feedback requires explanation.** A reviewer can't give negative feedback without a comment explaining why. This prevents drive-by downvoting.
- **Feedback is append-only.** Once recorded, feedback cannot be edited or deleted. The git history IS the audit trail.
- **Raw data is always visible.** Never just show a percentage — always show `37 positive, 4 neutral, 1 negative`. Let people make their own assessment.

## Observability

Trust, work, and collaboration events need to be observable — not just for debugging but for understanding how the network operates.

### Event Model

The Hive extends PAI's Signal observability framework with hive-specific event types:

| Event Type | What It Records | Layer |
|-----------|----------------|-------|
| `hive.join` | Operator joins a hive (identity provider, vouch if any) | Hub |
| `hive.leave` | Operator leaves or is removed from a hive | Hub |
| `spoke.publish` | Operator's spoke status updated | Spoke |
| `work.posted` | New work item created | Hub |
| `work.claimed` | Operator claims work | Hub + Local |
| `work.completed` | Work passes three-gate verification | Hub |
| `work.abandoned` | Work claimed but not delivered | Hub |
| `swarm.formed` | Swarm sealed with assigned roles | Hub |
| `swarm.dissolved` | Swarm completes and dissolves | Hub |
| `trust.feedback` | Positive/neutral/negative feedback recorded | Hub |
| `trust.promotion` | Operator promoted to new trust zone | Hub |
| `trust.vouch` | Operator vouches for another | Hub |
| `skill.installed` | Operator installs a skill from the network | Local |
| `skill.published` | Skill registered in hub's skill registry | Hub |
| `security.secret_detected` | Secret scanning blocked a commit | Local |
| `security.content_flagged` | Content filter flagged inbound content | Local |

### Three-Layer Observability (from PAI Signal)

| Layer | How | What You Get |
|-------|-----|-------------|
| **Layer 0: CLI** | `jq` queries on JSONL event files | Raw power, zero dependencies |
| **Layer 1: Scripts** | Pre-built query scripts (e.g., `trust-trends.sh`, `swarm-activity.sh`) | One-liner answers to common questions |
| **Layer 2: Dashboard** | `blackboard serve` with web UI, or Grafana stack | Visual browsing, trace waterfall, time-series trends |

### Event Storage

Events are stored as append-only JSONL — one file per day, per level:

```
~/.pai/events/local/2026-02-06.jsonl     # Local events
~/.pai/events/spoke/2026-02-06.jsonl     # Spoke events
~/.pai/events/hub/2026-02-06.jsonl       # Hub events (pulled from hive)
```

The same 90-day rotation and PII scrubbing from PAI Signal applies.

## Prior Art

| Source | What it provides |
|--------|-----------------|
| [pai-collab TRUST-MODEL.md](https://github.com/mellanon/pai-collab) | Three zones, six defense layers, two-level scoping |
| [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | Outbound security (Layers 1-2). 11 custom + ~150 built-in patterns. |
| [pai-content-filter](https://github.com/jcfischer/pai-content-filter) | Inbound security (Layers 4-5-6). 34 patterns, sandbox, audit trail. 389 tests. |
| ivy-blackboard event log | Append-only audit trail of all agent actions |
| pai-collab CONTRIBUTORS.yaml | Per-operator trust zone assignments |
| PAI Signal | Three-layer observability framework (CLI → scripts → Grafana). Event schema, JSONL storage, PII scrubbing, 90-day rotation. |
| TradeMe seller ratings | Positive/neutral/negative feedback model with visible breakdown. Social proof through verified transactions. |
| LinkedIn endorsements | Professional network with skill endorsements and recommendations — but self-reported, not work-verified. |
| Direct Connect hub referrals | Invitation-only access via trusted member vouching. Voucher reputation at stake. |
