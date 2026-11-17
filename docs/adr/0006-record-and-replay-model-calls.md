# ADR-0006: Record model calls once and replay them everywhere else

- Status: Accepted (planned; not yet implemented). Revised by amendment A1, 2026-11-17.
- Date: 2026-11-16

## Context

CI must run without an API key and produce the same results on every push. `claude-opus-5-5` accepts no sampling
parameters, so there is no `temperature` knob to make outputs repeatable.

The first version rejected every timestamp, UUID or absolute path anywhere in a request. Real source code, PR text and
test output contain all three, so that guard would have rejected legitimate repositories.

## Decision

Port faultline's transport layer. Requests are keyed by a hash of their canonical JSON, responses are stored as JSON
cassettes, and replay never constructs the SDK client. A replay miss fails closed (exit 3).

- **Volatile values:** the guard checks only harness-generated content (system prompts, tool definitions, wrapper text,
  metadata). Harness-generated tool output is normalized (paths, durations, ordering). Repository content is never
  scanned or rewritten; it is stable because the snapshot is frozen.
- **Ordering and ids:** fan-in results are put in canonical order, and every id is a content hash.
- **Sandbox runs** are recorded alongside model calls, so replay needs no Docker.
- **Provenance:** each recording session writes a manifest once (date, OS, Python, SDK version, sandbox image digest,
  corpus and prompt hashes). Regeneration never rewrites it.

## Reasons

1. **Reproducibility without a temperature setting.** Replay is the only route to byte-identical CI results.
2. **Keyless CI.** Pull requests from anyone can run the full suite and the benchmark replay.
3. **Free re-scoring.** Metric changes, the gate ablation and the baseline comparison re-use recorded outputs at zero
   cost.
4. **Proven pattern.** The same design runs in faultline-noc and cartridge.

## Consequences

- Any prompt change invalidates the affected cassettes; re-recording is a paid, logged step (gate spend ledgers).
- Cassettes are committed and scanned for secrets (SEC-08).
- Replay demonstrates deterministic orchestration and artifact generation. Re-executing recorded tests is a separate
  claim with its own CI job (`reexec`), whose runtime is measured before its timeout is set.

## Alternatives considered

- **HTTP-level recording (vcrpy and similar).** Works, but couples cassettes to transport details and headers.
- **Live calls in CI.** Needs a secret in CI, costs money per push, and is not reproducible.
- **Reject volatile values everywhere.** Simple, but rejects real repositories.
