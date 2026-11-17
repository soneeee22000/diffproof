# ADR-0002: Agents reach the repository only through an MCP server

- Status: Accepted (planned; not yet implemented)
- Date: 2026-11-16

## Context

Agents need to read the diff, read files, search code and run tests. Giving them direct filesystem access, or pasting
whole files into prompts, makes it impossible to know what an agent actually saw, and widens the attack surface for
path traversal and code execution.

## Decision

Expose exactly four tools (`get_diff`, `read_file`, `search_code`, `run_tests`) from an MCP server built with the
official MCP Python SDK. Agents call tools only through a bridge that converts MCP schemas into Anthropic tool
definitions and records every call and every returned line in a `RunTrace`. Even the diff placed in a prompt is a
traced `get_diff` result.

## Reasons

1. **One audited boundary.** Path guards, size caps and the sandbox hand-off live in one place (SEC-05, SEC-01).
2. **Provenance for free.** The `RunTrace` is what makes detector D1b ("cites code it never read") possible.
3. **Reusable.** Any MCP client (an IDE assistant, another agent) can attach to the same server.

## Consequences

- The bridge is ours even if the SDK offers MCP helpers, because the `RunTrace` must be recorded (SPEC §5.3).
- The tool list is pinned and tested; adding a tool is a contract change (SPEC §15).
- `run_tests` is the only tool that executes code, and it always goes through `SandboxPort`.

## Alternatives considered

- **Native function tools without MCP.** Simpler, but loses the reusable boundary.
- **Prompt stuffing (send the whole diff and files).** No provenance, large prompts, and no way to check citations.
