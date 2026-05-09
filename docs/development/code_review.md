# Code Review Standard

Use this standard for human review, Codex review, and Claude/Codex handoffs.

## Roles

- Codex is the primary coding and review agent for this repository.
- Minimax, Kimi, Claude Code, or a local Hermes runner may assist with implementation or review.
- Humans remain responsible for final approval and merge.

For higher-risk changes, prefer at least one second-pass review from Codex in review mode, Hermes,
or another auxiliary model. Do not let one AI agent write, review, and merge a production change
without human confirmation.

## Must Check

1. Correctness
   - Does the change solve the stated problem?
   - Are edge cases and error paths handled?
   - Does it preserve public APIs and blueprint behavior unless explicitly changed?

2. Robotics and runtime safety
   - Are hardware-facing paths guarded by the right config, replay, or simulation checks?
   - Could the change cause unsafe movement, runaway loops, missed stop signals, or stale commands?
   - Are module lifecycle, worker process, and transport failures handled cleanly?

3. Security and privacy
   - No committed secrets, tokens, robot credentials, or private IP assumptions.
   - No unsafe command execution, `eval`, pickle loading from untrusted input, or injection risk.
   - No unnecessary logging of PII, images, credentials, API keys, or hardware identifiers.

4. Tests
   - New behavior has tests where practical.
   - Bug fixes include regression tests when practical.
   - Tests are deterministic and avoid requiring hardware unless explicitly marked.
   - Slow, tool, or MuJoCo tests are marked correctly.

5. Maintainability
   - The patch is focused and matches existing DimOS patterns.
   - No unrelated refactors, formatting churn, or generated-file edits.
   - No new dependency without clear justification.

6. Performance and reliability
   - No obvious unbounded memory growth, busy loops, process leaks, or queue buildup.
   - Timeouts, retries, and cancellations are explicit where needed.
   - Stream and RPC interactions remain type-safe and lifecycle-safe.

## Severity

- P0: security vulnerability, data loss, unsafe robot behavior, secret leakage, auth bypass, broken production startup.
- P1: correctness bug, missing regression test for changed behavior, race condition, lifecycle leak, performance regression.
- P2: maintainability issue, unclear naming, incomplete docs, non-blocking test gap.

## Review Output Format

For each finding:

- Severity: P0 / P1 / P2
- File and line
- Problem
- Why it matters
- Suggested fix

Avoid nitpicks. Only report actionable issues introduced by the change.
