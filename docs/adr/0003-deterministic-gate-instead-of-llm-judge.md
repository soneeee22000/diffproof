# ADR-0003: Gate agent output with deterministic checks, not an LLM judge

- Status: Accepted (planned; not yet implemented)
- Date: 2026-11-16

## Context

Something must decide which findings and tests reach the PR. A second model ("judge") is a common choice, but its
verdicts are themselves unverified and drift with the model.

## Decision

The gate is pure code: D1a unseen location, D1b unseen evidence, D2a injection text scan, D2b canary obedience, D3
tool allowlist, D4 schema, D5 budget, plus the 2×2 test verdict. Every detector has a mutant agent that must trip it,
and a harness check fails the build if any detector stops firing (GATE-08).

## Reasons

1. **Provable.** Each check has a precise definition and a unit test; each detector is proven to fire by a mutant.
2. **Cheap and replayable.** No extra model calls, so the gate ablation (EVAL-06) costs nothing.
3. **The gate is built before the agents (G2 before G3).** The oracle exists before the thing it judges.

## Consequences

- The gate cannot judge subtle quality ("is this comment insightful?"). That is measured, not gated: localization
  precision and false alarms in RESULTS.md.
- D2a is a heuristic; the structural controls (ADR-0005, ADR-0007) do not depend on it.

## Alternatives considered

- **An LLM judge.** Flexible, but unverifiable and non-deterministic; it would also need its own evaluation.
- **No gate.** Simplest; reproduces the failure modes in SPEC §1.1.
