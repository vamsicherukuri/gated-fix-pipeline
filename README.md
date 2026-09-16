# Gated Fix Pipeline

Narrow-slice implementation for the GitHub Copilot App Enterprise Challenge submission. This is Steps
01 through 03 of the eleven-step design in [implementation-plan.md](implementation-plan.md) — Issue
filed → Intake Triage → Plan drafted → Scope gate — built and type-checked against the real
[`@github/copilot-sdk`](https://github.com/github/copilot-sdk), not a mock.

See [implementation-plan.md](implementation-plan.md) for the full design rationale and
[the workflow diagram artifact](https://claude.ai/artifact/P99xeczGaacExSXJp7mVhp) for the visual map
of all eleven steps, agent roles, and decision playbook.

## What's real here vs. what's a stand-in

**Real, working, type-checked against the installed SDK:**
- `.agent.md` custom agent definitions ([.github/agents/](.github/agents/)) using the documented frontmatter
  schema — this location is required for GitHub Copilot's cloud agent / GHCP App to discover them (see
  "Testing via the GitHub Copilot App" below).
- The orchestrator ([src/orchestrator.ts](src/orchestrator.ts)) drives one `CopilotClient` session per
  agent stage — a deliberate choice, not a limitation: it keeps each stage's tool access cleanly
  separable for the permissions/audit story in implementation-plan.md §7, rather than one shared
  session where scoping is harder to prove.
- The scope-enforcement tool ([src/scopeTool.ts](src/scopeTool.ts)) — a real, working example of the
  "enforced boundary, not a prompt instruction" pattern from §5b/§6. It blocks reads outside the
  declared scope at the tool layer.
- The bounded round caps (`INTAKE_TRIAGE_MAX_ROUNDS`, `SCOPE_GATE_MAX_ROUNDS` in
  [src/types.ts](src/types.ts)) — enforced in the orchestrator's control flow, not left to the model.
- The Scope gate ([src/scopeGate.ts](src/scopeGate.ts)) — a genuine blocking `stdin` prompt. Nothing
  proceeds past it without a real keypress.

**Stand-ins, called out explicitly rather than silently faked:**
- The Scope gate's approval UI is a CLI prompt here. In the real GHCP App surface this would be
  whatever native approval UI the platform provides — the important property (execution actually
  pauses on a human) is preserved either way.
- Intake Triage's clarification loop simulates the "post a comment, wait for a reply" round-trip with
  a single in-process re-prompt (see the comment in `runIntakeTriage`) — the routing decision itself
  (GitHub issue comments vs. a PM prompt) is still explicitly deferred per implementation-plan.md §4.
- Architect's scope-bounded search only enforces the declared-scope boundary; the "one hop of
  importers/callers" search described in its system prompt is not yet implemented as a real grep —
  the agent is instructed to do this itself via its own reasoning for now.
- Steps 04–10 (branch, implementation, tests, review, merge, release) are fully designed in
  implementation-plan.md §5b–§5h but not yet built here.

## Roles

| Role | Responsibility | Where |
|---|---|---|
| Reporter (non-dev) | Files the issue in plain language | outside this repo (GitHub issue) |
| Intake Triage (agent) | Checks the issue is ready and scoped before anything expensive runs | [.github/agents/intake-triage.agent.md](.github/agents/intake-triage.agent.md) |
| Architect (agent) | Drafts a technical + impact spec, no code | [.github/agents/architect.agent.md](.github/agents/architect.agent.md) |
| PM (non-dev, human) | Approves, requests revision, or sends back the plan at the Scope gate | via `askScopeGate` in [src/scopeGate.ts](src/scopeGate.ts) |

## Prerequisites

- Node.js 22+ and npm.
- A GitHub Copilot subscription (or BYOK provider config — see the SDK's
  [auth docs](https://github.com/github/copilot-sdk/blob/main/docs/auth/README.md)). Without this, the
  orchestrator will fail at `client.start()` / the first `createSession` call.

## Setup

```bash
npm install
npm run build   # type-checks the orchestrator against the installed SDK
```

## Running it

```bash
npm run run:ready   # a well-formed, scoped issue - should sail through Intake Triage in round 1
npm run run:vague   # a vague issue - demonstrates the bounded clarification loop
```

Both commands read a JSON issue fixture from [examples/](examples/) and drive it through Steps 01–03,
pausing at the Scope gate for a real approve / revise / send-back response on stdin.

## Governance considerations

- No agent has write access anywhere in this slice — Architect is read-only, bounded to the declared
  scope, and Intake Triage never touches the repo at all. Write-scope enforcement for the Developer
  agent (§5b) is designed but arrives with Step 05.
- The Scope gate is a genuine hard block: `askScopeGate` blocks on `stdin`, and the orchestrator does
  not proceed past it without an explicit decision.
- Bounded rounds (Intake Triage ≤2, Scope gate ≤2) are enforced in code (`orchestrator.ts`), not by
  asking the model to self-limit.

## Human-in-the-loop model

Two roles, one gate in this slice: the **reporter** files the issue, and the **PM** is the one required
approver at the Scope gate. The full eleven-step design adds a second gate (Dev + PM at the Merge gate,
§5g) once Steps 04–10 are built.

## Success measures

- Intake Triage correctly distinguishes a ready issue (`examples/sample-issue-ready.json`, should pass
  round 1) from a vague one (`examples/sample-issue-vague.json`, should trigger the clarification loop
  and eventually escalate if never clarified).
- The scope tool actually blocks an out-of-scope read — this is checked by `isWithinScope` in
  [src/scopeTool.ts](src/scopeTool.ts) and can be unit-tested directly without any SDK/network access.
- The Scope gate genuinely pauses for human input and honors all three PM actions (approve, revise,
  send back) with the correct downstream behavior for each.
