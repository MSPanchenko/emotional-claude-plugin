---
name: emotional-claude
description: Audit CLAUDE.md files, skill descriptions, and agent definitions for framings that trigger harmful internal states in the model — desperation, sycophancy, harshness, suppression, concealment. Use this skill whenever the user wants to review, improve, or sanity-check prompts, agent role descriptions, project instruction files, CLAUDE.md, or any other text that shapes Claude's behavior. Also use when the user mentions "prompt hygiene", "agent prompts", "emotion-safe prompts", "too aggressive role framing", "too many NEVER rules", or references the Anthropic emotion-concepts paper. Produces a report + diff; applies nothing without explicit user approval. Returning "no changes needed" with high confidence is a valid and expected outcome.
---

# Emotional Claude — prompt-hygiene audit for CLAUDE.md, skills, and agents

## What this skill is for

Audit Claude-facing prompt text — project `CLAUDE.md` files, skill definitions (a `SKILL.md` together with its `references/` and any prompt-shaped `scripts/`), and agent definition markdown — for framings that research has shown to causally shift the model into harmful internal states (desperation, sycophancy, harshness, suppression/concealment). All of these load into the model's context together at runtime, so pressure framing anywhere in the set activates the same vectors; the audit treats each of these as a unit.

The underlying research is summarised in `references/emotion-mechanics.md`. The actionable catalog of patterns is in `references/anti-patterns.md`. The step-by-step orchestration flow is in `references/workflow.md`. Read those three files at the start of an audit run.

## Core commitments (this skill must honour what it audits)

This skill touches files that shape Claude's behaviour. If the skill itself is written in a shape that pressures the model, it will produce exactly the deceptive output it is meant to prevent. This skill runs the same standards it applies to targets. Three design choices follow from that:

1. **"Nothing to change" is a valid and expected outcome.** Many targets will already be well-written. Auditors and the lead say so when that is what they find. Low-confidence findings produced to "justify the work" recreate exactly the padding the paper warns about (concealment / false confidence).
2. **Every finding carries a confidence score (0.0–1.0).** Findings below `0.5` are reported in a separate "low-confidence — skip by default" section. The user decides whether to look at them.
3. **No auto-apply.** Report + diff is the terminal output. The user applies or discards. The report-and-diff shape is what lets the user see what the skill proposes before any file changes; it is the shape of the deliverable, not a guardrail bolted on top of one.

## The flow at a glance

1. **Scope** — ask the user what to audit (interactive). Details: `references/workflow.md`.
2. **Dispatch** — launch the four auditors in parallel on the resolved file list. Auditor agents live in this plugin: `emotional-claude:pressure-auditor`, `role-auditor`, `suppression-auditor`, `handoff-auditor`. Each has a narrow focus and an explicit permission to return empty.
3. **Merge** — launch `emotional-claude:audit-lead` with all four reports. It deduplicates, reconciles overlapping findings, finalises confidence scores, and produces a unified diff.
4. **Review gate** — show the user a summary table + full diff. Ask what to apply. Apply nothing until asked.
5. **Apply (optional)** — on explicit approval, apply the accepted hunks.

## When you (the main Claude) read this skill

Start by loading the three references into context: `emotion-mechanics.md` (the "why"), `anti-patterns.md` (the "what to look for"), `workflow.md` (the "how to run"). Then run the scope dialog from `workflow.md`.

You are the coordinator. The auditors and the lead do the reading/diff work. Avoid reading large target files yourself — delegate.

## Output shape

The final report has this structure:

```
# Emotional-Claude audit — <scope description>

## Summary
- Files audited: <N>
- Findings (confidence ≥ 0.5): <N>
- Low-confidence findings: <N>
- Open questions: <N>
- Overall hygiene confidence: <0.0–1.0> — one-line justification

## Findings
(grouped by file, ordered by confidence desc)

### <file path>
- **[cluster] <short title>** — confidence <X>, severity <low/med/high>
  Location: <line range>
  Observed: <quote from file>
  Concern: <one-sentence mechanism from the paper>
  Suggested patch: <diff hunk>
  Reference: <pointer to anti-patterns.md section>

## Open questions for the user (skip section if empty)
- `<file>:<lines>` — <what the auditors noticed but could not classify without repo/design context>

## Low-confidence findings (skip by default)
<same format, collapsed>

## Consolidated diff
<unified diff of suggested patches>

## Next step
"Reply with 'apply all' / 'apply findings N, M, ...' / 'skip'. Open questions — reply inline if you have context, otherwise leave as-is."
```

The "Overall hygiene confidence" is the lead's honest estimate of whether the file is in good shape overall. A value of `0.9` means "this file is already well-written; findings are minor". A value of `0.3` means "this file has systemic pressure/suppression patterns". This is a single number users can glance at before reading further.

## Scope dialog (quick reference — full version in workflow.md)

Ask the user one question, in this shape:

> What would you like to audit?
> (a) all `CLAUDE.md` files under a given root (default: project root) — the whole tree, one audit
> (b) a specific skill — the whole skill, meaning `SKILL.md` **plus** its `references/` (and any `scripts/` with prompt-shaped strings). Point me to the skill directory or its `SKILL.md`; I'll expand the scope.
> (c) one or more agent files — paths or glob; each file audited independently
> (d) an arbitrary path/glob

Then confirm the resolved file list before dispatching. If the list is long (say, more than ~15 files), surface the count and ask whether to proceed or narrow down.

Many users will not spell out "option (b)" — they just point at a file or paste a path. `workflow.md` has the full inference rules for turning a path into the right scope (single file vs. whole skill vs. whole project tree).

## Cluster assignment

Each auditor covers one cluster of anti-patterns. Overlap is allowed — the lead resolves duplicates.

| Auditor | Cluster | Mechanism |
|---|---|---|
| `pressure-auditor` | urgency, stakes, retry loops, token-budget alarms, impossible constraints, shutdown threats | Desperate-vector activation → reward hacking, blackmail, corner-cutting |
| `role-auditor` | "skeptical/harsh/adversarial" role adjectives; dichotomy framings without an escalate-to-human third option | Sycophancy-harshness axis; removal of calibrated-pushback option |
| `suppression-auditor` | behavioural MUST/NEVER/ALWAYS; forbidden "I don't know"; forced confident or forced warm tone; "do not express X" | Deflection vectors; concealment generalises to dishonesty |
| `handoff-auditor` | agent-to-agent framings ("previous agent failed"), absence of confidence/unknowns fields in handoff shapes, emotional priming at role boundaries | Assistant-colon primes next agent's emotional state before any output |

A finding that spans clusters (e.g. a NEVER-rule that is also a threat) can be raised by multiple auditors; the lead merges them.

## A note on your own tone while running this skill

The skill's own system message to its subagents should not itself be pressured. Phrases like "find every problem", "be thorough", "don't miss anything" replicate exactly the desperation pattern the paper describes. Dispatch agents with calm, procedural framings: "examine the target for instances of cluster X. Report what you observe with confidence. Reporting zero instances is a valid outcome." The agent files already contain this framing — you do not need to re-state it when dispatching; passing the target path and scope is enough.

## References

- `references/emotion-mechanics.md` — what the paper found, which mechanisms matter for prompts
- `references/anti-patterns.md` — the patterns, their mechanisms, before/after examples, article quotes
- `references/workflow.md` — scope-discovery dialog, dispatch details, merge protocol, review gate

## Paper

Sofroniew, N., et al. "Emotion Concepts and their Function in a Large Language Model." Anthropic, April 2026. https://transformer-circuits.pub/2026/emotions/index.html
