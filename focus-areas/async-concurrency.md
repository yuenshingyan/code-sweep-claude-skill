# Async & Concurrency

**Async bugs:**
- **Lock held across a suspension point**: A mutex/lock guard still held while the code awaits/yields, risking deadlock
- **Sequential awaits that could be concurrent**: Two independent async operations awaited one after another when the runtime offers a way to run them concurrently instead
- **Fire-and-forget spawns**: A background task/goroutine/thread started without capturing its handle or errors — failures silently swallowed

**Concurrency bugs:**
- **Shared mutable state without synchronization**: Global/static mutable state accessed from multiple threads/tasks without a lock or atomic
- **Unsound thread-safety overrides**: Manual assertions or annotations claiming a type/value is safe to share across threads when that safety hasn't actually been verified
- **Weak memory ordering**: Atomics using a relaxed ordering where a stronger one is needed for correctness
- **Inconsistent lock ordering**: Multiple locks acquired in different orders across call sites, risking deadlock
