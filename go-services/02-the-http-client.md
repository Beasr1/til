# 2. The HTTP client

## The problem

A service that calls other services over HTTP slowly runs out of something: file
descriptors, memory, or ephemeral ports. Latency to a dependency is higher than the
dependency's own metrics say. Nothing in the code looks wrong, because the bug is a
single line that builds a new client for each call, and Go's HTTP client is designed
around the opposite assumption.

## What a `Transport` is

In Go, `http.Client` is a thin policy layer (redirects, cookies, an overall timeout).
The work happens in its `Transport`, which owns the **connection pool**: open TCP (and TLS)
connections to each host, kept alive between requests so the next request to the same host
can skip the handshake.

The `net/http` source says it outright, in the `Transport` documentation:

> Transports should be reused instead of created as needed. Transports are safe for
> concurrent use by multiple goroutines.

A Transport created for one request has a pool that one request uses. Afterwards its
connection sits idle in that pool. Nothing will ever ask the pool for it again, and
nothing closes it either.

## Measured: one Transport per request

Two hundred sequential GET requests to a local server, each response body read fully and
closed:

| Client | New TCP connections | Goroutines still alive afterwards |
|---|---|---|
| One shared `Transport` | 1 | 3 |
| `&http.Client{Transport: &http.Transport{}}` per request | 200 | 600 |

The 600 are about three per connection: the client side keeps a read loop and a write loop
for each idle connection, and the server keeps one goroutine per open connection. They
stayed alive after a forced garbage collection, because the goroutines themselves keep the
connections reachable. In a long-running service this grows with every request until
something runs out.

Why didn't anything time them out? Because a zero-value `Transport` has no idle timeout.
From the field's documentation: "IdleConnTimeout is the maximum amount of time an idle
(keep-alive) connection will remain idle before closing itself. Zero means no limit."

## A zero-value `Transport` isn't the default one

`&http.Transport{}` also throws away everything `http.DefaultTransport` sets up. The
default, from `transport.go`:

```go
var DefaultTransport RoundTripper = &Transport{
    Proxy: ProxyFromEnvironment,
    DialContext: defaultTransportDialContext(&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
    }),
    ForceAttemptHTTP2:     true,
    MaxIdleConns:          100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```

| Lost by using `&http.Transport{}` | Consequence |
|---|---|
| `Proxy: ProxyFromEnvironment` | `HTTPS_PROXY` / `NO_PROXY` ignored, so it breaks in proxied networks |
| Dial timeout, TCP keep-alive | Connecting to a black-holed address waits until the operating system gives up or the request's own deadline passes |
| `IdleConnTimeout: 90s` | Idle connections are never closed |
| `TLSHandshakeTimeout: 10s` | A stalled handshake waits on the overall deadline |
| `ForceAttemptHTTP2` | Harmless on a bare `&http.Transport{}`, but the moment you add a custom dialler or `TLSClientConfig`, "use of any those fields conservatively disables HTTP/2" |

If you need a custom transport (a TLS setting, an instrumentation wrapper), start from a
clone of the default and keep it for the life of the process:

```go
base := http.DefaultTransport.(*http.Transport).Clone()
base.MaxIdleConnsPerHost = 32 // the default is 2, low for one busy upstream

var client = &http.Client{
    Transport: metrics.InstrumentedRoundTripper{Next: base},
    Timeout:   10 * time.Second,
}
```

`MaxIdleConnsPerHost` deserves a look: `DefaultMaxIdleConnsPerHost = 2`. A service making
fifty concurrent calls to one upstream keeps only two connections idle between bursts and
opens the other forty-eight fresh each time.

## Reading the body is part of reuse

A connection goes back to the pool only once the response body has been read to the end
and closed. The client documentation: "If the Body is not both read to EOF and closed,
the Client's underlying RoundTripper (typically Transport) may not be able to re-use a
persistent TCP connection to the server for a subsequent "keep-alive" request." Always `defer resp.Body.Close()`, and read the body even
on error statuses, using `io.Copy(io.Discard, resp.Body)` if you don't need it.

## Where the timeout lives

There are several layers of timeout, and they compose:

| Layer | Setting | Covers |
|---|---|---|
| Whole request | `http.Client.Timeout` | "connection time, any redirects, and reading the response body" |
| Whole request | A context deadline on the request (`NewRequestWithContext`) | The same, and it can come from the caller |
| Connect | `net.Dialer.Timeout` | TCP connect only |
| TLS | `Transport.TLSHandshakeTimeout` | Handshake only |
| Headers | `Transport.ResponseHeaderTimeout` | Waiting for the first response byte after sending |

The most important choice is which **context** the request uses. A client helper that
does `context.WithTimeout(context.Background(), timeout)` internally has a timeout, but
it has cut the request off from its caller. If the inbound request that triggered this call
is cancelled, because the user went away or an upstream deadline passed, the outbound call
carries on regardless. Chapter 4 is about why that matters. The helper should take a
`ctx` parameter and derive from it.

## Two small traps

> ⚠️ **Logging headers logs credentials.** A debug line that prints the outbound request's
> headers prints `Authorization` and API keys with them. Debug logging is switched on in
> production during incidents, which is exactly when logs are copied around. Log header
> *names*, or redact a known list.

> ⚠️ **`InsecureSkipVerify` behind a flag is still a flag.** Turning off certificate
> verification "for staging only" through an environment variable means one wrong value
> in one deployment disables it in production, with no error. The usual root cause is a
> private certificate authority. Add that CA to the client's `RootCAs` instead, which keeps
> verification on everywhere.

## Check yourself

1. A helper builds `&http.Client{Transport: &http.Transport{}}` on every call. The service
   makes about 50 outbound calls a second to one host. Predict what you'd see after a day,
   and on which graph.
2. Why does the garbage collector not clean up the abandoned Transports and their
   connections?
3. You replace the per-call transport with one shared `&http.Transport{}`. Which problem is
   fixed, and which remain?
4. A handler gets a 500 from an upstream, logs the status and returns without reading or
   closing the body. What happens to the connection?
5. An HTTP helper creates its own `context.WithTimeout(context.Background(), 30*time.Second)`.
   The user's request is cancelled after one second. What does the outbound call do, and
   what should the helper's signature be?
