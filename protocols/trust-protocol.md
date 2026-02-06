# Trust Protocol

> **Status:** Planned

## Purpose

The trust protocol defines how trust is earned through verified contributions, how it's scored, and how it can be portable across hives.

## Principles

1. **Earned, not claimed.** Trust comes from verified work, not self-reported skills.
2. **Auditable.** Every trust-building event is recorded in an immutable ledger.
3. **Portable.** Trust earned in one hive should be recognizable in another.
4. **Decaying.** Inactive trust decays over time — reputation must be maintained.
5. **Git-based.** The trust ledger is auditable through git history, not blockchain.

## Trust Zones (from pai-collab)

The existing three-zone model:

| Zone | Description | Permissions |
|------|-------------|------------|
| **Untrusted** | Default for new operators | Full scanning, tool restrictions, detailed logging |
| **Trusted** | Demonstrated track record | Promoted explicitly, still passes security layers |
| **Maintainer** | Governance authority | Can promote/demote, merge, modify SOPs |

## Open Questions

1. How is trust scored quantitatively? (Contribution count? Review quality? Swarm success rate?)
2. How does trust transfer across hives? (Vouching? Federation? Shared ledger?)
3. What events build trust? (Code merged? Review completed? Swarm delivered?)
4. What events reduce trust? (Abandoned work? Failed reviews? Timeout?)
5. How is trust verified by third parties? (Cryptographic signatures? Public ledger?)

## Prior Art

- pai-collab [TRUST-MODEL.md](https://github.com/mellanon/pai-collab) — three zones, six defense layers
- ivy-blackboard event log — append-only audit trail of all agent actions
