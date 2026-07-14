# Contribution README — CodePath AI301 (Su26)

**Contributor:** Thy Tran (@ThyTran1402)
**Status:** Phase II Complete — fix implemented and PR open ([opensearch-project/k-NN#3419](https://github.com/opensearch-project/k-NN/pull/3419), awaiting review)

---

## Project & Issue

- **Project:** [opensearch-project/k-NN](https://github.com/opensearch-project/k-NN) — the OpenSearch plugin for approximate nearest-neighbor (vector) search, powering semantic search and RAG workloads.
- **Issue:** [#2415 — [BUG] Flaky test `WarmupIT.testKNNWarmupCustomLegacyFieldMapping`](https://github.com/opensearch-project/k-NN/issues/2415)
- **Labels:** `flaky-test`, `good first issue`
- **My fork:** https://github.com/ThyTran1402/k-NN (branch `fix/2415-flaky-warmup-custom-legacy`)

### Problem Summary

A backwards-compatibility (BWC) test, `WarmupIT.testKNNWarmupCustomLegacyFieldMapping`,
fails intermittently in CI and is currently muted with `@AwaitsFix` — so upgrade behavior
for indices using legacy field mappings is going untested. The failure cannot be reproduced
by re-running with the framework's own random seed (`tests.seed`), which suggests the
nondeterminism comes from outside the JVM test harness. The most recent change affecting
the test is [PR #2374](https://github.com/opensearch-project/k-NN/pull/2374), which switched
the test's legacy space type from `linf` to `innerproduct`.

**"Fixed" looks like:** the root cause is documented on the issue, the test is made
deterministic (or its assertions made appropriate for approximate search), the
`@AwaitsFix` annotation is removed, and the test passes reliably in the
`:qa:restart-upgrade:testRestartUpgrade` CI workflow.

### Why I Chose This Issue

- It's a `flaky-test` / `good first issue`: well-scoped (a single test plus possibly
  small test-fixture changes) and genuinely valued — muted tests silently erode the
  project's upgrade-safety coverage until someone fixes them.
- The issue has strong context to start from: an exact reproduction command with seed,
  three CI failure run links, and the suspect commit are all linked.
- I'm interested in AI/ML infrastructure, and this issue requires understanding how
  HNSW vector-search graphs are actually built (metric vs. non-metric spaces, build
  parameters, native-library randomness) — a chance to learn how vector databases work
  under the hood rather than just patching a symptom. I have Java/JUnit experience,
  so the test-framework side is a good fit.

### Issue Selection Checklist (Phase I, Step 5)

| Check | Result | Notes |
|---|---|---|
| 1. I understand the problem | Muted flaky BWC test; "fixed" = RCA documented, test deterministic, un-muted. |
| 2. Scope fits 3–4 weeks | One test, self-contained; no production code changes expected. |
| 3. Matches my skills | Java, JUnit, Gradle; HNSW/nmslib internals learnable with focused reading. |
| 4. Active and claimable | Active maintainers, no open PR solving it; assigned to me. |
| 5. Helpful context | Repro command + seed, three CI run links, suspect PR (#2374) linked in the issue. |
| 6. Clear setup documentation | Actively maintained `DEVELOPER_GUIDE.md` and `CONTRIBUTING.md` with per-OS instructions. |

**Score: 6/6 — claimed.**

### Known Risk (tracking)

The CI logs from the original January 2025 failures have expired and the issue contains
no stack trace, so the exact assertion that tripped is not confirmed. **My plan:** build
the evidence locally in Phase II — reproduce (or statistically bound) the flakiness by
looping the test scenario, confirm which assertion fails, then propose a fix and check
direction with maintainers before implementing (likely either realistic HNSW build
parameters for the inner-product test, or relaxing the exact top-k ordering assertion
that is too strict for approximate search).

---

## Phase II — Understand & Reproduce

### Understanding of the issue (root-cause analysis)

The investigation found **two** root causes — the second one was discovered while
tracing the test fixture's history and was not in the original issue at all.

**Root cause 1 — why the test flaked (the original #2415 report).**
[PR #2374](https://github.com/opensearch-project/k-NN/pull/2374) (commit `e2cd03e9`)
changed this test's legacy space type from `linf` to `innerproduct` while keeping the
*minimum* HNSW build parameters (`m=2`, `ef_construction=2`). That combination is
degenerate:

- The test's vectors are collinear with monotonically increasing norm (doc *i* is
  `[i, i, i, i, i]`). Under inner-product distance — a **non-metric** space — every
  point's nearest neighbor is the highest-norm point, so there is no local
  neighborhood structure for HNSW to exploit.
- With `m=2` (only ~4 level-0 links per node) and `ef_construction=2`, nmslib's
  neighbor-pruning heuristic collapses the graph into a hub topology where the
  reachable node set from the entry point is roughly `maxM0 + 1 = 5` nodes — exactly
  the `k=5` docs the test asserts, *in the best case*. Any perturbation drops or
  reorders one result and the exact top-k assertion in
  `KNNRestTestCase.validateKNNSearch` fails.
- The perturbation source is outside the JVM test framework, which is why re-running
  with the same `tests.seed` never reproduces it: nmslib's level RNG is a
  `static thread_local` seeded once per native thread, and its state advances across
  every graph any Lucene flush/merge thread builds; segment composition also varies
  with background-merge timing. Under the old `linf` space (a proper metric, where
  distance between doc *i* and doc *j* is `|i−j|`) the same heuristic builds a
  well-connected chain graph with perfect recall, which is why the test was stable
  for years before #2374.

**Root cause 2 — the test no longer tested anything (discovered during Phase II).**
While reproducing, I traced the fixture history with
`git log -S "KNNSettings.KNN_SPACE_TYPE" -- src/testFixtures/.../KNNRestTestCase.java`
and found that [PR #2564](https://github.com/opensearch-project/k-NN/pull/2564)
("3.0.0 Breaking Changes", March 2025) removed the legacy setting constants from
`KNNSettings` — and in doing so **emptied**
`createKNNIndexCustomLegacyFieldMappingIndexSettingsBuilder`: since then it returns
only shard/replica/`index.knn` settings and silently ignores its `spaceType`, `m`,
and `ef_construction` arguments. Every "custom legacy field mapping" BWC test has
been creating a plain default index ever since. So even un-muting the test wouldn't
have restored real legacy-mapping coverage — the fixture itself had to be repaired.

### Local setup notes

- Fork cloned as `origin` (`https://github.com/ThyTran1402/k-NN.git`) with
  `upstream` → `opensearch-project/k-NN`; working branch
  `fix/2415-flaky-warmup-custom-legacy` cut from `main`.
- JDK 21 (OpenJDK 21.0.9) on Windows 11; the Java/test-fixture side of the repo
  compiles cleanly and `spotless` checks run (`build/` and
  `qa/restart-upgrade/build/spotless-*` outputs present).
- Full BWC verification command (spins up a real 3-node old-version cluster, then
  restarts it on the new version):
  `./gradlew :qa:restart-upgrade:testRestartUpgrade --tests "org.opensearch.knn.bwc.WarmupIT.*" -Dtests.bwc.version=2.0.1`
  The restart-upgrade harness (native libs via CMake + multi-node clusters) is
  Linux-oriented, so the end-to-end loop runs are planned on WSL2/CI rather than
  native Windows — see "Remaining verification" below.

### Reproduction steps & evidence

The original January 2025 CI logs have expired and the issue has no stack trace, so
reproduction is **evidence-by-analysis** (code reading + git archaeology) rather than
a captured local failure. What was established:

1. **Timeline correlation:** flakiness reports began immediately after #2374 switched
   the space type to `innerproduct` (CI failures on PRs #2345, #2399, #2411; muted by
   [PR #2416](https://github.com/opensearch-project/k-NN/pull/2416), commit `f11bd9f6`).
2. **Mechanism:** the hub-topology analysis above explains both the intermittency and
   why `tests.seed` replays don't reproduce it (nondeterminism lives in nmslib's
   thread-local RNG and merge scheduling, not the JVM test framework).
3. **Fixture regression:** confirmed by diffing the builder before/after commit
   `e7d5304b` (PR #2564) — the parent version puts `index.knn.space_type`,
   `index.knn.algo_param.m`, and `index.knn.algo_param.ef_construction`; the current
   `main` version puts none of them.
4. **Why it can't flake with the fix:** with `m=50` / `ef_construction=1024` over 20
   vectors, the HNSW graph is effectively complete (every node links to every other),
   so recall is 1.0 regardless of RNG state — the exact top-k assertions become safe
   even in the non-metric inner-product space.

### Solution approach (implemented on the branch, commit `94dfed4a`)

Option A from the plan (realistic build parameters), **plus** the fixture repair that
root cause 2 forced:

| File | Change |
|---|---|
| `qa/restart-upgrade/.../WarmupIT.java` | Removed `@AwaitsFix`; `m`/`ef_construction` changed from min values (2/2) to suite constants `M=50` / `EF_CONSTRUCTION=1024`; comment documents why min values + inner product flake |
| `qa/restart-upgrade/.../IndexingIT.java` | Same parameter change for the sibling `testKNNIndexCustomLegacyFieldMapping` (same fragile pattern) |
| `src/testFixtures/.../KNNRestTestCase.java` | `createKNNIndexCustomLegacyFieldMappingIndexSettingsBuilder` again applies the legacy `space_type`/`m`/`ef_construction` settings (via new constants, since `KNNSettings` no longer has them) |
| `src/testFixtures/.../TestUtils.java` | New `LEGACY_KNN_SPACE_TYPE`, `LEGACY_KNN_ALGO_PARAM_M`, `LEGACY_KNN_ALGO_PARAM_EF_CONSTRUCTION` string constants (settings still accepted by 2.x clusters) |
| `CHANGELOG.md` | Maintenance entry referencing #2415 |

Why this shape: it is the smallest diff that (a) removes the degenerate
graph-topology configuration causing the flake, (b) restores the legacy-settings
coverage the test exists to provide, and (c) keeps the exact-ordering assertions
meaningful. Min-value parameter validation is already covered by non-BWC unit tests,
so nothing is lost there.

**Remaining verification before the PR is "done":** loop the un-muted test ≥30× on a
Linux environment (WSL2 or CI re-runs) across at least two `-Dtests.bwc.version`
values with zero failures, and run the full `WarmupIT` class end-to-end once. Local
runs so far cover compilation and formatting only — the BWC cluster harness has not
been executed on this Windows machine.

> This work has since been submitted upstream as
> [PR #3419](https://github.com/opensearch-project/k-NN/pull/3419) — see Phase IV
> for the PR summary and review log.

## Phase III — Implement & Test

- **Testing strategy:** _TODO_
- **Implementation notes:** _TODO_

## Phase IV — Pull Request

- **PR link:** [opensearch-project/k-NN#3419](https://github.com/opensearch-project/k-NN/pull/3419)
  (opened 2026-07-10; `Resolves #2415`; DCO signed-off; reviewer requested: @heemin32)
- **PR summary:** Re-enables the muted flaky BWC test
  `WarmupIT.testKNNWarmupCustomLegacyFieldMapping`: removes `@AwaitsFix`, replaces the
  degenerate minimum HNSW build parameters (`m=2`, `ef_construction=2`) with the suite
  constants (`M=50`, `EF_CONSTRUCTION=1024`) in `WarmupIT` and the sibling `IndexingIT`
  test, and repairs `createKNNIndexCustomLegacyFieldMappingIndexSettingsBuilder` so the
  legacy `index.knn.space_type` / `algo_param.m` / `algo_param.ef_construction` settings
  are actually applied again (they were silently dropped by the 3.0 breaking-changes
  PR #2564). Includes a CHANGELOG entry.
- **Maintainer feedback log:** _None yet — awaiting first review._
