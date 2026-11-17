# ADR-0001: Orchestrate with LangGraph, and keep routing deterministic

- Status: Accepted (planned; not yet implemented)
- Date: 2026-11-16

## Context

The review needs several cooperating agents (reviewer per file, test-writer, docs agent), a gate between them and the
PR, and a bounded repair loop. That is a graph with fan-out, fan-in and one conditional cycle. The routing decision
(which agents run on this diff) can be computed from the parsed diff alone.

## Decision

Build the pipeline as a LangGraph `StateGraph` with typed state. Use conditional edges and `Send` for fan-out. Every
routing function is a pure function of the parsed diff; no model call decides control flow. Agents are nodes whose
internal tool loops are bounded (`MAX_TOOL_TURNS`).

## Reasons

1. **Extend a framework, not a toy loop.** LangGraph already provides typed state, reducers for fan-in, `Send` for
   map-reduce and checkpointing. Writing these again would add code without adding insight.
2. **LLM for judgment, code for control flow.** Deterministic routing is testable with plain unit tests, and the same
   diff always runs the same agents (SPEC I4.1).
3. **Bounded behaviour.** The repair loop's round counter lives in state, so the graph cannot cycle (FR-10).

## Consequences

- Routing rules are documented in SPEC §4.3 and tested in `tests/test_routing.py`.
- The way agents produce typed output (structured outputs, or a strict `submit_result` tool) is decided at G0-T4 and
  recorded here as an amendment. `submit_result` would be a local finalization tool, separate from the four MCP
  repository tools (SPEC §5.4, amendment A1).
- The LangGraph version is pinned exactly; an upgrade is a deliberate change with its own test run.

## Alternatives considered

- **A hand-written loop.** Less dependency weight, but fan-out, fan-in and state handling would be re-implemented and
  less recognisable to readers.
- **An LLM supervisor that picks agents.** More "agentic", but non-deterministic routing would make replay and testing
  harder for no gain on this task.
- **A managed agent platform.** Hides the orchestration this repository exists to show.
