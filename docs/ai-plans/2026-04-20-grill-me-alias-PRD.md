# PRD: grill-me Skill Alias

> Status: draft
> Plan: ./2026-04-20-grill-me-alias-PLAN.md
> Created: 2026-04-20  |  Last touched: 2026-04-20

## Problem

No `/grill-me` skill exists in the blueprint plugin. Users who prefer a more visceral name for the brainstorm interview — or who have muscle memory for "grill me" — have no alias to invoke. They must know the skill is called `/brainstorm` to use it.

## Solution

Add `skills/grill-me/SKILL.md` as a thin redirect skill. The file contains a minimal frontmatter block (name, description) and a single instruction telling Claude to read `../brainstorm/SKILL.md` and follow those instructions exactly. The relative path is version-stable: both skills live in the same plugin directory, so the path holds across plugin version bumps.

## User stories

1. As a user, I want to invoke `/grill-me` and get the full brainstorm interview loop, so I can use a more memorable name for the skill.

## Architecture & module sketch

- **`skills/grill-me/SKILL.md`** — minimal skill file: frontmatter with `name: grill-me` and a description marking it as a brainstorm alias, plus a redirect instruction pointing to `../brainstorm/SKILL.md`

## Testing approach

- Manual smoke test: invoke `/grill-me` after adding the skill and verify the brainstorm interview loop runs (relentless Q&A, artifact gate at the end).
- No automated tests — this is an LLM skill file, not code.

## Out of scope

- Any behavior divergence between `/grill-me` and `/brainstorm`.
- Changes to `skills/brainstorm/SKILL.md` or any other skill.

## Open questions

- None.
