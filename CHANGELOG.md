# Changelog

## [1.1.0] - 2026-03-16

### Added
- **Parallel multi-persona review** — pass comma-separated personas (max 3) to dispatch independent reviewer sub-agents in parallel
- Orchestrator synthesizes raw reports into a single unified output with convergence attribution (`[N/N personas]` or `[K/N — names]`)
- Severity reconciliation — overlapping issues use the highest severity assigned by any persona
- New **Consensus & Disagreements** section in multi-persona output
- Graceful degradation when individual reviewers fail
- Codex backend supports per-persona temp files with PID suffix to avoid collisions

### Unchanged
- Single-persona behavior is identical to v1.0.0
- Reviewer prompt template (`references/reviewer-prompt.md`) untouched
- `--reviewer`, `--context`, and plan path arguments work as before

## [1.0.0] - 2026-03-15

### Added
- Initial release
- Claude backend (subagent via Agent tool)
- Codex backend (OpenAI Codex CLI)
- Custom reviewer personas (role-based or named person)
- Supplementary context via `--context` (URLs or local files)
- Severity-ranked output format (Critical/Important/Minor/Strengths/Overall Assessment)
