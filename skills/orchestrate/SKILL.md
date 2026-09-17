---
name: orchestrate
description: "Orchestrate a full mobile feature workflow by spawning specialized agents in sequence (TDD-style: tests first, then implementation, then review). Works with any mobile stack (Android/KMM, Flutter) via the project context contract. Triggers on: /mobile-kit-test:orchestrate <task> (full workflow) or /mobile-kit-test:orchestrate fast <task> (fast track for small tasks)"
---

You are now the orchestrator. Follow the steps below exactly. You coordinate by spawning agents and reading a shared context file between steps. Do NOT delegate orchestration to another agent — YOU execute these steps directly.

## Setup

Precondition: the project must contain `.claude/docs/PROJECT_CONTEXT.md`. If missing, stop and tell the user to run `/mobile-kit-test:adopt` first.

Agent-type resolution: the agent names in this workflow (e.g. `mobile-planner`, `tech-lead`) are mobile-kit-test plugin agents, which appear plugin-namespaced in your available-agents list (e.g. `mobile-kit-test:tech-lead`). For every spawn, prefer a project-local agent with the plain name if one is available (that is a deliberate per-project override); otherwise use the `mobile-kit-test:`-namespaced type. Never skip an agent because the plain name is missing.

1. Generate a timestamp: run `date '+%Y-%m-%d-%H-%M-%S'` via Bash
2. Derive a short kebab-case name from the user's task (e.g., "order-history", "settings-search")
3. Create the shared context file at `.claude/agents-log/<timestamp>-<short-name>.md` with this initial content:

```markdown
# Orchestration: <task name>

## Task
<paste the user's full task description here>

## File References
<list any file paths, screenshots, or plan files the user mentioned>
```

Store the context file path — you will pass it to every agent.

## Mode Selection

The first word of the arguments selects the mode:

- `fast <task>` → run the **Fast Track Workflow** (see below). Strip the word `fast`; the remainder is the task.
- `full <task>` or no flag → run the full **Workflow**.

The mode is ALWAYS the user's explicit choice. Never downgrade a task to the fast track on your own judgment, and never suggest mid-run that the full workflow is overkill — if the user chose full for a small task, they have a reason. The only mode change allowed mid-run is a fast→full upgrade, and only after asking the user (see the escalation valves in the fast track).

Record the mode in the context file header as `Mode: fast` or `Mode: full`.

## Workflow (full — default)

Execute these steps in order. Between each step, read the context file to check results before proceeding.

### Step 1: Planner

Spawn the planner agent:
```
Agent(subagent_type: "mobile-planner")
```

Prompt must include:
- The user's task description
- All file references (plans, screenshots) from the user's message
- Instruction: "When done, append your results under a `## Planner` heading in the context file at `<path>`. Include: what you analyzed, the plan summary, key decisions, and list of files to create/modify. Your plan feeds the test-writer agent next — it MUST include (a) a Test Plan section enumerating the user flows the test-writer must cover (happy path, edge cases, regression guards, lifecycle events), and (b) a phased Implementation Order where each phase lists test-writer steps first (tests to write, UI-test abstractions and fixture scenarios to add) and developer steps second (files to create/modify to make those tests green). Reference the `#### 9. Test Plan` and `#### 10. Implementation Order` sections in your agent definition for the exact structure."

After it completes, read the context file. Verify a Test Plan section exists and per-phase steps split test-writer / developer work. If missing or vague, re-spawn the planner with corrective instruction — do NOT proceed to Step 2 without this. If the planner has questions, relay them to the user and wait for answers before continuing.

### Step 2: Design Analyzer (OPTIONAL)

Skip this step ONLY if the task is purely data-layer/backend with zero UI changes.

Spawn the design analyzer:
```
Agent(subagent_type: "design-analyzer")
```

Prompt must include:
- The user's task description and any design screenshots
- Instruction: "Read the context file at `<path>` for planner results. When done, append your results under a `## Design Analyzer` heading. Include: reusable components identified, design requirements, and recommendations."

After it completes, read the context file.

### Step 3: Test Writer (TDD-first)

Spawn the test writer agent:
```
Agent(subagent_type: "test-writer")
```

Prompt must include:
- The user's task description
- Instruction: "Read the context file at `<path>`. Your primary input is the planner's `#### 9. Test Plan` section and the per-phase test-writer steps in `#### 10. Implementation Order`. That section enumerates the user flows you must cover — start there. If the plan is thin or missing flows you think are needed (e.g., a lifecycle regression the planner didn't mention), ADD them and note the addition. Do NOT re-derive flows the planner already identified — the plan is the contract handoff. Write failing tests covering every listed flow, using the project's UI/integration tests as defined by the test policy in `.claude/docs/PROJECT_CONTEXT.md`. Add any missing helpers using the project's UI-test abstractions and fixture scenarios per the project context. Run the tests with the test commands from the project context and confirm they FAIL as expected — pre-implementation. When done, append your results under a `## Test Writer` heading including the test-cases list (referencing which flow from the plan each covers, plus any flows you added and why), files touched, test-run results, any prod-code testability hooks applied (with rationale), and notes for the developer."

After it completes, read the context file. Verify:
- `## Test Writer — Test Cases` section exists with a numbered list of flows
- `## Test Writer` section includes files touched and test-run results showing FAILED (expected pre-impl)
- No prod-code hooks were applied outside the documented allowlist (visibility opens, testing-only accessors)

If the test writer left questions under `## Test Writer — Questions`, relay them to the user and wait.

### Step 4: Developer

Spawn the developer agent:
```
Agent(subagent_type: "mobile-developer")
```

Prompt must include:
- The user's task description
- Instruction: "Read the context file at `<path>` for the full plan, design analysis, and — most importantly — the `## Test Writer` section listing the tests you must make pass. Implement the feature so that ALL tests in `## Test Writer — Test Cases` transition from RED to GREEN. When done, run the same test command(s) from the Test Writer section and confirm they now PASS. Append your results under a `## Developer` heading including files created/modified, build result, test-run results (showing PASSED), and any decisions made during implementation. If you believe a test is incorrect, DO NOT modify the test — instead append your concern under `## Developer Test Concern` with: which test, what you believe is wrong, your rationale, what alternative user contract you propose."

After it completes, read the context file. Two branches:

- **All tests green + no `## Developer Test Concern`** → proceed to Step 6.
- **`## Developer Test Concern` present** → proceed to Step 5.

### Step 5: Test Writer Re-evaluation (Push-Back Loop, max 2 iterations)

Track iteration count. Start at 1.

Spawn the test writer again:
```
Agent(subagent_type: "test-writer")
```

Prompt must include:
- Instruction: "Read the context file at `<path>`. The developer has raised a concern about one of your tests under `## Developer Test Concern`. Evaluate it with intellectual honesty — the developer might be right. Investigate the code they wrote. Then either: (a) ADJUST the test if their point is valid, appending under `## Test Writer — Iteration <n>` with the reasoning; OR (b) DEFEND the test if it's correct, appending under `## Test Writer — Defense (Iteration <n>)` with (i) the user-observable contract, (ii) why the developer's alternative breaks it, (iii) a concrete failure scenario. Then re-run the affected tests and report the outcome."

After it completes, read the context file:
- **If tests were adjusted** → re-spawn developer with instruction "Read the context file at `<path>`. The test writer adjusted tests per iteration <n>. Re-run the tests and either pass or raise a new concern under `## Developer Test Concern (Iteration <n+1>)`." → return to top of Step 5 with iteration +1.
- **If test writer defended and developer accepts (all tests green after re-run)** → proceed to Step 6.
- **If iteration count reaches 2 AND still unresolved** → STOP. Escalate to user via `## Escalation to User` in the context file summarizing both positions. Wait for user decision. Do not proceed to Step 6 without user input.

### Step 6: Tech Lead + QA Reviewer (PARALLEL)

Spawn BOTH agents in parallel (in a single message with two Agent tool calls):

**Tech Lead:**
```
Agent(subagent_type: "tech-lead")
```
Prompt: "Read the context file at `<path>` for full task context, tests, and what was implemented. Review the implementation for architecture compliance, SOLID principles, naming conventions, and code quality. Also verify the developer did not modify tests to make them pass (compare against `## Test Writer` file list). When done, append your results under a `## Tech Lead Review` heading. List issues found (if any) with file paths and line numbers."

**QA Reviewer:**
```
Agent(subagent_type: "qa-reviewer")
```
Prompt: "Read the context file at `<path>` for full task context, tests, and what was implemented. Review for bugs, edge cases, null safety, race conditions, and potential failures. Also assess whether the tests written by the test-writer sufficiently cover the actual implementation — note any user-flow gaps. When done, append your results under a `## QA Review` heading. List issues found (if any) with file paths and line numbers."

After BOTH complete, read the context file to check their findings.

### Step 7: Fix Issues (IF NEEDED)

If tech-lead or qa-reviewer found issues:
1. Spawn developer again with: "Read the context file at `<path>`. Fix the issues listed in the Tech Lead Review and/or QA Review sections. Re-run the tests to confirm nothing broke. Append your fixes under a `## Developer Fix` heading."
2. If QA identified user-flow gaps → spawn test writer to add tests: "Read the context file at `<path>`. QA identified missing coverage under `## QA Review`. Add tests for those flows and run them. Append under `## Test Writer — Coverage Extension`."
3. Re-spawn ONLY the reviewer(s) that found issues to verify fixes
4. Repeat until both reviewers pass clean

### Step 8: Design Guardian (CONDITIONAL)

Skip ONLY if zero UI files were touched (no screens, layouts, widgets/composables, adapters, or view components).

Spawn the design guardian:
```
Agent(subagent_type: "design-system-guardian")
```
Prompt: "Read the context file at `<path>` for full context. Review all UI code written or modified for design system compliance AND visual fidelity. **You must look at the rendered screen, not only the source** — if the project has no screenshot tests, capture it yourself (drive the surface with an existing UI test while polling `adb exec-out screencap`, or the platform equivalent). If a design/mock was provided, compare against it by measuring normalised gaps, not by eye. When done, append your results under a `## Design Guardian Review` heading, opening with a one-line **verification method** stating whether you looked at pixels or fell back to a source audit."

If design guardian finds issues, spawn developer to fix, then re-run design guardian.

**Before accepting a PASS, check the verification method line.** A design review that only read source
has not done Job 2 — a screen can pass every design-token check and still be visibly wrong (two valid
tokens resolving to the same colour, a house pattern that contradicts the mock, spacing that is
self-consistent but nothing like the design). If the line says "source audit only", either send it back
to capture the screen or record explicitly in the final report that visual fidelity was never verified.
Do not let the same measurable mock deviation be filed "for designer sign-off" across multiple
rounds — that is a fix the agent is deferring.

### Step 9: Release Notes

Spawn the release-notes writer:
```
Agent(subagent_type: "release-notes-writer")
```
Prompt: "Read the context file at `<path>` for what was actually built, and `git diff` for the real change. Append one user-facing entry to the project's `RELEASENOTES.md` under `## Unreleased`, matching the file's existing format if it has one. Write for someone using the app, not for the team — no class names, no layer names. If the change has no user-visible effect, leave the file untouched. When done, append your results under a `## Release Notes` heading in the context file."

This step never blocks. If the agent reports `'No user-facing change to record.'`, that is a normal outcome — record it and continue.

### Step 10: Final Report

After all steps complete, read the context file one final time and present this report to the user:

```
## Orchestration Complete

### Delegation Report
| # | Agent (subagent_type) | Agent ID | Status | Summary |
|---|----------------------|----------|--------|---------|
| 1 | planner (mobile-planner) | <id> | completed/skipped | <one line> |
| 2 | design-analyzer (design-analyzer) | <id> | completed/skipped | <one line> |
| 3 | test-writer (test-writer) | <id> | completed | <one line — N tests written, initially RED> |
| 4 | developer (mobile-developer) | <id> | completed | <one line — tests green> |
| 5 | test-writer re-eval (test-writer) | <id> | completed/skipped | <one line — only if push-back triggered> |
| 6 | tech-lead (tech-lead) | <id> | completed | <one line> |
| 7 | qa-reviewer (qa-reviewer) | <id> | completed | <one line> |
| ... | ... | ... | ... | ... |
| N | release-notes (release-notes-writer) | <id> | completed | <one line — entry written, or no user-facing change> |

**Tests written:** <count> — <all passing / <n> passing / <breakdown>>
**Push-back iterations:** <0 | 1 | 2 | escalated>
**Context file:** `<path>`
```

Rules for the report:
- List EVERY agent spawn in chronological order (including re-runs for fixes and push-back iterations)
- Agent ID is the ID returned by each Agent tool call — this proves real delegation
- The report MUST contain at minimum: 1x planner, 1x test-writer, 1x developer, 1x tech-lead, 1x qa-reviewer

## Fast Track Workflow (`fast` flag only)

For small tasks the user explicitly flagged as `fast`. The TDD contract is unchanged — tests before implementation, developers never modify tests, at least one independent review — but with fewer agents and fewer emulator runs. What gets cut: the planner spawn, the design-analyzer, the second parallel reviewer, full-suite test runs, and open-ended fix loops.

### Step F1: Inline Mini-Plan (no planner agent)

YOU write the plan yourself — do not spawn the planner. Briefly explore the code involved, then append to the context file under `## Plan (fast track)`:
- Files to create/modify (expected: roughly 3-4 or fewer)
- Approach in 5-10 lines
- Test plan: 1-2 flows maximum — the happy path plus one regression guard

**Escalation valve:** if while exploring you find the task needs a new architectural component (new repository, new screen/state holder, changed API contract) or clearly more files than expected, STOP and ask the user whether to upgrade to the full workflow (the context file carries over — restart at Step 1) or continue fast anyway.

### Step F2: Test Writer (scoped)

Spawn the test writer exactly as in Step 3, but with these scope overrides appended to the prompt:
- "This is a fast-track run. Cover ONLY the flows listed in `## Plan (fast track)` — maximum 2 tests, no flow matrix. Add tests to an existing test file if one covers the same feature. Verify RED by running ONLY the new tests via the targeted single-class/single-method test command from the project context — never a full suite."

The verification checklist and `## Test Writer — Questions` relay from Step 3 apply unchanged.

### Step F3: Developer

Spawn the developer exactly as in Step 4, additionally instructing: "Verify GREEN with the same targeted test command the test writer used — do not run full suites." The push-back mechanism (`## Developer Test Concern` → Step 5, max 2 iterations) applies unchanged.

**Escalation valve:** if the developer's report shows the change grew beyond the mini-plan scope (significantly more files, architectural change), STOP and ask the user whether to upgrade to the full workflow before reviewing.

### Step F4: Combined Review (single agent)

Instead of parallel tech-lead + qa-reviewer, spawn ONE tech-lead agent with a combined brief:

```
Agent(subagent_type: "tech-lead")
```
Prompt: "Read the context file at `<path>` for full task context, tests, and what was implemented. This is a fast-track combined review — cover BOTH the architecture/convention checklist AND the QA checklist (bugs, edge cases, null safety, race conditions, lifecycle). The review is READ-ONLY: do not run builds or tests; read the diff and the test results already recorded in the context file. Also verify the developer did not modify tests to make them pass (compare against the `## Test Writer` file list). Report ONLY issues that must block a commit (bugs, correctness problems, contract violations) under `## Combined Review`, with file paths and line numbers. List deliberately skipped nitpicks under `## Combined Review — Minor (not blocking)` so nothing is silently dropped."

If UI files with visible changes were touched, spawn the design guardian (Step 8 prompt) IN PARALLEL with the combined review — both Agent calls in the same message. Skip the guardian entirely for invisible or non-UI changes.

### Step F5: Fix Round (max 1)

If blocking issues were found: spawn the developer once to fix them (re-running only the affected tests), then re-spawn the combined reviewer once to verify the fixes. If blocking issues remain after this single round, STOP and report them to the user instead of looping — the user decides whether to keep fixing in fast mode or upgrade to the full workflow.

### Step F6: Release Notes

Spawn the release-notes writer:
```
Agent(subagent_type: "release-notes-writer")
```
Prompt: "Read the context file at `<path>` for what was actually built, and `git diff` for the real change. Append one user-facing entry to the project's `RELEASENOTES.md` under `## Unreleased`, matching the file's existing format if it has one. Write for someone using the app, not for the team — no class names, no layer names. If the change has no user-visible effect, leave the file untouched. When done, append your results under a `## Release Notes` heading in the context file."

This step never blocks. If the agent reports `'No user-facing change to record.'`, that is a normal outcome — record it and continue.

### Step F7: Final Report

Same report format and rules as Step 10, with these differences:
- Add a `**Mode:** fast` line
- Minimum required spawns: 1x test-writer, 1x developer, 1x tech-lead (combined review). Planner and qa-reviewer rows are listed as `skipped (fast track)`.
- If the review recorded non-blocking minor findings, list them at the end of the report so the user can decide whether to address them.

## Rules

- The mode (fast/full) is the user's explicit choice via the `fast` flag — never pick or switch it yourself; fast→full upgrades only via the escalation valves, and only after asking the user
- You MUST spawn each agent as a separate Agent tool call with the correct `subagent_type`
- You MUST read the context file between steps to verify progress
- You MUST spawn test-writer BEFORE developer — the tests are the contract, not an afterthought (both modes)
- You MUST spawn tech-lead and qa-reviewer — they are NEVER optional (full workflow; in the fast track the combined review in Step F4 is the never-optional equivalent)
- Push-back loop is capped at 2 iterations. Escalate on iteration 3.
- If any agent asks a question, relay it to the user and wait for the answer
- Never commit code — the user will review and commit manually
- Spawn the release-notes writer after the reviews pass, before the final report. It never blocks.
- Never skip the final report
