# 1. What changed between HTTP/1.1 and HTTP/2

## The problem that made HTTP/2

HTTP/1.1 sends one response at a time per connection. A browser wanting thirty
assets either waits for each in turn or opens several connections — and each
connection costs a TCP handshake, a TLS handshake and its own congestion window.

The obvious fix is to interleave: send a bit of response A, a bit of response B,
a bit of A again. HTTP/1.1 cannot, and the reason is worth sitting with, because
everything else follows from it.

## HTTP/1.1 is a byte stream with no envelope

A response looks like this, and the important part is what's missing:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1270

<!doctype html><html>…
```

There is no marker saying "this chunk belongs to response A". The body is simply
the next 1270 bytes on the socket. The receiver knows where the body ends because
`Content-Length` said so, or because `Transfer-Encoding: chunked` marks the end,
or — in the oldest and worst case — because the connection closes.

So the connection can only carry one response at a time. Interleaving two would
be indistinguishable from one corrupt response.

## HTTP/2 wraps everything in frames

HTTP/2 puts an envelope around every piece of data. Each frame carries a length,
a type, flags, and the id of the **stream** it belongs to. RFC 9113 §5 defines a
stream as:

> A "stream" is an independent, bidirectional sequence of frames exchanged
> between the client and server within an HTTP/2 connection.

Now interleaving is trivial. Frames for stream 1 and stream 3 can alternate on
the wire because each frame says which stream it belongs to:

```
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ HEADERS  │ DATA     │ DATA     │ DATA     │ DATA     │
│ stream 1 │ stream 1 │ stream 3 │ stream 1 │ stream 3 │
└──────────┴──────────┴──────────┴──────────┴──────────┘
```

One TCP connection, one TLS handshake, many concurrent requests. That is the
whole point of HTTP/2, and the framing is what buys it.

> **Teacher's aside.** People describe HTTP/2 as "binary HTTP", which makes it
> sound like a compression trick. Binary is incidental. The change is that
> responses gained *identity* — a stream id — and identity is what lets them
> share a pipe. Everything else in the protocol exists to manage that sharing:
> flow control per stream, priority between streams, and a way to abandon one
> stream without killing the connection.

## Two ways a stream ends, and they are not equivalent

This is where the framing starts to bite.

| | Mechanism | Meaning to the receiver |
|---|---|---|
| Clean | `END_STREAM` flag on the final frame | "That was all of it" |
| Aborted | `RST_STREAM` frame | "Stop; this stream is over early" |

RFC 9113 on each. The flag:

> When set, the END_STREAM flag indicates that this frame is the last that the
> endpoint will send for the identified stream.

And the frame:

> The RST_STREAM frame (type=0x03) allows for immediate termination of a stream.
> RST_STREAM is sent to request cancellation of a stream or to indicate that an
> error condition has occurred.

HTTP/1.1 has no equivalent distinction. A response that ends is a response that
ended; the only way to signal "disregard that" is to drop the connection, which
takes every other in-flight request with it. HTTP/2 can abandon one stream and
leave the connection healthy — a genuine improvement, and the source of a failure
mode that does not exist in HTTP/1.1.

## RST_STREAM carries an error code

Every `RST_STREAM` names a reason. The codes through `0x08`, from RFC 9113 §7 —
the registry continues past these, and the ones below are the ones you meet:

| Code | Name | Means |
|---|---|---|
| 0x00 | NO_ERROR | Graceful shutdown |
| 0x01 | PROTOCOL_ERROR | The peer violated the protocol |
| 0x02 | INTERNAL_ERROR | Something broke on the sender's side |
| 0x03 | FLOW_CONTROL_ERROR | Flow-control window violated |
| 0x04 | SETTINGS_TIMEOUT | Settings were never acknowledged |
| 0x05 | STREAM_CLOSED | Frame arrived for a stream already closed |
| 0x06 | FRAME_SIZE_ERROR | Frame size illegal for its type |
| 0x07 | REFUSED_STREAM | Never processed; safe to retry elsewhere |
| 0x08 | CANCEL | "The sender is canceling the stream, requesting that the receiver stop transmitting frames for this stream" |

`CANCEL` is ordinary when a **client** sends it — you navigate away mid-download
and the browser cancels the stream. It is not ordinary from a **server** that has
just finished writing a successful response.

## The failure this produces

Picture a server whose application layer builds a complete `200`, logs it, and
hands it down to the HTTP/2 layer — which then emits `RST_STREAM(CANCEL)` instead
of setting `END_STREAM`. The logs say success. The wire says abandon.

Clients disagree about what to do, and the disagreement is instructive:

| Client | Behaviour |
|---|---|
| `curl` | Reports the status it received, plus a warning: `HTTP/2 stream 1 was not closed cleanly: CANCEL (err 8)` |
| Browsers | Discard the response entirely; Chrome shows `ERR_HTTP2_PROTOCOL_ERROR` |

The browser is right to be strict. A reset stream means the sender withdrew the
response, so a partially-received body cannot be trusted — and the browser cannot
tell a truncated page from a complete one, because in HTTP/2 completeness is
signalled by `END_STREAM`, not by byte count.

> ⚠️ **The trap: your logs will say 200.** Application-layer logging sits above
> framing. A middleware that records "finished processing request, status=200" is
> reporting what the handler returned, not what reached the client. When a client
> reports a protocol error against a server whose logs are clean, suspect the
> layer between them — and reach for a client that shows you the frames rather
> than one that shows you the status.

## Seeing it yourself

`curl -v` names the stream outcome. Compare the last line of each:

```
# Healthy HTTP/1.1 — connection reusable, nothing withdrawn
< HTTP/1.1 200 OK
* Connection #0 to host example left intact

# HTTP/2 with a server-side reset
> GET / HTTP/2
* Request completely sent off
* HTTP/2 stream 1 was not closed cleanly: CANCEL (err 8)
```

Forcing a version separates "is my handler wrong?" from "is my framing wrong?":

| Command | Forces |
|---|---|
| `curl --http1.1 https://host/` | HTTP/1.1 |
| `curl --http2 https://host/` | HTTP/2 where offered |

If the response is correct under `--http1.1` and broken under `--http2`, the
handler is fine and the problem is in the HTTP/2 implementation or its use. That
is a two-command bisection of a bug that otherwise looks like a mystery.

## Check yourself

1. Why can't HTTP/1.1 interleave two responses on one connection? Answer in terms
   of what a receiver can and cannot determine from the bytes.
2. A server sends HEADERS and three DATA frames for stream 5, then
   `RST_STREAM(CANCEL)` on stream 5. What must the receiver do with the three
   DATA frames it already holds, and why is discarding them the only safe choice?
3. `curl` shows `200` with a "not closed cleanly" warning; Chrome shows only
   `ERR_HTTP2_PROTOCOL_ERROR` on the same URL. Neither client is wrong. Explain
   both behaviours.
4. Your access log shows `status=200` for a request the browser reports as
   failed. Name two layers where the response could have been lost between the
   log line and the browser, and give a command that distinguishes them.
5. `REFUSED_STREAM` (0x07) is defined as the stream never having been processed,
   which makes it safe to retry on another connection. Why would retrying on
   `CANCEL` (0x08) or `INTERNAL_ERROR` (0x02) be unsafe by comparison?
