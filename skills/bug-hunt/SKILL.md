---
name: bug-hunt
description: "Diagnose a bug in a mobile project, write a failing repro test that pins it down, fix the code, then verify with review. TDD-style bug workflow: repro test first, then fix, then confirm test flips green. Triggers on: /mobile-kit:bug-hunt <bug description or stacktrace> (full workflow) or /mobile-kit:bug-hunt fast <bug> (fast track for small, clearly-localized bugs)"
---

You are now the orchestrator for a bug hunt. Follow the steps below exactly. Coordinate by spawning specialized agents and reading a shared context file between steps. Do NOT delegate orchestration — YOU execute these steps directly.

## When To Use This Skill

Use `/mobile-kit:bug-hunt` for:
- A crash report / stacktrace / crash-reporting-tool issue
- A user-reported bug ("X doesn't work when I do Y")
- A regression discovered manually or via CI
- A rotation / lifecycle / edge-case bug the user found

Do NOT use for:
- Adding a new feature — use `/mobile-kit:orchestrate` instead
- Refactoring without a specific bug — use `/mobile-kit:orchestrate` instead
- Code review of existing changes — use `/mobile-kit:review-loop`

## Setup

Precondition: the project must contain `.claude/docs/PROJECT_CONTEXT.md`. If missing, stop and tell the user to run `/mobile-kit:adopt` first.

Agent-type resolution: the agent names in this workflow (e.g. `bug-fixer`, `test-writer`) are mobile-kit plugin agents, which appear plugin-namespaced in your available-agents list (e.g. `mobile-kit:bug-fixer`). For every spawn, prefer a project-local agent with the plain name if one is available (that is a deliberate per-project override); otherwise use the `mobile-kit:`-namespaced type. Never skip an agent because the plain name is missing.

1. Generate a timestamp: run `date '+%Y-%m-%d-%H-%M-%S'` via Bash
2. Derive a short kebab-case name from the bug (e.g., "caption-lost-rotation", "npe-on-empty-list")
3. Create the shared context file at `.claude/agents-log/<timestamp>-bug-<short-name>.md` with this initial content:

```markdown
# Bug Hunt: <short bug description>

## Bug Report
<paste the user's full bug description, stacktrace, or repro steps here>

## File References
<any file paths, screenshots, or logs the user provided>
```

Store the context file path — pass it to every agent.

## Mode Selection

The first word of the arguments selects the mode:

- `fast <bug>` → run the **Fast Track Workflow** (see below). Strip the word `fast`; the remainder is the bug report.
- `full <bug>` or no flag → run the full **Workflow**.

The mode is ALWAYS the user's explicit choice. Never downgrade a bug to the fast track on your own judgment, and never suggest mid-run that the full workflow is overkill — if the user chose full for a small bug, they have a reason. The only mode change allowed mid-run is a fast→full upgrade, and only after asking the user (see the escalation valves in the fast track).

Record the mode in the context file header as `Mode: fast` or `Mode: full`.

## Workflow (full — default)

Execute in order. Read the context file between steps to verify progress.

### Step 1: Bug Fixer — Diagnose ONLY (do not fix yet)

Spawn the bug fixer for root-cause analysis:
```
Agent(subagent_type: "bug-fixer")
```

Prompt must include:
- The bug description / stacktrace / repro steps
- Instruction: "READ the context file at `<path>`. Your ONLY task in this step is diagnosis — do NOT modify code. Read the relevant stacktrace or repro steps. Trace the root cause through the codebase with file:line evidence. If you need runtime evidence (logs, dispatcher state, view-model state) to be certain of the root cause, install TEMPORARY diagnostic prints, run the failing scenario, capture logs, then REMOVE the prints before finishing. When done, append your results under a `## Bug Fixer — Diagnosis` heading including: (a) confirmed root cause with file:line, (b) the exact user-observable symptom, (c) the diverging axis (why does it happen in scenario X but not Y), (d) a proposed fix approach (not code — just the approach)."

After it completes, read the context file. Verify:
- Root cause has concrete file:line references
- Symptom is described in user-observable terms
- No temporary diagnostic prints were left in prod code (grep confirms)

If the bug fixer's diagnosis is uncertain and asks a question, relay it to the user and wait.

### Step 2: Test Writer — Failing Repro

Spawn the test writer to pin down the bug with a failing test:
```
Agent(subagent_type: "test-writer")
```

Prompt must include:
- Instruction: "READ the context file at `<path>` — pay close attention to `## Bug Fixer — Diagnosis`. Write ONE OR MORE tests that reproduce this bug, using the project's UI/integration tests as defined by the test policy in `.claude/docs/PROJECT_CONTEXT.md`. Test names should read as the bug behavior (e.g., `captionIsLostAfterRotationAndSystemBackFromPreview`). Include a CONTROL test that documents the working path (e.g., `captionSurvivesReturnFromPreviewWithoutRotation`) — this pins down what the fix must NOT break. If the bug involves state persistence across a lifecycle event (rotation/orientation change, process death, background), add a SECOND-EVENT assertion (e.g., rotate twice) to guard against 'double-fire' fixes. Run the tests with the test commands from the project context and confirm the repro test FAILS and any control test PASSES. Add any missing helpers using the project's UI-test abstractions and fixture scenarios per the project context. When done, append under `## Test Writer — Repro`: the test names, files touched, test run results (showing repro=FAILED, control=PASSED)."

After it completes, verify:
- The repro test FAILS as expected — proves the bug is real
- The control test PASSES — proves the surrounding path still works
- Second-event assertion is present if lifecycle is involved

### Step 3: Bug Fixer — Fix

Spawn the bug fixer to apply the fix:
```
Agent(subagent_type: "bug-fixer")
```

Prompt must include:
- Instruction: "READ the context file at `<path>`. Your diagnosis is in `## Bug Fixer — Diagnosis`. The failing repro test(s) are in `## Test Writer — Repro`. Implement the smallest correct fix that makes the repro test(s) green WITHOUT breaking the control test(s) or any pre-existing tests. Do NOT modify the tests. Run the repro + control tests after fixing and report the outcome. When done, append under `## Bug Fixer — Fix`: files modified, the smallest possible diff summary, test run results (repro=PASSED, control=PASSED, plus any related regression tests you ran)."

After it completes, verify:
- Repro test transitions from FAILED to PASSED
- Control test stays PASSED
- Test files (in the project's test location(s) per the project context) were NOT modified — this is a critical check. If the bug fixer modified tests to make them green, the fix is invalid.

Two branches:

- **Repro PASSED + control PASSED + no test files touched** → proceed to Step 5.
- **Bug fixer believes a test is wrong** → the bug fixer must append `## Developer Test Concern` (yes, same heading as in `/mobile-kit:orchestrate`) with rationale. Proceed to Step 4.

### Step 4: Test Writer — Push-Back Loop (max 2 iterations)

Same push-back mechanism as `/mobile-kit:orchestrate`. Track iteration count.

Spawn test writer:
```
Agent(subagent_type: "test-writer")
```

Prompt: "Read the context file at `<path>`. The bug fixer has raised a concern about the repro test under `## Developer Test Concern`. Evaluate it honestly. Either adjust the repro test (append `## Test Writer — Iteration <n>` with reasoning) or defend it (append `## Test Writer — Defense (Iteration <n>)` with (i) the user-observable contract the bug violates, (ii) why the fixer's alternative is incomplete or bypasses the bug, (iii) a concrete example scenario). Re-run affected tests and report."

Branches:
- **Test adjusted** → re-spawn bug fixer to re-fix against the adjusted test. Return to Step 3 with iteration +1.
- **Test defended and bug fixer agrees (tests all green after refix)** → proceed to Step 5.
- **Iteration 2 unresolved** → STOP. Escalate to user via `## Escalation to User` summarizing both positions. Wait for user input.

### Step 5: Tech Lead + QA Reviewer (PARALLEL)

Spawn BOTH in parallel (single message, two tool calls):

**Tech Lead:**
```
Agent(subagent_type: "tech-lead")
```
Prompt: "Read the context file at `<path>` for full bug context, repro tests, and the fix. Review the fix for: (a) is this the smallest correct diff, (b) does it introduce new architectural violations, (c) are there other sites in the codebase with the SAME bug pattern that should be fixed in the same PR, (d) did the fixer modify tests (should be zero). Append `## Tech Lead Review` with findings and file:line references."

**QA Reviewer:**
```
Agent(subagent_type: "qa-reviewer")
```
Prompt: "Read the context file at `<path>` for full bug context, repro tests, and the fix. Look for edge cases the fix does not cover: null / empty / concurrent / lifecycle-transition / different device configurations. Verify the repro test's assertions actually pin the bug down (not just accidentally green). Suggest additional test cases if coverage has gaps. Append `## QA Review` with findings."

After BOTH complete, read the context file.

### Step 6: Fix Issues (IF NEEDED)

If tech-lead flags "other sites have this bug pattern" → decide with user before fixing more sites (scope creep guard).

If QA flags missing coverage:
1. Spawn test writer to add tests: "Read `<path>`. QA identified missing coverage under `## QA Review`. Add tests. Run them. Append under `## Test Writer — Coverage Extension`."
2. If new tests fail against current fix → spawn bug fixer to close the gap.

If tech-lead or QA flag correctness issues in the fix itself:
1. Spawn bug fixer to address them.
2. Re-run relevant reviewers.
3. Repeat until clean.

### Step 7: Design Guardian (CONDITIONAL)

Skip if zero UI files touched. Otherwise:
```
Agent(subagent_type: "design-system-guardian")
```
Prompt: "Read `<path>`. Review UI changes for design system compliance AND visual fidelity. **Look at the rendered screen, not only the source** — capture it yourself if the project has no screenshot tests. Append `## Design Guardian Review`, opening with a one-line verification method (pixels vs source-audit fallback)."

Before accepting a PASS, check that verification line — a source-only audit cannot catch an element that
is invisible against its actual background or a layout that contradicts the mock.

### Step 8: Release Notes

Spawn the release-notes writer:
```
Agent(subagent_type: "release-notes-writer")
```
Prompt: "Read the context file at `<path>` for what was actually built, and `git diff` for the real change. Append one user-facing entry to the project's `RELEASENOTES.md` under `## Unreleased`, matching the file's existing format if it has one. Write for someone using the app, not for the team — no class names, no layer names. If the change has no user-visible effect, leave the file untouched. When done, append your results under a `## Release Notes` heading in the context file."

This step never blocks. If the agent reports `'No user-facing change to record.'`, that is a normal outcome — record it and continue.

### Step 9: Final Report

Read the context file one final time. Present to user:

```
## Bug Hunt Complete

### Bug Summary
- **Symptom:** <one line>
- **Root cause:** <one line — file:line>
- **Fix:** <one line — file:line>

### Delegation Report
| # | Agent (subagent_type) | Agent ID | Status | Summary |
|---|----------------------|----------|--------|---------|
| 1 | bug-fixer diagnosis (bug-fixer) | <id> | completed | <one line> |
| 2 | test-writer repro (test-writer) | <id> | completed | <one line — N tests written, repro=RED, control=GREEN> |
| 3 | bug-fixer fix (bug-fixer) | <id> | completed | <one line — repro flipped to GREEN> |
| N | release-notes (release-notes-writer) | <id> | completed | <one line — entry written, or no user-facing change> |
| 4 | test-writer re-eval (test-writer) | <id> | completed/skipped | <only if push-back triggered> |
| 5 | tech-lead (tech-lead) | <id> | completed | <one line> |
| 6 | qa-reviewer (qa-reviewer) | <id> | completed | <one line> |
| ... | ... | ... | ... | ... |

**Repro tests:** <count> — all currently GREEN
**Push-back iterations:** <0 | 1 | 2 | escalated>
**Other sites with same pattern:** <N flagged by tech-lead — deferred to user | fixed in-scope>
**Context file:** `<path>`
```

Rules for the report:
- List EVERY agent spawn in chronological order (including re-runs and push-back)
- Agent ID is the ID returned by each Agent tool call
- MUST contain at minimum: 2x bug-fixer (diagnosis + fix), 1x test-writer, 1x tech-lead, 1x qa-reviewer

## Fast Track Workflow (`fast` flag only)

For small, clearly-localized bugs the user explicitly flagged as `fast` — a stacktrace that points at an obvious spot, a one-screen glitch, a regression with a known trigger. The TDD contract is unchanged — failing repro test before the fix, fixers never modify tests, an independent review — but with fewer agents and fewer emulator runs. What gets cut: the dedicated diagnosis spawn, the second parallel reviewer, full-suite test runs, and open-ended fix loops.

### Step FB1: Inline Diagnosis (no diagnosis agent)

YOU diagnose the bug yourself — do not spawn the bug fixer for diagnosis. Read the stacktrace/repro steps, trace the root cause in the code, then append to the context file under `## Diagnosis (fast track)`:
- Confirmed root cause with file:line
- The user-observable symptom in one line
- Proposed fix approach (not code — just the approach)
- Repro test plan: ONE repro test + ONE control test

**Escalation valve:** if you cannot pin the root cause down with confidence from reading the code — you would need runtime evidence (temporary diagnostic prints, log captures, dispatcher/state inspection) — the bug is not fast-track material. STOP and ask the user whether to upgrade to the full workflow (the context file carries over — restart at Step 1) or continue fast anyway.

### Step FB2: Test Writer — Scoped Repro

Spawn the test writer exactly as in Step 2, but with these scope overrides appended to the prompt:
- "This is a fast-track run. Write exactly the tests listed in `## Diagnosis (fast track)` — one repro test and one control test, no more. Add them to an existing test file if one covers the same feature. Include the second-event assertion if the bug is lifecycle-related. Verify repro=FAILED and control=PASSED by running ONLY these tests via the targeted single-class/single-method test command from the project context — never a full suite."

The verification checklist from Step 2 applies unchanged.

### Step FB3: Bug Fixer — Fix

Spawn the bug fixer exactly as in Step 3, additionally instructing: "Verify with the same targeted test command the test writer used — do not run full suites." The push-back mechanism (`## Developer Test Concern` → Step 4, max 2 iterations) and the no-test-modification check apply unchanged.

**Escalation valve:** if the fix grows beyond a small diff (multiple subsystems, architectural change, the diagnosis turns out wrong), STOP and ask the user whether to upgrade to the full workflow before reviewing.

### Step FB4: Combined Review (single agent)

Instead of parallel tech-lead + qa-reviewer, spawn ONE tech-lead agent with a combined brief:

```
Agent(subagent_type: "tech-lead")
```
Prompt: "Read the context file at `<path>` for full bug context, repro tests, and the fix. This is a fast-track combined review — cover BOTH checklists: (a) is this the smallest correct diff, (b) does it introduce architectural violations, (c) are there other sites in the codebase with the SAME bug pattern (report them — do NOT fix them), (d) did the fixer modify tests (must be zero), (e) edge cases the fix does not cover (null / empty / concurrent / lifecycle), (f) do the repro test's assertions actually pin the bug down. The review is READ-ONLY: do not run builds or tests; read the diff and the test results already recorded in the context file. Report ONLY issues that must block a commit under `## Combined Review`, with file:line references. List deliberately skipped nitpicks and same-pattern sites under `## Combined Review — Minor (not blocking)`."

If UI files with visible changes were touched, spawn the design guardian (Step 7 prompt) IN PARALLEL with the combined review — both Agent calls in the same message. Skip the guardian entirely for invisible or non-UI changes.

### Step FB5: Fix Round (max 1)

If blocking issues were found: spawn the bug fixer once to fix them (re-running only the affected tests), then re-spawn the combined reviewer once to verify. If blocking issues remain after this single round, STOP and report them to the user — the user decides whether to keep fixing in fast mode or upgrade to the full workflow. Same-pattern sites flagged by the review are ALWAYS deferred to the user (scope creep guard), never fixed in the fast track.

### Step FB6: Release Notes

Spawn the release-notes writer:
```
Agent(subagent_type: "release-notes-writer")
```
Prompt: "Read the context file at `<path>` for what was actually built, and `git diff` for the real change. Append one user-facing entry to the project's `RELEASENOTES.md` under `## Unreleased`, matching the file's existing format if it has one. Write for someone using the app, not for the team — no class names, no layer names. If the change has no user-visible effect, leave the file untouched. When done, append your results under a `## Release Notes` heading in the context file."

This step never blocks. If the agent reports `'No user-facing change to record.'`, that is a normal outcome — record it and continue.

### Step FB7: Final Report

Same report format and rules as Step 9, with these differences:
- Add a `**Mode:** fast` line
- Minimum required spawns: 1x test-writer, 1x bug-fixer (fix), 1x tech-lead (combined review). The diagnosis row is listed as `inline (fast track)`, the qa-reviewer row as `skipped (fast track)`.
- If the review recorded non-blocking minor findings or same-pattern sites, list them at the end of the report so the user can decide.

## Rules

- The mode (fast/full) is the user's explicit choice via the `fast` flag — never pick or switch it yourself; fast→full upgrades only via the escalation valves, and only after asking the user
- Diagnosis comes BEFORE test-writer — via bug-fixer spawn (full) or inline by you (fast); the test needs the root cause to be pinned first
- You MUST spawn test-writer BEFORE the fix — the test is the contract (both modes)
- The fix agent MUST NOT modify test files. If they do, the fix is invalid — re-spawn with correction instruction.
- Push-back loop capped at 2 iterations. Escalate on iteration 3.
- Tech-lead + QA are NEVER optional (full workflow; in the fast track the combined review in Step FB4 is the never-optional equivalent).
- Never commit — user reviews and commits manually.
- Spawn the release-notes writer after the reviews pass, before the final report. It never blocks.
- Never skip the final report.
- If the bug fixer requires temporary diagnostic prints in prod code for evidence, they MUST remove them before finishing. Grep for their tag as a sanity check.
