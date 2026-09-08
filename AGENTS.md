<!-- TRELLIS:START -->
# Trellis Instructions

These instructions are for AI assistants working in this project.

This project is managed by Trellis. The working knowledge you need lives under `.trellis/`:

- `.trellis/workflow.md` — development phases, when to create tasks, skill routing
- `.trellis/spec/` — package- and layer-scoped coding guidelines (read before writing code in a given layer)
- `.trellis/workspace/` — per-developer journals and session traces
- `.trellis/tasks/` — active and archived tasks (PRDs, research, jsonl context)

If a Trellis command is available on your platform (e.g. `/trellis:finish-work`, `/trellis:continue`), prefer it over manual steps. Not every platform exposes every command.

If you're using Codex or another agent-capable tool, additional project-scoped helpers may live in:
- `.agents/skills/` — reusable Trellis skills
- `.codex/agents/` — optional custom subagents

Managed by Trellis. Edits outside this block are preserved; edits inside may be overwritten by a future `trellis update`.

<!-- TRELLIS:END -->

# Git workflow

`main` is protected by a GitHub ruleset (pull request required, `analyze-and-test` must pass, no force-push or deletion) and must stay releasable at every commit. Squash and rebase merges are disabled on the repository; only merge commits land.

- Never commit or push on `main`. Code and `.trellis/` bookkeeping alike reach `main` through a pull request. If you find uncommitted work on `main`, move it first: `git switch -c task/<...>` keeps the working tree.
- One branch per **leaf** task, named after its task directory: `task/<MM-DD-slug>` (e.g. `task/09-07-dependency-toolchain-health`). Parent tasks get no branch. A child too large for one PR is split by `implement.md` stage: `task/<MM-DD-slug>-<stage>` — hyphen, not slash, so the names cannot collide as refs.
- Order matters: `git switch -c task/<...>` first, then `python3 ./.trellis/scripts/task.py start <task>`. `start` records the checked-out branch in `task.json`; `archive` refuses a task whose `branch` equals `base_branch`.
- One commit per `implement.md` checkbox, using the commit message written there. No `wip` snapshot commits; fix up partial commits on the task branch before the PR.
- Before opening the PR: `trellis-check` passes, `git diff --check` is clean, and the branch is rebased onto `main` (`git rebase main` — a task branch belongs to one agent, rebasing it is safe). Resolve `pubspec.lock` / `Cargo.lock` conflicts by regenerating (`flutter pub get`, `cargo update`), never by hand.
- Finish the task on its branch: `add_session.py` (journal) and `task.py archive` are the branch's last commits and ride the same PR.
- The journal is append-only and `journal-*.md` merges with `merge=union`, which **interleaves** two sessions that were appended in parallel (common lines such as `### Git Commits` line up and the bodies cross). After every rebase that touched `.trellis/workspace/`, rebuild the file as `main`'s copy plus your own session block appended, re-point the block's commit hashes at the rebased commits, and check `rg -n '^## Session' journal-1.md` is strictly increasing before pushing. `index.md` conflicts are resolved the same way: `main`'s rows plus yours on top.
- `gh pr create --fill`, then `gh pr merge --merge` once CI is green. Roll back one step with `git revert <sha>`; roll back a whole child with `git revert -m 1 <merge-sha>`. Never `git reset --hard` or `git clean` on shared state.
- Parallel work means parallel checkouts: one agent per `git worktree` (`git worktree add ../Pixiv-func-<slug> task/<...>`). Two agents in one working tree is not allowed.
- Releases are tags on `main` (`vX.Y.Z`; `release.yml` is dispatched manually). A hotfix branches from the tag as `hotfix/X.Y.Z`, is tagged, and is merged back into `main`.
- `baseline-2026-09-07` tags the tree the 09-02 research counts were measured on; recount against HEAD before implementing.
