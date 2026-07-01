# Changelog

## Unreleased

- Added git history validation (Step 3): checks each finding against `git blame`/`git log` to demote or annotate likely-intentional patterns as false positives
- Fixed: `allowed-tools` was missing `Bash(git blame *)` / `Bash(git log *)`, so every git-history check triggered a manual permission prompt — large sweeps could silently stall partway through validation. Both commands are now pre-approved
- Step 3 now explicitly requires validating every finding (not just a sample) and gates Step 4 on the validated count matching the total finding count

## 1.0.0 — 2026-06-23

Initial public release.

- Parallel sweep of Rust codebases across 7 focus areas
- Interactive focus area selection before sweep begins
- Stack-aware orientation via Cargo.toml and entry point analysis
- Parallel subagent dispatch — one per selected focus area
- Severity classification: Bug, Inefficiency, Smell
- Ranked markdown report with clickable file:line references and fix sketches
- ORM-aware grep patterns for SeaORM, Diesel, sqlx, and raw queries
- Priority-by-blast-radius for large codebases (>30 files)
- Judgment calls to reduce false positives (framework idioms, small N, intentional patterns)
