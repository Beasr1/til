# 7. Time in a container

## The problem

A document's issue date is a day early for anything created in the last few hours of
each day. In development it was always right. The code asks for local time in a named
zone and formats a date. In production the zone failed to load, the code quietly fell back
to UTC, and the date is computed on the wrong side of midnight. Nothing crashed, and a log
line nobody reads said why.

## Where time zone rules come from

A time zone name such as `Asia/Dubai` or `Europe/London` isn't something Go knows. It's
looked up in the **IANA time zone database**, a set of files describing each zone's offsets
and daylight-saving rules through history. `time.LoadLocation` has to find those files at
run time. From its documentation:

> LoadLocation looks for the IANA Time Zone database in the following locations in order:
>
> - the directory or uncompressed zip file named by the ZONEINFO environment variable
> - on a Unix system, the system standard installation location
> - $GOROOT/lib/time/zoneinfo.zip
> - the time/tzdata package, if it was imported

On a developer laptop the second location exists, because the operating system installed
it. In a minimal container image, none of them may.

## Measured: two common base images

A static Go binary calling `time.LoadLocation("Asia/Dubai")`, run in each image:

| Base image | Result |
|---|---|
| `alpine:3.21` | `err=unknown time zone Asia/Dubai` |
| `scratch` | `err=unknown time zone Asia/Dubai` |

Alpine doesn't ship `/usr/share/zoneinfo` unless you `apk add tzdata`. `scratch` ships
nothing at all. There's no `$GOROOT` in either, because the Go toolchain lives only in the
build stage of a multi-stage build. This is the same shape of failure as a container with
no CA certificates, which trusts no TLS certificate: the program relies on a file the
operating system was supposed to provide, and the image left it out.

The danger is the fallback. Code like this:

```go
loc, err := time.LoadLocation("Asia/Dubai")
if err != nil {
    log.Error("failed to load timezone, falling back to UTC", "err", err)
    return time.Now().UTC()
}
```

turns a missing file into dates that are wrong only for part of each day. For a zone at
UTC+4, from 20:00 to 24:00 UTC the local date is already tomorrow:

```
21:30 UTC as a date in UTC:      2026-01-01
21:30 UTC as a date in UTC+4:    2026-01-02
```

Tests that run during working hours never see it.

## Three fixes

| Fix | How | When it's right |
|---|---|---|
| Embed the database | `import _ "time/tzdata"` in `main`, or build with `-tags timetzdata` | Any zone, any image. The package documentation: "Importing this package will increase the size of a program by about 450 KB" |
| Install it in the image | `apk add tzdata`, or copy `/usr/share/zoneinfo` from the build stage | When several programs in the image need it |
| Use a fixed offset | `time.FixedZone("GST", 4*60*60)` | Only for a zone with no daylight saving, now or in the past range you care about |

The fixed offset is correct only if the zone really has one offset. That's a fact about the
zone, and you can check it in the IANA source. The `asia` file's entry for Dubai:

```
# Zone  NAME        STDOFF   RULES  FORMAT  [UNTIL]
Zone    Asia/Dubai  3:41:12  -      LMT     1920
                    4:00     -      %z
```

A `-` in the RULES column means no daylight saving, and there's been one offset, +04:00,
since 1920. A fixed offset is therefore exact for Dubai. For London, New York or Sydney it
would be wrong for half the year. The embedded database is the general answer. The fixed
offset is a shortcut that's valid for particular zones only, and the reason it's valid
belongs in a comment next to it.

> ⚠️ **Fail loudly at startup, not quietly per request.** If a service needs a zone, load
> it once in `main` and exit if it's missing. A per-request fallback to UTC converts a
> deploy-time configuration error into months of slightly wrong data.

## Identifiers built from timestamps

The same code often builds human-readable identifiers from the current time plus a few
random digits, something like `MMDDhhmmss` followed by four random digits. Two properties
are worth checking before anyone relies on uniqueness.

**A format without a year repeats.** `0102150405` in Go's layout is month, day, hour,
minute, second. Every value it produces comes round again a year later. If uniqueness
matters across years, the year (or a counter) has to be in the identifier.

**Random suffixes collide sooner than intuition says.** With four random digits there are
10,000 possible suffixes per second. The chance that *k* identifiers issued in the same
second contain at least one collision follows the birthday bound, approximately
1 − e<sup>−k(k−1)/(2·10,000)</sup>:

| Identifiers in one second | Chance of a collision |
|---|---|
| 2 | 0.01% |
| 10 | 0.45% |
| 50 | 12% |
| 100 | 39% |

Those figures are computed from the formula, not measured. If the identifier must be
unique, the database should enforce it with a unique constraint, and the code should
retry generation on a violation. If the identifier only needs to be *distinctive*, the
table above tells you how distinctive.

A small related bias: drawing a random `uint16` (0–65,535) and taking it modulo 10,000
makes the suffixes 0000–5535 slightly more likely than 5536–9999 (7 chances in 65,536
against 6). It's harmless for display identifiers and worth knowing before you use the same
trick anywhere that needs uniform randomness.

## Schedules you parse yourself

A five-field cron expression looks simple enough to parse in fifty lines, and services do
it to avoid a dependency. The format has rules that aren't obvious from examples, and a
parser that gets one wrong still produces *a* schedule, just not the one written.

The POSIX `crontab` specification
([crontab](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/crontab.html)) settles
three of them:

| Rule | What the spec says | The usual mistake |
|---|---|---|
| Day of month and day of week | "any day matching either the month and day of month, or the day of week, shall be matched", when both are restricted | Requiring both, so `0 0 1 * 1` means "the 1st when it's a Monday" instead of "the 1st, and every Monday" |
| Weekday range | 0–6, 0 is Sunday | Expressions copied from elsewhere sometimes use 7 for Sunday; a strict parser rejects them (measured: `value 7 out of range [0, 6]`) |
| Steps (`*/15`) | Not in POSIX at all | An extension, so its edge cases depend on the implementation you copied |

Then there's the mistake the spec can't help with. Here's what a hand-written parser produced
when a single value was handled with the same loop as a step (`for i := v; i <= max; i += step`
with `step` defaulting to 1), measured from 12:00:30:

```
"0 * * * *":  12:01 12:02 12:03 12:04
"30 2 * * *": 12:30 12:31 12:32 12:33
parseCronField("5", 0, 59) size: 55
```

`0` became "every minute from 0 to 59", so "on the hour" ran every minute, and "02:30 daily"
started at lunchtime. Nothing errors. The job simply runs far more often than anyone
intended, which looks like load, not like a parsing bug.

Two more things to settle before trusting a schedule:

- **Whose clock.** `time.Now()` in a container is in the process's local zone, which is UTC in
  `scratch` and `alpine` images unless you set it (earlier in this chapter). "02:30" means
  02:30 UTC unless the code says otherwise.
- **Test the canonical expressions.** `0 * * * *`, `30 2 * * *`, `0 0 1 * 1`, `*/15 * * * *`,
  each with the next three expected times written out. Four table-driven cases would have
  caught everything above. Better still, use a maintained parser and test that it's configured
  the way you think.

## Check yourself

1. A service runs fine on a laptop and in a `golang:` image, and emits wrong dates in a
   `scratch` image. Explain the difference using LoadLocation's search order.
2. Why does the wrong date appear only for part of each day? For a zone at UTC−5, which
   part?
3. When is `time.FixedZone` exactly right, and what would you check before using it for a
   zone you don't know?
4. An identifier format is `MMDDhhmmss` plus four random digits. Name two separate ways two
   records can get the same identifier.
5. You need 200 identifiers a second, all unique. What's the collision probability per
   second with four random digits, roughly, and what would you change?
6. A scheduler parses `0 * * * *` and starts a forty-minute job at each match. The job seems
   to run continuously. Give two parser bugs that could explain it, and the test that would
   have caught both.
