# 2. The build cache

## The question Docker asks

On every build, Docker walks the instructions top to bottom and asks at each one:

> Is this instruction, executed on top of exactly this parent layer, something I
> have already done?

- **Yes** → reuse the stored layer. Instant.
- **No** → execute it, store the result.

## The rule that explains everything

> **Once one instruction misses the cache, every instruction after it also misses.**

Not because Docker is lazy — because it *cannot* know better. Step 5's saved
result was computed on top of step 4's filesystem. If step 4 produced something
different this time, step 5's stored output describes a world that no longer
exists. It has to be discarded.

You'll see this in build output as a run of `CACHED` lines that stops abruptly
and never resumes:

```
 => CACHED [2/6] RUN apt-get update && apt-get install -y cmake
 => CACHED [3/6] COPY Cargo.toml Cargo.lock ./
 => [4/6] RUN cargo build --release            ← miss
 => [5/6] COPY crates/ crates/                 ← miss, because 4 missed
 => [6/6] RUN cargo build --bin imgsvc          ← miss, because 5 missed
```

## The practical consequence: ordering is the design

Because a miss cascades downward, you order instructions by **how often they
change**, cheapest and most stable first:

```dockerfile
FROM rust:1.88-bookworm         # ~never
RUN apt-get install -y cmake    # rarely
COPY Cargo.toml Cargo.lock ./   # occasionally
RUN <build dependencies>        # slow, but only when the line above changes
COPY crates/ crates/            # constantly
RUN cargo build --bin imgsvc     # fast
```

Invert the last four lines and every one-character source edit reinstalls your
toolchain.

## How Docker decides "unchanged" — and this is the important part

It depends on the instruction:

**`RUN`** — the cache key is the **command string**, verbatim. Docker does not
execute it to see what would happen. This is why:

```dockerfile
RUN apt-get update && apt-get install -y curl
```

can be cached for months and quietly install a stale `curl`: the string never
changed, so Docker never re-ran it, so `apt-get update` never fetched a new index.
Adding a comment to the line changes the string and forces a re-run.

**`COPY` / `ADD`** — the cache key is a **checksum of the copied files' contents**.

And the Docker documentation is explicit about what is *not* in that checksum:

> the last-modified and last-accessed times of the file(s) are not considered

Read that twice. It is the whole of chapter 5.

**`FROM`** — the resolved base image digest.

**`ENV`, `WORKDIR`, `LABEL`** — the literal string.

## The consequence you can test in ten seconds

```bash
touch src/main.rs
docker build .
```

Everything comes back `CACHED`. You changed the file's timestamp, not its
contents, so the `COPY` checksum is identical and Docker sees nothing new.

This is correct behaviour and it is also exactly the trap in chapter 5, because
your *compiler* may care enormously about that timestamp.

## Cache on a fresh machine

CI runners usually start with **no layer cache at all**. Every build is a
complete miss from line 1. This matters more than it sounds: a Dockerfile that
works fine on your laptop, where the expensive layers have been cached for weeks,
takes a completely different path on a cold runner — including, sometimes, a
path that is broken. Chapter 5 is precisely that case.

You can push and pull the cache explicitly:

```bash
docker build --cache-from myregistry/app:cache -t myregistry/app:latest .
```

---

## Check yourself

1. You edit line 30 of a 40-line Dockerfile. How many instructions re-run?
2. You edit line 3. How many re-run?
3. Your `RUN apt-get update && apt-get install -y nginx` layer was cached six
   months ago. What version of nginx do you get today, and why?
4. `touch main.rs && docker build .` reports everything `CACHED`. Is that a bug?
5. Why might a Dockerfile that works on your laptop produce a different image on
   a fresh CI runner, with no change to the Dockerfile?
