# Data Integrity

- **Partial writes without rollback**: Multi-step mutations where a failure mid-way leaves data in an inconsistent state, not wrapped in a transaction
- **Orphaned records**: Deletion of a parent entity without cleaning up related child records
- **Foreign key assumptions**: Code assumes referential integrity that isn't enforced at the DB level
- **Inconsistent data sources**: Two functions that should return the same data but query different tables/fields
- **Duplicate logic**: Same business rule implemented in two places that could drift apart
- **Transaction boundaries**: Multi-step mutations that should be atomic but aren't wrapped in a transaction
- **Soft-delete leaks**: Queries that don't filter out soft-deleted rows when they should, returning already-deleted records
