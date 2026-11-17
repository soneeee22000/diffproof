# ADR-0007: Run untrusted code in a locked-down Docker sandbox, from a trusted runner

- Status: Accepted (planned; not yet implemented). Revised by amendment A1, 2026-11-17.
- Date: 2026-11-16

## Context

Two kinds of untrusted code run during a review: tests written by the model (which may have been steered by injected
text) and the PR's own code, including any `conftest.py`, pytest configuration, plugins and build hooks. The process
that runs diffproof holds the API key. On Linux, a child process running as the same user can read its parent's
environment through `/proc/<pid>/environ`, so passing a clean `env` to a subprocess is not enough.

The first version specified the Docker flags but not which revision supplies the reviewer itself. The demo PR is opened
against diffproof's own repository; if the Action installed diffproof from the PR checkout, a PR could change the
reviewer code that runs next to the key, and the sandbox would not help.

## Decision

1. **Trusted runner.** diffproof runs from a pinned revision (`uses: owner/diffproof@<sha>`, installed from the
   Action's own path), never from the PR checkout. The PR is checked out with `persist-credentials: false` and treated
   as data.
2. **PR source is data outside the sandbox.** The key-holding process reads PR files and parses them with `ast.parse`.
   It never imports, installs or runs pytest on them.
3. **Everything executable runs in the sandbox.** In CI that is Docker with `--network none`, no host environment, a
   read-only root, a tmpfs work directory, CPU, memory and PID limits, a non-root user, `--cap-drop ALL`,
   `no-new-privileges` and a wall-clock timeout. No Docker socket, host credentials or `.git` metadata is mounted;
   trees are materialized with `git archive`.
4. **One snapshot.** Diff, base tree, head tree and comment coordinates come from one frozen merge-base / head SHA pair,
   and a moved head is detected before publishing.

A local subprocess sandbox exists for trusted runs on machines without Docker, and it refuses to run in CI.

## Reasons

1. **The key cannot leak.** No env, no network and a separate PID namespace close the exfiltration paths, and no PR
   code runs in the process that holds the key.
2. **Same-repo plus label is a trigger policy, not a trust boundary.** It limits who can spend the key, not what a PR's
   files can do.
3. **Bounded damage.** Resource limits and timeouts stop runaway tests.
4. **Tested.** The isolation properties are tests (SPEC §8 I8.1–I8.5) and a YAML policy test, not assumptions.

## Consequences

- Docker-dependent tests run on ubuntu CI only.
- The sandbox image carries the target project's pinned test dependencies from the trusted lockfile; PRs that change
  dependencies get no test verdicts.
- With a `pull_request` trigger the workflow file comes from the PR, so a same-repository collaborator could edit it.
  Accepted and documented: such a PR already needs write access.
- GitLab CI needs either a `docker:dind` service or split jobs; decided at G0-T6 (amendment A2).
- Each sandbox run adds container start-up time; `FLAKE_RUNS = 3` keeps the count bounded.

## Alternatives considered

- **Subprocess with a scrubbed env.** Leaks through `/proc` on Linux.
- **Install the PR's diffproof and trust the same-repo condition.** Lets a PR replace the reviewer next to the key.
- **`pull_request_target`.** Runs the base workflow with secrets but makes it easy to check out and run PR code with
  them; the pinned `uses:` reference gets the benefit without that risk.
- **Separate CI jobs per phase.** Isolates secrets well, but breaks the in-run repair loop.
- **gVisor or Firecracker.** Stronger isolation, but heavier than this threat model needs.
