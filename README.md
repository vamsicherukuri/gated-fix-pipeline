# Gated Change — GitHub Copilot App Enterprise Challenge

This repository implements the workflow designed in [`implementation-plan.md`](implementation-plan.md) as a **GitHub Copilot App-native, governed issue-to-PR pattern**.

The project began as a narrow SDK proof of concept called **Gated Fix Pipeline**. That prototype is intentionally retained under `.github/agents/` and `src/` because it contains useful working examples of bounded control flow and deterministic scope checks. The **customer-facing challenge implementation now lives under `plugins/gated-change/`** and is intended to run inside the GitHub Copilot App.

## Source of truth

[`implementation-plan.md`](implementation-plan.md) is the authoritative workflow/design specification.

It defines the eleven functional stages, two hard human gates, bounded retry/clarification loops, agent permission boundaries, monorepo handling, deterministic-vs-LLM decisions, and token-efficiency principles. The plugin implementation must follow that document rather than silently simplifying the design.

## Target workflow

```text
GitHub Issue
    |
    v
Intake Triage
(issue context only)
    |
    v
Architect
(read/search, no writes)
    |
    v
HUMAN SCOPE GATE
    |
    v
Isolated App session / worktree
    |
    v
Developer
(fix + regression tests)
    |
    v
QA
(validate original acceptance criteria)
    |
    v
Reviewer
(read-only risk/quality flags)
    |
    v
GitHub PR + native CI
    |
    v
HUMAN MERGE GATE
Dev technical review -> PM business/scope review
    |
    v
Release-helper / post-merge handling
(planned later milestone)
```

## Repository layout

### App-native challenge implementation

```text
.github/plugin/marketplace.json
plugins/gated-change/
  plugin.json
  com.github.copilot/agents/
    gated-change-controller.agent.md
    gated-change-intake.agent.md
    gated-change-architect.agent.md
    gated-change-developer.agent.md
    gated-change-qa.agent.md
    gated-change-reviewer.agent.md
  skills/gated-change/SKILL.md
```

The plugin is the reusable field/customer pattern. It packages the workflow guidance and specialist agents for use in the GitHub Copilot App.

### Existing SDK prototype / test harness

```text
.github/agents/
  intake-triage.agent.md
  architect.agent.md
src/
  copilotAgent.ts
  loadAgent.ts
  orchestrator.ts
  scopeGate.ts
  scopeTool.ts
  types.ts
examples/
```

This code is **not being discarded**. It remains useful for validating deterministic controls, bounded loops, issue fixtures, and SDK behavior. It should not become the final user-facing workflow unless the design is explicitly changed.

## Agent responsibilities

| Role | Access | Responsibility | Must not do |
|---|---|---|---|
| Intake Triage | Issue context only | Definition of Ready + declared-scope check | Inspect repo; invent requirements |
| Architect | Read/search | Root cause, ADD/MODIFY/DELETE plan, one-hop blast radius, risk | Write code; approve scope expansion |
| Developer | Read/write approved scope + execute | Implement fix and regression tests | Silently broaden scope |
| QA | Read + execute tests | Validate original acceptance criteria and final-diff scope | Write source/test code |
| Reviewer | Read-only | Risk/quality flags for human Merge Gate | Fix code; rerun QA; auto-trigger retries |
| Release-helper | CI/recovery access | Planned pre/post-merge triage behavior | Write product source |

## Human-in-the-loop model

There are two hard blocking gates:

1. **Scope Gate** — code implementation cannot begin until the human approves the technical/impact plan and scope.
2. **Merge Gate** — merge requires sequential human review: developer technical approval first, then PM business/scope approval.

Scope expansion can never be approved by agents alone. Architect may confirm a real plan gap, but an expanded scope must return to the human Scope Gate.

## Bounded execution

The design intentionally avoids open-ended agent loops:

- Intake clarification: maximum **2 rounds**.
- Scope negotiation: maximum **2 cumulative rounds**.
- Developer -> QA -> Reviewer implementation loop: maximum **3 attempts**.

When a bound is reached, the workflow escalates instead of continuing to spend tokens.

## Deterministic controls and efficiency

The design prefers normal compute over LLM reasoning whenever the task does not require judgment:

- write-scope enforcement,
- baseline comparison for failing tests,
- one-time flaky-test reruns,
- known infrastructure-failure signature matching,
- cross-package reference sweep after the diff is finalized.

A dedicated Impact Auditor agent was intentionally removed from the design because the remaining cross-package detection problem is primarily deterministic search, not open-ended agent reasoning.

## Current implementation milestone

The first App-native milestone is intentionally narrow:

```text
real issue
-> Intake
-> Architect
-> human Scope Gate
-> Developer
-> QA
-> Reviewer
-> PR / native CI
-> human Merge Gate
```

The following are later milestones and should not be presented as implemented until they are verified in the GitHub Copilot App:

- deterministic `preToolUse` write-scope enforcement,
- machine-readable approval state,
- Gated Change Canvas,
- baseline/flaky/infra failure classifiers,
- deterministic cross-package sweep,
- Release-helper and post-merge auto-revert flow,
- per-stage token/cost instrumentation.

## Testing the first App-native slice

1. Use the `copilot-app-plugin-alignment` branch while this migration is being validated.
2. In the GitHub Copilot App, add this repository as a custom plugin marketplace.
3. Install `gated-change`.
4. Open a real GitHub issue from the App and start a **Plan** session.
5. Select the Gated Change controller using the custom-agent picker. In Copilot CLI, its qualified identifier is `gated-change:gated-change-controller`.
6. Ask it to run the Gated Change workflow for the issue.
7. Confirm Intake runs without repository access.
8. Confirm Architect plans without writing code.
9. Confirm no Developer work starts before explicit plan/scope approval.
10. After approval, confirm Developer -> QA -> Reviewer handoff order.
11. Create a PR in the App, allow native CI to run, and stop at the human Merge Gate.

Do not move to Canvas, scope-hook, or post-merge development until this vertical slice behaves predictably in the actual App.

## Governance / development guidance

- [`AGENTS.md`](AGENTS.md) contains durable instructions for Codex/automation working on this repository.
- [`CODEX-HANDOFF.md`](CODEX-HANDOFF.md) describes the current implementation state and exact next milestones.
- [`implementation-plan.md`](implementation-plan.md) remains authoritative for workflow behavior and design rationale.
