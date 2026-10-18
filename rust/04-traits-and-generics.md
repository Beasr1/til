# 4. Traits and generics, and `Send`/`Sync`

## The problem

Go gives you two ways to write code that works over many types: interfaces, which are
satisfied implicitly and called through a dynamic lookup, and (since Go 1.18) type
parameters, which are deliberately limited. Rust has one mechanism, the **trait**, used
two ways: as a bound on a generic, compiled to specialised code with no lookup, or as a
`dyn` object, called through a lookup like a Go interface. You choose per use site, and
the choice changes performance, binary size and what compiles. Traits also carry a kind
of meaning Go interfaces can't: `Send` and `Sync` have no methods at all, yet they're how
the compiler knows what may cross a thread boundary. Without that piece, half the errors
in chapters 5–7 make no sense.

## 4.1 Traits are interfaces you implement on purpose

```rust
trait Codec {
    fn encode(&self, buf: &mut Vec<u8>);
    fn name(&self) -> &str { "unnamed" }   // a default method
}

struct Ping { seq: u32 }

impl Codec for Ping {
    fn encode(&self, buf: &mut Vec<u8>) { buf.extend_from_slice(&self.seq.to_be_bytes()); }
}
```

Compared with Go:

| | Go interface | Rust trait |
|---|---|---|
| How a type satisfies it | Implicitly, by having the methods | Explicitly, `impl Trait for Type` |
| Default method bodies | No | Yes |
| Implement for a type you didn't write | Only by wrapping it | Yes, if the trait is yours |
| Implement someone else's interface for someone else's type | Only by wrapping it | No: the **orphan rule** |
| Methods required | At least the method set | Can be zero: a **marker trait** (§4.6) |
| Generic constraint | `[T Stringer]` | `<T: Display>` |
| Associated types and constants | No | Yes (`type Item;` in `Iterator`) |

The orphan rule exists so that two crates can't both implement the same trait for the
same type and leave the compiler unsure which to use. The Book: "we can't implement
external traits on external types… This rule ensures that other people's code can't
break your code and vice versa"
([ch10-02](https://doc.rust-lang.org/book/ch10-02-traits.html)). The workaround is the
same as Go's: wrap the foreign type in a local struct (the "newtype" pattern).

Explicit `impl` costs you a line and buys you something Go can't do: a trait can promise
something that isn't a method signature. §4.6 is the case that matters.

## 4.2 Generics with trait bounds

A generic function names its type parameter and says what the parameter must support:

```rust
use std::fmt::Display;

fn log_all<T: Display>(items: &[T]) {
    for i in items { println!("{i}"); }
}

// the same, with a `where` clause, which reads better when bounds pile up
fn send_all<T, W>(items: &[T], out: &mut W) -> std::io::Result<()>
where
    T: Codec,
    W: std::io::Write,
{
    let mut buf = Vec::new();
    for i in items { buf.clear(); i.encode(&mut buf); out.write_all(&buf)?; }
    Ok(())
}
```

`fn f(x: impl Display)` is shorthand for a generic with one bound. In return position,
`-> impl Iterator<Item = u32>` means "some concrete type I'm not naming", which is how you
return closures and iterator chains whose real types are unwriteable.

## 4.3 Static versus dynamic dispatch

This is where Rust and Go genuinely differ, not just in syntax.

**Generics are monomorphised.** The compiler stamps out a separate copy of the function
for each concrete type it's used with. The Book: "Monomorphization is the process of
turning generic code into specific code by filling in the concrete types that are used
when compiled… we pay no runtime cost for using generics"
([ch10-01](https://doc.rust-lang.org/book/ch10-01-syntax.html)). Each copy is optimised
for its type, method calls are direct and can be inlined. The price is compile time and
binary size.

**`dyn Trait` is a trait object.** A `&dyn Codec` or `Box<dyn Codec>` is a pointer to the
data plus a pointer to a table of that type's methods (a **vtable**), so one compiled
function handles every type. The Book: "When we use trait objects, Rust must use dynamic
dispatch… This lookup incurs a runtime cost that doesn't occur with static dispatch.
Dynamic dispatch also prevents the compiler from choosing to inline a method's code"
([ch18-02](https://doc.rust-lang.org/book/ch18-02-trait-objects.html)). That's exactly a
Go interface value.

```rust
let items: Vec<Box<dyn std::fmt::Display>> = vec![Box::new(1), Box::new("two"), Box::new(3.0)];
```

A `Vec<T>` must hold one type; a `Vec<Box<dyn Display>>` holds anything displayable. Use
`dyn` when you need a mixed collection, a plugin point chosen at run time, or to stop one
generic from multiplying through a large codebase.

Go's generics sit between the two. Go 1.18 doesn't stamp out one copy per type. Its
implementation groups types by "gcshape" and passes a **dictionary** of type information
at run time, so "a single function instantiation [can] run correctly for many distinct
type arguments" ([Go 1.18 generics implementation design](https://github.com/golang/proposal/blob/master/design/generics-implementation-dictionaries-go1.18.md)).
All pointer types share one shape. So a Go generic over pointer types does some lookup at
run time, where a Rust generic doesn't.

| | Rust generic `<T: Trait>` | Rust `dyn Trait` | Go interface | Go generic `[T C]` |
|---|---|---|---|---|
| Code generated | One copy per concrete type | One copy | One copy | One per gcshape, plus a dictionary |
| Method call | Direct, inlinable | Through a vtable | Through an itab | Direct for some shapes, dictionary for others |
| Mixed-type collections | No | Yes | Yes | No |
| Cost | Compile time, binary size | An indirect call per method | Same | Somewhere between |

> ⚠️ **Not every trait can be a `dyn`.** A trait must be **dyn compatible** (the Reference's
> term; it "was formerly known as object safety",
> [Reference: traits](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility)).
> A method with type parameters, one that uses `Self` outside the receiver, or one with
> a hidden return type (`-> impl Trait`, or `async fn`) can't be called through a
> vtable, which needs one fixed entry per method. Such a method makes the trait
> incompatible unless it's opted out with `where Self: Sized`. The one you'll hit: **`async fn` in a trait** compiles since
> Rust 1.75 ([announcement](https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits/)),
> but such a trait still can't be used as `dyn`. rustc 1.96 on `Box<dyn Store>` where
> `Store` has an `async fn`:
>
> ```
> error[E0038]: the trait `Store` is not dyn compatible
>   ...because method `get` is `async`
> ```
>
> If you need `Box<dyn Store>`, the established route is the
> [`async-trait`](https://docs.rs/async-trait/latest/async_trait/) crate, which rewrites
> each method to return a boxed future. Its docs note that "the stabilization of async
> functions in traits in Rust 1.75 did not include support for using traits containing
> async functions as dyn Trait." Check whether that's still true when you read this; it's
> an area under active work.

## 4.4 The standard traits you'll meet daily

Most of these you derive with `#[derive(…)]` rather than write.

| Trait | Means | Go nearest | Derive? |
|---|---|---|---|
| `Debug` | `{:?}` formatting, for programmers | `%+v` | Yes |
| `Display` | `{}` formatting, for users | `String()` | No, write it |
| `Clone` | `.clone()` makes an independent copy | Hand-written copy | Yes |
| `Copy` | Assignment copies instead of moving | Every value type | Yes, if all fields are `Copy` |
| `Default` | `T::default()` | Zero value | Yes |
| `PartialEq`, `Eq`, `Hash` | `==`; usable as a `HashMap` key | Comparable types | Yes |
| `From<T>` / `Into<U>` | Infallible conversion; `?` uses `From` | Constructor funcs | No |
| `Iterator` | `next()`, plus dozens of adaptor methods (`map`, `filter`, …) | `range` over a func (Go 1.23) | No |
| `Drop` | Runs when the value is dropped | `defer` at the call site | No ([chapter 5 §5.8](05-shared-state-and-threads.md#58-drop-versus-defer)) |
| `Deref` | `*x` and auto-deref, for smart pointers | — | No |
| `Send`, `Sync` | Safe to move / share across threads | — | **Automatic** (§4.6) |

## 4.5 Closures and the `Fn` traits

A Go `func` value is one kind of thing. A Rust closure is an anonymous struct holding
whatever it captured ([chapter 2 §2.7](02-ownership-and-borrowing.md#27-closures-borrow-or-move)),
and it implements one or more of three traits depending on what it does with those
captures:

| Trait | Can be called | Because the closure | Example bound |
|---|---|---|---|
| `FnOnce` | Once | Moves something out of its captures | `thread::spawn(f: F) where F: FnOnce() -> T` |
| `FnMut` | Many times, not concurrently | Mutates its captures | `iter.for_each(f)` |
| `Fn` | Many times, even concurrently | Only reads its captures | A callback shared across threads |

Every closure is `FnOnce`; `FnMut` closures are also `FnOnce`; `Fn` closures are all
three. A function asks for the weakest it can live with, which is why `thread::spawn`
takes `FnOnce`: it calls your closure exactly once. If you need to store a callback in a
struct, `Box<dyn Fn(Event) + Send>` is the trait-object form.

## 4.6 `Send` and `Sync`: marker traits

Here's the thing Go interfaces can't express. `Send` and `Sync` have no methods. They're
**marker traits**: a type has them or doesn't, and the compiler uses that to decide what
can cross a thread boundary. The Book
([ch16-04](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html)):

> The Send marker trait indicates that ownership of values of the type implementing Send
> can be transferred between threads.

> The Sync marker trait indicates that it is safe for the type implementing Sync to be
> referenced from multiple threads. In other words, any type T implements Sync if &T (an
> immutable reference to T) implements Send.

You almost never write them. They're **auto traits**: "Any type composed entirely of Send
types is automatically marked as Send as well," and the same for `Sync`. A struct of
`String`s and `u64`s is both. A struct containing one `Rc` is neither, without you writing
a word. Implementing them by hand requires `unsafe`.

The types where it matters, with the bounds as stated in the std docs:

| Type | `Send`? | `Sync`? | Why |
|---|---|---|---|
| `i32`, `String`, `Vec<T>` (of `Send`/`Sync` T) | Yes | Yes | Plain owned data |
| `Rc<T>` | No | No | Its reference count isn't atomic: two threads could update it at once |
| `Arc<T>` | If `T: Send + Sync` | If `T: Send + Sync` | Atomic count, but it hands out `&T` to many threads, so `T` must be shareable |
| `RefCell<T>` | If `T: Send` | **No** | Its runtime borrow tracking isn't thread-safe ([chapter 5 §5.3](05-shared-state-and-threads.md#53-interior-mutability-mutation-through-a-shared-reference)) |
| `Mutex<T>` | If `T: Send` | If `T: Send` | The lock makes sharing safe, so `T` needn't be `Sync` |
| `MutexGuard<'_, T>` | **No** | If `T: Sync` | On POSIX platforms there's "a requirement to release mutex locks on the same thread they were acquired" ([MutexGuard](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html)) |
| Raw pointers `*const T` | No | No | The compiler can't know |

Now the rest of the language falls into place. `std::thread::spawn` is declared with
`F: Send + 'static`, so anything the closure captures must be `Send`. Try to send an `Rc`:

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
    = help: within `{closure@rc.rs:6:19: 6:26}`, the trait `Send` is not implemented for `Rc<i32>`
note: required by a bound in `spawn`
128 |     F: Send + 'static,
```

That's the race detector's job, done by the type checker, on every path, before the
program exists. Go's race detector can only report races on code paths a test or a run
actually executed.

`Send` and `Sync` don't make your program *correct*: you can still deadlock, and you can
still have a logical race where two threads take turns in an order you didn't expect. The
Nomicon is careful about this: "Rust does not prevent general race conditions"
([races](https://doc.rust-lang.org/nomicon/races.html)). What they rule out is the
undefined behaviour of an unsynchronised read and write to the same memory.

> **Teacher's aside.** A Go interface describes what a type can *do*. A Rust trait can
> also describe what a type *is*, and that's how `Send` works: no method set could say
> "this is safe to move to another thread". It only works because traits are implemented
> deliberately and the compiler derives them structurally from a type's fields. When an
> error says some type "cannot be sent between threads safely", read it as: somewhere
> inside this value, possibly three layers deep, is a field whose type didn't promise
> thread-safety. The error's `note:` lines tell you which field.

## Check yourself

1. You want to add a `to_frame()` method to `std::time::Duration` in your crate. Can you?
   What's the idiomatic way, and why does Rust stop you doing the obvious thing?
2. A hot loop calls `codec.encode(&mut buf)` ten million times. When does it matter whether
   `codec` is `&dyn Codec` or a generic `C: Codec`, and which way does each choice trade?
3. You've written `trait Store { async fn get(&self, k: &str) -> Option<Bytes>; }` and want
   to pick a Redis or in-memory implementation from config at startup. What goes wrong,
   and what are your options?
4. A struct has fields `id: u64`, `name: String` and `cache: Rc<RefCell<HashMap<u64,
   String>>>`. Is it `Send`? Is it `Sync`? What one change makes it both, and what does
   that change cost?
5. Explain why `Mutex<T>` is `Sync` when `T` is only `Send`, but `Arc<T>` needs `T: Sync`
   to be `Send`.
6. A colleague says "Rust prevents race conditions, so we don't need to think about lock
   ordering." What's right and wrong in that sentence?

(Answers in [10-exercises.md](10-exercises.md).)
