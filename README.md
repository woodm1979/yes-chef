# yes-chef

Chef-themed planning and execution suite for Claude Code. Provides a MENU+RECIPES artifact workflow for persistent, resumable multi-section plans.

Pairs with [`woodm1979/less-opinionated-superpowers`](https://github.com/woodm1979/less-opinionated-superpowers) for full workflow coverage (TDD, debugging, code review, git worktrees, etc.).

## Installation

### Local (from this repo)

```bash
claude plugin marketplace add ./
claude plugin install yes-chef
```

### From GitHub

```bash
claude plugin marketplace add woodm1979/yes-chef
claude plugin install yes-chef
```

## Skill Catalog

| Skill | Description |
|-------|-------------|
| `yes-chef:plan-meal` | Interactive interview → MENU + RECIPES artifacts committed to `docs/ai-plans/`. Offers grill mode (no artifact) and optional GitHub-issue output. |
| `yes-chef:plan-menu` | Alias for `plan-meal`. |
| `yes-chef:cook` | Executes sections from a RECIPES plan file — runs one section at a time with per-section subagents and 2-stage review. |
| `yes-chef:add-to-menu` | Extends an existing MENU+RECIPES pair with new sections. Shortcut into `plan-meal`'s extend-existing path. |
| `yes-chef:add-to-meal` | Alias for `add-to-menu`. |
| `yes-chef:tdd` | Red-green-refactor TDD loop. Required by `cook`. |

## Workflow

```
/plan-meal   →  MENU + RECIPES artifacts
/cook        →  execute one section at a time
/add-to-menu →  extend existing plan with new sections
```

## License

MIT — see LICENSE file.
