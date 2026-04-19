---
name: add-to-menu
description: Use when extending an existing MENU+RECIPES pair with new sections. Part of the yes-chef suite. Shortcut into /plan-meal's extend-existing path. Pairs with /cook (executor).
---

# /add-to-menu

Shortcut into `/plan-meal`'s extend-existing path. Use when you already have a MENU+RECIPES pair in `docs/ai-plans/` and want to add more work to it — not start a new feature.

## REQUIRED BACKGROUND

**REQUIRED SKILL:** `/plan-meal` — contains the full interview protocol, UX rules (one-question-per-turn, `AskUserQuestion` with `"Let's discuss"`, tradeoff tables), MENU/RECIPES file formats, self-review checklist, and model-selection guidance. Read it before running this skill.

This skill inherits every UX rule from `/plan-meal`. The ONLY difference is the starting branch: `/add-to-menu` pre-selects "extend existing" so you skip the new-vs-extend question.

## Precedence

If a repo's `CLAUDE.md`, `AGENTS.md`, or explicit user instructions conflict with this skill, user instructions win. This skill's rules apply only where the user hasn't overridden them. Read those files before loading the target RECIPES.

## Core discipline — APPEND, do not fork

**When the user asks to extend existing work, append new sections to the existing RECIPES. Do NOT create a new `-v2-` file.**

- **Primary path:** load existing MENU + RECIPES, append `## Section N+1: ...` (and `N+2`, etc.) to the SAME `-RECIPES.md` file. If user stories or architectural decisions shift, edit the SAME `-MENU.md` file in place. Bump `Last touched:` on both.
- **New-file path:** only when the existing work is truly a different feature (no meaningful overlap in user stories, architecture, or module boundaries). In that case, STOP and suggest the user invoke `/plan-meal` to start a fresh MENU+RECIPES pair.

### Why append, not fork

Forking into `-v2-` files (or `-part-2-`, or `-<today>-<slug>-MENU.md`) fragments the execution history. `/cook` reads ONE RECIPES file at a time; splitting sections across files breaks continuity, hides the already-shipped context from future sessions, and makes cross-session resumption ambiguous. The MENU is the living contract for the feature; the RECIPES is the living queue. Neither is a dated snapshot.

If the shipped sections feel "frozen" as a historical record, that's fine — they stay in place with their `[x] complete` checkboxes and completion logs. New sections tacked on below don't disturb them.

## Process

### Step 1 — Locate the target RECIPES file

In order:

1. **If the user `@`-referenced a plan file** in their message, use it.
2. **If there's exactly one `docs/ai-plans/*-RECIPES.md` file**, use it.
3. **If there are multiple candidates**, call `AskUserQuestion` with one option per recent RECIPES file (up to 3) plus `"Let's discuss"`. Order by most-recently-modified first.
4. **If `docs/ai-plans/` doesn't exist or has no RECIPES file**, STOP. Tell the user there's nothing to extend and suggest `/plan-meal` for new work.

### Step 2 — Load MENU + RECIPES into context

Read both files. Confirm:

- The MENU's `Plan:` header points to the RECIPES you found (or vice versa).
- The RECIPES has at least one existing `## Section N: ...` block.

If either file is malformed or the pair is inconsistent, stop and tell the user what's wrong before proceeding.

### Step 3 — Scope check (escape hatch)

Before grilling on the delta, ask yourself: is this really an extension, or is it actually a new feature?

**Signals it's a new feature** (and `/add-to-menu` is the wrong skill):

- No overlap in user stories between what the user is asking for and what the MENU describes.
- The request introduces a new architectural boundary that doesn't fit the existing module sketch.
- The request is of comparable scope to the original MENU (not a delta — a sibling).

If you see these signals, STOP. Tell the user:

> "This looks more like a new feature than an extension of `<existing MENU>`. Consider invoking `/plan-meal` to start a fresh MENU+RECIPES pair. If you'd like to proceed anyway, say so explicitly."

Only continue when the user explicitly insists on extending.

### Step 4 — Grill on the delta

Following ALL `/plan-meal` UX rules (one question per turn, `AskUserQuestion` with `"Let's discuss"`, tradeoff tables before non-obvious decisions):

1. **What's the new work?** (Usually clear from the request.) Confirm your understanding in one turn.
2. **Does it invalidate existing user stories?** If so, edit the MENU in place.
3. **Does it change the architecture sketch?** If so, edit the MENU's "Architecture & module sketch" in place. Tradeoff table required if there's a real decision to make.
4. **Does it introduce new out-of-scope items?** Update the MENU's "Out of scope".
5. **What new sections are needed?** Propose them as tracer-bullet vertical slices, with `User stories covered:` and `Suggested Model:` per section, just like `/plan-meal` Step 5. Iterate until the user approves.

### Step 5 — Edit MENU in place (if needed)

If any MENU section changed (user stories, architecture, out of scope), edit the SAME `-MENU.md` file:

- Bump `Last touched:` to today's date.
- Append new user stories at the end of the list with the next available number — do NOT renumber existing stories (that breaks cross-references in shipped RECIPES sections).
- Add to architecture sketch with a `### Added <date>:` sub-heading if the change is substantive.

### Step 6 — Append sections to RECIPES

Edit the SAME `-RECIPES.md` file:

- Bump `Last touched:` to today's date.
- Append new `## Section N+1:`, `## Section N+2:`, etc. using the RECIPES section template (`Status: [ ] not started`, `Model:`, `User stories covered:`, `What to build`, `Acceptance criteria`, `Notes for executor`, `Completion log`). Find the highest existing section number and continue from there.
- Do NOT modify existing sections (their status, acceptance criteria, or completion logs). Past sections are frozen history.
- If the new work genuinely depends on an existing section being different (e.g., "the shipped Section 2 needs a hook that wasn't there"), add the refactor as its own new section — do not edit history.

### Step 7 — Self-review pass

Same checklist as `/plan-meal` Step 7: spec coverage, placeholder scan, type consistency, vertical-slice check, model coherence. Plus one extra check for this skill:

7. **Append-not-fork check.** Are you about to create a new file in `docs/ai-plans/` instead of editing the existing pair? If yes, stop — that's the anti-pattern this skill exists to prevent. Go back to Step 6.

Fix issues inline; no re-review needed.

### Step 8 — Commit

Single commit including the MENU and RECIPES edits. Message format: `Plan: extend <feature name> (+Section N+1...)`. No attribution trailers.

### Step 9 — Handoff

End with exactly:

> **REQUIRED NEXT SKILL:** Invoke `/cook` to execute the new sections in `docs/ai-plans/<existing>-RECIPES.md`. `/cook` will pick up at the first `[ ] not started` section.

Do not invoke `/cook` yourself.

## Rationalization table

| Excuse | Reality |
|---|---|
| "Shipped sections should be frozen, so new work needs its own file" | Shipped sections ARE frozen — their checkboxes are `[x]` and their completion logs are filled in. Appending new `[ ] not started` sections doesn't disturb them. One file per feature, forever. |
| "A new file dated today is cleaner" | Filesystem cleanliness is not the goal. Execution continuity is. `/cook` follows ONE RECIPES file; splitting breaks cross-session resume. |
| "The filename carries a creation date, so new work deserves a new filename" | The filename is the feature's birth date, not a version tag. `Last touched:` in the header IS the versioning signal — that's what changes when work is extended. The filename never changes over the feature's lifetime. |
| "The new feature is big enough to deserve its own plan" | Then it's a new feature. STOP and suggest `/plan-meal`. Don't split the difference with `-v2-`. |
| "The existing MENU is shipped, editing feels weird" | The MENU is a living contract for the feature as it evolves. Shipped ≠ frozen; the feature keeps existing. Edit in place; bump `Last touched:`. |
| "I'll renumber user stories so the new ones are grouped" | No — existing RECIPES sections reference story numbers. Renumbering breaks history. Append with the next number. |
| "I'll edit the shipped section to add what we now need" | No — shipped sections are frozen history. If the shipped code needs to change, that's a NEW section (a refactor section). |

## Red flags — STOP

- About to call `Write` on a path like `docs/ai-plans/<today>-<slug>-v2-MENU.md` or similar "v2" / "part 2" / dated-today-when-one-already-exists → STOP. Use the existing file.
- About to modify an existing `## Section N:` block that already has `[x] complete` → STOP. Add a new section.
- About to renumber existing user stories in the MENU → STOP. Append.
- Handoff that doesn't name `/cook` → rewrite.
- Skipping Step 3 scope check because "the user said extend, so it is" → do the check anyway. The scope check is cheap insurance.

## When NOT to use

- No existing MENU+RECIPES pair → use `/plan-meal` (new-feature path).
- The request has zero overlap with any existing MENU → use `/plan-meal` (new-feature path).
- User wants to add a single bullet to the open-questions list → just edit the MENU directly; no full skill invocation needed.
