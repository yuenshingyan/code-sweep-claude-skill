---
name: code-sweep
description: >-
  Sweep logical fallacies, inefficiencies, data integrity issues, 
  concurrency bugs, error handling problems, boundary validation gaps, 
  resource leaks, and rendering/state bugs. Runs the selected focus 
  area as one or more parallel subagents, sharded by file count on 
  large codebases. Read-only report with ranked findings.
when_to_use: >-
  TRIGGER when the user /code-sweep. Not for style/naming 
  (use /rust-best-practices) or security audits.
effort: max
allowed-tools:
  - Read
  - Bash(rg *)
  - Bash(grep *)
  - Bash(find *)
  - Bash(wc *)
  - Bash(git blame *)
  - Bash(git log *)
  - AskUserQuestion
  - Agent
---

**Prioritize by blast radius**: request handlers/routed pages > shared business logic/components & hooks > background jobs/one-off scripts & isolated utilities. This ordering drives shard order and truncation order in Step 2.

## Procedure

### 0. Ask the user which focus area to run

Before anything else, ask the user to pick exactly one focus area via a single-choice dialog:

**Prompt:**
> Which focus area should this sweep cover?

**Options (single-select):**
1. **Logic & Data Sources** — wrong queries, stale sources, semantic mismatches, incorrect assumptions
2. **Query & Computation Efficiency** — N+1 queries, redundant computation, over-fetching, unbounded queries
3. **Async & Concurrency** — lock-across-suspension, sequential awaits, fire-and-forget, shared mutable state, weak atomics
4. **Data Integrity** — partial writes, orphaned records, transaction boundaries, soft-delete leaks
5. **Error Handling** — silent swallowing, wrong mapping, lost context, crashes in library code
6. **Boundary Validation** — missing input validation, inconsistent checks, enum/type deserialization, path traversal
7. **Resource & Memory** — unbounded growth, unclosed streams, connection pool exhaustion, leaked tasks
8. **Frontend Logic & State** — incorrect conditionals, stale closures, wrong effect/watcher dependencies, race conditions in async state
9. **Frontend Bugs & Rendering** — missing/wrong list keys, unbounded re-renders, memory leaks, reactivity-rule violations

Run only the selected area, and wait for the response before proceeding.

### 1. Orient — understand the stack

Read the project's manifest/dependency file and entrypoint first to learn the language and frameworks in use — don't assume; let what you find determine which idioms are intentional.

Locate the data-access layer by reading code, not by grepping for one library's API by name. Read a few call sites that query, save, or mutate stored records to learn the real data-access vocabulary, then reuse it for later grep passes.

**If a frontend focus area is selected**, also identify the frontend framework/templating approach from the manifest and a representative file, so subagents know which state/effect/lifecycle idioms are intentional for it. If multiple frameworks have substantial file counts, or none is clearly identifiable despite frontend files existing, ask the user via `AskUserQuestion` rather than guess.

### 2. Dispatch the focus-area subagent(s)

Count the files relevant to the selected area (from Step 1):

- **≤30 files**: one subagent for the whole area.
- **>30 files**: shard into groups of at most 15, ordered by blast-radius priority (highest-priority first), one subagent per shard — up to a cap of 4 shards (60 files) — all in parallel. If files remain beyond that, note the lowest-priority remainder as unevaluated in the final report rather than dropping it silently.

Always dispatch as subagent(s), never review inline. This keeps each subagent's context limited to just its checklist and assigned files, isolated from the orchestrator's conversation (the picker dialog, orientation steps, etc.) — that isolation is what keeps findings accurate, whether one subagent runs or several shards do.

Each area's full checklist lives in its own file under this skill's `focus-areas/` directory (resolve paths relative to the skill's base directory, not the project being audited) — kept separate so neither the orchestrator nor a subagent has to load all nine at once:

- **Logic & Data Sources**: `focus-areas/logic-data-sources.md`
- **Query & Computation Efficiency**: `focus-areas/query-computation-efficiency.md`
- **Async & Concurrency**: `focus-areas/async-concurrency.md`
- **Data Integrity**: `focus-areas/data-integrity.md`
- **Error Handling**: `focus-areas/error-handling.md`
- **Boundary Validation**: `focus-areas/boundary-validation.md`
- **Resource & Memory**: `focus-areas/resource-memory.md`
- **Frontend Logic & State**: `focus-areas/frontend-logic-state.md`
- **Frontend Bugs & Rendering**: `focus-areas/frontend-bugs-rendering.md`

Don't read the selected area's file yourself before dispatching — that would defeat the point of splitting them out. Instead, give each subagent the absolute path to the checklist file (every shard of one area uses the same file) and instruct it to Read that file first, before anything else.

Each subagent should:
1. Read its assigned focus-area file (only that file) to get its checklist
2. Read only its assigned files (all of the area's relevant files if unsharded, or just its shard if sharded)
3. Produce findings in the standard severity format (Bug / Inefficiency / Smell)
4. Return its section of the report

### 3. Validate findings against git history

Mandatory for **every** finding from every subagent — Bug, Inefficiency, and Smell alike, no matter how many there are. Don't sample or stop early: a report that validates some findings and skips others is unreliable. If turn budget is tight, validate in smaller batches across multiple turns rather than skip the remainder.

For each finding, run these three commands to check whether the code was introduced deliberately:

```bash
# Who introduced this line and in which commit
git blame -L <line>,<line> --porcelain <file>

# What the introducing commit said
git log -1 --format="%s%n%b" <hash>

# Recent change activity on the file
git log --oneline -n 5 -- <file>
```

Multiple findings often land in the same file — cache the `git log --oneline -n 5 -- <file>` output per file and reuse it across findings in that file instead of re-running it, to cut down commands on large sweeps.

False-positive signals to look for:

- **Commit message explains the pattern** — e.g., "intentionally suppress error here", "workaround for upstream bug", "by design — no rollback needed". Strong evidence of deliberate code.
- **Pattern survived multiple subsequent commits** — the file changed several times after the line was introduced, yet the line was never touched. Reviewers likely accepted it.
- **Line introduced alongside a comment or test** acknowledging the same behavior.

Handle likely false positives without dropping them silently:

- **Strong signal** (commit message directly addresses the pattern): demote to **Smell** and append a note — e.g., `(git: a1b2c3 "workaround for upstream bug" — appears intentional; verify with author)`
- **Weak signal** (code is old and stable, no explanation in history): keep the original severity but append — e.g., `(git: unchanged since 2023-04 — may be intentional)`

Findings with no git signal indicating intent keep their original severity unchanged.

**Before moving to Step 4**, confirm the number of findings you validated equals the total number produced by all subagents. If they don't match, finish validating the remainder first — don't merge or finalize a report with unvalidated findings.

### 4. Merge and finalize report

Merge all subagents' findings into a single report. If the selected area was sharded (Step 2), this is a plain concatenation — shards cover disjoint files, so no cross-shard dedup is needed.

### 5. Fix findings

Loop until the user is done:

#### 5a. Present finding picker

Build a multi-select `AskUserQuestion` listing every fixable finding from the report. Findings demoted to likely-false-positive during Step 3's git validation are excluded from the picker by default, unless the user explicitly asks to include them.

Format each option as:

- **label**: `[Severity] file:line — short summary` (e.g., `[Bug] src/server.rs:142 — stale annotator query`)
- **description**: The fix sketch from the report so the user knows what will change

If the user's follow-up message already specifies an unambiguous scope (e.g., "fix all the bugs", "fix everything"), skip the picker for the first iteration and select the matching findings automatically — but still present the picker on subsequent iterations if unfixed findings remain.

#### 5b. Fix selected findings

1. **Partition by file to avoid conflicts.** Group the user's selected findings by file. Every finding for a given file must be handled by exactly one agent — never let two parallel agents hold edit access to the same file at the same time. Small unrelated single-finding files may be packed into one agent's batch to keep fan-out sane, but a single file's findings must never be split across two agents.
2. **Dispatch fix agents in parallel.** Launch one Agent (general-purpose type) per file-partition, all in a single message with multiple tool calls — the same "run in parallel, not sequentially" convention as Step 2. Give each agent the complete list of findings for its file(s) (exact `file:line`, the problem, and the fix sketch from the report) so it applies all fixes to that file in one pass. Scope each agent's prompt strictly to its assigned file(s) and instruct it not to touch anything else.
3. **Fix fidelity.** Agents must apply the fix sketch as written — the smallest diff that resolves the issue, per the "Fix sketches must be inline, not abstracted" judgment call below. If the code has drifted since the report was generated and the sketch no longer applies cleanly, the agent should skip that finding and report why rather than guessing.

#### 5c. Verify and report

After all fix agents complete, run whatever build/typecheck/lint command the project provides (check the manifest's scripts/tasks, or ask the user if none is obvious) to confirm the combined edits are still sound. Report which findings were fixed vs. skipped and why, grouped by file.

#### 5d. Loop or exit

If unfixed findings remain (unselected in 5a or skipped during 5b), loop back to **5a** and present the remaining findings in a new picker. The user can select more to fix, or pick nothing / select "Other" to stop. Exit the loop when:

- The user selects no findings (empty selection or explicitly declines), OR
- All findings have been fixed or skipped

## Report format

Markdown report grouped by severity. Each entry: `file:line` (clickable), the problematic code snippet, what's wrong, and a brief fix sketch.

Severity levels:
- **Bug** — produces wrong results, data loss, or incorrect state. Fix immediately.
- **Inefficiency** — correct results but wasteful (N+1, redundant work, unnecessary allocations). Fix when convenient.
- **Smell** — not wrong today but fragile; likely to break as code evolves. Note for awareness.

```md
## Logic & Data Sources sweep

### Bugs
- [handlers/assignments:142](handlers/assignments#L142) — `get_assignments` reads the live-assignment list for email lookup, but removed assignees disappear from that list. Should read the assignment-history record instead.
  **Fix**: Query the history table filtered by item IDs and extract unique assignee IDs.

### Inefficiencies
- [server/annotations:454](server/annotations#L454) — N+1 query: loops over items and issues one count query per item.
  **Fix**: Single grouped count query across all item IDs.

### Smells
- [server/export:170](server/export#L170) — Three set rebuilds from the same source collection look redundant but are correct (each mutates in between). Add a comment to prevent future "cleanup" that breaks it.

### Summary
| Bugs | Inefficiencies | Smells |
|---|---|---|
| 1 | 1 | 1 |
```

Name the report after the focus area that was run (as in the heading above) — since only one area runs per sweep, individual findings don't need area tags.

If no issues are found, say so explicitly with a summary of what was checked (file count, function count) so the user knows the sweep ran thoroughly.

## Judgment calls

- **Framework idioms override general rules.** Documented lifecycle/reactivity conventions of the identified framework are intentional — don't flag them just because a generic rule suggests otherwise.
- **Small N is fine.** An N+1 loop over 3-5 items on an admin-only endpoint is a smell, not an inefficiency. Focus on user-facing paths with unbounded N.
- **Transactions have cost.** Don't recommend wrapping everything in a transaction — only flag multi-step mutations where interleaving would corrupt data.
- **Don't flag tested patterns.** A pattern used consistently and working across the codebase is a convention, not a bug — flag only if you can show incorrect results.
- **Distinguish "wrong" from "suboptimal".** A late in-memory filter after a broad query is suboptimal, not wrong. A query against the wrong table/collection IS wrong — rank accordingly.
- **Context matters for data-source bugs.** The same field can be right for one question and wrong for another (e.g., a live-assignment list is correct for "who's currently assigned?" but wrong for "who's ever worked on this?") — understand intent before flagging.
- **Don't flag cheap copies.** Copying small primitives, short strings, or lightweight references is cheap in most runtimes — only flag copies of large structures or copies inside hot loops with large N.
- **Don't flag intentional error suppression** where a comment or context shows the error is deliberately ignored (e.g., best-effort logging, shutdown cleanup).
- **Fix sketches must be inline, not abstracted.** Never wrap a one-liner in a helper function. The fix should be the smallest diff that resolves the issue — a changed expression, a different call, an added limit/pagination clause.
- **Boundary validation is contextual.** Internal endpoints between trusted services need less validation than public APIs — understand the trust boundary before flagging.
- **Don't flag test/story-only code.** Files scoped to tests, stories, or fixtures are exempt from rendering/state findings — they intentionally exercise edge cases.
- **New identity per render isn't automatically a bug.** Many frameworks re-create inline functions/objects every render by default — only flag it as a Frontend Bugs finding when it demonstrably defeats memoization on an expensive child.

## What this skill does NOT do

- Edit files or fix issues automatically — fixes happen only if the user explicitly asks as a follow-up after reviewing the report (Step 5), never during the initial sweep.
- Commit anything.
- Run the application or write tests.
- Audit code style, naming, or formatting (use `rust-best-practices`).
- Audit for dead code (use `check-dead-backend-code`).
- Audit security vulnerabilities (OWASP-style, e.g. XSS/CSRF) — only logical correctness/efficiency, back-end or front-end (use `security-review` for security).
- Audit frontend accessibility (a11y) or CSS/visual styling — only logic and rendering correctness.
