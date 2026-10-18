# 9. Networks: layers, TCP and UDP

## The problem

Getting bytes from a program on one machine to a program on another involves electrical
signals, local hardware addresses, routing across networks owned by strangers, recovering
from lost and reordered packets, not overwhelming the receiver, not collapsing the
network, and delivering to the right program on the right machine. No one design handles
all of that cleanly, so the problem is split into **layers**, each solving one part and
relying on the one below. Interview questions here are about that split (OSI vs TCP/IP)
and about the one layer every backend engineer depends on daily: transport, meaning TCP
and UDP.

Assumes [file 01 §1.8](01-what-happens-when-a-program-runs.md#18-the-network-a-socket-is-a-file-descriptor):
a socket is a file descriptor, and the kernel runs the network stack.

## 9.1 Layers, by analogy

Sending a document to a colleague in another city. You put it in an envelope with their
name and office. The mailroom puts the envelope in a bag addressed to that city's sorting
office. The courier puts the bag on a truck. Each step wraps the previous one and adds
only the address *it* needs, and each step neither reads nor cares about what's inside.
At the other end, each layer unwraps its own wrapping and hands the contents up.

Networks do exactly that, and the wrapping is called **encapsulation**:

```
 application    │               HTTP request bytes                 │
                ▼                                                  ▼
 transport      │ TCP header │        HTTP request bytes           │  ← ports: which program
                ▼                                                  ▼
 network        │ IP header │ TCP header │   HTTP request bytes    │  ← IP addresses: which machine
                ▼                                                  ▼
 link           │ Eth hdr │ IP hdr │ TCP hdr │  HTTP bytes  │ Eth trailer │  ← MAC: which next hop
```

| Layer | Unit (usual name) | Address it adds | Answers |
|---|---|---|---|
| Application | Message | URL, hostname | What does the program want? |
| Transport | Segment (TCP) / datagram (UDP) | **Port** | Which program on that machine? |
| Network | Packet | **IP address** | Which machine, across the internet? |
| Link | Frame | **MAC address** | Which device on *this* local network, for the next hop? |
| Physical | Bits | None | How are bits put on the wire or radio? |

The MAC address changes at every hop (each router re-wraps the packet for its next link);
the IP addresses normally don't. Finding the MAC address for an IP on your local network
is **ARP**, the Address Resolution Protocol ([RFC 826](https://www.rfc-editor.org/rfc/rfc826.html),
1982).

## 9.2 OSI vs TCP/IP: two models, one of which won

There are two standard layer models, and interviewers ask for both:

| OSI (7 layers) | TCP/IP (4 layers, RFC 1122) | Kurose & Ross (5 layers) | Examples |
|---|---|---|---|
| 7 Application | Application | Application | HTTP, DNS, SMTP |
| 6 Presentation | ↑ (folded in) | ↑ | Encoding, encryption: TLS is often placed here, loosely |
| 5 Session | ↑ (folded in) | ↑ | Session management |
| 4 Transport | Transport | Transport | TCP, UDP, QUIC |
| 3 Network | Internet | Network | IP, ICMP |
| 2 Data link | Link | Link | Ethernet, Wi-Fi, ARP |
| 1 Physical | ↓ (folded into Link) | Physical | Cables, radio |

The **OSI model** is ISO/IEC 7498-1, published identically as ITU-T X.200
([X.200](https://www.itu.int/rec/T-REC-X.200-199407-I/en)), which sets out "The
principles used to determine the seven layers in the Reference Model". The **TCP/IP
model** comes from the internet's own engineering documents: RFC 1122 says "The protocol
layers used in the Internet architecture are as follows" and lists four: Application,
Transport, Internet, Link ([RFC 1122 §1.1.3](https://www.rfc-editor.org/rfc/rfc1122.html#section-1.1.3)).
The widely used textbook by Kurose and Ross (*Computer Networking: A Top-Down Approach*,
9th ed., 2025) teaches a five-layer hybrid, splitting physical from link.

Why two? OSI was a full protocol suite as well as a model, developed through formal
standards bodies in the late 1970s and 1980s. The internet's protocols were built and
deployed first, and won. Andrew Russell's history of it is titled "OSI: The Internet That
Wasn't" ([IEEE Spectrum, 2013](https://spectrum.ieee.org/osi-the-internet-that-wasnt)):
OSI "ultimately lost out to the Internet's Transmission Control Protocol/Internet Protocol
(TCP/IP) and is now all but forgotten." Its protocols are gone; its *numbering* survived,
which is why engineers say "a layer-7 load balancer" or "an L4 proxy".

> **Teacher's aside.** OSI's seven layers are a teaching model and a vocabulary, not a
> description of how real stacks are built. Nobody implements separate session and
> presentation layers. Real protocols also don't stay in their lanes: TLS sits between
> TCP and HTTP and fits no OSI layer cleanly; QUIC does transport and encryption together
> over UDP; ARP serves the network layer but runs at the link layer. The internet's own
> engineers have written against strict layering: RFC 3439's §3 is titled "Layering
> Considered Harmful" ([RFC 3439](https://www.rfc-editor.org/rfc/rfc3439.html#section-3)).
> When an interviewer asks you to place something in the OSI model, give the conventional
> answer and, if it's ambiguous, say so. That's the honest answer, and it's the one
> that sounds like experience.

## 9.3 IP: best effort, and nothing more

The network layer's protocol, IP, moves a packet from one machine towards another across
many networks. It promises remarkably little: packets can be lost, duplicated, reordered
or corrupted, and IP won't tell you. This is called **best-effort delivery**, and
everything reliable is built above it.

A few addressing facts come up often:

| Topic | What to know |
|---|---|
| **CIDR notation** | `10.1.0.0/16` means "the first 16 bits are the network": 65,536 addresses. Classful A/B/C addressing is obsolete; CIDR replaced it |
| **Private ranges** | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` ([RFC 1918 §3](https://www.rfc-editor.org/rfc/rfc1918.html)): not routed on the public internet |
| **NAT** | A router rewrites private source addresses to its public one, tracking connections so replies come back. Why your laptop's `192.168.x.x` can browse the internet |
| **IPv6** | 128-bit addresses, so NAT isn't needed for address shortage. Clients often try IPv4 and IPv6 in parallel ("Happy Eyeballs", RFC 8305) |

## 9.4 Transport: TCP and UDP

IP gets a packet to a machine. The transport layer gets data to the right *program* there
(by port), and optionally makes delivery reliable.

| | **TCP** | **UDP** |
|---|---|---|
| Connection | Connection-oriented: handshake first (§9.5) | Connectionless: just send |
| Delivery | Reliable: lost data is retransmitted | Best effort: lost datagrams are just lost |
| Order | In order, as a byte stream | Each datagram stands alone; may arrive in any order |
| Message boundaries | **None.** Two `write`s of 100 bytes may arrive as one read of 200, or 150 + 50 | Preserved: one send, one datagram |
| Flow and congestion control | Yes (§9.7) | No; the application must behave |
| Header | 20 bytes minimum | 8 bytes: source port, destination port, length, checksum ([RFC 768](https://www.rfc-editor.org/rfc/rfc768.html)) |
| Used by | HTTP/1.1 and HTTP/2, databases, SSH, Kafka | DNS queries, video calls, games, QUIC (and so HTTP/3) |

The "no message boundaries" row is the one that bites application code. TCP delivers a
stream of bytes, so any protocol on top of it must mark where messages end: a length
prefix, a delimiter, or HTTP's `Content-Length`. Code that assumes one `read` returns one
message works on a quiet local network and fails under load.

Choosing UDP isn't choosing unreliability; it's choosing to build exactly the reliability
you want. A video call would rather drop a late frame than wait for its retransmission,
which TCP would insist on. QUIC builds its own reliable, encrypted streams on top of UDP
([RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html)).

### What a socket server looks like in four languages

The same echo server in each language: listen on a port, accept connections, and send
back whatever arrives. All four were built and tested with `nc`.

```go
ln, err := net.Listen("tcp", "127.0.0.1:9000") // socket + bind + listen
if err != nil {
	log.Fatal(err)
}
for {
	conn, err := ln.Accept() // blocks until a client completes the handshake
	if err != nil {
		log.Fatal(err)
	}
	go func() { // one goroutine per connection
		defer conn.Close()
		io.Copy(conn, conn) // read bytes, write them back, until the client closes
	}()
}
```

<details><summary>Rust</summary>

```rust
use std::io;
use std::net::TcpListener;
use std::thread;

fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:9001")?;
    for stream in listener.incoming() {
        let stream = stream?;
        thread::spawn(move || {
            // one OS thread per connection; fine for a demo, not for 10k clients
            let mut reader = stream.try_clone().unwrap();
            let mut writer = stream;
            io::copy(&mut reader, &mut writer).ok();
        });
    }
    Ok(())
}
```

</details>

<details><summary>C++</summary>

The C++ standard library has no sockets (a Networking TS was proposed but isn't in the
standard), so this is the POSIX API that every other version wraps:

```cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>
#include <thread>

int main() {
    int ln = socket(AF_INET, SOCK_STREAM, 0);
    int yes = 1;
    setsockopt(ln, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9002);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    bind(ln, reinterpret_cast<sockaddr*>(&addr), sizeof addr);
    listen(ln, 128);                        // backlog of connections waiting for accept()
    for (;;) {
        int conn = accept(ln, nullptr, nullptr);
        std::thread([conn] {
            char buf[4096];
            ssize_t n;
            while ((n = read(conn, buf, sizeof buf)) > 0) write(conn, buf, n);
            close(conn);
        }).detach();
    }
}
```

</details>

<details><summary>JavaScript (Node)</summary>

```js
import net from "node:net";

// One thread, many connections: each chunk of data is a callback on the event loop.
net.createServer((socket) => socket.pipe(socket)).listen(9003, "127.0.0.1");
```

</details>

The four differ in exactly the way [file 02 §2.6](02-processes-and-threads.md#26-how-four-languages-do-it)
predicts: Go gives each connection a cheap goroutine, Rust and C++ an OS thread each
(which is why real Rust servers use async runtimes instead), and Node handles every
connection on one thread through callbacks. Underneath, all four make the same system
calls: `socket`, `bind`, `listen`, `accept`, `read`, `write`.

## 9.5 Opening and closing a TCP connection

### The three-way handshake

"The 'three-way handshake' is the procedure used to establish a connection"
([RFC 9293 §3.5](https://www.rfc-editor.org/rfc/rfc9293.html#section-3.5), the current
TCP specification, which obsoleted RFC 793 in 2022).

```
     Client                                         Server
       │                                              │  LISTEN
       │ ── SYN, seq = x ───────────────────────────► │
SYN-SENT                                              │  SYN-RECEIVED
       │ ◄── SYN + ACK, seq = y, ack = x + 1 ──────── │
ESTABLISHED                                           │
       │ ── ACK, ack = y + 1 ───────────────────────► │
       │                                              │  ESTABLISHED
       │ ── data ───────────────────────────────────► │
```

Why three messages, not two? TCP numbers every byte ("every octet of data sent over a TCP
connection has a sequence number", §3.4), and each side picks its own **initial sequence
number**. Both numbers have to be announced *and acknowledged*. The client's SYN announces
`x`; the server's reply acknowledges `x` and announces `y`; only the third message
acknowledges `y`. With two messages, the server would never learn whether its number
arrived, and couldn't distinguish a real new connection from a stale duplicate SYN left
over from an old one.

The handshake costs one round trip before any data flows, which is why connection reuse
matters so much ([file 10 §10.6](10-what-happens-when-you-type-a-url.md#106-counting-round-trips)).

> ⚠️ **SYN floods.** A server that allocates state on receiving a SYN can be exhausted by
> attackers sending SYNs and never completing the handshake. The standard defence, SYN
> cookies, "allocate no state at all for connections in SYN-RECEIVED. Instead, they
> encode most of the state ... into the sequence number transmitted on the SYN-ACK"
> ([RFC 4987 §3.6](https://www.rfc-editor.org/rfc/rfc4987.html)).

### Closing, and TIME-WAIT

Each direction is closed separately with a FIN, acknowledged by the other side, so a
full close is usually four segments (FIN, ACK, FIN, ACK). The side that closes first ends
up in **TIME-WAIT**: "When a connection is closed actively, it MUST linger in the
TIME-WAIT state for a time 2xMSL (Maximum Segment Lifetime)"
([RFC 9293 §3.6.1](https://www.rfc-editor.org/rfc/rfc9293.html#section-3.6.1)).

It lingers for two reasons: the final ACK might be lost, so it must be able to resend it
when the peer resends its FIN; and stray segments from this connection must die out
before a new connection with the same addresses and ports could mistake them for its
own.

How long is that? The RFC says "For this specification the MSL is taken to be 2 minutes"
(§3.4.2), so TIME-WAIT would be **4 minutes**. Linux doesn't follow it:
`include/net/tcp.h` has `#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy
TIME-WAIT state, about 60 seconds */`. The RFC itself calls the MSL value "an engineering
choice, and may be changed if experience indicates it is desirable to do so." So the
honest answer to "how long is TIME-WAIT?" is "2×MSL by the spec, 60 seconds on Linux".

The practical consequence: a client that opens and closes thousands of short connections
to one server accumulates sockets in TIME-WAIT, and can run out of local ports. The fix
is to reuse connections (HTTP keep-alive, connection pools) rather than to tune the
timeout.

## 9.6 Reliability: sequence numbers, ACKs and retransmission

TCP turns IP's best effort into a reliable stream with a few mechanisms:

| Mechanism | What it does |
|---|---|
| **Sequence numbers** | Number every byte, so the receiver can reorder and detect gaps and duplicates |
| **Cumulative ACKs** | "I've received everything up to byte *n*" |
| **Retransmission timeout (RTO)** | If no ACK arrives in time, resend. The timeout adapts to measured round-trip times |
| **Fast retransmit** | Several duplicate ACKs (the receiver repeating "still waiting for byte *n*") signal a lost segment, so resend without waiting for the timeout ([RFC 5681 §3.2](https://www.rfc-editor.org/rfc/rfc5681.html#section-3.2)) |
| **Checksum** | Detect corruption; a corrupt segment is dropped and treated as lost |

## 9.7 Flow control vs congestion control

These are different problems with similar-looking solutions, and confusing them is the
classic mistake.

| | **Flow control** | **Congestion control** |
|---|---|---|
| Protects | The **receiver** from being overwhelmed | The **network** from being overwhelmed |
| Analogy | Don't pour faster than the person can drink | Don't drive onto a jammed motorway at full speed |
| Signal | The receiver advertises its free buffer space: the **receive window** (`rwnd`) in every segment | Inferred by the sender from loss (and, in some algorithms, delay) |
| Variable | `rwnd`, set by the receiver | `cwnd` (congestion window), kept by the sender |
| Spec | RFC 9293 §3.8.6 | RFC 5681, which RFC 9293 points to |

RFC 5681 ties them together in one sentence: "The minimum of cwnd and rwnd governs data
transmission" ([§3](https://www.rfc-editor.org/rfc/rfc5681.html#section-3)). The sender
may have at most that much unacknowledged data in flight.

Congestion control's classic shape, from RFC 5681:

- **Slow start.** Start with a small `cwnd` and grow it by roughly one segment for every
  ACK, which doubles it every round trip. "Slow" is a historical joke: it's exponential.
- **Congestion avoidance.** Past a threshold (`ssthresh`), grow by about one segment per
  round trip: linear.
- **On loss**, cut back: halve on a fast retransmit, restart from small on a timeout. This
  **additive increase, multiplicative decrease (AIMD)** gives the famous sawtooth.

```
 cwnd
   │                 loss                 loss
   │                  ╱│                   ╱│
   │                ╱  │                 ╱  │
   │              ╱    │ halve         ╱    │
   │            ╱      │╲            ╱      │
   │      ┌───╱        │  ╲        ╱        │
   │     ╱  congestion    ╲      ╱
   │    ╱   avoidance       ╲  ╱
   │   ╱ slow start           (linear growth again)
   │  ╱  (exponential)
   └─────────────────────────────────────────────► time
```

Real stacks use refinements. CUBIC (RFC 9438, 2023) "has been adopted as the default TCP
congestion control algorithm in the Linux, Windows, and Apple stacks"
([RFC 9438 §1](https://www.rfc-editor.org/rfc/rfc9438.html)); Linux's default is set at
build time (`DEFAULT_CUBIC` in `net/ipv4/Kconfig`). Google's BBR (Cardwell et al., *ACM
Queue*, 2016) instead models the path's bandwidth and round-trip time rather than reacting
only to loss.

## 9.8 Head-of-line blocking, and why QUIC exists

TCP delivers bytes strictly in order. If one segment is lost, everything after it waits
in the receiver's buffer until the retransmission arrives, even data that has already
arrived. That's **head-of-line blocking** at the transport layer.

It matters for HTTP/2, which multiplexes many independent requests over one TCP
connection ([http course, file 02](../../http/02-versions-and-framing.md#http2-wraps-everything-in-frames)).
One lost packet belonging to one response stalls *all* of them, because TCP doesn't know
they're independent. QUIC fixes it by implementing streams itself over UDP, so a loss
stalls only the stream it belongs to. HTTP/3 is HTTP over QUIC
([RFC 9114](https://www.rfc-editor.org/rfc/rfc9114.html)).

## Asked in interviews

- "Explain the OSI model. How does it differ from TCP/IP?" → §9.2
- "At which layer does a router / switch / load balancer work?" → §9.1, §9.2
- "TCP vs UDP? When would you use UDP?" → §9.4
- "Explain the three-way handshake. Why three and not two?" → §9.5
- "What is TIME_WAIT and why does it exist?" → §9.5
- "How does TCP guarantee reliability?" → §9.6
- "Flow control vs congestion control?" → §9.7
- "What's head-of-line blocking? How does HTTP/3 address it?" → §9.8

## Check yourself

1. A packet travels from your laptop through two routers to a server. Which addresses in
   it change at each hop, and which stay the same? Why?
2. Where would you put TLS in the OSI model, and why is any answer slightly wrong?
3. A client sends two messages over TCP with two separate `write` calls, and the server's
   first `read` returns both, concatenated. Is this a bug in TCP? What must the
   application protocol do about it?
4. Explain why the handshake needs a third message. What concrete failure would a
   two-message handshake allow?
5. A service makes a new TCP connection for every outbound HTTP call, at 2,000 calls per
   second to one upstream. After a while, connections start failing with "cannot assign
   requested address". Explain the mechanism, and the right fix.
6. A fast server sends to a slow phone over an uncongested network, and separately a fast
   server sends to a fast client over a congested link. Which mechanism limits the
   sending rate in each case, and which variable is the binding one?

(Answers in [13-exercises.md](13-exercises.md).)
