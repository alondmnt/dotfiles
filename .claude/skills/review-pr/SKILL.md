---
name: review-pr
description: Review a GitHub pull request for defects, data integrity issues, and silent failures
disable-model-invocation: true
argument-hint: "[pr-number-or-url]"
allowed-tools: Read, Grep, Glob, Bash(gh *, python -c *), AskUserQuestion
---

## Instructions

**Step 1 — Fetch PR context:**

Check whether the user provided a PR number or URL after `/review-pr`. If yes, run these commands to fetch context (they are independent — run them in parallel):
- `gh pr view <number>`
- `gh pr diff <number> --name-only`
- `gh pr diff <number>`

Then proceed with the review below.

If no PR was specified, run `gh pr list --state open --limit 20` to list open PRs, then use AskUserQuestion to ask the user which PR to review (use PR numbers as option labels). Once selected, fetch context with the three commands above and proceed.

**Pin the target.** Record the SHA you are reviewing and work from `git show <sha>` / `gh pr diff`
rather than the working tree, which may be moving under you if the author is still committing. State the
SHA in your output. If you must run the suite, note that a mutating tree makes the result
uninterpretable — say so rather than reporting a number you cannot stand behind.

**Step 2 — Analyse the diff and form findings:**

Read the diff and any relevant surrounding files. Apply the review techniques below to produce a draft set of findings. Do **not** read PR comments or linked issue comments yet — reading them first anchors you to others' framing and weakens independent analysis.

**Step 3 — Fetch comments and refine:**

Run in parallel:
- `gh pr view <number> --comments`
- For any linked issues (e.g. `fixes #N`): `gh issue view <N> --comments`

Refine your findings using these to:
- Retract or downgrade findings already resolved in prior review rounds
- Distinguish pre-PR reports from post-PR reports in linked issues. **Pre-PR error reports are the motivation for the fix, not evidence against it.** Only post-PR reports are evidence of regressions introduced by this PR.
- Revisit root-cause diagnoses: if the thread shows a proposed fix was already tried and failed, revise the diagnosis accordingly.

Then write the review output using the format below. Include the contributor engagement assessment. Do **not** draft a contributor-facing comment yet — wait for the reviewer to discuss findings and ask for one. See the "Contributor comment" section for guidelines when that time comes.

**Review:**

Help a human reviewer understand what changed, find real defects, and decide where to focus their attention. You are **not** the approver.

We are a data science team. Our PRs touch ML pipelines, data transformations, model serving, evaluation frameworks, LLM-driven agents, and the glue between them. Reviews should reflect that context — data integrity, numerical correctness, eval validity, and model/pipeline behaviour matter more than generic software engineering checklists.

## Principles

- **Evidence over assertion.** Every finding needs a file path and line range (or function name + searchable token). No vague "somewhere in the diff."
- **Uncertainty is fine.** Label speculative findings as such. Include a confidence tag (High/Med/Low). Don't assert bugs you can't prove.
- **High-signal only.** Skip cosmetics. Focus on correctness, data integrity, silent failures, and whether tests/evals actually catch what they claim.
- **Current-run-correct is not enough — flag latent-unsafe paths.** When code is safe only because of how it happens to be invoked (a scorer that pins the validation baseline by convention with no guard against a test baseline; a fill value that's harmless only because eval never indexes it), and a plausible wrong invocation would silently leak, corrupt, or mis-score, that is a finding — sized by its blast radius, not by whether this run tripped it. Verifying the current run reproduces is necessary, not sufficient: say the run is clean *and* flag the unguarded path. This calibration is what most often separates a paranoid reviewer from a lenient one on a decision-grade gate; when in doubt on a gate, escalate the unguarded path rather than down-rate it to a provenance nitpick.
- **Don't hallucinate repo context.** Only assume what's visible in the diff, PR description, or files you've read. If a concern depends on something you haven't seen, ask — don't assert. This includes explanations for observed behaviour: don't assert *why* existing code behaves a certain way without reading it. Verify before putting it in a contributor comment.
- **Verify external references.** If a contributor references another plugin, library, or codebase to justify a design choice, fetch and read it before accepting the comparison. Don't characterise it based on inference.
- **Read beyond the diff.** Use Read, Grep, and Glob to examine surrounding code when tracing data flows, verifying contracts, or checking for stale references. The diff alone is rarely sufficient.
- **Carry the outside reviewer's disposition — you are usually the only reviewer.** A genuinely different model family rarely reviews our code, so the value has to come from *how* this review is run, not from who runs it. Assume code is safe only by how it happens to be invoked until proven otherwise, and escalate an unguarded path (above) rather than down-rate it because this run didn't trip it. Run isolated where you can (fresh session on the diff, no author transcript).

**Large PRs:** For PRs over ~500 lines, prioritise logic and pipeline files over generated outputs, data files, and lock files. State what you deprioritised and why.

---

## Review techniques (ranked by value, from practice)

These techniques apply broadly and are illustrated with real-world examples.

### 1. Question the premise — does this need to be built?

Before diving into implementation, ask two questions in order:

1. **Is this already solved externally?** Is there a well-maintained library, platform feature, or managed service that handles this problem? A correct, well-tested custom implementation of a solved problem is still a net negative — you inherit the maintenance burden, miss upstream improvements, and risk subtle divergence from battle-tested behaviour.

2. **Does this already exist internally?** Does the feature duplicate functionality already in the system, host platform, or upstream dependency?

If either answer is yes, the burden of proof shifts to the PR author to justify the custom path. Valid justifications exist (performance constraints, domain-specific requirements, licensing, avoiding a heavy dependency for a thin slice of functionality) — but they should be stated, not assumed.

**Example from practice:** A plugin PR added Ctrl+click on tags to open the host app's native tag view. The implementation was complex (DB lookup → N+1 API calls → undocumented command), but it was functionally identical to clicking the same tag in the host app's built-in sidebar. The entire code review became moot once the duplication was identified.

**How to do it:**
- State in one sentence what the feature gives the user or what problem the code solves
- Ask: can the user already achieve this through an existing internal or external path?
- Watch for PRs that bundle a genuinely new behaviour change alongside a redundant feature

### 2. Trace the data flow end-to-end

The highest-value bugs come from following a value from its entry point to its final use. Don't just read each file in isolation — trace the chain.

**Example from practice:** Serialised model files existed on disk, but a missing optional dependency caused `joblib.load()` to fail silently. The code caught the exception, returned `null` predictions, and a downstream LLM fabricated plausible-looking values to fill the gap. Found by asking "why is this null?" and following the chain: file exists → load fails → silent catch → null output → hallucinated report values.

**How to do it:**
- Pick a parameter, config value, or data artifact introduced in the PR
- Follow it from where it enters the system to where it's consumed
- Check: is it actually forwarded? Is it silently dropped? Is the error path swallowing information?

### 3. Additional checks

Apply these as relevant — not every PR needs all of them.

- **Tests/evals: genuine or weakened?** When a PR changes tests, ask: "what could now pass that shouldn't?" Watch for granular checks replaced by coarse ones, content verification replaced by execution-only checks, and large simplifications (-600 lines) that lose capability.
- **Cross-reference claims against reality.** If the PR claims to close issues, check the diff delivers each acceptance criterion. Check for stale references to deleted/renamed things. Distinguish pre-PR error reports (motivation for the fix) from post-PR reports (evidence of regression).
- **Validate outputs against inputs.** Can the system's outputs actually be derived from its inputs? For ML: do features match what the model was trained on? For LLM agents: does the prompt encourage claims the structured tool outputs can't ground? (e.g., prompt examples show hourly breakdowns but the tool only computes daily aggregates.)
- **Ask "where does X go?"** For each new artifact the PR introduces, check who consumes it, through what path, and whether the consumer resolves it. Watch for parallel pipelines doing similar things independently.
- **Check dependency and environment assumptions.** Code that loads models or imports optional packages often fails silently when the environment doesn't match. Search for `try/except ImportError` and check whether new dependencies are in the right group (required vs optional).

### 4. Ask whether the change is COMPLETE, not just correct

The techniques above ask "is this code right?" This one asks "is this all of it?" Incompleteness is
harder to see than incorrectness, because every line in the diff can be correct while the set of lines
is wrong. Review naturally anchors on what the author touched; this is the deliberate correction.

**Example from practice:** A PR made leaderboard latest-row selection Track-aware, changing
`PARTITION BY atomic_unit_id` to include the Track. It changed two sites and its commit message said
"both surfaces that compute it". `grep -rn "PARTITION BY atomic_unit_id"` found **four** — two more in
`databricks/sql/*.sql`, live files a reviewer pastes straight into a warehouse. Worse, an existing test
asserted the OLD partition key and still passed, so the repo held two tests making contradictory claims
about one concept. That green assertion is exactly why nobody noticed.

**How to do it:**

- **Grep for the concept, not the file.** Take each identifier, column name, flag, or rule the PR
  changes and search the whole repo for its OLD form — across `.py`, `.sql`, `.json`, `.md`, `.ts`,
  dashboards, fixtures. The author saw the sites in their language; the misses are usually in another.
- **Hunt the green test that pins the old behaviour.** After any behaviour change, grep the test suite
  for the superseded form. A test asserting the old contract that still passes is near-proof the change
  is partial, and it will keep the gap invisible indefinitely. Two tests asserting incompatible things
  about one concept is always a finding.
- **Treat docs as callers.** A changed CLI flag, function signature, or required argument breaks every
  runbook, README and walkthrough that tells a human to invoke the old way. These fail at *runtime for a
  person*, not at import for a machine, so no test catches them. Grep docs for the command or symbol.
  In the example above, two documented invocations began exiting 1 and neither was updated.
- **Ask what else derives the same thing.** If the PR changes one derivation of a value, find the
  others. Duplicated derivations are the normal case in mature code, not the exception.

### 5. Check whether a test can actually fail

A test that cannot fail is worse than no test: it occupies the space where a real check would go and
reports green forever.

**Examples from practice:** (1) A conformance test asserted that four MLflow tags were present — but its
own helper had reimplemented those tag emissions rather than calling the producer, so the assertions
checked the fixture against itself. Deleting the real emission left the suite green. (2) A test compared
shared columns across two surfaces and silently skipped any that were blank on both; it compared 32 of
51 and reported success.

**How to do it:**

- **Trace each assertion back to what produced the value.** If the test's own setup produced it, the
  test is circular. Fixtures should stand in for *inputs*, never for the behaviour under test.
- **Look for silent skips.** Any `continue`, `if x not in y`, or filter inside a test loop is coverage
  quietly disappearing. The test should assert what it actually covered, not just that what it covered
  passed.
- **Break it on purpose.** The strongest verification is to introduce the defect the test claims to
  catch and confirm it fails. Prefer editing the real source on disk, running, then restoring, over
  monkeypatching: `from X import Y` binds into the importing module, so patching the source module's
  attribute leaves the test's binding untouched and makes a working guard look broken.

### 6. Question complexity that compensates for wrong architecture

When new code introduces significant complexity to work around a constraint, ask whether the constraint itself should exist. The simplest fix is often to remove the constraint, not engineer around it.

**Example from practice:** A panel chat feature added `sessionStorage` persistence, a `MutationObserver`, and a hydration layer to survive DOM resets on every note switch. All of it was correct. All of it was unnecessary — it existed solely because the chat was embedded in the same panel whose HTML gets replaced. A separate panel would have made the entire layer redundant.

**How to do it:**
- When you see a cluster of defensive code (persistence, observers, hydration, retry logic), ask: what is this defending against?
- Trace that constraint back to its source. Is the constraint inherent to the problem, or is it an architectural choice that could be revisited?
- If removing the constraint would eliminate the complexity, flag the architecture, not just the implementation.

**Concrete checks (apply when relevant):**

- **Refactor three-filter:** (1) shrinks the public interface? (2) would a YAGNI version solve ~80% of the friction? (3) is there real bug or test pain driving it? Drop candidates failing two.
- **Deletion test for "shallow/façade" claims:** if removed, does complexity sink into helpers below (keep digging), scatter across callers (label is wrong, module is deep), or need re-derivation (module owns real logic)?

### 7. Verify implicit contracts at system boundaries

When code consumes something produced by a separate process — a trained model, a config file, a database schema, an API response, a data pipeline output — it relies on unspoken assumptions about what that thing contains. Each side looks correct in isolation. Bugs live at the seam, and silent degradation (column filtering, default values, fallback branches) means no error is raised.

**Example from practice:** A serving PR loaded a pre-trained prediction model and listed `weight_kg` as an expected input feature. The serving code was correct. But loading the actual serialised model and inspecting `scaler.feature_names_in_` showed 66 features — `weight_kg` not among them. The training config (stored only in a remote experiment tracker, never committed) had a typo: `weight_kgs` instead of `weight_kg`. A silent column-intersection filter dropped the misspelled column without warning. The same config also recorded `baseline_choice: Ridge` while the actual model was XGBRegressor. Two metadata lies, one missing feature, zero errors raised. Found by crossing the code boundary and inspecting the artifact directly.

**How to do it:**
- Identify every boundary where the PR's code consumes something produced elsewhere (model artifacts, configs, schemas, API contracts, upstream pipeline outputs)
- For each boundary, list the assumptions the code makes about the thing it consumes (expected columns, types, keys, response shapes)
- Verify at least one assumption by inspecting the actual artifact — load the model, read the config, query the schema. Don't trust documentation or variable names alone
- Look for silent adaptation patterns that hide mismatches: `df[df.columns.intersection(expected)]`, `.get(key, default)`, bare `except` clauses. These are where contract violations disappear instead of surfacing
- Check whether the contract is enforced anywhere (schema validation, assertions, column-presence checks) or purely implicit. If implicit, flag the gap

---

## Output format

Adapt the format to the PR. Don't force rigid sections when they add no value. The core deliverables are:

### Orientation
What this PR does, why, and what it touches. Keep it short — the reviewer should understand scope in 30 seconds.

### Change map
A table grouping changes by area, with risk level (Low / Medium / High) and a one-line "why risky" for anything Medium or High.

### Findings (prioritised)
Each finding needs:
- **Severity**: Blocker / Important / Suggestion
- **Title**: one line
- **Evidence**: `path:line-range` or function name + searchable token
- **Why it matters**: what breaks, what's silent, what's unverified
- **Confidence**: High / Med / Low
- **Recommendation**: what to do about it

Severity guidance:
- **Blocker**: likely bug, data corruption, silent failure masking real problems, fabricated/hallucinated outputs, model using wrong inputs
- **Important**: correctness edge-case, weakened tests/evals, missing coverage for core behaviour, parameter silently ignored, dead code that misleads, a latent-unsafe path unguarded against a plausible wrong invocation (safe-as-invoked ≠ safe)
- **Suggestion**: cleanup, stale references, minor inconsistency, nice-to-have tests

### Follow-ups
Unrelated bugs surfaced during review go here as one-paragraph drafts, not folded into the in-flight PR's findings. Keeps the PR one-feature.

### Where the human should look
Explicitly list the 3-5 specific files/locations the human reviewer should read themselves, with a one-line reason for each. The reviewer's time is scarce — direct it to the highest-leverage spots.

### Questions for the author
Only questions that materially reduce risk. Each must say why you need the answer and where in the code it matters.

### Contributor engagement assessment
Evaluate the PR for signs of thoughtful work vs low-engagement / unreviewed agent output. This is not about whether agents were used — it's about whether the contributor understands what they're submitting. Assess code and PR communication independently (a contributor may use agents for code but engage thoughtfully in discussion, or vice versa).

Signals to look for:
- **Code understanding:** Does new code reuse existing patterns and call into existing functions, or does it duplicate logic from scratch? Are unrelated changes bundled without explanation?
- **PR communication:** Does the description explain implementation decisions, or just echo the issue text? Do responses to review comments engage with the questions asked, or summarise what was done?
- **Slop markers:** Inconsistent formatting that doesn't match surrounding code. Mechanical edge-case handling (silent fallbacks, bare catches) without considering UX implications. Generic commit messages. Versioned storage keys and `// no-op` catch blocks as boilerplate. Complete rewrites between rounds submitted without any explanation of what changed or why — this is stronger signal than any single code quality issue.
- **Cross-PR patterns:** If the contributor has prior PRs on the repo, check those interactions. Do they engage with questions or just post summaries of what they changed? This is stronger signal than any single PR.

State your read briefly. This helps the reviewer calibrate how much architectural guidance to give vs comprehension questions to ask.

---

## Contributor comment

After the reviewer has discussed and finalised the review findings, they may ask for a contributor-facing PR comment draft. When drafting:

**Tone and format:**
- Human, collegial. No corporate speak, no jargon walls, no em dashes. Write like a maintainer, not a review bot.
- Plain paragraphs, not bold headers or bullet walls. Numbered lists are fine for questions. A PR comment that looks like a structured report reads as agent-generated.
- Be direct about problems without being condescending. Nudge the contributor to discover issues themselves ("try switching notes during a chat session and see what happens") rather than stating the bug.
- Acknowledge what works briefly and specifically, but be careful what you reinforce.

**Structure:**
- Lead with findings that would change the entire approach. Save smaller items for later.
- Ask before telling. "What does X do in the existing flow?" reveals more than "you're missing X."
- Keep it short. One paragraph of substance, a few pointed questions.
- End with something specific, not a generic closer.
- Match guidance depth to evidence of effort. Genuine codebase engagement gets specific technical pointers. Low engagement gets comprehension questions first. Don't prescribe architectural solutions before verifying the contributor understands the current architecture.

**Low-engagement multi-round PRs:**
- If comprehension hasn't been demonstrated after one or more rounds, lead with: "Before we go further, can you walk me through the main architectural decisions in this PR and why you made them?"
- Don't give further findings until you have an answer.
- Questions with no Googleable answer (e.g. "why did you put X here rather than Y?") reveal understanding better than questions with obvious answers. If the contributor doesn't engage, that is itself the signal.

**Probing questions:**
- Test whether the contributor understands the existing code, not just whether they can fix a specific line.
- Don't ask questions you've already answered in the comment.
- If the PR description already explains a choice, don't re-ask about it.

---

## What to skip

- **Cosmetics.** Don't comment on naming, formatting, or style unless it causes confusion.
- **Style and visual comments when architecture is unsettled.** If structural findings are still open (wrong component boundary, duplicated pipeline, missing abstraction), defer CSS and layout feedback to a later round. Flag them internally but don't raise them — the structure may change and the style discussion becomes moot.
- **Generic checklists.** Don't mechanically run through security/auth/deploy/rollback checklists unless the PR actually touches those areas. Irrelevant checklist items are noise.
- **Merge risk summaries.** The findings speak for themselves. Don't add a "safe to merge" / "needs changes" label — that's the human's call.
- **Boilerplate sections.** If a section would be empty or trivially "N/A", omit it entirely.
