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

## Caching a bearer token

Many services you call want an OAuth 2.0 access token, fetched from an identity provider with the
service's own credentials, the **client credentials grant** of
[RFC 6749 §4.4](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4). Fetching one per request
doubles your latency and hammers the provider, so the token is cached. Two symptoms say the cache
is wrong: intermittent 401s between successful calls, and a spike of token requests every time the
token expires.

### Expire it early

The token response carries `expires_in`, which [§5.1](https://www.rfc-editor.org/rfc/rfc6749.html#section-5.1)
defines as "The lifetime in seconds of the access token. For example, the value "3600" denotes that
the access token will expire in one hour from the time the response was generated." Two details in
that sentence matter. The clock starts when the response was generated, not when you received it,
so record the time *before* sending the token request and count from there. And a token that's
valid when you check it can expire while the request carrying it is in flight, or queued behind a
retry. Treat it as expired some margin before the real expiry.

A fixed margin alone fits badly at both ends: 30 seconds is most of a one-minute token, and
nothing for a token that lives a day and a slow upstream. A fraction alone gives a short-lived token
no room. So take the larger of the two, and cap it, because a misconfigured margin bigger than the
lifetime makes every token expired on arrival, and then the cache fetches on every call:

```go
// The larger of a fixed floor and a tenth of the lifetime, but never more than half of it.
func margin(lifetime time.Duration) time.Duration {
    return min(max(30*time.Second, lifetime/10), lifetime/2)
}
```

```
lifetime 1m0s -> margin 30s
lifetime 5m0s -> margin 30s
lifetime 1h0m0s -> margin 6m0s
```

The numbers are mine, not a standard's. For comparison, `golang.org/x/oauth2` treats a token as
expired 10 seconds early by default (`defaultExpiryDelta`,
[`token.go:22`](https://github.com/golang/oauth2/blob/v0.36.0/token.go#L22)), and
`ReuseTokenSourceWithExpiry` lets you choose. If the access token is a JWT, its `exp` claim
([RFC 7519 §4.1.4](https://www.rfc-editor.org/rfc/rfc7519.html#section-4.1.4)) can be earlier than
`expires_in` suggests, and the earliest of the two wins. Don't depend on it, though: RFC 6749 §1.4
says the access token string "is usually opaque to the client".

### Check and use in one step

A client had `ensureToken() error`, which refreshed the token if needed, and a separate getter that
read the cached token afterwards. Between the two calls another goroutine could clear or replace the
token, and now and then a request went out as `Authorization: Bearer ` with nothing after it. The
check and the use were two operations, so the answer to the check was stale by the time of the use.
A cache method has to return the token from the same critical section that decided it was valid.

### Collapse concurrent refreshes

When the token expires, every request in flight finds the cache empty at once, and each one fetches
a new token. Measured, with a 50 ms token endpoint and an empty cache:

```
check, then fetch:           100 concurrent callers, empty cache: 100 token requests
singleflight + re-check:     100 concurrent callers, empty cache: 1 token requests
```

[`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight) is built
for this. `Do` makes sure "that only one execution is in-flight for a given key at a time. If a
duplicate comes in, the duplicate caller waits for the original to complete and receives the same
results."

```go
type TokenCache struct {
    mu      sync.Mutex
    tok     string
    expires time.Time // already moved early by the margin
    flight  singleflight.Group
    fetch   func(ctx context.Context) (tok string, lifetime time.Duration, err error)
}

// cached checks and reads under one lock, so the token it returns is the one it checked.
func (c *TokenCache) cached() (string, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.tok != "" && time.Now().Before(c.expires) {
        return c.tok, true
    }
    return "", false
}

func (c *TokenCache) Token(ctx context.Context) (string, error) {
    if tok, ok := c.cached(); ok {
        return tok, nil
    }
    ch := c.flight.DoChan("token", func() (any, error) {
        if tok, ok := c.cached(); ok { // someone refreshed while we queued
            return tok, nil
        }
        // Not the caller's ctx: if the first caller gives up, the others still need a token.
        fctx, cancel := context.WithTimeout(context.WithoutCancel(ctx), 10*time.Second)
        defer cancel()
        start := time.Now() // expires_in counts from when the response was generated
        tok, lifetime, err := c.fetch(fctx)
        if err != nil {
            return "", err
        }
        c.mu.Lock()
        c.tok, c.expires = tok, start.Add(lifetime-margin(lifetime))
        c.mu.Unlock()
        return tok, nil
    })
    select {
    case r := <-ch:
        if r.Err != nil {
            return "", r.Err
        }
        return r.Val.(string), nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}
```

Two choices in there are easy to get wrong. The fetch runs on a context detached from the first
caller's, because with plain `Do` and the caller's own context, one cancelled request fails the
refresh for every request sharing it (measured with the version above: the caller that gave up got
`context canceled`, and the one behind it still got `tok-1`). And `DoChan` with a `select` lets each
waiter give up on its own deadline, which plain `Do` can't.

There's a simpler correct design, and `x/oauth2` uses it: hold the mutex across the fetch
([`oauth2.go:308-320`](https://github.com/golang/oauth2/blob/v0.36.0/oauth2.go#L308-L320)).
Concurrent callers queue on the lock, the first fetches, and the rest find a valid token when they
get it. The cost is that a waiter can't stop waiting when its context is cancelled. Either is fine.
What isn't fine is checking under the lock, releasing it, and fetching.

### A 401 means invalidate and retry once

[RFC 6750 §3.1](https://www.rfc-editor.org/rfc/rfc6750.html#section-3.1) defines `invalid_token`:
"The access token provided is expired, revoked, malformed, or invalid for other reasons. The
resource SHOULD respond with the HTTP 401 (Unauthorized) status code. The client MAY request a new
access token and retry the protected resource request." Clocks drift and providers revoke tokens
early, so however good the margin, a 401 will come. Treat it as a reason to drop that token and try
once more:

```go
func (c *Client) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
    for attempt := 0; ; attempt++ {
        tok, err := c.tokens.Token(ctx)
        if err != nil {
            return nil, err
        }
        r := req.Clone(ctx)
        r.Header.Set("Authorization", "Bearer "+tok)
        if attempt > 0 && req.GetBody != nil {
            if r.Body, err = req.GetBody(); err != nil {
                return nil, err
            }
        }
        resp, err := c.http.Do(r)
        replayable := req.Body == nil || req.GetBody != nil
        if err != nil || resp.StatusCode != http.StatusUnauthorized || attempt == 1 || !replayable {
            return resp, err
        }
        io.Copy(io.Discard, resp.Body)
        resp.Body.Close()
        c.tokens.Invalidate(tok)
    }
}

// Invalidate drops tok, but only if it's still the cached one.
func (c *TokenCache) Invalidate(tok string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.tok == tok {
        c.tok = ""
    }
}
```

Once, not in a loop: a second 401 with a brand-new token means the credentials or the permissions
are wrong, and retrying again just adds load to the provider. `Invalidate` should clear the token
only if it's still the one that failed, so a request holding an old token doesn't throw away a new
one another request just fetched. A request with a body can be retried only if the body can be
replayed: `http.NewRequestWithContext` fills in `GetBody` for a `*bytes.Buffer`, `*bytes.Reader` or
`*strings.Reader`, and the loop above relies on it.

<details><summary>Rust</summary>

```rust
// tokio's Mutex can be held across .await, so the x/oauth2 design is natural:
// one lock covers check, fetch and store, and the token returned is the one checked.
struct TokenCache {
    inner: tokio::sync::Mutex<Option<(String, std::time::Instant)>>,
}

impl TokenCache {
    async fn token(&self) -> Result<String, Error> {
        let mut guard = self.inner.lock().await;
        if let Some((tok, expires)) = guard.as_ref() {
            if std::time::Instant::now() < *expires {
                return Ok(tok.clone());
            }
        }
        let start = std::time::Instant::now();
        let (tok, lifetime) = fetch_token().await?;
        *guard = Some((tok.clone(), start + lifetime - margin(lifetime)));
        Ok(tok)
    }
}
```

</details>

<details><summary>C++</summary>

```cpp
// The same design with std::mutex: the lock covers check, fetch and store,
// and the function returns a copy taken under the lock.
class TokenCache {
    std::mutex mu_;
    std::string tok_;
    std::chrono::steady_clock::time_point expires_{};
public:
    std::string token() {
        std::lock_guard<std::mutex> lock(mu_);
        auto now = std::chrono::steady_clock::now();
        if (!tok_.empty() && now < expires_) return tok_;
        auto [tok, lifetime] = fetch_token();           // concurrent callers wait on mu_
        tok_ = tok;
        expires_ = now + lifetime - margin(lifetime);
        return tok_;
    }
};
```

</details>

<details><summary>JavaScript</summary>

```js
// One event loop, so no lock: cache the in-flight promise and every caller shares one fetch.
let cached = null;   // { token, expires }
let inflight = null; // Promise<string>

async function getToken() {
  if (cached && Date.now() < cached.expires) return cached.token;
  inflight ??= (async () => {
    const start = Date.now();
    try {
      const { token, lifetimeMs } = await fetchToken();
      cached = { token, expires: start + lifetimeMs - margin(lifetimeMs) };
      return token;
    } finally {
      inflight = null;
    }
  })();
  return inflight;
}
```

</details>

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
6. A token cache stores `time.Now().Add(expiresIn)` after the token response arrives, checks
   validity in `ensureToken() error`, and reads the token with a separate `getToken()`. Name three
   distinct ways a request ends up with a 401 or an empty bearer, and what happens to the identity
   provider at the moment the token expires under 200 concurrent requests.
