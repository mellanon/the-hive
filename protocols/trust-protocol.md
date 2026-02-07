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

## The Reflex Pipeline

The six defense layers are not independent tools — they form a **unified reflex pipeline** that fires at every boundary crossing between hive layers. The term "reflex" comes from [Arbor](https://github.com/trust-arbor/arbor)'s security architecture: fast, pattern-based checks that fire automatically before any authorization or processing occurs.

### Reflex Firing Points

Reflexes fire at **boundary crossings** — every time content moves from one layer to another:

```
OUTBOUND (operator → network)
═══════════════════════════════════════════════════════════════════

  LOCAL                        SPOKE                         HUB
  ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
  │ Agent writes │            │ .collab/     │            │ PR arrives   │
  │ code         │──commit──▶ │ manifest     │──push/PR──▶│              │
  │              │    │       │ status       │    │       │              │
  └──────────────┘    │       └──────────────┘    │       └──────────────┘
                      ▼                           ▼
               ┌─────────────┐             ┌─────────────┐
               │ REFLEX A    │             │ REFLEX B    │
               │ Pre-commit  │             │ CI gate     │
               │             │             │             │
               │ L1: Secret  │             │ L2: Secret  │
               │   scanning  │             │   scanning  │
               │ Commit      │             │ Signing     │
               │   signing   │             │   verify    │
               │ enforced    │             │ Schema      │
               └─────────────┘             │   checks    │
                                           └─────────────┘


INBOUND (network → operator)
═══════════════════════════════════════════════════════════════════

  HUB                          SPOKE                        LOCAL
  ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
  │ Shared       │            │ Cloned       │            │ Agent loads  │
  │ content      │──clone/──▶ │ content      │──context──▶│ content into │
  │ (repos,PRs,  │  pull │    │ (repos,      │  load │    │ LLM context  │
  │  skills)     │       │    │  skills)     │       │    │              │
  └──────────────┘       │    └──────────────┘       │    └──────────────┘
                         ▼                           ▼
                  ┌─────────────┐             ┌─────────────┐
                  │ REFLEX C    │             │ REFLEX D    │
                  │ Acquisition │             │ Context     │
                  │             │             │ load        │
                  │ Sandbox     │             │             │
                  │   enforcer  │             │ L4: Content │
                  │ Quarantine  │             │   scanning  │
                  │   clone dir │             │ L5: Tool    │
                  └─────────────┘             │   restrict  │
                                              │ L6: Audit   │
                                              │   log       │
                                              └─────────────┘
```

### The Four Reflexes

| Reflex | Boundary | Direction | What Fires | Enforcement |
|--------|----------|-----------|------------|-------------|
| **A: Pre-commit** | Local → Spoke | Outbound | Secret scanning, commit signing | Git pre-commit hook (local) |
| **B: CI Gate** | Spoke → Hub | Outbound | Secret scanning, signing verification, schema validation | GitHub Actions (hub CI) |
| **C: Acquisition** | Hub → Spoke | Inbound | Sandbox enforcer, quarantine directory | PreToolUse hook on `git clone`, `curl` |
| **D: Context Load** | Spoke → Local | Inbound | Content filter, tool restrictions, audit trail | LoadContext hook, PreToolUse hook |

### How Each Reflex Is Enforced (Non-Optional)

The critical question: how do reflexes fire without the operator choosing to enable them?

**Reflex A (Pre-commit):** Installed locally by the operator. This is the ONE step that requires operator action — `pai-secret-scanning install` adds the git pre-commit hook. If the operator skips this, **Reflex B catches it** at the hub. Defense in depth.

**Reflex B (CI Gate):** Runs in the hub's GitHub Actions pipeline. The operator does NOT install this — it's part of the hive's infrastructure. Every PR triggers it. No opt-out. This is the enforcement backstop for Reflex A.

**Reflex C (Acquisition):** Enforced via the agent's hook system (PreToolUse). When an agent runs `git clone` or `curl` on external content, the sandbox enforcer intercepts the command and redirects to a quarantine directory for scanning. The operator installs pai-content-filter once; it hooks into the agent's tool pipeline automatically.

**Reflex D (Context Load):** Enforced via the agent's hook system (LoadContext/PreToolUse). When an agent loads markdown from a shared repository into LLM context, content filtering scans it first. Flagged content triggers tool restrictions (read-only mode). This fires automatically on every context load — no per-file opt-in.

### Enforcement Summary by Layer

| Layer | What Operator Installs | What Fires Automatically |
|-------|----------------------|-------------------------|
| **Local** | pai-secret-scanning (pre-commit hook), pai-content-filter (agent hooks) | Reflex A on every commit, Reflex D on every context load |
| **Spoke** | Nothing — spoke is just YAML files | Reflexes A and D fire during spoke generation (commit + read) |
| **Hub** | Nothing — CI is hive infrastructure | Reflex B on every PR, Reflex C on every acquisition |

**Key principle:** The operator installs two things locally (secret scanning + content filter). Everything else fires automatically at the hub via CI. If an operator contributes without installing local tools, the hub's CI catches outbound issues (Reflex B) and the receiving agent's hooks catch inbound issues (Reflex D).

### Per-Operation Reflex Map

Every operation in the hive triggers specific reflexes:

| Operation | Reflexes That Fire | What Gets Checked |
|-----------|-------------------|-------------------|
| `git commit` (local) | A | Secrets in staged changes, commit is signed |
| `git push` to spoke | — | No new reflex (Reflex A already fired at commit) |
| Submit PR to hub | B | Secrets (re-scan), signing verification, schemas, issue references |
| Merge PR | — | Human review (Layer 3) — maintainer decision |
| `git clone` external repo | C | Sandbox enforcer quarantines, content scanned before access |
| Agent loads markdown from PR | D | Content filter scans, flagged → tool restrictions, audit logged |
| `blackboard skills install` | C + D | Quarantine clone (C), scan skill content before loading (D) |
| Agent reads JOURNAL.md from PR | D | Content filter scans entry before LLM context |
| `blackboard pull --level hub` | D | Spoke status data scanned before processing |

### Reflex Configuration

Hives can configure reflex behavior in `hive.yaml`:

```yaml
trust:
  reflexes:
    outbound:
      secret_scanning: required          # pre-commit + CI
      patterns: default + custom         # gitleaks config location
    inbound:
      content_filter: required           # content scanning on context load
      quarantine_external: true          # sandbox enforcer for git clone/curl
      tool_restrictions_on_flag: true    # read-only mode when content flagged
    audit:
      log_all_loads: true               # log every content loading decision
      log_format: jsonl                  # append-only JSONL
      retention_days: 90                # rotation policy
```

### Prior Art

The reflex pipeline concept is adapted from [Arbor](https://github.com/trust-arbor/arbor)'s reflex system, which implements fast pattern-based checks (regex, path matching) that fire before capability authorization. Arbor's reflexes are in-process (Elixir GenServer); The Hive's reflexes are at git boundaries (hooks and CI). Same principle, different enforcement mechanism:

| Arbor Reflex | Hive Reflex | Parallel |
|-------------|-------------|----------|
| `rm_rf_root` pattern block | Secret scanning catches `rm -rf /` in scripts | Dangerous command detection |
| `ssh_private_keys` path block | Secret scanning catches `~/.ssh/id_*` content | Credential leak prevention |
| `ssrf_metadata` pattern block | Content filter catches cloud metadata URLs | SSRF prevention |
| Custom reflex registration | Custom gitleaks patterns + content filter rules | Extensible pattern matching |

## Commit Signing as Trust Foundation

Signed commits are the cryptographic backbone of the trust protocol. Every operator's Ed25519 SSH key (see [Operator Identity](operator-identity.md)) signs their commits, creating a verifiable chain of authorship.

**Trust signals from signing:**

| Signal | What It Means |
|--------|--------------|
| Commit signed, key in allowed-signers | Known operator, verified authorship |
| Commit signed, key NOT in allowed-signers | Unknown operator — requires onboarding |
| Commit unsigned | No cryptographic identity — treated as untrusted regardless of GitHub account |
| Commit signed with revoked key | Operator's signing authority was revoked — reject |

**Signing enhances every defense layer:**

| Layer | Without Signing | With Signing |
|-------|----------------|--------------|
| Layer 1 (pre-commit) | Scans for secrets | Scans for secrets + ensures commit will be signed |
| Layer 2 (CI gate) | Scans for secrets | Scans for secrets + rejects unsigned commits |
| Layer 3 (fork-and-PR) | Review boundary | Review boundary + cryptographic authorship verification |
| Layers 4-6 | Applied by trust zone | Applied by trust zone + signing status is an input to zone determination |

## Content Provenance

Every content artifact entering the hub carries a provenance label — tracking where it came from and how it was produced. This is a lightweight form of taint tracking (inspired by [Arbor](https://github.com/trust-arbor/arbor)'s four-level taint model) adapted for git-based workflows.

### Provenance Labels

| Label | Meaning | Trust Implication |
|-------|---------|-------------------|
| `origin: operator` | Directly authored by a known operator | Highest trust — human judgment applied |
| `origin: agent` | Generated by the operator's AI agent(s) | Operator attests to quality, but content is machine-generated |
| `origin: external` | Imported from outside the hive | Lower trust — source may not follow hive standards |
| `origin: mixed` | Combination of operator and agent authorship | Common case — operator guides, agent produces |

### How Provenance is Tracked

Provenance is declared via **git commit trailers** (structured metadata at the end of commit messages):

```
Add spoke validation schema

Implements manifest.yaml and status.yaml validation against
the spoke protocol schema definitions.

Origin: agent
Attested-By: mellanon
Signed-off-by: Andreas <andreas@example.com>
```

**Rules:**
- `Origin` trailer is recommended on all commits to hive repositories
- `Attested-By` is required when `Origin: agent` — the operator takes responsibility
- Provenance does not change trust zone — it informs review decisions
- Reviewers can use provenance to calibrate scrutiny (agent-generated code may need closer review)
- Provenance is append-only in git history — it cannot be retroactively changed

### Why This Matters

When reviewing a PR, knowing whether content was human-authored vs. LLM-generated vs. imported matters for trust decisions. A security-critical change authored by an operator carries different weight than one generated by an agent. Provenance makes this visible without adding friction — it's a commit trailer, not a ceremony.

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

### Dimension Ratings

In addition to the overall positive/neutral/negative, each review includes dimension-specific ratings (1-5 stars) — inspired by Airbnb's category ratings. These capture what _kind_ of collaborator someone is:

| Dimension | What It Measures |
|-----------|-----------------|
| **Technical Quality** | Did the work meet standards? Tests pass? Code clean? |
| **Communication** | Were they responsive, clear, and proactive? |
| **Reliability** | Did they meet commitments and deadlines? |
| **Collaboration** | Were they good to work with? Constructive in reviews? |

Plus an independent **Overall** rating that captures the gestalt — the overall experience of working with this operator. The overall is NOT computed from the dimensions. An operator can score 5 on every dimension and still get a 4 overall (or vice versa).

**Display:** Dimension ratings are visible on the operator's hive-scoped profile. The aggregate across all ratings shows a per-dimension average:

```
@andreas · security-tools hive
  Technical: ★★★★★ (4.9)    Communication: ★★★★★ (4.8)
  Reliability: ★★★★☆ (4.6)  Collaboration: ★★★★★ (5.0)
  Overall: ★★★★★ (4.9)      37 reviews
```

### Double-Blind Reviews

Reviews are revealed simultaneously. Neither party sees the other's review until both have submitted (or the 14-day window expires).

**Why this is critical for professional networks:** Without double-blind, reviews become a coordination game — both parties leave positive reviews to avoid retaliation. Research by Fradkin et al. (Marketing Science, 2021) showed that Airbnb's simultaneous reveal increased honest negative feedback by 12-17%.

**Mechanism:**
1. Work completes (PR merged, swarm dissolved)
2. Both parties have 14 days to submit reviews
3. Reviews are stored encrypted until both submit or the window closes
4. Reveal: both reviews become visible simultaneously
5. If only one party reviews, that review is revealed after 14 days

**Who reviews whom:**

| Interaction | Party A Reviews | Party B Reviews |
|-------------|----------------|----------------|
| PR submission | Reviewer reviews contributor | Contributor reviews reviewer's feedback quality |
| Swarm work | Peers review each other | Each operator reviews their swarm experience |
| Maintainer review | Maintainer reviews operator | Operator reviews maintainer's governance |

### Design Principles

- **Feedback is from verified work, not votes.** You can't "like" someone's profile. Feedback comes from code reviews, PR merges, swarm completions — auditable events.
- **Negative feedback requires explanation.** A reviewer can't give negative feedback without a comment explaining why. This prevents drive-by downvoting.
- **Feedback is append-only.** Once recorded, feedback cannot be edited or deleted. The git history IS the audit trail.
- **Raw data is always visible.** Never just show a percentage — always show `37 positive, 4 neutral, 1 negative`. Let people make their own assessment.
- **Double-blind is non-negotiable.** Simultaneous reveal prevents retaliation and review inflation. No exceptions.

## Dimension 5: Badges (Designed)

Badges are earned certifications that signal sustained excellence. Inspired by Airbnb's Superhost program: measurable criteria, rolling-window evaluation, you can lose them.

**Design constraint:** Few badges, high bar. Badge inflation destroys trust signals. Four badges maximum — each represents a distinct type of value to the network.

### The Four Badges

#### 1. Hive Star

The primary excellence badge — like Airbnb's Superhost. Signals: "this operator consistently delivers quality work."

| Criterion | Threshold | Window |
|-----------|-----------|--------|
| Overall rating | 4.8+ stars | Trailing 12 months |
| Positive feedback | 90%+ positive | Trailing 12 months |
| Completed work items | 10+ | Trailing 12 months |
| Abandonment rate | < 5% | Trailing 12 months |
| Response time | Signal interest within 48h of matching work posted | Trailing 12 months |
| Security record | Zero security violations | Trailing 12 months |

**What it signals:** Reliable, high-quality contributor. Safe to work with.
**What it unlocks:** Priority in work matching, visible badge on profile, faster trust promotion consideration.

#### 2. Guardian

Security and quality assurance excellence. Signals: "this operator makes the network safer."

| Criterion | Threshold | Window |
|-----------|-----------|--------|
| Reviews given | 15+ peer reviews | Trailing 12 months |
| Review helpfulness | 85%+ rated helpful | Trailing 12 months |
| Security contributions | 3+ security-related work items completed | Trailing 12 months |
| Clean scanning record | Zero secrets detected in own submissions | Trailing 12 months |
| Technical Quality dimension | 4.7+ average | Trailing 12 months |

**What it signals:** Trusted security reviewer. Catches issues others miss.
**What it unlocks:** Eligible for security-sensitive work items, trusted reviewer role in swarms.

#### 3. Architect

Technical leadership excellence. Signals: "this operator designs systems that work."

| Criterion | Threshold | Window |
|-----------|-----------|--------|
| Architect role in swarms | 3+ swarms as architect | Trailing 12 months |
| Swarm success rate | 90%+ of architect-led swarms completed | Trailing 12 months |
| Collaboration dimension | 4.8+ average from swarm peers | Trailing 12 months |
| Skill published | 1+ verified skill in network registry | Ever |
| Hive Star badge | Must hold Hive Star | Current |

**What it signals:** Proven technical leader. Can decompose complex work and guide a swarm.
**What it unlocks:** Auto-considered for architect role when swarms form, voice in governance decisions.

#### 4. Catalyst

Community building and mentorship excellence. Signals: "this operator makes others better."

| Criterion | Threshold | Window |
|-----------|-----------|--------|
| Vouches given | 3+ successful vouches | Trailing 12 months |
| Vouchee success rate | 80%+ of vouched operators remain in good standing | Trailing 12 months |
| Reviews given | 20+ reviews | Trailing 12 months |
| Review quality | 90%+ rated helpful | Trailing 12 months |
| Cross-hive activity | Active in 2+ hives | Current |

**What it signals:** Community builder. Invests in others. Grows the network.
**What it unlocks:** Increased vouch limit, recognized mentor status, eligible for governance roles.

### Badge Evaluation

**Cadence:** Quarterly (every 90 days). Same as Airbnb Superhost.

**Rolling window:** Each evaluation looks at the trailing 12 months of data. Not a fixed period — a sliding window.

**How you lose a badge:**
- Fail any criterion at a quarterly evaluation → badge removed immediately
- No grace period, no "one strike" policy
- Recovery: meet all criteria again → badge restored at next quarterly evaluation
- Badge history is permanent: "Hive Star (3 consecutive quarters)" or "Hive Star (lost Q3 2026, regained Q4 2026)"

**Badge hierarchy:** Architect requires Hive Star. This prevents someone from earning a leadership badge without first proving they're a reliable contributor.

### Badge Schema

```yaml
# In operator.yaml (Tier 2 — hive-scoped)
hives:
  - hive: mellanon/security-tools
    badges:
      - badge: hive-star
        earned: 2026-04-01
        consecutive_quarters: 3
        next_evaluation: 2026-07-01
        status: active
      - badge: guardian
        earned: 2026-07-01
        consecutive_quarters: 1
        next_evaluation: 2026-10-01
        status: active

# In hive.yaml (configurable thresholds)
trust:
  badges:
    enabled: true
    evaluation_cadence_days: 90
    thresholds:
      hive_star:
        min_overall_rating: 4.8
        min_positive_percentage: 90
        min_completed_items: 10
        max_abandonment_rate: 5
        max_response_hours: 48
      guardian:
        min_reviews_given: 15
        min_review_helpfulness: 85
        min_security_items: 3
      architect:
        min_swarms_as_architect: 3
        min_swarm_success_rate: 90
        min_collaboration_rating: 4.8
        requires_badge: hive_star
      catalyst:
        min_successful_vouches: 3
        min_vouchee_success_rate: 80
        min_reviews_given: 20
        min_review_helpfulness: 90
        min_hive_count: 2
```

### Why Four and Only Four

| Badge | Value Type | Network Need |
|-------|-----------|-------------|
| **Hive Star** | Consistent delivery | "Can I trust this person to deliver?" |
| **Guardian** | Quality assurance | "Can I trust this person to catch problems?" |
| **Architect** | Technical leadership | "Can I trust this person to lead a complex effort?" |
| **Catalyst** | Community growth | "Does this person make the network stronger?" |

These four cover the essential trust questions in a collaboration network. Adding more dilutes the signal. If a new badge is proposed, it must answer a trust question that none of the existing four address.

### Anti-Patterns

- **No "years of service" badge.** Tenure is not trust. An operator who's been around for 2 years with mediocre work doesn't deserve a badge.
- **No "number of contributions" badge.** Volume without quality is noise. The Hive Star already requires 10+ items — but also requires 90%+ positive and 4.8+ rating.
- **No self-nominated badges.** All criteria are measured from auditable events. No applications, no committees.
- **No permanent badges.** Every badge is re-evaluated quarterly. Past performance does not guarantee future status.

## Observability

Trust, work, and collaboration events need to be observable — not just for debugging but for understanding how the network operates.

### Event Model

The Hive defines hive-specific event types following the PAI Signal observability design (Signal is designed and merged into PAI but not yet deployed — the event schema and JSONL storage patterns are the reference, not a running dependency):

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
| [Arbor](https://github.com/trust-arbor/arbor) | Ed25519 identity, capability-based access, four-level taint tracking, reflex system. The Hive adapts: crypto identity (via git SSH signing), content provenance (taint tracking lite), unified reflex pipeline. See [security story](https://azmaveth.com/posts/arbor-security-story/). |
| [pai-collab TRUST-MODEL.md](https://github.com/mellanon/pai-collab) | Three zones, six defense layers, two-level scoping |
| [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | Outbound security (Layers 1-2). 11 custom + ~150 built-in patterns. |
| [pai-content-filter](https://github.com/jcfischer/pai-content-filter) | Inbound security (Layers 4-5-6). 34 patterns, sandbox, audit trail. 389 tests. |
| ivy-blackboard event log | Append-only audit trail of all agent actions |
| pai-collab CONTRIBUTORS.yaml | Per-operator trust zone assignments |
| PAI Signal | Three-layer observability framework (CLI → scripts → Grafana). Event schema, JSONL storage, PII scrubbing, 90-day rotation. |
| TradeMe seller ratings | Positive/neutral/negative feedback model with visible breakdown. Social proof through verified transactions. |
| LinkedIn endorsements | Professional network with skill endorsements and recommendations — but self-reported, not work-verified. |
| Direct Connect hub referrals | Invitation-only access via trusted member vouching. Voucher reputation at stake. |
