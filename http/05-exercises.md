# 5. Exercises

Worked answers to every **Check yourself** question, plus things to try.

---

## File 01 — What changed between HTTP/1.1 and HTTP/2

**1. Why can't HTTP/1.1 interleave two responses on one connection?**

Because nothing in the bytes says which response they belong to. A receiver
learns where a body ends from `Content-Length`, from chunked encoding's
terminator, or from the connection closing — all of which describe *one*
contiguous run of bytes. Insert bytes of another response into the middle and
the receiver has no way to detect it: it will hand the intruding bytes to the
first response's body, because as far as the framing is concerned that is what
they are. Interleaving isn't forbidden so much as unrepresentable.

**2. HEADERS, three DATA frames, then RST_STREAM(CANCEL). What must the receiver
do with the data it holds?**

Discard it. `RST_STREAM` withdraws the stream; the sender has stated it is not
finishing. The three DATA frames are therefore a prefix of an unknown whole —
and the receiver has no way to tell whether they're the first 3 KB of 3 KB or of
3 MB, because HTTP/2 signals completeness with `END_STREAM`, not by counting
bytes. Rendering a prefix as if complete is how you show a user a truncated
document that looks finished. Note this is exactly what HTTP/1.1's
close-delimited bodies get wrong, and part of why `Content-Length` matters there.

**3. curl shows 200 with a warning; Chrome shows only a protocol error.**

Both are consistent with the wire. The HEADERS frame genuinely carried `:status:
200`, so curl reports the status it received, and warns that the stream was
withdrawn afterwards — curl leaves the judgement to you, which suits a debugging
tool. A browser has to decide whether to render, and a withdrawn response cannot
be shown to a user as if it were complete, so it discards everything and reports
the protocol failure. The lesson is that a `200` in a client's output is not
proof the client *accepted* the response.

**4. Log says 200, browser says failed. Two layers, and a command.**

The two layers:

- **Below the application** — the HTTP/2 framing (the reset described in file
  01), or TLS, or the socket. The handler returned; the bytes never landed
  intact.
- **Above the server** — a proxy, CDN or service worker between server and
  browser, rewriting or failing the response after your log line was written.

`curl --http1.1 https://host/path` distinguishes them cheaply. If it succeeds
where HTTP/2 fails, the framing layer is implicated and everything above is
exonerated. If both fail identically, suspect the path above the server. Run it
from the same network as the browser, or you've changed two variables.

**5. Why is `REFUSED_STREAM` retryable when `CANCEL` and `INTERNAL_ERROR`
aren't?**

`REFUSED_STREAM` asserts the request was **never processed** — no handler ran, so
no side effects exist, so re-sending it cannot duplicate anything. `CANCEL` and
`INTERNAL_ERROR` make no such promise: the server may have processed the request
fully and failed only while delivering the response. Retry a cancelled `POST
/payments` and you may take payment twice. The error code is a statement about
side effects, not about how bad the failure was — which is why the retryable one
is defined in terms of processing rather than severity.

---

## File 02 — How the version gets chosen

**1. Why does ALPN cost no extra round trip?**

Because it rides inside a handshake that was happening regardless. The protocol
list goes in the ClientHello and the selection comes back in the ServerHello —
the same two flights that establish TLS. `Upgrade` runs *after* a connection is
established: a full HTTP/1.1 request and response must complete before the
switch, and that exchange exists only to ask the question.

**2. Client offers `["h2", "http/1.1"]`, server supports only HTTP/1.1.**

The server returns `http/1.1` in the ServerHello and the connection proceeds as
HTTP/1.1. No error — not a warning, not a downgrade notice. The client's list is
a preference in descending order, and the server picks the best it supports. This
silence is a feature: it's how a mixed fleet upgrades without coordination.

**3. Server offers only `["http/1.1"]`; client offers only `["h2"]`.**

The handshake fails. RFC 7301: where there is no overlap, "the server SHALL
respond with a fatal `no_application_protocol` alert". No TLS session is
established, so no HTTP of any version happens. Contrast with question 2 — the
difference is entirely whether the client kept `http/1.1` in its list.

**4. Disabling TLS to fix an HTTP/2 bug — what does it hide?**

Two classes:

- **Secure-context and transport behaviour** — `Secure` cookies not being sent,
  mixed-content blocking, HSTS, referrer policy differences, and anything
  certificate-related such as chain or SNI problems.
- **The bug itself** — it hasn't been diagnosed, only routed around. It will
  return the moment anything deploys over TLS, which is everywhere real.

Instead, keep TLS and remove `h2` from the server's ALPN list. That isolates
HTTP/2 as the variable while leaving the scheme matching production. It is still
avoidance rather than a fix, which is fine if you say so — the honest note is
"h2 disabled pending diagnosis", not "fixed".

**5. Browser reports HTTP/2, origin offers only `http/1.1`.**

TLS is being terminated in front of the origin — CDN, load balancer or reverse
proxy. The browser negotiates ALPN with *that*, which offers `h2`; the hop from
there to the origin is a separate connection with its own version, commonly
HTTP/1.1. To confirm, run `openssl s_client -alpn h2,http/1.1` against the public
hostname and against the origin directly and compare the selected protocol; a CDN
also usually names itself in a response header.

---

## Things to try

**Watch a real negotiation.** Pick any HTTPS site and run:

```
openssl s_client -connect example.com:443 -alpn h2,http/1.1 </dev/null 2>/dev/null | grep ALPN
```

Then again with `-alpn http/1.1` alone. The server's selection changes with your
offer — you are driving the choice from the client side.

**Make the failure happen on purpose.** Write a handler that sets a
`Content-Length` larger than the body it actually writes. Request it over
HTTP/1.1 and over HTTP/2 and compare what each client does. This is the cheapest
way to feel the difference between "the connection hangs waiting for bytes" and
"the stream is reset".

**Count connections.** Load a page with many images over HTTP/1.1 and over
HTTP/2, with the browser's network panel showing the connection id. Multiplexing
stops being abstract when you see thirty requests share one connection.


---

## File 03 — Reaching localhost from a web page

**1. Which two specification sections show a private CA is unnecessary?**

Mixed Content §4.4 and Secure Contexts §3.1, chained through a definition. The
blocking algorithm in Mixed Content §4.4 returns **allowed** at step 1.2 when
"request's URL is a potentially trustworthy URL". That term is not defined in
Mixed Content at all — it belongs to Secure Contexts §3.1, which enumerates what
qualifies and includes "origins matching the CIDR notations `127.0.0.0/8` or
`::1/128`". So the chain is: loopback → potentially trustworthy origin →
potentially trustworthy URL → the a priori authenticated exemption → not blocked.

The reason people miss it is that neither document answers the question alone.
Mixed Content looks like it should, and the actual carve-out lives one
cross-reference away in a spec about a different subject.

**2. The program binds `0.0.0.0` and is reached by `192.168.x.x`.**

Mixed content blocking now applies, and the request is refused. `127.0.0.0/8` and
`::1/128` are the only loopback ranges the exemption names; a private LAN address
is an ordinary network address for this purpose, so `http://192.168.1.5` from an
HTTPS page is blockable content like any other.

Worth noticing what changed and what didn't. The program is identical, the port is
identical, the page's code may be identical. The *route* changed — the bytes now
traverse a network that an attacker could sit on, which is precisely the exposure
the exemption was reasoning about. Binding `0.0.0.0` also means the service is
reachable from other machines at all, which is usually an unintended consequence
of the same decision.

**3. "The page can't see the local service" — what to ask for?**

Ask whether the `WebSocket` constructor **threw**, or whether an `error`/`close`
event fired later. That single observable splits the causes cleanly: a
synchronous throw is a CSP `connect-src` denial, because the connection was never
attempted; anything asynchronous means the connection was attempted and failed —
nothing listening, wrong port, service not installed, handshake refused.

In practice the useful form of the question is "open the console and paste what
`new WebSocket('ws://127.0.0.1:PORT')` does when you type it directly", because
that also bypasses whatever the application's own error handling has done to the
evidence.

**4. Why did Private Network Access's preflight design fail?**

It required the *local device* to opt in by answering a CORS preflight, which
means the fix had to be deployed to the least-maintained software in the system:
embedded HTTP servers in routers, printers and appliances, much of it shipped
years earlier and some of it unmodifiable. A permission model whose rollout
depends on that population upgrading cannot complete.

The general principle: **place an opt-in with a party who is present, capable of
acting, and has an incentive to.** LNA moves the decision to the browser and the
user — both present at the moment of the request, and the browser ships updates
on its own schedule. Whenever a security mechanism requires coordinated action by
a long tail of unmaintained software, expect it to stall regardless of technical
merit.

**5. What does the try/catch wrapper destroy?**

The only distinction between a policy denial and a connection failure. Once both
paths return `null`, a CSP misconfiguration and an uninstalled local program are
indistinguishable to every caller and to every log line.

The class of bug this makes unfalsifiable is a *header* problem reported as a
*hardware* problem. The person debugging has no way to tell that the browser
refused before touching the socket, so every hypothesis points at the local
program: is it running, is it the right port, is the device plugged in. The
evidence that would have ended the investigation in one line was discarded inside
a helper written to make the API tidier.

**6. A page that must reach loopback but is embedded where you don't control CSP.**

Technically, nothing. `connect-src` is enforced from the response headers of the
document doing the connecting; a script cannot loosen the policy it runs under,
and there is no negotiation. Your options are all social: document the required
`connect-src` entry, detect the synchronous throw and report a specific,
actionable error naming the directive, and fail loudly enough that the customer's
web team is the one who reads it.

What that tells you is where the constraint actually lives: **this integration's
hardest dependency is a header owned by someone who is not in the room.** That is
worth surfacing at design time rather than discovering at deployment, and it is a
good argument for making the required policy part of the integration's published
prerequisites rather than a troubleshooting note.

## Things to try — file 03

**Watch the two failure modes side by side.** Serve a page over HTTPS with
`connect-src 'self'` and call `new WebSocket('ws://127.0.0.1:9')` in the console
— port 9 has nothing on it. You get the throw. Now add
`ws://127.0.0.1:9` to the directive and repeat: the constructor returns, and the
failure arrives as an event instead. Same dead port, two entirely different
shapes of failure.

**Confirm the loopback exemption yourself.** From an HTTPS page, `fetch` a plain
`http://127.0.0.1:PORT` endpoint, then the same service by its LAN IP. The first
succeeds and the second is blocked as mixed content — the clearest demonstration
that the carve-out is about the address, not the scheme.

**Check whether the LNA note above is still true.** Open the Chrome blog post and
look for whether WebSockets have become gated. This chapter's most perishable
claim is exactly the sort of thing to verify rather than inherit.

---

## File 04 — Certificates, and where trust comes from

**1. "The connection is encrypted, so it's secure." Encrypted against whom?**

Against passive observers on the path — anyone merely reading the wire learns
nothing. What it does not give you is any assurance about *who is on the other
end*. A key exchange works with whoever answers, so an attacker who intercepts the
connection and performs the exchange themselves gets a channel just as private as
the one you wanted, with themselves at the far end. Secrecy and identity are
separate properties; encryption delivers the first and certificates the second.

**2. You serve a copy of someone else's certificate. What happens?**

Check 1 catches you, at the `CertificateVerify` step. The certificate is public
and copying it is trivial, but RFC 8446 §4.4.3 requires a signature "over the
entire handshake using the private key corresponding to the public key in the
Certificate message" — and you don't have that private key. You cannot forge the
signature, and you cannot replay a recorded one because the handshake it covers
includes fresh random values from both sides. The name check (2) and the chain
check (3) would both have passed; it is possession of the key that stops you.

**3. CN matches, in date, still a name mismatch.**

The name is in the Common Name only, and not in the Subject Alternative Name
extension. RFC 9525 §2 settles it: "The Common Name RDN MUST NOT be used to
identify a service because it is not strongly typed (it is essentially free-form
text) and therefore suffers from ambiguities in interpretation." Check the SAN
list. The confusion is worth expecting, because RFC 9525 (2023) obsoletes RFC
6125, which did permit CN as a fallback — so older tools and older documentation
genuinely disagree with current clients, and both are describing a real standard.

**4. Fixing the name error revealed an unknown-authority error.**

Because the three checks are ordered and the client stops at the first failure.
While the name didn't match, validation never got as far as building a chain, so
the missing root was invisible. Correcting the name let check 2 pass, and
validation proceeded to check 3 — which had been broken the whole time.

This is the general shape of layered validation and it is worth internalising:
**fixing one error does not reveal a new problem, it reveals an older one.** The
second error was always there. An error message tells you where validation stopped,
not how many things are wrong — so a sequence of different errors from one endpoint
is usually progress, not a cascade of new faults.

**5. Six months of happy traffic, then the first outbound HTTPS call fails.**

The missing trust store is inert until something actually needs to validate a
certificate. It causes no build error (nothing references the file), no startup
error (the root pool is read lazily, on first use), and no error for inbound TLS
(where the server presents a certificate rather than verifying one) or for any
plaintext call. Services that talk to their neighbours over plain HTTP inside a
trusted network therefore never exercise it.

So the delay is structural, not luck: the defect is dormant by construction and
is triggered by the first outbound TLS connection, whenever that happens to
arrive. The trap is that it then looks like a problem with whatever host was
finally dialled — the thing that changed is the most suspicious-looking
explanation and the wrong one.

**6. `SSL_CERT_FILE` pointed at one private root.**

It is worse than doing nothing because it *replaces* the search list rather than
adding to it. Go tries its `certFiles` paths in order and uses the first that
exists; `SSL_CERT_FILE` overrides that entirely. So you have gained one private
root and silently lost all hundred-odd public ones — connections that worked
before now fail, and the variable you set to fix trust is what broke it.

Two ways to get what you wanted:

- Concatenate the private root onto a copy of the public bundle and point
  `SSL_CERT_FILE` at the combined file.
- Use `SSL_CERT_DIR` instead, which names directories to search rather than a
  single replacement file, so the private root sits alongside the system set.

---

## Things to try — file 04

**Read a real chain.** Pick any HTTPS host and look at what the server actually
sends:

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -ext subjectAltName
```

Note that the issuer is an intermediate, not a root — then look for where the root
itself is on your machine. It was never sent.

**Prove the trust store is just a file.** Point `SSL_CERT_FILE` at `/dev/null` for
one command and watch every HTTPS call fail with an unknown-authority error, on a
machine where nothing is wrong. Then unset it. This is the container failure
reproduced in one line, and it is worth feeling once.

**Find your language's list.** Locate the equivalent of Go's `certFiles` in
whatever you use most. The point is to see that it is a hardcoded catalogue of
distribution conventions rather than anything a standard specifies — and therefore
that two programs on one machine can legitimately disagree about trust.

---

## Questions worth asking me

- Where does HTTP/3 change this? QUIC moves framing below HTTP and gives each
  stream its own delivery guarantees — what problem with HTTP/2 was that solving?
- What is head-of-line blocking, and why does HTTP/2 only solve half of it?
- How do flow control and stream priority interact with multiplexing — and why
  did the RFC 9113 revision deprecate the original priority scheme?
- When is HTTP/2 the wrong choice? Where does one connection carrying everything
  become a liability rather than a saving?
- What is the threat model for a program listening on loopback? Any local process
  can reach it, so what actually authenticates the caller — and does anything in
  the browser help?
- How do browser extensions and native messaging compare with a loopback socket
  for reaching local hardware, and why has that trade-off moved?

- Certificates expire and can be revoked before they expire — how does a client
  learn about revocation, and why is that the weakest part of the whole design?
- What changes when *both* ends present certificates, and what problem does that
  solve that a bearer token doesn't?
- What is certificate pinning, what attack does it stop that a trust store
  doesn't, and why has it fallen out of favour on the web?
- Who decides which roots are in a public bundle, on what evidence, and what
  happens to everyone when one of them misbehaves?
