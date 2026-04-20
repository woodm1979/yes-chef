# PLAN: Brainstorm Plan-Mode Compatibility

> PRD: ./2026-04-19-brainstorm-plan-mode-compat-PRD.md
> Executor: /build
> Created: 2026-04-19  |  Last touched: 2026-04-19

## Architectural decisions

- Only `skills/brainstorm/SKILL.md` is modified — no other skills are touched.
- The `Skill` tool invocation in Step 3 is removed entirely; the post-decision action becomes a text instruction to the user.
- The compatibility note is added as a standalone section near the top of the skill (after the suite context table, before UX rules) so it is read early in any invocation context.
- The AskUserQuestion call at the artifact gate is retained — only the Skill tool dispatch is removed.

## Conventions

- TDD per section (test → impl → commit)
- Minimum one commit per completed section
- Review checkpoint between sections (spec compliance + code quality)
- Default implementer model: `sonnet`

---

## Section 1: Make brainstorm plan-mode compatible

**Status:** [x] complete
**Model:** haiku
**User stories covered:** 1, 2

### What to build

Edit `skills/brainstorm/SKILL.md` to (a) remove the `Skill` tool invocation from Step 3's artifact gate and replace it with a text instruction directing the user to run `/blueprint` manually, and (b) add a plan-mode compatibility note near the top of the file stating that all brainstorm steps are read-only.

### Acceptance criteria

- [x] When the user picks "Write new PRD + PLAN" or "Extend existing PRD + PLAN" at the artifact gate, the skill no longer calls the `Skill` tool — instead it outputs a message telling the user to invoke `/blueprint`
- [x] The artifact gate still uses `AskUserQuestion` with the same four options (Write new, Extend existing, No files, Let's discuss)
- [x] A plan-mode compatibility note appears in the skill before the UX rules section, stating that all steps are read-only and the skill should proceed normally when plan mode is active
- [x] The "Let's discuss" path still works: discuss inline, then re-ask via `AskUserQuestion`
- [x] The "No files — end here" path still works: end cleanly with no skill invocation
- [x] No other sections of the skill are modified

### Notes for executor

- The file to edit is `skills/brainstorm/SKILL.md` in the repo root (source file, not the plugin cache copy at `~/.claude/plugins/cache/`)
- The compatibility note should be specific: name the three steps, confirm all are read-only, and explicitly state that Claude should NOT switch to plan-mode behavior when this skill is active
- The text instruction replacing the Skill invocation should name `/blueprint` explicitly so the user knows exactly what to type

### Completion log

- Commits: 378efd9
- Tests added: 0
- Deviations from plan: none
