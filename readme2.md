# Contribution README — CodePath AI301 (Su26)

**Contributor:** Thy Tran (@ThyTran1402)
**Status:** Phase I Complete

---

## Project & Issue

- **Project:** [opensearch-project/k-NN](https://github.com/opensearch-project/k-NN) — the OpenSearch plugin for approximate nearest-neighbor (vector) search, powering semantic search and RAG workloads.
- **Issue:** [#2415 — [BUG] Flaky test `WarmupIT.testKNNWarmupCustomLegacyFieldMapping`](https://github.com/opensearch-project/k-NN/issues/2415)
- **Labels:** `flaky-test`, `good first issue`
- **My fork:** https://github.com/ThyTran1402/k-NN  <!-- update after forking -->

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

- **Understanding of the issue:** _TODO_
- **Local setup notes:** _TODO_ (clone fork, JDK 21, `./gradlew :qa:restart-upgrade:testRestartUpgrade --tests "org.opensearch.knn.bwc.WarmupIT.*" -Dtests.bwc.version=2.0.1`)
- **Reproduction steps & evidence:** _TODO_
- **Solution approach:** _TODO_

## Phase III — Implement & Test

- **Testing strategy:** _TODO_
- **Implementation notes:** _TODO_

## Phase IV — Pull Request

- **PR link:** _TODO_
- **PR summary:** _TODO_
- **Maintainer feedback log:** _TODO_
