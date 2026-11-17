# ADR-0005: Publish comment-only reviews; no approve, merge or push path

- Status: Accepted (planned; not yet implemented)
- Date: 2026-11-16

## Context

PR text and code comments are attacker-controlled. If the bot can approve, request changes, push or merge, a
successful prompt injection has real consequences.

## Decision

Every forge adapter publishes one review with event `COMMENT`. `ReviewDraft` has no field that can express approval.
No module contains an approve, request-changes, push or merge API call, and a test scans the package source for those
paths (SEC-03).

## Reasons

1. **Structural, not behavioural.** The worst outcome of an injection is a misleading comment, never an approval.
2. **Matches the scope.** The bot advises; humans decide (SPEC §1.3).

## Consequences

- The bot cannot block a merge. Teams that want blocking can add a required status check on their own; that is out
  of scope.
- A tainted run still publishes, with a warning banner.

## Alternatives considered

- **Request changes on error-severity findings.** Useful signal, but hands a lever to anyone who can write a PR body.
