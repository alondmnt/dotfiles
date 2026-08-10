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

Read the diff and any relevant surrounding files. Apply the techniques in `references/review_techniques.md` to produce a draft set of findings. Do **not** read PR comments or linked issue comments yet — reading them first anchors you to others' framing and weakens independent analysis.

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

## Review techniques

Seven techniques, ranked by value from practice, live in
[`references/review_techniques.md`](references/review_techniques.md), each with the example
that makes it recognisable. Read that file before forming findings.

1. **Question the premise** - does this need to be built, or does it already exist internally or externally?
2. **Trace the data flow end-to-end** - follow one value from entry to final use; the silent drop is in the middle.
3. **Additional checks** - weakened tests, stale references, outputs not derivable from inputs, orphaned artifacts, environment assumptions.
4. **Is the change COMPLETE, not just correct** - grep the concept's old form repo-wide; hunt the green test pinning the old contract; treat docs as callers.
5. **Can this test actually fail** - circular assertions, silent skips, break it on purpose.
6. **Complexity compensating for wrong architecture** - trace defensive clusters back to the constraint that forces them.
7. **Implicit contracts at system boundaries** - inspect the artifact the code consumes; don't trust its variable names.

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
