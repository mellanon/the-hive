# Skill Protocol

> **Status:** Draft
> **Resolved:** 2026-02-06

![Skill Marketplace — downloadable capabilities](../assets/concepts/hivemind-sketch-5-skills.png)

## Purpose

The skill protocol defines how operator capabilities (AI skills) are packaged, published to the network, discovered by other operators, and installed locally.

## What is a Skill?

A downloadable capability for an operator's AI. Like installing expertise — analysis frameworks, security scanning, content filtering, domain-specific workflows. Skills distribute through the network. Creators earn recognition from adoption.

## Packaging Format

A skill is a directory with a standard structure, packaged as a git repository (or subdirectory of one). The canonical format follows the PAI skill system:

### skill-manifest.yaml

Every network-distributable skill has a `skill-manifest.yaml` at its root:

```yaml
schemaVersion: "1.0"
name: <skill-name>
version: <semver>                       # e.g., "1.2.0"
description: <one-line description>
author:
  handle: <github-handle>
  name: <display-name>                  # optional
license: MIT | Apache-2.0 | BSD-2-Clause | BSD-3-Clause
repository: <org/repo>                  # source of truth
platform:
  minimum: <platform-version>           # e.g., "PAI 2.5"
  tested:
    - "PAI 2.5"
    - "Claude Code 1.x"
dependencies:                           # optional
  skills: []                            # other skills required
  tools: []                             # external tools required (e.g., "bun", "gitleaks")
keywords:
  - <topic-tag>
  - <domain-tag>
structure:
  entry: SKILL.md                       # main skill file
  workflows: workflows/                 # workflow directory
  components: Components/               # component directory
```

### Directory Structure

```
my-skill/
├── skill-manifest.yaml    # Network metadata (new — for distribution)
├── SKILL.md               # Skill definition (existing PAI format)
├── workflows/             # Workflow definitions
│   ├── workflow-a.md
│   └── workflow-b.md
├── Components/            # Reusable components
│   └── ...
└── README.md              # Human documentation
```

**Key insight:** `skill-manifest.yaml` is the ONLY new file added for network distribution. Everything else follows the existing PAI skill structure. `SKILL.md` remains the skill definition. The manifest wraps it for the network.

## Design Decisions

### 1. How are skills published to the network?

**Decision:** Git push to a public repository + registration in a hub's skill registry.

**Mechanism:**
1. **Author creates skill** following the PAI structure + adds `skill-manifest.yaml`
2. **Author validates** with `blackboard validate --level spoke` (checks manifest schema, SKILL.md presence, license)
3. **Author pushes** to their public git repository
4. **Author registers** by submitting a PR to a hive's `skills/REGISTRY.yaml`:

```yaml
# skills/REGISTRY.yaml (at hub level)
skills:
  - name: specflow
    version: "1.0.0"
    repository: jcfischer/specflow
    author: jcfischer
    license: MIT
    keywords: [specification, development, lifecycle]
    verified: true                      # maintainer has reviewed
  - name: content-filter
    version: "0.1.0"
    repository: jcfischer/pai-content-filter
    author: jcfischer
    license: MIT
    keywords: [security, inbound, prompt-injection]
    verified: true
```

5. **Hub maintainer reviews** the registration PR (validates manifest, checks license, reviews quality)
6. **Merged = published.** The skill is now discoverable through the hub.

**Rationale:** Git repo as package, registry PR as publish. Zero custom infrastructure. The skill's git repo is the source of truth. The hub registry is discovery.

### 2. How do operators discover skills?

**Decision:** Three tiers, matching the work discovery model.

| Tier | Mechanism | When |
|------|-----------|------|
| **Direct link** | Someone shares the repo URL | Word of mouth, docs, recommendations |
| **Hub registry** | Browse/search `skills/REGISTRY.yaml` | Looking for capabilities in a hive |
| **Cross-hive** | Operator profiles list installed skills | Seeing what other operators use |

**CLI support:**
```bash
blackboard skills list --level hub              # List registered skills in current hive
blackboard skills search --keyword security     # Search by keyword
blackboard skills info <skill-name>             # Show manifest details
```

**Rationale:** Same discovery model as hives and work items — direct, registry, cross-reference. No centralized marketplace needed initially.

### 3. How is skill authorship attributed and verified?

**Decision:** Git history is the source of truth. The `skill-manifest.yaml` declares the author. The git repo proves ownership.

**Mechanism:**
- `author.handle` in manifest matches the repo owner
- Git commit history shows contribution timeline
- Hub registry `verified: true` flag means a hive maintainer has reviewed the skill
- Skill enforcer (existing tool) validates structural integrity

**Rationale:** Git already solves authorship and provenance. No additional attribution system needed.

### 4. How do skill versions and updates propagate?

**Decision:** Semver in `skill-manifest.yaml`. Operators pull updates explicitly. No auto-update.

**Mechanism:**
1. Author bumps `version` in `skill-manifest.yaml` and pushes
2. Author updates the hub registry entry via PR (or automated CI)
3. Operators see version changes when they check the registry
4. Operators update explicitly: `blackboard skills update <skill-name>`

**Rationale:** Auto-update for AI capabilities is dangerous — a skill update could change agent behavior in unexpected ways. Operators must consciously choose to update. This matches how PAI skills work locally today.

### 5. Can skills have dependencies on other skills?

**Decision:** Yes, declared in `skill-manifest.yaml` under `dependencies.skills`. Resolved at install time.

**Mechanism:**
- `dependencies.skills` lists skill names required for this skill to function
- `dependencies.tools` lists external tools required (e.g., `bun`, `gitleaks`, `llamaparse`)
- At install time, the CLI checks if dependencies are present and warns if missing
- No automatic transitive dependency resolution (keep it simple — skills are not npm packages)

**Rationale:** Some skills naturally build on others (e.g., a security review skill might depend on content-filter). But deep dependency trees create fragility. Flat dependencies with manual resolution is the right tradeoff for now.

### 6. How are skills installed?

**Decision:** Clone + symlink, managed by the blackboard CLI.

**Mechanism:**
```bash
blackboard skills install <org/repo>            # Clone and link
blackboard skills install <skill-name>          # Install from hub registry by name
blackboard skills list --level local            # Show installed skills
blackboard skills remove <skill-name>           # Unlink and optionally remove
```

Installed skills are cloned to a local skills directory and symlinked into the operator's PAI skill path. The operator's `operator.yaml` is updated to reflect installed skills.

**Rationale:** Git clone is the install mechanism. Symlinks are how PAI already manages skills (Layer 2 of PAI's symlink architecture). No package manager needed.

## Prior Art

- PAI skill system — local skill structure with SKILL.md, workflows, components
- PAI three-layer symlink architecture — how skills are linked into the active environment
- pai-collab [skill-enforcer](https://github.com/mellanon/pai-collab) — validates skill structure
- pai-collab SpecFlow — skill specification and development lifecycle
