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
