# Error Handling

- **Silent swallowing**: Broad catch-and-ignore patterns that discard a failure without acting on it — distinguish intentional suppression from accidental
- **Wrong error mapping**: Catching a specific error but mapping everything to one generic failure response when some cases warrant a more specific one
- **Lost error context**: Re-wrapping or mapping an error in a way that discards the original message or type
- **Retry on non-retryable**: Retrying operations that failed due to validation or permission errors, not transient ones
- **Crashes in shared/library code**: An unguarded failure (unhandled exception, forced unwrap, assertion) in shared code paths where the caller has no way to catch or recover from it
- **Error type mismatches**: A function's declared/returned error type doesn't match what callers actually expect, papered over by a lossy conversion
