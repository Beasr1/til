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

There's no Go equivalent. Go's build tags are per build, not negotiated across
dependencies.

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
```

`-D warnings` turns warnings into errors in CI only, so a new compiler's new warning
doesn't break local builds.

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

(Answers in [10-exercises.md](10-exercises.md).)
