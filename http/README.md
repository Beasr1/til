# HTTP Versions and Protocol Negotiation — A Course

A course on the layer beneath your handler: what an HTTP exchange is and what it
rides on, how a response is framed
on the wire, how the client and server agree on which version to speak, why
that layer can throw away a response your logs recorded as successful, and what
QUIC changes by moving streams down into the transport — plus the browser's rules
about which pages may read which responses, how it keeps a user signed in, and what a
cache does with a response it has stored.

**This is reference learning material.** Everything here is general HTTP, TLS and
QUIC, checked against the RFCs listed below. The motivating failure is
real and mundane: a server that logged `status=200`, a browser that showed
`ERR_HTTP2_PROTOCOL_ERROR`, and nothing anywhere naming the actual cause.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how — the problem that forced each mechanism comes before its name.
- Every file ends with **Check yourself** questions. Answers are in
  [10-exercises.md](10-exercises.md).
- Every protocol claim is quoted from the RFC, not recalled.

## The one thing to understand first

Almost everything surprising about HTTP/2 — why responses can be abandoned
mid-flight, why a browser and `curl` disagree about the same URL, why your access
log and your user disagree about whether a page loaded — follows from one change:

> **HTTP/1.1 responses are anonymous runs of bytes; HTTP/2 responses are
> identified streams of frames. Identity is what lets many responses share one
> connection — and it also creates a way to say "disregard that one" that
> HTTP/1.1 never had.**

The second fact sits underneath the first, and is where the debugging leverage
is:

> **Which version you're speaking was decided during the TLS handshake, by the
> server, from a list the client offered. You can change that list. No TLS means
> no negotiation, which is why plain HTTP behaves differently.**

## Reference implementations

| Source | What it settles |
|---|---|
| [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html) | §1.2 semantics split from each version's wire format, §3.3 statelessness, §3.6–3.7 origin servers and intermediaries, §8.8 validators (`ETag`, `Last-Modified`), weak and strong, and §8.8.3.2 how each comparison treats them, §9.2 safe and idempotent methods, §13.1 `If-Match`, `If-None-Match` and `If-Modified-Since`, §15 status-code classes, §15.4.5 what a `304` must carry and should leave out |
| [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html) | §3.2 how a cache updates stored header fields, and its four exceptions; §4 when a stored response may be reused; §4.2 freshness and §4.2.2 heuristic freshness; §4.3.4 freshening a stored response from a `304`; §5.2.2 `max-age`, `no-cache` and `no-store` |
| [RFC 8246 — HTTP Immutable Responses](https://www.rfc-editor.org/rfc/rfc8246.html) | §2, no conditional request during a response's freshness lifetime, not even on a reload |
| Browser source: [Chromium](https://github.com/chromium/chromium/blob/2b5d649769f63662fd563c9e41d9e5a2c65acd0a/net/http/http_response_headers.cc#L89-L109), [Firefox](https://github.com/mozilla-firefox/firefox/blob/e5b6db0c400c71559261e4ab18f0fbdb48d69ef0/netwerk/protocol/http/nsHttpResponseHead.cpp#L830-L876), [WebKit](https://github.com/WebKit/WebKit/blob/e427188d73fb165e49a4aade8c172b47ea6fbe02/Source/WebCore/platform/network/CacheValidation.cpp#L47-L94) | The header fields each engine refuses to take from a `304`. Everything else overwrites the stored copy |
| [nginx headers module](https://nginx.org/en/docs/http/ngx_http_headers_module.html#add_header) and [map module](https://nginx.org/en/docs/http/ngx_http_map_module.html) | `add_header` applies to `304` responses; the order in which `map` tries its values |
| [nginx CHANGES](https://nginx.org/en/CHANGES) (1.7.3) and Apache httpd [`DeflateAlterETag`](https://httpd.apache.org/docs/2.4/mod/mod_deflate.html#deflatealteretag) | What two compressing proxies do to a strong `ETag`: nginx makes it weak, Apache appends a suffix |
| [RFC 6454 — The Web Origin Concept](https://www.rfc-editor.org/rfc/rfc6454.html) | §3.2, an origin is scheme + host + port |
| [WHATWG Fetch Standard](https://fetch.spec.whatwg.org/#http-cors-protocol) | The CORS protocol: safelisted requests, preflight, the credentials table, `Vary: Origin`; §4.6 HTTP-network-or-cache fetch, where a `304` updates the stored response's headers |
| [RFC 6265 — HTTP State Management (cookies)](https://www.rfc-editor.org/rfc/rfc6265.html) | `Set-Cookie` and `Cookie`: how a stateless protocol gets sessions; §4.1.2.5, "secure" is "defined by the user agent" |
| [draft-ietf-httpbis-rfc6265bis](https://datatracker.ietf.org/doc/draft-ietf-httpbis-rfc6265bis/) | The cookie revision in the RFC Editor's queue: §5.7 ignores a `Secure` cookie from a connection the browser doesn't call secure, §5.8.3 leaves "secure" undefined, §4.1.3.2 the `__Host-` prefix |
| [W3C Fetch Metadata Request Headers](https://w3c.github.io/webappsec-fetch-metadata/) | §2.3, the four values of `Sec-Fetch-Site` |
| [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) | Fetch Metadata as "the primary signal for CSRF protection", and the `Origin` fallback it makes mandatory |
| [Go `net/http.CrossOriginProtection`](https://pkg.go.dev/net/http#CrossOriginProtection) | A Fetch Metadata check in a standard library (Go 1.25), and its fail-open answer for requests with neither header |
| [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) | What each session-cookie attribute is for; idle and absolute timeouts; invalidating the session on the server when the user signs out |
| [NIST SP 800-63B-4](https://pages.nist.gov/800-63-4/sp800-63b.html) | §2.1.3, §2.2.3 and §2.3.3, the overall and inactivity timeouts for each assurance level; what resets them |
| [OWASP HTML5 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html) | Why session identifiers don't belong in web storage |
| [RFC 7519 — JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html) | §1, what a JWT is; §4.1.4, `exp` |
| [RFC 6750 — Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html) | §5.3, no bearer tokens in cookies "that can be sent in the clear", nor in page URLs |
| [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html) | §4.14, refresh-token rotation, and how a stolen refresh token gives itself away |
| [RFC 10017 — OAuth 2.0 for Browser-Based Applications](https://www.rfc-editor.org/rfc/rfc10017.html) | §6, the patterns "in decreasing order of security", with the backend for frontend first |
| [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) | Session identifiers and access tokens among the data to exclude from logs |
| [RFC 6066 — TLS Extensions](https://www.rfc-editor.org/rfc/rfc6066.html) | §3, Server Name Indication |
| [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html) | Frames, streams, `END_STREAM`, `RST_STREAM`, the error-code table, the `h2` identifier |
| [RFC 7301 — ALPN](https://www.rfc-editor.org/rfc/rfc7301.html) | The TLS extension, who selects, and what happens with no overlap |
| [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html) | Message framing: `Content-Length`, chunked encoding, close-delimited bodies; §3.2 mandatory `Host`; §9.3 persistent connections by default |
| [W3C Mixed Content](https://w3c.github.io/webappsec-mixed-content/) | §4.4, the blocking algorithm, and the step that exempts a potentially trustworthy URL |
| [W3C Secure Contexts](https://w3c.github.io/webappsec-secure-contexts/) | §3.1, what makes an origin potentially trustworthy — including the loopback CIDR ranges; §5.2, the condition on trusting the name `localhost` |
| [W3C Content Security Policy Level 3](https://w3c.github.io/webappsec-csp/) | §2.2.2 a response's policies are read from its header fields, §6.1.3 `default-src` as the fallback for every fetch directive |
| [Chrome — Local Network Access](https://developer.chrome.com/blog/local-network-access) | The permission prompt: which version, what it covers, and what it does not yet gate |
| WebKit bugs [218980](https://bugs.webkit.org/show_bug.cgi?id=218980), [232088](https://bugs.webkit.org/show_bug.cgi?id=232088), [281149](https://bugs.webkit.org/show_bug.cgi?id=281149) | Loopback is a secure context in WebKit but not yet a secure connection for cookies, and the April 2026 commit that is only part of a fix |
| [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html) | §4.4.2 what the `Certificate` message carries, §4.4.3 what `CertificateVerify` signs and therefore proves |
| [RFC 9525 — Service Identity in TLS](https://www.rfc-editor.org/rfc/rfc9525.html) | §2 that Common Name MUST NOT identify a service, §6 the client's duty to match the name it asked for. Obsoletes RFC 6125 |
| [RFC 5280 — X.509 / PKIX](https://www.rfc-editor.org/rfc/rfc5280.html) | §6.1, the certification-path validation algorithm |
| [RFC 9000 — QUIC transport](https://www.rfc-editor.org/rfc/rfc9000.html) | §2 streams and their ordering rules, §4 flow control, §5.1 connection IDs, §7 the combined handshake, §8–9 address validation and migration, §13 loss confined to the streams in the lost packet |
| [RFC 9001 — Using TLS to secure QUIC](https://www.rfc-editor.org/rfc/rfc9001.html) | §3 TLS without the record layer, §4.2 TLS 1.3 minimum, §5.2 why Initial packets are readable, §8.1 mandatory ALPN, §9.2 0-RTT replay is the application's problem |
| [RFC 9002 — QUIC loss detection and congestion control](https://www.rfc-editor.org/rfc/rfc9002.html) | §4 how QUIC's recovery differs from TCP's, §7 the NewReno-like default and freedom to choose another |
| [RFC 8999 — Version-independent properties of QUIC](https://www.rfc-editor.org/rfc/rfc8999.html) | What every QUIC version keeps fixed on the wire |
| [RFC 9221 — Unreliable datagrams in QUIC](https://www.rfc-editor.org/rfc/rfc9221.html) | §2 motivation (real-time media, tunnels), §5 datagrams are congestion controlled but not retransmitted, flow controlled or fragmented |
| [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html) | §1 why HTTP left TCP, §3.1 the `h3` token, `Alt-Svc` discovery and TCP fallback, §6 how requests map onto QUIC streams, §8.1 error codes |
| [RFC 9204 — QPACK](https://www.rfc-editor.org/rfc/rfc9204.html) | Why HTTP/3 couldn't keep HPACK: header compression that limits cross-stream blocking |
| [RFC 9308 — Applicability of QUIC](https://www.rfc-editor.org/rfc/rfc9308.html) | §2 UDP blocking and fallback, §3 0-RTT and NAT timeouts, §4 designing a stream layout |
| [RFC 8470 — Using Early Data in HTTP](https://www.rfc-editor.org/rfc/rfc8470.html) | §4 which requests a client may send in 0-RTT, §5.2 `425 Too Early` |
| [RFC 9170 — Long-Term Viability of Protocol Extension Mechanisms](https://www.rfc-editor.org/rfc/rfc9170.html) | Appendix A.5, why TCP Fast Open couldn't deploy; §4.3, encryption as a defence against ossification |
| [RFC 9369 — QUIC Version 2](https://www.rfc-editor.org/rfc/rfc9369.html) | §1, a second QUIC version that exists to stop the first one ossifying |
| [RFC 9297 — HTTP Datagrams](https://www.rfc-editor.org/rfc/rfc9297.html) | Carrying QUIC datagrams through HTTP/3, tied to a request stream |
| [draft-ietf-moq-transport](https://datatracker.ietf.org/doc/draft-ietf-moq-transport/) | Media over QUIC Transport: an Internet-Draft, not an RFC, that carries media on both streams and datagrams |

## Reading order

The file numbers are the reading order.

### Part 1 — Foundations (prerequisite for everything else)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [**Foundations: what HTTP is, and what it rides on**](01-foundations.md) | ⭐ Read a request and a response, say what a method's safety and idempotency promise, read status classes, explain "one meaning, three wire formats", count the round trips before a first byte, and read `curl -v` line by line |

### Part 2 — The protocol on the wire

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Versions and framing](02-versions-and-framing.md) | Explain why a `200` in your log can still be a failed page, and read a stream reset |
| 3 | [How the version gets chosen](03-alpn-and-negotiation.md) | Control which HTTP version a server speaks, and know where that decision is actually made |
| 4 | [QUIC: a transport built on UDP](04-quic.md) | Explain what TCP can't fix and how QUIC gets round it, say what 0-RTT risks, choose between streams and datagrams for a protocol of your own, and map HTTP/3 back onto HTTP/2 |

### Part 3 — Trust, and what the browser allows

| # | File | After this you can… |
|---|------|---------------------|
| 5 | [Certificates and where trust comes from](05-certificates-and-trust.md) | Tell three different TLS failures apart by their error, and say where a trusted root actually lives |
| 6 | [Origins, the same-origin policy and CORS](06-origins-and-cors.md) | Say what an origin is and how it differs from a site, explain why the same-origin policy blocks reading but not sending, defend against CSRF, and debug a CORS or preflight failure |
| 7 | [Reaching localhost from a web page](07-loopback-and-local-network.md) | Explain why an HTTPS page may open `ws://127.0.0.1` with no certificate, tell a policy denial apart from a dead port, and say why a `Secure` cookie on `http://localhost` works in two browser engines and not the third |
| 9 | [Keeping a browser signed in: cookies, tokens and where they live](09-keeping-a-browser-signed-in.md) | Say what an injected script can take from each place a browser keeps a credential, set a session cookie's attributes and say what each buys, say what signing out can and can't end, set timeouts that don't take a user's unsaved work with them, and choose between a cookie, tokens in memory and a server that holds the tokens |

### Part 4 — Caching

| # | File | After this you can… |
|---|------|---------------------|
| 8 | [Caching, revalidation and what a 304 rewrites](08-caching-and-revalidation.md) | Say when a browser reuses a response without asking and when it revalidates, choose `Cache-Control` for pages and for hashed assets, explain why a header your server computes per response can blank a cached page on reload, and keep a compressing proxy from failing every `If-Match` save |

### Reference

| # | File | |
|---|------|---|
| 10 | [Exercises & answers](10-exercises.md) | Worked answers to every Check yourself, things to try, and questions for going deeper |

Read file 01 first. After it, Parts 2, 3 and 4 are independent of each other. Inside Part 2,
read in order: file 04 assumes file 02's streams and file 03's ALPN. Inside
Part 3, file 05 stands alone, and file 06 must come before file 07, which uses origins,
CORS preflights and CSRF without explaining them. File 09 also comes after file 06,
whose CSRF section it builds on. It sits in Part 3 although it was written after
Part 4, which is why its number is out of step. File 05 also pairs naturally with
file 03, which covers the handshake the certificate arrives in. File 08 needs only
file 01. Its incident turns on a `Content-Security-Policy` header, which file 07
introduces, but it explains what it uses.

File 01 teaches the TCP you need for this course. For more depth on sequence numbers,
retransmission, and flow and congestion control, see the
[cs-fundamentals networks chapter](../interview/cs-fundamentals/09-networks-layers-and-tcp.md).
File 04 is written about QUIC the transport, with HTTP/3 as one user of it, so it's also
the one to read if you're building something other than HTTP on QUIC.

Still missing: shared caches and CDNs in depth (`s-maxage`, `stale-while-revalidate`,
purging), stream prioritisation (HTTP/2's deprecated scheme and what replaced it), QUIC's
operational cost, and what network operators can still see. The questions at the end of
file 10 are the honest list.

## If you're short on time

- **New to HTTP, or lost in file 02:** file 01, all of it. It's the missing first
  chapter.
- **10 minutes:** file 02, the section "Two ways a stream ends".
- **Debugging a protocol error right now:** file 02's last two sections, then
  file 03's "Observing and controlling it".
- **Choosing a version for a service:** file 03, the table at the end.
- **A web page can't reach a program on localhost:** file 07, the CSP section —
  then its final diagram, which separates a synchronous throw from an async error.
- **A `Secure` cookie works on `http://localhost` in Chrome and Firefox but not
  Safari:** file 07, "A secure context doesn't mean `Secure` cookies work".
- **A page works on the first load and is blank on reload:** file 08, "What a cache
  does with a 304", then the ⚠️ in "The fix, and the rule it generalises to".
- **Saves fail with `412` in the browser but work with curl:** file 08, "When a proxy
  changes the body: tags for writes".
- **Deciding where a dashboard keeps its sign-in:** file 09, the table in "What an
  attacker gets from each place", then "How a session ends".
- **Choosing session timeouts without losing users' work:** file 09, "How long, and what
  an idle timeout does to unsaved work".
- **A TLS error you can't place:** file 05, the diagram in "The three questions,
  and the three errors" — it maps each error message to the check that produced it.
- **"Unknown authority" from a container:** file 05, "The trap: an empty
  filesystem trusts nothing".
- **A CORS error in the console:** file 06, "What CORS is not" (check the real status
  first), then "Preflight: asking before sending".
- **"Why HTTP/3?" in 15 minutes:** file 04, the table under "The problem", then
  "Ordered within, independent across" and its teacher's aside.
- **Building a real-time protocol on QUIC:** file 04, "Datagrams: when you don't
  want reliability", then "0-RTT, and the replay problem" before you turn 0-RTT
  on.

## The one-paragraph summary of everything

HTTP is one set of meanings — methods that promise to be safe or idempotent or
neither, three-digit status codes, header fields — defined once in RFC 9110 and
written onto the wire three different ways, carried over TLS and TCP (or QUIC), where
a fresh connection costs round trips before the first byte arrives. HTTP/1.1 sends one response at a time as a plain run of bytes whose end is
implied by `Content-Length`, chunked encoding, or the connection closing — so
two responses cannot be interleaved without becoming indistinguishable from one
corrupt response. HTTP/2 wraps every piece of data in a **frame** tagged with a
**stream** id, which makes interleaving trivial and one connection enough. That
identity brings a new ability: a stream can be ended cleanly with the
`END_STREAM` flag, or abandoned with a `RST_STREAM` frame carrying a reason such
as `CANCEL`. A server whose application layer builds a perfectly good `200` but
whose HTTP/2 layer then resets the stream will log success while the browser
discards the response and reports a protocol error — `curl` reports both the
status and the reset, which is what makes it the better tool here. Which version
you speak was settled earlier, during the TLS handshake: under **ALPN** the
client lists protocols in preference order and the server picks one, so removing
`h2` from the server's list silently moves everyone to HTTP/1.1, and dropping TLS
removes the negotiation altogether. Both are ways to avoid an HTTP/2 bug rather
than fix it, and the difference matters — the first keeps your local setup on the
same scheme as production, and the second hides every HTTPS-only bug along with
the one you were chasing. That same handshake carries the server's **certificate**,
which exists because a key exchange gives you a private channel to *whoever
answered* and says nothing about who that is. A client turns secrecy into identity
by asking three separate questions — does the far end hold the private key for
this certificate, does the name on it match the name I dialled, and does a chain
of signatures reach a root I already trusted? Each fails with its own error, which
is the fastest way to place a TLS problem. The third is the one that surprises
people, because "a root I already trusted" turns out to be a file at a hardcoded
path on the local filesystem, put there by the operating system and found by the
TLS library at runtime — so a statically linked binary in an empty image trusts
nothing at all, and reports every certificate on earth as signed by an unknown
authority. HTTP/2's streams still ride on TCP, which delivers one ordered byte
stream and so stalls every stream when any packet is lost. **QUIC** moves streams
into the transport, on top of UDP: loss stalls only the streams in the lost
packet, the TLS 1.3 handshake is folded into the transport's so a new connection
costs one round trip, a **connection ID** rather than the address-and-port
4-tuple identifies the connection so it survives a network change, and nearly
everything is encrypted so middleboxes can't freeze the protocol the way they
froze TCP. A returning client can send data with no round trip at all (0-RTT),
but that data can be replayed, so only requests that are safe to repeat belong
in it. Streams are reliable; an extension adds unreliable **datagrams** for data
that's worthless late, such as live audio. HTTP/3 is HTTP mapped onto all this,
negotiated with ALPN `h3`. Above all of this, the browser enforces a boundary of its
own: an **origin** (scheme, host and port), and a same-origin policy that stops a page
*reading* another origin's responses but not *sending* it requests. That gap is why
CSRF exists and why cookies grew a `SameSite` attribute. **CORS** is the server's way
to opt in to being read, with a **preflight** first for any request an HTML form
couldn't have sent. A signed-in browser carries a credential on every request, and
where it keeps it matters more than its format: an `HttpOnly` cookie is out of reach
of injected script but sent by the browser on its own, so it needs a CSRF defence,
while a token in web storage needs none and any script on the page can read it.
Ending a session before it expires takes state on the server, whatever the token
looks like. Responses can also be reused. `Cache-Control` says how long a
stored response stays **fresh**, usable without asking; after that, or before every
use under `no-cache`, the cache **revalidates** by sending the stored `ETag` in
`If-None-Match`, and a `304 Not Modified` says "yours is still good" without resending
the body. A 304 isn't only a yes, though: the cache copies the header fields it carries
over the stored ones. So a header the server computes per response, such as a
Content-Security-Policy chosen by `Content-Type`, which a 304 doesn't carry, can rewrite
the policy of a page the browser already holds. Send such a header the same on the 304,
or leave it off.

## How to use me

Ask me things like:

- "Walk me through what my server sends on the wire for this response."
- "This client reports a protocol error and my logs are clean — where do I look?"
- "Should this service speak HTTP/2? What am I actually buying?"
- "Show me the ALPN exchange for this host and explain what got chosen."
- "What does HTTP/3 change about all of this?"
- "Walk me through this `curl -v` output, layer by layer."
- "Why is my browser blocking this request with a CORS error — and is it really CORS?"
- "I'm designing a protocol on QUIC — how should I lay out streams, and what
  should go in datagrams?"
- "Is it safe to turn on 0-RTT for this endpoint?"
- "This TLS error — which of the three checks produced it, and where do I look?"
- "Show me what certificate chain this host actually sends, and what it leaves out."
- "Why does this certificate work from my laptop but not from the container?"
- "My page works on the first load and is blank on reload. What is the cache doing?"
- "What should this response's `Cache-Control` be, and what will a `304` for it
  overwrite?"
- "Why does Safari drop my `Secure` cookie on `http://localhost` when
  `isSecureContext` is true?"
- "Where should our dashboard keep its sign-in, and what does our sign-out actually
  end?"
