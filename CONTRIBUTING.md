# Contributing

Thanks for your interest in improving this skill.

## Reporting bugs

Open an issue using the **Bug report** template. Include:

- What you asked Claude Code to sweep (project type, approximate file count)
- Which focus areas were selected
- What went wrong (missed findings, false positives, wrong severity, crash)
- Your Claude Code version (`claude --version`)

## Suggesting features

Open an issue using the **Feature request** template. Describe the use case and any alternatives you considered.

## Making changes

The skill has one part:

- **SKILL.md** — the full specification that tells Claude how to orient on the codebase, dispatch parallel subagents for each focus area, and produce the ranked markdown report. Edit this to change the sweep procedure, focus area checklists, severity definitions, or report format.

### Testing locally

1. Clone the repo into your skills directory:
   ```bash
   git clone https://github.com/yuenshingyan/code-sweep-claude-skill ~/.claude/skills/code-sweep
   ```
2. Make your changes to `SKILL.md`.
3. Open any Rust project in Claude Code and run `/code-sweep`.
4. Verify the report covers the expected focus areas and findings.

### Pull requests

- Keep PRs focused — one change per PR.
- Test that the sweep still produces correct findings after your changes.
- Update `CHANGELOG.md` with a summary under an "Unreleased" heading.

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
