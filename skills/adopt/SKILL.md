---
name: adopt
description: "Onboard the current project to the mobile-kit agents: analyze the codebase, generate the project context pack (.claude/docs/PROJECT_CONTEXT.md + reference docs), scaffold agent memory, and register the marketplace. Re-run anytime to refresh the context pack. Triggers on: /mobile-kit:adopt"
---

You are onboarding this project to the mobile-kit agent suite. The agents are generic; this skill generates the **project context pack** that adapts them to this specific codebase. Work directly — do not delegate the writing, but you may spawn Explore agents for codebase analysis.

## Step 0: Detect mode

- If `.claude/docs/PROJECT_CONTEXT.md` does not exist → **fresh adoption** (all steps).
- If it exists → **re-adoption**: re-verify every section against the current codebase, update stale facts, and bump nothing else. Compare its `CONTRACT_VERSION` with the one in the plugin's `docs/PROJECT_CONTEXT_TEMPLATE.md`; if the template is newer, add any new required sections. Preserve hand-written content — refresh facts, don't rewrite prose. Skip Steps 4–5 unless missing.

## Step 1: Detect the stack

Check in order:
1. `pubspec.yaml` at root → **Flutter**
2. `settings.gradle` / `settings.gradle.kts` including a shared multiplatform module (look for `kotlin("multiplatform")` or a `shared/` module with `commonMain`) → **KMM**
3. `build.gradle(.kts)` with `com.android.application` only → **native Android**
4. Otherwise → ask the user what the stack is before continuing.

## Step 2: Analyze the codebase

Spawn 1–2 Explore agents (or search directly in small projects) to gather, with concrete file paths and real class names:

- Module layout and package/namespace root
- Architecture patterns actually in use: repository/service base classes, DI framework and module locations, state management (ViewModel/Bloc/Cubit/...), mapper/conversion layers
- Feature folder structure (pick 2–3 real features as evidence)
- Build commands: how the app is built (Gradle tasks, `flutter build`, flavors), and whether a "show all errors" variant exists
- Test setup: test types present, where tests live, how a single test class/method is run, existing fakes/fixtures/scenario helpers, UI-test abstractions (robots/page objects/finders), base test classes
- Naming conventions (derive from real classes, don't invent)
- Design system: theme files, color/typography/spacing token locations, shared UI components, string resource mechanism
- Existing convention documents (architecture docs, contribution guides, CLAUDE.md) — these become reference documents, do NOT duplicate their content
- Commit message style: `git log --oneline -30` and derive the observed format

## Step 3: Generate the context pack

Create `.claude/docs/` containing:

1. **`PROJECT_CONTEXT.md`** — fill in the plugin's `docs/PROJECT_CONTEXT_TEMPLATE.md` (find it in the installed plugin directory; if unavailable, reconstruct its section list from this skill: Identity, Reference documents, Build commands, Test commands and policy, Test infrastructure, Source layout and naming, Code style, Commit conventions, Design system, Project quirks — with `CONTRACT_VERSION: 1`). Every REQUIRED section must be filled with verified facts from Step 2. Link to existing project docs instead of duplicating them.
2. **`ARCHITECTURE.md`** — only if the project has NO existing architecture doc: describe the observed patterns (layers, base classes, DI, state management, data flow) with real examples. If the project already documents this, reference that doc in PROJECT_CONTEXT.md instead.
3. **`TEST_INFRASTRUCTURE.md`** — only if not already documented: the test policy, infrastructure classes, how to run things, and 1–2 annotated examples of existing good tests.
4. **`AGENT_ROSTER.md`** — always (re)generate: a table of all mobile-kit agents with their spawn type (`mobile-kit:<name>`), one-line role, and project memory directory (`.claude/agent-memory/mobile-kit-<name>/`); the list of `/mobile-kit:*` skills; and a short "how to override an agent per project" note (copy to `.claude/agents/<name>.md`; a local override uses the unprefixed memory dir). This restores at-a-glance visibility of the agent suite inside the project tree. Derive names/roles from the installed plugin's `agents/` directory rather than hardcoding, so the roster stays correct as the plugin evolves. Optionally offer the user a browse symlink (`.claude/mobile-kit-agents` → the plugin repo's or installed plugin's `agents/` directory) — machine-local, so only if `.claude/` is not committed or the user opts in.

Everything you write must be verifiable from the codebase. Mark anything uncertain with `<!-- VERIFY: ... -->` and list those items in your final report.

## Step 4: Scaffold agent memory

Create empty directories (no files — memory grows organically). Plugin agents resolve their `memory: project` directory under a plugin-prefixed name:

```
.claude/agent-memory/{mobile-kit-mobile-planner,mobile-kit-mobile-developer,mobile-kit-bug-fixer,mobile-kit-test-writer,mobile-kit-qa-reviewer,mobile-kit-tech-lead,mobile-kit-code-optimizer,mobile-kit-design-analyzer,mobile-kit-design-system-guardian,mobile-kit-release-notes-writer}/
```

If the directories already exist, leave them completely untouched.

If the project has pre-plugin memory directories with unprefixed names (e.g. `.claude/agent-memory/tech-lead/` from a local-agent setup), do NOT move them yourself — tell the user they can migrate that memory with `git mv .claude/agent-memory/<name> .claude/agent-memory/mobile-kit-<name>` once they retire the local agent definitions.

## Step 5: Register the marketplace in project settings

Merge into `.claude/settings.json` (create if missing, preserve existing keys):

```json
{
  "extraKnownMarketplaces": {
    "mobile-agent-kit": {
      "source": { "source": "github", "repo": "ChristianEichmueller/mobile-agent-kit" }
    }
  }
}
```

## Step 6: Wire up CLAUDE.md

Append to the project's `CLAUDE.md` (create if missing) a short orchestration section — adapt, don't overwrite existing content:

```markdown
## Agent Orchestration (mobile-kit plugin)

Agents and workflows come from the mobile-kit plugin. Project specifics live in `.claude/docs/PROJECT_CONTEXT.md`.

- New features / refactoring / migrations → `/mobile-kit:orchestrate <task>`
- Bugs / crashes / regressions → `/mobile-kit:bug-hunt <bug description>`
- Review current changes until clean → `/mobile-kit:review-loop`
- Implement an existing plan phase-by-phase → `/mobile-kit:implement-plan <plan-path>`

Tests are written BEFORE implementation (TDD). Implementers may not modify tests; disagreements go through the `## Developer Test Concern` push-back protocol.
```

If CLAUDE.md already references local copies of these skills/agents (a pre-plugin setup), point that out to the user instead of editing those references silently.

## Step 7: Report

Present to the user:
- Detected stack and the evidence
- Files created/updated (context pack, settings, CLAUDE.md)
- Every `<!-- VERIFY: ... -->` item that needs their confirmation
- Any REQUIRED contract section you could not fill (e.g. no test infrastructure exists yet) — with a recommendation
- Reminder: project-specific overrides are possible by placing a same-named agent in `.claude/agents/`

Do NOT commit anything — the user reviews and commits.
