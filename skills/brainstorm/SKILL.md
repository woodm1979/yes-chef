---
name: brainstorm
description: Open-ended exploration and Q&A about a feature or problem. No artifacts produced. Primary entry point for the blueprint suite — leads into /blueprint for committed artifacts.
---

# /brainstorm

## Overview

Primary intake skill of the blueprint suite. Interviews the user through a structured dialogue, reaches shared understanding, then asks what to do with it. Produces no artifacts itself — artifacts come from `/blueprint`.

## Suite context

| Skill | Role |
|---|---|
| `/brainstorm` (this skill) | Intake — structured interview, decision summary, artifact gate |
| `/blueprint` | File-creation — writes PRD.md + PLAN.md from brainstorm context |
| `/build` | Executor — runs each section in a fresh subagent with 2-stage review |

## UX rules (non-negotiable)

These rules apply to EVERY user-facing message during intake. Violating any of them means the skill has been violated.

1. **One question per turn.** If a topic needs multiple questions, split them into multiple turns. Never bundle.
2. **Use `AskUserQuestion` for any decision with 2–4 known discrete options.** Never hand-roll a numbered list of choices in prose when `AskUserQuestion` would fit.
3. **Every `AskUserQuestion` call MUST include an option whose `label` is exactly `"Let's discuss"`.** This is the escape hatch when the primary options don't fit. The label is literal — not "Something else", not "Other", not "None of these". Exactly `"Let's discuss"`.
4. **Lead with a tradeoff table before any non-obvious decision.** A decision is "non-obvious" if a competent engineer could reasonably pick more than one option. The table precedes the `AskUserQuestion` call and uses the format:

   | Option | Pro | Con | When it fits |
   |---|---|---|---|

   Heuristic: **if you can write the table in under a minute, write it.** The only legitimate reason to skip is when one option is clearly dominant (and in that case, just make the decision and move on — don't ask).

5. **Use open prose (not multiple-choice) for problem framing and user stories.** These are exploratory by nature; structure emerges from dialogue.
6. **Never launch into a design proposal in turn 1.** Turn 1 is codebase recon. Always.

### Red flags — STOP and restart the message

- About to send a message with 2+ question marks in it → split.
- About to send a numbered list of A/B/C options in prose → switch to `AskUserQuestion`.
- About to send an `AskUserQuestion` call without a `"Let's discuss"` option → add it.
- About to announce a design before the user has been asked anything → back up.

## Process

### Step 1 — Codebase recon

Dispatch a single `Explore` subagent (model: `sonnet`) to answer: what's the current architecture, what patterns does the repo already use, and what existing modules will this feature likely touch or extend? Keep the subagent's report under 300 words.

Pass ONLY the user's original feature request to the subagent. Do not pass your own speculation; let the subagent observe cleanly.

### Step 2 — Interview

Following the UX rules above, ask — one turn at a time:

1. **Problem** (open prose) — "In your own words, what problem does this solve? Who feels the pain today?"
2. **Solution framing** (open prose) — "How do you see this working from the user's perspective? A couple of sentences is fine."
3. **User stories** — draft 2–5 stories in the format `As <actor>, I want <feature>, so <benefit>` and present them for critique in one message. Iterate until the user approves.
4. **Architecture & module sketch** — at every durable architecture/approach decision point (routes, schema, key models, auth boundaries, third-party integrations), **propose 2-3 approaches with a tradeoff table BEFORE asking the user to choose**:

   | Option | Pro | Con | When it fits |
   |---|---|---|---|
   | Approach A | … | … | … |
   | Approach B | … | … | … |
   | Approach C | … | … | … |

   Lead with your recommended option and reasoning, then `AskUserQuestion` with `"Let's discuss"`. Do NOT include file names or code — only module boundaries and interfaces. Apply YAGNI: if an approach is only justified by a speculative future need, cut it.

5. **Testing approach** — one turn: "What makes a good test for this? Any behaviors that absolutely must be covered?"
6. **Out of scope** — one turn: list the things you're explicitly NOT doing, and ask for confirmation or additions.
7. **Open questions** — capture anything still unresolved.

### Step 3 — Decision summary

Give a 3–5 bullet summary of the key decisions reached during the interview. Keep it tight — one line per decision.

### Step 4 — Artifact gate

Call `AskUserQuestion`:

> Question: "What should we do with this?"
>
> Options:
> - `Write new PRD + PLAN` — Invoke `/blueprint` to write fresh `docs/ai-plans/<date>-<slug>-PRD.md` + `PLAN.md` from this conversation.
> - `Extend existing PRD + PLAN` — Invoke `/blueprint`; it will detect the matching pair and append new sections.
> - `No files — end here` — The summary above is the output. Nothing is written or committed.
> - `Let's discuss` — Continue the conversation inline, then re-ask.

If the user chooses **Write new PRD + PLAN** or **Extend existing PRD + PLAN**, end with:

> **REQUIRED NEXT SKILL:** Invoke `/blueprint`. It will read this conversation's context and handle writing or extending the plan files.

If **No files — end here**, end cleanly. The decision summary is the output.

If **Let's discuss**, discuss inline, then call `AskUserQuestion` again with the same four options.

## Precedence

If a repo's `CLAUDE.md`, `AGENTS.md`, or explicit user instructions conflict with this skill, user instructions win. Read those files before starting the interview.

## Red flags — STOP

- More than one `?` in a single user-facing message
- Inline numbered options instead of `AskUserQuestion`
- `AskUserQuestion` without a `"Let's discuss"` option
- Architecture decision asked without a preceding 2-3-approach tradeoff table
- Designing a proposal in the first message before asking anything
- Writing any file, making any commit, or creating any GitHub issue

## When NOT to use

- User wants to skip interview and write plan files directly → use `/blueprint` with the feature described inline.
- Task is a one-line bugfix or chore → just fix it.
- User wants to execute an existing PLAN file → use `/build`.
