# PRD: Worktree-per-feature Parallel Isolation

> Status: draft
> Plan: ./2026-04-23-worktree-per-feature-PLAN.md
> Created: 2026-04-23  |  Last touched: 2026-04-23

## Problem

Running multiple `/build` executions in parallel on the same repo causes agents to step on each other's files and git state. The blueprint suite currently assumes a single active build at a time — the moment two agents run `/build` simultaneously, they share the same working tree, the same branch, and the same PLAN file. Commits collide, section checkboxes race, and the result is an unpredictable mess. `afk-build.sh` (the shell-script entry point for background builds) has no isolation mechanism either.

Developers who want to work on multiple features simultaneously have no supported path: they must either serialize their builds or maintain multiple clones of the repository — neither of which scales. The parallel-agent promise of the blueprint suite cannot be delivered without filesystem isolation.

## Solution

Embed a `Worktree:` line in the PLAN header at `/blueprint` time. The value is the absolute path of a dedicated git worktree created specifically for this feature. `/build` reads the field and enters the worktree before doing any work; `afk-build.sh` does the same for background builds. A new `/cleanup-worktree` skill handles safe teardown with guards against data loss.

The worktrees are created by a `WorktreeCreate` hook shipped with the blueprint plugin. The hook handles branch creation, environment file symlinking, `.claude/` config symlinking, and language-specific dependency installation. Worktrees are placed at `<parent>/<repo>-worktrees/<slug>` — a sibling directory to the repo — so no `.gitignore` management is needed and the worktrees never appear in git status.

The entire worktree lifecycle (create, use, remove) is owned by the plugin, making the experience zero-config for users who install blueprint.

## User stories

1. As a developer, I want to run multiple `/build` executions in parallel without collision, so I can develop multiple features simultaneously.
2. As a developer, I want `/blueprint` to automatically create an isolated worktree for my feature, so I don't have to set up git isolation manually.
3. As a developer, I want `/build` to automatically enter the correct worktree, so all agent commits land on the right branch without manual intervention.
4. As a developer, I want `afk-build.sh` to run in the correct worktree, so background builds are also isolated.
5. As a developer, I want to safely remove a worktree when done, with guards that prevent me from losing uncommitted or unpushed work.
6. As a plugin user, I want the worktree hook to set up my development environment automatically (deps, config symlinks), so I can immediately start coding in the new worktree without manual setup.
7. As a plugin user, I want to provide a custom setup script for my repo's specific needs, so unusual toolchains, monorepos, and non-standard layouts are fully supported.

## Architecture & module sketch

- **`WorktreeCreate` hook** (`hooks/worktree-create.sh` + `scripts/worktree-create`) — Responds to `EnterWorktree name:` events. Creates worktree at `$(dirname $REPO_ROOT)/$(basename $REPO_ROOT)-worktrees/<name>`. If `.claude/worktree-setup.sh` exists in the repo, runs it exclusively (replaces auto-detection). Otherwise auto-detects language and installs deps: Elixir (mix), Node (npm/yarn/pnpm with hard-linked `node_modules`), Python (uv or pip), Rust (cargo fetch), Go (go mod download). Symlinks gitignored `.env*` files and untracked `.claude/` config files into the worktree.

- **`WorktreeRemove` hook** (`hooks/worktree-remove.sh` + `scripts/worktree-remove`) — Responds to `WorktreeRemove` events. Runs `git worktree remove --force` + `git worktree prune`. Returns the branch name on stdout for use by callers.

- **`/blueprint` skill** (Step 8.5 addition) — After committing the PRD+PLAN, determines the feature slug from the PLAN filename, sanitizes it (replaces `/` with `-`), and appends a counter suffix (`-2`, `-3`, …) if the branch already exists. Detects an existing worktree at the target path and prompts the user: reuse as-is, delete+recreate, or abort. On success, calls `EnterWorktree name: <slug>`, waits for the hook, and embeds the resolved absolute path as `Worktree: <abs-path>` in the PLAN header.

- **`/build` skill** (Step 1 addition) — Reads `Worktree:` from the PLAN header. If the path exists on disk, calls `EnterWorktree path: <abs-path>` to enter it. If the path is absent from disk, auto-recreates via `EnterWorktree name: <branch>`. If no `Worktree:` field is present, proceeds without worktree entry (backwards-compatible with pre-worktree PLANs). After entry, re-derives the PLAN file path as `<worktree-abs-path>/docs/ai-plans/<plan-filename>`.

- **`afk-build.sh`** — After locating the PLAN file, parses the `Worktree:` line from the header. If present and the directory exists, sets `WORK_DIR` to the worktree path and re-derives `PLAN_FILE` to the worktree's copy. If present but missing from disk, exits with repair instructions. Passes `WORK_DIR` to `docker sandbox run`. If no `Worktree:` field, `WORK_DIR` defaults to `.` (current behavior preserved).

- **`/cleanup-worktree` skill** (new) — Accepts a PLAN file path. Resolves the worktree path from the `Worktree:` field. Runs sanity checks in order: (1) block if worktree has uncommitted changes, (2) block if branch has commits not pushed to remote, (3) warn and require confirmation if branch is not merged into main. On confirmation, triggers `WorktreeRemove` for the worktree path. No annotation is written to the PLAN — stale `Worktree:` paths are an accepted artifact, consistent with other file references in PLANs.

- **`hooks/hooks.json`** — Plugin hook manifest wiring `WorktreeCreate` → `worktree-create.sh` and `WorktreeRemove` → `worktree-remove.sh`.

## Testing approach

Primarily manual end-to-end verification. Key scenarios to walk:

1. **Happy path — single feature**: Run `/blueprint` → confirm PLAN has `Worktree:` line and `<parent>/<repo>-worktrees/<slug>/` directory exists with correct branch checked out.
2. **Parallel isolation**: Open two editor windows, each pointing at a different worktree. Run `/build` in both simultaneously. Confirm commits land on separate branches and neither PLAN's checkboxes affect the other.
3. **`afk-build.sh` isolation**: Run `afk-build.sh path/to/PLAN.md` → confirm docker is invoked with the worktree directory, not `.`.
4. **Auto-recreate**: Delete the worktree directory manually, run `/build` again → confirm the worktree is recreated and build resumes correctly.
5. **Cleanup guards**: Run `/cleanup-worktree` with uncommitted changes → confirm it blocks. Run with unpushed commits → confirm it blocks. Run clean → confirm worktree is removed.
6. **Backwards compatibility**: Run `/build` against a pre-worktree PLAN (no `Worktree:` field) → confirm it runs in the main repo without error.

Stretch goal: a shell script `scripts/smoke-test-worktree.sh` that automates scenario 1 using a throwaway branch and fake changes, then cleans up.

## Out of scope

- PR creation, branch merging, or post-merge workflow automation
- CI/CD integration or remote worktree synchronization
- Multi-machine or shared-repo worktree management
- Windows or WSL compatibility
- GUI or TUI for worktree management
- Automatic modification of `~/.claude/settings.json` during plugin install (migration is documented, not automated)

## Open questions

- [ ] Does `EnterWorktree path: <abs-path>` (entering an existing worktree) scope all subsequent tool calls correctly without re-triggering `WorktreeCreate`? Needs verification against the Claude Code tool schema before `/build` Step 1 is implemented.
