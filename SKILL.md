---
name: code-sweep
description: >-
  Sweep logical fallacies, inefficiencies, data integrity issues, 
  concurrency bugs, error handling problems, boundary validation gaps, 
  resource leaks, and rendering/state bugs. Uses parallel subagents 
  for each focus area. Read-only report with ranked findings.
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

**Prioritize by blast radius**: request handlers / top-level routed pages/components > shared business logic / shared components & hooks > background jobs / one-off scripts & isolated utility components. If >30 files in a language track, audit the top 10-15 by importance and note unevaluated files in the report.

## Procedure

### 0. Ask the user which focus areas to run

Before doing anything else, present a selection dialog to the user. Use the tool or UI available in your environment to ask a multiple-choice question:

**Prompt:**
> Which focus areas should this sweep cover?

**Options (multi-select):**
1. **All** — run all 9 focus areas
2. **Logic & Data Sources** — wrong queries, stale sources, semantic mismatches, incorrect assumptions
3. **Query & Computation Efficiency** — N+1 queries, redundant computation, over-fetching, unbounded queries
4. **Async & Concurrency** — lock-across-suspension, sequential awaits, fire-and-forget, shared mutable state, weak atomics
5. **Data Integrity** — partial writes, orphaned records, transaction boundaries, soft-delete leaks
6. **Error Handling** — silent swallowing, wrong mapping, lost context, crashes in library code
7. **Boundary Validation** — missing input validation, inconsistent checks, enum/type deserialization, path traversal
8. **Resource & Memory** — unbounded growth, unclosed streams, connection pool exhaustion, leaked tasks
9. **Frontend Logic & State** — incorrect conditionals, stale closures, wrong effect/watcher dependencies, race conditions in async state
10. **Frontend Bugs & Rendering** — missing/wrong list keys, unbounded re-renders, memory leaks, reactivity-rule violations

If the user selects **All**, run all 9. Otherwise run only the selected areas. Wait for the user's response before proceeding.

If the user already specified focus areas in their original message (e.g., "just check async and error handling"), skip this dialog and use their selection directly.

### 1. Orient — understand the stack

Before any analysis, read the project's manifest/dependency file and entrypoint to learn the language and any frameworks in use. Don't assume a specific language or library — discover it by reading, and let what you find determine which idioms are intentional and which "don't flag this" exceptions apply.

Locate the data-access layer by reading the code, not by grepping for one library's API by name. Read a handful of call sites that read or write persisted data — functions/methods whose names or usage suggest they query, save, or mutate stored records — to learn the actual data-access API in use, then use that vocabulary for your own grep passes for the rest of this focus area.

**If Frontend Logic & State or Frontend Bugs & Rendering are selected**, also identify the frontend framework (or templating/rendering approach) in use from the manifest and a representative file, so subagents know which idioms — state declarations, effects/derivations, lifecycle hooks — are intentional for it. If more than one frontend approach is present with substantial file counts, or none is clearly identifiable despite frontend files existing, ask the user which to target via `AskUserQuestion` before dispatching the frontend subagents — do not guess.

### 2. Dispatch parallel subagents

After orientation, launch **the selected focus areas as parallel subagents** (from Step 0). Each subagent gets its own context window and works independently. Do NOT run them sequentially — use parallel tool calls so they all run at the same time. Only dispatch focus areas the user selected.

Each subagent should:
1. Read only the files relevant to its focus area
2. Produce findings in the standard severity format (Bug / Inefficiency / Smell)
3. Return its section of the report

The 9 focus areas and their instructions:

---

#### Focus Area A: Logic & Data Sources

Check that queries match their intent:

- **Stale source**: Function queries a cached/denormalized field/table when it should query the source of truth
- **Wrong join direction**: A join or relation traversal expressed backwards relative to the actual foreign-key direction
- **Missing filter**: Returns all rows/records when it should filter by project/user/status/tenant
- **Overly broad filter**: Filters by a superset when it should filter by a subset
- **Stale cache**: Reads a denormalized/cached field that could be out of sync with the source table
- **Semantic mismatches**: Function name doesn't match behavior; return values ignored; invariants assumed but not enforced; invalid state transitions possible
- **Incorrect assumptions**: Race conditions on read-then-write without a transaction/lock; silent data loss via broad exception swallowing or silent fallback values in place of surfacing a failure; integer overflow/truncation via unchecked numeric conversions; crashes on empty collections

---

#### Focus Area B: Query & Computation Efficiency

- **N+1 queries**: A query/fetch call inside a loop that hits the DB — helper functions may hide the query
- **Unbatched queries**: Multiple sequential single-row queries that could use `IN (...)`, a join, or a bulk fetch
- **Over-fetching**: Full records/objects fetched when only a count or single field is needed
- **Missing index hints**: Queries filtering on columns that likely aren't indexed
- **SELECT * for existence checks**: Fetching a full record just to check if something exists
- **Unbounded queries**: No limit/pagination on user-facing list endpoints
- **Redundant queries**: Same data fetched multiple times within the same request/handler
- **Recomputed values**: Values computed repeatedly inside a loop that could be computed once before it
- **Duplicate collection passes**: Multiple iterations over the same collection that could be merged into one
- **Unnecessary copies**: Deep-copying/cloning large structures in loops (ignore cheap copies of small primitives or reference-counted handles)
- **Collect-then-iterate**: Materializing a collection immediately before iterating it exactly once
- **Rebuilding data structures**: A lookup structure (map/set) rebuilt on every call when it could be built once and reused

---

#### Focus Area C: Async & Concurrency

**Async bugs:**
- **Lock held across a suspension point**: A mutex/lock guard still held while the code awaits/yields, risking deadlock
- **Sequential awaits that could be concurrent**: Two independent async operations awaited one after another when the runtime offers a way to run them concurrently instead
- **Fire-and-forget spawns**: A background task/goroutine/thread started without capturing its handle or errors — failures silently swallowed

**Concurrency bugs:**
- **Shared mutable state without synchronization**: Global/static mutable state accessed from multiple threads/tasks without a lock or atomic
- **Unsound thread-safety overrides**: Manual assertions or annotations claiming a type/value is safe to share across threads when that safety hasn't actually been verified
- **Weak memory ordering**: Atomics using a relaxed ordering where a stronger one is needed for correctness
- **Inconsistent lock ordering**: Multiple locks acquired in different orders across call sites, risking deadlock

---

#### Focus Area D: Data Integrity

- **Partial writes without rollback**: Multi-step mutations where a failure mid-way leaves data in an inconsistent state, not wrapped in a transaction
- **Orphaned records**: Deletion of a parent entity without cleaning up related child records
- **Foreign key assumptions**: Code assumes referential integrity that isn't enforced at the DB level
- **Inconsistent data sources**: Two functions that should return the same data but query different tables/fields
- **Duplicate logic**: Same business rule implemented in two places that could drift apart
- **Transaction boundaries**: Multi-step mutations that should be atomic but aren't wrapped in a transaction
- **Soft-delete leaks**: Queries that don't filter out soft-deleted rows when they should, returning already-deleted records

---

#### Focus Area E: Error Handling

- **Silent swallowing**: Broad catch-and-ignore patterns that discard a failure without acting on it — distinguish intentional suppression from accidental
- **Wrong error mapping**: Catching a specific error but mapping everything to one generic failure response when some cases warrant a more specific one
- **Lost error context**: Re-wrapping or mapping an error in a way that discards the original message or type
- **Retry on non-retryable**: Retrying operations that failed due to validation or permission errors, not transient ones
- **Crashes in shared/library code**: An unguarded failure (unhandled exception, forced unwrap, assertion) in shared code paths where the caller has no way to catch or recover from it
- **Error type mismatches**: A function's declared/returned error type doesn't match what callers actually expect, papered over by a lossy conversion

---

#### Focus Area F: Boundary Validation

- **Missing input validation**: API endpoints that accept user input without checking length, range, format, or type constraints
- **Inconsistent validation**: One endpoint validates a field (e.g., email format) but another endpoint accepting the same field doesn't
- **String length before DB insert**: Strings accepted without length limits that will hit DB column limits and error at write time
- **Enum/type deserialization**: Accepting serialized values that the business logic can't handle or doesn't expect
- **Negative/zero values**: Numeric inputs (pagination, quantities, IDs) not checked for negative or zero when those are nonsensical
- **Path traversal**: File paths or identifiers built from user input without sanitization

---

#### Focus Area G: Resource & Memory

- **Unbounded collection growth**: Collections that grow with user-controlled input without any cap
- **Unclosed streams/connections**: DB connections, file handles, sockets, or HTTP streams opened but never explicitly closed or released when the language/runtime requires explicit cleanup
- **Connection pool exhaustion**: Long-held connections (e.g., streaming results while doing other work) that could starve the pool
- **Large allocations in loops**: Allocating large buffers or strings inside a loop that could be allocated once and reused
- **Unbounded queues/channels**: An unbounded queue or channel where a slow consumer causes unbounded memory growth
- **Leaked background tasks**: Spawned tasks/threads/goroutines that never complete and have no cancellation mechanism

---

#### Focus Area H: Frontend Logic & State

Check for logical fallacies and faulty reasoning in state/derivations — bugs that render or build fine but produce the wrong value or wrong screen:

- **Incorrect boolean/conditional logic**: De Morgan's-law mistakes, negation errors, operator precedence bugs in boolean chains
- **Stale closures**: Callbacks or reactive effects capturing outdated state/props from an earlier render/update cycle
- **Wrong reactive dependencies**: Missing dependencies in a reactive computation (an effect, memo, computed value, or watcher) causing stale reads, or extra dependencies causing thrashing
- **Duplicated derived state**: A value copied into local state instead of computed on render/update, able to drift out of sync with its source
- **Off-by-one errors**: List slicing, pagination offsets, or index math that's one short/long
- **Reference-equality bugs**: Comparing objects/arrays by reference instead of value, causing missed updates; loose vs. strict equality/type-coercion bugs
- **Async state races**: Out-of-order fetch/promise responses applied to state with no request-id or cancellation guard, so a stale response can overwrite a fresher one
- **Wrong truthy/falsy assumptions**: Treating `0`, `''`, or `NaN`/`None` as "no value" incorrectly, or vice versa
- **Direct state mutation**: Mutating an array/object in place instead of via the framework's setter/store API, silently missing re-renders
- **Lossy merges**: Spread/merge operations that drop nested fields the caller expected to survive

---

#### Focus Area I: Frontend Bugs & Rendering

- **Missing/incorrect list key**: List rendering without a stable unique key, or keyed by array index where items reorder — causes state to bleed across the wrong items
- **Unbounded re-renders**: State set directly in the render body, or an effect whose own state update re-triggers itself
- **Memory leaks**: Event listeners, timers, subscriptions, or observers registered without a corresponding cleanup/teardown
- **Update-after-unmount**: State/store updates from an async callback that resolves after the component has unmounted, with no cleanup or abort
- **Defeated memoization**: Inline function/object/array literals passed as props to a memoized child, forcing it to re-render every time anyway
- **Reactivity-rule violations**: Reactive primitives invoked conditionally, in loops, or after an early return, when the framework requires a stable call/registration order
- **Controlled/uncontrolled input flip**: An input's bound value switches between `undefined`/unset and a defined value across renders
- **Duplicate fetch/subscription**: An effect re-running (e.g. double-invoke in strict/dev mode, or a changed dependency) fires the same fetch or subscription again without guarding against duplicates

---

### 3. Validate findings against git history

This step is mandatory for **every** finding from every subagent — Bug, Inefficiency, and Smell alike — no matter how many there are. Do not sample or stop early: a report that validates some findings and skips others is inconsistent and unreliable. If turn budget is tight, validate in smaller batches across multiple turns rather than skipping the remainder.

For each finding, run these three commands to check whether the code was introduced deliberately:

```bash
# Who introduced this line and in which commit
git blame -L <line>,<line> --porcelain <file>

# What the introducing commit said
git log -1 --format="%s%n%b" <hash>

# Recent change activity on the file
git log --oneline -n 5 -- <file>
```

Multiple findings often land in the same file — cache the `git log --oneline -n 5 -- <file>` output per file and reuse it across findings in that file instead of re-running it, to cut down the total number of commands on large sweeps.

Look for these false-positive signals:

- **Commit message explains the pattern** — e.g., "intentionally suppress error here", "workaround for upstream bug", "by design — no rollback needed". Strong evidence the code is deliberate.
- **Pattern survived multiple subsequent commits** — the file was changed several times after the suspicious line was introduced, yet the line was never touched. Reviewers likely accepted it.
- **Line introduced alongside a comment or test** that acknowledges the same behavior.

Handle likely false positives without silently dropping them:

- **Strong signal** (commit message directly addresses the pattern): demote the finding to **Smell** and append a note — e.g., `(git: a1b2c3 "workaround for upstream bug" — appears intentional; verify with author)`
- **Weak signal** (code is old and stable, no explanation in history): keep the original severity but append — e.g., `(git: unchanged since 2023-04 — may be intentional)`

Findings with no git signal indicating intent keep their original severity unchanged.

**Before moving to Step 4**, confirm the number of findings you validated equals the total number of findings produced by all subagents. If they don't match, resume Step 3 on the remaining findings first — do not merge or finalize a report with unvalidated findings.

### 4. Merge and finalize report

After all subagents complete and git validation is done, merge their findings into a single report. Deduplicate any findings that overlap across focus areas (e.g., a transaction boundary issue found by both Logic and Data Integrity). Keep the higher severity classification if they differ.

### 5. Optional follow-up: fix findings (only if explicitly requested)

This step never runs as part of a normal sweep — the sweep itself stays read-only. Only enter this step if, in a later message, the user explicitly asks Claude to fix, apply, or resolve the findings from a prior sweep report (e.g. "fix these", "apply the fixes", "fix all issues you found").

Run the following loop until the user is done:

#### 5a. Present finding picker

Build a multi-select `AskUserQuestion` listing every fixable finding from the report. Findings demoted to likely-false-positive during Step 3's git validation are excluded from the picker by default unless the user explicitly asks to include them.

Format each option as:

- **label**: `[Severity] file:line — short summary` (e.g., `[Bug] src/server.rs:142 — stale annotator query`)
- **description**: The fix sketch from the report so the user knows what will change

If the user's follow-up message already specifies an unambiguous scope (e.g., "fix all the bugs", "fix everything"), skip the picker for the first iteration and select the matching findings automatically — but still present the picker on subsequent iterations if unfixed findings remain.

#### 5b. Fix selected findings

1. **Partition by file to avoid conflicts.** Group the user's selected findings by file. Every finding for a given file must be handled by exactly one agent — never let two parallel agents hold edit access to the same file at the same time. Small unrelated single-finding files may be packed into one agent's batch to keep fan-out sane, but a single file's findings must never be split across two agents.
2. **Dispatch fix agents in parallel.** Launch one Agent (general-purpose type) per file-partition, all in a single message with multiple tool calls — the same "run in parallel, not sequentially" convention as Step 2. Give each agent the complete list of findings for its file(s) (exact `file:line`, the problem, and the fix sketch from the report) so it applies all fixes to that file in one pass. Explicitly scope each agent's prompt to only its assigned file(s) and instruct it not to touch anything else.
3. **Fix fidelity.** Agents must apply the fix sketch as written — the smallest diff that resolves the issue, per the "Fix sketches must be inline, not abstracted" judgment call below. If the code has drifted since the report was generated and the sketch no longer applies cleanly, the agent should skip that finding and report why rather than guessing.

#### 5c. Verify and report

After all fix agents complete, run whatever build/typecheck/lint command the project provides (check for one in the manifest's scripts/tasks, or ask the user if none is obvious) to confirm the combined edits are still sound. Report which findings were fixed vs. skipped and why, grouped by file.

#### 5d. Loop or exit

If unfixed findings remain (either unselected in 5a or skipped during 5b), loop back to **5a** and present the remaining findings in a new picker. The user can select more to fix, or pick nothing / select "Other" to stop. Exit the loop when:

- The user selects no findings (empty selection or explicitly declines), OR
- All findings have been fixed or skipped

## Report format

Markdown report grouped by severity. Each entry: `file:line` (clickable), the problematic code snippet, what's wrong, and a brief fix sketch.

Severity levels:
- **Bug** — produces wrong results, data loss, or incorrect state. Fix immediately.
- **Inefficiency** — correct results but wasteful (N+1, redundant work, unnecessary allocations). Fix when convenient.
- **Smell** — not wrong today but fragile; likely to break as code evolves. Note for awareness.

```md
## Logic & efficiency sweep

### Bugs
- [handlers/assignments:142](handlers/assignments#L142) — **[Logic]** `get_assignments` reads the live-assignment list for email lookup, but removed assignees disappear from that list. Should read the assignment-history record instead.
  **Fix**: Query the history table filtered by item IDs and extract unique assignee IDs.
- [components/Search:58](components/Search#L58) — **[Frontend Logic]** Fetch results are applied to state with no request-id guard, so a slow earlier keystroke's response can overwrite a newer one.
  **Fix**: Track a request id/cancellation token; ignore responses that don't match the latest request.

### Inefficiencies
- [server/annotations:454](server/annotations#L454) — **[Efficiency]** N+1 query: loops over items and issues one count query per item.
  **Fix**: Single grouped count query across all item IDs.

### Smells
- [server/export:170](server/export#L170) — **[Efficiency]** Three set rebuilds from the same source collection look redundant but are correct (each mutates in between). Add a comment to prevent future "cleanup" that breaks it.
- [components/ItemList:31](components/ItemList#L31) — **[Frontend Bugs]** List keyed by array index while items are reorderable via drag-and-drop, causing input state to bleed onto the wrong row after a reorder.
  **Fix**: Key by item id instead of index.

### Summary
| Focus area | Bugs | Inefficiencies | Smells |
|---|---|---|---|
| Logic & Data Sources | 1 | 0 | 0 |
| Query & Computation | 0 | 1 | 1 |
| Async & Concurrency | 0 | 0 | 0 |
| Data Integrity | 0 | 0 | 0 |
| Error Handling | 0 | 0 | 0 |
| Boundary Validation | 0 | 0 | 0 |
| Resource & Memory | 0 | 0 | 0 |
| Frontend Logic & State | 1 | 0 | 0 |
| Frontend Bugs & Rendering | 0 | 0 | 1 |
| **Total** | **2** | **1** | **2** |
```

Tag each finding with its focus area in brackets (e.g., **[Logic]**, **[Async]**, **[Boundary]**, **[Frontend Logic]**, **[Frontend Bugs]**) so the user knows which area surfaced it.

If no issues are found, say so explicitly with a summary of what was checked (file count, function count) so the user knows the sweep ran thoroughly.

## Judgment calls

- **Framework idioms override general rules.** Once you've identified the framework/library in use, its documented lifecycle and reactivity conventions are intentional — don't flag them as bugs just because a generic rule would suggest otherwise.
- **Small N is fine.** An N+1 loop over 3-5 items in an admin-only endpoint is a smell, not an inefficiency. Focus on user-facing paths with unbounded N.
- **Transactions have cost.** Don't recommend wrapping everything in a transaction — only flag multi-step mutations where interleaving would corrupt data.
- **Don't flag tested patterns.** If a pattern is used consistently across the codebase and works, it's a convention, not a bug. Only flag if you can demonstrate incorrect results.
- **Distinguish "wrong" from "suboptimal".** A late-stage in-memory filter after a broad query is suboptimal but not wrong. A query against the wrong table/collection IS wrong. Rank accordingly.
- **Context matters for data source bugs.** The same field can be correct for one question and wrong for another (e.g., a live assignment list is correct for "who is currently assigned?" but wrong for "who has ever worked on this?") — understand the intent before flagging.
- **Don't flag cheap copies.** Copying small primitives, short strings, or lightweight reference handles is cheap in most languages/runtimes. Only flag copies of large structures or copies inside hot loops with large N.
- **Don't flag intentional error suppression** where a comment or context makes clear the error is deliberately ignored (e.g., best-effort logging, cleanup on shutdown).
- **Fix sketches must be inline, not abstracted.** Never suggest wrapping a one-liner in a helper function. The fix sketch should be the smallest diff that resolves the issue — a changed expression, a different method call, an added limit/pagination clause. If the existing code is already concise, the fix must be equally concise.
- **Boundary validation is contextual.** Internal-only endpoints between trusted services need less validation than public-facing APIs. Understand the trust boundary before flagging.
- **Don't flag test/story-only code.** Files clearly scoped to tests, stories, or fixtures (following whatever testing convention the project uses) are exempt from rendering/state findings — they intentionally exercise edge cases.
- **New identity per render isn't automatically a bug.** Many frameworks re-create inline functions/objects on every render by default; only flag this as a Frontend Bugs finding when it demonstrably defeats a memoization mechanism on an expensive child, not as a blanket style rule.

## What this skill does NOT do

- Edit files or fix issues automatically — fixes only happen if the user explicitly asks as a follow-up after reviewing the report (see Step 5), never as part of the initial sweep.
- Commit anything.
- Run the application or write tests.
- Audit code style, naming, or formatting (use `rust-best-practices` for that).
- Audit for dead code (use `check-dead-backend-code` for that).
- Audit for security vulnerabilities (OWASP-style, e.g. XSS/CSRF) — only logical correctness and efficiency, back-end or front-end (use `security-review` for security).
- Audit frontend accessibility (a11y) or CSS/visual styling — only logic and rendering correctness.
