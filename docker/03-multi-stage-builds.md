# 3. Multi-stage builds

## The problem

Compiling needs a toolchain. Running does not.

A Rust build image is ~1.5 GB: rustc, cargo, the standard library source, a
linker, often cmake and a C++ compiler for native dependencies. The thing you
actually want to ship is one statically-ish linked binary of maybe 20 MB.

Chapter 1 explained why you can't fix this by deleting: layers only stack, so
`RUN rm -rf /usr/local/rustup` leaves every byte in the image.

## The solution

Use two `FROM`s. Build in the first, copy the result into the second.

```dockerfile
# ── Builder ───────────────────────────────────────────────
FROM rust:1.88-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# ── Runtime ───────────────────────────────────────────────
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/imgsvc /app/imgsvc
ENTRYPOINT ["/app/imgsvc"]
```

The final image contains Debian plus one binary. Everything about the builder —
the compiler, the 400 compiled dependency crates, the intermediate object files —
is discarded. It is never in a layer of the final image, so it cannot be
extracted from it, cannot be scanned as a vulnerability in it, and does not count
toward its size.

`AS builder` names the stage; `--from=builder` reaches into it. You can have as
many stages as you like and copy between any of them.

## What to install in which stage

This is the part people get wrong, and it produces one of two failures.

**Build-time-only** — compilers, `cmake`, `-dev` header packages, `git`. Builder
stage only.

**Runtime** — shared libraries your binary links against, `ca-certificates` for
outbound TLS, and anything your program shells out to. Runtime stage.

Get it wrong one way and your image is bloated. Get it wrong the other way and
the container dies on start with a missing `.so` — a failure that never appears
during the build, because the build stage had it.

Static linking sidesteps the second problem: nothing to find at runtime.

## A worked example

Before, imgsvc's runtime stage carried `libopencv-core-dev` and this line:

```dockerfile
COPY --from=builder /app/crates/nfiq2/vendor/NFIQ2 /app/crates/nfiq2/vendor/NFIQ2
```

That is **1.2 GB of C++ source** copied into a *runtime* image — to reach one
~1 MB model file the program loaded at startup. A `TODO` above it admitted this
was wrong.

When the scoring library was replaced with one that embeds its model in the
binary, both the `COPY` and the OpenCV package became unnecessary. The runtime
stage became:

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates wget && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /app/target/release/imgsvc /app/imgsvc
```

Final image: **182 MB**, of which 22.8 MB is the binary.

The lesson generalises: **anything your runtime stage has to copy out of the
build tree is worth questioning.** It usually means a path is baked into the
program that shouldn't be.

---

## Check yourself

1. Why can't you shrink a builder image by deleting the compiler at the end?
2. Your image builds fine but the container exits immediately with
   `error while loading shared libraries: libpq.so.5`. Which stage is wrong?
3. Should `ca-certificates` go in the builder, the runtime, or both?
4. A runtime stage does `COPY --from=builder /app/config/defaults.yaml
   /app/config/defaults.yaml`. What does that tell you about the program?
