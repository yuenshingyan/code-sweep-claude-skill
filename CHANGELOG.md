# Changelog

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
