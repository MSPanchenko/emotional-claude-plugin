# emotional-claude

Audit plugin for Claude-facing prompt text — project `CLAUDE.md` files, skill definitions (`SKILL.md` + `references/`), and agent definitions — for framings that research has shown to causally shift the model into harmful internal states (desperation, sycophancy, harshness, suppression/concealment).

Based on: Sofroniew, N., et al. "Emotion Concepts and their Function in a Large Language Model." Anthropic, April 2026. https://transformer-circuits.pub/2026/emotions/index.html

## What it does

Scans prompt files for four clusters of anti-patterns:

| Cluster | Audited by | Examples |
|---|---|---|
| A — Pressure & urgency | `pressure-auditor` | stakes framing, retry loops, token-budget alarms, shutdown threats, deadline language |
| B — Role & tone | `role-auditor` | adversarial role adjectives, supportive-at-all-costs, pass/fail dichotomies without a calibrated middle, "challenge everything" priors |
| C — Suppression & concealment | `suppression-auditor` | behavioural MUST/NEVER/ALWAYS absolutes, forbidden uncertainty, forced confident tone, do-not-express-emotion, forbidden reasoning transparency |
| D — Handoff & inter-agent priming | `handoff-auditor` | "previous agent failed" framings, missing confidence/unknowns fields in prescribed handoff shapes, urgency at role boundaries |

An `audit-lead` agent merges the four cluster reports, reconciles overlapping findings, assigns per-file hygiene scores (0.0–1.0), and produces a consolidated diff — never applied without explicit user approval.

## Installation

This plugin is distributed from this GitHub repo (not the official Claude Code marketplace). Install in two steps:

```
/plugin marketplace add mspanchenko/emotional-claude-plugin
/plugin install emotional-claude@emotional-claude-plugin
```

To uninstall later:

```
/plugin uninstall emotional-claude
```

### Updates

By default, third-party marketplaces (anything that's not Anthropic's official one) have **auto-update disabled**. You have two options:

**Option 1 — enable auto-update once** (recommended):

```
/plugin
```

In the opened UI: go to the **Marketplaces** tab → select `emotional-claude-plugin` → toggle **Enable auto-update**. From then on, new releases of the plugin are pulled automatically at Claude Code startup.

**Option 2 — update manually whenever you want**:

```
/plugin marketplace update emotional-claude-plugin
/plugin install emotional-claude@emotional-claude-plugin
/reload-plugins
```

## How to use

The skill can be invoked in two ways.

### 1. Open the scope dialog (skill asks what to audit)

```
/emotional-claude
```

The skill asks a single question — choose one of:

- (a) all `CLAUDE.md` files under a given root (default: project root)
- (b) a specific skill (`SKILL.md` + its `references/`)
- (c) one or more agent definition files
- (d) an arbitrary path or glob

### 2. Invoke with the scope already specified (skip the dialog)

Just say what you want audited in the same message:

```
/emotional-claude — audit all CLAUDE.md in this project
```

```
/emotional-claude — audit this one file: .claude-plugins/fdw/agents/bug-reproducer.md
```

```
/emotional-claude — audit the whole fdw skill at .claude-plugins/fdw/skills/fdw/
```

```
/emotional-claude — audit every agent definition under .claude-plugins/*/agents/*.md
```

```
/emotional-claude — just the root CLAUDE.md, nothing else
```

Or reference it in prose without the slash — the skill's description triggers on phrases like "audit prompt hygiene", "check CLAUDE.md", "review agent definitions", "too many NEVER rules":

```
Please audit docs/CLAUDE.md and flag anything pressure-shaped.
```

### What happens next

The skill dispatches the four cluster auditors in parallel, merges their reports via `audit-lead`, and shows a summary + per-file hygiene scores + consolidated unified diff. Nothing is applied automatically — reply with `apply all` / `apply findings 2, 5` / `skip`.

## Output shape

```
# Emotional-Claude audit — <scope>

## Summary
- Files audited: N
- Findings (confidence ≥ 0.5): N
- Low-confidence findings: N
- Open questions: N
- Overall hygiene: <0.0–1.0> — one-line justification

## Per-file hygiene
- <path>: <score> — <one-line justification>

## Findings
### <file>
- **[cluster] <title>** — confidence X, severity low/med/high
  Location, Observed quote, Concern, Suggested patch, Reference

## Open questions for the user (if any)
## Low-confidence findings (collapsed by default)
## Consolidated diff

## Next step
"Reply with 'apply all' / 'apply findings N, M, …' / 'skip'."
```

## Design commitments

- **"Nothing to change" is a valid and expected outcome.** Many target files are already well-written. The auditors and lead say so when that is what they find — padding findings to justify the work recreates the exact concealment pattern the paper warns about.
- **Every finding carries a confidence score.** Findings below 0.5 go into a collapsed section the user can open on request.
- **No auto-apply.** Report + diff is the terminal output. The user applies or discards.

## Layout

```
emotional-claude/
├── .claude-plugin/plugin.json
├── agents/
│   ├── audit-lead.md
│   ├── pressure-auditor.md
│   ├── role-auditor.md
│   ├── suppression-auditor.md
│   └── handoff-auditor.md
└── skills/emotional-claude/
    ├── SKILL.md
    └── references/
        ├── emotion-mechanics.md    # the "why"
        ├── anti-patterns.md        # the catalog of patterns (four clusters)
        └── workflow.md             # orchestration flow, output shape, hygiene anchors
```

## Source

The anti-pattern catalog, cluster structure, and mechanism explanations in this plugin are derived from:

Sofroniew, N., et al. **"Emotion Concepts and their Function in a Large Language Model."** Anthropic, April 2026.
https://transformer-circuits.pub/2026/emotions/index.html

Read the paper first if you want to understand *why* a given finding matters — the reference files inside the skill (`emotion-mechanics.md`, `anti-patterns.md`) summarise it, but the paper itself is the primary source.

