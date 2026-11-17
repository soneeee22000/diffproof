# ADR-0004: Judge generated tests by running them on base and on head

- Status: Accepted (planned; not yet implemented). Revised by amendment A1, 2026-11-17.
- Date: 2026-11-16

## Context

A generated test that passes proves little: it may not exercise the change at all. Coverage gain is one filter
(TestGen-LLM, Cover-Agent), but it still accepts tests that cannot tell correct code from incorrect code.

The first version of this decision treated a syntax error, import error or timeout as an ordinary failure. That let a
test that cannot be collected on head become a "regression", and a test that cannot import on base become "pins the
change". Those are execution failures, not statements about behaviour.

## Decision

Run every generated test in the sandbox on the base tree and on the head tree, three times each. Collect a per-test
outcome from pytest's JUnit XML report (`PASSED`, `FAILED`, `COLLECTION_ERROR`, `SETUP_ERROR`, `TIMEOUT`,
`NONE_COLLECTED`, `SKIPPED`); the exit code is only a cross-check. Classify:

| Base \ Head      | PASSED             | FAILED               | COLLECTION_ERROR  |
| ---------------- | ------------------ | -------------------- | ----------------- |
| PASSED           | NON_DISCRIMINATING | REGRESSION_CANDIDATE | HEAD_IMPORT_ERROR |
| FAILED           | PINS_CHANGE        | INVALID              | INVALID           |
| COLLECTION_ERROR | HEAD_ONLY          | INVALID              | INVALID           |

Any setup error, timeout, empty collection or skip is INVALID. Any disagreement across runs is FLAKY. INVALID tests get
one repair round. Every verdict keeps its run outcomes so the evidence can be inspected.

## Reasons

1. **Behavioural verdicts only when the test ran.** A behavioural label needs the test to execute on both trees.
   Import breakage on head is reported separately, never as a regression.
2. **Honest wording.** Base-pass / head-fail shows a behavioural difference under the generated assertion. An intended
   breaking change looks the same, so outside the seeded benchmark it is a _candidate_. Base-pass / head-pass shows
   only that the test did not distinguish the revisions on its inputs.
3. **One rule, two modes.** On a feature PR, PINS_CHANGE (or HEAD_ONLY for a new symbol) is the useful test. On a
   benchmark PR with a seeded bug, REGRESSION_CANDIDATE is a kill.
4. **Borrows the strongest idea from mutation-guided generation.** A test counts when it distinguishes the faulty
   version.
5. **Flakiness is caught by construction.** The repeated-run check from TestGen-LLM.

## Consequences

- Each test costs 6 sandbox runs (2 trees × 3); `TEST_TIMEOUT_S` bounds the time.
- The base tree must be materialised (`git archive` of the merge base in CI, directories in the benchmark).
- Sandbox runs are recorded so replay needs no Docker; a separate CI job re-executes them (ADR-0006).

## Alternatives considered

- **Keep tests that pass on head.** Accepts tautological tests (the `WritesTautologicalTest` mutant shows why).
- **Coverage gain.** Useful, but blind to whether the assertions check anything.
- **Exit status only.** Cannot tell an assertion failure from a collection error or an empty run.
