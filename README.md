# code-sweep

A Claude Code skill that sweeps Rust codebases for logical errors, inefficiencies, data integrity issues, concurrency bugs, error handling problems, boundary validation gaps, and resource leaks. Uses parallel subagents for each focus area. Read-only — produces a ranked markdown report directly in the chat. Makes no edits, no commits.

## Focus Areas

1. **Logic & Data Sources** — wrong queries, stale sources, semantic mismatches, incorrect assumptions
2. **Query & Computation Efficiency** — N+1 queries, redundant computation, over-fetching, unbounded queries
3. **Async & Concurrency** — lock-across-await, sequential awaits, fire-and-forget, shared mutable state
4. **Data Integrity** — partial writes, orphaned records, transaction boundaries, soft-delete leaks
5. **Error Handling** — silent swallowing, wrong mapping, lost context, panic in library code
6. **Boundary Validation** — missing input validation, inconsistent checks, enum deserialization, path traversal
7. **Resource & Memory** — unbounded growth, unclosed streams, connection pool exhaustion, leaked tasks

## How It Works

- Asks which focus areas to run (or runs all 7)
- Orients on your stack by reading `Cargo.toml` and entry points
- Dispatches parallel subagents — one per selected focus area
- Merges and deduplicates findings into a single ranked report
- Classifies each finding as **Bug**, **Inefficiency**, or **Smell**

## Install

Clone this repo directly into your Claude Code skills directory:

**macOS / Linux:**

```bash
git clone https://github.com/yuenshingyan/code-sweep-claude-skill ~/.claude/skills/code-sweep
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/yuenshingyan/code-sweep-claude-skill "$env:USERPROFILE\.claude\skills\code-sweep"
```

That's it. Claude Code auto-discovers skills in `~/.claude/skills/` (`%USERPROFILE%\.claude\skills\` on Windows).

### Update

**macOS / Linux:**

```bash
cd ~/.claude/skills/code-sweep && git pull
```

**Windows (PowerShell):**

```powershell
cd "$env:USERPROFILE\.claude\skills\code-sweep"; git pull
```

### Uninstall

**macOS / Linux:**

```bash
rm -rf ~/.claude/skills/code-sweep
```

**Windows (PowerShell):**

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\code-sweep"
```

## Usage

In any Rust project, use one of:

- `/code-sweep` — invokes the skill directly
- "sweep for bugs" — natural language trigger
- "check logic and async" — natural language with specific focus areas

Claude will ask which focus areas to run, analyze your `.rs` files, and produce a ranked markdown report grouped by severity (Bug > Inefficiency > Smell) with clickable file:line references and fix sketches.

## Output

The report is rendered directly in the chat as markdown — no external files are generated. Each finding includes:

- Clickable `file:line` reference
- The problematic code snippet
- What's wrong
- A brief fix sketch
- Focus area tag (e.g., [Logic], [Async], [Boundary])

## License

MIT
