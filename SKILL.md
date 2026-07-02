---
name: code-sweep
description: >-
  Sweep Rust and/or frontend (JS/TS, React, Vue, Svelte, Angular)
  code for logical errors, logical fallacies, inefficiencies, data
  integrity issues, concurrency bugs, error handling problems,
  boundary validation gaps, resource leaks, and rendering/state
  bugs. Uses parallel subagents for each focus area. Read-only
  report with ranked findings.
when_to_use: >-
  TRIGGER when the user asks to "check logic",
  "find inefficiencies", "sweep for bugs", "code sweep",
  "find bugs in the code", "audit the code",
  "sweep the frontend", "find bugs in my React/Vue/Svelte components",
  or similar. Not for style/naming (use /rust-best-practices)
  or security audits.
effort: max
allowed-tools:
  - Read
  - Bash(rg *)
  - Bash(grep *)
  - Bash(find *)
  - Bash(cargo clippy *)
  - Bash(wc *)
  - Bash(git blame *)
  - Bash(git log *)
  - AskUserQuestion
  - Agent
---

# code-sweep

Read-only sweep of a Rust and/or frontend (JS/TS) codebase across 9 focus areas, run in parallel. Finds bugs that compile fine but produce wrong results, code that works but wastes resources, and frontend logic/rendering bugs. Makes no edits, no commits — just a ranked report.

## Scope

If the user names specific files or directories, audit only those. Otherwise audit all `.rs` files under `src/` (skipping `target/` and generated code) and, when frontend focus areas are selected, all `.js`/`.jsx`/`.ts`/`.tsx`/`.vue`/`.svelte` files under typical frontend roots (`src/`, `app/`, `components/`, `pages/`), skipping `node_modules/`, `dist/`, `build/`, `.next/`, and other generated/build output.

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
4. **Async & Concurrency** — lock-across-await, sequential awaits, fire-and-forget, shared mutable state, atomics
5. **Data Integrity** — partial writes, orphaned records, transaction boundaries, soft-delete leaks
6. **Error Handling** — silent swallowing, wrong mapping, lost context, panic in library code
7. **Boundary Validation** — missing input validation, inconsistent checks, enum deserialization, path traversal
8. **Resource & Memory** — unbounded growth, unclosed streams, connection pool exhaustion, leaked tasks
9. **Frontend Logic & State** — incorrect conditionals, stale closures, wrong effect/watcher dependencies, race conditions in async state
10. **Frontend Bugs & Rendering** — missing/wrong `key` props, unbounded re-renders, memory leaks, rules-of-hooks violations

If the user selects **All**, run all 9. Otherwise run only the selected areas. Wait for the user's response before proceeding.

If the user already specified focus areas in their original message (e.g., "just check async and error handling"), skip this dialog and use their selection directly.

### 1. Orient — understand the stack

Before any analysis, skim `Cargo.toml` for the ORM, DB driver, and async runtime, and `src/main.rs` or `src/lib.rs` for module structure. This tells you which query patterns to look for and which framework idioms are intentional.

Identify server functions and data-fetching code. Adapt grep patterns to the ORM in use:

**SeaORM:**
```bash
rg '#\[server\]' src/ --type rust -l
rg 'Entity::find|::find_by_id|::find_related|execute.*query' src/ --type rust -l
```

**Diesel:**
```bash
rg 'table::table\.filter|\.load::<|\.execute\(' src/ --type rust -l
```

**sqlx:**
```bash
rg 'sqlx::query|query_as!|query!' src/ --type rust -l
```

**Raw / other:**
```bash
rg '\.fetch_one|\.fetch_all|\.execute\(' src/ --type rust -l
```

**If Frontend Logic & State or Frontend Bugs & Rendering are selected**, also detect the frontend framework so subagents know which idioms are intentional:

1. Read `package.json` `dependencies`/`devDependencies` for `react`, `vue`, `svelte`, `@angular/core`, `solid-js`, `next`, `nuxt`.
2. Confirm with a grep pass:

```bash
rg 'useEffect|useState|useMemo|useCallback' -g '*.tsx' -g '*.jsx' -l   # React
rg '<script setup>|defineComponent|ref\(|reactive\(' -g '*.vue' -l    # Vue
rg '\$:|export let ' -g '*.svelte' -l                                  # Svelte
rg '@Component|@Injectable|@Input|@Output' -g '*.ts' -l                # Angular
```

3. If exactly one framework is clearly present, proceed using its idioms. If `package.json` lists more than one frontend framework with substantial file counts, or no framework is detected despite frontend files existing, ask the user which framework to target via `AskUserQuestion` before dispatching the frontend subagents — do not guess.

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

- **Stale source**: Function queries a cached/denormalized field when it should query the source-of-truth table
- **Wrong join direction**: Joins A→B when the relationship is B→A
- **Missing filter**: Returns all rows when it should filter by project/user/status
- **Overly broad filter**: Filters by a superset when it should filter by a subset
- **Stale cache**: Reads a denormalized field that could be out of sync with the source table
- **Semantic mismatches**: Function name doesn't match behavior; return values ignored; invariants assumed but not enforced; invalid state transitions possible
- **Incorrect assumptions**: Race conditions on read-then-write without transactions; silent data loss via `.ok()`, `unwrap_or_default()`, `let _ =`; integer overflow/truncation via unchecked `as` casts; panics on empty collections

---

#### Focus Area B: Query & Computation Efficiency

- **N+1 queries**: `.await` calls inside `for`/`while` loops that hit the DB — helper functions may hide the query
- **Unbatched queries**: Multiple sequential queries that could use `IN (...)` or `JOIN`
- **Over-fetching**: Full models fetched when only a count or single column is needed
- **Missing index hints**: Queries filtering on columns that likely aren't indexed
- **SELECT * for existence checks**: Fetching full models just to check if something exists
- **Unbounded queries**: No `.limit()` on user-facing list endpoints
- **Redundant queries**: Same data fetched multiple times in the same request handler
- **Recomputed values**: Values computed in a loop that could be computed once before it
- **Duplicate collection passes**: Multiple `.iter()` passes that could be merged
- **Unnecessary clones**: `.clone()` of large structs in loops (ignore small types: `i32`, short `String`, `Arc`)
- **Collect-then-iterate**: `.collect::<Vec<_>>()` immediately followed by `.iter()`
- **Rebuilding data structures**: `HashMap`/`HashSet` rebuilt on every call when it could be built once

---

#### Focus Area C: Async & Concurrency

**Async bugs:**
- **Lock held across `.await`**: A `Mutex`/`RwLock` guard alive across an await point risks deadlock
- **Sequential awaits**: Two independent async calls awaited one after the other — should use `join!`/`try_join!`
- **Fire-and-forget spawns**: `tokio::spawn` without storing or awaiting the `JoinHandle` — failures silently swallowed

**Concurrency bugs:**
- **Shared mutable state**: `static mut`, global mutable state without synchronization
- **Unsound Send/Sync**: Manual `unsafe impl Send` or `unsafe impl Sync` that may not be valid
- **Atomic ordering**: Atomics with `Ordering::Relaxed` where `SeqCst` or `AcqRel` is needed for correctness
- **Lock ordering**: Multiple locks acquired in inconsistent order across call sites, risking deadlock

---

#### Focus Area D: Data Integrity

- **Partial writes without rollback**: Multi-step mutations where a failure mid-way leaves data in an inconsistent state, not wrapped in a transaction
- **Orphaned records**: Deletion of a parent entity without cleaning up related child records
- **Foreign key assumptions**: Code assumes referential integrity that isn't enforced at the DB level
- **Inconsistent data sources**: Two functions that should return the same data but query different tables/fields
- **Duplicate logic**: Same business rule implemented in two places that could drift apart
- **Transaction boundaries**: Multi-step mutations that should be atomic but aren't wrapped in a transaction
- **Soft-delete leaks**: Queries that don't filter by `deleted_at IS NULL` when they should, returning "deleted" records

---

#### Focus Area E: Error Handling

- **Silent swallowing**: `.ok()`, `unwrap_or_default()`, or `let _ =` hiding meaningful failures — distinguish intentional suppression from accidental
- **Wrong error mapping**: Catching a specific error but mapping all errors to generic 500 when some should be 400/404/409
- **Lost error context**: `.map_err(|_| ...)` or `.map_err(|e| SomeError::Generic)` discarding the original error message/type
- **Retry on non-retryable**: Retrying operations that failed due to validation or permission errors, not transient issues
- **Panic in library code**: `unwrap()` or `expect()` in shared code paths where the caller can't catch the panic
- **Error type mismatches**: Function returns `Result<_, A>` but callers expect `Result<_, B>`, hiding via `.map_err` that loses information

---

#### Focus Area F: Boundary Validation

- **Missing input validation**: API endpoints that accept user input without checking length, range, format, or type constraints
- **Inconsistent validation**: One endpoint validates a field (e.g., email format) but another endpoint accepting the same field doesn't
- **String length before DB insert**: Strings accepted without length limits that will hit DB column limits and error at write time
- **Enum deserialization**: Accepting serialized enum values that the business logic can't handle or doesn't expect
- **Negative/zero values**: Numeric inputs (pagination, quantities, IDs) not checked for negative or zero when those are nonsensical
- **Path traversal**: File paths or identifiers built from user input without sanitization

---

#### Focus Area G: Resource & Memory

- **Unbounded Vec growth**: Collections that grow with user-controlled input without any cap
- **Unclosed streams/connections**: DB connections, file handles, or HTTP streams opened but not explicitly closed or dropped
- **Connection pool exhaustion**: Long-held DB connections (e.g., streaming results while doing other work) that could starve the pool
- **Large allocations in loops**: Allocating large buffers or strings inside a loop that could be allocated once and reused
- **Unbounded channels**: `mpsc::unbounded_channel` or similar where a slow consumer causes unbounded memory growth
- **Leaked tasks**: Spawned tasks that never complete (infinite loops without cancellation tokens)

---

#### Focus Area H: Frontend Logic & State

Check for logical fallacies and faulty reasoning in state/derivations — bugs that render or build fine but produce the wrong value or wrong screen:

- **Incorrect boolean/conditional logic**: De Morgan's-law mistakes, negation errors, operator precedence bugs in `&&`/`||` chains
- **Stale closures**: Callbacks or effects capturing outdated state/props from an earlier render
- **Wrong dependency arrays**: Missing deps in `useEffect`/`useMemo`/`useCallback` (or a watcher/computed equivalent) causing stale reads, or extra deps causing thrashing
- **Duplicated derived state**: A value copied into local state instead of computed on render, able to drift out of sync with its source
- **Off-by-one errors**: List slicing, pagination offsets, or index math that's one short/long
- **Reference-equality bugs**: Comparing objects/arrays by reference instead of value, causing missed updates; `==` vs `===` type-coercion bugs
- **Async state races**: Out-of-order fetch/promise responses applied to state with no request-id or `AbortController` guard, so a stale response can overwrite a fresher one
- **Wrong truthy/falsy assumptions**: Treating `0`, `''`, or `NaN` as "no value" incorrectly, or vice versa
- **Direct state mutation**: Mutating an array/object in place (`.push`, field assignment) instead of via the setter/store API, silently missing re-renders
- **Lossy spread merges**: `{...a, ...b}` or array spreads that drop nested fields the caller expected to survive the merge

---

#### Focus Area I: Frontend Bugs & Rendering

- **Missing/incorrect `key` prop**: List rendering without a stable unique key, or keyed by array index where items reorder — causes state to bleed across the wrong items
- **Unbounded re-renders**: State set directly in the render body, or an effect whose own state update re-triggers itself
- **Memory leaks**: Event listeners, timers, subscriptions, or observers registered without a corresponding cleanup/teardown
- **Update-after-unmount**: State/store updates from an async callback that resolves after the component has unmounted, with no cleanup or abort
- **Defeated memoization**: Inline function/object/array literals passed as props to a `memo`/`useMemo`-wrapped child, forcing it to re-render every time anyway
- **Rules-of-hooks violations**: Hooks called conditionally, in loops, or after an early return
- **Controlled/uncontrolled input flip**: An input's value prop switches between `undefined` and a defined value across renders
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

1. **Confirm scope.** Ask via `AskUserQuestion` which severities to fix: "All findings" (Bugs + Inefficiencies + Smells), "Bugs + Inefficiencies only" (recommended default), or "Bugs only" — show the finding count in each bucket. Skip the dialog only if the user's follow-up message already specifies scope unambiguously (e.g. "fix all the bugs"), matching the same skip pattern as Step 0. Findings demoted to likely-false-positive during Step 3's git validation are excluded from the fixable set by default, regardless of scope, unless the user explicitly overrides.
2. **Partition by file to avoid conflicts.** Group the in-scope findings by `file`. Every finding for a given file must be handled by exactly one agent — never let two parallel agents hold edit access to the same file at the same time. Small unrelated single-finding files may be packed into one agent's batch to keep fan-out sane, but a single file's findings must never be split across two agents.
3. **Dispatch fix agents in parallel.** Launch one Agent (general-purpose type) per file-partition, all in a single message with multiple tool calls — the same "run in parallel, not sequentially" convention as Step 2. Give each agent the complete list of findings for its file(s) (exact `file:line`, the problem, and the fix sketch from the report) so it applies all fixes to that file in one pass. Explicitly scope each agent's prompt to only its assigned file(s) and instruct it not to touch anything else.
4. **Fix fidelity.** Agents must apply the fix sketch as written — the smallest diff that resolves the issue, per the "Fix sketches must be inline, not abstracted" judgment call below. If the code has drifted since the report was generated and the sketch no longer applies cleanly, the agent should skip that finding and report why rather than guessing.
5. **Verify and summarize.** After all agents complete, run the stack-appropriate check already used elsewhere in this skill (`cargo check`/`cargo clippy` for Rust, `tsc --noEmit` or the project's lint/build script for frontend, if available) to confirm the combined edits compile/typecheck cleanly. Report a final summary of which findings were fixed vs. skipped and why, grouped by file.

## Report format

Markdown report grouped by severity. Each entry: `file:line` (clickable), the problematic code snippet, what's wrong, and a brief fix sketch.

Severity levels:
- **Bug** — produces wrong results, data loss, or incorrect state. Fix immediately.
- **Inefficiency** — correct results but wasteful (N+1, redundant work, unnecessary allocations). Fix when convenient.
- **Smell** — not wrong today but fragile; likely to break as code evolves. Note for awareness.

```md
## Logic & efficiency sweep

### Bugs
- [src/server.rs:142](src/server.rs#L142) — **[Logic]** `get_assignments` queries `item.annotators` array for email lookup, but removed annotators disappear from that array. Should query `annotations` table for `created_by` IDs instead.
  **Fix**: `Annotation::find().filter(annotation::Column::ItemId.is_in(item_ids)).all(db).await` and extract unique `created_by` values.
- [src/components/Search.tsx:58](src/components/Search.tsx#L58) — **[Frontend Logic]** Fetch results are applied to state with no request-id guard, so a slow earlier keystroke's response can overwrite a newer one.
  **Fix**: Track a request id/`AbortController`; ignore responses that don't match the latest request.

### Inefficiencies
- [src/annotation_server.rs:454](src/annotation_server.rs#L454) — **[Efficiency]** N+1 query: loops over items and issues one `COUNT(*)` per item.
  **Fix**: Single query with `.group_by(annotation::Column::ItemId).count().all(db).await`.

### Smells
- [src/export_server.rs:170](src/export_server.rs#L170) — **[Efficiency]** Three `HashSet<i32>` rebuilds from `data.items` look redundant but are correct (each `retain` mutates). Add a comment to prevent future "cleanup" that breaks it.
- [src/components/ItemList.tsx:31](src/components/ItemList.tsx#L31) — **[Frontend Bugs]** List keyed by array index while items are reorderable via drag-and-drop, causing input state to bleed onto the wrong row after a reorder.
  **Fix**: Key by `item.id` instead of index.

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

- **Framework idioms override general rules.** Dioxus `#[server]` functions, `use_server_future`, `use_signal` patterns are intentional — don't flag them.
- **Small N is fine.** An N+1 loop over 3-5 items in an admin-only endpoint is a smell, not an inefficiency. Focus on user-facing paths with unbounded N.
- **Transactions have cost.** Don't recommend wrapping everything in a transaction — only flag multi-step mutations where interleaving would corrupt data.
- **Don't flag tested patterns.** If a pattern is used consistently across the codebase and works, it's a convention, not a bug. Only flag if you can demonstrate incorrect results.
- **Distinguish "wrong" from "suboptimal".** A Rust-side filter after a DB query is suboptimal but not wrong. A query against the wrong table IS wrong. Rank accordingly.
- **Context matters for data source bugs.** `item.annotators` is correct for "who is currently assigned?" but wrong for "who has ever annotated?" — understand the intent before flagging.
- **Don't flag `.clone()` on small types.** Cloning an `i32`, short `String`, or `Arc` is cheap. Only flag clones of large structs or inside hot loops with large N.
- **Don't flag intentional `.ok()` / `let _ =`** where a comment or context makes clear the error is deliberately ignored (e.g., best-effort logging, cleanup on shutdown).
- **Fix sketches must be inline, not abstracted.** Never suggest wrapping a one-liner in a helper function. The fix sketch should be the smallest diff that resolves the issue — a changed expression, a different method call, an added `.limit()`. If the existing code is already concise, the fix must be equally concise.
- **Boundary validation is contextual.** Internal-only endpoints between trusted services need less validation than public-facing APIs. Understand the trust boundary before flagging.
- **Frontend framework idioms override general rules.** A `useEffect(() => {...}, [])` "run once on mount" pattern, Vue's `watchEffect`, and Svelte's reactive `$:` statements are intentional — don't flag the "missing"/"extra" dependency as a bug just because a generic rule would suggest otherwise.
- **Don't flag test/story-only code.** Files under `__tests__`, `*.test.tsx`, `*.spec.ts`, or `*.stories.tsx` are exempt from rendering/state findings — they intentionally exercise edge cases.
- **New identity per render isn't automatically a bug.** React re-creates inline functions/objects on every render by default; only flag this as a Frontend Bugs finding when it demonstrably defeats a `memo`/`useMemo`/`useCallback` on an expensive child, not as a blanket style rule.

## What this skill does NOT do

- Edit files or fix issues automatically — fixes only happen if the user explicitly asks as a follow-up after reviewing the report (see Step 5), never as part of the initial sweep.
- Commit anything.
- Run the application or write tests.
- Audit code style, naming, or formatting (use `rust-best-practices` for that).
- Audit for dead code (use `check-dead-backend-code` for that).
- Audit for security vulnerabilities (OWASP-style, e.g. XSS/CSRF) — only logical correctness and efficiency, back-end or front-end (use `security-review` for security).
- Audit frontend accessibility (a11y) or CSS/visual styling — only logic and rendering correctness.
