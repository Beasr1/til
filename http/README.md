# HTTP Versions and Protocol Negotiation — A Course

A short course on the layer beneath your handler: how an HTTP response is framed
on the wire, how the client and server agree on which version to speak, and why
that layer can throw away a response your logs recorded as successful.

**This is reference learning material.** Everything here is general HTTP and TLS,
checked against RFC 9113 (HTTP/2) and RFC 7301 (ALPN). The motivating failure is
real and mundane: a server that logged `status=200`, a browser that showed
`ERR_HTTP2_PROTOCOL_ERROR`, and nothing anywhere naming the actual cause.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how — the problem that forced each mechanism comes before its name.
- Every file ends with **Check yourself** questions. Answers are in
  [05-exercises.md](05-exercises.md).
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
| [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html) | Frames, streams, `END_STREAM`, `RST_STREAM`, the error-code table, the `h2` identifier |
| [RFC 7301 — ALPN](https://www.rfc-editor.org/rfc/rfc7301.html) | The TLS extension, who selects, and what happens with no overlap |
| [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html) | Message framing: `Content-Length`, chunked encoding, close-delimited bodies |
| [W3C Mixed Content](https://w3c.github.io/webappsec-mixed-content/) | §4.4, the blocking algorithm, and the step that exempts a potentially trustworthy URL |
| [W3C Secure Contexts](https://w3c.github.io/webappsec-secure-contexts/) | §3.1, what makes an origin potentially trustworthy — including the loopback CIDR ranges |
| [Chrome — Local Network Access](https://developer.chrome.com/blog/local-network-access) | The permission prompt: which version, what it covers, and what it does not yet gate |
| [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html) | §4.4.2 what the `Certificate` message carries, §4.4.3 what `CertificateVerify` signs and therefore proves |
| [RFC 9525 — Service Identity in TLS](https://www.rfc-editor.org/rfc/rfc9525.html) | §2 that Common Name MUST NOT identify a service, §6 the client's duty to match the name it asked for. Obsoletes RFC 6125 |
| [RFC 5280 — X.509 / PKIX](https://www.rfc-editor.org/rfc/rfc5280.html) | §6.1, the certification-path validation algorithm |

## Reading order

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Versions and framing](01-versions-and-framing.md) | Explain why a `200` in your log can still be a failed page, and read a stream reset |
| 2 | [How the version gets chosen](02-alpn-and-negotiation.md) | Control which HTTP version a server speaks, and know where that decision is actually made |
| 3 | [Reaching localhost from a web page](03-loopback-and-local-network.md) | Explain why an HTTPS page may open `ws://127.0.0.1` with no certificate, and tell a policy denial apart from a dead port |
| 4 | [Certificates and where trust comes from](04-certificates-and-trust.md) | Tell three different TLS failures apart by their error, and say where a trusted root actually lives |
| 5 | [Exercises & answers](05-exercises.md) | Check the model formed, and go deeper |

Four chapters for now. Files 03 and 04 are both independent of 01–02: file 03 is
about what the browser permits a page to connect *to* rather than how a response
is framed, and file 04 is about what TLS proves rather than what gets negotiated
inside it. Read either on its own if that's the question in front of you. File 04
pairs naturally with file 02, which covers the handshake the certificate arrives
in.

HTTP/3 and QUIC, head-of-line blocking and flow control all belong in this course
and aren't written — the questions at the end of file 05 are the honest list of
what's missing.

## If you're short on time

- **10 minutes:** file 01, the section "Two ways a stream ends".
- **Debugging a protocol error right now:** file 01's last two sections, then
  file 02's "Observing and controlling it".
- **Choosing a version for a service:** file 02, the table at the end.
- **A web page can't reach a program on localhost:** file 03, the CSP section —
  then its final diagram, which separates a synchronous throw from an async error.
- **A TLS error you can't place:** file 04, the diagram in "The three questions,
  and the three errors" — it maps each error message to the check that produced it.
- **"Unknown authority" from a container:** file 04, "The trap: an empty
  filesystem trusts nothing".

## The one-paragraph summary of everything

HTTP/1.1 sends one response at a time as a plain run of bytes whose end is
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
authority.

## How to use me

Ask me things like:

- "Walk me through what my server sends on the wire for this response."
- "This client reports a protocol error and my logs are clean — where do I look?"
- "Should this service speak HTTP/2? What am I actually buying?"
- "Show me the ALPN exchange for this host and explain what got chosen."
- "What does HTTP/3 change about all of this?"
- "This TLS error — which of the three checks produced it, and where do I look?"
- "Show me what certificate chain this host actually sends, and what it leaves out."
- "Why does this certificate work from my laptop but not from the container?"
