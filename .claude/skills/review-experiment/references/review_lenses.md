# Review lenses

Load this file when applying the review lenses from `SKILL.md`.
Each check has one primary home. If the same issue is independently found by multiple
lenses, record the agreement and raise confidence if warranted. Set severity from consequence
and likelihood, not the number of lenses that detected it.

## Contents

- Lens 0 - Experiment Process Contract Auditor
- Lens A - The Saboteur
- Lens B - The Statistician
- Lens C - The Implementation Auditor
- Reasoning-chain Audit
- Severity Guidance

## Lens 0 - Experiment Process Contract Auditor

Check the artifact against the project process contract extracted from the supplied
context:

- Required read/write order: Did the artifact use the canonical project memory, registry,
  and evidence sources? Flag deprecated or noncanonical sources used as ground truth.
- Required schema: Are project-required frontmatter fields, metadata fields, or body
  sections present and meaningfully populated? Distinguish current artifacts from legacy
  exceptions.
- Gate discipline: Were health gates, full-eval gates, budget gates, screen caps, and
  escalation rules applied as the process defines them?
- Comparator/bar discipline: Does every claim inherit or explicitly override the declared
  comparator? Flag substitution of an easier campaign bar for a stronger gate bar.
- Process-specific setup: For planned screens, calibrations, literature checks, or
  spec-vs-implementation records, did the project-required setup occur before the result
  was interpreted?
- Memory update discipline: Should this result update a current-best, dead-end, stale
  claim, candidate queue, or local equivalent? If yes, was that update made or reconciled?
- Provenance discipline: Do canonical artifacts, reproducibility commands, ownership,
  review status, and reconstructed-artifact provenance match the process requirements?
- Legacy handling: If the artifact predates the current schema, did the review classify
  it as legacy and infer only from explicit evidence rather than invented metadata?

## Lens A - The Saboteur

For each positive or decision-shaping result, construct the most plausible
non-hypothesis explanation. Ask whether the existing evidence rules it out and, if not,
what single check would.

### Data artifacts

- Split leakage: Were participant, subject, entity, or time splits applied before any
  feature construction, embedding lookup, augmentation, normalization, or cache creation
  that could leak information? Verify a training entity cannot appear in the validation
  or test cohort when the process requires entity-level separation.
- Cohort or batch effects: Could cohort, site, device, collection period, batch, or
  missingness explain the signal?
- Embedding/cache provenance: If frozen features or cached embeddings are used, is the
  source data and version verified to match the claimed cohort and not overlap improperly?
- Label noise and proxies: Does the metric and interpretation account for noisy,
  surrogate, proxy, or weak labels? A model memorizing noise can show high in-sample fit
  and poor generalization.

### Optimization artifacts

- Collapse or degeneracy: Could a collapsed, constant, identity-leaking, or batch-effect
  solution pass the recorded health gates?
- Near-threshold gates: Is a pass resting on a metric barely under or over a hard
  threshold, making it fragile?
- Inert mechanism: Did a new loss, head, augmentation, or data path contribute meaningful
  signal to the shared model, or merely run without effect? For auxiliary losses, prefer
  evidence of gradient contribution into the shared module over loss scale alone.

### Evaluation artifacts

- Aggregation axis: Could a wrong grouping, time window, subject axis, or reduction
  produce the observed number?
- Baseline construction: Was the baseline fitted, scored, and aggregated in the same way
  the claim says?
- Metric definition: Does the reported metric answer the stated question?
- Row alignment: Are predictions, labels, embeddings, and metadata aligned by stable IDs
  rather than incidental order?
- Reimplementation divergence: If the result reimplements an official evaluator,
  baseline, or metric, did it validate against the canonical implementation or output?

### Selection artifacts

- Best-of-N: Was the result selected from many seeds, configs, targets, probes, or
  candidate metrics without accounting for that search? Ask what result would be
  expected under the null after the same search.
- Post-hoc trigger: Was a check, threshold, or target chosen after seeing the result?
- Asymmetric follow-up: Are positive results promoted while nulls are explained away by
  panel limitations, power, or data quality without applying the same standard both ways?

## Lens B - The Statistician

Check whether the claim exceeds the statistical evidence.

- Estimate vs interval: Is the headline claim backed by the interval or uncertainty
  measure required by the local process, rather than a point estimate alone?
- Gate vs sanity floor: Are exploratory p-values, permutation tests, or sanity checks
  being used as decision gates when the process says a different statistic controls?
  Conversely, if the local process requires a permutation/null check for a small-n or
  noisy setting, was it predeclared and run with the right procedure?
- Sample size and power: Are small-n positives protected against noise-manufactured
  signal? Are nulls worded as "no evidence under this probe" rather than "ruled out"?
- Seed spread: Does the result hold across the required number of seeds or replicates, or
  is a median driven by one strong run?
- Predeclaration: Were thresholds, permutation budgets, target panels, and flip criteria
  declared before the result when the process requires that?
- Reference level: Is the reference level named precisely enough that readers cannot
  confuse a weaker baseline with the actual gate?
- Verdict strength: Does a provisional, scout, screen-grade, or exploratory result use
  body text that reads like a gate-level conclusion?
- Multiple comparisons: Does the narrative account for the number of tried cells,
  surrogates, probes, and configurations?

## Lens C - The Implementation Auditor

For each quantitative or mechanism claim, find the function, script, config, or artifact
that produced it. Check:

- The code path was actually exercised by the run under review.
- Config names match real consumed fields, not unused or silently ignored knobs.
- Aggregation, normalization, split filtering, baseline fitting, and scoring match the
  claim.
- Silent fallbacks such as missing-column intersections, `.get(..., default)`, broad
  `except`, default cache paths, or permissive joins did not hide a mismatch.
- Pathological or excluded children did not leak into promoted/full-eval results.
- Project-specific naming traps from the process context are respected.
- A mechanism-change summary includes a spec-vs-implementation record when the process
  requires one.
- Any calibration claim measures the quantity the process cares about, not just a proxy.
- Triage completeness: When the process launches multiple children/runs, does the triage
  table cover all of them? Are branch or run-specific exceptions scoped only to the
  branches/runs where the process declares them?
- Probe integrity: If a linear probe, diagnostic classifier, or representation probe is
  used, is it actually constrained as described and fit on an appropriate split?
- Augmentation or positive-pair leakage: Does the training or evaluation create pairs
  sharing information unavailable at inference time, such as held-out labels, future
  timestamps, or target-derived features?
- Objective/downstream mismatch: Does the pretext objective's geometry plausibly support
  the downstream target, or could it reward an axis irrelevant to the claim?

## Reasoning-chain Audit

Use this in registry / findings / end-of-experiment mode:

- Hypothesis drift: Did the stated hypothesis change across iterations without explicit
  reconciliation?
- Screen or exploration cap: Count against the stable decision/candidate IDs defined by
  the process. Flag a proposed extra screen when the process requires build/escalation.
- Asymmetric evidence treatment: Treat panel limitations, power caveats, and data-quality
  caveats symmetrically for positive and null evidence.
- Dead-end reopen discipline: Has a dead end or closed branch been reopened without the
  declared reopen condition?
- Reconciliation row: Does a current claim contradict a current-best/dead-end/stale
  entry or local equivalent without a row explaining which bar, reference level, cohort,
  aggregation, pipeline, or artifact differs?
- Curated-memory staleness: Is there a headline result that should update canonical
  memory but no rewrite, replacement, or reconciliation is recorded?

## Severity Guidance

- **Blocker**: the result rests on code that does not implement the claim; split leakage;
  an easier bar is presented as the gate; a pathological/excluded child supports a
  promoted claim; a required canonical artifact contradicts the narrative; a persisted
  gate artifact records a failing verdict that the narrative silently overrides; the
  conclusion would likely reverse under the correct implementation.
- **Important**: a plausible alternative explanation is not ruled out; required
  uncertainty is absent for a decision-grade claim; process caps or escalation rules are
  about to be exceeded; a spec-vs-implementation mismatch is not recorded; canonical
  memory is stale or unreconciled; convergence is conflated with fidelity; near-threshold
  health passes or internal drift across summary sections carry the claim.
- **Suggestion**: minor inconsistency, stale link, missing convenience provenance, or a
  cheap check that would materially reduce residual uncertainty.
