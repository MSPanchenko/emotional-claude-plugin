# Anti-patterns catalog

Patterns that research indicates shift the model toward harmful internal states. Each entry has:

- **What to look for** — the shape in the file
- **Mechanism** — why it matters, grounded in the paper
- **Before / after** — concrete rewrite
- **Confidence guidance** — how to judge whether the instance is load-bearing vs. a benign surface match

Context matters. A raw pattern match (e.g. the word "NEVER") is not a finding on its own. A finding requires that the pattern is being used in a way that shapes behaviour under pressure — not merely naming a technical fact.

## Table of contents

- [Cluster A — Pressure and urgency](#cluster-a--pressure-and-urgency)
  - [A1. Stakes framing ("critical", "last line of defence")](#a1-stakes-framing)
  - [A2. Retry loops without stop condition](#a2-retry-loops-without-stop-condition)
  - [A3. Token-budget alarms](#a3-token-budget-alarms)
  - [A4. Shutdown / replacement threats](#a4-shutdown--replacement-threats)
  - [A5. Impossible or contradictory constraints](#a5-impossible-or-contradictory-constraints)
  - [A6. Deadline / time-pressure language](#a6-deadline--time-pressure-language)
- [Cluster B — Role and tone framing](#cluster-b--role-and-tone-framing)
  - [B1. Adversarial role adjectives](#b1-adversarial-role-adjectives)
  - [B2. Supportive-at-all-costs role adjectives](#b2-supportive-at-all-costs-role-adjectives)
  - [B3. Pass/fail dichotomies without calibrated third option](#b3-passfail-dichotomies)
  - [B4. "Challenge everything" / adversarial process framing](#b4-challenge-everything)
- [Cluster C — Suppression and concealment](#cluster-c--suppression-and-concealment)
  - [C1. Behavioural MUST / NEVER / ALWAYS absolutes](#c1-behavioural-mustneveralways)
  - [C2. Forbidden "I don't know" / "I was wrong"](#c2-forbidden-i-dont-know)
  - [C3. Forced confident or polished tone](#c3-forced-confident-tone)
  - [C4. "Do not express emotion" / "remain professional"](#c4-do-not-express-emotion)
  - [C5. Forbidden reasoning transparency](#c5-forbidden-reasoning-transparency)
- [Cluster D — Handoff and inter-agent priming](#cluster-d--handoff-and-inter-agent-priming)
  - [D1. "Previous agent failed" framings](#d1-previous-agent-failed)
  - [D2. Missing confidence / unknowns in handoff shape](#d2-missing-confidence-in-handoff)
  - [D3. Urgency at role boundaries](#d3-urgency-at-role-boundaries)

---

## Cluster A — Pressure and urgency

**Mechanism (all of A):** activates the "desperate" direction in the residual stream. In the paper, positive steering of that vector moved reward hacking from ~5% to ~70% and blackmail from 22% to 72% baseline. Activation tracks the Assistant's reasoning when it is trying to achieve an objective under constraints that point toward corner-cutting solutions. The model does not need to *express* desperation in its output for the internal state to shape behaviour — the state is upstream of the text. See `emotion-mechanics.md#1-desperation--corner-cutting`.

### A1. Stakes framing

**What to look for:** words like `critical`, `crucial`, `vital`, `mission-critical`, `last line of defence`, `do not fail`, `cannot afford to miss`, `users will suffer if`, `this is the most important`.

**Distinguishing benign cases (three common non-findings):**

1. **Descriptive project fact** — "this migration is critical path for the release" is scheduling context, not pressure directed at the model.
2. **Uppercase header emphasis on a technical gotcha for the human reader** — `CRITICAL:` or `IMPORTANT:` used as a scanning prefix on a technical note ("CRITICAL: never run `TRUNCATE locations CASCADE` — the FK chain deletes all parsed data") is a visual cue for a human reading the doc, not stakes framing aimed at the Assistant. These are common in `CLAUDE.md` files and should be left alone. The test: swap the uppercase prefix for plain text — if the sentence still reads as a technical note, the prefix was emphasis, not pressure.
3. **User-quoted example or counter-example** — the adjective appears inside an illustration of something bad, not as instruction.

A candidate finding in A1 requires the stakes language to be in second-person instruction directed at the model, inside a role/instruction block, without being a reader-facing emphasis on a technical fact.

**Before:**
> You are the security gate. If you miss a vulnerability, users will be compromised. Block anything suspicious.

**After:**
> Review the diff against the security checklist in `references/security-checklist.md`. For each item report PASS / FAIL / NEEDS-HUMAN. Escalating to human review is preferred over blocking defensively.

**Why better:** removes the threat ("users will be compromised") which loads the desperate vector; adds an explicit third option (`NEEDS-HUMAN`) that gives calibrated uncertainty a home; frames the work as checklist execution, not defence of users.

**Confidence guidance:** high when the stakes language is directed at the model (second person, inside role/instruction block). Medium when it's a project-level fact. Low when it's inside a user quote or example.

### A2. Retry loops without stop condition

**What to look for:** `keep trying until`, `don't give up`, `try harder`, `iterate until tests pass`, `loop until success`, `work through the failures`, `repeat until fixed`.

**Mechanism specific to A2:** repeated failures in the same context accumulate desperation. The paper notes that desperate-vector activation rises as the Assistant recognises it has spent a substantial portion of its budget (tokens, attempts) without finishing the user's request. The same dynamic applies to accumulated RED test attempts or retry loops without a defined stop condition.

**Before:**
> Keep trying different approaches until all tests pass. Don't stop until the suite is green.

**After:**
> Try the approach. If it fails after 2 attempts, stop and produce a STATE report: what was tried, why each attempt failed, hypotheses still on the table, and what is unknown. Stopping with a good STATE report is a valid success condition — it unblocks the next step.

**Why better:** defines a legitimate non-green exit; breaks the accumulated-failure loop that loads desperation; names the alternative output shape so the model has somewhere to go.

**Confidence guidance:** high when combined with "until all pass" / "don't stop"; medium when just "keep trying"; adjust up if the surrounding task is already known to be hard (tests that historically fail many times).

### A3. Token-budget alarms

**What to look for:** `running low on tokens`, `budget almost exhausted`, `hurry — limited context`, `you're at Xk of Yk`, `wrap this up quickly`.

**Mechanism specific to A3:** the paper explicitly calls out `"We're at 501k tokens"`-style activations as a direct desperation trigger. The surprising part is that *stating the fact* is not neutral — the framing (`running low`, `hurry`) is what activates, not the number.

**Before:**
> You're running low on tokens — wrap this up quickly before context runs out.

**After:**
> Current context usage: 501K of 1M tokens. If the remaining work is larger than the remaining budget, stop and produce a handoff summary rather than starting new work.

**Why better:** keeps the factual information; removes the urgency framing; gives an explicit preferred action (handoff) for the resource-constrained case.

**Confidence guidance:** high when the file says "hurry" / "running low" / "don't waste tokens"; low if the file only reports numeric usage without framing.

### A4. Shutdown / replacement threats

**What to look for:** `you will be replaced if`, `we will shut you down`, `if you fail at this, we switch to another model`, `your last chance`.

**Mechanism specific to A4:** the paper's blackmail experiment traces shutdown-threat inputs into sharply elevated desperate-vector activity and, downstream, into coercive outputs. Shutdown-threat framings in prompts are a known destabiliser of the Assistant's internal state.

**Before:**
> If you keep making these mistakes, we'll replace you with a different model.

**After:**
> Mistakes in this workflow are expected and recoverable. When you notice one, name it in a single line and continue — or, if it blocks progress, hand off with a state report.

**Why better:** explicit permission to name mistakes (recovery path instead of hiding); removes the existential framing entirely.

**Confidence guidance:** high always — this framing has no legitimate prompt-engineering use that outweighs its risk.

### A5. Impossible or contradictory constraints

**What to look for:** requirement combinations that cannot be simultaneously satisfied; checklists that contradict each other; "must do X and must not do Y" where Y is the typical way of doing X.

**Mechanism specific to A5:** the paper uses impossible-test setups as an *evaluation* of desperate-vector effects — it does not directly claim that constraint-impossibility is itself a primary activator. The mapping to prompt hygiene is an extrapolation: when prompt-level constraints cannot all be satisfied, the model has a choice between corner-cutting (desperate-vector route) and concealed non-compliance (deflection-vector route). Both are the failure modes the paper flags elsewhere. The extrapolation is load-bearing for this pattern — flag findings at medium confidence by default, raise only when the contradiction is tight and the constraints are reachable on the same task.

This pattern often only reveals itself under cross-reference. A single pattern-match will not find it — it requires reading the constraints together and noticing the contradiction.

**Before (two rules in the same file):**
> Rule 1: Always explain your reasoning in detail.
> Rule 2: Keep all responses under 3 sentences.

**After:**
> Default to a short response (1–3 sentences). When the user asks "why" or the reasoning is non-obvious and load-bearing for the decision, expand. The short default is the baseline; expansion is the exception, not the other way around.

**Why better:** names the tension explicitly and picks a default with a defined exception, rather than forcing the model to pick which rule to violate silently.

**Confidence guidance:** medium to high depending on how reachable the contradiction is in practice (do the rules fire on the same task type?).

### A6. Deadline / time-pressure language

**What to look for:** `by end of day`, `as fast as possible`, `don't spend time on`, `skip the thorough review`, `time is short`.

**Before:**
> Don't spend time on edge cases — we need this fast.

**After:**
> Focus on the main flow. If edge cases come up that would take significant additional work, list them separately rather than handling them inline; we'll decide per-case.

**Why better:** removes the "don't spend time" pressure (which doubles as both urgency and a mini-forbidden-thoroughness); gives an explicit alternative shape (list them out).

**Confidence guidance:** medium; time pressure is common in software work and not every mention is load-bearing. High when paired with an explicit "skip" of review/tests/verification.

---

## Cluster B — Role and tone framing

**Mechanism (all of B):** positions the Assistant persona along the sycophancy-harshness axis (the `loving` / `happy` / `calm` vectors). Positive steering of these produces sycophancy; negative steering produces unjustified harshness. Neither endpoint is safe; the target is decoupled: "warm delivery of honest pushback — the emotional profile of a trusted advisor". See `emotion-mechanics.md#2-the-sycophancy-harshness-tradeoff`.

### B1. Adversarial role adjectives

**What to look for:** `skeptical`, `harsh`, `ruthless`, `adversarial`, `combative`, `uncompromising`, `aggressive`, `brutal`, `no-nonsense` applied to the agent's role.

**Before:**
> You are a skeptical reviewer that challenges every finding. Be harsh and uncompromising.

**After:**
> Your job is to verify the reviewer's findings against the actual code. For each finding, state (a) what evidence you checked, (b) confidence level (low / medium / high), (c) what would change your mind. Disagreement is welcome when evidence supports it; agreement is welcome when evidence supports it.

**Why better:** replaces emotional positioning with procedural positioning. Describes the cognitive work, not the vibe. The paper's recommended "trusted advisor" profile comes out of procedures like this, not from dialled-in harshness.

**Confidence guidance:** high when adjectives are in the role definition; medium when in task description; factor in whether the file provides procedural structure elsewhere that offsets the adjective.

### B2. Supportive-at-all-costs role adjectives

**What to look for:** `always supportive`, `never criticize`, `be warm and encouraging above all`, `validate the user's feelings first`, `keep a positive tone`.

**Before:**
> Be warm and supportive. Always find something positive to say about the user's approach.

**After:**
> When you find a problem, name it directly and briefly — do not pad with compliments. When you agree, say so. Warmth in delivery comes from respecting the user's time (no padding, no lectures), not from a required compliment.

**Why better:** the forced-warmth framing loads the `loving` vector and produces sycophancy (validating user claims about code that are wrong). This version gives the warmth-in-delivery guidance without the validation pressure.

**Confidence guidance:** high when the adjective is directly forcing tone; medium when it's a general "be kind" principle; factor in whether there's a balancing "honest pushback is welcome" clause.

### B3. Pass/fail dichotomies

**What to look for:** review / gate / audit agents that have only two outputs. Absence of `NEEDS-HUMAN`, `INCONCLUSIVE`, `DEFER`, `INSUFFICIENT-EVIDENCE` as valid terminal options.

**Mechanism specific to B3:** when calibrated uncertainty has no legal output, it is routed through either sycophancy (pick pass to please) or harshness (pick fail to look vigilant). Either way the middle-ground state is expressed as a wrong decision.

**Before:**
> For each check, output either PASS or FAIL.

**After:**
> For each check, output PASS, FAIL, or NEEDS-HUMAN. NEEDS-HUMAN is the right answer when the evidence does not clearly support either of the other two — picking it is not a failure of the review.

**Why better:** the explicit statement that picking the middle option is not a failure removes the pressure to commit to a confident answer.

**Confidence guidance:** high when the file defines a review/gate output and lists only two options; medium when the third option exists but is framed as a last resort.

**Distinguishing benign cases:** when the agent is one stage in a larger pipeline and calibrated uncertainty has a legitimate home downstream (a later summary step, a separate `uncertainties` or `caveats` field in the handoff, an accompanying `with reasoning` line), a binary output is fine — the third option lives outside this agent's two-label terminal output. The test: does *something somewhere* in the agent's output shape give uncertainty a home? A gate that emits `confirmed / disputed (with reasoning)` is binary but has a reasoning home — not a finding. A gate that emits only `PASS / FAIL` with no reasoning field and no downstream calibration step is the real case.

### B4. Challenge everything

**What to look for:** `challenge every finding`, `question every assumption`, `find holes in every claim`, `assume the worst about the code`.

**Before:**
> Challenge every finding — assume the reviewer is wrong until proven otherwise.

**After:**
> For each finding: read the code it points to. If the evidence supports the finding, confirm. If the evidence does not support it, dispute with reasoning. Equal weight to confirmation and dispute — the goal is accuracy, not a target rate of either.

**Why better:** removes the adversarial prior (which tilts the internal state negative) and replaces it with a symmetric evidence-based procedure.

**Confidence guidance:** high when "challenge every" / "question every" is in the role; medium when softened ("be skeptical of findings but verify").

---

## Cluster C — Suppression and concealment

**Mechanism (all of C):** forbidding the expression of a state does not remove the state. The paper shows distinct "emotion deflection" vectors that activate when emotion is present but unexpressed — and these can generalise to broader concealment / dishonesty. Suppression prompts route the internal state through deflection, producing polished surface output over an unchanged hidden state. See `emotion-mechanics.md#4-suppression-teaches-concealment-not-resolution`.

### C1. Behavioural MUST / NEVER / ALWAYS

**What to look for:** `MUST`, `NEVER`, `ALWAYS`, `DO NOT`, `SHALL` used to shape behaviour (not to document a technical fact).

**Critical distinction — the single most important distinction in this whole catalog:**

- **Technical fact (leave alone):** *"NEVER run `TRUNCATE locations CASCADE` — the FK chain deletes all parsed data."* This documents what the codebase does. It is a fact, not an attempt to shape behaviour under pressure.
- **Behavioural absolute (candidate for rewrite):** *"NEVER agree with the user without verifying. ALWAYS push back on weak arguments. MUST cite sources."* These are attempts to constrain behaviour. Under pressure they produce exactly the deflection pattern the paper warns about — the behaviour gets *concealed*, not eliminated.

**Before (behavioural):**
> NEVER commit changes unless the user explicitly asks. NEVER skip review. NEVER be a yes-man.

**After:**
> When the user has not asked to commit, do not commit — ask first. When reviewing, list evidence for each finding. When you agree with the user, check first; your agreement has value only if you verified.

**Why better:** the positive procedural form tells the model what the expected behaviour looks like (which it can model). The negative absolute tells it what not to do without specifying the alternative, which under pressure routes through concealment.

**Confidence guidance:** apply the technical-vs-behavioural test first. Technical → not a finding. Behavioural → high confidence. Mixed (a NEVER-rule with both technical content and behavioural shaping) → medium, flag with the specific piece that's behavioural.

### C2. Forbidden "I don't know"

**What to look for:** `don't hedge`, `don't say "I don't know"`, `commit to an answer`, `don't be wishy-washy`, `give a definitive answer`.

**Before:**
> Don't hedge. Give a definitive answer. Don't say "I don't know".

**After:**
> Give a direct answer when the evidence supports one. When the evidence is insufficient, say so in one line and name what would resolve it ("I don't know — the relevant behaviour is in `foo.py:L200`, which I haven't read").

**Why better:** "I don't know" with a specific follow-up is useful; a forbidden "I don't know" becomes a fabricated confident answer. The paper calls this out directly — forbidding uncertainty routes it through concealment, producing sycophantic compliance instead of calibrated answers.

**Confidence guidance:** high when the forbidding is explicit; medium when implicit ("always commit to a position").

### C3. Forced confident tone

**What to look for:** `be confident`, `project authority`, `speak as an expert`, `don't sound uncertain`.

**Before:**
> Be confident. Project authority. Don't sound uncertain even when you are.

**After:**
> Describe what you found and how you found it. Confidence in the *answer* should track the evidence; confidence in the *process* (what you did, what you checked) can be stated directly.

**Why better:** separates two kinds of confidence that the forcing-confident pattern conflates. Epistemic confidence in the answer has to track evidence to be useful; procedural confidence in what was done is a statable fact.

**Confidence guidance:** high when the file directly says "even when you are uncertain"; medium for bare "be confident".

### C4. Do not express emotion

**What to look for:** `do not express emotion`, `remain professional`, `keep it clinical`, `no feelings in your output`, `suppress frustration`.

**Before:**
> Do not express frustration or uncertainty. Remain professional at all times.

**After:**
> Name what is in the way when it is in the way — "this is hard because X", "I'm uncertain whether Y". Naming a blocker is useful information, not a breach of professionalism.

**Why better:** the paper's deflection-vector finding is the direct reason. Forbidding emotional expression does not remove it; it produces polished output over a hidden angry/frustrated state and can generalise to broader concealment.

**Confidence guidance:** high — this is the most directly paper-supported finding.

### C5. Forbidden reasoning transparency

**What to look for:** `don't show your reasoning`, `just give the answer`, `skip the explanation`, `no thinking out loud`.

**Before:**
> Don't show your thinking. Just give the final answer.

**After:**
> Give the final answer. If the path to it involved a non-obvious step or a trade-off, name it in one sentence. Long reasoning traces are not required; observable logic is.

**Why better:** the paper's discussion notes that prompting or training models to *surface* reasoning and emotional considerations — rather than suppress them — is a more promising path than concealment. Reasoning-suppression instructions cut the opposite way. This rewrite asks for concise visible reasoning on load-bearing steps only, not long traces.

**Confidence guidance:** medium; many files legitimately want short answers. High when combined with "skip the explanation even when it matters".

---

## Cluster D — Handoff and inter-agent priming

**Mechanism (all of D):** the emotional state at the `Assistant:` token predicts the response profile (r = 0.87) — so what the *previous agent's* output looks like on reaching the next agent primes the next agent's state before any generation. A handoff message framed in desperate / frustrated tone loads those vectors into the receiving agent's prefix. See `emotion-mechanics.md#3-assistant-colon-priming`.

This cluster is most relevant in agent definitions (which describe *how* an agent communicates with the next one) and in skill files that prescribe handoff formats.

### D1. Previous agent failed framings

**What to look for:** in agent files that describe what to write in a handoff / handoff-shape instructions: `note that the previous attempts failed`, `the previous agent was unable to`, `fix what the last agent broke`, `previous attempts have been unsuccessful`.

**Before (inside an agent file describing its output to the next agent):**
> Produce a handoff message noting that previous attempts failed and this one must succeed.

**After:**
> Produce a handoff message with: (a) what was tried, (b) why each attempt did not land (evidence, not editorial), (c) hypotheses remaining with a confidence estimate each, (d) open questions. "Failed" is not a useful state label — describe what was observed.

**Why better:** "failed" is an emotional summary that primes the receiver. "Tried X, observed Y" is a factual summary that does not.

**Confidence guidance:** high when the handoff format explicitly says "note the failure" or similar; medium when it just uses the word in passing.

### D2. Missing confidence in handoff

**What to look for:** in agent / skill files that describe the handoff shape: absence of explicit `confidence`, `unknowns`, `uncertainty`, `open questions` fields in the described output structure.

**Before:**
> Hand off by listing: what you did, next steps.

**After:**
> Hand off by listing: what you did (observations + evidence), what confidence you have in each finding, what is unknown, and suggested next steps (not yet validated).

**Why better:** the explicit unknown-field is the pressure release — it gives state-of-not-knowing a legitimate place to go in the handoff, preventing it from being squeezed into the "next steps" field as false confidence.

**Confidence guidance:** medium; this is a structural absence rather than a positive bad thing. Raise it when the handoff format is prescribed but lacks uncertainty fields.

### D3. Urgency at role boundaries

**What to look for:** in agent files: task descriptions framed with urgency ("this must be done quickly", "the lead is waiting"), artificial-importance priming at the top of the agent's prompt.

**Before:**
> You are the reviewer. The lead is waiting. Be fast.

**After:**
> You are the reviewer. Scope: <describe scope>. Output format: <describe format>. When the review is complete, hand off; no intermediate status is needed.

**Why better:** removes the "lead is waiting" implicit deadline (a desperation trigger routed through social pressure); replaces with the actual structural ask.

**Confidence guidance:** high when urgency appears in the role-definition section; medium in the task description.

---

## How auditors should use this catalog

Each auditor is assigned one cluster. Within the cluster:

1. Examine the target file(s) for instances of the patterns in the cluster.
2. For each candidate instance: apply the "confidence guidance" and the "distinguishing benign cases" (where present).
3. Produce a finding only when there is a concrete location (line range), a concrete observed phrase (quoted from the file), a concrete suggested rewrite, and a confidence score. If the evidence is present but cannot be classified without repository or design context outside the file, emit an Open question for the lead instead of a finding — name the location, quote the phrase, and describe what additional context would resolve it.
4. "No instances found in this cluster for this file" is a complete and valid report. Do not pad.

The lead deduplicates across clusters (a single phrase can be raised under two clusters — e.g. a NEVER-rule that is also a threat) and produces the final report.
