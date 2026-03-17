# Parallel Multi-Persona Review

Enhance the review-plan skill so that when multiple personas are provided (comma-separated), independent reviewer sub-agents are dispatched in parallel, and the orchestrating agent synthesizes their reports into a single unified output with convergence attribution.

## Motivation

A single-persona review is valuable. Multiple independent personas reviewing the same plan — without seeing each other's work — produce qualitatively different information: convergence signals. When 3 independent reviewers flag the same issue, that's stronger than one reviewer flagging it. Today, getting this requires running `/review-plan` three separate times and manually comparing reports. This enhancement automates the dispatch, parallelism, and synthesis into a single invocation.

## Design

### 1. Argument Parsing & Validation

The `--persona` argument accepts comma-separated values. Each segment (trimmed of whitespace) becomes a distinct persona.

**Rules:**
- **1 persona** (or no `--persona`) → current behavior unchanged. Single sub-agent, report presented as-is. No synthesis step.
- **2-3 personas** → parallel multi-persona flow (new behavior).
- **4+ personas** → reject with: *"Maximum 3 personas supported. Please narrow your selection."*
- **No `--persona`** → default persona applies (unchanged): `"senior/staff software engineer reviewing this plan with fresh eyes"`
- The default persona is **not** auto-included when explicit personas are given — the user controls all perspectives.
- **Duplicate personas** are deduplicated before dispatch. Deduplication is case-insensitive and whitespace-normalized (trim + collapse internal whitespace). No semantic equivalence — `"security eng"` and `"security engineer"` are treated as distinct.
- **Empty segments** (e.g., `"security engineer, , performance engineer"`) are ignored.

**Example:**
```
/review-plan --persona "security engineer, Tri Dao, performance engineer"
```
→ 3 personas: `["security engineer", "Tri Dao", "performance engineer"]`

### 2. Parallel Dispatch

When 2-3 personas are detected after parsing:

1. Build the full reviewer prompt for **each persona independently** — same process as today: persona instruction + context materials + full plan content + review template from `references/reviewer-prompt.md`. Only the `{PERSONA_INSTRUCTION}` differs between prompts. All `--context` materials are provided to every sub-agent identically.
2. Dispatch **all sub-agents in a single message** using multiple Agent tool calls. This ensures true parallel execution.
3. Each sub-agent has full codebase access (Read, Glob, Grep, Bash) — same as today.
4. No sub-agent knows about the others or their personas.

**No changes to `references/reviewer-prompt.md`** — the prompt template stays as-is. Per-persona prompts are built exactly the same way as for a single persona today.

**Codex backend (`--reviewer codex`):** Each persona gets its own Codex process with a unique temp file using a slugified persona name (lowercase, spaces to hyphens, special characters stripped — e.g., `/tmp/review-plan-output-security-engineer.md`). All temp files cleaned up after synthesis.

### 3. Synthesis

After all parallel sub-agents return, the **orchestrating agent** (the main Claude session running the skill) synthesizes their raw reports into a single unified output. This step only applies when 2+ personas were dispatched.

**Synthesis process:**

1. **Collect** all N raw reports (held transiently in context).
2. **Deduplicate** — identify issues that overlap across personas, even if framed differently (e.g., "auth gap" from a security engineer and "missing component" from an architect are the same underlying issue). This is a best-effort heuristic using the orchestrator's judgment — exact semantic matching is not expected.
3. **Merge overlapping issues** into a single entry with:
   - The strongest/most actionable framing as the primary description.
   - **Severity reconciliation:** when personas assign different severity levels to the same underlying issue, use the highest severity assigned by any persona.
   - Attribution: which personas flagged it (e.g., `[flagged by: security engineer, systems architect]`).
   - Convergence count visible in the entry.
4. **Preserve unique issues** — issues flagged by only one persona are kept and attributed to that persona.
5. **Rank by convergence first, then severity** — within each severity tier (Critical/Important/Minor), issues flagged by more personas sort higher.
6. **Discard raw reports** — only the unified synthesis is presented. Raw reports are internal working artifacts.

**Output format** — same structure as today with additions:

```
### Plan Review: [plan name]

**Reviewed by:** security engineer, Tri Dao, performance engineer

#### Critical Issues
- **[Issue title]** [3/3 personas]: [Description] → Suggested fix: [...]
- **[Issue title]** [1/3 — security engineer]: [Description] → Suggested fix: [...]

#### Important Issues
- **[Issue title]** [2/3 — Tri Dao, performance engineer]: [Description] → Suggested fix: [...]
...

#### Minor Issues
...

#### Strengths
[Merged — deduplicated, attributed where a strength was persona-specific]

#### Consensus & Disagreements
[Brief section noting where personas strongly agreed, and any notable disagreements or contradictions between perspectives]

#### Overall Assessment
[Synthesized verdict — weighted toward consensus]
```

**Attribution format rule:** When all N personas converge on an issue, show `[N/N personas]` without listing names. When fewer than all converge, show `[K/N — persona1, persona2]` with the names of the flagging personas.

**Strengths:** Best-effort merge and deduplication. No convergence counts on strengths — attribute to the originating persona where a strength is perspective-specific.

**Consensus & Disagreements:** This section is entirely orchestrator-generated (sub-agents do not produce it). It surfaces where personas strongly agreed and any notable contradictions.

**New elements vs. today's format:**
- `Reviewed by:` header listing all personas.
- `[N/N personas]` or `[K/N — persona names]` tags on each issue.
- New `Consensus & Disagreements` section before the overall assessment.

### 4. Edge Cases & Error Handling

- **Sub-agent failure:** If one sub-agent fails (timeout, error), synthesize from the reports that did return. Note which persona's review was unavailable: *"Note: the [persona] review could not be completed. Synthesis is based on N-1 perspectives."*
- **Identical personas:** Deduplicate before dispatch — don't waste a sub-agent on a duplicate.
- **Mixed with `--reviewer codex`:** Each persona gets its own Codex process with a unique temp file. Temp files cleaned up after synthesis regardless of success/failure.
- **Empty persona segment:** Ignore empty segments after comma-splitting, proceed with valid ones.
- **All sub-agents fail:** Fall back to informing the user that the review could not be completed, suggest retrying with fewer personas or a single persona.

## Files Changed

| File | Change |
|------|--------|
| `SKILL.md` | Update argument table, add multi-persona parsing rules, add parallel dispatch section, add synthesis section, update "Present the report" step |
| `README.md` | Update usage examples, argument table, and output format to reflect multi-persona support |
| `references/reviewer-prompt.md` | No changes |

## Context Budget

With 3 personas, the orchestrator holds 3 full review reports simultaneously during synthesis. This is acceptable given current context window sizes (200k+ tokens). The 3-persona cap keeps this bounded. If future experience shows reports are excessively long, the cap provides a natural throttle.

## What This Does NOT Change

- Single-persona behavior is entirely unchanged.
- The reviewer prompt template (`references/reviewer-prompt.md`) is untouched.
- The `--reviewer`, `--context`, and plan path arguments work exactly as before.
- The review dimensions (Feasibility, Architecture, Risk) are unchanged.
