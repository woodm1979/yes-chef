# blueprint

Planning and execution suite for Claude Code. Provides a PRD+PLAN artifact workflow for persistent, resumable multi-section plans.

Pairs with [`woodm1979/less-opinionated-superpowers`](https://github.com/woodm1979/less-opinionated-superpowers) for full workflow coverage (TDD, debugging, code review, git worktrees, etc.).

## Installation

### Local (from this repo)

```bash
claude plugin marketplace add ./
claude plugin install blueprint
```

### From GitHub

```bash
claude plugin marketplace add woodm1979/blueprint
claude plugin install blueprint
```

## Skill Catalog

| Skill | Description |
|-------|-------------|
| `blueprint:brainstorm` | Structured interview → decision summary → artifact gate. Primary entry point. No files written. |
| `blueprint:blueprint` | Reads brainstorm context → writes and commits PRD.md + PLAN.md. Handles new features and extensions. |
| `blueprint:build` | Executes sections from a PLAN file — runs one section at a time with per-section subagents and 2-stage review. |
| `blueprint:tdd` | Red-green-refactor TDD loop. Required by `build`. |

## Workflow

```
/brainstorm  →  shared understanding
/blueprint   →  PRD.md + PLAN.md artifacts committed
/build       →  execute one section at a time
```

## License

MIT — see LICENSE file.
