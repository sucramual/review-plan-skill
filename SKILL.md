---
name: review-plan
description: Dispatch an independent reviewer (Claude subagent or Codex) to critique an implementation plan with zero chat context. Use when seeking a second opinion on a plan, validating feasibility before execution, critiquing architecture decisions, or assessing risk. Supports cross-model review via --reviewer codex for a genuinely different perspective, custom personas (e.g., "security engineer", a named expert), multiple comma-separated personas for parallel independent review with synthesis, and supplementary context via URLs or local files.
---

# Review Plan

Dispatch an independent, context-free reviewer to critique an implementation plan as a senior engineer. The reviewer explores the codebase to verify assumptions but knows nothing about the conversation that produced the plan. Returns a severity-ranked report with gap analysis and actionable suggestions.

**Two reviewer backends:**
- **Claude** (default) — subagent via Agent tool. Same model, fresh context. Zero setup.
- **Codex** — via the `codex` skill plugin (`codex:codex-rescue` subagent). Different model, genuinely independent perspective. Requires the codex plugin installed and authenticated — run `/codex:setup` to verify.

## Invocation

```
/review-plan [path/to/plan.md] [--reviewer claude|codex] [--persona "..."] [--context <url-or-file> ...]
```

All arguments are optional.

## Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| Plan path | Auto-detect from conversation, or prompt | Path to the plan file |
| `--reviewer` | `claude` | Reviewer backend — `claude` (subagent) or `codex` (via codex skill plugin) |
| `--persona` | "senior/staff software engineer" | Reviewer perspective — a role, discipline, or named person. Comma-separated for multiple (max 3) |
| `--context` | None | URLs or file paths providing supplementary material (RFCs, blog posts, API docs, persona references) |

## Process

### 1. Resolve the plan

1. If a path argument was provided, use it
2. If a plan was written during this conversation, use that path
3. Otherwise, ask the user

Read the plan content.

### 2. Gather context materials

For each `--context` value:
- URLs: fetch via WebFetch
- File paths: read via Read

If a fetch fails, warn the user and continue with what succeeded.

### 3. Build the reviewer persona(s)

#### Parse the `--persona` argument

If `--persona` contains commas, split on commas and trim whitespace. Each segment becomes a distinct persona.

- **Empty segments** (e.g., `"security engineer, , performance engineer"`) are ignored.
- **Duplicate personas** are deduplicated before dispatch. Deduplication is case-insensitive and whitespace-normalized (trim + collapse internal whitespace). No semantic equivalence — `"security eng"` and `"security engineer"` are treated as distinct.
- **1 persona** (or no `--persona`) → single-reviewer flow (current behavior, unchanged).
- **2-3 personas** → parallel multi-persona flow (see step 4).
- **4+ personas** → reject with: *"Maximum 3 personas supported. Please narrow your selection."*
- The default persona (`"senior/staff software engineer"`) is **not** auto-included when explicit personas are given.

#### Construct persona instructions

For each persona, construct the persona instruction:
- **No `--persona`**: "You are a senior/staff software engineer reviewing this plan with fresh eyes."
- **Role-based** (e.g., "security engineer"): Shape priorities around that discipline — threat models, auth boundaries, etc.
- **Named person** (e.g., "Andrej Karpathy"): Review through the lens of that person's known expertise and philosophy. If `--context` includes material about them, use it to ground the perspective.

### 4. Dispatch the reviewer(s)

For each persona, build the full reviewer prompt by combining: persona instruction + context materials + full plan content + review instructions and report format from [references/reviewer-prompt.md](references/reviewer-prompt.md). All `--context` materials are provided to every reviewer identically. Only the persona instruction differs.

#### Single persona

Same as before — dispatch one reviewer.

#### Multiple personas (2-3)

Dispatch **all reviewers in a single message** using multiple parallel tool calls. This ensures true parallel execution. Each reviewer operates independently with no knowledge of the others.

#### Claude backend (default)

Use the Agent tool. Paste the full prompt directly — do not pass a file path for the subagent to find. For multiple personas, use multiple Agent tool calls in a single message.

- **Subagent tools:** Read, Glob, Grep, Bash (for codebase verification).
- **Subagent does NOT receive:** chat history, design specs, or knowledge of why the plan was written.

#### Codex backend (`--reviewer codex`)

First verify Codex is available by invoking the `codex:setup` skill via the Skill tool. If Codex is not installed or not authenticated, inform the user to run `/codex:setup` and fall back to Claude.

For each persona, dispatch a `codex:codex-rescue` subagent via the Agent tool. Pass the full reviewer prompt (persona instruction + context materials + plan content + review instructions) as the agent's task. Explicitly state in the task that this is a **read-only review** — no file modifications should be made.

For multiple personas, dispatch all `codex:codex-rescue` subagents in a single message using multiple parallel Agent tool calls.

The `codex:codex-rescue` subagent handles all Codex invocation details (companion script, process management, output collection) via the `codex:codex-cli-runtime`. No manual `codex exec` commands or temp file management is needed.

### 5. Present the report

#### Single persona

Display the reviewer's report as-is.

#### Multiple personas: synthesize then present

When 2-3 persona reviews return, synthesize them into a single unified report before presenting. The orchestrating agent (you) performs the synthesis — do not dispatch another subagent for this.

**Synthesis process:**

1. **Collect** all raw reports (held transiently in context).
2. **Deduplicate** — identify issues that overlap across personas, even if framed differently (e.g., "auth gap" from a security engineer and "missing component" from an architect are the same underlying issue). This is best-effort using your judgment.
3. **Merge overlapping issues** into a single entry:
   - Use the strongest/most actionable framing as the primary description.
   - **Severity reconciliation:** when personas assign different severity levels to the same issue, use the highest severity assigned by any persona.
   - Add attribution showing which personas flagged it.
4. **Preserve unique issues** — issues flagged by only one persona are kept and attributed.
5. **Rank by convergence first, then severity** — within each severity tier (Critical/Important/Minor), issues flagged by more personas sort higher.
6. **Discard raw reports** — only the unified synthesis is presented.

**Attribution format:** When all N personas converge on an issue, show `[N/N personas]` without listing names. When fewer converge, show `[K/N — persona1, persona2]` with names.

**Strengths:** Best-effort merge. No convergence counts — attribute to the originating persona where a strength is perspective-specific.

**Synthesized output format:**

```
### Plan Review: [plan name]

**Reviewed by:** persona1, persona2, persona3

#### Critical Issues
- **[Issue title]** [3/3 personas]: [Description] → Suggested fix: [...]
- **[Issue title]** [1/3 — security engineer]: [Description] → Suggested fix: [...]

#### Important Issues
- **[Issue title]** [2/3 — persona1, persona2]: [Description] → Suggested fix: [...]

#### Minor Issues
...

#### Strengths
[Merged — deduplicated, attributed where persona-specific]

#### Consensus & Disagreements
[Where personas strongly agreed, and any notable contradictions between perspectives]

#### Overall Assessment
[Synthesized verdict — weighted toward consensus]
```

**Error handling:**
- If one reviewer fails, synthesize from the reports that did return. Note: *"Note: the [persona] review could not be completed. Synthesis is based on N-1 perspectives."*
- If all reviewers fail, inform the user and suggest retrying with fewer personas or a single persona.

### 6. Prompt the user for next steps

<HARD-GATE>
Do NOT automatically fix, modify, or act on any issues from the review. You MUST present the options below and wait for the user to respond. This prompt is mandatory — do not skip it, even if the report has no critical issues.
</HARD-GATE>

After presenting the report, prompt the user with:

```
**What would you like to do?**
1. **Fix all** — I'll update the plan to address all issues
2. **Fix critical and important** — I'll update the plan to address critical and important issues only
3. **Don't fix, proceed as-is** — Move on without changes
```

Accept both numbered responses ("1") and natural language ("fix all", "fix everything") as valid selections.

**Behavior by selection:**
- **Option 1 or 2:** Apply the suggested fixes from the report directly to the plan file. Present a short summary of what was changed, then stop.
- **Option 3:** Acknowledge and stop.
- **Free-text response** (anything other than selecting an option): Acknowledge the input without modifying the plan or taking any other action.

## Superpowers Integration

This skill complements the [superpowers](https://github.com/anthropics/claude-code-superpowers) pipeline as an optional review gate before execution:

```
brainstorming → writing-plans → /review-plan (optional) → executing-plans
```

The superpowers writing-plans skill includes an internal plan-document-reviewer that checks *structure* (TODOs, placeholders, task decomposition, spec alignment). This skill provides a different kind of review — *critical evaluation of feasibility, architecture, and risk* by an independent reviewer with zero chat context and full codebase access.

**Recommended usage:** After writing-plans completes and presents "Plan complete. Ready to execute?", run `/review-plan` on the saved plan file before proceeding to execution. Address any critical or important issues, then execute.

```
/review-plan docs/superpowers/plans/YYYY-MM-DD-feature-name.md
```
