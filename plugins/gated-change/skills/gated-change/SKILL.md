---
name: gated-change
description: Run the repository's governed issue-to-PR workflow with specialist agents, bounded retries, explicit human scope approval, independent validation, and human merge review.
---

# Gated Change

Use this skill when a user wants to take a real GitHub issue through the governed change workflow defined in `implementation-plan.md`.

## Workflow contract

1. Start from real GitHub issue context.
2. Run Intake Triage against the Definition of Ready.
3. If ready, run Architect to produce the technical + impact specification.
4. Stop at the human Scope Gate. No code changes before explicit approval. Approval is a two-part act: the human switches the session from Plan mode to Agent mode (the environment-enforced control — Developer's write tools stay inert in Plan mode no matter what is typed) AND explicitly confirms both the mode switch and approval in their reply. Do not delegate to Developer speculatively to test whether the mode switch happened — that wastes a full Developer turn on a foregone conclusion; wait for the human's confirming reply instead.
5. After approval, implement in the Copilot App session's isolated workspace/worktree through the Developer agent.
6. Developer writes the fix and regression tests.
7. QA independently validates the original acceptance criteria and final-diff scope compliance.
8. Reviewer performs independent read-only risk/quality review.
9. Open a pull request and let the repository's native CI run.
10. Stop at the human Merge Gate: human developer technical approval first, then PM business/scope approval.
11. Post-merge Release-helper behavior is added only when that planned stage is implemented and tested.

## Human-control rules

- Scope approval is a hard control boundary.
- Merge approval is a hard control boundary.
- A scope expansion always returns to the human Scope Gate.
- Architect may confirm that expansion is technically justified, but cannot authorize it.
- Reviewer findings inform human decisions and never automatically consume retry budget.

## Bounded-loop rules

- Intake clarification: <= 2 rounds.
- Scope negotiation: <= 2 cumulative rounds.
- Developer -> QA -> Reviewer implementation attempts: <= 3.
- Escalate rather than loop beyond a bound.

## Efficiency rules

- Carry structured outputs forward instead of asking later agents to re-discover prior-stage context.
- Prefer deterministic compute over LLM reasoning for baseline failure comparison, flaky reruns, infrastructure signature detection, and monorepo reference sweeps.
- Do not add an Impact Auditor agent; the design intentionally removed it.
- Do not spend extra reasoning on incidental observations merely to make them loggable.

## Current implementation boundary

The first App-native vertical slice covers:

Issue -> Intake -> Architect -> Scope Gate -> Developer -> QA -> Reviewer -> PR/CI -> Merge Gate.

The following are planned subsequent milestones and must not be represented as implemented until verified in the GitHub Copilot App:

- deterministic write-scope enforcement hook,
- machine-readable approval state,
- Canvas control surface,
- baseline/flaky/infra classifiers,
- deterministic cross-package sweep,
- Release-helper and post-merge auto-revert flow,
- token/cost instrumentation.
