# PRD: Brainstorm Plan-Mode Compatibility

> Status: draft
> Plan: ./2026-04-19-brainstorm-plan-mode-compat-PLAN.md
> Created: 2026-04-19  |  Last touched: 2026-04-19

## Problem

When a user invokes `/brainstorm` while plan mode is active, Claude abandons the grill-me interview entirely and switches to plan-mode behavior — writing a plan file and calling ExitPlanMode. The user loses the interactive intake experience they invoked and ends up with an unwanted plan artifact instead.

The root cause is Step 3 of the brainstorm skill: it invokes the `Skill` tool to trigger `/blueprint` automatically. Plan mode's system-reminder carries "supercedes any other instructions" language, which causes Claude to treat the entire brainstorm session as a plan-mode task — even though Steps 1 and 2 (grill-me interview and decision summary) are purely read-only operations that don't conflict with plan mode at all.

## Solution

Remove the `Skill` tool invocation from Step 3 of the brainstorm skill. Replace it with a text prompt telling the user to run `/blueprint` manually once the artifact gate decision is made. Add an explicit plan-mode compatibility note to the brainstorm SKILL.md so that Claude, upon reading the skill while plan mode is active, understands the skill is read-only and should not be abandoned in favor of plan-mode behavior.

This is a small UX tradeoff: users must type `/blueprint` themselves rather than having it auto-triggered. In exchange, brainstorm works correctly regardless of whether plan mode is active.

## User stories

1. As a user running `/brainstorm` while plan mode is active, I want the grill-me interview to proceed normally through all steps, so I can explore a feature without being forced to exit plan mode first.
2. As a user who reaches the artifact gate in brainstorm, I want a clear prompt telling me to run `/blueprint` manually, so I know exactly what to do next.

## Architecture & module sketch

- **brainstorm SKILL.md — Step 3 (artifact gate)** — remove `Skill` tool invocation; replace the post-decision action with a text instruction directing the user to run `/blueprint`
- **brainstorm SKILL.md — compatibility note** — add a new top-level note stating that all brainstorm steps are read-only and the skill is compatible with plan mode

## Testing approach

- Testing is behavioral — invoke the skill and observe Claude's behavior
- Key scenario 1: run `/brainstorm` with plan mode active; verify the grill-me interview (Step 1) proceeds normally and is not abandoned
- Key scenario 2: reach the artifact gate (Step 3) with plan mode active; verify Claude displays the text prompt to run `/blueprint` rather than invoking the Skill tool
- Key scenario 3: run `/brainstorm` without plan mode; verify behavior is unchanged (grill-me proceeds, artifact gate prompts user to run `/blueprint`)
- No automated test suite exists for skill files; all verification is manual

## Out of scope

- Changes to `/blueprint`, `/build`, or any other skill
- Any automated testing infrastructure for skill files
- Passing file-path hints from brainstorm to `/blueprint` (separate feature)

## Open questions

- (none)
