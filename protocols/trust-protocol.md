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

## Dimension 4: Trust Scoring (Planned)

Quantitative trust that compounds over time and is portable across hives.

### Open Questions

1. How is trust scored quantitatively? (Contribution count? Review quality? Swarm success rate?)
2. What events build trust? (Code merged? Review completed? Swarm delivered? Vouching record?)
3. What events reduce trust? (Abandoned work? Failed reviews? Timeout? Vouched operator misconduct?)
4. How is trust verified by third parties? (Cryptographic signatures? Public ledger?)

### Design Considerations

- Trust scoring must build on the security infrastructure (Dimensions 1-2) and vouching (Dimension 3), not replace them
- Vouching history is a trust signal — operators who vouch well (their vouchees succeed) build reputation; operators who vouch poorly lose credibility
- Scores should be derived from auditable events — the same audit trail that powers security
- ivy-blackboard's event log and pai-content-filter's audit trail are the raw data sources
- The spoke contract (`.collab/status.yaml`) is the projection mechanism for trust data to the hub

## Prior Art

| Source | What it provides |
|--------|-----------------|
| [pai-collab TRUST-MODEL.md](https://github.com/mellanon/pai-collab) | Three zones, six defense layers, two-level scoping |
| [pai-secret-scanning](https://github.com/jcfischer/pai-secret-scanning) | Outbound security (Layers 1-2). 11 custom + ~150 built-in patterns. |
| [pai-content-filter](https://github.com/jcfischer/pai-content-filter) | Inbound security (Layers 4-5-6). 34 patterns, sandbox, audit trail. 389 tests. |
| ivy-blackboard event log | Append-only audit trail of all agent actions |
| pai-collab CONTRIBUTORS.yaml | Per-operator trust zone assignments |
