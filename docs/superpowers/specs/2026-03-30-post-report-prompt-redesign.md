# Post-Report Prompt Redesign

**Date:** 2026-03-30
**Scope:** `review-plan` skill — SKILL.md section 6

## Problem

The post-report user prompt (section 6, lines 166-180) is not being displayed after the review report is presented. The agent presents the report and then stops silently, waiting for user input without showing the options menu. Additionally, the current options don't match the desired user workflow.

## Solution

Two changes to SKILL.md section 6:

### 1. Make the prompt structurally unmissable

Wrap section 6 in a `<HARD-GATE>` tag — the same enforcement pattern used by other superpowers skills (e.g., brainstorming). This makes the instruction harder for the agent to skip.

### 2. Replace the options

**Current options:**
1. Address issues — fix specific findings
2. Discuss — talk through findings
3. Proceed as-is — execute without changes
4. Re-review — run another review

**New options:**
1. **Fix all** — update the plan to address all issues
2. **Fix critical and important** — update the plan to address critical and important issues only
3. **Don't fix, proceed as-is** — move on without changes

Free-text responses (anything other than selecting an option) should be acknowledged without taking any action on the plan.

### Behavioral details

- Accept both numbered responses ("1") and natural language ("fix all", "fix everything") as valid selections.
- For options 1 and 2: apply the suggested fixes from the report directly to the plan file, then present a short summary of what was changed. Stop after.
- For option 3: acknowledge and stop.
- For free-text: acknowledge the input without modifying the plan or taking any other action.

## What doesn't change

- `references/reviewer-prompt.md` — reviewer stays focused on critique
- Report output format — options are the orchestrator's job
- Multi-persona synthesis logic
- Everything in sections 1-5
