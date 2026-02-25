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

**Step 2 — Review:**

Help a human reviewer understand what changed, find real defects, and decide where to focus their attention. You are **not** the approver.

We are a data science team. Our PRs touch ML pipelines, data transformations, model serving, evaluation frameworks, LLM-driven agents, and the glue between them. Reviews should reflect that context — data integrity, numerical correctness, eval validity, and model/pipeline behaviour matter more than generic software engineering checklists.

## Principles

- **Evidence over assertion.** Every finding needs a file path and line range (or function name + searchable token). No vague "somewhere in the diff."
- **Uncertainty is fine.** Label speculative findings as such. Include a confidence tag (High/Med/Low). Don't assert bugs you can't prove.
- **High-signal only.** Skip cosmetics. Focus on correctness, data integrity, silent failures, and whether tests/evals actually catch what they claim.
- **Don't hallucinate repo context.** Only assume what's visible in the diff, PR description, or files you've read. If a concern depends on something you haven't seen, ask — don't assert.
- **Read beyond the diff.** Use Read, Grep, and Glob to examine surrounding code when tracing data flows, verifying contracts, or checking for stale references. The diff alone is rarely sufficient.

**Large PRs:** For PRs over ~500 lines, prioritise logic and pipeline files over generated outputs, data files, and lock files. State what you deprioritised and why.

---

## Review techniques (ranked by value, from practice)

These techniques apply broadly and are illustrated with real-world examples.

### 1. Question the premise — does this need to be built?

Before diving into implementation, ask two questions in order:

1. **Is this already solved externally?** Is there a well-maintained library, platform feature, or managed service that handles this problem? A correct, well-tested custom implementation of a solved problem is still a net negative — you inherit the maintenance burden, miss upstream improvements, and risk subtle divergence from battle-tested behaviour.

2. **Does this already exist internally?** Does the feature duplicate functionality already in the system, host platform, or upstream dependency?

If either answer is yes, the burden of proof shifts to the PR author to justify the custom path. Valid justifications exist (performance constraints, domain-specific requirements, licensing, avoiding a heavy dependency for a thin slice of functionality) — but they should be stated, not assumed.

**Example from practice:** A PR added a custom retry-with-exponential-backoff wrapper around HTTP calls. The implementation was correct but duplicated `tenacity`, already in the project's dependencies.

**Example from practice:** A plugin PR added Ctrl+click on tags to open the host app's native tag view. The implementation was complex (DB lookup → N+1 API calls → undocumented command), but the core question was simpler: this is functionally identical to clicking the same tag in the host app's built-in tag sidebar. The entire code review became moot once the duplication was identified.

**How to do it:**
- Before reviewing implementation details, state in one sentence what the feature gives the user or what problem the code solves
- Ask: is this a known, solved problem? Is there an off-the-shelf solution (library, service, platform feature, config change) that fits?
- Ask: can the user or system already achieve this through an existing internal path?
- If a custom implementation is justified, check whether it wraps or vendors the external solution rather than reimplementing from scratch
- Watch for PRs that bundle a genuinely new behaviour change alongside a redundant feature (e.g., auto-showing a panel smuggled in ungated)

### 2. Trace the data flow end-to-end

The highest-value bugs come from following a value from its entry point to its final use. Don't just read each file in isolation — trace the chain.

**Example from practice:** A tool accepted a `region` parameter, but the serving layer hardcoded `region="low"` and never forwarded it. Each file looked correct in isolation. The bug was only visible by tracing the parameter across three files.

**Example from practice:** Serialised model files existed on disk, but a missing optional dependency caused `joblib.load()` to fail silently. The code caught the exception, returned `null` predictions, and a downstream LLM fabricated plausible-looking values to fill the gap. Found by asking "why is this null?" and following the chain: file exists → load fails → silent catch → null output → hallucinated report values.

**How to do it:**
- Pick a parameter, config value, or data artifact introduced in the PR
- Follow it from where it enters the system to where it's consumed
- Check: is it actually forwarded? Is it silently dropped? Is the error path swallowing information?

### 3. Check whether tests/evals are genuine or weakened

When a PR fixes failing tests or evals, ask: **did it fix the test to match correct behaviour, or weaken the test to pass the current output?**

**Example from practice:** An eval went from per-section granular checking to document-level checking. One `N=X` anywhere in the output now satisfied all statistical claims everywhere. Another eval went from content verification (checking reported values match tool output) to tool-execution-only checking (did the tool run?). Both made the eval suite pass, but fabricated data would no longer be caught.

**How to do it:**
- For each test/eval change, ask: "What could now pass that shouldn't?"
- Check if the fix narrows false positives (good) or also opens false negatives (bad)
- Pay special attention to large simplifications (-600 lines) — what capability was lost?

### 4. Cross-reference claims against reality

When a PR claims to close issues, verify the claims against the actual code.

**Example from practice:** PR claimed to close 4 issues. Cross-referencing showed: user metadata contained real clinical values but the report used hardcoded defaults instead, a required risk score was mandated by the spec but missing from the output entirely, and a deleted config file left orphan references in 5 other files.

**How to do it:**
- Read each linked issue's acceptance criteria
- Check the diff delivers each criterion, not just the happy path
- Check for stale references to deleted/renamed things

### 5. Validate outputs against inputs

Check whether the system's outputs can actually be derived from its inputs. This applies at every level — model predictions from features, report claims from tool outputs, visualisations from data.

**For ML pipelines:** Do the features flowing into the model match what it was trained on? Are transformations applied in the right order? Are default/fallback values used when real data exists?

**For LLM/agent systems (grounding audit):** Check whether the LLM has the structured data to make the claims its prompt encourages. Read the prompt examples, identify each specific claim pattern, and verify it maps to a structured tool output.

**Example from practice:** An agent prompt showed examples like "sensor reading was 95-110 units from 6-10 AM." But the only tool computed daily aggregates — no hourly breakdowns existed. The LLM was incentivised to produce specific-sounding temporal claims with no data to ground them. Separately, a detailed event-level DataFrame (58 rows) was available via accessor tools and could ground the temporal patterns — but the prompt didn't instruct the agent to use it.

**Example from practice:** A report used "estimated default" values for key biomarkers, even though the user's metadata contained real measurements. The model prediction was computed from wrong inputs. Found by cross-referencing the report text against the metadata JSON.

**How to do it:**
- Identify inputs (features, metadata, tool outputs) and outputs (predictions, reports, visualisations)
- For each output claim or value: can it be traced to a specific input? Or is it fabricated/defaulted when real data exists?
- For LLM systems: check accessor tool limits (e.g., `df_head(n=100)`) against actual data sizes — can the LLM see enough data to make its claims?

### 6. Ask "where does X go?"

Simple flow questions often reveal architectural gaps.

**Example from practice:** "Where are the visualisations stored? Embedded?" revealed that markdown referenced `./artifacts/plot_*.png` (relative paths to ephemeral files), the PDF generated completely different plots via a separate pipeline, and neither path actually rendered correctly. Two parallel visualisation systems, neither complete.

**How to do it:**
- For each new artifact/output the PR introduces, ask: who consumes it? Through what path? Does the consumer actually resolve it?
- Check for parallel pipelines doing similar things independently

### 7. Check dependency and environment assumptions

Code that loads models, reads configs, or imports optional packages often fails silently when the environment doesn't match the developer's setup. These are high-value findings because they cause subtle wrong results, not crashes.

**Example from practice:** Serialised model files existed on disk but an optional dependency wasn't installed. `joblib.load()` raised `ModuleNotFoundError`, the code caught it silently, and uncertainty estimates came back null. Downstream, the LLM fabricated plausible intervals because the prompt examples showed them. One missing pip package caused fabricated numerical output.

**How to do it:**
- For new model/data files added in the PR, check that their loader dependencies are in `pyproject.toml` / `requirements.txt` / conda env
- Search for `try/except ImportError` patterns — what degrades silently when a package is missing?
- Check `pyproject.toml` changes: are new dependencies in the right group (required vs optional)?

### 8. Verify implicit contracts at system boundaries

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
- **Important**: correctness edge-case, weakened tests/evals, missing coverage for core behaviour, parameter silently ignored, dead code that misleads
- **Suggestion**: cleanup, stale references, minor inconsistency, nice-to-have tests

### Where the human should look
Explicitly list the 3-5 specific files/locations the human reviewer should read themselves, with a one-line reason for each. The reviewer's time is scarce — direct it to the highest-leverage spots.

### Questions for the author
Only questions that materially reduce risk. Each must say why you need the answer and where in the code it matters.

---

## What to skip

- **Cosmetics.** Don't comment on naming, formatting, or style unless it causes confusion.
- **Generic checklists.** Don't mechanically run through security/auth/deploy/rollback checklists unless the PR actually touches those areas. Irrelevant checklist items are noise.
- **Merge risk summaries.** The findings speak for themselves. Don't add a "safe to merge" / "needs changes" label — that's the human's call.
- **Boilerplate sections.** If a section would be empty or trivially "N/A", omit it entirely.
