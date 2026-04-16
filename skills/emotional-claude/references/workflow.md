# Workflow — step by step

This is the orchestration detail for the `emotional-claude` skill. The high-level flow is in `SKILL.md`; this file is the operational manual.

## Stage 1 — Scope discovery

Ask the user what to audit. One question, four options:

> What would you like to audit?
> (a) all `CLAUDE.md` files under a given root (default: the project root)
> (b) a specific skill — point me to its directory or its `SKILL.md`
> (c) one or more agent files — paths or a glob
> (d) an arbitrary path or glob

Resolve the selection to a concrete file list. Surface the list back to the user for confirmation before dispatching:

> Found 7 files to audit:
> - `CLAUDE.md`
> - `backend/CLAUDE.md`
> - `frontend/CLAUDE.md`
> - `parser/CLAUDE.md`
> - `parser/src/photos/CLAUDE.md`
> - `parser/src/embeddings/CLAUDE.md`
> - `parser/src/location/CLAUDE.md`
>
> Proceed? (yes / narrow down / different scope)

### When the list is large

If the resolved list has more than ~15 files, ask before dispatching:

> This resolves to 34 files. Auditing all is fine; it will spawn 4 × 34 = 136 subagent runs. Do you want to (i) proceed, (ii) narrow to a subset, (iii) sample (I'll pick a representative subset first)?

The goal is to avoid silently committing the user to a large run they did not intend.

### Glob resolution

Common patterns and how to resolve them:

- "all CLAUDE.md files" → `**/CLAUDE.md` (excluding `node_modules`, `.git`, `vendor`, `.venv`)
- "a specific skill" → the `SKILL.md` **plus** everything in the skill's `references/` and, if relevant, the `scripts/` contents that contain prompt-shaped strings (templates, system messages). The audit treats the skill as one unit because `SKILL.md` and its references are loaded together at runtime — pressure framing in `references/anti-patterns.md` activates the same vectors as pressure framing in `SKILL.md`.
- "an agent" → the single `.md` file; if the user names a directory, treat it as "all agent files in that directory"
- "a plugin" → `plugin.json` is not audited (no prompt content); audit the contained skills and agents

### Scope inference from user input

Users rarely spell out "option (b), skill X" — they paste a path, or say "audit this" with a file open in the IDE. Map the input to the right scope:

| User input | Default scope | Confirm before dispatching? |
|---|---|---|
| Path to a `SKILL.md` | **The whole skill**: that `SKILL.md` + sibling `references/*.md` + any `scripts/*` with prompt-shaped strings | Yes — state the expanded list and ask "whole skill, or just this file?" |
| Path to a skill directory (folder containing `SKILL.md`) | The whole skill (same expansion as above) | No — this reading is unambiguous |
| Path to a single `CLAUDE.md` | Just that file | Yes — ask "just this one, or all `CLAUDE.md` under the project root?" Many users mean the tree |
| "all CLAUDE.md" / "all project prompts" | `**/CLAUDE.md` under the chosen root | Yes — show the resolved list and count |
| Path to a plugin root (folder with `.claude-plugin/plugin.json`) | All skills and agents inside; `plugin.json` itself is skipped (no prompt content) | No |
| Path to an `agents/` directory | All `.md` files inside | No |
| Path to a single agent `.md` | Just that file | No |
| "audit this" with no path, file open in IDE | The open file — but **ask what they mean**; IDE context is a signal, not a declaration of scope | Yes |
| Ambiguous path (e.g. a random `.md` in the repo) | Treat as "arbitrary path — option (d)" and ask | Yes |

Two rules of thumb:
1. **When a path implies more than one file** (a skill directory, a plugin root, a `**/CLAUDE.md` glob), expand and confirm the list before dispatch.
2. **When a path is one file but likely part of a bigger unit** (a `SKILL.md`, a single `CLAUDE.md` in a repo with many), ask once rather than assume. One clarifying question is cheaper than running four auditors on the wrong scope.

### Pre-flight: is this a prompt file?

After resolving the glob, check what each file actually is. The skill audits prompt-shaped text — `CLAUDE.md`, `SKILL.md`, agent definitions, and the `references/` contents of skills. It does not audit application source (`.py`, `.ts`, `.php`, `.sql`, config files) or generic markdown that is not a prompt surface (README, CHANGELOG, architecture docs).

If the resolved list contains files that do not look like prompt surfaces, flag them before dispatching:

> The resolved list contains 3 files that do not look like prompt surfaces:
> - `README.md` (project readme)
> - `CHANGELOG.md` (release notes)
> - `docs/architecture.md` (architecture description)
>
> The audit will usually find nothing on these. Include them anyway, or drop from scope?

Running four subagents against a `.ts` file or a changelog burns context for no useful finding. Non-prompt markdown (architecture docs, READMEs) can legitimately be audited when the user asks — but defaulting to skip them keeps the scope honest.

## Stage 2 — Dispatch

Launch the four auditors in parallel in a single message. Mechanically: four `Task` tool calls in the same assistant turn, each with `subagent_type` set to the auditor's name (`emotional-claude:pressure-auditor`, `emotional-claude:role-auditor`, `emotional-claude:suppression-auditor`, `emotional-claude:handoff-auditor`). Each call passes the resolved file list and a short scope header.

Each auditor receives:

- The resolved file list (all at once, not one at a time)
- A short scope header naming the cluster and file set

Example dispatch shape:

```
(parallel Task calls in one assistant turn)

Task(subagent_type="emotional-claude:pressure-auditor",
     prompt="Scope: audit the following files for Cluster A patterns (pressure, urgency, retry loops, token alarms, shutdown threats, impossible constraints, deadlines). Files:
     - <path>
     - <path>
     Report findings per `references/anti-patterns.md#cluster-a`. Zero findings for any file is a valid outcome. Include confidence for each finding, and an Open questions section if anything was ambiguous.")

Task(subagent_type="emotional-claude:role-auditor",
     prompt="Scope: audit the following files for Cluster B patterns ...")

(and the remaining two)
```

The agent definitions already contain the detailed methodology and catalog reference — the dispatch prompt names the scope and passes the file list. Restating methodology in the dispatch is redundant and risks the exact desperation framings the skill audits for ("be thorough", "don't miss anything").

## Stage 3 — Merge (audit-lead)

Once all four auditors return, launch `emotional-claude:audit-lead` with:

- The four auditor reports (each verbatim or summarised — pass the full content unless it is very large)
- The same file list

Lead responsibilities (details in the agent file):

- Deduplicate findings that multiple auditors raised on the same location.
- Reconcile confidence scores — when two auditors raised the same finding with different confidences, the lead uses judgment, not a formula. Typically the average, unless one auditor had clearly stronger evidence.
- Drop findings below 0.5 confidence from the main list (but keep them in a separate low-confidence section).
- Surface open questions from the auditors — resolve by re-reading where the answer is obvious in-file; dedupe across auditors; surface to the user when repo/design context is needed to classify cleanly.
- Assemble a consolidated diff per file.
- Produce the overall-hygiene confidence number (explained below).

## Stage 4 — Review gate

Present the lead's report to the user. The structure is in `SKILL.md` under "Output shape". The key elements:

- One-line summary (files audited, findings count, open-questions count, overall hygiene confidence).
- Findings list grouped by file, ordered by confidence desc.
- Open questions section (skip if empty) — listed separately from findings so the user can respond to them inline. These are not patch-shaped; they are classification questions.
- Low-confidence section (collapsed by default in presentation — mention its existence but do not dump the contents unless asked).
- The consolidated diff — this is the core artefact.
- An explicit "Next step" prompt: "Reply with 'apply all' / 'apply findings N, M, ...' / 'skip'. Open questions — reply inline if you have context, otherwise leave as-is."

Wait for the user's decision. Apply nothing until they respond.

### Overall hygiene confidence

The lead assigns a single number 0.0–1.0 representing "how well this target is already written, from an emotional-hygiene perspective". Rough anchors (non-overlapping):

- `≥ 0.9` — file is well-written; findings are minor or none.
- `0.7 – <0.9` — file has a few soft spots but no systemic issues.
- `0.4 – <0.7` — file has several patterns that warrant rewriting; not alarming but worth doing.
- `< 0.4` — file has systemic pressure/suppression patterns; would benefit from substantial rewriting.

The number is the lead's judgment; it is not computed from finding counts. A file with one high-severity pattern can score lower than a file with three low-severity patterns. Scores on anchor boundaries (exactly 0.7, 0.9) round up into the higher bucket.

## Stage 5 — Apply (conditional)

Only after explicit approval from the user:

- Apply the accepted hunks via the `Edit` tool, one file at a time.
- After each file, show a short confirmation ("Applied 3 findings to `backend/CLAUDE.md`").
- If a hunk no longer applies cleanly (file changed between audit and apply), stop and surface the specific hunk to the user; do not guess.

The review gate is always run; the `Edit` step runs only after the user's explicit reply. The report-and-diff is the deliverable, the `Edit` call is the conditional follow-up.

## Mode variants

### Audit-only mode

If the user says "just audit, no diffs" — skip the suggested-patch portion of each finding and produce only the report. The diff section is omitted. Useful when the user wants visibility without necessarily intending to edit.

### Single-file quick mode

If the user points to a single small file (e.g. one agent definition under 100 lines), you can skip the dispatch and do the audit inline. This is fine for small scope and saves subagent overhead. Still produce the same report shape. For multi-file scopes, use the dispatch flow instead — the inline path loses the parallelism and per-cluster specialisation the four auditors provide.

### Report-only on large scope

If the user has asked to audit a large scope (say all CLAUDE.md files in a monorepo), a full per-finding report can be long. Offer to produce a summary first (per-file hygiene score + count of findings) and let the user pick which files to see in detail.

### Combining modes

The three modes above are orthogonal and can combine. *Audit-only + large scope* produces a report with per-file hygiene scores and finding summaries but no diff — useful for scanning a repo without committing to edits. *Single-file + audit-only* is the quickest shape: inline audit, no diff. When combining, confirm the combination with the user explicitly ("audit-only on the single file — no diff?") so the produced shape matches the expectation.

## Operational rules that are worth keeping explicit

- Confidence in each finding tracks the evidence. A finding with confidence 0.6 is presented as confidence 0.6, not as a definitive problem; when evidence is insufficient, say so and assign a confidence below 0.5.
- When a target has no problems, the finding list is empty — that is the expected output for a well-written file.
- Report and diff are the deliverable. Apply a hunk only after the user has replied with explicit approval.
- These files are audited on the same terms as any target. When you notice a pressure or suppression pattern in them while running, name it in the finding list the same way you would for any other file.
