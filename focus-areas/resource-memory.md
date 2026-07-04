# Resource & Memory

- **Unbounded collection growth**: Collections that grow with user-controlled input without any cap
- **Unclosed streams/connections**: DB connections, file handles, sockets, or HTTP streams opened but never explicitly closed or released when the language/runtime requires explicit cleanup
- **Connection pool exhaustion**: Long-held connections (e.g., streaming results while doing other work) that could starve the pool
- **Large allocations in loops**: Allocating large buffers or strings inside a loop that could be allocated once and reused
- **Unbounded queues/channels**: An unbounded queue or channel where a slow consumer causes unbounded memory growth
- **Leaked background tasks**: Spawned tasks/threads/goroutines that never complete and have no cancellation mechanism
