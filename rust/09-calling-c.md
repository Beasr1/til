# 9. Calling C: FFI, `unsafe` and dynamic loading

## The problem

Sooner or later a Rust program has to call code that isn't Rust: a device driver's SDK, a
codec, an operating-system API, a vendor library that ships as a `.dll` or `.so` and a
header file. Everything the earlier chapters promised stops at that boundary. The
compiler can't see the C side, so it can't check that the function you named exists, that
it takes the arguments you said, that it uses the calling convention you assumed, that it
succeeded, or that the memory it hands back is yours to free. You write those facts down
as declarations, and the compiler believes you. This chapter is how to write those
declarations, how to wrap them so the *rest* of the program gets its guarantees back, and
the handful of things no wrapper can check for you.

Go has the same boundary in cgo, and the Go side appears wherever it differs. Most of the
rules come from C, not from either language, so they're the same in both.

Assumes chapter 2 (ownership, borrowing, §2.6 lifetimes) and chapter 3 (errors).

## 9.1 What `unsafe` means, and what it doesn't

Safe Rust's promise is that no sequence of safe calls can cause undefined behaviour. FFI
can't keep that promise on its own, because a C function can do anything. So Rust has a
keyword for "I have checked what the compiler can't", and the Book lists exactly what it
unlocks, the "unsafe superpowers"
([ch20-01](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html)):

1. Dereference a raw pointer.
2. Call an unsafe function or method.
3. Access or modify a mutable static variable.
4. Implement an unsafe trait.
5. Access fields of `union`s.

Calling a foreign function is the second one. Everything else stays on. The Book again:
"`unsafe` doesn't turn off the borrow checker or disable any of Rust's other safety
checks". An `unsafe` block is a small region where *you* carry some of the proof, and the
idiom is to write that proof down in a comment starting `SAFETY:`.

> **Teacher's aside.** People read `unsafe` as "dangerous code here". The better reading
> is "a promise is made here". The danger is usually somewhere else: a wrong declaration
> in an `extern` block is written once and makes every call to it wrong, while each call
> site looks innocent. When you review FFI code, read the declarations harder than the
> call sites.

The 2024 edition (stabilised in Rust 1.85) tightened three rules, all in the direction of
making promises visible. A crate moving from 2021 to 2024 meets them all at once:

| Rule | 2021 | 2024 | Why ([Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2024/index.html)) |
|---|---|---|---|
| [Extern blocks](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html) | `extern "C" { … }` | `unsafe extern "C" { … }` required | "it is the responsibility of the author of the `extern` block to ensure that the signatures are correct" |
| [Link-affecting attributes](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html) | `#[no_mangle]` | `#[unsafe(no_mangle)]`; same for `export_name` and `link_section` | Symbol names are one global namespace across every linked library; a clash breaks code that contains only safe Rust |
| [`unsafe_op_in_unsafe_fn`](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html) | Allowed | Warns | An `unsafe fn` body is no longer one big unsafe block; each unsafe operation inside needs its own `unsafe { }` |

The third one changes how you read an `unsafe fn`. Its `unsafe` now means only "callers
have obligations" (listed in a `# Safety` doc section); the operations inside get their
own blocks and their own `SAFETY:` comments.

## 9.2 Declaring a foreign function

A foreign function is declared, not defined, in an **extern block**. The Reference
([external blocks](https://doc.rust-lang.org/reference/items/external-blocks.html)): "A
function declared in an extern block is implicitly `unsafe` unless the `safe` function
qualifier is present."

```rust
use std::ffi::{c_char, c_int};

unsafe extern "C" {
    // strlen requires a valid, NUL-terminated pointer: stays unsafe to call.
    fn strlen(s: *const c_char) -> usize;
    // sqrt (from libm) is defined for every f64: you may promise that, and call it
    // without an unsafe block.
    safe fn sqrt(x: f64) -> f64;
}
```

That compiled with rustc 1.96 (edition 2024), and `unsafe { strlen(c"hello".as_ptr()) }`
returned 5. `safe` is itself a promise: you're asserting that *no* argument value can
cause undefined behaviour. That's a stronger claim than it looks. C's `abs` seems
harmless, but its behaviour "is undefined if the result cannot be represented by the
return type", which is the case for the most negative `int`
([cppreference](https://en.cppreference.com/w/c/numeric/math/abs), C17 §7.22.6.1). Most
C functions that take pointers can't honestly be marked `safe` either.

The types come from `std::ffi`: `c_int`, `c_char`, `c_long` and friends match the C type
on the current target. That matters because C's `long` is 4 bytes under Microsoft's
compilers even on 64-bit Windows ([data type ranges](https://learn.microsoft.com/en-us/cpp/cpp/data-type-ranges)),
and 8 bytes on 64-bit macOS (`sizeof(long)` printed 8 with clang on arm64) and Linux.
Never write `i32` or `i64` because you know what the header's type is on the machine in
front of you.

### The ABI string

The `"C"` after `extern` is the **ABI string**: which calling convention to use, meaning
where the arguments go (registers or stack, in what order), who cleans up the stack, and
where the return value comes back. Both sides must agree, and nothing checks that they do.
The Reference's list, trimmed to the ones you'll meet:

| ABI string | Means | Use when the header says |
|---|---|---|
| `"C"` | "the default ABI chosen by the dominant C compiler for the target". Also the default if you write no string | Nothing (plain C function) |
| `"system"` | Same as `"C"`, "except on Windows x86_32 where it is equivalent to `"stdcall"`" | `WINAPI`, `CALLBACK`, `__stdcall`, or a vendor macro that expands to one of them |
| `"stdcall"`, `"cdecl"`, `"fastcall"` | The 32-bit x86 conventions by name; x86_32 only | Only when you target 32-bit x86 alone |
| `"win64"`, `"sysv64"` | The two 64-bit x86 conventions by name | Rarely; to call one from the other |
| `"C-unwind"`, `"system-unwind"` | As above, but a panic or C++ exception may unwind across the call | The foreign code calls back into Rust that may panic, or throws |

The `"system"` row is where a latent bug hides. Microsoft's docs describe the two 32-bit
conventions: under `__cdecl` the "Calling function pops the arguments from the stack"
([`__cdecl`](https://learn.microsoft.com/en-us/cpp/cpp/cdecl)), under `__stdcall` the
"Called function pops its own arguments from the stack"
([`__stdcall`](https://learn.microsoft.com/en-us/cpp/cpp/stdcall)). Declare a `__stdcall`
function as `extern "C"` and, on 32-bit x86, caller and callee disagree about who removes
the arguments, so the stack pointer is wrong after every call. That's undefined behaviour,
and what you see depends on the surrounding code.

But on 64-bit Windows, the same docs say: "On ARM and x64 processors, `__stdcall` is
accepted and ignored by the compiler", and the x64 ABI "uses a four-register, fast-call
calling convention by default"
([x64 calling convention](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention)).
The `__cdecl`/`__stdcall` distinction doesn't exist there, and the Reference defines
`"system"` as `"C"` everywhere except 32-bit Windows, so the two produce identical code. The wrong
declaration works perfectly on every 64-bit build and breaks the first time someone builds
for 32-bit Windows, which is exactly when nobody remembers it was ever a question.

> ⚠️ **Copy the calling convention from the header, not from what works.** If the header
> says `__stdcall` (or `WINAPI`, or a macro that expands to it), write `extern "system"`,
> even if every target you build today ignores the difference. The Reference gives the same
> advice for the Windows API: "it's best to use `extern "system"` when attempting to link
> Windows API functions that don't use an explicitly defined ABI."

## 9.3 Loading a library at run time

There are two ways to reach a C library:

| | Linked at build time | Loaded at run time |
|---|---|---|
| How | `unsafe extern` block plus `#[link(name = "foo")]` or a build script | `dlopen`/`LoadLibrary`, in Rust usually through the [`libloading`](https://docs.rs/libloading/) crate |
| Missing library | Build fails, or (for a shared library) the program fails to start | `Library::new` returns `Err`; you choose what to do |
| Missing function | Link error at build time | `Library::get` returns `Err` at run time |
| Typical use | System libraries, libraries you build from source | Vendor SDKs shipped as a binary, optional plugins, a library that might not be installed |
| Go nearest | cgo with `#cgo LDFLAGS: -lfoo` | `syscall.LoadDLL` on Windows; cgo can't call a C function pointer directly ("Calling C function pointers is currently not supported", [cmd/cgo](https://pkg.go.dev/cmd/cgo)) |

Run-time loading is common for vendor hardware SDKs, and it adds two problems that build
time linking doesn't have.

**Loading runs code.** The `libloading` docs on `Library::new`
([docs](https://docs.rs/libloading/latest/libloading/struct.Library.html#method.new)):
"When a library is loaded, initialisation routines contained within it are executed. For
the purposes of safety, the execution of these routines is conceptually the same calling
an unknown foreign function". So `Library::new` is `unsafe`, and the path is part of the
safety argument. On Windows, a bare file name is looked up through the DLL search order,
and Microsoft warns that "If an attacker gains control of one of the directories that's
searched, then it can place a malicious copy of the DLL in that folder"
([DLL search order](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order)).
Load from a full path that comes from trusted configuration, never from a request or an
environment variable an attacker can set.

I built a toy C library with a constructor that prints, and loaded it with libloading
0.8.9 on macOS. The library's code ran before the Rust program called anything:

```
[lib] initialiser ran on load
[rust] loaded
```

**Symbols are strings, and their types are claims.** You look a function up by name and
tell `libloading` its type:

```rust
let open: Symbol<unsafe extern "system" fn(c_int) -> c_int> =
    unsafe { lib.get(b"scanner_open\0")? };
```

The name is a byte string, so a typo or a copy-paste from the wrong line compiles. If the
name exists, the lookup succeeds whatever type you wrote; `Library::get`'s only safety
requirement is "Users of this API must specify the correct type of the function or
variable loaded". The type, the calling convention and the name are three facts that
come from the header and that no tool compares against it.

The defence is to make each fact appear once, beside the others, in a table you review
against the header line by line:

```rust
// Header: int SCANAPI scanner_open(int port);   with SCANAPI defined as __stdcall
type StatusFn = unsafe extern "system" fn() -> c_int;
type OpenFn = unsafe extern "system" fn(port: c_int) -> c_int;

pub struct Api {
    init: StatusFn,
    open: OpenFn,
    begin_capture: StatusFn,
    end_capture: StatusFn,
    close: StatusFn,
    _lib: Library, // keeps the code mapped for as long as the pointers above exist
}

impl Api {
    /// # Safety
    /// `path` must name a trusted build of the scanner library (loading runs its
    /// initialisers), and the signatures above must match its header.
    pub unsafe fn load(path: &Path) -> Result<Api, libloading::Error> {
        // SAFETY: the caller vouches for the path; signatures checked against the header.
        unsafe {
            let lib = Library::new(path)?;
            Ok(Api {
                init: *lib.get::<StatusFn>(b"scanner_init\0")?,
                open: *lib.get::<OpenFn>(b"scanner_open\0")?,
                begin_capture: *lib.get::<StatusFn>(b"scanner_begin_capture\0")?,
                end_capture: *lib.get::<StatusFn>(b"scanner_end_capture\0")?,
                close: *lib.get::<StatusFn>(b"scanner_close\0")?,
                _lib: lib,
            })
        }
    }
}
```

Three things this buys. Every symbol is resolved at load, so a missing one fails at
startup rather than halfway through a session. Every name sits on one line next to the
field it fills, so a reviewer can hold the header beside it. And the function pointers
live in the same struct as the `Library`, which matters because `*lib.get(…)` copies out
a plain function pointer: a `fn` type carries no lifetime, so once copied, nothing ties it
to the library. Keeping them together means they're dropped together.

## 9.4 Typestate: making the call order a type error

A C device API usually has an order the header can only describe in a comment: call
`init` once, then `open`, then `begin_capture`; `end_capture` before `close`; nothing
after `close`. Call them out of order and you get an error code if you're lucky and a
crash if you're not.

Rust can turn that comment into types. **Typestate** programming, in the Embedded Rust
Book's words, "describes the encoding of information about the current state of an object
into the type of that object"
([typestate programming](https://docs.rust-embedded.org/book/static-guarantees/typestate-programming.html)).
Each state is a type; each method exists only on the state where it's legal; each
transition consumes the old value and returns the new one.

```rust
pub struct Idle;
pub struct Open;
pub struct Capturing;

pub struct Scanner<'api, S> { api: &'api Api, _state: PhantomData<S> }

impl<'api> Scanner<'api, Idle> {
    pub fn init(api: &'api Api) -> Result<Self, ScanError> {
        check("scanner_init", unsafe { (api.init)() })?;
        Ok(Scanner { api, _state: PhantomData })
    }
    pub fn open(self, port: c_int) -> Result<Scanner<'api, Open>, ScanError> {
        check("scanner_open", unsafe { (self.api.open)(port) })?;
        Ok(self.into_state())
    }
}

impl<'api> Scanner<'api, Open> {
    pub fn begin_capture(self) -> Result<Scanner<'api, Capturing>, (Self, ScanError)> {
        match check("scanner_begin_capture", unsafe { (self.api.begin_capture)() }) {
            Ok(()) => Ok(self.into_state()),
            Err(e) => Err((self, e)), // give the open device back: retry or close it
        }
    }
    pub fn close(self) -> Result<Scanner<'api, Idle>, ScanError> {
        check("scanner_close", unsafe { (self.api.close)() })?;
        Ok(self.into_state())
    }
}

impl<'api> Scanner<'api, Capturing> {
    pub fn end_capture(self) -> Result<Scanner<'api, Open>, ScanError> {
        check("scanner_end_capture", unsafe { (self.api.end_capture)() })?;
        Ok(self.into_state())
    }
}

fn check(call: &'static str, code: c_int) -> Result<(), ScanError> {
    if code == 0 { Ok(()) } else { Err(ScanError { call, code }) }
}
```

(`into_state` is a private helper that rebuilds the struct with a new `S`.) Now
`scanner.begin_capture()` on an `Idle` scanner is a compile error, there's no method to
call; and after `s.close()`, `s` has been moved, so using it again is E0382 from
[chapter 2 §2.2](02-ownership-and-borrowing.md#22-moves-assignment-hands-ownership-over).
`PhantomData<S>` costs nothing at run time; the state exists only in the type.

The design decision that makes this mean anything is in `check`. **A transition that
ignores the status code is a lie.** If `open` called the C function, threw away its
return value and returned `Scanner<Open>`, the type would say the device is open when it
isn't, and every later call would be made from a state the device isn't in. The
typestate would have made a wrong program look more trustworthy. So every transition
checks the foreign call's result and refuses to produce the next state on failure.

Against the toy library, an `open` on a port that doesn't exist produced no `Open`
scanner at all:

```
[lib] init
[rust] ScanError { call: "scanner_open", code: -3 }
```

What to return on failure is a second decision. `open` failing from `Idle` can just drop
the old state. `begin_capture` failing from `Open` shouldn't, because the device is still
open and someone has to close it, so it hands `self` back inside the error.

> ⚠️ **You can't write `Drop` for one state.** It's tempting to make an `Open` scanner
> close itself when dropped. rustc 1.96 on `impl Drop for Scanner<Open>`:
> ``error[E0366]: `Drop` impls cannot be specialized``. A `Drop` impl must cover every
> `S`. If you want automatic teardown per state, use a separate type per state (a guard
> for the open device, another for the capture), and make the capture guard *borrow* the
> device guard so it can't outlive it. Getting that borrow right is a lifetime question,
> worked through in [chapter 2 §2.6](02-ownership-and-borrowing.md#a-child-handle-must-borrow-its-parent).

### What the types can't check

Typestate checks which calls are reachable from which state. It checks nothing about the
call itself:

| Fact | Checked by typestate? | Your defence |
|---|---|---|
| `begin_capture` only after `open` | ✅ Yes | — |
| The call succeeded | Only if each transition checks the status | `check()` in every transition |
| The symbol name is the right function | ❌ No | One table of names, reviewed against the header |
| The function type and argument types | ❌ No | Same table; `std::ffi` types, never guessed widths |
| The calling convention | ❌ No | Copy it from the header (§9.2) |
| The library is the build you think | ❌ No | Load from a trusted, full path (§9.3) |

The third row is easy to get wrong, because functions in one C API often share a shape.
`end_capture` and `close` are both `fn() -> c_int` here. Paste the wrong name into the
table:

```rust
end_capture: *lib.get::<StatusFn>(b"scanner_close\0")?,   // should be scanner_end_capture
```

It compiles, it loads, and the typestate happily calls "end capture" before "close". The
toy library logged what really happened:

```
[lib] begin capture
[lib] close (open=1 capturing=1)
[lib] close (open=0 capturing=1)
```

Close was called twice and end-capture never. Here the second close returned an error and
`check` caught it, which is the status check earning its keep as a second line of defence.
A library that returns success on a redundant close would have shown nothing at all.

## 9.5 Sizes from the other side are untrusted

A device reports a frame's width and height as two `c_int`s and you allocate a buffer.
The careful version checks for overflow:

```rust
let w = usize::try_from(width).map_err(|_| Error::BadDims)?;
let h = usize::try_from(height).map_err(|_| Error::BadDims)?;
let len = w.checked_mul(h).ok_or(Error::BadDims)?;
let frame = vec![0u8; len];
```

On a 64-bit target that doesn't overflow even for `c_int::MAX × c_int::MAX`, because the
product is about 4.6 × 10¹⁸ and `usize` holds up to about 1.8 × 10¹⁹. So `checked_mul`
returns `Some`. It's also below `isize::MAX` bytes (about 9.2 × 10¹⁸), the line above which `Vec`
panics instead ("Panics if the new capacity exceeds `isize::MAX` *bytes*",
[`Vec::with_capacity`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.with_capacity)).
So the request goes to the allocator: 4.6 exabytes. With rustc 1.96 on
macOS:

```
memory allocation of 4611686014132420609 bytes failed
```

and the process exited with status 134, SIGABRT. Wrapping the allocation in
`std::panic::catch_unwind` didn't help: it isn't a panic. The default handler for a
failed allocation, `std::alloc::handle_alloc_error`, is documented to "print a message to
standard error and abort the process" when the program links `std`, though "Future
versions of Rust may panic by default instead"
([docs](https://doc.rust-lang.org/std/alloc/fn.handle_alloc_error.html)).

Overflow checking answers "is this number representable?". The question you needed was
"is this number reasonable?". Bound untrusted sizes against a maximum before allocating:

```rust
const MAX_FRAME_BYTES: usize = 64 * 1024 * 1024;   // whatever the largest real sensor needs
if len > MAX_FRAME_BYTES { return Err(Error::BadDims); }
```

If you genuinely can't pick a maximum, `Vec::try_reserve` returns an error instead of
aborting: "If the capacity overflows, or the allocator reports a failure, then an error is
returned" ([docs](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.try_reserve)).
In the same program it returned `memory allocation failed because the memory allocator
returned an error`. But an allocation the system can satisfy and shouldn't, a few
gigabytes on a machine with enough swap, succeeds and hurts you more slowly, so the bound
is the real defence. This applies to any length that crosses a trust boundary: a C
library's out-parameter, a length prefix on a network frame, a header field in a file.

The test that catches this is the boring one: call the function with the largest values
the C type allows. It's worth writing for every constructor that takes sizes from
outside, because "overflow-safe" sounds like "safe" and isn't.

## 9.6 Whoever allocates must free

C has no destructors and no owner, so every pointer that crosses the boundary comes with
an unwritten rule about who frees it, and with what. The rule that always holds: **memory
is freed by the same allocator that allocated it, which in practice means by a function
exported from the library that allocated it.**

Rust's standard library says so for strings it hands out. `CString::into_raw` gives C a
`*mut c_char`, and "The pointer which this function returns must be returned to Rust and
reconstituted using `CString::from_raw` to be properly deallocated. Specifically, one
should *not* use the standard C `free()` function to deallocate this string"
([docs](https://doc.rust-lang.org/std/ffi/struct.CString.html#method.into_raw)). And in
the other direction, `from_raw` on memory "allocated by foreign code" is "likely to lead to
undefined behavior or allocator corruption".

Why isn't `free()` fine, if Rust uses the system allocator? Because "system allocator"
isn't one thing:

| Fact | Source |
|---|---|
| `std::alloc::System` "is based on `malloc` on Unix platforms and `HeapAlloc` on Windows", and "it is not valid to mix use of the backing system allocator with `System`" | [`System`](https://doc.rust-lang.org/std/alloc/struct.System.html) |
| The default global allocator in a Rust program "is unspecified"; a `cdylib` or `staticlib` uses `System` by default; and any crate can swap in another with `#[global_allocator]` | Same page |
| On Windows, "Each copy of the CRT library has its own heap manager", and freeing memory allocated by one copy in another "can cause heap corruption". A DLL built against a static CRT has its own copy | [Microsoft: CRT objects across DLL boundaries](https://learn.microsoft.com/en-us/cpp/c-runtime-library/potential-errors-passing-crt-objects-across-dll-boundaries) |

So a consumer that frees your string with its own `free` (or C#'s `Marshal.FreeHGlobal`,
or Swift's `free`) may get away with it on Linux and macOS, where the allocators often
happen to coincide, and corrupt the heap on Windows. Tests on the first two platforms
passing proves nothing about the third. The fix is part of the API, not the consumer:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn mylib_describe(code: c_int) -> *mut c_char {
    CString::new(format!("status {code}")).unwrap().into_raw()
}

/// Frees a string returned by any `mylib_*` function. Passing NULL is a no-op.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn mylib_string_free(s: *mut c_char) {
    if s.is_null() { return; }
    // SAFETY: the contract says `s` came from `CString::into_raw` in this library
    // and is freed exactly once.
    drop(unsafe { CString::from_raw(s) });
}
```

Every function that returns an owned pointer gets a matching free function, documented
next to it, and every struct with owned fields gets one that frees all of them. When you
*consume* a C library, the same rule runs the other way: free what it gives you with the
free function it exports, and borrow its strings with `CStr` rather than taking ownership.

Go's cgo has the same rule, written into its docs: the string from `C.CString` "is
allocated in the C heap using malloc. It is the caller's responsibility to arrange for it
to be freed, such as by calling C.free" ([cmd/cgo](https://pkg.go.dev/cmd/cgo)).

```go
cs := C.CString(name)           // malloc'd in the C heap; Go's GC won't touch it
defer C.free(unsafe.Pointer(cs))
C.scanner_set_name(cs)
```

<details>
<summary>C++: the same rule as a smart pointer with the library's deleter</summary>

```cpp
#include <memory>

extern "C" {
    char* mylib_describe(int code);
    void  mylib_string_free(char* s);
}

struct MylibStringFree { void operator()(char* s) const { mylib_string_free(s); } };
using MylibString = std::unique_ptr<char, MylibStringFree>;

MylibString describe(int code) { return MylibString(mylib_describe(code)); }
// Never std::unique_ptr<char> (which calls delete) or a bare free():
// neither is the allocator that produced the pointer.
```

</details>

## 9.7 Memory lent for asynchronous completion

The hardest version of the ownership rule is when you hand memory *to* a library that
will finish with it later. USB transfers are the standard example. libusb's asynchronous
API is "a 5 step process": allocate a transfer, fill it, submit it, handle completion in
a callback, deallocate
([libusb async I/O](https://libusb.sourceforge.io/api-1.0/group__libusb__asyncio.html)).
Between submission and the callback, the transfer, its buffer and your `user_data`
pointer belong to libusb, and the docs are blunt about freeing early: "It is not legal to
free an active transfer (one which has been submitted and has not yet completed)".

Cancellation doesn't give it back either. "libusb_cancel_transfer() is
asynchronous/non-blocking in itself"; it "returns immediately, but this does not indicate
cancellation is complete", and the callback comes "at some later time with a transfer
status of LIBUSB_TRANSFER_CANCELLED". Until it does, "Freeing the transfer after it has
been cancelled but before cancellation has completed will result in undefined behaviour."

Now write a `Drop` for the stream that owns those transfers. The obvious version cancels
everything, waits a while for the callbacks, then frees every transfer. It works on every
clean shutdown and is a use-after-free whenever the wait times out, because a transfer
whose callback hasn't run still belongs to libusb, and libusb will write to it later.
Unplugging the device mid-stream is the classic way to get there, along with any shutdown
where the thread that handles libusb events has already stopped.

The sound options, in order:

1. **Wait for every callback.** If you can guarantee they'll arrive (events are still
   being handled), this is the only fully clean answer.
2. **If you can't, leak.** Give up ownership of the unfinished transfers on purpose, with
   `std::mem::forget` or `Box::leak`, and log a warning with the count. A bounded leak on
   a rare path is a bug report; a use-after-free is memory corruption that shows up
   somewhere else, later.

Leaking is safe Rust: `mem::forget` isn't `unsafe`, because "Rust's safety guarantees do
not include a guarantee that destructors will always run"
([docs](https://doc.rust-lang.org/std/mem/fn.forget.html), and chapter 5 §5.8). It feels
wrong because Go's GC never makes you choose. But the choice here isn't between leaking
and freeing; it's between leaking and handing freed memory to a library that still thinks
it owns it.

Go draws the same line in cgo's pointer rules: "C code may not keep a copy of a Go
pointer after the call returns", unless the memory is pinned with `runtime.Pinner`
([cmd/cgo](https://pkg.go.dev/cmd/cgo)). Go's answer to "C still has it" is to stop the GC
moving or freeing it. Rust's is that you don't free it.

| Situation | Who owns the memory | When you may free it |
|---|---|---|
| Rust returns a `CString::into_raw` pointer to C | C holds it, Rust's allocator owns it | When C calls your exported free function |
| C returns a pointer from its own allocator | The C library | Through the free function the library exports |
| You submit a buffer for asynchronous completion | The library, until the callback | After the callback has run, never before |
| Cancellation requested, callback not yet run | Still the library | Still after the callback. If it will never come, leak |

## Check yourself

1. Your crate moves from the 2021 edition to 2024 and three kinds of FFI line stop
   compiling or start warning. Name them, and say what each new `unsafe` is a promise
   about.
2. A vendor header declares `int __stdcall scanner_open(int port);`. Your Rust declares
   `unsafe extern "C" fn(c_int) -> c_int` and the program has worked on 64-bit Windows for
   a year. Why does it work, what happens on a 32-bit build, and what's the one-word fix?
3. A colleague says the typestate wrapper makes the scanner API "impossible to misuse".
   Give two misuses it can't prevent, and say which one the status check would catch and
   which it might not. Then suppose `open` returned `Scanner<Open>` without looking at
   the C function's return value: why would that be worse than having no typestate at all?
4. A constructor computes `width.checked_mul(height)` from device-reported `c_int`s and
   allocates. A test with both set to `c_int::MAX` kills the test binary rather than
   failing the test. Explain each step, and why `catch_unwind` doesn't save it.
5. Your Rust library returns strings with `CString::into_raw`. The C# client frees them
   with `Marshal.FreeHGlobal`, the Swift client with `free`, and CI on Linux and macOS is
   green. What's wrong, why didn't CI notice, and what do you change in *your* library?
6. A USB stream's `Drop` cancels every in-flight transfer, waits 500 ms for the
   callbacks, then frees all of them. Describe the run where this corrupts memory, and
   what `Drop` should do instead.

(Answers in [11-exercises.md](11-exercises.md).)
