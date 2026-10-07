# 9. API design

## The problem

An API is the one part of a system you can't quietly fix later. Change a database schema
and you run a migration; change an API that a mobile app shipped against two years ago and
the app breaks on phones you can't update. So the decisions here, which style, how lists
are paged, what happens when a request is retried, how versions change, are cheap to get
right on day one and expensive afterwards. In the interview, the API step
([file 01 §1.4](01-foundations.md#14-a-working-method)) is where you show you've thought about
the client, not just the servers.

## 9.1 REST, gRPC or GraphQL

They solve different problems, which is the useful way to compare them.

**REST** is Roy Fielding's name, from his 2000 dissertation, for the architectural style of
the web. Its defining feature, in his words: "The central feature that distinguishes the
REST architectural style from other network-based styles is its emphasis on a uniform
interface between components" ([chapter 5](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)).
In practice, "REST API" means resources at URLs (`/orders/42`), HTTP methods as verbs, JSON
bodies, and HTTP status codes, with semantics defined by RFC 9110.

**gRPC** is remote procedure calls with a schema. "By default, gRPC uses protocol buffers as
the Interface Definition Language (IDL) for describing both the service interface and the
structure of the payload messages" ([gRPC core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)),
carried over HTTP/2 ([protocol spec](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)).
It supports four call shapes: unary, server streaming, client streaming, and bidirectional
streaming.

**GraphQL** lets the client send a query describing exactly the fields it wants, across
related objects, in one request. It's specified by the GraphQL Foundation; the current
edition is [September 2025](https://spec.graphql.org/September2025/).

| | REST + JSON | gRPC | GraphQL |
|---|---|---|---|
| Contract | Convention, optionally OpenAPI | `.proto` schema, code generated | Schema, typed |
| Payload | JSON text | Protobuf binary: smaller, faster to parse | JSON |
| Transport | HTTP/1.1 or 2 | HTTP/2 | Usually HTTP POST |
| Streaming | Awkward (SSE, WebSockets on the side) | Built in, all four shapes | Subscriptions, implementation-specific |
| Browsers | Native | Need gRPC-Web and a proxy | Native |
| HTTP caching | Works: GETs on URLs | No | Hard: everything's a POST to one URL |
| Best at | Public APIs, simple CRUD, anything cacheable | Internal service-to-service, polyglot, low latency | Many client types needing different slices of a rich data graph |

Two details that come up as follow-ups:

- **gRPC in browsers.** gRPC-Web exists because browsers don't expose HTTP/2 framing: "Due to
  browser limitation, the Web client library implements a different protocol than the
  native gRPC protocol" ([spec](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md)),
  and clients "connect to gRPC services via a special proxy" (Envoy by default). Client and
  bidirectional streaming aren't supported there. So the common shape is REST or GraphQL at
  the edge, gRPC inside.
- **GraphQL's N+1 problem.** A query for 50 posts and each post's author can naively make 1
  query for the posts and 50 for authors. The standard fix is batching per request, as in
  [DataLoader](https://github.com/graphql/dataloader), which "will coalesce all individual
  loads which occur within a single frame of execution… and then call your batch function
  with all requested keys." (The README doesn't use the name "N+1"; that's general usage.)

The interview default: **REST for the external API, and say gRPC for internal calls if
latency or streaming matters.** Pick GraphQL when the requirements include many clients with
different data needs, and be ready to talk about caching and query cost.

## 9.2 Resources, methods and status codes

For a REST API, the method carries meaning that infrastructure relies on. RFC 9110 defines
two properties ([§9.2](https://www.rfc-editor.org/rfc/rfc9110#section-9.2)):

- **Safe**: "essentially read-only": `GET`, `HEAD`, `OPTIONS`.
- **Idempotent**: "the intended effect on the server of multiple identical requests with
  that method is the same as the effect for a single such request." `PUT`, `DELETE`, and the
  safe methods are idempotent. `POST` and `PATCH` are not.

| Operation | Method and path | Success |
|---|---|---|
| Create an order | `POST /orders` | `201 Created` with a `Location` header |
| Read one | `GET /orders/42` | `200` |
| Replace | `PUT /orders/42` | `200` or `204` |
| Partial update | `PATCH /orders/42` | `200` |
| Delete | `DELETE /orders/42` | `204` |
| List | `GET /users/7/orders?limit=20&cursor=…` | `200` |

For error bodies, RFC 9457, *Problem Details for HTTP APIs* ([link](https://www.rfc-editor.org/rfc/rfc9457)),
defines a standard JSON shape ("a 'problem detail' to carry machine-readable details of
errors"), so you don't invent one per service.

## 9.3 Pagination

Any endpoint that returns a list needs paging, because lists grow. Two approaches:

| | Offset (`?page=3&size=20`) | Cursor / keyset (`?after=<opaque token>`) |
|---|---|---|
| Query | `LIMIT 20 OFFSET 40` | `WHERE (created_at, id) < ($1, $2) ORDER BY … LIMIT 20` |
| Cost of page 1,000 | Reads and discards 20,000 rows | Same as page 1, with an index |
| New rows inserted while paging | Items shift: you skip or repeat some | Stable |
| Jump to page N | Yes | No; next/previous only |

Cursor pagination is the right default for feeds and anything large or changing. The
Postgres course has the SQL, including why the sort must be unique (add the primary key as a
tiebreaker) and how offsets move under you:
[postgres/05, "Keyset pagination"](../../postgres/05-pagination.md#keyset-pagination) and
[postgres/05, "`OFFSET` also moves when the data moves"](../../postgres/05-pagination.md#offset-also-moves-when-the-data-moves).

Make the cursor **opaque** to clients (base64 of the last row's sort key, say), so you can
change what's inside it. GraphQL APIs commonly follow the
[Relay Cursor Connections spec](https://relay.dev/graphql/connections.htm), which
standardises `edges`, `cursor`, and `pageInfo { hasNextPage, endCursor }` with
`first`/`after` arguments.

## 9.4 Idempotency and idempotency keys

A mobile client sends `POST /payments`. The network drops the response. Did the payment
happen? The client can't know. If it retries, the user may pay twice; if it doesn't, the
user may not have paid. RFC 9110 is explicit that a client "SHOULD NOT automatically retry a
request with a non-idempotent method unless it has some means to know that the request
semantics are actually idempotent".

An **idempotency key** is that means. The client generates a unique key (a UUID) per
*logical* operation and sends it with every attempt. The server records the key with the
result of the first attempt; a repeat with the same key gets the stored result instead of
running again.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant D as Store
    C->>S: POST /payments  Idempotency-Key: k1
    S->>D: insert key k1 (unique) → ok
    S->>S: charge card
    S->>D: save response for k1
    S--xC: 201 (response lost)
    C->>S: retry, Idempotency-Key: k1
    S->>D: insert k1 → conflict; load saved response
    S->>C: 201 (same response, no second charge)
```

Stripe's [idempotent requests](https://docs.stripe.com/api/idempotent_requests) are the
widely copied design. Details worth borrowing from its (v1) documentation:

- It saves "the resulting status code and body of the first request made for any given
  idempotency key, regardless of whether it succeeds or fails. Subsequent requests with the
  same key return the same result, including 500 errors."
- It "compares incoming parameters to those of the original request and errors if they're
  not the same", so a key can't be accidentally reused for a different payment.
- Keys can be removed after they're at least 24 hours old. (Stripe's newer v2 API uses a
  30-day window, so the right retention depends on how long clients may retry.)

The hard part is the window between "insert key" and "save response": two concurrent
requests with one key, or a crash in the middle. A unique constraint on the key makes the
insert the arbiter; Brandur Leach's
[*Implementing Stripe-like Idempotency Keys in Postgres*](https://brandur.org/idempotency-keys)
goes through recovery points in detail. The Postgres primitive underneath is
[postgres/06, "The shape"](../../postgres/06-upserts-and-duplicate-events.md#the-shape).

There was an IETF draft to standardise an `Idempotency-Key` header; as of October 2026 it
has expired without becoming an RFC. The header name is a widespread convention, not a
standard.

> **Teacher's aside.** Idempotency keys are usually explained as protection against client
> retries. Their bigger value is that they make *every* retry in the system safe: the
> client's, the gateway's, your own outbound retry to a payment provider
> ([file 08 §8.3](08-reliability-patterns.md#83-retries-backoff-and-jitter)), and redelivery
> from a queue ([file 07 §7.4](07-asynchronous-work.md#74-idempotent-consumers)). Once
> "creating a payment" is idempotent by key, the question "is it safe to retry?" has one
> answer everywhere, and that's what makes the reliability patterns usable on writes.

## 9.5 Versioning

APIs change. The question is how old clients keep working.

| Approach | Example | Notes |
|---|---|---|
| **Path** | `/v1/orders`, `/v2/orders` | Obvious, easy to route; coarse: a whole new version for one change |
| **Header** | `Accept: application/vnd.example.v2+json` | Clean URLs; harder to test in a browser |
| **Date-pinned** | Stripe: an account is pinned to the API version current at its first request | Fine-grained; needs a compatibility layer |
| **Additive only** | Never remove or change fields; only add | Simplest when you can stick to it. Protobuf's field numbering is built for this |

Stripe's version is the interesting one to know. Brandur Leach's
[*APIs as infrastructure*](https://stripe.com/blog/api-versioning) (2017): versions are
"named with the date they're released", and "the first time a user makes an API request,
their account is automatically pinned to the most recent version available". Internally,
the latest code runs and a chain of version-change modules transforms responses "backwards
from the current API version" for older clients.

What counts as breaking: removing or renaming a field, changing a type, making an optional
field required, changing an error code clients branch on. Adding an optional field or a new
endpoint usually isn't, *if* clients are written to ignore unknown fields. Say that in the
interview: compatibility is a property of both sides.

## Asked in interviews

| Prompt | Where |
|---|---|
| "Design the API for this." | §9.2, and the API step in every [file 10](10-worked-designs.md) design |
| "REST or gRPC?" "When would you use GraphQL?" | §9.1 |
| "How do you paginate a feed?" "What's wrong with offset pagination?" | §9.3 |
| "The client retries a payment. How do you avoid charging twice?" | §9.4 |
| "Which HTTP methods are idempotent?" | §9.2 |
| "How do you version the API?" "What's a breaking change?" | §9.5 |

## Check yourself

1. A browser app needs live order updates from a gRPC service that uses bidirectional
   streaming. What stops you calling it directly from the browser, and what would you put
   in between or use instead?
2. `GET /feed?page=2` shows a post the user already saw on page 1. Explain the mechanism,
   and what the cursor version of this endpoint returns instead.
3. A client sends `POST /payments` with idempotency key k1 and ₹500, times out, then retries
   with key k1 and ₹600. What should the server do, and why is that better than either
   charging ₹600 or silently returning the ₹500 result?
4. Two requests with the same idempotency key arrive 5 ms apart, before either finishes.
   Walk through what happens with a unique constraint on the key, and what the second
   request should receive.
5. Your mobile app's v1 reads `order.total` as an integer number of rupees. You need paise
   precision. List two ways to ship it without breaking installed v1 apps.
6. Why is HTTP caching (a CDN, a browser cache) natural for a REST API and hard for
   GraphQL? What do GraphQL deployments do about it?
