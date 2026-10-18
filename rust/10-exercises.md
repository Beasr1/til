# 10. Exercises and answers

Try to answer before reading. Getting one wrong is data, not failure. The answers are
worked rather than stated, because the reasoning is the part that transfers.

---

## File 01 — Foundations

**1. "Rust has no runtime, so it can't do what goroutines do."**

Right that Rust ships no scheduler or garbage collector in every binary. Wrong that it
can't do the job: the runtime is a library you add. tokio provides a work-stealing
scheduler over a pool of OS threads, a reactor that waits on sockets, and cheap tasks,
which is M:N scheduling in the same family as Go's. The real differences are elsewhere:
tasks are scheduled cooperatively (only at `.await`), blocking calls inside them are your
problem, and you choose the runtime and its configuration. "No *built-in* runtime" is the
accurate version.

**2. Adding `Reconnecting { after: Duration }` to `Conn`.**

Every `match` on `Conn` without a wildcard arm fails to compile with E0004, naming the
missing variant. The compiler gives you the list of places that need a decision. In the
Go `iota` version, adding a constant compiles silently: `switch` statements without
`default` simply don't handle it, and those with `default` route it there, which may or
may not be right. You find out at run time, if at all. (A `_ =>` arm in Rust opts back
into the Go behaviour, which is why some teams avoid wildcards on their own enums.)

**3. `serde = "1.0.100"` with 1.0.210 published, no lockfile.**

`"1.0.100"` is a caret requirement: at least 1.0.100, below 2.0.0. Cargo "prefers the
highest version currently available", so it picks 1.0.210. Go's minimal version selection
picks the lowest version that satisfies every requirement in the graph, so it would build
with 1.0.100 (or whatever higher minimum some other module demands). Go's build is
therefore reproducible from `go.mod` alone; Cargo's isn't, which is why `Cargo.lock`
carries the reproducibility and should be committed for applications.

**4. Rust slower than Go in a latency benchmark.**

"Was it a release build?" `cargo build` and `cargo run` default to the unoptimised `dev`
profile, and the gap can be large. After that: is the async code blocking a worker
([chapter 6 §6.5](06-async-and-tokio.md#65-blocking-inside-async-code)), and is the
benchmark measuring the same thing in both (connection reuse, buffer sizes, logging)?

**5. Why `sorted` can hold `&String`.**

`counts.iter()` yields references into the map's own storage, so `sorted` is a vector of
borrowed pointers, not copies. That's allowed as long as `counts` outlives `sorted` and
isn't modified while `sorted` exists: no insert, no removal, no drop. Both are true in
`main`, since `counts` is declared first and never touched again. Insert into `counts`
while `sorted` is alive and the borrow checker refuses, because the insert could
reallocate the table and leave `sorted` pointing at freed memory.

---

## File 02 — Ownership and borrowing

**1. Why `let b = a;` can't share a `String`'s buffer.**

Both variables would own the buffer, so both would free it when they went out of scope:
a double free. Rust's answer is to make one of them stop owning it, which is a move. Go
can copy the header and share the array because the garbage collector, not either
variable, decides when the array is freed. Without a GC, "who frees this?" must have
exactly one answer.

**2. `let first = &v[0]; v.push(4);`**

`push` may need more capacity. If so, it allocates a bigger buffer, copies the elements
across and frees the old one. `first` points into the old buffer, so using it afterwards
reads freed memory. The checker can't know whether this particular push reallocates, so
it forbids any mutation while a shared borrow is alive. The Go equivalent compiles and
runs: `append` reallocates, `first` keeps pointing at the old array (which the GC keeps
alive because `first` refers to it), and a write through `first` lands there. Measured,
`v[0]` printed `0` and `*first` printed `100`: a silent lost write.

**3. `fn name(&self) -> &str` with no lifetimes written.**

Elision: with a `&self` receiver, the output is assumed to borrow from `self`. So the
returned `&str` is valid as long as the struct is, and the caller can't drop or mutate the
struct while holding it. If the method built a local `String` and returned `&local`,
it would fail with E0515 ("cannot return reference to local variable"), because the
local is dropped when the method returns. Fix: return `String`.

**4. "Everything I pass to a task leaks until exit."**

`T: 'static` means the value contains no borrowed references with a shorter lifetime, so
it would be *safe* to keep for ever. It says nothing about how long the value is actually
kept. An owned `String` moved into a task is `'static` and is dropped the moment the task
finishes with it, like any other owned value. The tokio tutorial: "all that means is that
it would not be incorrect to keep that value around forever".

**5. Config fields: `&str` or `String`?**

`String`. A config struct with `&'a str` fields borrows from whatever buffer the config
was parsed from, so that buffer must outlive the config, and every struct or task that
holds the config inherits the lifetime parameter. You couldn't move the config into a
`tokio::spawn` without the source buffer being `'static`. Owned fields cost one copy of a
few strings at startup and remove all of that. Borrowed fields are for hot paths where the
copy is the cost you're avoiding.

**6. Which rule prevents data races?**

Shared XOR mutable. A data race needs two concurrent accesses to the same memory with at
least one write and no synchronisation. If, at any moment, there's either one `&mut` or
any number of `&`, then two accessors where one is writing can't both exist. Across
threads, `Send` and `Sync` extend the same rule: a type that would allow shared mutation
without synchronisation (`Rc`, `RefCell`) isn't allowed to cross. So "no data races" is a
consequence, not an extra rule. Interior mutability types are the controlled exceptions,
each with its own synchronisation.

---

## File 03 — Errors

**1. `(*User, error)` with `nil, nil` for "not found".**

`fn find_user(id: UserId) -> Result<Option<User>, DbError>`. The caller must handle three
cases (`Ok(Some(u))`, `Ok(None)`, `Err(e)`) to get at a `User`, and can't confuse "not
found" with "failed". The Go caller can check `err`, forget the `nil` user, and crash on
the dereference; the type doesn't say that `nil, nil` is possible.

**2. `?` on `read_to_string` in a function returning `Result<Config, ConfigError>`.**

`?` converts the `io::Error` with `From`, so it needs `impl From<std::io::Error> for
ConfigError`, which doesn't exist. Two fixes: add a variant with `#[from]` in a
`thiserror` enum (`Io(#[from] std::io::Error)`), which generates the `From`; or convert at
the call site with `.map_err(ConfigError::Read)?` when you want to attach which file it
was. A third, if no caller needs to branch on it, is to return `anyhow::Result<Config>`.

**3. Frame decoder with two different callers.**

`thiserror`. One caller distinguishes "bad frame, skip it" from "connection broken,
give up", and the other wants to close on anything. Both need to *inspect* the error,
which means a typed enum (`Malformed`, `UnknownType`, `Io(…)`), so that one caller can
`match` and the other can convert to `anyhow` with `?`. With `anyhow` alone, the first
caller would be stuck downcasting or matching on message strings.

**4. A `panic!` in one connection's tokio task.**

The runtime catches it at the task boundary. That task ends, its locals are dropped
(closing that connection), the panic message is printed to stderr, and every other task
carries on. If someone awaits the `JoinHandle`, they get a `JoinError` with `is_panic()`
true. Is it better? It's more available, since one bad request doesn't kill every other
connection. But it's quieter: if nobody awaits the handle, the failure is only a line on
stderr. Decide deliberately: supervise handles (log or count `JoinError`s), or set
`panic = 'abort'` and let your process supervisor restart the whole thing as Go would.

**5. A thread panics mid-update holding `Mutex<Vec<Order>>`.**

The mutex is poisoned, so the next `lock()` returns `Err(PoisonError)`. The usual
`.lock().unwrap()` turns that into a panic in the next thread too, so a half-updated
`Vec` is never silently read as if it were consistent. If you know how to repair it,
`PoisonError::into_inner` gives you the guard anyway. In Go, a panic recovered somewhere
up the stack leaves the data half-written and the next `Lock()` succeeds normally (or
the mutex stays locked for ever if the unlock wasn't deferred). Nothing flags it.

---

## File 04 — Traits and generics

**1. Adding `to_frame()` to `std::time::Duration`.**

You can't add an inherent method to a type from another crate. You *can* define your own
trait (`trait ToFrame { fn to_frame(&self) -> Frame; }`) and implement it for `Duration`,
because the trait is local: the orphan rule only forbids implementing a foreign trait for
a foreign type. Callers `use` your trait to get the method. The rule exists so two crates
can't both provide an implementation and leave the compiler unable to choose.

**2. `&dyn Codec` versus `C: Codec` in a hot loop.**

With the generic, the compiler emits a copy of the loop for each concrete codec, and
`encode` is a direct call it can inline, which often lets it optimise across the call.
With `&dyn Codec`, every call goes through the vtable and can't be inlined. Ten million
calls of a tiny method is exactly where that shows up. The generic costs compile time and
binary size per codec type; `dyn` costs a little per call. If the loop is hot and the
codec is known at compile time, use the generic. If the codec is chosen at run time from
config, `dyn` is natural, and you can still move the hot loop inside a generic function
called once per batch.

**3. `trait Store` with an `async fn`, chosen at startup.**

Picking an implementation at run time means `Box<dyn Store>`, and a trait with an `async
fn` isn't dyn compatible: `E0038 … because method get is async`. Options: use the
`async-trait` crate, which rewrites the method to return
`Pin<Box<dyn Future + Send>>` and makes the trait dyn compatible at the cost of an
allocation per call; write that boxed return type by hand; or avoid `dyn` with an enum
(`enum AnyStore { Redis(RedisStore), Mem(MemStore) }`) that implements `Store` by
matching, which is static dispatch and often the simplest when the set is closed.

**4. A struct with `Rc<RefCell<HashMap<…>>>`.**

Neither `Send` nor `Sync`: `Rc` is neither, and auto traits require every field to have
them. Replace it with `Arc<Mutex<HashMap<u64, String>>>`, which is `Send + Sync` because
`HashMap<u64, String>` is `Send`. The cost: atomic reference-count updates on every clone
and drop, and a lock on every access, uncontended in the common case but no longer free.

**5. Why `Mutex<T>: Sync` needs only `T: Send`, but `Arc<T>: Send` needs `T: Sync`.**

A `Mutex` hands out access to its `T` to one thread at a time, so the `T` effectively
moves between threads (one thread uses it, then another) but is never accessed from two
at once. That's exactly what `Send` permits. An `Arc` hands out `&T` to every thread that
holds a clone, at the same time, with no lock. For that to be safe, `&T` must be shareable
across threads, which is the definition of `T: Sync`. And `T: Send` is also needed because
whichever thread drops the last `Arc` drops the `T`.

**6. "Rust prevents race conditions, so no need to think about lock ordering."**

Right about data races: safe Rust can't have unsynchronised concurrent access to memory.
Wrong about race conditions in general and about deadlock. Two threads can still
interleave correctly synchronised operations in an order that breaks your logic (check
then act, with the lock released in between), and two threads taking two mutexes in
opposite orders deadlock exactly as in Go. The Nomicon says "Rust does not prevent general
race conditions" and that it's considered safe for Rust to deadlock. Lock ordering is as
necessary as ever ([cs-fundamentals 04 §4.10](../interview/cs-fundamentals/04-synchronisation-and-deadlock.md#410-four-ways-to-handle-deadlock)).

---

## File 05 — Shared state and threads

**1. `orders.push(o)` on an `Arc<Vec<Order>>`.**

`Arc` gives only `&Vec<Order>`, and `push` needs `&mut`. Shared XOR mutable. Restructure
one of two ways: `Arc<Mutex<Vec<Order>>>` and `orders.lock().unwrap().push(o)`; or give the
`Vec` to one owner (a thread or task) and send new orders to it over a channel. If the
three reader threads only need a snapshot, a third option is to build a new `Vec` and swap
the `Arc` (copy-on-write, via `RwLock<Arc<Vec<…>>>` or the `arc-swap` crate).

**2. Parent and children holding `Rc`s to each other.**

The parent's count never reaches zero, because each child still holds a strong reference
to it, and each child's count never reaches zero because the parent holds them. Nothing
is freed: a leak, which is memory-safe in Rust but still a leak. Make the child-to-parent
links `Weak<Parent>`. A `Weak` doesn't keep the parent alive; `upgrade()` returns
`Option<Rc<Parent>>`, `None` once the parent has gone.

**3. `mu.Lock(); defer mu.Unlock()` around a long function.**

The direct translation is `let guard = m.lock().unwrap();` at the top, which holds the
lock until the function returns, the same as `defer`. To cover only three lines, put them
in a block:

```rust
let total = {
    let mut state = m.lock().unwrap();
    state.count += 1;
    state.total
};   // unlocked here
slow_io(total);
```

or call `drop(state)` explicitly. Go can only scope this by moving the three lines into a
separate function so its `defer` fires early.

**4. `rx.iter()` hangs after every worker finishes.**

A `Sender` is still alive. The receiver's iterator ends only when every sender has been
dropped, and the usual culprit is the original `tx` created in `main` and cloned for each
worker but never dropped. Add `drop(tx)` after spawning the workers. It's Go's
"forgot to `close` the channel, so `range` never ends", with "close" spelled "drop the last
sender".

**5. `RefCell` versus `Mutex` when the rule is broken.**

`RefCell` tracks borrows with a plain counter and panics ("RefCell already borrowed") on a
second `borrow_mut`. `Mutex` blocks the second locker until the first releases. `RefCell`
can't be shared between threads because its counter isn't atomic: two threads could both
read "not borrowed" and both take a mutable borrow. That's why `RefCell` isn't `Sync`,
which the compiler enforces.

**6. `let _ = span.enter();` gives near-zero timings.**

`_` doesn't bind, so the guard returned by `enter()` is dropped at the end of that
statement. The span is entered and exited immediately, before the work you meant to time.
Write `let _guard = span.enter();`, which binds and lives to the end of the scope.

---

## File 06 — Async and tokio

**1. `let f = fetch_user(id);` then an early return.**

No request went out. Calling an `async fn` builds the future and runs none of its body;
returning drops it unpolled. The compiler warns if you never use a future, but here it's
used on a path you didn't take. With Go's `go fetchUser(id)` the goroutine starts
immediately, and an early return in the caller doesn't stop it, so the request would go
out and its result might be lost. If you want the Go behaviour in Rust, `tokio::spawn`
it.

**2. An 80 ms synchronous JSON parse in a handler, four cores.**

Each parse holds one of the four worker threads for 80 ms with no `.await`, and every
other task queued on that worker waits. At low load, other workers absorb it. As load
rises, all four workers spend most of their time parsing, and unrelated requests (health
checks, small reads, heartbeats) see latency in multiples of 80 ms, even though the CPU
isn't necessarily saturated. Change: `let v = tokio::task::spawn_blocking(move ||
serde_json::from_slice(&body)).await??;`, ideally with a `Semaphore` bounding how many
parses run at once, since the blocking pool's default limit is 512 threads.

**3. Guard across `.await`: why `tokio::spawn` rejects it but a direct await accepts it.**

`tokio::spawn` requires `F: Send`, and a future that holds a `MutexGuard` across an
`.await` isn't `Send`. A plain `async fn` you await from another future has no such bound,
so it compiles. Whether that outer future must be `Send` depends on how *it* is run; under
`#[tokio::main]` the main future is run with `block_on` and needn't be. It isn't safe in
practice, though. tokio's docs say the compiler won't stop it "in situations where the
task is not movable between threads" but that it "virtually never leads to correct
concurrent code in practice as it can easily lead to deadlocks": another task on the same
thread that tries to lock will block the thread while the holder is parked waiting to be
polled on that thread.

**4. A for-ever loop reading a blocking device API.**

`std::thread::spawn`. First, `spawn_blocking` is documented as being for "bounded blocking
work that eventually finishes"; a for-ever loop permanently occupies one of the blocking
pool's threads. Second, `spawn_blocking` tasks can't be aborted, and "runtime shutdown will
wait indefinitely for all started spawn_blocking to finish", so shutdown hangs. A dedicated
thread also lets you name it, set its priority, and give it an explicit stop flag or
channel.

**5. Replacing every `std::sync::Mutex` with `tokio::sync::Mutex`.**

Cost: every lock becomes an async operation with a FIFO wait queue, more expensive than a
plain mutex for the common case of a short, uncontended critical section. It also makes
it *easier* to hold a lock across an `.await`, which is the habit you should be avoiding.
And `tokio::sync::Mutex::lock` isn't cancellation safe in a `select!`. It's needed only
when the guard genuinely must be held across an `.await`, for example serialising use of
one connection. tokio's docs recommend the std mutex for plain data and a dedicated owning
task for I/O resources.

**6. A `for { }` loop that calls nothing, in each language.**

Go: since 1.14, goroutines are asynchronously preemptible. The runtime interrupts the loop
after a time slice and runs other goroutines on that thread; the loop wastes a CPU but
doesn't freeze anyone. tokio: tasks only give up their thread at `.await`. A loop with no
`.await` never yields, so the worker thread running it never runs anything else. On a
`current_thread` runtime the whole program stops making progress; on the multi-threaded
runtime one worker is gone for good and the tasks queued on it are stuck until stolen by
another worker. The tokio docs' phrase is "a task is allowed to run until it yields".

---

## File 07 — Concurrency patterns

**1. "close of closed channel" from three closers.**

tokio has no `close` for senders. A channel closes when its last `Sender` is dropped, so
"who closes it?" is answered by ownership: each holder drops its clone when done, and the
receiver sees `None` after the last one. Double-close can't be written, and a send after
close returns `Err` instead of panicking. The new bug is the opposite one: a `Sender` clone
that's never dropped (stored in a struct, captured by a long-lived task) keeps the
receiver waiting for ever.

**2. `broadcast::channel(16)` with one slow client.**

The channel doesn't wait for the slow receiver. When it's full, the oldest value is
overwritten, and the slow task's next `recv()` returns `Err(RecvError::Lagged(n))`, telling
it `n` messages were skipped; after that it continues from the oldest retained value. The
task must handle `Lagged`: for a chat stream, maybe tell the client it missed messages;
for state updates, resynchronise from the source of truth (or use a `watch` channel, which
only ever holds the latest state). Treating `Lagged` as fatal and closing the connection
is also a legitimate policy for clients that can't keep up.

**3. `select!` over `rx.recv()` and `socket.read_exact(&mut header)`.**

`read_exact` isn't cancellation safe. When a message arrives on `rx` while `read_exact`
has read, say, 3 of 8 header bytes, the `read_exact` future is dropped and those 3 bytes
are gone from the socket. The next iteration starts a fresh `read_exact` mid-header and
misparses everything after. Fix: keep the socket read out of the `select!` (a dedicated
reader task that owns the socket and sends whole frames into a channel), or select on a
cancellation-safe call such as `read_buf` into a buffer that lives outside the loop and
parse complete frames from that buffer.

**4. `else => continue` for a non-blocking loop.**

`else` runs only when every branch is disabled, meaning each one's pattern failed to match
or its `if` precondition was false. While any branch is enabled, `select!` waits, so the
loop is exactly as blocking as without `else`. For a non-blocking check, use `try_recv()`
on the channel. (Go's `default` really is "nothing ready right now", which is why the
mistake is natural.)

**5. Early return with `?` from a function holding a `JoinSet`.**

Returning drops the `JoinSet`, and "when the JoinSet is dropped, all tasks in the JoinSet
are immediately aborted". The other four lookups are cancelled at their current `.await`,
their locals dropped. In Go, plain goroutines keep running after the function returns:
they finish their lookups, try to send results nobody reads, and, if the channel is
unbuffered, block for ever and leak (the failure mode in
[go-services 05](../go-services/05-fanning-out.md)). `errgroup.WithContext` gets closer to
the `JoinSet` behaviour, but only for goroutines that check their context.

**6. Callback thread to tokio task handoff.**

The callback must never block, lock, allocate, do I/O or print, because missing its
deadline is an audible glitch, and any of those can take unbounded time. So: allocate a
fixed-size SPSC ring buffer (such as `rtrb`) at startup; the callback pushes each frame
with a non-blocking push and, if the buffer is full, drops the frame and bumps an atomic
counter. On the other side, a normal thread or task pops frames, puts them on a tokio
channel or straight into the network task, and does all the allocation and I/O. When the
consumer falls behind, frames are dropped at the callback, deliberately and counted,
rather than the callback stalling. If a tokio `mpsc` is used directly, use `try_send`,
never `blocking_send`, and be aware its sends may allocate; for a strict real-time thread
the preallocated ring buffer is the safer choice.

---

## File 08 — Cargo and project hygiene

**1. Testing the parser from `tests/` in a binary-only crate.**

Integration tests are separate crates that can only `use` a library's public API, and a
binary-only package has no library. Create `src/lib.rs`, move the parser (and most other
code) into modules declared there, make the parser `pub`, and have `main.rs` call into the
library. Alternatively, test the parser with unit tests in a `#[cfg(test)] mod tests`
inside its own file, which can see private items without any restructuring.

**2. `proto` builds in the workspace, fails alone.**

Feature unification. In the workspace build, `server` depends on tokio with `net`
enabled, Cargo builds one tokio with the union of all requested features, and `proto`
gets `net` for free. Built alone, `proto` gets only the features it declares itself (here
`rt`), so `tokio::net::TcpStream` doesn't exist. Fix: declare every feature `proto`
actually uses in its own `Cargo.toml`, and add `cargo build -p <pkg>` for each package to
CI.

**3. A `no-tls` feature.**

Features are unified across the graph, and enabling one must only add. If any crate in a
user's dependency tree enables `no-tls`, TLS disappears for every other crate that relied
on it, with no error until run time. Invert it: a `tls` feature (probably in `default`)
that adds TLS support. Users who don't want it set `default-features = false`.

**4. Deleting `Cargo.lock` and rebuilding.**

Cargo re-resolves every dependency to the newest version matching each requirement, so
any transitive dependency with a newer semver-compatible release changes, possibly many.
The code is "compatible" by semver, which is a promise authors try to keep, not a
guarantee. In Go, deleting `go.sum` doesn't change versions: minimal version selection
derives them from the `go.mod` files, and the build complains about missing checksums
until `go mod tidy` rewrites them.

**5. Testing backoff without waiting 31 seconds.**

`#[tokio::test(start_paused = true)]` with tokio's `test-util` feature. Time starts
frozen and auto-advances to the next timer whenever every task is idle, so the 1 + 2 + 4 +
8 + 16 s of sleeps complete almost instantly in wall-clock time while
`tokio::time::Instant` reports 31 s elapsed. Measured, a one-hour `timeout` resolved in
about 50 µs. This only works if the code uses tokio's timers, not `std::thread::sleep`.

**6. Passes in release, panics in debug.**

Integer overflow. In the `dev` profile, arithmetic overflow panics; in `release` it wraps.
The test is probably overflowing somewhere (a counter, a checksum, a timestamp
subtraction) and the release build silently wraps, which may be a real bug the debug
build is correctly reporting. If wrapping is intended, say so with `wrapping_add` or
`std::num::Wrapping`; if not, use `checked_add` and handle `None`. A `debug_assert!` that
only fires in debug builds is the other candidate.

---

## Things to try

Roughly in order of effort.

1. **Reproduce three errors on purpose.** Write the use-after-move (E0382), the
   `push`-while-borrowed (E0502) and the `Rc`-into-a-thread (E0277) from chapters 2 and 4.
   Read each error to the end, including the `note:` and `help:` lines, and apply each
   suggested fix. The compiler's suggestions are the fastest teacher you have.
2. **Port a small Go function that returns `(T, error)`** to `Result`, first with a
   `thiserror` enum, then with `anyhow`. Notice which callers you'd have written
   differently.
3. **Run the blocking demo yourself** ([chapter 6 §6.5](06-async-and-tokio.md#65-blocking-inside-async-code))
   on `current_thread`, then on the multi-threaded runtime with as many blocking tasks as
   you have cores. Find the point where ticks start arriving late.
4. **Write the `time.After` bug in both languages** ([chapter 7 §7.4](07-concurrency-patterns.md#74-cancellation-safety)),
   then fix both. Then write a `select!` with `read_exact` and corrupt a stream on purpose.
5. **Build the owning-task store** from chapter 7 §7.2 and add a `Delete` command. Count
   how many places the compiler sends you to.
6. **Write a TCP echo server in tokio** with a `JoinSet` of connections, a
   `CancellationToken`, and Ctrl-C handling. Then kill it mid-transfer and check that every
   connection task logged its shutdown.
7. **Bridge a plain thread to a task.** A `std::thread` that produces a value every 1 ms
   with `try_send` into a bounded tokio channel; a task that consumes slowly. Count drops.
   Then swap in an `rtrb` ring buffer and a relay task.
8. **Set up a workspace** with a library and two binaries, `[workspace.dependencies]`,
   `[lints]`, and a CI script running `fmt --check`, `clippy -D warnings`, tests, and a
   per-package build. Then reproduce the feature-unification trap from chapter 8 §8.3.
9. **Write a paused-time test** for a reconnect loop with exponential backoff and jitter.
   Assert on the exact sequence of delays.

---

## Questions worth asking me

- "Walk me through what the compiler turns this `async fn` into, state by state."
- "Here's a borrow checker error I can't make sense of. What is it actually asking?"
- "Should this shared state be `Arc<Mutex<…>>` or a task with a channel?"
- "Is this `select!` loop cancellation safe? Go branch by branch."
- "How would I design the error types for this module? Who matches on what?"
- "Where could this code block a tokio worker?"
- "Explain `Pin` properly, with a self-referential example."
- "What would this look like with `unsafe`, and what invariants would I be promising?"
- "How do I profile a tokio program? What's `tokio-console` for?"
- "Is my mental model right? Here's what I think happens: …"
