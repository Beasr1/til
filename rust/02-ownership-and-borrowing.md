# 2. Ownership, moves, borrowing and lifetimes

## The problem

Without a garbage collector, something has to decide when each piece of memory is
freed, and it has to get it exactly right: free too early and something reads freed
memory, free twice and the allocator corrupts, never free and you leak. C leaves that to
you. Go hands it to a collector at run time. Rust decides it at compile time, from rules
about who owns each value and who may look at it while it's owned. Those same rules turn
out to rule out a set of Go bugs that have nothing to do with freeing memory: a slice
pointer that silently goes stale after `append`, a "copy" of a struct whose map is still
shared, a counter incremented from two goroutines. This chapter is the rules and what
each one buys.

## 2.1 Ownership: every value has exactly one owner

The Rust Book states the whole system in three lines
([ch04-01](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)):

> Each value in Rust has an owner.
> There can only be one owner at a time.
> When the owner goes out of scope, the value will be dropped.

"Dropped" means its destructor runs and its memory is released. For a `String` that's
the heap buffer; for a file handle it's the `close`; for a mutex guard it's the unlock.
The compiler knows statically where every owner's scope ends, so it inserts those calls
itself. There's no collector deciding later.

```rust
fn main() {
    let name = String::from("ada");   // `name` owns a heap buffer
    {
        let greeting = format!("hi {name}");   // `greeting` owns another
        println!("{greeting}");
    }                                  // `greeting` dropped here: buffer freed
    println!("{name}");
}                                      // `name` dropped here
```

## 2.2 Moves: assignment hands ownership over

If only one owner may exist, what does `let b = a;` do for a `String`? It can't copy the
pointer and leave both owning the buffer, because then both would free it. So it
**moves**: `b` becomes the owner and `a` is no longer usable. The same happens when you
pass a value to a function or return it.

```rust
fn consume(v: Vec<i32>) -> usize { v.len() }

fn main() {
    let v = vec![1, 2, 3];
    let n = consume(v);           // ownership moves into `consume`
    println!("{n} {}", v.len());  // v was moved
}
```

rustc 1.96:

```
error[E0382]: borrow of moved value: `v`
3 |     let v = vec![1, 2, 3];
  |         - move occurs because `v` has type `Vec<i32>`, which does not implement the `Copy` trait
4 |     let n = consume(v);
  |                     - value moved here
5 |     println!("{n} {}", v.len());
  |                        ^ value borrowed here after move
```

The compiler then suggests two fixes, and they're the two you'll use most: change the
parameter to borrow (`&Vec<i32>`, or better `&[i32]`), or call `.clone()` if you really
want an independent copy.

Small, plain values such as integers, `bool`, `char` and tuples of them implement the
**`Copy`** trait, and assignment copies them instead of moving. Anything that owns a
resource (a heap buffer, a file, a lock) can't be `Copy`.

### Contrast with Go: assignment that half-copies

In Go, assigning a struct copies its fields, but a field that's a slice or map copies
only a header or reference. The data behind it is shared, so writes through the "copy"
reach the original. The go-services course has a whole chapter on that bug,
[Copies that share memory](../go-services/06-copies-that-share-memory.md).

Rust has no half-copy. `let b = a;` moves everything, so there's still one owner, and
`a.clone()` produces a fully independent value: `Clone` for `Vec` or `HashMap` clones
every element. If you want sharing, you have to ask for it by name, with `Rc` or `Arc`
([chapter 5](05-shared-state-and-threads.md)).

| Operation | Go | Rust |
|---|---|---|
| `b := a` / `let b = a;` on a struct containing a map | Copies fields; map is shared | Moves; `a` unusable |
| Independent copy | Write it by hand, or `maps.Clone` (shallow) | `.clone()`; depth follows each field's `Clone` |
| Shared ownership | Any pointer, implicitly | `Rc<T>` / `Arc<T>`, explicitly |

## 2.3 Borrowing: look without taking

Moving everything into every function would be unworkable, so you can **borrow** a value
with a reference instead. There are two kinds:

- `&T`, a **shared reference**: read-only access. Any number may exist at once.
- `&mut T`, a **mutable reference**: read-write access. Only one may exist, and no shared
  references alongside it.

The Book's rule ([ch04-02](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)):

> At any given time, you can have either one mutable reference or any number of
> immutable references.
> References must always be valid.

People call the first line **aliasing XOR mutation**: data can be shared or mutated, never
both at once. Here's what it catches. Take a reference to the first element of a vector,
push onto the vector, then use the reference:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4);
println!("{first}");
```

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(4);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{first}");
  |                ----- immutable borrow later used here
```

Why does Rust care? Because `push` may reallocate the buffer, and then `first` points at
freed memory. Go has the same shape of bug, but with a GC it doesn't crash; it goes
stale. This Go program, run with Go 1.24:

```go
v := make([]int, 3, 3) // len 3, cap 3: the next append must reallocate
first := &v[0]
v = append(v, 4)
*first = 100
fmt.Println(v[0], *first)
```

```
0 100
```

The write through `first` went to the old backing array, which the GC keeps alive
because `first` still points at it. `v` never sees it. No crash, no warning, just a
write that went nowhere. Rust refuses to compile the equivalent.

The same rule rejects mutating a collection while iterating over it
(`for x in &v { v.push(…) }` fails with the same E0502), which in Go is legal and
produces behaviour you have to look up.

## 2.4 What each rule prevents

| Rule | Rust bug it prevents | Go version of the bug | How Go copes |
|---|---|---|---|
| One owner; drop at scope end | Double free, leak by forgetting | None for memory (the GC handles it) | GC |
| Use after move is an error | Use after free; using a resource after handing it off | Using a `*sql.Tx` after `Commit`; touching a buffer after handing it to another goroutine | Run-time errors (`sql.ErrTxDone`), or discipline |
| Shared XOR mutable | Iterator invalidation, dangling pointers into a reallocated buffer | Stale pointer after `append`; writes through a "copy" landing in shared maps | Nothing; the program runs |
| Shared XOR mutable, across threads | **Data races** | Unsynchronised access from two goroutines | `go test -race`, on the paths you happen to exercise |
| References must be valid | Dangling references | None (escape analysis moves the value to the heap) | GC |

The fourth row is the one that matters most for concurrent code, and it isn't a separate
rule. A data race needs two accesses to the same memory at the same time, at least one a
write, with no synchronisation (the Book lists exactly those three conditions). "Many
readers or one writer" makes that impossible to express in safe code. The Rustonomicon
says it flatly: "Safe Rust guarantees an absence of data races"
([Nomicon: races](https://doc.rust-lang.org/nomicon/races.html)). The mechanics across
threads are the `Send` and `Sync` traits in [chapter 4 §4.6](04-traits-and-generics.md#46-send-and-sync-marker-traits).

To see what the Go side costs, this program increments a shared counter from 1,000
goroutines with no lock:

```go
counter := 0
var wg sync.WaitGroup
for range 1000 {
	wg.Add(1)
	go func() { defer wg.Done(); counter++ }()
}
wg.Wait()
fmt.Println(counter)
```

Three runs on my machine printed `972`, `957` and `931`. With `-race` it reports
`WARNING: DATA RACE`. That's a measurement of one laptop, not a rate you should quote;
the point is that the plain build compiles and runs. The Rust equivalent, closures
mutating a captured counter from several threads, doesn't compile (§2.7).

The last row is worth a second look, because it's the one place Rust is *stricter* than
Go without preventing a Go bug. Returning a pointer to a local is fine in Go; escape
analysis puts the value on the heap and the GC keeps it alive. Rust has no GC to do that:

```rust
fn make_greeting(name: &str) -> &str {
    let s = format!("hello, {name}");
    &s
}
```

```
error[E0515]: cannot return reference to local variable `s`
```

The fix is to return the owned `String`. Ownership moves out to the caller, which is
cheap: it moves the pointer, length and capacity, not the bytes.

## 2.5 Owned and borrowed twins: `String` and `&str`, `Vec` and slices

Go has one `string` type and one `[]T`. Rust splits each into an owner and a view:

| Owns its data | Borrowed view | Go nearest | Use for |
|---|---|---|---|
| `String` | `&str` | `string` | Function parameters take `&str`; struct fields usually hold `String` |
| `Vec<T>` | `&[T]` / `&mut [T]` | `[]T` | Parameters take `&[T]`; you keep a `Vec` |
| `Box<T>` | `&T` | `*T` | Heap allocation with one owner ([chapter 5](05-shared-state-and-threads.md)) |
| `PathBuf` | `&Path` | `string` | File paths |

The habit to build: **parameters borrow, fields own.** A function that only reads takes
`&str` or `&[T]`, and then accepts both a `String` (via `&s`) and a string literal. A
struct that keeps data stores the owned type, so it doesn't need a lifetime (next
section). A `&String` parameter works but is needlessly narrow; Clippy will tell you so.

## 2.6 Lifetimes: names for how long a borrow is valid

"References must always be valid" needs the compiler to know how long each reference may
live. Inside one function it works that out itself. Across a function boundary it
sometimes needs a hint:

```rust
fn longest(a: &str, b: &str) -> &str { if a.len() > b.len() { a } else { b } }
```

```
error[E0106]: missing lifetime specifier
  = help: this function's return type contains a borrowed value, but the signature does
    not say whether it is borrowed from `a` or `b`
```

The question is fair. The caller needs to know how long the returned reference stays
valid, and that depends on which input it came from. You answer with a **lifetime
parameter**:

```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str { … }
```

Read `'a` as "some region of the program". This says: the result is valid for as long as
*both* inputs are. Most signatures don't need this, because of **lifetime elision**: with
one reference input, or a `&self` method, the compiler assumes the output borrows from
that input.

Structs that hold references need a lifetime too, and that ties the struct to whatever it
borrows from:

```rust
struct Request<'a> { path: &'a str }

fn parse(line: &str) -> Request<'_> {
    Request { path: line.split(' ').nth(1).unwrap_or("/") }
}

fn main() {
    let r;
    {
        let line = String::from("GET /health HTTP/1.1");
        r = parse(&line);
    }
    println!("{}", r.path);
}
```

```
error[E0597]: `line` does not live long enough
8 |     }
  |     - `line` dropped here while still borrowed
9 |     println!("{}", r.path);
  |                    ------ borrow later used here
```

`Request` points into `line`'s buffer, and `line` was freed at the brace. That's a
use-after-free, caught at compile time. The usual fix in application code is to make the
field `String` and copy the path out. Borrowing structs are for hot paths such as
zero-copy parsers, where the copy is the thing you're trying to avoid.

> **Teacher's aside.** Lifetimes don't change how long anything lives. Nothing you write
> in a lifetime annotation extends or shortens a value's life; drops still happen where
> the owners go out of scope. Annotations are *claims* about how references relate, which
> the compiler checks against the code. When you hit a lifetime error, adding `'a`
> everywhere almost never helps. Ask instead: what owns this data, and does that owner
> outlive everyone who's borrowing it? If not, either move the owner up a scope, or stop
> borrowing and own a copy.

### `'static`

One lifetime has a name built in. `&'static str` is a reference valid for the whole
program, like a string literal. But you'll mostly meet `'static` as a *bound*, `T:
'static`, in `thread::spawn` and `tokio::spawn`, and that means something narrower. The
tokio tutorial ([spawning](https://tokio.rs/tokio/tutorial/spawning)) puts it well:

> When we say that a value is 'static, all that means is that it would not be incorrect
> to keep that value around forever.

An owned `String` is `'static` in that sense: it contains no borrowed references, so
nothing can be freed out from under it. A `&str` pointing into a local is not.
`T: 'static` means "owns everything it touches", not "lives forever".

## 2.7 Closures: borrow or move

A Go closure captures variables by reference, always, and the GC keeps them alive as long
as the closure exists. A Rust closure captures each variable in the least demanding way
that works: by shared borrow if it only reads, by mutable borrow if it writes, by move if
it needs to own. Put a `move` before the closure to force moves.

That's why this doesn't compile:

```rust
let mut counter = 0;
let h = std::thread::spawn(|| { counter += 1; });
h.join().unwrap();
```

```
error[E0373]: closure may outlive the current function, but it borrows `counter`,
which is owned by the current function
note: function requires argument type to outlive `'static`
help: to force the closure to take ownership of `counter` (and any other referenced
variables), use the `move` keyword
```

The new thread might outlive `main`'s stack frame, so it can't borrow `main`'s local.
`move` makes the closure own a copy of `counter` (integers are `Copy`), which compiles but
won't do what a Go programmer means, because each thread would increment its own copy.
To share one counter across threads you need `Arc<Mutex<i32>>` or an atomic, which is
[chapter 5](05-shared-state-and-threads.md). If you only need the threads to finish before
the local goes away, `std::thread::scope` lets them borrow (chapter 5 §5.6).

Go's own closure-capture bug, every goroutine in a loop sharing one loop variable, was
fixed in Go 1.22 by giving loops "per-iteration scope instead of per-loop scope"
([Go blog](https://go.dev/blog/loopvar-preview)). Rust never had it: a `for` loop's
variable is a fresh binding each iteration, and a `move` closure takes its own.

## 2.8 When the borrow checker says no

Most errors fall into a handful of shapes, with standard fixes:

| Symptom | Usual cause | Fix, roughly in order of preference |
|---|---|---|
| "borrow of moved value" | You passed ownership away and kept using it | Borrow instead (`&x`); restructure; `.clone()` if cheap or rare |
| "cannot borrow as mutable because it is also borrowed as immutable" | A reference is still alive across a mutation | Shorten the borrow: finish using the reference first, or copy the value out (`let first = v[0];`) |
| "does not live long enough" | A borrower outlives the owner | Own the data (`String` not `&str`), or move the owner to an outer scope |
| A struct that needs to point at itself, or a graph with back-pointers | Shared ownership the rules can't express as one owner | Indices into a `Vec`; `Rc`/`Arc` with `Weak` for back-pointers (chapter 5) |
| Closure "may outlive the current function" | Spawned thread or task borrowing a local | `move`, plus `Arc` for shared data; or scoped threads |

> ⚠️ **Cloning to get past the checker is fine while you learn, and worth revisiting
> later.** A `.clone()` on a `String` copies the bytes; on an `Arc` it bumps a counter.
> Neither is a bug. But when you find yourself cloning the same value in three places to
> satisfy the compiler, the design usually wants a different owner, and the borrow
> checker is pointing at it.

## Check yourself

1. Why can't `let b = a;` copy a `String`'s pointer and leave both variables usable, the
   way Go copies a slice header?
2. Explain, in terms of memory, why `let first = &v[0]; v.push(4);` is rejected, and what
   the equivalent Go program actually does at run time.
3. A function signature is `fn name(&self) -> &str`. No lifetimes are written. What does
   the compiler assume about the returned reference, and what would stop compiling if the
   method returned a reference into a local `String` instead?
4. `tokio::spawn` requires `T: 'static`. A colleague says "so everything I pass to a task
   leaks until the program exits". Correct them.
5. You're writing a config struct loaded once at startup. Should its fields be `&str` or
   `String`? Argue from what each choice forces on the rest of the program.
6. Which row of the table in §2.4 is the reason safe Rust can't have a data race, and why
   doesn't it need a separate "no data races" rule?

(Answers in [10-exercises.md](10-exercises.md).)
