# Devin first-class Trellis platform

## Goal

Land the `.devin/` "Trellis first-class platform" prototype on `main`: hook-driven context injection plus custom sub-agent profiles, so Trellis task context reaches Devin sub-agents without manual loading.

## Requirements

- Register a `PreToolUse` hook on `run_subagent` that rewrites `tool_input.task` for `trellis-implement` / `trellis-check` / `trellis-research`, prepending the `<!-- trellis-hook-injected -->` marker and injecting task artifacts (prd/design/implement + jsonl context for task-requiring agents, spec tree for research).
- Register a `UserPromptSubmit` hook emitting the `<workflow-state>` breadcrumb each turn.
- Ship `.devin/agents/trellis-{implement,check,research}.md` custom profiles pinned to `model: swe-2-max`; trellis-research detects context source via the marker and reports `Context: hook-injected` / `self-loaded`.
- `trellis-start.md` documents that SessionStart context is now hook-injected.
- Hooks fail open (exit 0, no output) on non-Trellis repos, missing task state, or unsupported profiles; built-in profiles pass through unwrapped.
- All four wrapper builders (`implement`/`check`/`finish`/`codex`/`research`) emit the marker as the first line — consistent self-detection for child agents.

## Acceptance Criteria

- [x] `trellis-research` dispatch: received task line 1 = `<!-- trellis-hook-injected -->`, caller's first line lands under `## Your Task`, report opens with `Context: hook-injected`.
- [x] `swe-2-max` resolves: no `subagent-free-default` fallback ERROR on trellis-profile spawn; no `load_custom_subagents` warnings.
- [x] Regression: `subagent_explore` receives task text unwrapped; `exec` normal; `<workflow-state>` breadcrumb present.
- [ ] Changes committed on `task/09-18-devin-trellis-platform` and landed via PR.

## Notes

- Verified 2026-09-18 against `~/.devin-server` logs (session `quill-supply`, log dir `20260918T203621`).
- Upstream follow-up: consider porting the Devin wiring back into `templates/` (`devin.ts`, shared-hooks table).
