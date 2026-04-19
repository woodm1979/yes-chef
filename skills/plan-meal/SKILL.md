---
name: plan-meal
description: Use when you want both a PRD and a plan as persistent committed artifacts in docs/ai-plans/. Part of the yes-chef suite. Pairs with /cook (executor). Also offers grill mode (relentless interview, no artifact) and optional GitHub-issue output.
---

# /plan-meal

*"Mise en place: everything in its place, before service begins."*

## Overview

Primary intake skill of the yes-chef suite. Interviews the user and produces two committed artifacts under `docs/ai-plans/`:

- **MENU** — the PRD: problem, solution, user stories, architecture sketch, testing approach, out-of-scope, open questions.
- **RECIPES** — the plan: architectural decisions + a series of tracer-bullet vertical-slice sections, each with acceptance criteria, implementer-model guidance, and an empty completion log.

After the two files are written, committed, and self-reviewed, hand off to `/cook` for section-by-section execution with fresh-context subagents.

This skill also offers two additional paths selected at Step 1:

- **Grill mode** — relentless one-question-at-a-time interview that stress-tests an idea to shared understanding, producing NO artifact.
- **GitHub issue output** — after MENU+RECIPES are committed, optionally file a GitHub issue using the PRD content as its body.

## Suite context

| Skill | Role |
|---|---|
| `/plan-meal` (this skill) | Intake — writes MENU + RECIPES (also grill mode + optional GH issue) |
| `/plan-menu` | Pure alias for `/plan-meal` |
| `/add-to-menu` | Shortcut into this skill's *extend-existing* path |
| `/add-to-meal` | Pure alias for `/add-to-menu` |
| `/cook` | Executor — runs each section in a fresh subagent with 2-stage review |

## Embedded principles

This skill incorporates two sets of patterns from its lineage:

- **Interview patterns** — one question at a time, propose 2–3 approaches with tradeoff tables, YAGNI, no design proposals in turn 1.
- **Self-review checklist** — spec coverage, placeholder scan, type consistency (see Step 7).

These are fully built in. No external skill load is required.

## Precedence

If a repo's `CLAUDE.md`, `AGENTS.md`, or explicit user instructions conflict with this skill, user instructions win. This skill's rules apply only where the user hasn't overridden them. Read those files before starting the interview.

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
6. **Never launch into a design proposal in turn 1.** Turn 1 is the new-vs-extend (vs grill-mode) decision. Always.

### Red flags — STOP and restart the message

- About to send a message with 2+ question marks in it → split.
- About to send a numbered list of A/B/C options in prose → switch to `AskUserQuestion`.
- About to send an `AskUserQuestion` call without a `"Let's discuss"` option → add it.
- About to announce a design before the user has been asked anything → back up.

## Process

### Step 1 — First decision: new vs extend vs grill

BEFORE anything else (before reading the repo, before restating the user's request), call `AskUserQuestion`:

> Question: "Is this a new feature, an extension of existing work, or do you just want to be grilled?"
>
> Options:
> - `New feature` — I'll write a fresh MENU and RECIPES.
> - `Extend existing` — I'll load an existing MENU+RECIPES and append new sections. (You may also invoke `/add-to-menu` directly to skip this question.)
> - `Grill mode` — Interview me relentlessly about my plan without producing any artifact.
> - `Let's discuss` — the situation doesn't fit any of these.

If **Extend existing**, stop here and invoke `/add-to-menu`.

If **Grill mode**, jump to the **Grill mode** section below. Do NOT continue to Step 2.

Otherwise (New feature), continue.

### Step 2 — Codebase recon (one subagent, sonnet)

Dispatch a single `Explore` subagent (model: `sonnet`) to answer: what's the current architecture, what patterns does the repo already use, and what existing modules will this feature likely touch or extend? Keep the subagent's report under 300 words.

Pass ONLY the user's original feature request to the subagent. Do not pass your own speculation; let the subagent observe cleanly.

### Step 3 — Interview

Following the UX rules above, ask — one turn at a time:

1. **Problem** (open prose) — "In your own words, what problem does this solve? Who feels the pain today?"
2. **Solution framing** (open prose) — "How do you see this working from the user's perspective? A couple of sentences is fine."
3. **User stories** — draft 2–5 stories in the format `As <actor>, I want <feature>, so <benefit>` and present them for critique in one message. Iterate until the user approves.
4. **Architecture & module sketch** — at every durable architecture/approach decision point (routes, schema, key models, auth boundaries, third-party integrations), **propose 2-3 approaches with a tradeoff table BEFORE asking the user to choose**. This is the "Exploring approaches" pattern inherited from brainstorming. Use this format:

   | Option | Pro | Con | When it fits |
   |---|---|---|---|
   | Approach A | … | … | … |
   | Approach B | … | … | … |
   | Approach C | … | … | … |

   Lead with your recommended option and your reasoning, then `AskUserQuestion` with `"Let's discuss"`. Do NOT include file names or code — only module boundaries and interfaces. Apply YAGNI: if an approach is only justified by a speculative future need, cut it.

5. **Testing approach** — one turn: "What makes a good test for this? Any behaviors that absolutely must be covered?"
6. **Out of scope** — one turn: list the things you're explicitly NOT doing, and ask for confirmation or additions.
7. **Open questions** — capture anything still unresolved. These become the MENU's "Open questions" section.

### Step 4 — Write MENU

Derive a slug from the feature name (lowercase, hyphenated, ≤ 5 words). Target path: `docs/ai-plans/<today's-ISO-date>-<slug>-MENU.md`.

**Before writing**, check whether `docs/ai-plans/` appears in any `.gitignore` in the repo (including parent repos). If it does, `AskUserQuestion`:

> Question: "`docs/ai-plans/` is in `.gitignore`. The MENU+RECIPES are supposed to be committed artifacts — they're the plan's authoritative state. How should I proceed?"
>
> Options:
> - `Un-ignore — remove the line from .gitignore so these files can be committed` (Recommended)
> - `Write anyway as untracked — you'll handle committing manually`
> - `Let's discuss`

Wait for a decision before writing. The gitignore check is load-bearing: silently writing artifacts that will be excluded from history defeats the whole "plan is a committed contract" model.

If `docs/ai-plans/` doesn't exist, create it (no gitignore concern). Write the MENU using the template in **File formats** below.

### Step 5 — Section breakdown (tracer-bullet vertical slices)

Present a proposed list of sections. Each section is one vertical slice that cuts end-to-end through every layer the feature touches (schema → API → UI → tests) — NOT a horizontal layer.

For each section in the proposal, show:

- **Title** — short imperative noun phrase.
- **User stories covered** — which story numbers from the MENU.
- **Suggested `Model:`** — `haiku` if mechanical (1–2 files, clear spec), `sonnet` if integration/judgment, `opus` if architectural. Default when in doubt: `sonnet`.

Iterate with the user until the breakdown is approved. If a section feels too big ("two features smooshed"), split it. If two sections are tiny and trivially related, merge them.

**Writing acceptance criteria — coaching note.** Strong success criteria let the executor loop independently — if a criterion is vague, the executor will ask rather than proceed. Write criteria that are testable from the outside: observable behaviors, not internal implementation choices. "User can log in with email+password and receives a session cookie" is a criterion. "Uses bcrypt for password hashing" is not — it's an internal choice that belongs in the RECIPES architectural-decisions section or is simply up to the executor. Good criteria survive refactoring; implementation-detail criteria don't.

### Step 6 — Write RECIPES

Write to `docs/ai-plans/<today's-ISO-date>-<slug>-RECIPES.md` using the template in **File formats** below. Include:

- Header with cross-reference to the MENU
- `## Architectural decisions` section (distilled from Step 3 Q4 + Q5)
- `## Conventions` section (TDD per section, one commit minimum, 2-stage review, default model `sonnet`)
- One `## Section N: <Title>` block per approved section, each with `Status: [ ] not started`, `Model:` field, "What to build", acceptance criteria checkboxes, notes for executor, empty completion log

Apply the acceptance-criteria coaching note from Step 5 to every section.

### Step 7 — Self-review pass

This is a checklist YOU run yourself — not a subagent dispatch. Scan both files with fresh eyes:

1. **Spec coverage.** Walk each user story in the MENU. Can you point to a section in RECIPES that delivers it? List any gaps; add sections inline if missing.
2. **Placeholder scan.** Any `TBD`, `TODO`, `implement later`, `fill in details`, "add appropriate error handling", "similar to Section N"? Fix them. These are MENU/RECIPES failures.
3. **Type consistency.** Do module names, route paths, schema shapes referenced in later sections match what you set in earlier sections and in the MENU architecture sketch? A `UserSession` in Section 1 becoming `Session` in Section 4 is a bug.
4. **Vertical-slice check.** Is each section demoable on its own, or are you describing horizontal layers (all-schema-first, then all-API, then all-UI)? If horizontal, restructure into verticals.
5. **Model coherence.** Is the `Model:` field appropriate to each section's complexity? An `opus` section that's just "rename a function" or a `haiku` section that's "design a new auth boundary" is a smell.
6. **Acceptance-criteria externality.** Are criteria observable from the outside, or do they bake in internal implementation choices? Rewrite any that aren't externally testable.

Fix any issues inline. No need to re-review — just fix and move on.

### Step 8 — Commit

Commit both files in a single commit. Message format: `Plan: <feature name> (MENU + RECIPES)`. Do not include attribution trailers.

### Step 9 — Optional GitHub-issue output

After the MENU+RECIPES commit, the MENU+RECIPES files are the primary, authoritative artifacts — they are always written and committed. GitHub-issue creation is an additive, optional output for users whose team workflow expects a trackable issue.

Call `AskUserQuestion`:

> Question: "Also create a GitHub issue with the PRD content? (MENU+RECIPES are already committed either way.)"
>
> Options:
> - `Yes, create a GitHub issue` — file a GitHub issue using the MENU content as its body, formatted with the PRD template below.
> - `No, just proceed to cook` — default; skip to handoff.
> - `Let's discuss`

If the user chooses **Yes**, build the issue body from the committed MENU using the PRD template in **File formats → GitHub issue PRD template** below. Pull `Problem` from MENU's `## Problem`, `Solution` from MENU's `## Solution`, `User Stories` from MENU's `## User stories`, and derive `Implementation Decisions` from the RECIPES `## Architectural decisions` section. Use `gh issue create` (requires the `gh` CLI to be authenticated). Title: the feature name. Report the issue URL back to the user. Then proceed to Step 10.

If **No**, proceed directly to Step 10.

### Step 10 — Handoff

End the session with exactly this handoff message (adjust paths):

> **REQUIRED NEXT SKILL:** Invoke `/cook` to execute the sections in `docs/ai-plans/<date>-<slug>-RECIPES.md`. Each section will run in a fresh subagent with 2-stage review.

Do not invoke `/cook` yourself. The user runs it when ready.

## Grill mode

Invoked when the user selects `Grill mode` at Step 1, or asks to "be grilled", "stress-test a plan", or similar.

**Purpose.** Reach shared understanding of the user's plan through relentless interviewing. No artifact is produced — no MENU, no RECIPES, no GitHub issue, no committed plan file.

**Rules:**

1. **One question at a time.** Asking multiple questions per turn makes later questions irrelevant once earlier ones are answered. Split, always.
2. **Recommend an answer for every question.** Don't ask "what do you think?" in the void — say "I'd recommend X because …, but I want to hear your take." The grill is adversarial-but-cooperative: the user gets to push back on your recommendation, which is where the learning happens.
3. **Walk the decision tree depth-first.** Each decision has dependencies. Before moving on to a sibling branch, fully resolve the current branch — ask follow-ups until the sub-tree is settled. Don't bounce between unrelated topics.
4. **If a question can be answered by exploring the codebase, explore the codebase.** Don't ask the user things the repo already tells you. Use `Read`, `Grep`, `Glob`, and an `Explore` subagent as needed.
5. **Do not produce an artifact.** No files written, no commits, no GitHub issues. The output of grill mode is conversation, nothing more.
6. **Exit condition.** Grill mode ends when shared understanding is reached — signaled either by the user ("I think we're good", "that's enough") or by your own honest assessment ("I've run out of pressure points to push on — do you want to keep going or are we done?"). When ending, give a brief 3-5 bullet summary of the key decisions you both settled on, and stop.

**Do not:**

- Drift into Step 2 (codebase recon for a new plan) — grill mode is not a pre-MENU phase, it's its own terminal path.
- Offer to write up the conversation as a MENU at the end unless the user explicitly asks. If they do ask, re-enter Step 1 with `New feature` and start clean.

## File formats

### MENU template

```markdown
# MENU: <Feature Name>

> Status: draft
> Plan: ./<date>-<slug>-RECIPES.md
> Created: <YYYY-MM-DD>  |  Last touched: <YYYY-MM-DD>

## Problem

<1–3 paragraphs, user's perspective, open prose>

## Solution

<1–3 paragraphs, user's perspective, open prose>

## User stories

1. As <actor>, I want <feature>, so <benefit>
2. ...

## Architecture & module sketch

- **Module A** — responsibility, interface
- **Module B** — ...

<Durable decisions only. No file paths. No code.>

## Testing approach

- What makes a good test here
- Key behaviors to cover
- Prior-art references (existing test files, fixtures, patterns to follow)

## Out of scope

- ...

## Open questions

- [ ] <anything unresolved after intake>
```

### RECIPES template

```markdown
# RECIPES: <Feature Name>

> MENU: ./<date>-<slug>-MENU.md
> Executor: /cook
> Created: <YYYY-MM-DD>  |  Last touched: <YYYY-MM-DD>

## Architectural decisions

<Durable decisions that apply across sections — routes, schema, key models, auth approach, third-party boundaries. Derived from the MENU's module sketch + any detail added during Section breakdown.>

## Conventions

- TDD per section (test → impl → commit)
- Minimum one commit per completed section
- Review checkpoint between sections (spec compliance + code quality)
- Default implementer model: `sonnet`

---

## Section 1: <Title>

**Status:** [ ] not started
**Model:** sonnet
**User stories covered:** 1, 3

### What to build

<1–3 sentences describing the end-to-end vertical slice.>

### Acceptance criteria

- [ ] <testable behavior 1 — externally observable>
- [ ] <testable behavior 2 — externally observable>
- [ ] <testable behavior 3 — externally observable>

### Notes for executor

- <gotchas, pointers to related code, non-obvious constraints>

### Completion log

<!-- Executor fills in after section completes -->
- Commits:
- Tests added:
- Deviations from plan:

---

## Section 2: <Title>

<...same structure...>
```

Use literal backtick-bounded status checkboxes (`[ ]` and `[x]`) — `/cook` greps for these to find the next unstarted section.

### GitHub issue PRD template

Used at Step 9 when the user opts to also file a GitHub issue. The issue body should be derived from the committed MENU and RECIPES — do not re-interview the user.

```markdown
## Problem Statement

<From MENU's "## Problem" section, user's perspective>

## Solution

<From MENU's "## Solution" section, user's perspective>

## User Stories

<Numbered list, copied from MENU's "## User stories">

1. As a <actor>, I want a <feature>, so that <benefit>
2. ...

## Implementation Decisions

<Derived from RECIPES's "## Architectural decisions" section. Include:>

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated quickly.

## Testing Decisions

<From MENU's "## Testing approach" section. Include:>

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (similar types of tests in the codebase)

## Out of Scope

<From MENU's "## Out of scope" section>

## Further Notes

<Link back to the committed MENU+RECIPES paths so the issue references the authoritative plan.>
```

## Rationalization table

| Excuse | Reality |
|---|---|
| "This feature is too simple for a full PRD + plan" | Simple features still benefit from a MENU+RECIPES — it's 15 minutes of interview, not an afternoon. Do it. If it's truly trivial, the MENU is short; the format doesn't scale with feature size. |
| "The user wants to move fast, so I'll skip the interview" | Moving fast WITH a plan is faster than re-doing work without one. The interview is the moving-fast step. |
| "I'll just ask everything I need in one message" | One question per turn. If you're tempted to bundle, split. |
| "`AskUserQuestion` is overkill for 2 options" | No — 2 options is exactly when it's most useful. It formats cleanly and logs the decision. |
| "The escape hatch feels redundant" | The user's actual situation may not fit your options. Always include `"Let's discuss"`. |
| "Tradeoff tables are a lot of ceremony" | A table is 3 columns × 3 rows. Cheaper than recovering from a wrong default. |
| "I'll skip proposing 2-3 approaches — one obvious choice" | If one approach is truly dominant, just make the decision and move on without asking. If you're actually asking the user, there's ambiguity, which means a tradeoff table earns its keep. |
| "I can merge MENU and RECIPES into one file" | No. Two files, two purposes. The MENU is the what (stable); the RECIPES is the how (evolves per section). `/cook` greps RECIPES for status. |
| "I'll commit after /cook finishes" | Commit AFTER writing MENU+RECIPES, BEFORE `/cook` runs. The plan is the contract; it must exist in history before execution. |
| "Grill mode should probably produce a write-up at the end" | No. Grill mode's value is the conversation. If the user wants an artifact, they re-enter with `New feature`. Don't auto-write one. |
| "Acceptance criteria about implementation details are fine — they're specific" | Specific ≠ testable. "Uses bcrypt" is specific but not externally observable. If a reviewer can't verify it from outside the module, it's the wrong kind of criterion. |
| "The user already asked for a GitHub issue, I'll skip the MENU+RECIPES" | No. MENU+RECIPES are always primary and always written. The GitHub issue is additive — it mirrors the PRD for team-workflow reasons. |

## Red flags — STOP

- Skipping the new-vs-extend-vs-grill decision
- More than one `?` in a single user-facing message
- Inline numbered options instead of `AskUserQuestion`
- `AskUserQuestion` without a `"Let's discuss"` option
- Architecture decision asked without a preceding 2-3-approach tradeoff table
- Writing a "PRD" without the MENU template or to a filename other than `docs/ai-plans/<date>-<slug>-MENU.md`
- Writing a plan without per-section `Status:` and `Model:` fields
- Horizontal slices (Phase 1 = all schema, Phase 2 = all API) instead of vertical slices
- Acceptance criteria that describe internal implementation choices instead of externally observable behaviors
- Skipping the self-review pass because "I wrote it carefully"
- Skipping the commit because the repo "isn't ready"
- Filing a GitHub issue BEFORE committing MENU+RECIPES (order is: write → self-review → commit → optional issue)
- Producing any file, commit, or issue while in grill mode
- Handoff message that doesn't say `/cook`

## When NOT to use

- User wants to extend an existing MENU+RECIPES → use `/add-to-menu` directly (or choose `Extend existing` at Step 1).
- User wants a quick scratch plan with no PRD → a single Markdown file in `plans/` is fine; this skill is for the MENU+RECIPES dual artifact specifically.
- Task is a one-line bugfix or chore → just fix it.
- User wants only a GitHub issue with no committed plan → this skill always writes MENU+RECIPES first; the issue is additive. If they truly want issue-only with no repo artifacts, that's a different workflow — ask them to confirm they'd rather skip the committed plan entirely before proceeding.
