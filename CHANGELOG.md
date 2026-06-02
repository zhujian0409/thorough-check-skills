# Changelog

## 1.1.1 - 2026-06-02

### Added

- Stable report labels for `Old value / New value`, `Upstream / Downstream`, `Excluded files`, and `Next run`.
- Dirty Git/SVN working-copy guidance to separate commit/deploy scope from temporary, generated, QA, report, cache, and build files.
- Regression eval coverage for selective commit boundaries in dirty working copies.

### Changed

- Updated Codex display metadata so implicit prompts emphasize old/new values, upstream/downstream tracing, and excluded files.

## 1.1.0 - 2026-05-20

### Added

- Default read-only reviewer agents for real verification runs:
  - `impact reviewer` for missed blast-radius discovery.
  - `final reviewer` for disproving unsupported pass claims.
- Block 0 impact mapping before the existing verification blocks.
- State-machine and lifecycle migration checks for behavior-changing edits.
- Regression-surface guidance for choosing checks by risk.
- Codex display metadata in `thorough-check/agents/openai.yaml`.
- Regression prompts in `thorough-check/evals/evals.json`.
- Codex installation instructions alongside Claude Code installation instructions.

### Changed

- Expanded call-chain checks to include concept aliases, old/new data, queued work, cache, rollback, mixed deployment, and external integration boundaries.
- Updated English and Chinese READMEs to document impact mapping and default reviewer agents.
- Reworded project description as Codex- and Claude Code-compatible.
- Bumped plugin metadata to `1.1.0`.

### Fixed

- Removed ambiguous wording that could make agent review appear optional for token-saving reasons.
- Added fallback report examples for environments where subagents cannot be spawned.
- Corrected the outdated `4 required directions` heading after the checklist expanded.

## 1.0.0 - 2026-04-22

### Added

- Initial `thorough-check` skill.
- Evidence-first A/B/C/E verification blocks:
  - change-site verification,
  - boundary-condition simulation,
  - file-integrity checks,
  - upstream/downstream call-chain review.
- English and Chinese README documentation.
