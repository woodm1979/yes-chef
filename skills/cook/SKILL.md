---
name: cook
description: Use when executing sections from a RECIPES plan file produced by /plan-meal or /add-to-menu — runs sections one at a time with per-section subagents and 2-stage review. Part of the yes-chef suite.
---

# /cook

*"Yes, Chef!"*

## Overview

Executor of the yes-chef suite. Given a `-RECIPES.md` file produced by `/plan-meal` or `/add-to-menu`, this skill runs each `[ ] not started` section in order:

1. Dispatch a **fresh-context** implementer subagent with ONLY the information it strictly needs.
2. Dispatch a **spec-compliance** reviewer subagent (opus) — does the code match the section's acceptance criteria?
3. Dispatch a **code-quality** reviewer subagent (opus) — is the code well-built?
4. On approval, flip the section's `Status:` to `[x] complete`, check acceptance criteria boxes, fill in the completion log, bump `Last touched:`, commit the plan-file update.
5. **Continue automatically to the next section** unless all sections are complete or the user interrupts.

The entire workflow is resumable across sessions: the RECIPES file IS the state.

If the Agent tool is unavailable in the current harness, a **no-subagent fallback mode** is provided (see "No-subagent fallback" below). The subagent path is the default and preferred mode — output quality is significantly higher with subagent dispatch.

## REQUIRED BACKGROUND

**REQUIRED SUB-SKILL FOR DISPATCHED IMPLEMENTERS:** `yes-chef:tdd` — every implementer prompt must instruct the subagent to follow TDD.

**Granularity:** `/cook` operates at SECTION granularity (an end-to-end tracer-bullet vertical slice). A section's implementer is expected to write multiple commits and produce several files — that's fine. The reviewers still run once per section, not once per commit.

## Precedence

If a repo's `CLAUDE.md`, `AGENTS.md`, or explicit user instructions conflict with this skill, user instructions win. This skill's rules apply only where the user hasn't overridden them. Read those files before dispatching the first section.

## The discipline: focus, not isolation

**The real risk when running sections isn't context pollution — it's future-section overreach.** An implementer who reads ahead is tempted to add hooks, parameters, or abstractions that "might help Section 4 later." That's a YAGNI violation; the reviewer catches some of it, but not all.

What the implementer SHOULD do:

- Read the plan file freely for context. Prior sections' completion logs (including "Deviations from plan" notes) are useful — the implementer learns what actually shipped vs. what the plan originally said.
- Grep the codebase freely. The shipped code is the ground truth for interfaces that prior sections built.
- Focus on the current section's acceptance criteria. Implement exactly what's required to satisfy them, plus the tests. Nothing else.

What the implementer MUST NOT do:

- Pre-build infrastructure for sections that haven't started yet (fields, parameters, call sites, "extension points").
- Claim the current section is done if any acceptance criterion would need work in a future section to pass.
- Invent architectural decisions that weren't in the plan's `## Architectural decisions` block. If the plan is ambiguous, stop and ask the controller — don't fill the gap yourself.

### Controller hygiene

Subagents dispatched via the `Agent` tool have no access to the controller's conversation history. That's automatic — you don't need to engineer it. Two things the controller should still watch:

1. **Don't embed your own reasoning in the subagent prompt.** If you've been thinking about Section 3 in this controller session, that thinking stays out of Section 3's prompt. Include plan-file content; exclude your commentary on it.
2. **When a prior section deviated, the plan file's completion log already captures that.** The implementer will read it. No need to restate it inline.

If the plan file is incomplete — a section's "What to build" can't actually be implemented from the plan + codebase alone — STOP execution, update the plan, and re-dispatch. Don't paper over plan gaps with private controller context; that couples sections to the current session and breaks cross-session resumability.

## Process

### Step 1 — Locate and read the RECIPES file

In order:

1. If the user `@`-referenced a `-RECIPES.md` path, use it.
2. If there's exactly one `docs/ai-plans/*-RECIPES.md`, use it.
3. If multiple candidates, `AskUserQuestion` with each + `"Let's discuss"`.
4. If none, tell the user to run `/plan-meal` first.

Read the RECIPES file end-to-end. Extract:

- The `## Architectural decisions` block (verbatim).
- Every `## Section N:` block.

Bump `Last touched:` to today's date in the RECIPES header and commit that single-line change with message `cook: begin execution`.

### Step 2 — Select the next unstarted section

Grep the RECIPES file for `**Status:** [ ] not started` (literal). The first match's section is the one to run.

If no match: announce "All sections complete" and stop.

### Step 3 — Capture pre-section SHA, then dispatch implementer subagent

**First, capture the pre-section SHA.** Before dispatching, run:

```
git rev-parse HEAD
```

Store this SHA — you'll need it in Step 4 to scope the diff. This is the parent of whatever commits the implementer will make.

Determine the implementer's model from the section's `Model:` field (default `sonnet` if absent or malformed).

Construct the implementer prompt using EXACTLY the template below. Don't embed your own reasoning or commentary; stick to plan content.

```
You are implementing Section <N>: <Title> of a feature plan.

## Repo root
<absolute-path-to-repo-root>

## Plan file
<absolute-path-to-RECIPES.md>

## Architectural decisions (binding — apply to ALL sections of this plan)
<verbatim copy of the plan's "## Architectural decisions" block>

## Section <N>: <Title>

### What to build
<verbatim copy>

### Acceptance criteria
<verbatim copy of the checkbox list>

### Notes for executor
<verbatim copy>

## Before you begin

If you have questions about requirements, acceptance criteria, approach, or anything unclear — ask them now, before starting work. It is always OK to pause and clarify. Don't guess or make assumptions.

## TDD protocol

Before writing any code, invoke the Skill tool with skill name `yes-chef:tdd` to load the full TDD iron-law protocol.

## Discipline

- Follow TDD per `yes-chef:tdd`: test first, watch it fail, minimal code to green, commit.
- Produce at least one commit when the section is done. Multiple commits are fine.
- Before finishing, re-read this section's acceptance criteria and verify each one is satisfied.
- Focus on Section <N>. You MAY read the plan file for context (prior sections' completion logs and deviations are useful). You MAY grep the shipped codebase freely — that's the ground truth for interfaces earlier sections built.
- Do NOT pre-build anything for a section that hasn't started. Sections you haven't been assigned are not yours to anticipate — implementing them now is YAGNI. Add only what this section's acceptance criteria require.
- If the plan is ambiguous or a decision isn't captured in the Architectural decisions block, stop and report back — do NOT fill the gap on your own.

## Code organization

- Follow the file structure defined in the plan.
- Each file should have one clear responsibility.
- If a file you're creating is growing beyond the plan's intent, stop and report DONE_WITH_CONCERNS — don't restructure on your own without plan guidance.
- In existing codebases, follow established patterns visible in the surrounding code.

## When you're in over your head

It is always OK to stop and say "this is too hard for me." Bad work is worse than no work. You will not be penalized for escalating. STOP and escalate when:

- The section requires architectural decisions with multiple valid approaches not resolved by the plan.
- You need to understand code beyond what was provided and can't find clarity.
- You feel genuinely uncertain about whether your approach is correct.
- You've been reading file after file without making progress.

## Before reporting back: self-review

Review your work with fresh eyes:

- **Completeness:** Did I satisfy every acceptance criterion? Any edge cases missed?
- **Quality:** Is this my best work? Are names clear? Is code clean?
- **Discipline:** Did I avoid over-building? Did I stay inside this section's scope?
- **Testing:** Do tests verify real behavior (not mock theater)? Did I follow TDD?

Fix issues now before reporting.

## Report back

When done, report (briefly):

1. **Status:** one of `DONE` | `DONE_WITH_CONCERNS` | `BLOCKED` | `NEEDS_CONTEXT`
2. Files created or modified (absolute paths)
3. Commit SHA(s)
4. Test count and whether the project's test runner passes
5. Any deviations from the spec and why
6. Any concerns (if `DONE_WITH_CONCERNS`) or the specific blocker (if `BLOCKED` / `NEEDS_CONTEXT`)
```

Use the `Agent` tool with `subagent_type: "general-purpose"`, the chosen model, and the prompt above.

### Step 3a — Handle implementer status

The implementer reports one of four statuses. Handle each appropriately before moving to review:

- **DONE:** Proceed to Step 4 (spec-compliance review).
- **DONE_WITH_CONCERNS:** The implementer completed the work but flagged doubts. Read the concerns. If they're about correctness or scope, address them (re-dispatch with clarification, or fold into remediation) before review. If they're observations (e.g., "this file is getting large"), note them in the eventual completion log and proceed to review.
- **NEEDS_CONTEXT:** The implementer needs information that wasn't provided. Provide the missing context — preferably by updating the plan file if the gap is cross-section, or inline if truly current-section-local — and re-dispatch.
- **BLOCKED:** Assess the blocker:
  1. If it's a context problem, provide more context and re-dispatch with the same model.
  2. If the task requires more reasoning, re-dispatch with a more capable model.
  3. If the section is too large, STOP and surface to the user — the plan likely needs to split this section.
  4. If the plan itself is wrong, STOP and surface to the user to update the plan.

**Never** silently ignore an escalation or force the same model to retry without changing something. If the implementer said it's stuck, something needs to change.

### Step 4 — Dispatch spec-compliance reviewer (opus)

After the implementer reports `DONE` (or concerns have been resolved), use the **pre-section SHA** you captured in Step 3 to scope the diff:

```
git log --oneline <pre-section-sha>..HEAD   # list section's commits
git diff <pre-section-sha>..HEAD            # combined diff for review
```

The `<pre-section-sha>` is the commit that existed immediately before you dispatched the implementer — that's why you captured it before dispatch. For the very first section after `/cook` begins, this is `HEAD` right after the `cook: begin execution` commit from Step 1.

Dispatch a reviewer with model `opus`:

```
You are a spec-compliance reviewer. Review the just-completed implementation of Section <N>: <Title> against its acceptance criteria. Flag only deviations from the spec — do NOT suggest improvements beyond it.

## Repo root
<absolute-path>

## Section <N>: <Title>

### What to build
<verbatim>

### Acceptance criteria
<verbatim>

## Architectural decisions (binding)
<verbatim>

## What the implementer claims they built
<from the implementer's report>

## The diff under review
<inline diff, or `git show <sha>` for each commit in the section>

## CRITICAL: do not trust the report

The implementer's report may be incomplete, inaccurate, or optimistic. Verify everything independently.

DO NOT:
- Take the implementer's word for what was built
- Trust their claims about completeness
- Accept their interpretation of requirements

DO:
- Read the actual code they wrote
- Compare actual implementation to acceptance criteria line by line
- Check for missing pieces they claimed to implement
- Look for extra features they didn't mention

## Output

For each acceptance criterion, state PASS or FAIL with a one-line justification grounded in the actual code (cite file:line where useful). Also flag: any EXTRA behavior that wasn't requested (over-building), any binding architectural decision that was violated.

End with exactly one of:
- `APPROVED`
- `NEEDS FIXES` followed by a numbered list of fixes
```

### Step 5 — Dispatch code-quality reviewer (opus)

Only if Step 4 returned `APPROVED`.

```
You are a code-quality reviewer. Review the code for Section <N>: <Title> for quality. A separate reviewer has already verified spec compliance — focus on craft.

## Repo root
<absolute-path>

## The diff under review
<inline diff or git show>

## What to assess

- Is the code idiomatic for the language/framework it's in?
- Are tests meaningful (not mock-verification theater)?
- Any obvious security, concurrency, or correctness pitfalls?
- Is the code appropriately sized for the scope (not under- or over-engineered)?
- Does it follow project conventions visible in the surrounding codebase?
- Does each new file have one clear responsibility with a well-defined interface?
- Did this change create files that are already large, or grow existing files significantly? (Don't flag pre-existing size — focus on what this change contributed.)

## Output

Bullet list of findings. Tag each as BLOCKING or SUGGESTION. If none: say "No issues found."

End with exactly one of:
- `APPROVED`
- `NEEDS FIXES` followed by a numbered list of BLOCKING fixes
```

### Step 6 — Remediation (only if a reviewer returns NEEDS FIXES)

Dispatch a fresh remediation subagent (NOT continuing the implementer's context — fresh again). Pass it:

- Section title + "What to build" + acceptance criteria + architectural decisions
- The diff of the just-completed work
- The specific NEEDS FIXES list from whichever reviewer flagged it
- Repo + plan-file paths

Instruction: fix the listed items; do not introduce changes beyond the list; re-run tests; commit.

After remediation, re-run the reviewer that rejected. If both reviewers eventually `APPROVED`, proceed. If the same reviewer rejects twice in a row, STOP and surface to the user — don't loop indefinitely.

### Step 7 — Update the plan file

Edit the RECIPES file with the Edit tool:

- Flip this section's `**Status:** [ ] not started` → `**Status:** [x] complete`
- Check off every `- [ ]` in the section's acceptance criteria → `- [x]` (reviewers validated these)
- Fill in `### Completion log`:
  - `Commits: <SHA-list>`
  - `Tests added: <count or test-file path, from implementer report>`
  - `Deviations from plan: <from implementer report; "none" if clean>`
- Bump the `Last touched:` header to today's date

Commit that single RECIPES edit with message `cook: complete Section <N> (<Title>)`. No attribution trailers.

### Step 8 — Continue automatically

Go back to Step 2 and select the next `[ ] not started` section.

**Only stop the loop when:**

1. No `[ ] not started` sections remain → announce completion and exit.
2. A reviewer rejected the same section twice in a row → surface the issue to the user for guidance.
3. The implementer returned `BLOCKED` with a blocker that requires user/plan intervention.
4. The user interrupts the session.

The controller MAY emit a one-sentence summary to the user between sections ("Section 2 complete, moving to Section 3") but this summary NEVER enters the next section's subagent context.

### Step 9 — Completion announcement

When all sections are complete, announce exactly:

> All sections in `<path-to-RECIPES.md>` are complete. MENU: `<path-to-MENU.md>`. Last section committed in `<SHA>`.

## No-subagent fallback mode

**When to use:** Only when the Agent tool is unavailable in the current harness (no subagent support). The subagent-driven path above is strictly preferred — quality is significantly higher with fresh-context subagents and independent reviewers. Tell your human partner that `/cook` works much better with access to subagents, and if possible switch to a harness that supports them.

**What changes:** The controller (you) performs implementation, review, and remediation sequentially in the single session. There is no fresh context per section and no independent reviewer process. The RECIPES state machine and section-level granularity are unchanged — you still run one section at a time, in order, and update the plan file between sections.

**Process in fallback mode:**

1. **Step 1–2 unchanged:** locate the RECIPES file, bump `Last touched:`, commit `cook: begin execution`, then grep for the next `**Status:** [ ] not started`.
2. **Step 3 (sequential implementation):** Capture the pre-section SHA. Read the plan's architectural decisions + the full section block. Review the section critically — if you have questions or concerns about feasibility or ambiguity, raise them with your human partner BEFORE starting. If no concerns, proceed.
3. **Implement sequentially:** Follow TDD per `yes-chef:tdd`. Test first, watch it fail, minimal code to green, commit. Produce at least one commit per section. Do not pre-build future sections. If you hit a blocker (missing dependency, failing test, unclear instruction, plan gap), **stop and ask your human partner** — do not guess.
4. **Manual checkpoint (spec compliance):** After implementation, explicitly re-read the section's acceptance criteria. For each criterion, write down PASS or FAIL with a justification grounded in the actual code you just wrote. Flag any extra behavior you added beyond the spec. If any criterion is FAIL or if you added extras, remediate before the next checkpoint.
5. **Manual checkpoint (code quality):** Review the diff you just produced against the code-quality checklist in Step 5 above (idiomaticity, meaningful tests, obvious pitfalls, sizing, conventions, single responsibility). Document findings. Fix anything BLOCKING; SUGGESTION-level can be noted in the completion log.
6. **Present to your human partner:** Before updating the plan file, present the diff and both checkpoint reports. Get explicit approval, then proceed.
7. **Step 7–9 unchanged:** update the RECIPES file, commit `cook: complete Section <N> (<Title>)`, continue to the next section.

**Fallback-mode discipline:**

- Review the plan critically first; raise concerns before starting.
- Follow plan steps exactly; don't skip verifications.
- Stop and ask when blocked; don't force through.
- Never start implementation on `main` / `master` without explicit user consent.
- The two manual checkpoints are not optional — they are the only quality gates in this mode.
- You lack the fresh-context guarantee of a subagent dispatch, so be especially vigilant about over-building and future-section overreach.

## Model selection

| Role | Default model | Why |
|---|---|---|
| Implementer | from section's `Model:` field; default `sonnet` | Section complexity varies; plan-author picks. |
| Spec-compliance reviewer | `opus` | Rigorous fit-to-spec analysis. |
| Code-quality reviewer | `opus` | Best judgment on craft and subtle pitfalls. |
| Remediation implementer | same as original implementer | Match the section's complexity. |

Use the least powerful model that can handle each role to conserve cost and increase speed — but do not downgrade reviewers below `opus`.

## Rationalization table

| Excuse | Reality |
|---|---|
| "Section 4 will need a `Foo.with_bar/2` helper — might as well add it now in Section 2" | That's YAGNI. Implement Section 2's acceptance criteria; stop there. Section 4 gets built when it's Section 4's turn. |
| "The reviewer is just going to approve — I'll skip the dispatch" | Skipping reviewers means skipping the only checkpoint that catches drift. Every section gets both reviewers. |
| "Using opus for reviewers is expensive; sonnet is close enough" | Reviewer quality determines whether the shipped code is trustworthy. Spend the opus tokens. |
| "The implementer asked a question the plan doesn't answer — I'll just answer it inline" | If the answer should apply to future sections too, the PLAN has a gap. Stop, update the plan, commit, re-dispatch. If the answer is current-section-local, answering inline is fine. |
| "Let me batch sections into one subagent to save dispatches" | Sections get review checkpoints individually. One subagent per section is the discipline — not for isolation, but so each gets reviewed before the next begins. |
| "The user said 'run sections 2 and 3' — I'll skip picking next-unstarted and just go by that" | Grep for `[ ] not started` anyway. The user may have misremembered; the plan file is authoritative. |
| "I'll mark the section complete even though the quality reviewer had suggestions" | Only BLOCKING findings block completion. SUGGESTION findings can be noted in the Completion log's "Deviations" line — they don't prevent approval. |
| "The plan is ambiguous on a decision — I'll have the implementer just pick something" | No. The plan is the contract; ambiguity means the contract is incomplete. Pause, clarify with the user, edit the plan, commit, then dispatch. |
| "The implementer reported DONE — the spec reviewer can skim the report instead of reading code" | No. The reviewer verifies by reading code, not by trusting the report. Optimistic reports are a known failure mode. |
| "The implementer returned BLOCKED — I'll just re-dispatch with the same prompt" | Something needs to change. More context, a more capable model, or a smaller scope. Re-dispatching unchanged is just burning tokens. |
| "I'm in no-subagent fallback, so I'll skip the manual checkpoints since I wrote the code myself" | The checkpoints are the ONLY quality gates in fallback mode. Skipping them means there is no review at all. Run them. |

## Red flags — STOP

- Implementer prompt includes pre-built instructions to add parameters or hooks for a section that hasn't started
- Implementer prompt embeds the controller's own reasoning (rather than plan-file content)
- Reviewer dispatched with model `sonnet` (should be `opus`)
- Next section selected via "user said to" rather than grep for `[ ] not started`
- Plan file not updated after section completes
- Plan-file update not committed before moving to next section
- `Last touched:` not bumped
- Remediation loop on same reviewer runs more than twice without stopping
- Progressing to next section while any acceptance criterion is still `- [ ]`
- Pre-section SHA not captured before dispatch (Step 3), leaving no clean way to scope the review diff
- Reviewer approving based on implementer's report without reading the actual diff
- Implementer returned `BLOCKED` or `NEEDS_CONTEXT` and controller re-dispatched unchanged
- Implementer returned `DONE_WITH_CONCERNS` and controller dispatched reviewer without reading the concerns
- Starting implementation on `main` / `master` without explicit user consent
- In no-subagent fallback mode: skipping either manual checkpoint, or marking a section complete without presenting the diff to the user

## Resumption across sessions

Because the RECIPES file IS the state:

- Starting `/cook` in a new session works identically to continuing in the same session. The skill reads the RECIPES file, greps for the next `[ ] not started`, and runs that section. No in-memory state is required.
- The only reminder the controller needs is the RECIPES file path — ideally supplied by the user via `@`-reference.

## When NOT to use

- RECIPES file doesn't exist yet → run `/plan-meal` first.
- Only small, inline edits are needed (no sections) → just make them.
- The plan uses a different format (not yes-chef `-RECIPES.md` with the `[ ] not started` / `[x] complete` state machine) → `/cook` expects that specific format.
