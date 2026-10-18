# Rust for a Go Programmer — A Course

A course on Rust taught by contrast with Go, for someone who already writes Go
confidently (HTTP servers, CLIs, concurrent services) and is about to build networked,
concurrent programs in Rust: async networking with tokio, plus work on dedicated OS
threads. It covers the language's working model, ownership and borrowing, errors, traits,
shared state, async, the tokio concurrency patterns that replace channels and `context`,
and the Cargo habits that keep a project healthy.

**This is reference learning material.** It's about Rust, tokio and Go as documented
upstream, not about any particular program. Claims are checked against the Rust Book, the
Reference, the standard library and crate documentation, and the Go spec and blog. Where
behaviour mattered and the docs left room for doubt, I compiled or ran a small program
and show its output, labelled with the versions used: **rustc 1.96.0, tokio 1.53.2,
tokio-util 0.7.19, anyhow 1.0.104, thiserror 2.0.21, Go 1.24.4, on macOS arm64.** Timings
are from one laptop and are there to show a shape, not to be quoted as rates. Compiler
output is trimmed to the lines that matter.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the Go habit, then what Rust does instead and what it buys.
- Every chapter ends with **Check yourself** questions. Answers are in
  [10-exercises.md](10-exercises.md).
- Go-equivalent tables everywhere, because that's the fastest way in for this reader.

## The one thing to understand first

> **Most of what the Go runtime does while your program runs, Rust does while it
> compiles, and the rest is a library you choose.** The garbage collector's question
> ("who frees this, and can anyone still see it?") becomes ownership and borrowing. The
> race detector's question becomes the `Send` and `Sync` traits. The scheduler becomes
> tokio, which schedules cooperatively at `.await`, so blocking is your problem. And
> because a future can be dropped at any `.await`, cancellation becomes something that
> happens *to* your code, not something it checks for.

Every chapter is one face of that sentence.

## Primary sources

| Source | What it settles |
|---|---|
| [The Rust Programming Language](https://doc.rust-lang.org/book/) ("the Book") | Ownership (ch04), errors (ch09), traits and lifetimes (ch10), smart pointers (ch15), threads and `Send`/`Sync` (ch16), async (ch17) |
| [The Rust Reference](https://doc.rust-lang.org/reference/) | Destructor order, overflow behaviour, dyn compatibility |
| [The Rustonomicon: races](https://doc.rust-lang.org/nomicon/races.html) | Exactly what "no data races" does and doesn't promise |
| [Rust standard library docs](https://doc.rust-lang.org/std/) | `Mutex` poisoning, `Send`/`Sync` impls, `thread::scope`, `mpsc`, `mem::forget`, `Future` |
| [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) (the async book) | Background on async; parts are still being written (it contains "chapter TODO" placeholders), so I lean on the Book's ch17 and tokio's docs instead |
| [tokio tutorial](https://tokio.rs/tokio/tutorial) and [topics](https://tokio.rs/tokio/topics/shutdown) | Spawning, `'static` and `Send` bounds, shared state, channels, `select!`, graceful shutdown, bridging with sync code |
| [docs.rs/tokio](https://docs.rs/tokio/latest/tokio/) | `spawn`, `spawn_blocking`, `select!` cancellation safety, `Mutex`, channels, `JoinSet`, runtime defaults |
| [docs.rs/tokio-util](https://docs.rs/tokio-util/latest/tokio_util/) | `CancellationToken`, `TaskTracker` |
| [docs.rs/anyhow](https://docs.rs/anyhow/latest/anyhow/) and [thiserror README](https://github.com/dtolnay/thiserror) | Application versus library error handling |
| [Alice Ryhl, "Async: What is blocking?"](https://ryhl.io/blog/async-what-is-blocking/) (2020) | The 10–100 µs between `.await`s rule of thumb, from a tokio maintainer |
| [The Cargo Book](https://doc.rust-lang.org/cargo/) | Targets, workspaces, features and unification, resolver versions, profiles, `[lints]` |
| [Clippy docs](https://doc.rust-lang.org/clippy/) | Lint groups and default levels |
| [The Go spec](https://go.dev/ref/spec), [Go blog on errors](https://go.dev/blog/go1.13-errors), [loop variables](https://go.dev/blog/loopvar-preview), [defer](https://go.dev/blog/defer-panic-and-recover), [Go modules reference](https://go.dev/ref/mod), [generics implementation](https://github.com/golang/proposal/blob/master/design/generics-implementation-dictionaries-go1.18.md) | The Go side of every comparison |
| [Ross Bencina, "Real-time audio programming 101"](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing) (2011) and [`rtrb`](https://docs.rs/rtrb/latest/rtrb/) | What a real-time thread must not do, and a lock-free ring buffer for it |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations](01-foundations.md) | ⭐ Say where each part of Go's runtime went, read Rust syntax by Go equivalent, use enums and `match`, tell a crate from a module from a package, and run the Cargo commands. **Prerequisite for everything else** |

### Part 1 — The language

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Ownership, moves, borrowing and lifetimes](02-ownership-and-borrowing.md) | ⭐ Predict which borrow-checker error a piece of code will get, name the Go bug each rule prevents, and fix the common errors without cloning everything |
| 3 | [Errors](03-errors.md) | Translate Go error handling to `Result` and `?`, choose between `thiserror` and `anyhow`, and say what a panic does in a thread or task |
| 4 | [Traits and generics, and `Send`/`Sync`](04-traits-and-generics.md) | Choose static or dynamic dispatch, know why an `async fn` trait can't be `dyn`, and read a `Send` error |

### Part 2 — Concurrency

| # | File | After this you can… |
|---|------|---------------------|
| 5 | [Shared state, threads and `Drop`](05-shared-state-and-threads.md) | Pick between `Box`, `Rc`, `Arc`, `Mutex`, `RefCell` and atomics; use OS and scoped threads; replace `defer` with `Drop` |
| 6 | [Async and tokio](06-async-and-tokio.md) | ⭐ Explain lazy futures and cooperative scheduling, satisfy `Send + 'static` on spawn, and find and fix blocking in async code |
| 7 | [Concurrency patterns](07-concurrency-patterns.md) | ⭐ Choose the tokio channel for a Go pattern, write cancellation-safe `select!` loops, shut down gracefully, and hand data between real-time threads and tasks |

### Part 3 — Projects

| # | File | After this you can… |
|---|------|---------------------|
| 8 | [Cargo and project hygiene](08-cargo-and-project-hygiene.md) | Lay out a lib-plus-bin project and a workspace, avoid the feature-unification trap, test with paused time, and set up fmt and Clippy in CI |

### Reference

| # | File | |
|---|------|---|
| 9 | [Glossary](09-glossary.md) | Look things up |
| 10 | [Exercises & answers](10-exercises.md) | Worked answers to every Check yourself, things to try, questions to ask |

Read 1 first, then 2. Chapters 3 and 4 can be read in either order. Chapter 5 needs 2
and 4 (§4.6 on `Send`/`Sync`). Chapter 6 needs 4 and 5; chapter 7 needs 6. Chapter 8 is
independent after chapter 1, and is worth reading before you set up a real project.

**Background, not prerequisites,** in sibling courses:
[interview/cs-fundamentals 02](../interview/cs-fundamentals/02-processes-and-threads.md)
for threads, threading models and Go's scheduler, and
[cs-fundamentals 04](../interview/cs-fundamentals/04-synchronisation-and-deadlock.md) for
races, locks and deadlock. [realtime-audio 02](../realtime-audio/02-realtime-audio-threads.md) covers real-time
audio threads and the lock-free ring buffer in depth.
[go-services 05](../go-services/05-fanning-out.md) and
[06](../go-services/06-copies-that-share-memory.md) are the Go side of two comparisons.
This course links to them instead of re-teaching them.

### Scope, and what changed from the first outline

The outline this course started from had eight topic chapters; they're all here, with
three adjustments:

- **Closures** were missing. They're split between chapter 2 §2.7 (what they capture, and
  `move`), because that's an ownership question, and chapter 4 §4.5 (the `Fn` traits),
  because that's a trait question. A Go programmer needs both to read any `spawn` call.
- **OS threads** (`std::thread`, scoped threads, `std::sync::mpsc`) went into chapter 5
  alongside the shared-state types, since `Arc<Mutex<T>>` makes no sense without a thread
  to share it with.
- **Crossing between OS threads and async tasks** got its own section, chapter 7 §7.8,
  including what a real-time thread must avoid. The outline didn't name it, but a program
  that combines tokio networking with work on dedicated threads will hit it early, and it's
  where the Go intuition (just send on a channel) is most wrong.

**Not yet written, and belonging here:** `unsafe` and FFI (calling C libraries, which
real-time and audio APIs often are), macros, `Pin` in depth, iterators and closures as a
performance topic, `serde`, structured logging with `tracing`, and profiling
(`tokio-console`, `perf`, flame graphs). Each would get the next free chapter number.

## If you're short on time

- **30 minutes before reading Rust code:** chapter 1 §1.1–1.3, then chapter 2 §2.1–2.4.
- **Starting a tokio project this week:** chapters 6 and 7, with chapter 4 §4.6 open for
  the `Send` errors.
- **Fighting the borrow checker:** chapter 2 §2.8, then §2.6 on lifetimes.
- **Mixing async with dedicated threads (audio, devices, blocking libraries):** chapter 6
  §6.5, then chapter 7 §7.8.

## The one-paragraph summary of everything

Rust moves Go's run-time work to compile time or into libraries. Every value has one
owner and is dropped when the owner's scope ends, so there's no GC; assignment moves
rather than half-copies; and borrows follow "one writer or many readers", which rules out
stale pointers after a reallocation and, across threads, data races. Lifetimes are the
compiler's names for how long borrows are valid; they describe, never extend. Errors are
`Result` values that can't be used without checking, `?` returns early and converts with
`From`, `thiserror` builds enums for callers that branch and `anyhow` adds context for
callers that don't. A panic kills only its thread or task, which is more available than Go
and quieter. Traits are interfaces implemented explicitly; generics are monomorphised
(static dispatch) and `dyn Trait` uses a vtable like a Go interface; traits with
`async fn` can't be `dyn`. `Send` and `Sync` are marker traits derived from a type's
fields, and they are how the compiler keeps `Rc`, `RefCell` and mutex guards on one
thread. Shared ownership is `Rc` or `Arc`; mutation through sharing is `RefCell`, `Mutex`
or atomics, and `Mutex<T>` owns its data so you can't skip the lock, though deadlock is
still yours to prevent. Cleanup is `Drop`, scoped to blocks and owned values, and
`let _ =` drops immediately. Async functions return lazy futures compiled to state
machines; tokio runs them as tasks, cooperatively, so anything that runs long without an
`.await` (sync I/O, `thread::sleep`, heavy CPU) stalls the worker. Use `spawn_blocking`
for short blocking work and a dedicated thread for long-lived loops. `tokio::spawn` needs
`Send + 'static`, which in practice means nothing non-`Send`, such as a std
`MutexGuard`, may be alive across an `.await`. The std mutex is usually right in async
code. tokio has a channel per pattern (`mpsc`, `oneshot`, `broadcast`, `watch`); channels
close when every sender is dropped, never with a panic. `select!` drops the losing
futures, so every branch in a loop must be cancellation safe, and long-lived futures such
as deadlines are created once, pinned and polled by `&mut`. `CancellationToken` is
`context`; dropping or aborting a future is forced cancellation with no Go equivalent;
`JoinSet` aborts its tasks when dropped. Threads that must never wait hand data over with
`try_send` or a preallocated lock-free ring buffer, never a lock or an allocation. Cargo
picks the newest compatible versions, so commit `Cargo.lock`; features are unified across
the graph and must be additive; put code in `lib.rs` so it can be integration-tested;
paused time makes timeout tests instant; and debug builds panic on overflow where release
builds wrap.

## How to use me

Ask me things like:

- "Here's a compiler error. Which of the three ownership questions is it asking?"
- "Translate this Go function, `context` and all, into tokio."
- "Is this `select!` loop cancellation safe?"
- "Where in this code could a tokio worker be blocked?"
- "Should this be `Arc<Mutex<T>>`, an owning task, or an atomic?"
- "Design the error types for this module. Who will match on what?"
- "Review this Cargo workspace for feature and lockfile problems."
- "Is my mental model right? Here's what I think happens: …"
