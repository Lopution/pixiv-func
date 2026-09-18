# Devin Trellis sub-agent dispatch mode

## Goal

Switch Devin's Trellis working mode from inline (skill-loading via `trellis-before-dev` / `trellis-check` skills in the main session) to sub-agent dispatch (`run_subagent` with the `trellis-implement` / `trellis-check` / `trellis-research` profiles landed in 09-18-devin-trellis-platform).

## Requirements

- `workflow.md`: move `Devin` out of the five `[codex-inline, Kilo, Antigravity, Devin, DeepSeek Harness]` blocks into the matching dispatch blocks — Active Task Routing, research spawn, 1.3 context curation, 2.1 implement dispatch, 2.2 check dispatch. For implement, Devin joins the hook-injection variant (`[Claude Code, Cursor, OpenCode, codex-sub-agent, CodeBuddy, Droid, Pi, ZCode, Snow, Oh My Pi]`), since the Devin `PreToolUse` hook rewrites `task` exactly like that group's platform prelude.
- `workflow.md`: add `Devin` to the step-1.3 sub-agent-dispatch platform list, and document the Devin dispatch mechanism in the sub-agent dispatch protocol paragraph (`run_subagent` with `profile` set to the Trellis agent name).
- `task_store.py`: add `.devin` to `_SUBAGENT_CONFIG_DIRS` and update the two comments that classify Devin as skill-loading — Devin now consumes `implement.jsonl` / `check.jsonl` through hook injection.
- Breadcrumb `[workflow-state:*]` blocks need no change: generic `in_progress`/`planning` blocks already describe dispatch flow; `-inline` variants are `codex.dispatch_mode`-gated only.
- No changes to `.devin/agents/*` profiles — they already implement the dispatch side.

## Acceptance Criteria

- [x] `get_context.py --mode phase` for Devin surfaces dispatch guidance — 2.1 shows "Spawn the implement sub-agent" with the hook-injection prelude block; 2.2 shows "Spawn the check sub-agent". `trellis-start.md` routing table updated to `run_subagent` routes.
- [x] `task.py create` on a `.devin`-only layout would produce jsonl manifests (`.devin` added to `_SUBAGENT_CONFIG_DIRS`; this repo already has other agent dirs so behavior is unchanged here).
- [x] Python files compile; `git diff --check` clean.
- [ ] Committed on `task/09-18-devin-subagent-mode`, landed via PR.

## Notes

- Upstream counterparts (`templates/`, `src/types/ai-tools.ts` agent-capable list, `test/registry-invariants.test.ts`) need the same flip when this wiring is ported back; recorded as follow-up.
