# review-plan

An [Agent Skill](https://agentskills.io) that dispatches an independent, context-free reviewer to critique your implementation plan before you execute it.

The reviewer has codebase access to verify assumptions but knows nothing about the conversation that produced the plan. Returns a severity-ranked report with gap analysis and actionable suggestions.

## Why

When you write a plan, you're anchored by the conversation that produced it. A fresh perspective catches blind spots. This skill automates the "spin up a separate LLM instance to review my plan" workflow.

**Two reviewer backends:**
- **Claude** (default) — subagent with zero chat context. Same model, fresh eyes. Zero setup.
- **Codex** — OpenAI Codex CLI. Different model, genuinely independent perspective.

## Install

### Claude Code

```bash
git clone https://github.com/sucramual/review-plan-skill.git ~/.claude/skills/review-plan
```

### Codex

```bash
git clone https://github.com/sucramual/review-plan-skill.git ~/.codex/skills/review-plan
```

## Usage

```
/review-plan                                          # review plan from current context
/review-plan docs/plans/my-plan.md                    # explicit plan path
/review-plan --reviewer codex                         # cross-model review via Codex
/review-plan --persona "security engineer"            # custom reviewer perspective
/review-plan --persona "Andrej Karpathy" --context https://karpathy.ai/blog/...
/review-plan --context https://some-rfc.com/spec.md   # supplementary material
```

### Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| Plan path | Auto-detect or prompt | Path to the plan file |
| `--reviewer` | `claude` | `claude` (subagent) or `codex` (Codex CLI) |
| `--persona` | senior/staff engineer | Role, discipline, or named person |
| `--context` | None | URLs or file paths — RFCs, blog posts, API docs, persona references |

### Output

A structured report ranked by severity:

```
### Plan Review: [plan name]

#### Critical Issues
- **[Issue]**: [Gap explanation] -> Suggested fix: [what to do]

#### Important Issues
...

#### Minor Issues
...

#### Strengths
...

#### Overall Assessment
[Ready to execute / needs revisions / needs rework]
```

### What the reviewer checks

- **Feasibility** — Do referenced files exist? Are dependencies real? Missing steps?
- **Architecture** — Right approach? Over/under-engineered? Better alternatives?
- **Risk** — Riskiest parts? Wrong assumptions? What to validate first?

## Requirements

- **Claude backend**: Works out of the box in Claude Code
- **Codex backend**: Requires [Codex CLI](https://github.com/openai/codex) installed and authenticated (`npm i -g @openai/codex`)

## License

MIT
