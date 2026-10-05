# 1. Images and layers

## What an image actually is

An image is not a file. It is a **stack of filesystem snapshots**, called layers,
plus a small blob of metadata saying what command to run.

Each instruction in a Dockerfile that changes the filesystem produces one layer.
A layer records only the *difference* from the layer below it:

```dockerfile
FROM debian:bookworm-slim     # layer 0: a whole Debian filesystem
RUN apt-get install -y cmake  # layer 1: the files cmake added
COPY app /app                 # layer 2: one new file
```

Layer 2 doesn't contain Debian. It contains `/app`. When you run the container,
Docker stacks them and shows you the union.

## Why "delete" doesn't shrink an image

Because layers only stack, they never subtract. This is the single most
counter-intuitive consequence:

```dockerfile
COPY secret.key /tmp/secret.key
RUN rm /tmp/secret.key
```

The second layer records "this file is deleted" — a whiteout marker. The first
layer **still contains the file**, and anyone with the image can read it.

Same for size:

```dockerfile
RUN git clone --huge-repo /tmp/src && \
    build /tmp/src

RUN rm -rf /tmp/src           # image is NOT smaller
```

To actually not have something in the image, it has to never enter a layer that
survives — either do it all in one `RUN` (so the delete happens before the layer
is snapshotted), or use a multi-stage build (chapter 3), which is the real answer.

The one-`RUN` version:

```dockerfile
RUN git clone --huge-repo /tmp/src && \
    build /tmp/src && \
    rm -rf /tmp/src           # image IS smaller — one layer, cleaned before snapshot
```

This is why you see those long `&&`-chained `RUN` lines with `rm -rf
/var/lib/apt/lists/*` at the end. It isn't style. Split it into two `RUN`s and
the apt cache ships forever.

## Layers are shared

Two images built `FROM debian:bookworm-slim` share that layer on disk and over
the network. Pulling the second image doesn't re-download it. This is also why
changing your base image is expensive and changing your last line is cheap.

## What this buys you

Three things, and they all matter later:

1. **Caching** — an unchanged instruction can reuse its saved layer (chapter 2).
2. **Sharing** — identical layers are stored once.
3. **Transfer** — pushing an image only uploads layers the registry lacks.

---

## Check yourself

1. You add `RUN rm -rf /usr/share/doc` as the last line of your Dockerfile. Does
   the image get smaller?
2. Why do so many Dockerfiles end an `apt-get install` line with
   `&& rm -rf /var/lib/apt/lists/*` instead of putting it on the next line?
3. Two images share the same first four layers. The second image is 900 MB
   according to `docker images`. How much extra disk does it actually use?
4. You accidentally `COPY`d a credentials file, then deleted it in a later
   instruction and rebuilt. Is the credential safe?
