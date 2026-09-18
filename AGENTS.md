# AGENTS.md — Gated Change Challenge Project

## Project objective

Build the Gated Change workflow as a **GitHub Copilot App-native workflow** for the GitHub Copilot App Enterprise Challenge.

The user-facing execution experience must run inside the GitHub Copilot App. Do not turn the SDK prototype or a terminal/CLI orchestrator into the primary product experience.

The reusable distribution mechanism is a **GitHub Copilot plugin** containing custom agents, skills, hooks/policies, and later the Canvas extension.

## Source of truth

Read `implementation-plan.md` before making architectural changes.

`implementation-plan.md` is the authoritative workflow/design specification. Preserve its semantics unless the user explicitly changes a decision.

Do not silently simplify, merge, or remove roles, gates, retry rules, permission boundaries, or cost-control mechanisms from the plan.

## Scenario constraint

The decided challenge scenario is a generic software-delivery bug fix. Do not reintroduce migration/GHEC-specific scenarios.

## Required functional lifecycle

1. GitHub issue filed by a non-developer/support/PM.
1.5. Intake Triage.
2. Architect drafts technical + impact plan.
3. Human Scope Gate — hard block before implementation.
4. Native isolated branch/worktree after scope approval.
5. Developer implements fix and regression tests.
6. QA validates against the original acceptance criteria.
7. Reviewer performs independent risk/quality review.
8. Pull request opens and repository-native CI runs.
9. Human Merge Gate — Dev technical approval, then PM business/scope approval.
10. Release-helper handles the planned post-merge monitoring/triage behavior.

The Developer -> QA -> Reviewer loop is bounded to at most 3 implementation attempts.

## Agent boundaries

### Intake Triage
- Issue text/metadata only.
- Must not inspect the repository.
- Owns completeness, Definition of Ready, and declared-scope check.
- Maximum 2 clarification rounds.

### Architect
- Read/search only.
- Produces technical + impact spec, not code.
- Blast-radius analysis is bounded to declared scope + one hop of direct callers/importers.
- May confirm that scope expansion is technically required, but may never approve it.

### Developer
- Only code-writing agent.
- Receives the approved structured plan and original acceptance criteria.
- Writes implementation and regression tests.
- Must never silently modify outside approved scope.
- Legitimate scope expansion returns through Architect confirmation and the human Scope Gate.

### QA
- Read + execute tests.
- Must not write source or test code.
- Validates against original acceptance criteria.
- Re-verifies final-diff scope compliance.

### Reviewer
- Read-only.
- Reviews final diff plus any deterministic cross-package hits.
- Does not re-run QA tests.
- Does not fix code.
- Does not autonomously trigger retries.
- Flags are informational for the human Merge Gate.

### Release-helper
- Reads CI and performs only the recovery actions explicitly defined in `implementation-plan.md`.
- Must not write product source code.

## Human gates

There are two hard human control boundaries:

1. Scope Gate before code is written.
2. Merge Gate before merge.

Do not replace these with prompt-only approval language.

Scope approval must ultimately be explicit and machine-readable so deterministic enforcement can consume it.

## Scope enforcement

Branch/worktree isolation is not scope enforcement.

Every Developer write must eventually be protected by deterministic policy/hook enforcement against the approved scope. Canonicalize paths before comparison and protect approval state itself from agent edits.

## Efficiency principles

- Gate expensive work with cheaper stages.
- Intake clarification <=2 rounds.
- Scope negotiation <=2 cumulative rounds.
- Developer/QA/Reviewer loop <=3 attempts.
- Prefer deterministic compute over LLM reasoning for baseline comparison, flaky reruns, infra signature checks, and monorepo reference sweeps.
- Do not add an Impact Auditor agent.
- Do not investigate incidental observations if doing so requires extra LLM reasoning.

## Implementation direction

Prefer GitHub Copilot App-native primitives for the user experience:

- real GitHub issue context,
- App agent sessions,
- Plan mode / explicit human approval,
- custom agents,
- isolated worktrees/branches,
- plugin packaging,
- policy hooks for deterministic enforcement,
- Canvas for the human control surface after the vertical slice works,
- native pull request and CI experience.

Keep `src/` SDK code as prototype/test-harness material unless the user explicitly changes this direction.

## Implementation order

1. Plugin loads in GitHub Copilot App.
2. Real issue -> Intake -> Architect -> explicit scope approval.
3. Isolated implementation -> Developer.
4. Separate QA.
5. Separate Reviewer.
6. PR + native CI + human Merge Gate.
7. Deterministic scope-enforcement hook.
8. Canvas-based control surface.
9. Deterministic classifiers, sweep, retry/escalation mechanics.
10. Release-helper/post-merge flow.
11. Token/cost instrumentation and challenge polish.

At each phase, verify behavior in the actual GitHub Copilot App before proceeding.

## When uncertain

Do not invent GitHub Copilot App/plugin behavior. Verify current official GitHub documentation or surface the uncertainty.

Do not change a decision from `implementation-plan.md` merely to make implementation easier. Propose an explicit amendment instead.
