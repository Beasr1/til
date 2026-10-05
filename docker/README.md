# Docker Build Caching — A Course

A course on the part of Docker that silently wastes your time, and occasionally
ships you a broken image without a single error message: **the build cache.**

**This is reference learning material.** The concepts are general Docker. The
worked example running through it is real (names changed) — `acme/imgsvc`'s Dockerfile, which on
2026-08-26 built successfully and produced an image containing a 445 KB
`fn main() {}` stub instead of the 22.8 MB service. It started. It exited 0. It
printed nothing. Nothing in the build log said anything was wrong.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- I use the mental model before the commands, and *why* before *how*.
- Every file ends with **Check yourself** questions. Answers are in `09-exercises.md`.
- Every claim about cache behaviour was measured, not assumed. Numbers are real.

## The one thing to understand first

Almost every confusing thing about Docker builds — why your build takes 60
seconds after a one-line edit, why a `RUN` you didn't touch re-ran anyway, why
`touch` didn't invalidate anything — follows from a single fact:

> **A layer's cache validity depends on every layer beneath it. Docker cannot
> reason about whether your change was relevant, so the moment one step differs,
> every step after it is discarded and re-run.**

That is why Dockerfiles are ordered the way they are, why dependency installs
come before source copies, and why the whole `fn main() {}` trick in chapter 4
exists at all.

And the bug in chapter 5 comes from a second fact that sits underneath it:

> **Docker decides "changed" by looking at file *contents*. Your build tool may
> decide it by looking at *timestamps*. `COPY` preserves the original
> timestamps. When those two disagree, nobody raises an error.**

## Reading order

### Part 1 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Images and layers](01-images-and-layers.md) | Say what a layer actually is, and why images are stacks |
| 2 | [The build cache](02-the-build-cache.md) | Predict which steps will re-run before you press build |
| 3 | [Multi-stage builds](03-multi-stage-builds.md) | Ship a 180 MB image from a 2 GB toolchain |

### Part 2 — The pattern and its trap

| # | File | Covers |
|---|------|--------|
| 4 | [Caching dependencies](04-caching-dependencies.md) | Why Dockerfiles copy a manifest before the source, and the fake-entrypoint trick |
| 5 | [**When two caches disagree**](05-when-two-caches-disagree.md) | ⭐ The real bug. Content vs timestamps, and the silent broken image |
| 6 | [Debugging a build](06-debugging-a-build.md) | How the bug was actually found, and how to prove a fix |

### Part 3 — The rest

| # | File | Covers |
|---|------|--------|
| 7 | [Build context and .dockerignore](07-build-context.md) | What you're really uploading, and why it's 6 GB |
| 8 | [Glossary](08-glossary.md) | Look things up |
| 9 | [Exercises & answers](09-exercises.md) | Worked answers plus things to try |

## If you're short on time

- **10 minutes:** chapter 2, then chapter 5.
- **You maintain a Dockerfile:** 2, 4, 5.
- **Your build is slow:** 2, 3, 4, 7.
- **Your build is fast but the image is wrong:** 5, 6.

## The one-paragraph summary of everything

Docker builds an image as a stack of **layers**, one per instruction, and caches
each one. On rebuild it reuses a layer only if that instruction *and everything
below it* are unchanged — so a single early miss invalidates the whole rest of
the file. That is why you put slow, stable things (installing dependencies) near
the top and fast, volatile things (copying your source) near the bottom. For
compiled languages this means copying the dependency manifest alone, building
dependencies against it, and copying real source afterwards — which often
requires fabricating a placeholder entry point, because most build tools have no
"dependencies only" mode. That works until the day the dependency layer actually
re-runs, at which point its freshly-built artifacts carry a *newer* timestamp
than the source `COPY` brings in — and a build tool that judges staleness by
timestamp concludes there is nothing to do. The image builds green and contains
the placeholder.
