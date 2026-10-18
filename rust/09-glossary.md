# 9. Glossary

Reference, not reading. Each entry points at the chapter that explains it. Where a Go
term is the nearest equivalent, it's in the meaning.

| Term | Meaning | Where |
|---|---|---|
| **`'static`** | As a bound (`T: 'static`): the value holds no borrowed references, so it's fine to keep for ever. Not "lives for ever" | [02](02-ownership-and-borrowing.md), [06](06-async-and-tokio.md) |
| **Aliasing XOR mutation** | Data may be shared or mutated, never both at once; the borrowing rule in one phrase | [02](02-ownership-and-borrowing.md) |
| **`anyhow`** | Crate providing one catch-all error type plus `.context()`; for application code | [03](03-errors.md) |
| **`Arc<T>`** | Atomically reference-counted shared ownership; usable across threads. Gives `&T` only | [05](05-shared-state-and-threads.md) |
| **`async fn`** | A function that returns a future instead of running; its body becomes a state machine | [06](06-async-and-tokio.md) |
| **Auto trait** | A trait the compiler implements automatically from a type's fields: `Send`, `Sync` | [04](04-traits-and-generics.md) |
| **`.await`** | Polls a future to completion, yielding to the executor whenever it's pending. The only point where a task can be suspended or cancelled | [06](06-async-and-tokio.md) |
| **`biased;`** | Option to `select!` that polls branches in written order instead of randomly | [07](07-concurrency-patterns.md) |
| **Blocking** | Holding a thread without yielding: sync I/O, `thread::sleep`, long CPU work. Inside async code it stalls every task on that worker | [06](06-async-and-tokio.md) |
| **Borrow** | A reference (`&T` or `&mut T`) that uses a value without owning it | [02](02-ownership-and-borrowing.md) |
| **Borrow checker** | The compiler pass that enforces ownership, borrowing and lifetimes | [02](02-ownership-and-borrowing.md) |
| **`Box<T>`** | Single-owner heap allocation; also how you own a `dyn Trait` | [05](05-shared-state-and-threads.md) |
| **`broadcast`** | tokio channel where every receiver gets every value; slow receivers get `Lagged` | [07](07-concurrency-patterns.md) |
| **`CancellationToken`** | tokio-util type for cooperative cancellation; the nearest thing to `context.WithCancel` | [07](07-concurrency-patterns.md) |
| **Cancellation safe** | A future that can be dropped before completing and recreated without losing anything | [07](07-concurrency-patterns.md) |
| **Cargo** | Rust's build tool and package manager: `go build`, `go test`, `go mod` in one | [01](01-foundations.md), [08](08-cargo-and-project-hygiene.md) |
| **`Cargo.lock`** | Records the exact dependency versions Cargo resolved; makes builds reproducible | [08](08-cargo-and-project-hygiene.md) |
| **`cargo check`** | Type- and borrow-check without generating code; the fast feedback loop | [01](01-foundations.md) |
| **`Cell<T>`** | Single-threaded interior mutability by copying values in and out | [05](05-shared-state-and-threads.md) |
| **Clippy** | Rust's lint tool, `cargo clippy` | [08](08-cargo-and-project-hygiene.md) |
| **`Clone`** | Trait for explicit duplication with `.clone()`; depth follows each field's `Clone` | [02](02-ownership-and-borrowing.md) |
| **Cooperative scheduling** | Tasks run until they yield (at `.await`); the runtime can't preempt them. Goroutines are preempted | [06](06-async-and-tokio.md) |
| **`Copy`** | Marker for plain values that are copied, not moved, on assignment | [02](02-ownership-and-borrowing.md) |
| **Crate** | Rust's compilation unit: a library or a binary, built from one root file | [01](01-foundations.md) |
| **Data race** | Two threads accessing the same memory at once, at least one writing, unsynchronised. Impossible in safe Rust | [02](02-ownership-and-borrowing.md), [04](04-traits-and-generics.md) |
| **`derive`** | Attribute asking the compiler to write trait impls: `#[derive(Debug, Clone)]` | [04](04-traits-and-generics.md) |
| **Drop** | Running a value's destructor and freeing it when its owner goes out of scope; `Drop` is the trait | [02](02-ownership-and-borrowing.md), [05](05-shared-state-and-threads.md) |
| **Dyn compatible** | A trait that can be used as `dyn Trait` (formerly "object safe"). Traits with `async fn` aren't | [04](04-traits-and-generics.md) |
| **`dyn Trait`** | A trait object: data pointer plus vtable; dynamic dispatch, like a Go interface value | [04](04-traits-and-generics.md) |
| **Edition** | Opt-in, per-crate language version (2021, 2024); not a toolchain minimum | [08](08-cargo-and-project-hygiene.md) |
| **Elision** | Rules that let the compiler infer lifetimes in common signatures | [02](02-ownership-and-borrowing.md) |
| **Executor** | The part of a runtime that polls futures; tokio's scheduler | [06](06-async-and-tokio.md) |
| **Feature** | Named, additive compile-time option of a package; unified across the dependency graph | [08](08-cargo-and-project-hygiene.md) |
| **`Fn` / `FnMut` / `FnOnce`** | Traits for closures that only read, mutate, or consume their captures | [04](04-traits-and-generics.md) |
| **Future** | A value representing a computation in progress, driven by `poll`. Lazy: does nothing until polled | [06](06-async-and-tokio.md) |
| **Interior mutability** | Mutation through a shared reference, with the check moved to run time (`RefCell`) or made safe by a lock or atomic | [05](05-shared-state-and-threads.md) |
| **`JoinError`** | What awaiting a `JoinHandle` returns if the task panicked or was cancelled | [03](03-errors.md), [06](06-async-and-tokio.md) |
| **`JoinHandle`** | Handle to a spawned thread or task; await or join it for the result. Dropping a tokio one detaches the task | [06](06-async-and-tokio.md), [07](07-concurrency-patterns.md) |
| **`JoinSet`** | tokio collection of tasks returned in completion order; aborts them all when dropped | [07](07-concurrency-patterns.md) |
| **Lifetime** | A compile-time name (`'a`) for the region where a reference is valid. Describes, never extends | [02](02-ownership-and-borrowing.md) |
| **Marker trait** | A trait with no methods that states a property of a type (`Send`, `Sync`, `Copy`) | [04](04-traits-and-generics.md) |
| **`match`** | Exhaustive pattern matching; every variant must be handled | [01](01-foundations.md) |
| **Module** | A namespace within a crate, declared with `mod`; private by default. Nearest Go term: package | [01](01-foundations.md) |
| **Monomorphisation** | Compiling a separate copy of a generic function per concrete type; static dispatch | [04](04-traits-and-generics.md) |
| **Move** | Transfer of ownership on assignment, argument passing or return; the source becomes unusable | [02](02-ownership-and-borrowing.md) |
| **`mpsc`** | Multi-producer, single-consumer channel; in both `std::sync` and `tokio::sync` | [05](05-shared-state-and-threads.md), [07](07-concurrency-patterns.md) |
| **`Mutex<T>`** | A lock that owns the data it protects; access only through the guard from `lock()` | [05](05-shared-state-and-threads.md) |
| **`MutexGuard`** | The value `lock()` returns; unlocks when dropped. Not `Send` | [05](05-shared-state-and-threads.md), [06](06-async-and-tokio.md) |
| **`oneshot`** | tokio channel carrying exactly one value; the standard reply channel | [07](07-concurrency-patterns.md) |
| **`Option<T>`** | `Some(T)` or `None`; replaces `nil` | [01](01-foundations.md), [03](03-errors.md) |
| **Orphan rule** | You may implement a trait for a type only if the trait or the type is local to your crate | [04](04-traits-and-generics.md) |
| **Ownership** | Every value has exactly one owner, and is dropped when the owner goes out of scope | [02](02-ownership-and-borrowing.md) |
| **Package** | A directory with a `Cargo.toml`, building at most one library and any number of binaries. Nearest Go term: module | [01](01-foundations.md) |
| **Panic** | Unrecoverable-error mechanism for bugs; unwinds the thread. Kills only that thread or task, unlike a Go goroutine panic | [03](03-errors.md) |
| **`Pin`** | Wrapper guaranteeing a value won't move in memory; needed to poll self-referential futures | [06](06-async-and-tokio.md) |
| **Poisoning** | A `std::sync::Mutex` whose holder panicked reports it on the next `lock()` | [03](03-errors.md), [05](05-shared-state-and-threads.md) |
| **`poll`** | The one method of `Future`: make progress, return `Ready` or `Pending` and arrange a wake-up | [06](06-async-and-tokio.md) |
| **`?`** | Return early with the error (converted via `From`) if a `Result` is `Err`; unwrap if `Ok` | [03](03-errors.md) |
| **`Rc<T>`** | Single-threaded reference-counted shared ownership; neither `Send` nor `Sync` | [05](05-shared-state-and-threads.md) |
| **Reactor** | The part of an async runtime that waits on OS events and wakes tasks; Go's netpoller | [06](06-async-and-tokio.md) |
| **`RefCell<T>`** | Single-threaded interior mutability with borrow rules checked at run time; panics on violation | [05](05-shared-state-and-threads.md) |
| **Resolver** | Cargo's dependency and feature resolution algorithm; version `"1"`, `"2"` or `"3"` | [08](08-cargo-and-project-hygiene.md) |
| **`Result<T, E>`** | `Ok(T)` or `Err(E)`; replaces the `(T, error)` pair | [03](03-errors.md) |
| **Scoped thread** | A thread from `std::thread::scope`, guaranteed joined before the scope ends, so it may borrow locals | [05](05-shared-state-and-threads.md) |
| **`select!`** | tokio macro that waits on several futures and drops the losers | [07](07-concurrency-patterns.md) |
| **`Send`** | Marker: ownership of the type may move to another thread | [04](04-traits-and-generics.md) |
| **`spawn_blocking`** | Runs a blocking closure on tokio's separate blocking thread pool; can't be aborted | [06](06-async-and-tokio.md) |
| **Static dispatch** | Method calls resolved at compile time, via generics | [04](04-traits-and-generics.md) |
| **`Sync`** | Marker: `&T` may be shared between threads | [04](04-traits-and-generics.md) |
| **Task** | A future spawned onto an async runtime; tokio's goroutine | [06](06-async-and-tokio.md) |
| **`TaskTracker`** | tokio-util type for waiting on tasks without keeping their outputs or aborting on drop | [07](07-concurrency-patterns.md) |
| **`thiserror`** | Derive macro for hand-designed error enums; for code whose callers branch on errors | [03](03-errors.md) |
| **tokio** | The usual async runtime for networked Rust: scheduler, reactor, timers, sync primitives | [06](06-async-and-tokio.md) |
| **Trait** | A named set of methods (or a marker) that types implement explicitly; Rust's interface | [04](04-traits-and-generics.md) |
| **Trait object** | See `dyn Trait` | [04](04-traits-and-generics.md) |
| **Unwinding** | Running destructors up the stack after a panic; `panic = 'abort'` skips it | [03](03-errors.md) |
| **Vtable** | Table of function pointers behind a `dyn Trait`; Go's itab | [04](04-traits-and-generics.md) |
| **`watch`** | tokio channel holding only the latest value; for config and state changes | [07](07-concurrency-patterns.md) |
| **Workspace** | Several Cargo packages built together with one lockfile and target directory | [08](08-cargo-and-project-hygiene.md) |
