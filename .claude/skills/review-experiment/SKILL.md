---
name: review-experiment
description: Critically review research experiment artifacts against a supplied project process context. Use when asked to review, audit, stress-test, or red-team an iteration summary, experiment registry, findings document, sweep result, or reasoning chain; focuses on claim/evidence consistency, local process-contract compliance, statistical validity, implementation fidelity, and unresolved assumptions.
---

# Review Experiment

Use this skill as an adversarial reviewer for iterative research workflows. Treat the
project's process context as the source of truth; this skill supplies the review method,
not project-specific rules.

## Modes

- **Per-iteration**: one iteration summary, sweep result, or experiment report is in
  scope. Review the current artifact's claims, implementation, process compliance, and
  internal consistency.
- **Registry / end-of-experiment**: a registry, findings document, campaign memory, or
  closeout summary is in scope. Review the accumulated reasoning chain: drift,
  inconsistency, stale memory, and whether conclusions follow from evidence.

If both are provided, start per-iteration, then extend to registry mode.

## Inputs

Accept, when available:

- iteration summary path
- process context path
- registry, findings, or campaign-memory path
- referenced code or artifact paths

If the process context is missing, ask for it before doing a process-compliance review.
If the user explicitly asks for an artifact-only review, proceed but mark all process
claims as unverified.

Use read-only inspection. Do not mutate project files during review.
Do not infer project process rules from conversation history; use the supplied process
context and artifacts.

## Review Isolation

Prefer a fresh sub-agent, task, fork, or new thread for this skill. Use isolation by
default when any of these are true:

- the current session helped plan, run, analyze, or write the artifact under review
- the review is decision-grade, registry-level, end-of-experiment, or likely to update
  canonical memory
- the result is positive, surprising, contested, or reverses prior project memory
- the review depends on subtle code/artifact/provenance consistency

Pass the sub-agent raw inputs only: the skill name or path, the user request, process
context path, artifact paths, registry/findings paths, and any explicitly relevant code or
artifact paths. Do not pass suspected bugs, expected findings, prior conclusions, or
private reasoning unless the user explicitly asks for a second-opinion review of those
conclusions.

Main-session review is acceptable for lightweight schema checks, quick artifact triage,
or when sub-agents are unavailable. If the review is done in the main session, state that
the review was not isolated and treat inherited context as a residual risk. If isolation
is available and skipped for a decision-grade review, record that as a review-method
limitation.

## Operating Stance

Be a critical reviewer, not a validator. Find the highest-consequence risks the
scientist may have missed, especially mismatches between what was claimed and what was
actually computed. Do not make the final proceed / do-not-proceed decision.

Prefer evidence over assertion. Every finding needs a file path, field name, artifact
identifier, function name, or line reference. If a load-bearing artifact is unavailable,
do not assert a defect and do not silently skip it; record "unverified - depends on X"
and name the artifact or check that would settle it. An unchecked load-bearing assumption
is itself a finding.

Treat code and persisted artifacts as ground truth when they conflict with narrative
claims. Surface the mismatch; do not resolve it away. Label speculative findings and tag
confidence as High, Medium, or Low.

## Workflow

### Step 1 - Build The Project Process Contract

Read the supplied process context first. Extract the local rules without importing
project assumptions from this skill:

- loop phases and the decision the process supports
- required artifacts and canonical vs deprecated memory sources
- artifact schema, required frontmatter/body sections, and legacy exceptions
- health gates, full-eval gates, thresholds, screen caps, and escalation rules
- comparator/bar discipline and required statistical evidence
- project-specific naming traps, provenance rules, and artifact-routing rules
- what must be treated as unverified when missing

If the process context does not define one of these, report that as a process-contract
gap rather than filling it from this skill.

### Step 2 - Orient To The Artifact

Read in order:

1. Process context and the extracted process contract.
2. The iteration summary or registry/finding artifact under review.
3. Frontmatter or equivalent metadata: type, status, claim scope, comparator/bar,
   decision-gated IDs, provenance, canonical artifacts, reproducibility, and any
   project-defined required fields.
4. Body sections: hypothesis, triage/health gate, promoted results, spec-vs-implementation
   record, calibration/provenance, decision, and next step. Enumerate sections first if
   the summary contains multiple appended passes.
5. Registry/findings/campaign memory if provided, especially candidate queues, current
   bests, dead ends, stale claims, or local equivalents.
6. Referenced code and artifacts behind headline claims, gates, and produced numbers.
   For remote artifacts, attempt read-only inspection when available; if unreachable,
   apply the unverified-finding discipline.

Avoid reading prior iteration summaries until registry-chain review or until chasing a
specific discrepancy; prior narratives can anchor the review too early.

### Multi-section Summaries

A single summary may contain appended sections from successive commits, synthetic then
real-data runs, recalibrations, or post-hoc corrections. Audit each section's claim
against its own declared bar and process stage. Check for internal drift across sections:
later verdicts contradicting earlier ones, recalibrated thresholds retroactively changing
a verdict, or persisted artifacts recording one verdict while the narrative reports
another because code changed after the artifact was written.

### Step 3 - Apply Four Lenses

Use `references/review_lenses.md` for detailed checks.

Each check should have one primary home. Overlap is not intended, but independent
detection is meaningful: promote a finding one severity level when it is reached by two
or more lenses.

- **Lens 0 - Process Contract Auditor.** Does the artifact obey the local process it
  claims to follow?
- **Lens A - The Saboteur.** Can the result be explained as a data, optimization,
  evaluation, or selection artifact rather than the stated hypothesis?
- **Lens B - The Statistician.** Do claims say more than the estimates, intervals,
  sample sizes, seed spread, or predeclared bars support?
- **Lens C - The Implementation Auditor.** Does the code or artifact path implement what
  the summary says it implements?

If a lens finds no defect, state the single strongest assumption that lens relies on and
why it is plausible. Do not manufacture cosmetic findings to fill a quota.

Self-review trap breaker: if a lens comes up empty, read the relevant code bottom-up,
state each function's contract before its body, and assume every external input could be
malformed and every artifact load could silently return the wrong thing.

### Step 4 - Audit The Reasoning Chain

When a registry, findings document, or end-of-experiment summary is in scope, check:

- hypothesis drift without reconciliation
- screen, analysis, or exploration caps defined by the local process
- asymmetric treatment of positive and null evidence
- dead-end reopening without the declared reopen condition
- contradictions with current bests, dead ends, stale claims, or local equivalents
- headline results that should update campaign memory but were not reflected there
- local reconciliation requirements when a current claim contradicts a prior curated
  finding, candidate, or dead-end

### Step 5 - Prioritize

Rank findings by consequence if wrong times likelihood of being wrong. Surface the top
2-3 first. A correctly ranked short list beats a comprehensive grab bag.

## Legacy Or Migrating Artifacts

If an artifact predates the current schema, classify it as legacy. Infer only what is
explicit in the body, registry, or process context. Do not invent missing frontmatter
values. Missing schema is usually a suggestion for historical artifacts, but can be
important or blocking for new, reconstructed, or decision-gating artifacts when the
current process requires schema fields.

## Output Format

Adapt to the artifact; do not force empty sections.

- **Orientation**: one paragraph covering the claim, decision informed, headline result,
  and why this review is higher or lower risk than typical.
- **Review isolation**: say whether the review used a fresh sub-agent/fork/thread or the
  main session; if main-session, name the residual contamination risk.
- **Process contract extracted**: short list of the local rules that mattered most.
- **Risk map**: table of area | risk | one-line reason for Medium/High.
- **Findings, prioritized**: severity, lens(es), title, evidence, why it matters, and
  confidence or "unverified - depends on X". Note severity promotion when two or more
  lenses independently detect the issue.
- **Assumptions surfaced**: for each lens with no finding, the strongest remaining
  assumption.
- **Questions**: 2-3 questions that would most reduce residual risk; do not bury a
  blocker as a question.
- **What to verify next**: 1-3 files, functions, or artifacts the scientist should read.

## What To Skip

Skip cosmetic prose issues, generic ML advice not grounded in the artifact, process rules
that were already correctly followed, and findings depending entirely on inaccessible
files unless framed as unverified load-bearing assumptions. Do not issue a proceed /
do-not-proceed verdict; that remains the scientist's call.
