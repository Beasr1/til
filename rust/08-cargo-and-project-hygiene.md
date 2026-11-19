# 8. Cargo and project hygiene

## The problem

Go's toolchain is opinionated to the point of having few knobs: one build mode, one
formatter, tests next to the code, a module per repository unless you choose otherwise.
Cargo has more knobs, and the defaults are good, but a few of them behave in ways a Go
programmer won't predict. Features combine across the whole dependency graph. A binary
with no library can't be integration-tested. Debug builds are a different program from
release builds. And the lockfile is doing a job that `go.sum` doesn't. This chapter is the
project layout and tooling that keeps a growing Rust codebase easy to build, test and
review.

## 8.1 Layout: put the code in a library, keep `main` thin

Cargo finds targets by convention ([Cargo targets](https://doc.rust-lang.org/cargo/reference/cargo-targets.html)):

```
my-app/
├── Cargo.toml
├── Cargo.lock
├── src/
│   ├── lib.rs          the library crate: almost all the code
│   ├── main.rs         the binary crate: parse args, build runtime, call the library
│   ├── bin/
│   │   └── loadgen.rs  extra binaries, each its own crate
│   ├── protocol.rs     a module, declared in lib.rs with `mod protocol;`
│   └── net/
│       ├── mod.rs      the `net` module (or src/net.rs beside the folder)
│       └── conn.rs
├── tests/
│   └── handshake.rs    integration tests: each file is a separate crate
├── examples/
│   └── echo.rs         `cargo run --example echo`
└── benches/
```

Why split `lib.rs` from `main.rs` even for a program nobody will depend on? Because
integration tests can only reach a library. The Book: "If our project is a binary crate
that only contains a src/main.rs file and doesn't have a src/lib.rs file, we can't create
integration tests in the tests directory and bring functions defined in the src/main.rs
file into scope with a use statement"
([ch11-03](https://doc.rust-lang.org/book/ch11-03-test-organization.html)). The same
split lets `examples/` and extra binaries reuse the code. `main.rs` ends up as a few
dozen lines: read config, set up logging, start the runtime, call `my_app::run(config)`.

Go's nearest habit is `cmd/app/main.go` plus `internal/…` packages, and the motive is the
same.

## 8.2 Workspaces: several packages, one lockfile

When the code grows into several packages (a protocol library, a server, a client, a
load generator), a **workspace** builds them together. From the Cargo book
([workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html)): "All packages
share a common Cargo.lock file which resides in the workspace root" and "a common output
directory".

```toml
# Cargo.toml at the root: a "virtual manifest", no [package]
[workspace]
members = ["proto", "server", "client"]
resolver = "3"

[workspace.package]
edition = "2024"

[workspace.dependencies]
tokio = { version = "1", features = ["rt-multi-thread", "net", "sync", "time", "macros"] }
anyhow = "1"
```

```toml
# server/Cargo.toml
[package]
name = "server"
version = "0.1.0"
edition.workspace = true

[dependencies]
proto = { path = "../proto" }
tokio.workspace = true
anyhow.workspace = true
```

| Go | Cargo |
|---|---|
| One module, many packages | One package, many modules (the usual Rust default) |
| Several modules plus a `go.work` | A workspace with several member packages |
| "It is generally inadvisable to commit go.work files" ([Go modules reference](https://go.dev/ref/mod#workspaces)) | The workspace `Cargo.toml` and `Cargo.lock` are committed |
| Shared versions by convention | `[workspace.dependencies]` plus `dep.workspace = true` |

> ⚠️ **Set `resolver` explicitly in a virtual workspace.** The resolver version normally
> follows the package's edition (`"2"` for edition 2021, `"3"` for 2024), but a virtual
> manifest has no package to take an edition from. The Cargo book says resolver "must be
> set explicitly in virtual workspaces", and that the default otherwise is `"1"`
> ([resolver versions](https://doc.rust-lang.org/cargo/reference/resolver.html#resolver-versions)).
> Resolver 1 unifies features more aggressively (next section), which can surprise you.

## 8.3 Features: additive, and unified

A **feature** is a named compile-time switch in a package, used to make dependencies or
code optional. tokio is the example you meet first: `rt-multi-thread`, `net`, `sync`,
`time`, `macros`, and the catch-all `full`.

The rule that catches people: features are unified across the whole build. The Cargo book
([features](https://doc.rust-lang.org/cargo/reference/features.html#feature-unification)):
"When a dependency is used by multiple packages, Cargo will use the union of all features
enabled on that dependency when building it… A consequence of this is that features
should be additive. That is, enabling a feature should not disable functionality, and it
should usually be safe to enable any combination of features."

So:

- **A feature must only add.** Never write `#[cfg(feature = "fast")]` to *replace*
  behaviour that another crate in the graph might depend on. Someone else enabling it
  changes your build.
- **Your crate may compile only because a sibling enabled a feature for you.** A crate
  that uses `tokio::net` but only declares `features = ["rt"]` builds fine in the
  workspace, because the server crate turned `net` on, and fails when built alone. I set
  that up with the workspace above: `cargo build` at the root succeeded, and
  `cargo build -p proto` failed with
  ``error[E0425]: cannot find type `TcpStream` in module `tokio::net` ``. Build each package
  on its own in CI to catch it.
- **Libraries should ask for the minimum** and leave `full` to applications. Use
  `default-features = false` to opt out of a dependency's defaults.
- **Enable only documented features.** Cargo will accept any feature name in the
  dependency's manifest, including ones meant for its own testing. reqwest 0.13.5's
  manifest has a block of double-underscore features headed "Internal (PRIVATE!)
  features used to aid testing" and "Don't rely on these whatsoever. They may disappear
  at any time" ([Cargo.toml](https://github.com/seanmonstar/reqwest/blob/master/Cargo.toml)).
  `__rustls` there pulls in the rustls crates but not the crypto provider and certificate
  verifier that the public `rustls` feature adds. It compiles, and you have a client
  whose TLS was never finished. With `default-features = false`, pick names from the
  crate's documented feature list ([reqwest's](https://docs.rs/reqwest/latest/reqwest/#optional-features)),
  and re-check them when you bump a major version.

There's no Go equivalent. Go's build tags are per build, not negotiated across
dependencies.

### Build every feature combination you ship

Features are also how teams ship half-finished work: a new field behind
`#[cfg(feature = "new-field")]`. Rust lets you put `#[cfg]` on a struct field, a function
parameter or a match arm, and that's the trap. Every place that constructs, destructures
or passes the gated item needs the same gate, and the build with the feature off is a
different program from the build with it on. If CI only builds one, the other rots
silently until someone needs it.

The fix is to build the combinations. [`cargo-hack`](https://github.com/taiki-e/cargo-hack)
does it: `--each-feature` builds "for each feature which includes default features and
`--no-default-features`", and `--feature-powerset` builds "the feature powerset", with
`--depth` to "Specify a max number of simultaneous feature flags" when the powerset gets
large. A gate that costs a hundred lines across five files is also a sign it should be
short-lived: remove it once the feature has shipped.

## 8.4 Versions and the lockfile

From [chapter 1 §1.5](01-foundations.md#15-crates-modules-and-cargo-versus-go-modules-and-packages):
Cargo picks the newest semver-compatible version of each dependency, and records the
choice in `Cargo.lock`. Caret requirements are the default: "Default requirements specify
a minimum version with the ability to update to SemVer compatible versions"
([specifying dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html)).

| | Go | Cargo |
|---|---|---|
| Version chosen | Minimal version satisfying all requirements (MVS) | Highest compatible version at resolve time |
| Without the lock / sum file | Same versions (MVS is deterministic from `go.mod`) | **Possibly newer versions** |
| Lockfile holds | `go.sum`: checksums | `Cargo.lock`: exact versions and checksums |
| Upgrade within range | `go get -u` | `cargo update` |
| Commit it? | Yes | `cargo new` "defaults to tracking Cargo.lock in version control" ([FAQ](https://doc.rust-lang.org/cargo/faq.html)). Commit it for applications |

For a networked program, commit `Cargo.lock`. Without it, two builds a week apart can
differ in every transitive dependency.

**Editions** are Rust's version line for the language itself, set per package with
`edition = "2024"`. The 2024 edition was stabilised in Rust 1.85
([announcement](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/)). An edition is
opt-in and per crate, and crates of different editions link together. It's not a
toolchain requirement the way Go's `go` line can be ([go-services 08](../go-services/08-when-a-dependency-moves-your-toolchain.md));
the field for that is `rust-version`, which declares the minimum supported Rust.

## 8.5 Tests

| Kind | Where | Can see | Go nearest |
|---|---|---|---|
| Unit tests | `#[cfg(test)] mod tests { … }` at the bottom of the same file | Private items in that module | `foo_test.go` in `package foo` |
| Integration tests | `tests/*.rs`, each file a separate crate | Only the library's `pub` API | `package foo_test` |
| Doc tests | Code blocks in `///` comments | The `pub` API | `Example` functions |
| Async tests | `#[tokio::test]` on an `async fn` | Either | — |

```rust
// src/protocol.rs
pub fn checksum(data: &[u8]) -> u8 { data.iter().fold(0, |a, b| a ^ b) }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn checksum_of_empty_is_zero() {
        assert_eq!(checksum(&[]), 0);
    }
}
```

`#[cfg(test)]` means the module is only compiled for `cargo test`, so test helpers don't
end up in the binary. Tests in a binary run in parallel on threads by default, like
`t.Parallel()` everywhere; anything sharing global state needs care.

For async code, `#[tokio::test]` gives each test its own runtime. "The default test
runtime is single-threaded" ([tokio::test](https://docs.rs/tokio/latest/tokio/attr.test.html)),
which hides some bugs a multi-threaded runtime would show; use
`#[tokio::test(flavor = "multi_thread")]` for tests about concurrency.

The feature Go doesn't have: **paused time.** With tokio's `test-util` feature,
`#[tokio::test(start_paused = true)]` freezes the clock and advances it to the next timer
whenever every task is idle. A test of a one-hour idle timeout:

```rust
#[tokio::test(start_paused = true)]
async fn idle_timeout_fires_after_an_hour() {
    let r = tokio::time::timeout(Duration::from_secs(3600), std::future::pending::<()>()).await;
    assert!(r.is_err());
}
```

ran in 49.5 µs of wall-clock time on my machine. Retries, backoff, heartbeats and
timeouts become fast, deterministic tests instead of slow, flaky ones. Only tokio's own
timers are paused; `std::time::Instant::now()` and `std::thread::sleep` aren't.

**Poll, don't sleep, when the clock isn't yours.** End-to-end tests against a real
server, database or child process can't pause time. The tempting version is a fixed
`sleep(Duration::from_secs(5))` before each assertion: too short on a loaded CI runner,
pure waste everywhere else, and when it fails it says nothing. Poll the observable state
against a deadline instead, and on timeout report every state you saw:

```rust
async fn await_state(job: &Client, id: JobId, want: State, within: Duration) -> Result<(), String> {
    let deadline = tokio::time::Instant::now() + within;
    let mut seen = Vec::new();
    loop {
        let s = job.state(id).await.map_err(|e| e.to_string())?;
        if seen.last() != Some(&s) { seen.push(s.clone()); }
        if s == want { return Ok(()); }
        if tokio::time::Instant::now() >= deadline {
            return Err(format!("wanted {want:?} within {within:?}; saw {seen:?}"));
        }
        tokio::time::sleep(Duration::from_millis(50)).await;
    }
}
```

A failure now reads "wanted Done; saw [Queued, Running, Failed]", which is usually the
whole diagnosis.

**A coverage drop with a green suite is a measurement problem until proven otherwise.**
Source-based coverage ([rustc book](https://doc.rust-lang.org/rustc/instrument-coverage.html))
works by instrumenting the test binaries: "the counter values are written to a `profraw`
file at program termination", and the raw files are combined afterwards with
`llvm-profdata merge`. Anything that changes how each binary is invoked, how it
terminates, or where it writes can change what gets recorded, with no warning, because a
missing profile just looks like code that didn't run. I've seen a whole-package coverage figure fall by several points
with every test passing, and come back on the same tree when the test thread count was
set through the environment instead of as an argument after `--` to the coverage
wrapper. I didn't establish the mechanism, so take the specific fix as a lead, not a
rule. Note too that the rustc book marks `RUST_TEST_THREADS` as deprecated in favour of
`--test-threads` ([tests](https://doc.rust-lang.org/rustc/tests/index.html)), so the
environment variable isn't a long-term answer either. What holds regardless is the
method: when the number moves and the tests didn't, measure the *same tree* two ways
before you go looking for missing tests.

## 8.6 Formatting and lints

| Tool | Does | Go nearest |
|---|---|---|
| `cargo fmt` | Formats; `--check` fails CI on unformatted code | `gofmt -l` |
| `cargo clippy` | "over 800 lints", per its [README](https://github.com/rust-lang/rust-clippy), in groups | `go vet` + `staticcheck` |
| `[lints]` in `Cargo.toml` | Per-package lint levels, checked in | `.golangci.yml` |

Clippy's groups and their default levels ([Clippy docs](https://doc.rust-lang.org/clippy/)):

| Group | Catches | Default |
|---|---|---|
| `clippy::correctness` | "code that is outright wrong or useless" | deny |
| `clippy::suspicious` | "code that is most likely wrong or useless" | warn |
| `clippy::style`, `complexity`, `perf` | Unidiomatic, convoluted or slow code | warn |
| `clippy::pedantic` | Stricter lints "with occasional false positives" | allow |
| `clippy::restriction` | Bans on language features; the docs say it "should, emphatically, not be enabled as a whole" | allow |

A reasonable starting point for an application, checked in so everyone gets the same:

```toml
[lints.rust]
unsafe_code = "forbid"

[lints.clippy]
unwrap_used = "warn"      # from `restriction`: pick individual lints, not the group
```

And in CI:

```
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
cargo build -p <each package>      # catches the feature-unification trap
cargo hack check --each-feature    # catches a feature combination that rotted (§8.3)
```

`-D warnings` turns warnings into errors in CI only, so a new compiler's new warning
doesn't break local builds.

In a workspace, set the lints once: "The `workspace.lints` table is where you define lint
configuration to be inherited by members of a workspace", and each member opts in with
`[lints] workspace = true` (respected from Cargo 1.74,
[workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html)). A crate that
wraps a C library can keep `unsafe_code = "deny"` at the workspace level and put an
explicit `#[allow(unsafe_code)]` on each FFI module, so every new `unsafe` is a visible
decision ([chapter 9](09-calling-c.md)).

## 8.7 Profiles

| | `dev` (default) | `release` |
|---|---|---|
| Optimisation | None | Full |
| Debug info | Yes | No (by default) |
| Integer overflow | Panics | Wraps ([chapter 1 §1.6](01-foundations.md#16-the-commands-youll-run)) |
| `debug_assert!` | Checked | Compiled out |

Profiles are configured in `Cargo.toml` ([profiles](https://doc.rust-lang.org/cargo/reference/profiles.html)).
Two settings worth knowing early:

```toml
[profile.release]
debug = "line-tables-only"   # usable backtraces and profiles without full debug info
# panic = "abort"            # make any panic fatal (chapter 3 §3.5)

[profile.dev.package."*"]
opt-level = 2                # optimise dependencies even in debug builds
```

The second is a common trick when debug builds are too slow to test with (audio or
crypto code, for instance): your crate stays quick to compile and debug, the
dependencies run at near release speed.

> **Teacher's aside.** Go trained you to think of "the build" as one thing. In Rust,
> `cargo build` and `cargo build --release` produce programs that differ in speed by a
> large factor and in arithmetic behaviour, and the feature set of a dependency can
> depend on which sibling packages are in the build. Most "it works on my machine"
> problems in Rust projects come from one of those, not from the code. When something
> behaves differently in CI, check the profile, the feature set and the lockfile before
> the logic.

## 8.8 CI that builds and runs everything

A test suite that no pipeline runs isn't a test suite. It compiles on the day it's
written and then drifts: fixtures go stale, an API changes underneath it, and months
later someone finds dozens of failures that each need archaeology. Four habits keep the
whole project honest.

**Compile every target.** Without target flags, `cargo check` "will check all binary and
library targets of the selected packages", and `cargo clippy` behaves the same way: I put
a type error in a file under `tests/`, and plain `cargo clippy` (0.1.96) finished clean
while `cargo clippy --all-targets` failed with E0308. So `tests/`, `benches/`,
`examples/` and every `#[cfg(test)]` module are never compiled by a plain lint run. `--all-targets` "is equivalent to specifying `--lib --bins --tests
--benches --examples`" ([cargo check](https://doc.rust-lang.org/cargo/commands/cargo-check.html)).
Then actually run the tests in a CI stage, on a host that has the native libraries they
need, not only inside the release container.

**Never gate a build on "the output exists".** A script that runs `cargo build` only if
`target/…/module.wasm` is missing will keep serving a stale artifact after the code it
depends on changes. When that artifact is bound to an interface (a plugin built against
a host's imports, a library against a header), the stale copy fails at load time with an
error that points nowhere near the cause. Always invoke the build and let Cargo decide:
an up-to-date `cargo build` is a quick no-op. External build systems that need to know
when to re-run Cargo should read the "dep info" files Cargo writes next to each artifact,
which list "all of the file dependencies required to rebuild the artifact"
([build cache](https://doc.rust-lang.org/cargo/reference/build-cache.html)).

**In a build script, ask Cargo about the target; `#[cfg]` describes the host.** A
`build.rs` is compiled for, and runs on, the machine doing the build. So
`#[cfg(target_arch = "aarch64")]` inside it tests the *build machine*, which is wrong the
moment you cross-compile. The Cargo reference says it directly: "`CARGO_CFG_*` variables
should be used instead of the `cfg!` macro or `#[cfg]` attribute in build scripts, those
check the *host* platform, not the *target*"
([environment variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html#environment-variables-cargo-sets-for-build-scripts)).

```rust
// build.rs: link an architecture-specific static library only where it exists.
fn main() {
    let arch = std::env::var("CARGO_CFG_TARGET_ARCH").unwrap();   // the target, e.g. "aarch64"
    let os = std::env::var("CARGO_CFG_TARGET_OS").unwrap();       // the target, e.g. "linux"
    println!("cargo:rustc-link-lib=static=codec");
    if arch == "aarch64" {
        println!("cargo:rustc-link-lib=static=codec_neon");   // only built for ARM64
    }
    if os == "linux" {
        println!("cargo:rustc-link-lib=dylib=stdc++");
    }
}
```

**Make generated output deterministic.** A build script that reads a table and generates
code, or a program that prints a summary, must not iterate a `HashMap` to do it. The std
docs: the default hasher "is randomly seeded", "each `HashMap` instance uses a different
seed", and iteration visits entries "in arbitrary order"
([HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html)). The same
five keys, iterated in three runs of one binary:

```
["alpha", "delta", "bravo", "echo", "charlie"]
["echo", "bravo", "alpha", "delta", "charlie"]
["charlie", "alpha", "bravo", "echo", "delta"]
```

Generated code that changes on every build defeats caching and review, and breaks any
test that compares output. Use `BTreeMap` (sorted by key) or an insertion-ordered map
such as the `indexmap` crate whenever order is observable. Go programmers have met this
before: "The iteration order over maps is not specified and is not guaranteed to be the
same from one iteration to the next" ([Go spec](https://go.dev/ref/spec#For_range)).
Sort the keys before you print.

## Check yourself

1. Your project is a single `src/main.rs` of 2,000 lines, and you want to test the
   protocol parser from `tests/`. What do you have to change, and why?
2. The `proto` crate in your workspace compiles in CI as part of the workspace but fails
   with "cannot find type `TcpStream` in module `tokio::net`" when a colleague builds it
   alone. Explain.
3. A library you maintain adds a feature `no-tls` that removes TLS support when enabled.
   Why is that design wrong in Cargo's model, and what should the feature be instead?
4. You delete `Cargo.lock` to fix a merge conflict and rebuild. What might have changed
   in the binary? Would deleting `go.sum` in a Go project have the same effect?
5. A reconnect-with-backoff function waits 1 s, 2 s, 4 s, 8 s, 16 s. How do you test that
   it gives up after five attempts without the test taking 31 seconds?
6. A test passes with `cargo test --release` and panics with plain `cargo test`. Name the
   most likely cause.
7. After a CI change, the coverage report for one package drops by several points. Every
   test still passes and nobody deleted a test. What do you suspect first, and what's the
   experiment that settles it? While you're there, CI's lint step is `cargo clippy --
   -D warnings` and there's no test stage: what's been rotting?
8. A `build.rs` links an ARM-only static library inside `#[cfg(target_arch = "aarch64")]`
   and generates a lookup table by iterating a `HashMap`. Cross-compiling for ARM64 on an
   x86-64 machine fails to link, and the generated file shows a diff on every build.
   Explain both, and fix both.

(Answers in [11-exercises.md](11-exercises.md).)
