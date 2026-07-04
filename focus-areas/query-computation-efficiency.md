# Query & Computation Efficiency

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
