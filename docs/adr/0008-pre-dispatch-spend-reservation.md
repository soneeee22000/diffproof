# ADR-0008: Reserve spend before dispatch, in a persisted ledger

- Status: Accepted (planned; not yet implemented). Added by amendment A1, 2026-11-17.
- Date: 2026-11-17

## Context

The first version stopped making model calls after reported usage passed a cap. That is not a hard cap: several
fan-out calls can start against the same remaining balance, and one request can overshoot before its response
arrives. The $30 project cap also has to hold across separate recording processes and restarts.

## Decision

A `SpendLedger` admits every request before it is sent:

1. Reserve the request's maximum possible charge: the canonical request's UTF-8 byte length plus a measured overhead,
   priced at the highest input rate (cache write), plus `max_tokens` at the output rate.
2. Admit under one lock against the run cap, the session cap and the remaining project cap; refuse to send otherwise.
3. Share one ledger across every agent, schema retry and repair round.
4. Settle to the actual cost from `usage` after the response; keep the full reservation when a request is interrupted.
5. Persist the project total in a gitignored JSONL file under a file lock, read at the start of each recording process.

## Reasons

1. **A cap that is actually hard.** Nothing is sent unless its worst case fits.
2. **Cheap to build and test.** About forty lines plus tests; the `RunsPastBudget` mutant proves no request goes out
   past the cap.
3. **Conservative where uncertain.** Interrupted requests and cache pricing are charged at their worst case.

## Consequences

- Every request must carry an explicit `max_tokens`; `MODEL_PROFILES` sets it per agent.
- Reservations are pessimistic, so a run may stop slightly before its true budget is used.
- The ledger bounds what diffproof sends; it is not the provider's billing record. Gate sheets reconcile it against the
  Console usage page after each paid run.

## Alternatives considered

- **Check usage after each response.** Simple, but overshoots under parallel calls.
- **Call a token-counting endpoint before each request.** Tighter bound, but one more network dependency per call and
  awkward in replay.
- **Label it a best-effort guard.** Honest, but weaker than a reservation that costs little to build.
