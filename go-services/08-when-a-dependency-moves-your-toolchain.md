# 8. When a dependency moves your toolchain

## The problem

A security advisory names a dependency. You bump it, run the tests, and push. The image
build fails with an error about the Go version, in a Dockerfile you didn't touch. A
one-line security fix has turned into a toolchain upgrade, on a deadline. It happens
because since Go 1.21 the `go` line in `go.mod` is a requirement, not a hint, and
dependencies can raise it for you.

## The `go` line is a minimum

From the Go documentation on [toolchains](https://go.dev/doc/toolchain): "Go 1.21 changed
the `go` line to be a mandatory requirement instead", and:

> The Go toolchain refuses to load a module or workspace that declares a minimum required
> Go version greater than the toolchain's own version.

The rule that connects it to dependencies:

> A module's `go` line must declare a version greater than or equal to the `go` version
> declared by each of the modules listed in `require` statements.

So when you `go get` a dependency whose own `go.mod` says `go 1.25.0`, your `go.mod`'s
`go` line is raised to at least 1.25.0. The documentation gives the same example: `go get`
"will update the main module's `go` line". The diff for the security fix includes a
`go 1.24.0` → `go 1.25.0` change that's easy to scroll past.

This happens in practice. For example, `github.com/jackc/pgx/v5` declares `go 1.23.0` at
v5.7.6 and `go 1.25.0` at v5.9.0, so moving between those versions raises the floor for
every module that depends on it.

## Why your laptop builds and CI doesn't

Locally, the default `GOTOOLCHAIN=auto` means a Go 1.24 installation that meets a module
requiring 1.25 downloads a newer toolchain and switches to it, printing something like
"requires go >= 1.25; switching to go 1.25.x". The build works.

The official `golang` Docker images switch that off. Their Dockerfile template:

```dockerfile
# don't auto-upgrade the gotoolchain
# https://github.com/docker-library/golang/issues/472
ENV GOTOOLCHAIN=local
```

So `FROM golang:1.24-alpine` refuses to build a module that now says `go 1.25.0`. That's
the right default for reproducible images, since a build shouldn't download a different
compiler from the one the image names. But it means the base image tag in the Dockerfile
is a second copy of the Go version, and it has to move with `go.mod`.

## What to do about it

| Practice | Why |
|---|---|
| Read the `go.mod` diff of every dependency bump, not just the `require` lines | A raised `go` line is a toolchain change hiding in a dependency change |
| Keep the Dockerfile's `golang:` tag and the `go` line in step, ideally checked in CI | They're two copies of one fact |
| Stay within the Go release support window | The [release policy](https://go.dev/doc/devel/release#policy): "Each major Go release is supported until there are two newer major releases." Running older ones makes every security bump a toolchain jump |
| Build with `go mod download` and the committed `go.sum`, not `go mod tidy`, inside the image | `tidy` can rewrite `go.mod`/`go.sum` during the build, so the image isn't built from what was reviewed |

## When the vulnerable module is an indirect dependency

The advisory often names a module you don't import: it's pulled in by something you do.
There are two ways to force the patched version, and they behave very differently.

**Raise the requirement.** `go get example.com/lib@v1.79.3` adds or raises a `require` line.
Minimal version selection then picks the highest version anyone requires, so the build uses
at least v1.79.3, and if a dependency later requires v1.80, it gets v1.80.

**Replace it.** `replace example.com/lib => example.com/lib v1.79.3` is common in security
fixes because it "pins" the version. The [Go modules reference](https://go.dev/ref/mod)
describes what the missing left-hand version means: "If the left version is omitted, all
versions of the module are replaced." All versions, including newer ones. Measured, with a
module that requires a newer gRPC than the replacement:

```
$ go list -m google.golang.org/grpc
google.golang.org/grpc v1.80.0 => google.golang.org/grpc v1.79.3
```

The build silently uses the older version. A version-less `replace` added for a security
patch becomes a downgrade the day a dependency needs something newer, with no error and only
`go list -m` to show it. The same reference adds that "`replace` directives only apply in the
main module's `go.mod` file and are ignored in other modules", so in a library a replace
protects nobody downstream.

Either way, commit the regenerated `go.sum`. Since Go 1.16 the `go` command defaults to
`-mod=readonly`: "if any changes to `go.mod` are needed, the `go` command reports an error and
suggests a fix." A hand-edited `go.mod` without its `go.sum` entries builds on the laptop that
ran `go mod tidy` and fails in CI with `missing go.sum entry for module providing package`
(measured: exactly that error from `go build` on a module whose `go.sum` lacked the new
entries).

## A scanner matches versions; it doesn't know what you call

Image and dependency scanners compare module versions with advisories. They flag a module
whether or not your program uses the vulnerable function. `govulncheck` does the call-graph
analysis. From its [tutorial](https://go.dev/doc/tutorial/govulncheck), for an advisory in a
module whose vulnerable code isn't reachable: "Found 1 vulnerability in packages that you
import, but there are no call stacks leading to the use of this vulnerability. You may not
need to take any action." A server-side flaw in a library you use only as a client is the
classic case. Patching is still cheap insurance, but the urgency, and whether a `replace`
workaround is justified, depends on that answer.

> **Teacher's aside.** People treat the toolchain as infrastructure and dependencies as
> code, upgraded by different people at different times. Since Go 1.21 they're coupled:
> any dependency can require a newer toolchain. The cheapest way to keep security bumps
> small is to keep the toolchain current on a schedule, so a dependency's floor is never
> above where you already are.

## When the dependency itself moves

An organisation moves its Git hosting to a new domain. For most languages that's a change to a few
manifests and CI settings. For Go it's a rename of every internal module, and an edit to every
import statement in every service that uses one, because the host is part of each module's name.

### The path is the name

From the [Go modules reference](https://go.dev/ref/mod#module-path): "A module path is the
canonical name for a module, declared with the `module` directive in the module's `go.mod` file. A
module's path is the prefix for package paths within the module." And: "Typically, a module path
consists of a repository root path, a directory within the repository (usually empty), and a major
version suffix". So a library hosted at `git.oldhost.example/libs/ledger` is imported in every
consumer as `git.oldhost.example/libs/ledger/...`, and the hosting domain is written into source
files all over the organisation.

Other ecosystems keep the location in one manifest:

| Ecosystem | Where the location lives | What the source says | A host move changes |
|---|---|---|---|
| Go | The module path, in `go.mod` and in every import | `import "git.oldhost.example/libs/ledger"` | Every import in every consumer, and the library's own `module` line |
| Rust (Cargo) | `Cargo.toml`: `ledger = { git = "https://…" }` | `use ledger::…` | One line per consumer |
| JavaScript (npm) | `package.json`: `"ledger": "git+https://…"` | `import … from "ledger"` | One line per consumer |
| C++ | The build system, such as CMake's `FetchContent` | `#include <ledger/ledger.h>` | Build files only |

### What a move breaks

If the library's `go.mod` is changed to the new host while consumers still require the old path,
and the old path still reaches the code (a redirect, a mirror), the `go` command refuses it.
Measured, with a local file-based module proxy serving `example.com/old` whose `go.mod` says
`module example.com/new`:

```
go: example.com/old@v1.0.0 requires example.com/old@v1.0.0: parsing go.mod:
	module declares its path as: example.com/new
	        but was required as: example.com/old
```

If the old host is simply gone, fetches by the old path fail outright. A public module may still be
served from the public proxy's copy, but a private one, matched by `GOPRIVATE`, which marks modules
that "should therefore not use the proxy or checksum database" (`go help private`), has no copy to
fall back on. Every consumer has to change its `require` line and its imports, and the new path is
published from a new tag.

During the migration both paths can be in one build. The modules reference treats distinct paths as
distinct modules, in its words for major versions: "modules with distinct suffixes are treated as
separate modules, and their packages—even packages in same subdirectory relative to their module
roots—are distinct." Two copies of the library get compiled in, and a value of a type from one copy
can't be passed where the other's is expected. So a rename goes in dependency order, libraries
before the services that use them, and the last release under the old path should say where to go:

```go
// Deprecated: use go.example.com/ledger instead.
module git.oldhost.example/libs/ledger
```

"Since Go 1.17, `go list -m -u` checks for information on all deprecated modules in the build list.
`go get` checks for deprecated modules needed to build packages named on the command line"
([Deprecation](https://go.dev/ref/mod#go-mod-file-module-deprecation)).

### Vanity import paths

The way to avoid paying this twice is to stop putting the host in the name. A **vanity import
path** is a module path on a domain you control, which tells the `go` command where the code really
lives. From `go help importpath`: for a path on a host it doesn't recognise, the `go` command
requests `https://example.org/pkg/foo?go-get=1` and reads a tag of the form

```html
<meta name="go-import" content="import-prefix vcs repo-root">
```

"The import-prefix is the import path corresponding to the repository root", and the command then
verifies that the prefix's own page carries the same tag. For the ledger library:

```html
<!-- served at https://go.example.com/ledger?go-get=1 -->
<meta name="go-import" content="go.example.com/ledger git https://git.newhost.example/libs/ledger">
```

Consumers import `go.example.com/ledger/...`. When the code moves host again, the tag's last field
changes and no importer notices. With modules there's also a variant whose `vcs` field is `mod`,
pointing at a module proxy instead of a repository, which `go help importpath` says "is preferred
over those listing version control systems".

A vanity path isn't free. The domain and its HTTPS endpoint become build infrastructure: a fetch
that doesn't go through a proxy fails when it's down, and the domain has to be kept for as long as
anyone might build old code. For private modules the `go` command fetches directly unless you run a
private module proxy, so the endpoint must be reachable from every build machine, alongside `GOPRIVATE` and repository credentials. Those
costs are small next to an organisation-wide rename, but they belong to whoever owns the domain.

With modules, the `module` line in `go.mod` is what fixes the path. The old `// import "path"`
comments that enforced vanity paths before modules no longer do anything: "Import path comments are
obsoleted by the go.mod file's module statement" (`go help importpath`).

## Check yourself

1. A dependency bump changes `go.mod`'s `go` line from 1.24 to 1.25. The build passes on a
   developer machine with Go 1.24 installed. Why?
2. The same change fails in a Docker build `FROM golang:1.24-alpine`. Quote the setting
   responsible and say why the image sets it.
3. Can you avoid raising your `go` line while still taking the patched dependency version?
   What are your options?
4. Why is `RUN go mod tidy` in a Dockerfile a reproducibility problem even when it
   usually changes nothing?
5. A service pins a patched transitive dependency with `replace lib => lib v1.79.3`. Six
   months later another dependency is upgraded and requires `lib v1.81.0`. What version is
   built, what tells you, and what should the pin have been?
6. A scanner flags a gRPC advisory about request authorisation on the server side. The
   service only uses gRPC as a client. How do you decide how urgent the fix is?
7. Your organisation's internal Go modules all live under `git.oldhost.example/…`, and hosting is
   moving to a new domain next quarter. What exactly has to change in a service that imports three
   of them, what goes wrong if one library is renamed and another isn't yet, and what would you set
   up so the next move costs nothing?
