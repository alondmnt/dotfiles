# Review techniques

Ranked by value, from practice. Read this before forming findings. Not every PR needs all
seven; the index in `SKILL.md` says which are likely to pay on a given diff. Each technique
is illustrated with a real-world example, because the example is usually what makes the
technique recognisable in a new diff.

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
  catch and confirm it fails. Do it in the throwaway worktree - see "Execution sandbox" in `SKILL.md`
  for the protocol and for why editing the real source beats monkeypatching.

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
