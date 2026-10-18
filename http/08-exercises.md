# 8. Exercises

Worked answers to every **Check yourself** question, plus things to try.

---

## File 01 — Foundations: what HTTP is, and what it rides on

**1. A `PUT` timed out. Safe to resend? And a `POST`?**

Yes for `PUT`. It's idempotent: "the intended effect on the server of multiple
identical requests … is the same as the effect for a single such request". If the
first one reached the server, the second replaces the resource with the same thing
again. A `POST` makes no such promise. If the first one created an order and only
the response was lost, a retry creates a second order. The property that decides it
is idempotency, not safety: `PUT` isn't safe (it changes state) but it is
idempotent. The usual fix for `POST` is an idempotency key the server uses to spot
the duplicate, which turns the retry into a lookup.

**2. `502` from behind a load balancer. Who generated it?**

The load balancer. `502 Bad Gateway` is an intermediary saying it couldn't get a
valid response from the server behind it. Your server didn't send the `502`. It
either refused the connection, closed it early, sent something malformed, or
wasn't there. That points you at your server's health, its connection handling, or
the protocol between the two, which may not be the one the browser negotiated
(file 03). Check the load balancer's logs for the upstream error before you search
your own access log for a request it may never have received.

**3. Fresh HTTPS over TCP and TLS 1.3, 150 ms away.**

Three round trips: TCP handshake, TLS 1.3 handshake, then the request and the
response. That's about 450 ms to the first byte, ignoring DNS and server think
time. A reused connection skips both handshakes, so the request costs one round
trip, about 150 ms, saving roughly 300 ms. Bandwidth can't help because nothing
in those 300 ms is waiting for *data*. Each step waits for a reply to cross the
distance and come back. Round-trip time is set by distance and the speed of light
in fibre, and a fatter pipe doesn't make the light go faster.

**4. The same `GET` over HTTP/1.1 and HTTP/2.**

Identical, and owned by RFC 9110 (semantics): the method `GET`, the target
`/docs/intro`, the meaning of the status code and of each header, for example
`Content-Type: text/html`. Different, and owned by the version's own RFC: the wire
encoding, text lines in RFC 9112 versus binary frames in RFC 9113; and where the
host goes, a `Host` header in HTTP/1.1 versus the `:authority` pseudo-header in
HTTP/2. Others you could name: HTTP/2 compresses headers (HPACK) and carries the
request on a numbered stream; HTTP/1.1 ends the body by `Content-Length` or
chunking, HTTP/2 by `END_STREAM`.

**5. Why is `Host` mandatory?**

Because one IP address commonly serves many sites, and the TCP connection only
tells the server which IP and port the client dialled. Without `Host`, a server
with 500 sites on one address receives `GET /index.html` and has no way to tell
which site's `/index.html` you mean. It could only serve one default site. TLS has
the same problem one layer down, and earlier: the server has to choose which
certificate to present *before* any HTTP is exchanged. **SNI**, in the
ClientHello, carries the hostname for that purpose.

**6. "We're behind a CDN, so our origin is the CDN."**

They've mixed up the two meanings. In RFC 9110's sense, the **origin server** is
the program that produces authoritative responses: yours, behind the CDN. The CDN
is an intermediary, specifically a caching reverse proxy. The *other* "origin" is
the web origin, the `(scheme, host, port)` of the URL in the address bar, for
example `https://www.example.com`, which the browser uses for its security rules.
Browsers never see your origin server's address because they never connect to it.
They connect to the CDN using the hostname in the URL, so every same-origin and
CORS decision is made on that URL's scheme, host and port. Where the bytes really
came from is invisible to them, and irrelevant.

## Things to try — file 01

1. **Count the round trips yourself.** Run
   `curl -so /dev/null -w 'dns %{time_namelookup}\ntcp %{time_connect}\ntls %{time_appconnect}\nfirst byte %{time_starttransfer}\n' https://example.com`.
   The gaps between successive numbers are roughly one round trip each. Run it
   twice and compare with a host on another continent.
2. **Watch reuse.** `curl -v https://example.com https://example.com` fetches twice
   on one connection. Look for "Re-using existing connection" and note the
   handshake lines appear only once.
3. **Speak HTTP by hand.** `printf 'GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' | openssl s_client -quiet -connect example.com:443 -servername example.com`.
   Then delete the `Host` line and see what the server does with a request that
   doesn't say which site it wants.
4. **See one request three ways.** Fetch the same URL with `--http1.1`, `--http2`
   and, if your curl supports it, `--http3`. Compare the `>` lines.

---

## File 02 — What changed between HTTP/1.1 and HTTP/2

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

## File 03 — How the version gets chosen

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

## Things to try — files 02 and 03

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

## File 04 — QUIC: a transport built on UDP

**1. Thirty images, one lost packet carrying part of image 7.**

Over HTTP/2, the receiving TCP stack gets the later packets, including the ones
carrying images 8 and 9, and holds them in its buffer. It can't pass them up,
because TCP delivers one byte stream in order and there's a gap before them.
HTTP/2 never sees those bytes until the retransmission of the lost packet
arrives, at least one round trip later. Every image stalls, including ones that
had nothing in the lost packet. TCP doesn't know they're separate.

Over HTTP/3, the QUIC receiver places each `STREAM` frame by its stream ID and
offset. Images 8 and 9 are on their own streams with no gap, so their data goes
straight to the application. Only image 7's stream waits.

Image 8 stalls too if **its data was in the same lost packet**. A QUIC packet
can carry frames from several streams, and RFC 9000 §13 says loss of such a
packet "blocks all those streams from making progress." The other
circumstance is above the transport: if decoding image 8's headers depends on
QPACK state that arrives on another stream, it can wait on that. QPACK exists to
limit this, and RFC 9114 §4.2.1 says it gives the encoder "some control over how
much head-of-line blocking can be caused by compression", which means some is
still allowed.

**2. Why per-stream congestion control would be a mistake.**

Congestion control protects the path through the network, and every stream on
a connection uses the same path. Give each stream its own congestion window and
a connection with twenty streams pushes twenty fair shares into a bottleneck
that should get one. When loss appears, each stream's controller backs off for
itself, and the connection as a whole is still far too aggressive.

Flow control is different because it protects the *receiver's* memory, which
really is split between streams. One stream filling the buffer is a real
problem, so per-stream limits make sense there. Hence RFC 9114 §1.2's summary:
"reliability at the stream level and congestion control across the entire
connection."

The earlier mistake was HTTP/1.1's habit of opening several TCP connections to
one server to get parallelism. RFC 9114 §1.1: that "has a negative impact on
congestion control and network efficiency, since TCP does not share congestion
control across multiple connections." HTTP/2 fixed it by using one connection,
and QUIC keeps the fix while removing the head-of-line cost that came with it.

**3. Wi-Fi to mobile mid-download, then a 4-tuple load balancer.**

Over TCP+TLS, the phone's new address means a new 4-tuple, which TCP treats as a
different connection. The old one is dead; packets from the new address
belong to nothing. The client must do a new TCP handshake and a new TLS
handshake, and the download resumes only if the application retries (with a
`Range` request, if the server supports it) or restarts from scratch.

Over QUIC, the client keeps sending from the new address, switching to a fresh
connection ID it was given earlier so the two networks can't be linked. The
server finds the connection by connection ID and accepts the packets, but limits
what it sends to the new address to three times what it received from there
(RFC 9000 §8) until a `PATH_CHALLENGE`/`PATH_RESPONSE` exchange proves the
address is real. The congestion controller and RTT estimate are reset for the
new path (§9.4), since mobile data isn't Wi-Fi. The streams, their offsets and
the TLS keys all carry on. The application sees a pause, not a failure.

With a load balancer hashing by source address and port, the migrated packets
hash to a different backend. That server has no state for the connection
ID and can't decrypt the packet. The connection fails just as it would have over
TCP. RFC 9308 §3.2 names this case. The load balancer has to route on the
**connection ID** instead, which is why the server chooses the IDs the client
uses (RFC 9000 §5.1): it can encode which backend owns the connection in them,
by agreement with the load balancer.

**4. `POST /transfer` in 0-RTT.**

The attack: someone on the path records the client's first flight, including
the 0-RTT packets, and sends it again, to the same server or another server
sharing the session-ticket keys. The 0-RTT keys come from the earlier session,
not from anything fresh in this handshake, so the copy decrypts as well as the
original. RFC 8446 §2.3: "There are no guarantees of non-replay between
connections." If the server acts on it, the transfer happens twice. The attacker
never needed to read the request.

The document that assigns responsibility is RFC 9001 §9.2: "the responsibility
for managing the risks of replay attacks with 0-RTT lies with an application
protocol." QUIC's own frames are safe to replay; the application's semantics
aren't. For HTTP, RFC 8470 is that profile. A conforming client wouldn't have
sent this at all, since §4 says clients "MUST NOT send unsafe methods" in early
data. A server facing it can, per §3, reject early data at the TLS layer,
defer processing until the handshake completes (by which point a replay would
have failed), or answer `425 Too Early`, which tells the client to retry
without early data.

The contrast with `REFUSED_STREAM`: that question was whether a request that
*failed* might nonetheless have been processed, and a retry would then do it
twice. `REFUSED_STREAM` (and HTTP/3's `H3_REQUEST_REJECTED`) is a promise that
it wasn't. In 0-RTT nothing failed. The duplicate comes from an attacker, not
from your retry logic, and nobody involved has a signal that it happened. The
common thread is the one from file 02: whether repetition is safe depends on
whether the request has side effects.

**5. Why encrypt the header, and why the server name still shows.**

TCP Fast Open is the evidence. Its design was sound and it was standardised, and
it still couldn't be deployed, because enough middleboxes on enough paths
dropped SYNs that looked unusual. RFC 7413 §7.1 cites 6% of probed paths dropping
SYNs carrying data or unknown options, and RFC 9170 Appendix A.5 concludes that
"TCP Fast Open could not be deployed as a result." Any field a middlebox can
read, some middlebox will eventually depend on, and from then on the field can't
change. Visible fields freeze.

So a QUIC whose packet numbers and flags were readable would, within a few
years, have middleboxes that "understand" QUIC version 1, and the next version
would break on them as TCP extensions did. Encrypting and authenticating nearly
everything means middleboxes can't rely on those fields or rewrite them, and the
protocol stays changeable. RFC 9170 §4.3 cites QUIC as exactly this technique.
What operators lose in visibility is the deliberate price. RFC 9312
(Manageability of the QUIC Transport Protocol) covers what remains visible to
them, such as the optional latency spin bit; this course doesn't summarise it.

The server name still shows because the Initial packets' keys aren't secret.
They're derived from a salt published in RFC 9001 §5.2 and the client's
Destination Connection ID, which travels in the clear. Any observer can derive
the same keys and read the ClientHello, SNI and ALPN list included. That
protection raises the bar for middleboxes that would casually parse or modify
the handshake (RFC 9369 §1 discusses the ossification angle); it doesn't hide
anything from an observer willing to implement QUIC's key derivation. Hiding
the server name needs a separate mechanism in TLS itself, which QUIC doesn't
provide.

**6. A video-call protocol on QUIC.**

| Data | Choice | Reason |
|---|---|---|
| Signalling (join, leave, mute) | A stream, possibly one long-lived bidirectional stream | Must arrive, and in order: "mute" then "unmute" reversed is a bug |
| Audio frames | Datagrams | Small, frequent, each one stale within tens of milliseconds; a retransmitted audio frame arrives after it should have played. Fits comfortably in one packet |
| Chat message | A stream, one per message or one shared | Must arrive complete; a few hundred milliseconds late is fine |
| Video delta frames | Datagrams **or** a stream per group of frames, reset when stale | See below |
| Video keyframes | Usually a stream | Often too large for one packet, and DATAGRAM frames can't be fragmented (RFC 9221 §5). Losing one breaks every delta frame that depends on it |

For datagrams on video: each frame is delivered the moment it arrives with no
waiting on anything, and a lost frame costs nothing extra on the network. The
application has to handle loss itself (concealment, or asking for a new
keyframe), and it has to split large frames across several datagrams and
reassemble them, which is reinventing part of a stream.

For streams on video: a keyframe and the deltas that depend on it are a natural
ordered unit, so put them on one stream. Delivery is in order, any size works,
lost packets are recovered if there's time, and when the group goes stale the
sender resets the stream and moves on to the next one. RFC 9308 §4 calls this
emulating partial reliability. The cost is that a loss inside a group stalls the
rest of that group until it's retransmitted.

Neither is simply right. The IETF's Media over QUIC Transport draft supports
both and leans on streams for dependent data, which is a useful sign that "media
means datagrams" is a starting point, not a rule. If you have to pick a default:
datagrams for audio, streams-per-group for video, and measure.

---

## Things to try — file 04

**See a server advertise HTTP/3.** Most large sites announce it in a header over
an ordinary HTTP/2 connection:

```bash
curl -sI https://www.google.com | grep -i alt-svc
```

When this was written, that returned `alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000`.
`h3` is the ALPN token from RFC 9114; `ma` is how long, in seconds, the client
may remember the advertisement; `h3-29` is an identifier from a pre-standard
draft. This header is how a browser learns to try QUIC on its next connection.

**Check whether your curl can actually speak it.** `curl --help all | grep
http3` will likely show `--http3` and `--http3-only`. That doesn't mean they
work. Run `curl -V` and look for `HTTP3` in the Features line. The curl shipped
with macOS at the time of writing lists the flags but not the feature, and
`--http3-only` fails with "the installed libcurl version doesn't support this".
A curl built with HTTP/3 support should print `3` for
`curl -s --http3-only -o /dev/null -w '%{http_version}\n' https://cloudflare.com`.

**Watch the fallback.** In a browser's network panel, add the Protocol column
and load a site that advertises `h3`. The first load is often `h2`, and a reload
`h3`, which is `Alt-Svc` discovery happening in front of you. Then block
outbound UDP on port 443 with your OS firewall and reload: the site should
still work, over `h2`. That is the fallback RFC 9114 §3.1 asks clients for.

---

## Two questions that used to be open here

This list used to ask these. File 04 now covers them, so here are the short
answers.

**What problem with HTTP/2 was QUIC's stream design solving?** TCP
head-of-line blocking. HTTP/2 multiplexes streams over one TCP connection, but
TCP only knows about one ordered byte stream, so a single lost packet stalls
every HTTP/2 stream until it's retransmitted. RFC 9113 §1 admits it: "TCP
head-of-line blocking is not addressed by this protocol." QUIC moves streams into
the transport, so loss recovery knows which stream a lost byte belongs to and
stalls only that one.

**Why does HTTP/2 only solve half of head-of-line blocking?** There were two
layers of it. HTTP/1.1 had **application-layer** head-of-line blocking: one
response at a time per connection, and pipelining "still suffers from
application-layer head-of-line blocking" (RFC 9113 §1). HTTP/2's frames and
stream IDs solved that half. The **transport-layer** half lives in TCP, which
HTTP/2 can't change. By that reasoning one HTTP/2 connection could fare worse on
a lossy link than several HTTP/1.1 connections, where one loss stalls only one
of them; I haven't cited a measurement of how often that happens in practice.
QUIC solves the transport half, and leaves a smaller form of it inside each
stream.

---

## File 05 — Certificates, and where trust comes from

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

## Things to try — file 05

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

## File 06 — Origins, the same-origin policy and CORS

**1. Which are the same origin as `https://shop.example.com`?**

- `https://shop.example.com:443/cart`: same. 443 is the default port for
  `https`, and the path doesn't count.
- `http://shop.example.com/`: different scheme.
- `https://cdn.shop.example.com/`: different host. A subdomain is a separate
  origin, even though it's the same *site*.
- `https://shop.example.com:8080/`: different port.

**2. A hidden form `POST`s to `bank.example/transfer`. Does it happen?**

It can. The same-origin policy blocks *reading* cross-origin responses, not
*sending* cross-origin requests, and form submissions across origins were allowed
long before the policy existed. CORS doesn't come into it: a form post isn't a
`fetch()`, and the attacker doesn't need to read the response anyway. Whether the
transfer goes through depends on the bank's CSRF defences. A CSRF token the
attacker can't read would stop it. So would a `SameSite=Lax` or `Strict` session
cookie, which isn't attached to a cross-site `POST`. If the cookie has no
`SameSite` attribute, the outcome depends on the browser: Chromium treats it as
`Lax` and withholds it; Safari doesn't.

**3. JSON plus `Authorization`, and the `OPTIONS` gets a `404`.**

The request isn't "simple": a JSON `Content-Type` and the `Authorization` header
each force a preflight. The browser sends `OPTIONS` first. It gets a `404` with no
`Access-Control-Allow-*` headers, treats that as "not permitted", and **never sends
the real request**. Your handler never ran. The user sees whatever your front end
does with a failed `fetch`. The console shows a CORS error about the preflight.
Contrast a plain `GET` that fails CORS: that request *was* sent and *was*
processed. The server ran the handler and returned a response, and only the
browser's final check stopped the page reading it. Same console message, very
different server-side effects.

**4. `*` with `Allow-Credentials: true` and `credentials: "include"`.**

The Fetch Standard forbids it outright: "If credentials mode is "include", then
`Access-Control-Allow-Origin` cannot be `*`." If browsers allowed it, *every* site
on the internet could make requests carrying the user's cookies to your API and
read the answers. That's precisely the attack in this chapter's opening, the one
the same-origin policy exists to prevent, re-enabled by one header. Forcing the
server to name a specific origin makes it decide, origin by origin, who may read
logged-in responses.

**5. `SameSite=Strict` and an attacker on `blog.example.com`.**

No, `SameSite` doesn't help. It decides whether to attach a cookie by comparing
**sites**, and `blog.example.com`, `app.example.com` and `api.example.com` are all
the same site: the same scheme and the same registrable domain, `example.com`. A
request from the blog to the API isn't cross-site at all, so even `Strict` sends
the cookie. Whether the blog's page can *read* the response is a separate question
for CORS, which works on origins, and the blog is a different origin from the API.
So the defence is CORS (don't allow-list `blog.example.com`) plus CSRF tokens for
state-changing requests. And treat everything under your registrable domain as one
trust zone for cookies, because the browser does.

**6. Echoing `Origin`, a CDN, and intermittent CORS failures.**

`Vary: Origin` is missing. The response's `Access-Control-Allow-Origin` depends on
the request's `Origin`, but the CDN doesn't know that, so it caches the first
response it sees for the URL and serves it to everyone. If front end A's request
filled the cache, front end B's users get a response saying
`Access-Control-Allow-Origin: https://a.example`, and their browser rejects it.
It's intermittent because it depends on who filled the cache since it last
expired. `Vary: Origin` tells every cache to keep a separate copy per `Origin`
value. That's exactly the case the Fetch Standard's "CORS protocol and HTTP
caches" section describes.

## Things to try — file 06

1. **Watch a preflight pass, then fail.** Open the browser console on any page
   and run `fetch("https://api.github.com/zen", {headers: {"X-Requested-With": "x"}})`.
   In the Network tab, find the `OPTIONS` request before the `GET`. GitHub allows
   that header, so the real request follows. Then repeat with
   `{"X-Test": "1"}`: the preflight still returns `204`, but `X-Test` isn't in
   `Access-Control-Allow-Headers`, so the `GET` is never sent. While you're there,
   note `access-control-max-age: 86400` (checked 8 October 2026), which a Chromium
   browser caps at 7200.
2. **Prove CORS isn't server security.** Find an endpoint that fails CORS in the
   browser, then fetch it with `curl`. It works, because CORS is enforced by
   browsers, not servers.
3. **Break credentials three ways.** Run a tiny local server that echoes `Origin`
   with `Allow-Credentials: true`. Then change it to send a trailing slash, then
   `*`, then `True`, and read the console error each time.

---

## File 07 — Reaching localhost from a web page

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

## Things to try — file 07

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

## Questions worth asking me

- How do flow control and stream priority interact with multiplexing — and why
  did the RFC 9113 revision deprecate the original priority scheme? QUIC says it
  "does not provide a mechanism for exchanging prioritization information"
  (RFC 9000 §2.3) — so how does HTTP/3 prioritise?
- What does QUIC cost? It runs in user space and encrypts every packet
  separately — what does that do to CPU per byte compared with kernel TCP, and
  what have kernels and network cards added to close the gap?
- Which parts of QUIC can a network operator still see, and what does RFC 9312
  say they can still measure?
- How would you load-balance QUIC across a fleet so that connection migration
  survives — what goes into the connection ID, and who has to agree on it?
- When is QUIC the wrong transport? Inside a data centre with little loss and no
  middleboxes, what does it buy over TCP, and what does it cost?
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
