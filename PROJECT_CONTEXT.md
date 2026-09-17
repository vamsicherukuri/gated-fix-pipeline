# Project Context — Gated Change / GitHub Copilot App Enterprise Challenge

## Purpose

This file is the model-neutral handoff for any coding assistant used in VS Code or another IDE. Read this file together with `implementation-plan.md`, which remains the authoritative design specification.

## Challenge objective

Build a reusable, enterprise-ready workflow that runs **inside the GitHub Copilot App** and demonstrates:

- GitHub-native context
- multi-agent collaboration
- connected tooling where justified
- governance and deterministic policy enforcement
- explicit human review gates
- repeatability as a field/customer pattern

The workflow is not intended to be a standalone SDK/CLI application. The existing TypeScript SDK code remains useful as a prototype/test harness, but the customer-facing execution surface is the GitHub Copilot App.

## Working name

**Gated Change**

A governed issue-to-PR software-delivery workflow where agents can investigate, plan, implement, test, and review, while humans retain control over scope and merge decisions.

## Source of truth

Read these in order:

1. `PROJECT_CONTEXT.md` — current state and orientation
2. `implementation-plan.md` — authoritative workflow and design decisions
3. `AGENTS.md` — durable instructions for coding agents
4. `CODEX-HANDOFF.md` — implementation handoff / milestone guidance
5. `README.md` — repository usage and architecture summary

Do not silently override decisions in `implementation-plan.md` to make implementation easier.

## Scenario decision

The challenge scenario is a **generic software-delivery bug fix**.

Do not reintroduce migration/GHEC-specific scenarios into this project.

## Required workflow

Preserve this lifecycle:

1. GitHub issue filed by non-developer/support/PM
1.5. Intake Triage
2. Architect technical + impact plan
3. Human Scope Gate — hard block before source changes
4. Native isolated worktree/branch
5. Developer implements fix + regression tests
6. QA validates against original acceptance criteria
7. Reviewer independently assesses final diff / risk / quality
8. Pull request opens; native repository CI runs
9. Human Merge Gate — developer technical approval, then PM/business approval
10. Release-helper handles post-merge triage/recovery per `implementation-plan.md`

## Agent boundaries

### Intake Triage

- issue text / metadata only
- no repository access
- checks Definition of Ready
- requires reproduction or expected-vs-actual, acceptance criteria, and declared scope
- maximum 2 clarification rounds

### Architect

- read/search repository only
- no code writes
- creates technical + impact spec
- bounded to declared scope + one hop of callers/importers
- may identify that scope expansion is technically required
- may never approve scope expansion

### Developer

- only code-writing agent
- receives the approved structured plan instead of re-solving the raw issue
- writes fix and regression tests together
- must remain inside approved scope
- any legitimate scope expansion must route back through Architect confirmation and the human Scope Gate

### QA

- read + execute tests
- no source edits
- validates against original acceptance criteria
- re-checks final diff scope compliance
- deterministic baseline/flaky classification should happen before expensive reasoning where practical

### Reviewer

- read-only
- reviews final diff and any deterministic cross-package sweep hits
- does not re-run tests
- does not fix code
- does not autonomously consume retry budget
- findings feed the human Merge Gate

### Release-helper

- CI/post-merge triage role defined later in `implementation-plan.md`
- does not write product source code

## Human control boundaries

There are two hard gates:

1. **Scope Gate** before implementation
2. **Merge Gate** before merge

Do not replace either gate with prompt-only language such as “ask before proceeding.” The final design should use explicit machine-readable approval state.

## Bounded-loop rules

- Intake clarification: max 2 rounds
- Scope negotiation/revision: max 2 cumulative rounds
- Developer → QA → Reviewer implementation attempts: max 3
- escalate rather than loop indefinitely

Do not create additional retry counters unless the design is explicitly amended.

## Efficiency principles

Prefer deterministic compute over LLM reasoning when possible.

Examples from the design:

- baseline comparison for failing tests
- one-time flaky rerun
- infra-signature matching
- deterministic cross-package symbol/reference sweep
- scope path checks

The project intentionally removed a separate Impact Auditor agent because a deterministic sweep is cheaper and less drift-prone.

## Repository architecture

### App-native implementation

`plugins/gated-change/`

Contains the GitHub Copilot plugin package and current custom-agent definitions.

Current plugin agents:

- `gated-change-controller`
- `gated-change-intake`
- `gated-change-architect`
- `gated-change-developer`
- `gated-change-qa`
- `gated-change-reviewer`

### Existing prototype / harness

`src/`

Contains the earlier Copilot SDK implementation. Preserve it as a prototype/regression harness unless explicitly migrated later.

Notable existing pieces:

- SDK orchestration
- CLI scope gate prototype
- scope-checking logic
- bounded Intake / Scope loops
- JSON issue fixtures

Do not make this terminal/SDK experience the final customer-facing workflow.

## Current working branch

`copilot-app-plugin-alignment`

Draft PR: #2

This branch adds:

- `.github/plugin/marketplace.json`
- `plugins/gated-change/plugin.json`
- plugin custom agents
- Gated Change skill
- `AGENTS.md`
- `CODEX-HANDOFF.md`
- revised README

## Current validation status

### What is working

- custom marketplace is registered
- `gated-change` plugin is listed as installed
- installed plugin cache contains:
  - `plugin.json`
  - `skills/gated-change/SKILL.md`
  - `com.github.copilot/agents/*.agent.md`
- GitHub Copilot CLI version tested: `1.0.86-2`
- repository-level `.github/agents/architect.agent.md` and `intake-triage.agent.md` are discovered by `/agent`

### Current blocker

The installed plugin agents are physically present under the Copilot plugin cache, but `/agent` does **not** show `gated-change-controller`.

Observed `/agent` output currently shows only:

- Default
- architect · project
- intake-triage · project

The specialist plugin agents are intentionally non-user-invocable, but `gated-change-controller` is configured `user-invocable: true` and should be discoverable if plugin-agent loading is working as expected.

### Separate warnings

The old repository-level agents emit warnings like:

`unknown field ignored: metadata`

Treat this as a compatibility/cleanup issue separate from plugin discovery.

### Plugin cache / Windows issue encountered

Uninstall/reinstall of the marketplace plugin produced:

`Access is denied. (os error 5)`

This appears to be a Windows file-lock/permission issue around `~/.copilot/installed-plugins` and is separate from the workflow design.

A direct local-path `copilot plugin install .\plugins\gated-change` attempt was rejected because this CLI expects plugin specs in supported marketplace/repository/URL forms rather than that local-directory syntax.

## Immediate debugging objective

Do **not** continue building Canvas, hooks, post-merge recovery, or telemetry until plugin-agent discovery is understood.

Debug in this order:

1. Verify `gated-change-controller.agent.md` is installed under the plugin cache.
2. Reduce controller frontmatter to the smallest valid form if needed.
3. Add a trivial smoke-test plugin agent with minimal frontmatter.
4. Restart Copilot completely and run `/agent`.
5. If smoke-test agent appears, isolate which controller frontmatter field causes the controller to be ignored.
6. If smoke-test agent still does not appear, investigate whether this CLI/App build loads plugin-provided custom agents differently than repository-level agents.
7. Only after controller discovery works, validate controller → subagent orchestration.

Do not assume undocumented behavior. Check current official GitHub Copilot App / CLI plugin documentation where necessary.

## First functional milestone after discovery is fixed

Prove this end-to-end inside the GitHub Copilot App:

GitHub issue
→ Intake
→ Architect
→ human Scope Gate
→ isolated implementation session/worktree
→ Developer
→ QA
→ Reviewer
→ PR + native CI
→ human Merge Gate

No Canvas yet. No deterministic scope hook yet. No Release-helper yet.

## Second milestone

Add deterministic write-scope enforcement.

Requirements:

- canonicalize paths
- deny writes outside human-approved scope
- protect the approval/scope state itself
- demonstrate in-scope allow
- demonstrate out-of-scope deny
- route legitimate expansion back through Scope Gate

Prompt instructions are not sufficient as the hard control.

## Third milestone

Add a Gated Change Canvas as the human control surface.

It should show at minimum:

- issue/readiness
- original acceptance criteria
- root cause
- proposed changed files
- approved scope
- blast-radius risk
- validation plan
- current stage
- QA result
- Reviewer findings
- CI status
- PR status

Human controls should include:

- Approve
- Reject — request revision
- Reject — send back entirely

Only a human action should be able to broaden approved scope.

## Guidance for any IDE LLM

When asked to continue implementation:

- inspect the repository before editing
- read `implementation-plan.md`
- preserve existing decisions and user-authored work
- make small, reviewable changes
- verify behavior in the real GitHub Copilot App/CLI rather than assuming
- clearly distinguish platform limitations from bugs in this repository
- do not add agents just because a task can be agentic
- prefer deterministic controls where the design calls for them

## Suggested first prompt for an IDE assistant

> Read `PROJECT_CONTEXT.md`, `implementation-plan.md`, `AGENTS.md`, `CODEX-HANDOFF.md`, and the current `plugins/gated-change` tree. Do not redesign the workflow. First diagnose why the installed plugin's `gated-change-controller` agent is present on disk but is not shown by `/agent` in GitHub Copilot CLI 1.0.86-2. Propose the smallest test that distinguishes a plugin-loading problem from an invalid agent-frontmatter problem. Make no broad architectural changes until that discovery issue is resolved.
