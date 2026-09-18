---
name: gated-change
description: Run the repository's governed issue-to-PR workflow with specialist agents, bounded retries, explicit human scope approval, independent validation, and human merge review.
---

# Gated Change

Use this skill when a user wants to take a real GitHub issue through the governed change workflow defined in `implementation-plan.md`.

## Workflow contract

1. Start from real GitHub issue context. The controller has no direct shell tool; the default fetch path is asking the human to run `gh issue view --json ...` and paste the raw output — not a last-resort fallback, but the primary mechanism, because it is the only link in the chain with no LLM narration between the real GitHub API and the data the workflow acts on. A subagent-delegated fetch may be attempted instead, but its returned JSON must be treated as unverified until the human explicitly confirms it matches the real issue — in practice, subagent fetches in this environment have been observed to fabricate complete, plausible-looking JSON for content that doesn't exist, so quoting the JSON is necessary but not sufficient proof. Never reconstruct issue content from model memory or a chat attachment preview, even if the same issue was discussed earlier in the session. If the fetched (and, for subagent fetches, human-confirmed) issue has no substantive content (blank/whitespace-only body, no comments — GitHub disallows a blank title, so a bare title alone never counts as substantive content), the controller stops and asks the human to add content to the issue itself — it does not invoke Intake for a content-empty issue.
2. Run Intake Triage against the Definition of Ready — only for issues that have real content to evaluate; Intake's job is judging sufficiency, not detecting total absence.
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
- Never substitute a generic/general-purpose agent for a named specialist, even as a fallback when routing to the named agent fails — a generic agent has none of the specialist's tool restrictions, and inline prompt instructions are not a substitute for enforced tool boundaries. A failed invocation (the agent never started) costs little and is not a failed implementation attempt, so retry the same named agent up to 4 times; if it still won't start, stop and put the decision to the human — never substitute another agent.

## Bounded-loop rules

- Intake clarification: <= 2 rounds.
- Scope negotiation: <= 2 cumulative rounds.
- Developer -> QA -> Reviewer implementation attempts: <= 3.
- A Developer invocation that never started (routing/tool-level failure, no side effects) does not consume an implementation attempt \u2014 retry it directly. A Developer stall mid-task (real edits may already exist) does consume one attempt, and must resume by first checking the actual git state rather than assuming a clean worktree.
- Escalate rather than loop beyond a bound.

## Efficiency rules

- Carry structured outputs forward instead of asking later agents to re-discover prior-stage context.
- Prefer deterministic compute over LLM reasoning for baseline failure comparison, flaky reruns, infrastructure signature detection, and monorepo reference sweeps.
- Fetching the source issue is deterministic compute too: always use real `gh issue view` output verbatim, never a remembered or paraphrased version, even on retries. A fabricated issue body is a critical integrity failure, not an acceptable degradation.
- Anti-fabrication is not just an instruction, it needs a forcing function — and even the forcing function has a known limit. The controller must quote the exact raw JSON it received, verbatim, before saying anything about the issue's content or readiness; a prose description with no quoted output backing it is a fabrication. But quoted JSON alone is not sufficient proof for a subagent-delegated fetch: in practice, a subagent has fabricated a complete, structurally-valid JSON blob for an issue that was actually empty, and the controller quoted it faithfully without knowing it was fake. This is why human-pasted output is the default path, and why any subagent-delegated fetch requires an explicit human "yes this matches" confirmation before the controller may treat it as ground truth — the human is the only verification step in this chain that cannot itself hallucinate a tool result.
- Checking whether the fetched issue is content-empty is also deterministic compute, done by the controller before spending an Intake invocation: blank body + no comments (regardless of title, since GitHub requires a non-blank title but a bare title never satisfies the Definition of Ready) costs zero agent turns, not one. Missing/incomplete (but non-empty) fields are still Intake's job to flag, bounded by the 2-round clarification cap; the fix in either case is updating the GitHub issue itself as the source of truth, never inventing values in-session.
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
