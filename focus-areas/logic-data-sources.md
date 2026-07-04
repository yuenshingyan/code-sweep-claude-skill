# Logic & Data Sources

Check that queries match their intent:

- **Stale source**: Function queries a cached/denormalized field/table when it should query the source of truth
- **Wrong join direction**: A join or relation traversal expressed backwards relative to the actual foreign-key direction
- **Missing filter**: Returns all rows/records when it should filter by project/user/status/tenant
- **Overly broad filter**: Filters by a superset when it should filter by a subset
- **Stale cache**: Reads a denormalized/cached field that could be out of sync with the source table
- **Semantic mismatches**: Function name doesn't match behavior; return values ignored; invariants assumed but not enforced; invalid state transitions possible
- **Incorrect assumptions**: Race conditions on read-then-write without a transaction/lock; silent data loss via broad exception swallowing or silent fallback values in place of surfacing a failure; integer overflow/truncation via unchecked numeric conversions; crashes on empty collections
