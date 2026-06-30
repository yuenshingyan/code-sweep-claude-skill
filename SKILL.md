---
name: code-sweep
description: >-
  Sweep Rust codebase for logical errors, inefficiencies, data
  integrity issues, concurrency bugs, error handling problems,
  boundary validation gaps, and resource leaks. Uses parallel
  subagents for each focus area. Read-only report with ranked
  findings.
when_to_use: >-
  TRIGGER when the user asks to "check logic",
  "find inefficiencies", "sweep for bugs", "code sweep",
  "find bugs in the code", "audit the code",
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
  - AskUserQuestion
  - Agent
---

# code-sweep

Read-only sweep of a Rust codebase across 7 focus areas, run in parallel. Finds bugs that compile fine but produce wrong results, and code that works but wastes resources. Makes no edits, no commits — just a ranked report.

## Scope

If the user names specific files or directories, audit only those. Otherwise audit all `.rs` files under `src/`, skipping `target/` and generated code.

**Prioritize by blast radius**: request handlers > shared business logic > background jobs > one-off scripts. If >30 `.rs` files, audit the top 10-15 by importance and note unevaluated files in the report.

## Procedure

### 0. Ask the user which focus areas to run

Before doing anything else, present a selection dialog to the user. Use the tool or UI available in your environment to ask a multiple-choice question:

**Prompt:**
> Which focus areas should this sweep cover?

**Options (multi-select):**
1. **All** — run all 7 focus areas
2. **Logic & Data Sources** — wrong queries, stale sources, semantic mismatches, incorrect assumptions
3. **Query & Computation Efficiency** — N+1 queries, redundant computation, over-fetching, unbounded queries
4. **Async & Concurrency** — lock-across-await, sequential awaits, fire-and-forget, shared mutable state, atomics
5. **Data Integrity** — partial writes, orphaned records, transaction boundaries, soft-delete leaks
6. **Error Handling** — silent swallowing, wrong mapping, lost context, panic in library code
7. **Boundary Validation** — missing input validation, inconsistent checks, enum deserialization, path traversal
8. **Resource & Memory** — unbounded growth, unclosed streams, connection pool exhaustion, leaked tasks

If the user selects **All**, run all 7. Otherwise run only the selected areas. Wait for the user's response before proceeding.

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

### 2. Dispatch parallel subagents

After orientation, launch **the selected focus areas as parallel subagents** (from Step 0). Each subagent gets its own context window and works independently. Do NOT run them sequentially — use parallel tool calls so they all run at the same time. Only dispatch focus areas the user selected.

Each subagent should:
1. Read only the files relevant to its focus area
2. Produce findings in the standard severity format (Bug / Inefficiency / Smell)
3. Return its section of the report

The 7 focus areas and their instructions:

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

### 3. Merge reports

After all 7 subagents complete, merge their findings into a single report. Deduplicate any findings that overlap across focus areas (e.g., a transaction boundary issue found by both Logic and Data Integrity). Keep the higher severity classification if they differ.

## Report format

Markdown report grouped by severity. Each entry: `file:line` (clickable), the problematic code snippet, what's wrong, and a brief fix sketch.

Severity levels:
- **Bug** — produces wrong results, data loss, or incorrect state. Fix immediately.
- **Inefficiency** — correct results but wasteful (N+1, redundant work, unnecessary allocations). Fix when convenient.
- **Smell** — not wrong today but fragile; likely to break as code evolves. Note for awareness.

```md
## Rust logic & efficiency sweep

### Bugs
- [src/server.rs:142](src/server.rs#L142) — **[Logic]** `get_assignments` queries `item.annotators` array for email lookup, but removed annotators disappear from that array. Should query `annotations` table for `created_by` IDs instead.
  **Fix**: `Annotation::find().filter(annotation::Column::ItemId.is_in(item_ids)).all(db).await` and extract unique `created_by` values.

### Inefficiencies
- [src/annotation_server.rs:454](src/annotation_server.rs#L454) — **[Efficiency]** N+1 query: loops over items and issues one `COUNT(*)` per item.
  **Fix**: Single query with `.group_by(annotation::Column::ItemId).count().all(db).await`.

### Smells
- [src/export_server.rs:170](src/export_server.rs#L170) — **[Efficiency]** Three `HashSet<i32>` rebuilds from `data.items` look redundant but are correct (each `retain` mutates). Add a comment to prevent future "cleanup" that breaks it.

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
| **Total** | **1** | **1** | **1** |
```

Tag each finding with its focus area in brackets (e.g., **[Logic]**, **[Async]**, **[Boundary]**) so the user knows which area surfaced it.

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

## What this skill does NOT do

- Edit files or fix issues automatically.
- Commit anything.
- Run the application or write tests.
- Audit code style, naming, or formatting (use `rust-best-practices` for that).
- Audit for dead code (use `check-dead-backend-code` for that).
- Audit for security vulnerabilities (OWASP-style) — only logical correctness and efficiency.
