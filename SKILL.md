---
name: review-plan
description: Dispatch an independent reviewer (Claude subagent or Codex CLI) to critique an implementation plan with zero chat context. Use when seeking a second opinion on a plan, validating feasibility before execution, critiquing architecture decisions, or assessing risk. Supports cross-model review via --reviewer codex for a genuinely different perspective, custom personas (e.g., "security engineer", a named expert), and supplementary context via URLs or local files.
---

# Review Plan

Dispatch an independent, context-free reviewer to critique an implementation plan as a senior engineer. The reviewer explores the codebase to verify assumptions but knows nothing about the conversation that produced the plan. Returns a severity-ranked report with gap analysis and actionable suggestions.

**Two reviewer backends:**
- **Claude** (default) — subagent via Agent tool. Same model, fresh context. Zero setup.
- **Codex** — OpenAI Codex CLI in non-interactive mode. Different model, genuinely independent perspective. Requires `codex` CLI installed and authenticated.

## Invocation

```
/review-plan [path/to/plan.md] [--reviewer claude|codex] [--persona "..."] [--context <url-or-file> ...]
```

All arguments are optional.

## Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| Plan path | Auto-detect from conversation, or prompt | Path to the plan file |
| `--reviewer` | `claude` | Reviewer backend — `claude` (subagent) or `codex` (Codex CLI) |
| `--persona` | "senior/staff software engineer" | Reviewer perspective — a role, discipline, or named person |
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

### 3. Build the reviewer persona

Construct the persona instruction:
- **No `--persona`**: "You are a senior/staff software engineer reviewing this plan with fresh eyes."
- **Role-based** (e.g., "security engineer"): Shape priorities around that discipline — threat models, auth boundaries, etc.
- **Named person** (e.g., "Andrej Karpathy"): Review through the lens of that person's known expertise and philosophy. If `--context` includes material about them, use it to ground the perspective.

### 4. Dispatch the reviewer

Build the full reviewer prompt by combining: persona instruction + context materials + full plan content + review instructions and report format from [references/reviewer-prompt.md](references/reviewer-prompt.md).

#### Claude backend (default)

Use the Agent tool. Paste the full prompt directly — do not pass a file path for the subagent to find.

- **Subagent tools:** Read, Glob, Grep, Bash (for codebase verification).
- **Subagent does NOT receive:** chat history, design specs, or knowledge of why the plan was written.

#### Codex backend (`--reviewer codex`)

First verify `codex` is available: `which codex`. If not found, inform the user to install it (`npm i -g @openai/codex`) and fall back to Claude.

Write the full reviewer prompt to a temporary file, then invoke:

```bash
codex exec \
  --sandbox read-only \
  --output-last-message /tmp/review-plan-output.md \
  "$(cat /tmp/review-plan-prompt.md)"
```

- `--sandbox read-only` — reviewer can explore the codebase but cannot modify it.
- Read the output from `/tmp/review-plan-output.md` when complete.
- Clean up temp files after.

### 5. Present the report

Display the reviewer's report. The user decides what to act on.
