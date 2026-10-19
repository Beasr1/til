# 10. Routing and frameworks

## The problem

A new Go service needs a router, and the choice is usually made by habit. Teams that learnt Go
before 1.22 reach for a framework such as gin, because the standard library's router couldn't
match a method or capture an id from a path. It can now. Teams that pick the standard library
find out later what it doesn't do, and write validation and route groups by hand. Neither choice
is a disaster. The expensive mistake is quieter: a gin handler passes `c` to a database call that
wants a `context.Context`. It compiles, and the query keeps running after the client has gone.

This chapter covers what `http.ServeMux` does since Go 1.22, what a framework adds (gin is the
worked example), the middle ground (chi), and how to choose. It assumes
[chapter 1](01-how-a-go-service-runs.md) (`context`, handlers running concurrently) and
[chapter 3](03-the-http-server.md) (handler contexts and `TimeoutHandler`). Outputs are
measurements on Go 1.26.9 and gin v1.12.0, the latest gin release, unless they say otherwise.

## What `http.ServeMux` does since Go 1.22

Before Go 1.22, a `ServeMux` pattern was a fixed path such as `/favicon.ico` or a subtree such as
`/images/`. It couldn't match a method or capture part of a path, so third-party routers filled
the gap. Go 1.22 added both. The [`ServeMux` documentation](https://pkg.go.dev/net/http#ServeMux)
gives a pattern's shape as `[METHOD ][HOST]/[PATH]`:

| Feature | Example | Behaviour |
|---|---|---|
| Method | `GET /items/{id}` | "A pattern with the method GET matches both GET and HEAD requests. Otherwise, the method must match exactly" |
| Wildcard | `/items/{id}` | Matches one path segment, read with `r.PathValue("id")`. "Wildcards must be full path segments", so `/b_{bucket}` is invalid |
| Remainder | `/files/{path...}` | "matches the remainder of the URL path, including slashes". Only at the end of a pattern |
| End anchor | `/{$}` | "matches only the end of the URL": `/{$}` is just `/`, while `/` "matches every path" |
| Matched pattern | `r.Pattern` | Since Go 1.23, the pattern that matched: a route label for logs and metrics |

Measured: six routes and nine requests. The handler prints `r.Pattern` and any path values, and
the client shows redirects rather than following them.

```go
mux.HandleFunc("GET /items/{id}", show)
mux.HandleFunc("DELETE /items/{id}", show)
mux.HandleFunc("GET /items/latest", show)
mux.HandleFunc("GET /files/{path...}", show)
mux.HandleFunc("GET /docs/", show)
mux.HandleFunc("GET /{$}", show)
```

```
GET    /items/42                  200  "GET /items/{id}" id=42
GET    /items/latest              200  "GET /items/latest"
HEAD   /items/42                  200
POST   /items/42                  405  Allow: DELETE, GET, HEAD
GET    /files/2026/q3/report.pdf  200  "GET /files/{path...}" path=2026/q3/report.pdf
GET    /                          200  "GET /{$}"
GET    /nothing-here              404
GET    /docs                      307  Location: /docs/
GET    /items//42                 307  Location: /items/42
```

**`POST /items/42` got 405, not 404.** When no pattern matches the method, the mux checks whether
one matches the path with another method, and if so answers `405 Method Not Allowed` with an
`Allow` header listing them
([`server.go`](https://github.com/golang/go/blob/go1.26.9/src/net/http/server.go#L2719-L2731)).
HEAD is listed because GET implies it.

**`/items/latest` went to its own handler**, though `/items/{id}` matches it too. That's the
precedence rule, below.

**The redirects are 307s.** Registering `/docs/` makes the mux redirect `/docs` to `/docs/`, and it
redirects paths with repeated slashes or `..` to clean ones. Go 1.24.4 sent both as 301s. The
[Go 1.26 release notes](https://go.dev/doc/go1.26#nethttppkgnethttp): "ServeMux trailing slash
redirects now use HTTP status 307 (Temporary Redirect) instead of 301 (Moved Permanently)." That
matters for anything but GET, because Go's client turns a followed 301 into a GET "with no body",
while a 307 "preserves the original HTTP method and body". Measured, `POST /orders` to a mux with
only `POST /orders/`:

```
go1.24.4: POST /orders, redirect [301 -> GET /orders/], final 405 Method Not Allowed
go1.26.9: POST /orders, redirect [307 -> POST /orders/], final 200 created from body "{\"sku\":\"A1\"}"
```

The status follows the toolchain, not the `go` line: a module declaring `go 1.22`, built with Go
1.27.2, also sent 307.

### Most specific wins, and a tie panics at startup

From the documentation:

> If two or more patterns match a request, then the most specific pattern takes precedence. A
> pattern P1 is more specific than P2 if P1 matches a strict subset of P2's requests; that is, if
> P2 matches all the requests of P1 and more. If neither is more specific, then the patterns
> conflict.

`/items/latest` matches a subset of what `/items/{id}` matches, so it wins, in either registration
order. When neither pattern is a subset of the other, the mux refuses to guess. `/posts/{id}` and
`/{resource}/latest` both match `/posts/latest`, and registering the second panics:

```
panic: pattern "/{resource}/latest" (registered at example.com/routing/cmd/muxdemo/main.go:76) conflicts with pattern "/posts/{id}" (registered at example.com/routing/cmd/muxdemo/main.go:74):
/{resource}/latest and /posts/{id} both match some paths, like "/posts/latest".
But neither is more specific than the other.
/{resource}/latest matches "/resource/latest", but /posts/{id} doesn't.
/posts/{id} matches "/posts/id", but /{resource}/latest doesn't.
```

An ambiguous route table is a crash at startup, with an explanation, instead of a request quietly
sent to the wrong handler. The other routers in this chapter are more forgiving, which isn't
obviously better. Measured on chi v5.5.0 and gin v1.12.0: both accepted that pair and sent
`/posts/latest` to `/posts/{id}`, and chi let a second `GET /a` silently replace the first.

### The escape hatch, and the trap inside it

The new syntax changed what some old patterns mean: in Go 1.21, `/{x}` matched only the literal
path `/{x}`. So the documentation offers a way back: "set the GODEBUG environment variable to
"httpmuxgo121=1". This setting is read once, at program startup". Run with it, the same binary
answered 404 to every request that had matched, and kept only the clean-path redirect, as a 301.
Three of the nine lines:

```
GET    /items/42                  404
POST   /items/42                  404
GET    /items//42                 301  Location: /items/42
```

The old mux treats a pattern that doesn't begin with `/` as host plus path
([`servemux121.go`](https://github.com/golang/go/blob/go1.26.9/src/net/http/servemux121.go#L75)),
so `GET /items/{id}` waits for requests to a host called `GET `. The conflicting pair registered
without complaint, and `r.Pattern` stayed empty, which the [Go 1.23 notes](https://go.dev/doc/go1.23)
document: it "is not set when GODEBUG=httpmuxgo121=1 is set".

You can get this without setting anything. GODEBUG defaults follow the `go` line in `go.mod`, as
`GOMAXPROCS` does in chapter 1, and Go's
[settings table](https://github.com/golang/go/blob/go1.26.9/src/internal/godebugs/table.go#L50)
records `httpmuxgo121` as changed in 1.22 with an old value of `1`. Measured: the same code in a
module declaring `go 1.21`, built with Go 1.26.9 and no GODEBUG set, printed the same output. It
compiles and registers every route. `go vet` flags a call to `r.PathValue` in such a module
("requires go1.22 or later"), but a method inside a pattern is just part of a string.

The [GODEBUG policy](https://go.dev/doc/godebug) keeps compatibility settings "for a minimum of
two years (four Go releases)". Four releases after 1.22 is 1.26, so this one is past its
guarantee. Go 1.27 removed several other 1.22-era settings and kept it, but treat it as a
migration aid, not somewhere to stay.

## What a framework adds: gin

gin is a framework, not just a router. It routes with "a custom version of HttpRouter", in its
[README](https://github.com/gin-gonic/gin/blob/v1.12.0/README.md)'s words, and adds binding and
validation, rendering, a middleware chain and its own request context.

### Its own handler type

```go
type HandlerFunc func(*Context)   // gin.go:51
```

Every gin handler and middleware has this shape, not `http.Handler`'s. The engine is itself an
`http.Handler`, so it plugs into `http.Server` or `TimeoutHandler`, but inside, the two kinds don't
mix. `gin.WrapH` and `gin.WrapF` run a standard handler as the last link of a chain. gin v1.12.0
has no adapter for standard middleware, `func(http.Handler) http.Handler`, so middleware written
for `net/http` needs a gin version or a wrapper you write. Parameters come from `c.Param("id")`,
and gin never calls `SetPathValue`, so `c.Request.PathValue("id")` is `""` (measured).

### Binding and validation

`ShouldBindJSON` decodes the body into a struct and validates it with
[go-playground/validator](https://github.com/go-playground/validator), which gin configures to read
the `binding` tag
([`default_validator.go:93`](https://github.com/gin-gonic/gin/blob/v1.12.0/binding/default_validator.go#L93)).
Measured, posting `{"email":"not-an-email","qty":0}` to a handler that binds this struct and
returns any error with a 400:

```go
type CreateItem struct {
    Name  string `json:"name"  binding:"required"`
    Email string `json:"email" binding:"required,email"`
    Qty   int    `json:"qty"   binding:"gte=1,lte=100"`
}
```

```
400  Key: 'CreateItem.Name' Error:Field validation for 'Name' failed on the 'required' tag
Key: 'CreateItem.Email' Error:Field validation for 'Email' failed on the 'email' tag
Key: 'CreateItem.Qty' Error:Field validation for 'Qty' failed on the 'gte' tag
```

The `Bind…` variants also abort the request with a 400 themselves, per gin's
[docs](https://github.com/gin-gonic/gin/blob/v1.12.0/docs/doc.md#model-binding-and-validation).
None of this needs a framework. The validator is an ordinary library, and you can call it on any
struct you decoded with `encoding/json`. What gin gives you is the wiring, done once.

### Groups and the middleware chain

`r.Group("/api", requireAuth)` gives a set of routes a prefix and their own middleware. A
middleware calls `c.Next()` to run the rest of the chain inside itself, and `c.Abort()` to stop
the handlers after it. Abort's doc comment adds: "Note that this will not stop the current
handler" ([`context.go:203`](https://github.com/gin-gonic/gin/blob/v1.12.0/context.go#L203)).
Measured, with a logger and an auth middleware that aborts and then records a step:

```
chain, authorised:   logger:before → auth:ok → handler → logger:after
chain, no auth:      logger:before → auth:aborted, but this line still runs → logger:after
```

Code after `Abort` runs unless you `return`, and the outer middleware's second half runs either
way.

gin's routing defaults differ from `ServeMux` in ways API clients notice, all measured. A wrong
method gets **404**, because `HandleMethodNotAllowed` defaults to false. A GET route doesn't answer
HEAD. And `RedirectTrailingSlash`, on by default, redirects `/items/42/` to `/items/42`, with 301
for GET and 307 for other methods.

### The context is pooled

From gin's `ServeHTTP`
([`gin.go:662-675`](https://github.com/gin-gonic/gin/blob/v1.12.0/gin.go#L662-L675)):

```go
c := engine.pool.Get().(*Context)
c.writermem.reset(w)
c.Request = req
c.reset()
engine.handleHTTPRequest(c)
engine.pool.Put(c)
```

When the handler returns, its `*gin.Context` goes back to a `sync.Pool`, and a later request gets
the same struct with new contents. Hence the rule in gin's docs, under
"[Goroutines inside a middleware](https://github.com/gin-gonic/gin/blob/v1.12.0/docs/doc.md#goroutines-inside-a-middleware)":
"When starting new Goroutines inside a middleware or handler, you **SHOULD NOT** use the original
context inside it, you have to use a read-only copy." The copy is `c.Copy()`, a new struct with its
own params and keys. Measured, 500 sequential requests to `/users/:id`, each handler starting a
goroutine that reads `c.Param("id")` 5 ms later:

```
kept c       : 490 of 500 goroutines read another request's id
  e.g. goroutine for id=u0 read id="u71", path="/users/u71"
kept c.Copy(): 0 of 500 goroutines read another request's id
```

A `-race` build reports `(*Context).Param` in the goroutine racing with `(*Context).reset` in
`(*Engine).ServeHTTP`. It's [chapter 6](06-copies-that-share-memory.md)'s shared-memory bug
stretched over time: two requests, one struct.

### `c` is a `context.Context` that doesn't follow the request

`*gin.Context` implements `context.Context`, so `db.QueryContext(c, …)` compiles. Each of the four
methods consults one engine field
([`context.go`](https://github.com/gin-gonic/gin/blob/v1.12.0/context.go#L1434-L1485)):

```go
func (c *Context) hasRequestContext() bool {
    hasFallback := c.engine != nil && c.engine.ContextWithFallback
    hasRequestContext := c.Request != nil && c.Request.Context() != nil
    return hasFallback && hasRequestContext
}
```

`ContextWithFallback` is false unless you set it. While it's false, `Done()` returns nil, a channel
that never fires. `Err()` returns nil, `Deadline()` reports none, and `Value()` finds only gin's own
keys. `Done`'s comment says it returns nil "when c.Request has no Context", but it does so whenever
the flag is off, and gin's `docs/doc.md` never mentions the flag.

> ⚠️ **With a default gin engine, passing `c` as a `context.Context` cuts the work off from the
> request.** A query given `c` keeps running when the client disconnects, ignores a
> `TimeoutHandler` deadline, and can't see values that `net/http` middleware put on the request's
> context, such as a trace span. Nothing errors. It's a third break in
> [chapter 4](04-deadlines-and-retries.md)'s chain, and the hardest to spot in review, because `c`
> looks like the request's context. Pass `c.Request.Context()`, or set
> `engine.ContextWithFallback = true` where the engine is built.

Measured on gin v1.12.0. `work(ctx, d)` waits for `d` or for `ctx.Done()`, whichever comes first:

| What the handler did | `ContextWithFallback: false` (default) | `true` |
|---|---|---|
| Client gave up at 100 ms; `work(c, 2s)` | Ran the full 2 s, `err=nil` | Stopped at 100 ms, `context canceled` |
| Same, `work(c.Request.Context(), 2s)` | Stopped at 100 ms, `context canceled` | The same |
| Behind `http.TimeoutHandler(engine, 200ms, …)`; `work(c, 1s)` | Client got 503; work ran 1 s; `c.Deadline()` reported none | Stopped at 200 ms, `context deadline exceeded` |
| `c.Value(k)` for a value set by outer `net/http` middleware | `nil` | The value |
| Goroutine after the handler returned: `work(c.Copy(), 10ms)` | `err=nil` | `context canceled` |
| Same goroutine, `context.WithoutCancel(c.Request.Context())` | `err=nil`, value kept | `err=nil`, value kept |

The last two rows are why the default is false. gin v1.8.0 made `c` follow the request's context,
and [issue #3166](https://github.com/gin-gonic/gin/issues/3166) followed: "With gin v1.8.0,
gin.Context is canceled within goroutine". Code that handed `c.Copy()` to a goroutine, as the docs
advise, broke. The copy keeps the same `*http.Request`, and `net/http` cancels that request's
context when the handler returns. "Add a new flag to make it backward compatible," a maintainer
replied, and v1.8.1 did ([#3172](https://github.com/gin-gonic/gin/pull/3172)), defaulting to off.
So the default protects one documented pattern by disconnecting cancellation everywhere else.

With the flag set, give background work a context of its own.
`context.WithoutCancel(c.Request.Context())` keeps the values and drops the cancellation, and a
`context.WithTimeout` on top of it bounds the work again.

### What it brings with it

gin v1.12.0's [`go.mod`](https://github.com/gin-gonic/gin/blob/v1.12.0/go.mod) declares
`go 1.25.0` and lists 15 direct and 20 indirect requirements, so adding it raises your `go` line to
at least 1.25 ([chapter 8](08-when-a-dependency-moves-your-toolchain.md)). Measured, a one-route
program per router, built with Go 1.26.9 for darwin/arm64:

| Router | Modules compiled in | Non-standard packages | Binary | Stripped (`-s -w`) |
|---|---|---|---|---|
| `http.ServeMux` | 0 | 0 | 8.0 MB | 5.5 MB |
| chi v5.5.0 | 1 | 1 | 8.4 MB | 5.7 MB |
| gin v1.12.0 | 19 | 101 | 20.5 MB | 13.8 MB |

The nineteen include quic-go, an HTTP/3 implementation that `gin.go` imports for `RunQUIC`, plus
protobuf, MongoDB's BSON package, and YAML and TOML parsers for the formats gin binds and renders.
A program that never speaks HTTP/3 still links it, and every module is another version for
scanners to match against advisories, so chapter 8's `govulncheck` section applies.

## The middle ground: chi

[chi](https://github.com/go-chi/chi) is a router that keeps the standard types. Its `Router`
interface embeds `http.Handler`, middleware is `func(http.Handler) http.Handler`, and the README
promises "No external dependencies - plain ol' Go stdlib + net/http". Its `go.mod` requires
nothing. What it adds to `ServeMux` is structure:

```go
r := chi.NewRouter()
r.Route("/api", func(api chi.Router) {      // a sub-router at a prefix
    api.Use(requireAuth)
    api.Get("/items/{id}", getItem)
    api.Group(func(admin chi.Router) {       // same prefix, more middleware
        admin.Use(requireAdmin)
        admin.Delete("/items/{id}", deleteItem)
    })
})
r.Mount("/debug", middleware.Profiler())    // any http.Handler, at a prefix
```

From the [`Router` interface](https://github.com/go-chi/chi/blob/v5.5.0/chi.go#L68-L87): `Group`
"adds a new inline-Router along the current routing path, with a fresh middleware stack", and
`Mount` "attaches another http.Handler along ./pattern/*". chi also accepts two things `ServeMux`
rejects, both shown in its README: regular-expression parameters (`{articleSlug:[a-z-]+}`) and
several parameters in one segment (`/{month}-{day}-{year}`).

Since v5.0.12 chi copies its parameters into the request with `r.SetPathValue`, and since v5.3.0
it sets `r.Pattern` too
([`mux.go:487-489`](https://github.com/go-chi/chi/blob/v5.5.0/mux.go#L487-L489)). So a handler that
reads `r.PathValue("id")` runs unchanged under either router (measured: `id=42` from both), and
moving from `ServeMux` to chi changes the route table, not the handlers. Its defaults sit between
the other two: a wrong method gets 405 with `Allow`, and HEAD on a GET route gets 405 unless you
add `middleware.GetHead`.

## Speed

Router READMEs lead with speed, so it's worth measuring once: the same 26-route API in each
router, a handler that reads its parameters, and a response writer that discards everything.
`go test -bench`, six runs, medians from benchstat, Go 1.26.9 on an Apple M4 Pro:

| Request | `ServeMux` | chi | gin |
|---|---|---|---|
| `GET /invoices/42` | 75 ns, 1 alloc | 202 ns, 4 allocs | 28 ns, 0 allocs |
| `GET /users/42/orders/7` | 140 ns, 3 allocs | 255 ns, 5 allocs | 47 ns, 1 alloc |

For scale, measured on the same machine:

| Operation | Median |
|---|---|
| The slowest route above | 255 ns |
| One HTTP request over a kept-alive loopback connection | 30 µs |
| One HTTPS request to a public server over a kept-alive connection, 20 requests | 14 ms |

gin is the fastest, and it allocated nothing on the one-parameter route, as its "Zero allocation
router" claim says. Its "up to 40 times faster" compares it with Martini, an older framework, not
with `ServeMux`. chi's allocations come from cloning the request to attach its routing context, as
its README explains. None of it matters much. The slowest route costs under 1% of a loopback
request, the cheapest network call a service can make. Computed: the roughly 200 ns between chi
and gin, at 10,000 requests a second, is 2 ms of CPU per second, 0.2% of one core.

## How to choose

| | `http.ServeMux` | chi | gin |
|---|---|---|---|
| Handler type | `http.Handler` | `http.Handler` | `func(*gin.Context)` |
| Middleware | `func(http.Handler) http.Handler`, composed by hand | The same, with `Use`, `With` and `Group` | gin's own, with `c.Next()` and `c.Abort()` |
| Path parameters | `r.PathValue` | `chi.URLParam`, and `r.PathValue` | `c.Param` only |
| Regex, several parameters per segment | No | Yes | No |
| Groups and sub-routers | By hand | `Route`, `Group`, `Mount` | `Group` |
| Wrong method | 405 with `Allow` | 405 with `Allow` | 404, unless `HandleMethodNotAllowed` |
| HEAD on a GET route | Answered | 405, unless `middleware.GetHead` | 404 |
| Ambiguous routes | Panic at registration | Accepted | Accepted |
| Duplicate routes | Panic | Last one wins, silently | Panic |
| Binding and validation | No | No | Yes |
| Context for downstream calls | `r.Context()` | `r.Context()` | `c.Request.Context()`, or `c` with `ContextWithFallback` |
| Modules compiled in | 0 | 1 | 19 |

Three practitioners agree on these facts and differ on taste:

- Alex Edwards [compared 30 routers](https://www.alexedwards.net/blog/which-go-router-should-i-use)
  (updated May 2025). His shortlist, `http.ServeMux`, httprouter, chi, flow and gorilla/mux, is all
  `http.Handler`-compatible. He recommends "`http.ServeMux` as your go-to router" and choosing
  "based on the specific features that you need rather than performance". gin isn't in it: the
  article compares routers, and gin is a framework.
- Eli Bendersky, [before Go 1.22 shipped](https://eli.thegreenplace.net/2023/better-http-server-routing-in-go-122/),
  expected that "many will find the new stdlib mux sufficient for their needs", while many
  programmers "opt for lightweight frameworks like Gin, which provide a router but also additional
  tools for building web backends".
- Jon Calhoun ([ServeMux vs chi](https://calhoun.io/go-servemux-vs-chi)) says chi "is what I
  personally prefer for my production apps", for its nested-route helpers, regex parameters and
  track record. He'll still try `ServeMux` on new projects: if he has to swap in chi later,
  "historically this hasn't been a painful refactor".

Where that leaves a decision:

- **Start with `ServeMux`** when the routes and middleware are simple. There's nothing to update or
  scan, and it has the strictest route table of the three.
- **Move to chi** when you find yourself writing groups, sub-routers and per-group middleware by
  hand. Handlers keep their type and their `r.PathValue` calls, so the move is cheap.
- **Choose gin** when the team knows it well, or wants binding, validation and rendering wired in.
  Familiarity is a real reason: a team fluent in a framework makes fewer mistakes with it and
  reviews it faster. The price is a handler type nothing else accepts, middleware written for
  `http.Handler` that you can't use directly, and a 19-module tree with a `go 1.25` floor. Then set
  `ContextWithFallback = true` and `HandleMethodNotAllowed = true` where the engine is built.

> **Teacher's aside.** People compare routers on speed, because it's the number in every README,
> and on syntax, because it's what the examples show. The choice that lasts is the handler
> signature, because every handler and middleware you write is coupled to it.
> `func(http.ResponseWriter, *http.Request)` is accepted by `ServeMux`, chi and every router on
> Edwards' shortlist, so moving between them touches the route table and, at most, how handlers
> read parameters. `func(*gin.Context)` is accepted by gin alone, so leaving it means rewriting
> every handler and middleware. Choose the router for its features, knowing which of the two
> signatures you're committing the codebase to.

## Check yourself

1. A mux has `GET /items/{id}`, `DELETE /items/{id}` and `GET /items/latest`. What do
   `GET /items/latest` and `PUT /items/42` get back, and why doesn't registration order matter?
   What happens when someone then registers `/{kind}/latest`?
2. A service's `go.mod` still says `go 1.21`. Someone adds `mux.HandleFunc("GET /health", h)` and
   builds with Go 1.26. What does `GET /health` return, and why did nothing fail at compile time or
   at registration?
3. A gin handler runs `go audit(c)`, which logs `c.Param("id")` 10 ms later. The tests send one
   request at a time and always see the right id. What does it log in production, mechanically
   why, and what would a `-race` build show?
4. A gin engine is wrapped in `http.TimeoutHandler(engine, 2*time.Second, …)`, and every handler
   passes `c` to its database calls. A dependency hangs for 30 seconds. What does the client see,
   what are the queries doing, and what's the one-line fix?
5. A team that knows gin well is told to rewrite a service on `ServeMux` "for performance". It
   serves 2,000 requests a second, and each request makes two database queries. Using this
   chapter's measurements, what would the rewrite save, and what would it actually change?
6. After `ContextWithFallback = true` is set, goroutines written the way gin's docs recommend
   (`cp := c.Copy(); go publish(cp, event)`) start failing with `context canceled`. Explain why,
   and what the goroutine should use instead.
