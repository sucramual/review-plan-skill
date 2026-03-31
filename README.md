# review-plan

An [Agent Skill](https://agentskills.io) that dispatches an independent, context-free reviewer to critique your implementation plan before you execute it.

The reviewer has codebase access to verify assumptions but knows nothing about the conversation that produced the plan. Returns a severity-ranked report with gap analysis and actionable suggestions.

## Why

When you write a plan, you're anchored by the conversation that produced it. A fresh perspective catches blind spots. This skill automates the "spin up a separate LLM instance to review my plan" workflow.

**Two reviewer backends:**
- **Claude** (default) — subagent with zero chat context. Same model, fresh eyes. Zero setup.
- **Codex** — via the codex skill plugin. Different model, genuinely independent perspective.

## Install

### Claude Code

```bash
git clone https://github.com/sucramual/review-plan-skill.git ~/.claude/skills/review-plan
```

### Codex backend (optional)

The Codex reviewer backend requires the [codex plugin](https://github.com/openai/codex) for Claude Code. Run `/codex:setup` to verify installation and authentication.

## Usage

```
/review-plan                                          # review plan from current context
/review-plan docs/plans/my-plan.md                    # explicit plan path
/review-plan --reviewer codex                         # cross-model review via Codex
/review-plan --persona "security engineer"            # custom reviewer perspective
/review-plan --persona "Andrej Karpathy" --context https://karpathy.ai/blog/...
/review-plan --persona "security engineer, Tri Dao, performance engineer"  # parallel multi-persona
/review-plan --context https://some-rfc.com/spec.md   # supplementary material
```

### Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| Plan path | Auto-detect or prompt | Path to the plan file |
| `--reviewer` | `claude` | `claude` (subagent) or `codex` (via codex plugin) |
| `--persona` | senior/staff engineer | Role, discipline, or named person. Comma-separated for multiple (max 3) — dispatches parallel independent reviewers and synthesizes a unified report |
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

**Multi-persona output** — when multiple personas are provided, reports are synthesized into a single unified output with convergence attribution:

```
### Plan Review: [plan name]

**Reviewed by:** security engineer, Tri Dao, performance engineer

#### Critical Issues
- **[Issue]** [3/3 personas]: [Description] → Suggested fix: [...]
- **[Issue]** [1/3 — security engineer]: [Description] → Suggested fix: [...]

#### Important Issues
...

#### Minor Issues
...

#### Strengths
[Merged, attributed where persona-specific]

#### Consensus & Disagreements
[Where personas strongly agreed, and any notable contradictions between perspectives]

#### Overall Assessment
[Synthesized verdict — weighted toward consensus]
```

Within each severity tier, issues flagged by more personas sort higher — convergence = confidence. If one persona's review fails, the remaining reviews are still synthesized.

### What the reviewer checks

- **Feasibility** — Do referenced files exist? Are dependencies real? Missing steps?
- **Architecture** — Right approach? Over/under-engineered? Better alternatives?
- **Risk** — Riskiest parts? Wrong assumptions? What to validate first?

## Superpowers Integration

If you use the [superpowers](https://github.com/anthropics/claude-code-superpowers) plugin, this skill fits as an optional gate before execution:

```
brainstorming → writing-plans → /review-plan (optional) → executing-plans
```

The superpowers internal plan reviewer checks structure (TODOs, task decomposition). This skill checks substance (feasibility, architecture, risk) with an independent reviewer that has zero chat context and full codebase access. Run it after writing-plans completes, before you start executing:

```
/review-plan docs/superpowers/plans/YYYY-MM-DD-feature-name.md
```

## Requirements

- **Claude backend**: Works out of the box in Claude Code
- **Codex backend**: Requires the [codex plugin](https://github.com/openai/codex) installed and authenticated (run `/codex:setup` to verify)

## License

MIT
