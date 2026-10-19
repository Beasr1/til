# 5. Calling URLs you don't control

Assumes [file 01](01-foundations.md) for the vocabulary — workflow, step, upstream, run,
mapping — and [go-services file 02](../go-services/02-the-http-client.md) for Go's
`http.Client` and the `Transport` that owns its connection pool. It doesn't need Part 1.

[http file 07](../http/07-loopback-and-local-network.md) is this chapter's mirror image.
There, a web page out on the internet tries to reach a program on the user's own machine,
and four browser mechanisms stand in the way. Here the engine is the one reaching inward,
and nothing stands in the way unless you build it.

## 5.1 The problem

Take the running example: fetch a user; then fetch their orders and preferences in
parallel; then post a summary to a CRM.

```
user    GET  https://api.example.com/users/{{input.userId}}
orders  GET  https://shop.example.com/orders?user={{user.id}}       after user
prefs   GET  https://prefs.example.com/v1/{{user.id}}               after user
crm     POST https://crm.example.com/notes  body {{orders}},{{prefs}}  after orders, prefs
```

Someone typed those four URLs into a workflow. The engine will call them, and it calls
them from wherever it runs: inside your private network, on a host that may have a cloud
role attached, next to databases, caches and admin interfaces that treat anything arriving
from a neighbour as trusted.

The person building a workflow isn't the person running the engine (file 01's
one-paragraph summary). So the host of any one of those four steps is just a string in a
config, and a different author can put a different string there — one that points not at a
public API but at an address that only something *inside* your network can reach. The
engine does what it was built to do: it resolves the name and sends the request, from its
own privileged position. That's **server-side request forgery** (SSRF): persuading a
server to issue a request, from its network, that the requester couldn't issue themselves.
MITRE catalogues it as [CWE-918](https://cwe.mitre.org/data/definitions/918); the
[OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
lists "Custom WebHook (users have to specify Webhook handlers or Callback URLs)" as a
classic enabler, which is exactly what a workflow step is.

Two kinds of address make this dangerous, and both are reachable from a normal server with
no exploit — just an HTTP GET to the right host.

**Cloud metadata services.** Every major cloud exposes instance metadata — including
temporary credentials for the role attached to the machine — at a fixed link-local
address, reachable only from the instance itself:

| Cloud | Address(es) | Serves, among other things |
|---|---|---|
| AWS EC2 | `169.254.169.254`, `[fd00:ec2::254]` | role credentials under `/latest/meta-data/iam/security-credentials/` ([docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html)) |
| AWS ECS / EKS | `169.254.170.2`, `169.254.170.23` | task-role / Pod-Identity credentials ([ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html), [EKS](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-how-it-works.html)) |
| GCP | `metadata.google.internal`, `169.254.169.254`, `[fd20:ce::254]` | OAuth2 access tokens for the service account ([docs](https://cloud.google.com/compute/docs/metadata/querying-metadata)) |
| Azure | `169.254.169.254` (IMDS) | managed-identity tokens ([IMDS docs](https://learn.microsoft.com/en-us/azure/virtual-machines/instance-metadata-service)) |
| Alibaba Cloud | `100.100.100.200` | STS credentials ([docs](https://www.alibabacloud.com/help/en/ecs/user-guide/view-instance-metadata)) |

Azure also runs a platform channel, WireServer, at `168.63.129.16`
([docs](https://learn.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16)),
which Microsoft calls "a virtual public IP address". It sits in no IANA special-purpose
range, so the generic block-list in §5.4 treats it as public and *won't* refuse it
(measured). That's a concrete reason the block-list is a floor, not a ceiling (§5.8): a
cloud-specific magic address needs its own explicit deny entry, or an egress proxy that
knows about it (§5.10).

A request to one of those is an ordinary GET. The response is a credential. That is why
metadata endpoints are the first thing an SSRF probe looks for.

**Internal admin ports.** Services on the same host often bind a control interface to
loopback on the assumption that only local, trusted software can reach it. Envoy's
documentation is blunt about its own admin interface: it "both allows destructive
operations to be performed (e.g., shutting down the server) as well as potentially exposes
private information", so "it is critical that access to the administration interface is
only allowed via a secure network… optimally only allowing access to the administration
listener via localhost"
([Envoy admin docs](https://www.envoyproxy.io/docs/envoy/latest/operations/admin)). An
engine that will `POST` to any URL a workflow names can reach `http://127.0.0.1:9901/` as
easily as a public API.

For most applications SSRF is a bug that slipped in. For an orchestration engine, calling
URLs that other people typed *is the feature*, so the defence can't be "don't do that" — it
has to be a limit the engine enforces on every call.

## 5.2 Why validating the URL string doesn't work

The instinct is to vet the string: parse the URL, pull out the host, reject it if it's
private or loopback, otherwise proceed. Every part of that is defeatable, and the reasons
are worth seeing because they all share one shape.

**Alternative spellings.** `127.0.0.1` has many forms that a naive check misses but a
resolver accepts. Running the host strings below through the operating system's resolver
(macOS `getaddrinfo`, measured) returns loopback for all of them:

| You typed | The resolver / parser gives you |
|---|---|
| `2130706433` | `127.0.0.1` (32-bit decimal) |
| `0x7f000001` | `127.0.0.1` (hex) |
| `0177.0.0.1` | `127.0.0.1` via `inet_aton`; `177.0.0.1` via `getaddrinfo` — they *disagree* |
| `127.1` | `127.0.0.1` (shorthand) |
| `[::ffff:127.0.0.1]` | `127.0.0.1` (IPv4-mapped IPv6) |

The `0177.0.0.1` row is the whole problem in miniature: two functions on the same machine
read the same string as two different addresses. A validator that uses one and a dialler
that uses the other can be walked straight past. This isn't hypothetical — the npm `ip`
package was found to misclassify `0x7f.1` as public
([CVE-2023-42282](https://nvd.nist.gov/vuln/detail/CVE-2023-42282)) and, after an
incomplete fix, `127.1` and `::ffff:127.0.0.1` too
([CVE-2024-29415](https://nvd.nist.gov/vuln/detail/CVE-2024-29415)). Go's own standard
library hit the same class: since Go 1.17 `net.ParseIP` rejects IPv4 with leading-zero
decimal parts, because "some operating systems treat them as octal… This mismatch could
hypothetically lead to security issues", and the release note's advice is the key one:
"always re-encode values after validation"
([Go 1.17 release notes](https://go.dev/doc/go1.17)).

**Redirects.** Even a host that validates clean can hand you somewhere else. The upstream
returns `302 Location: http://169.254.169.254/…` and a client that follows redirects
dials that next, having vetted only the first URL. OWASP's advice for this case is flat:
disable redirect following, or re-apply the whole destination policy to every hop.

**DNS rebinding.** This is the sharpest one. A hostname is not an address; it becomes one
only when resolved, and the attacker controls the name's DNS. At validation time the name
resolves to a public address and passes. A moment later, when the client actually dials, it
resolves again — now to `127.0.0.1`. Nothing about the string changed; the mapping under it
did. The technique is old and well documented (Jackson, Barth, Bortz, Shao, Boneh,
*Protecting Browsers from DNS Rebinding Attacks*,
[CCS '07](https://crypto.stanford.edu/dns/dns-rebinding.pdf)), and the browser-era fix,
"DNS pinning", doesn't rescue a server-side client. OWASP states the consequence directly:
"Domain allowlisting alone does not prevent DNS rebinding."

> **Teacher's aside.** All three defeats are the same bug: a **time-of-check /
> time-of-use** gap. You inspect one thing (a string, or a name's resolution now) and then
> act on a different thing (the address actually dialled, later, possibly twice). Any SSRF
> defence that lives earlier than the connection is guarding a value that can still change
> before the packet leaves. The fix isn't a better parser. It's to move the check to the
> last possible moment — the address the socket is about to connect to — so that what you
> check and what you use are the same thing.

This is OWASP's "Case 2": an application that may be asked to call *any* external URL,
where an allow-list of destinations isn't available. Its guidance is to validate, then
"connect only to an IP address validated", disable redirects, and "apply the destination
policy to retries and fallback connections as well."

## 5.3 The fix: check the resolved IP at connection time

Check the actual IP address the socket is about to connect to, on **every** connection,
including every redirect hop, and refuse anything that isn't a public address.

Moving the check there closes all three holes at once:

- **Rebinding** — you check the address you're dialling, not a name you resolved earlier.
  The second, malicious resolution is the one you inspect.
- **Redirects** — each hop opens its own connection, so each hop is checked. You don't
  have to predict where a redirect goes; you catch it when it dials.
- **Spellings** — by connection time the string is long behind you. `2130706433`,
  `0x7f.1` and `::ffff:127.0.0.1` have all become the same 32- or 128-bit number, and you
  judge the number.

What you're inspecting is no longer "did the user type something that looks private" but
"is this packet about to leave for a public host." That's the question that actually
matters, and it has a definite answer.

## 5.4 Which ranges to refuse

"Public" means globally reachable. The authority is the two IANA special-purpose address
registries, whose "Globally Reachable" column (renamed from "Global" by
[RFC 8190](https://www.rfc-editor.org/rfc/rfc8190.txt)) records exactly this: whether a
datagram to that block "is forwardable beyond a specified administrative domain." Refuse
everything the registries mark not globally reachable, plus multicast. Verified against the
[IPv4](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml)
and [IPv6](https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml)
registries (both last updated 2025-10-09):

| Range | Name | Why it's dangerous |
|---|---|---|
| `0.0.0.0/8` | "This host on this network" (RFC 1122) | unspecified / this-host |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Private-Use (RFC 1918) | your private network |
| `100.64.0.0/10` | Shared Address Space ([RFC 6598](https://www.rfc-editor.org/rfc/rfc6598.txt)) | carrier-grade NAT — private, not RFC 1918 |
| `127.0.0.0/8` | Loopback (RFC 1122) | the engine's own host |
| `169.254.0.0/16` | Link-Local (RFC 3927) | includes `169.254.169.254` — cloud metadata |
| `192.0.0.0/24` | IETF Protocol Assignments | infrastructure anycast |
| `198.18.0.0/15` | Benchmarking (RFC 2544) | |
| `224.0.0.0/4`, `240.0.0.0/4` | Multicast / Reserved | `255.255.255.255` broadcast falls in `240/4` |
| `::1/128` | Loopback (RFC 4291) | IPv6 loopback |
| `::/128` | Unspecified | |
| `::ffff:0:0/96` | IPv4-mapped | `::ffff:127.0.0.1` *is* `127.0.0.1` |
| `fc00::/7` | Unique-Local ([RFC 4193](https://datatracker.ietf.org/doc/html/rfc4193)) | IPv6 private |
| `fe80::/10` | Link-Local Unicast | IPv6 link-local |
| `ff00::/8` | Multicast | |
| `64:ff9b::/96` | IPv4/IPv6 translation (NAT64) | can carry a mapped RFC 1918 address |

For IPv6 there's a cleaner way to think about it than a block-list. IANA assigns globally
routable IPv6 only from **`2000::/3`** (the "Global Unicast" row of the
[IPv6 address space registry](https://www.iana.org/assignments/ipv6-address-space/ipv6-address-space.xhtml));
everything else is reserved, loopback, ULA, link-local or multicast. So the robust rule is:
refuse any IPv6 address *outside* `2000::/3`, then additionally refuse the special-purpose
blocks that sit *inside* it (documentation `2001:db8::/32`, 6to4 `2002::/16`, and so on).
That's the design the Go library [`code.dny.dev/ssrf`](https://pkg.go.dev/code.dny.dev/ssrf)
takes, and it keeps you safe against future allocations you've never heard of.

> ⚠️ **Don't lean on a standard-library "is this private" helper alone.** They don't mean
> what the name suggests. Go's `netip.Addr.IsGlobalUnicast` returns `true` for
> `10.0.0.0/8` *and* for IPv6 unique-local space — its own docs say so — so it is not a
> public-address test. `IsUnspecified` matches only the single address `0.0.0.0` or `::`,
> not the `0.0.0.0/8` block. And every check must **unmap IPv4-mapped IPv6 first**
> (`::ffff:127.0.0.1`), or an attacker reaches loopback through the IPv6 path. Build the
> set from the registries and test it; don't assume one predicate covers it.

## 5.5 Refusing non-public addresses at dial time

In Go the hook is [`net.Dialer.Control`](https://pkg.go.dev/net#Dialer): a function the
dialler calls "after creating the network connection but before actually dialling", with
the resolved `address` it's about to connect to. Return an error and the connection never
happens. It runs for every connection the client opens — every DNS answer and, because a
redirect opens a fresh connection, every redirect hop.

Here is the client, exactly as run (the file is `egress/guard.go`; the only things trimmed
from the chapter are the test `main` that drove it and the address-table loop):

```go
// Package egress builds HTTP clients for calling URLs that someone else typed.
package egress

import (
	"errors"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/netip"
	"slices"
	"syscall"
	"time"
)

// IANA assigns global unicast IPv6 only from 2000::/3, so everything outside
// it is refused: ::1, ::, 64:ff9b::/96, fc00::/7, fe80::/10, ff00::/8 and more.
var globalUnicast6 = netip.MustParsePrefix("2000::/3")

// Every block in the IANA special-purpose registries (updated 2025-10-09),
// plus IPv4 multicast. The IPv6 entries are the ones inside 2000::/3.
var blocked = mustPrefixes(
	"0.0.0.0/8", "10.0.0.0/8", "100.64.0.0/10", "127.0.0.0/8", "169.254.0.0/16",
	"172.16.0.0/12", "192.0.0.0/24", "192.0.2.0/24", "192.31.196.0/24",
	"192.52.193.0/24", "192.88.99.0/24", "192.168.0.0/16", "192.175.48.0/24",
	"198.18.0.0/15", "198.51.100.0/24", "203.0.113.0/24", "224.0.0.0/4", "240.0.0.0/4",
	"2001::/23", "2001:db8::/32", "2002::/16", "2620:4f:8000::/48", "3fff::/20",
)

// IsPublic reports whether addr is somewhere a workflow step may connect to.
func IsPublic(addr netip.Addr) bool {
	// ::ffff:127.0.0.1 is 127.0.0.1. A zone (fe80::1%eth0) makes
	// Prefix.Contains return false, so strip it before comparing.
	addr = addr.Unmap().WithZone("")
	if !addr.IsValid() || (addr.Is6() && !globalUnicast6.Contains(addr)) {
		return false
	}
	for _, p := range blocked {
		if p.Contains(addr) {
			return false
		}
	}
	return true
}

// Guard refuses connections to non-public addresses. Its check runs inside
// the dialer, on the address the socket is about to connect to, so it sees
// every DNS answer and every redirect hop.
type Guard struct {
	// Allow is the explicit escape hatch for tests and local development:
	// exact ip:port pairs that may be reached although they aren't public.
	// Leave it empty in production.
	Allow []netip.AddrPort
}

// BlockedError is what a refused connection fails with.
type BlockedError struct{ Addr netip.AddrPort }

func (e *BlockedError) Error() string {
	return fmt.Sprintf("refused to connect to %s: not a public address", e.Addr)
}

// control has the signature net.Dialer.Control wants. The dialer calls it
// after creating each socket and before connecting it.
func (g *Guard) control(network, address string, _ syscall.RawConn) error {
	ap, err := netip.ParseAddrPort(address)
	if err != nil {
		return err
	}
	if slices.Contains(g.Allow, netip.AddrPortFrom(ap.Addr().Unmap(), ap.Port())) {
		return nil
	}
	if !IsPublic(ap.Addr()) {
		return &BlockedError{Addr: ap}
	}
	return nil
}

// NewClient returns a client for upstream URLs that workflow authors typed.
func NewClient(g *Guard) *http.Client {
	dialer := &net.Dialer{Timeout: 5 * time.Second, Control: g.control}

	t := http.DefaultTransport.(*http.Transport).Clone()
	t.Proxy = nil // with a proxy, the dialer would check the proxy's address, not the upstream's
	t.DialContext = dialer.DialContext
	t.ResponseHeaderTimeout = 10 * time.Second
	t.MaxResponseHeaderBytes = 64 << 10

	return &http.Client{
		Transport: t,
		Timeout:   30 * time.Second, // the whole call: connect, redirects, body
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			if len(via) >= 5 {
				return errors.New("stopped after 5 redirects")
			}
			return nil // a new hop dials again, so the guard checks it there
		},
	}
}

// ReadLimited reads the whole body, or fails once it passes limit bytes.
func ReadLimited(body io.Reader, limit int64) ([]byte, error) {
	b, err := io.ReadAll(io.LimitReader(body, limit+1))
	if err != nil {
		return nil, err
	}
	if int64(len(b)) > limit {
		return nil, fmt.Errorf("response body larger than %d bytes", limit)
	}
	return b, nil
}

func mustPrefixes(cidrs ...string) []netip.Prefix {
	out := make([]netip.Prefix, len(cidrs))
	for i, c := range cidrs {
		out[i] = netip.MustParsePrefix(c)
	}
	return out
}
```

Driving it against a test matrix produced, measured (error text abbreviated). The run set
`Allow: {127.0.0.1:18081}` — the §5.9 dev override — so the local lab's redirector and
big-body endpoint could be reached; every *other* loopback target, including the far side
of those redirects, is still refused:

```
REFUSED http://127.0.0.1:18080/                                       dial tcp 127.0.0.1:18080: not a public address
REFUSED http://localtest.me:18080/                                    dial tcp [::1]:18080: not a public address
REFUSED http://[::ffff:127.0.0.1]:18080/                              dial tcp 127.0.0.1:18080: not a public address
REFUSED http://2130706433:18080/                                      dial tcp 127.0.0.1:18080: not a public address
OK      https://example.com/                                          200, 577 bytes
REFUSED https://httpbin.org/redirect-to?url=http://127.0.0.1:18080/   dial tcp 127.0.0.1:18080: not a public address
REFUSED http://127.0.0.1:18081/redirect?to=http://localtest.me:18080/ dial tcp [::1]:18080: not a public address
REFUSED http://127.0.0.1:18081/redirect?to=http://127.0.0.1:18080/    dial tcp 127.0.0.1:18080: not a public address
size cap: response body larger than 1048576 bytes
```

`localtest.me` is a real public name that resolves to loopback (`127.0.0.1` and `::1`) — a
standing one-host stand-in for DNS rebinding. The guard refuses it at the IP, which is the
point: the name passed every string check and the address still didn't. The public
redirector `httpbin.org/redirect-to` sends a `302` to `http://127.0.0.1:18080/`; the entry
host is public, and the guard catches loopback on the redirect hop. The two local-redirector
rows show the same thing from the allowed entry host: the `302` target is still refused at
the hop. With an empty `Allow` (the production default) the first connection to
`127.0.0.1:18081` would itself be refused.

The optional library [`code.dny.dev/ssrf`](https://pkg.go.dev/code.dny.dev/ssrf) (MIT) is
the same idea, kept in sync with the IANA registries by code generation, and wired in the
same place — its `Guardian.Safe` is designed to be a `net.Dialer.Control`. The Go library
[`doyensec/safeurl`](https://github.com/doyensec/safeurl) (Apache-2.0) independently reaches
for the same hook. Two separate projects converging on `Control` is the signal that it's
the right layer.

<details><summary>Rust</summary>

In reqwest 0.13, the resolved-address hook is the [`Resolve`](https://docs.rs/reqwest/0.13.5/reqwest/dns/trait.Resolve.html)
trait, installed with `.dns_resolver(...)`. You resolve the name yourself and drop every
non-public answer. **But there's a Rust-specific gap, and it's the lesson of this
collapsible:** reqwest sits on hyper-util, whose connector skips DNS entirely for an
IP-*literal* host ("If the host is already an IP addr … skip resolving the dns" —
`hyper-util` `connect/http.rs`). So `http://127.0.0.1/` never reaches your resolver. The
guard therefore also rejects non-public literal hosts up front, and disables redirects so
a redirect *to* a literal can't slip past the resolver either (see the trap in §5.6).

```rust
use std::net::{IpAddr, Ipv6Addr, SocketAddr};
use std::sync::Arc;
use std::time::Duration;

use reqwest::dns::{Addrs, Name, Resolve, Resolving};
use reqwest::Url;

// 2000::/3 is the only globally-routable IPv6 range IANA assigns.
fn is_public(ip: IpAddr) -> bool {
    let ip = match ip {
        // ::ffff:127.0.0.1 is 127.0.0.1.
        IpAddr::V6(v6) => match v6.to_ipv4_mapped() {
            Some(v4) => IpAddr::V4(v4),
            None => IpAddr::V6(v6),
        },
        v4 => v4,
    };
    match ip {
        IpAddr::V4(v4) => {
            let b = v4.octets();
            !(v4.is_private()            // 10/8, 172.16/12, 192.168/16
                || v4.is_loopback()      // 127/8
                || v4.is_link_local()    // 169.254/16
                || v4.is_broadcast()
                || v4.is_documentation()
                || v4.is_unspecified()   // 0.0.0.0
                || b[0] == 0             // 0.0.0.0/8
                || (b[0] == 100 && (b[1] & 0xc0) == 64) // 100.64/10 CGNAT
                || b[0] >= 224           // multicast 224/4 and reserved 240/4
                || (b[0] == 192 && b[1] == 0 && b[2] == 0)) // 192.0.0/24
        }
        IpAddr::V6(v6) => {
            let seg = v6.segments();
            (0x2000..=0x3fff).contains(&seg[0]) // inside 2000::/3
                && v6 != Ipv6Addr::LOCALHOST
                && !(seg[0] == 0x2001 && seg[1] == 0x0db8) // 2001:db8::/32
                && seg[0] != 0x2002                        // 2002::/16 (6to4)
        }
    }
}

// Resolver guard: resolve with the system resolver, drop non-public answers.
struct GuardResolver;

impl Resolve for GuardResolver {
    fn resolve(&self, name: Name) -> Resolving {
        let host = name.as_str().to_string();
        Box::pin(async move {
            let addrs = tokio::net::lookup_host((host.as_str(), 0)).await?;
            let kept: Vec<SocketAddr> = addrs.filter(|sa| is_public(sa.ip())).collect();
            if kept.is_empty() {
                return Err("no public address for host".into());
            }
            let out: Addrs = Box::new(kept.into_iter());
            Ok(out)
        })
    }
}

// Reject a URL whose host is an IP literal that isn't public. Names go to the
// resolver guard above. (reqwest's URL layer canonicalises 2130706433 and the
// like to a literal, so those are caught here too.)
fn literal_host_ok(url: &Url) -> bool {
    match url.host() {
        Some(url::Host::Ipv4(v4)) => is_public(IpAddr::V4(v4)),
        Some(url::Host::Ipv6(v6)) => is_public(IpAddr::V6(v6)),
        _ => true,
    }
}

fn client() -> reqwest::Client {
    reqwest::Client::builder()
        .dns_resolver(Arc::new(GuardResolver))
        .no_proxy() // ignore HTTP(S)_PROXY and the macOS/Windows system proxy
        .redirect(reqwest::redirect::Policy::none()) // follow hops yourself, re-validating each
        .connect_timeout(Duration::from_secs(5))
        .timeout(Duration::from_secs(30))
        .build()
        .unwrap()
}
```

reqwest has no built-in response-size cap; enforce one by streaming `resp.chunk()` and
counting bytes against a limit. Each request should call `literal_host_ok` before
`client.get(...)`; with `Policy::none` the client returns a `3xx` unfollowed, so the engine
re-runs the whole guard on each `Location` and dispatches the next hop itself.

</details>

<details><summary>C++ (libcurl)</summary>

libcurl gives you [`CURLOPT_OPENSOCKETFUNCTION`](https://curl.se/libcurl/c/CURLOPT_OPENSOCKETFUNCTION.html)
(since curl 7.17.1), which replaces libcurl's own `socket()` call. It receives the
**resolved** peer address and may "refuse to connect completely" by returning
`CURL_SOCKET_BAD`. It fires for every connection, including each redirect hop. This mirrors
curl's own [`docs/examples/block_ip.c`](https://github.com/curl/curl/blob/master/docs/examples/block_ip.c).
(The alternative, [`CURLOPT_PREREQFUNCTION`](https://curl.se/libcurl/c/CURLOPT_PREREQFUNCTION.html),
since 7.80.0, runs just *after* the connection is established but before the request is
sent, and is also handed the primary IP and also "called multiple times if redirections
are enabled" — refusing after connect rather than before.)

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <curl/curl.h>

// IPv4 in host byte order.
static bool v4_public(uint32_t a) {
    auto in = [&](uint32_t net, int bits) {
        uint32_t mask = bits == 0 ? 0 : (0xFFFFFFFFu << (32 - bits));
        return (a & mask) == (net & mask);
    };
    if (in(0x00000000, 8) || in(0x0A000000, 8) || in(0x64400000, 10) ||   // 0/8, 10/8, 100.64/10
        in(0x7F000000, 8) || in(0xA9FE0000, 16) || in(0xAC100000, 12) ||  // 127/8, 169.254/16, 172.16/12
        in(0xC0A80000, 16) || in(0xC0000000, 24) || in(0xC0000200, 24) || // 192.168/16, 192.0.0/24, TEST-NET-1
        in(0xC6120000, 15) || in(0xC6336400, 24) || in(0xCB007100, 24) || // 198.18/15, TEST-NET-2, TEST-NET-3
        in(0xE0000000, 4)  || in(0xF0000000, 4))                          // 224/4 multicast, 240/4 reserved
        return false;
    return true;
}

// 16 raw bytes, network order.
static bool v6_public(const uint8_t* b) {
    // IPv4-mapped ::ffff:0:0/96 -> judge as the IPv4 address it carries.
    static const uint8_t mapped[12] = {0,0,0,0,0,0,0,0,0,0,0xff,0xff};
    if (std::memcmp(b, mapped, 12) == 0)
        return v4_public((uint32_t(b[12]) << 24) | (b[13] << 16) | (b[14] << 8) | b[15]);
    // Only 2000::/3 is globally routable; everything else (::1, ::, fc00::/7,
    // fe80::/10, ff00::/8, 64:ff9b::/96 ...) is refused here.
    if ((b[0] & 0xE0) != 0x20) return false;
    if (b[0] == 0x20 && b[1] == 0x01 && (b[2] & 0xFE) == 0x00) return false; // 2001::/23
    if (b[0] == 0x20 && b[1] == 0x01 && b[2] == 0x0d && b[3] == 0xb8) return false; // 2001:db8::/32
    if (b[0] == 0x20 && b[1] == 0x02) return false; // 2002::/16 (6to4)
    if (b[0] == 0x3f && b[1] == 0xff && (b[2] & 0xf0) == 0x00) return false; // 3fff::/20
    return true;
}

static curl_socket_t open_guarded(void* clientp, curlsocktype purpose,
                                  struct curl_sockaddr* addr) {
    (void)clientp;
    if (purpose == CURLSOCKTYPE_IPCXN) {
        bool ok = true;
        if (addr->family == AF_INET) {
            auto* s = reinterpret_cast<struct sockaddr_in*>(&addr->addr);
            ok = v4_public(ntohl(s->sin_addr.s_addr));
        } else if (addr->family == AF_INET6) {
            auto* s = reinterpret_cast<struct sockaddr_in6*>(&addr->addr);
            ok = v6_public(s->sin6_addr.s6_addr);
        }
        if (!ok)
            return CURL_SOCKET_BAD; // libcurl fails the transfer with CURLE_COULDNT_CONNECT
    }
    return socket(addr->family, addr->socktype, addr->protocol);
}

static void configure(CURL* c) {
    curl_easy_setopt(c, CURLOPT_OPENSOCKETFUNCTION, open_guarded);
    curl_easy_setopt(c, CURLOPT_FOLLOWLOCATION, 1L);
    curl_easy_setopt(c, CURLOPT_MAXREDIRS, 5L);
    curl_easy_setopt(c, CURLOPT_PROTOCOLS_STR, "http,https");
    curl_easy_setopt(c, CURLOPT_REDIR_PROTOCOLS_STR, "http,https");
    curl_easy_setopt(c, CURLOPT_PROXY, ""); // ignore http_proxy/https_proxy from the environment
    curl_easy_setopt(c, CURLOPT_MAXFILESIZE_LARGE, (curl_off_t)(1 << 20));
    curl_easy_setopt(c, CURLOPT_TIMEOUT_MS, 30000L);
    curl_easy_setopt(c, CURLOPT_CONNECTTIMEOUT_MS, 5000L);
}
```

The only things trimmed are the `main` that looped the test matrix, a response sink, and a
line lowering TLS verification for the lab's self-signed certificate — keep verification on
in production.

> ⚠️ `CURLOPT_MAXFILESIZE_LARGE` only aborts before the transfer when the server sends a
> `Content-Length`; a chunked response without one isn't stopped by it. Count bytes in your
> write callback as a backstop.

</details>

<details><summary>JavaScript (Node)</summary>

Node has [`net.BlockList`](https://nodejs.org/docs/latest-v24.x/api/net.html#class-netblocklist),
checked inside the socket's connect, *after* the DNS lookup and before the handshake
(`lib/net.js`, `internalConnect`). It covers IP-literal hosts too, because the literal path
reaches the same check. You pass it as the `blockList` option on a socket. Support for
`blockList` on `net.connect` arrived in Node 22.13.0 / 23.4.0.

```js
import net from 'node:net'
import tls from 'node:tls'
import { Agent, interceptors, request } from 'undici'

const blockList = new net.BlockList()
for (const [addr, n] of [
  ['0.0.0.0', 8], ['10.0.0.0', 8], ['100.64.0.0', 10], ['127.0.0.0', 8],
  ['169.254.0.0', 16], ['172.16.0.0', 12], ['192.0.0.0', 24], ['192.0.2.0', 24],
  ['192.88.99.0', 24], ['192.168.0.0', 16], ['198.18.0.0', 15], ['198.51.100.0', 24],
  ['203.0.113.0', 24], ['224.0.0.0', 4], ['240.0.0.0', 4],
]) blockList.addSubnet(addr, n, 'ipv4')
blockList.addAddress('::1', 'ipv6')
blockList.addAddress('::', 'ipv6')
for (const [addr, n] of [
  ['fc00::', 7], ['fe80::', 10], ['ff00::', 8], ['2001:db8::', 32],
  ['2002::', 16], ['64:ff9b::', 96],
]) blockList.addSubnet(addr, n, 'ipv6')

// Connect the raw socket first, with the block list, then upgrade to TLS.
// A plain tls.connect() does NOT honour blockList (see the trap below).
function guardedConnect (opts, callback) {
  const https = opts.protocol === 'https:'
  const port = Number(opts.port) || (https ? 443 : 80)
  const raw = net.connect({ host: opts.hostname, port, blockList })
  const onError = (err) => callback(err, null)
  raw.once('error', onError)
  raw.once('connect', () => {
    raw.removeListener('error', onError)
    if (!https) return callback(null, raw)
    const tlsSock = tls.connect({ socket: raw, servername: opts.servername || opts.hostname })
    tlsSock.once('secureConnect', () => callback(null, tlsSock))
    tlsSock.once('error', (err) => callback(err, null))
  })
}

const agent = new Agent({
  connect: guardedConnect,
  connectTimeout: 5000,
  headersTimeout: 10000,
  bodyTimeout: 30000,
  maxResponseSize: 1 << 20, // undici's own cap; default is -1 (unlimited)
})
// Each redirect hop opens a connection through guardedConnect, so each hop is checked.
const dispatcher = agent.compose(interceptors.redirect({ maxRedirections: 5 }))

const res = await request('https://example.com/', { dispatcher })
```

Trimmed: the loop over the test matrix and a `rejectUnauthorized: false` for the lab's
self-signed certificate (the default, `true`, is what you want). A refused address surfaces
as an error with `code === 'ERR_IP_BLOCKED'`.

</details>

## 5.6 Where each stack lets you see the address

The hook points differ, and so does what each one covers. This is the part worth
internalising, because a guard that's airtight in one language has a hole in another.

| Stack | Hook | Fires | Re-runs per redirect hop | Sees IP-literal hosts |
|---|---|---|---|---|
| Go | `net.Dialer.Control` | before connect | yes (new dial per hop) | yes |
| C++ (libcurl) | `CURLOPT_OPENSOCKETFUNCTION` | before connect | yes | yes |
| C++ (libcurl) | `CURLOPT_PREREQFUNCTION` | after connect, before request | yes | yes |
| Node | `net.BlockList` on the raw socket | at connect, after lookup | yes (new socket per hop) | yes |
| Rust (reqwest) | `Resolve` trait | at name resolution | yes, for name hops | **no** — literals skip DNS |

Go, libcurl and Node check the address *itself*, so a literal and a redirect to a literal
are both caught. All of this is measured: a public→loopback redirect was refused on the hop
in Go (`dial tcp 127.0.0.1`), in C++ (`refused 127.0.0.1`, transfer failed with HTTP 302
recorded), and in Node (`ERR_IP_BLOCKED`).

Rust's `Resolve` hook is different in kind: it sits at *name resolution*, so it never sees a
destination given as an IP literal. Measured, with redirects left on: a request to
`https://httpbin.org/redirect-to?url=http://127.0.0.1:18080/` followed the `302` and
**reached loopback** (returned the local server's body), because the redirect target was a
literal that the resolver guard never inspected. Setting `redirect::Policy::none()` stopped
it — the client returned the `302` unfollowed — which is why the Rust guard disables
redirects and re-validates each hop itself.

> ⚠️ **Two silent holes to know about.** In Rust, a custom `Resolve`/`dns_resolver` alone
> does not guard IP-literal URLs; pair it with an up-front literal check *and* disabled
> redirects. In Node, `tls.connect()` silently ignores the `blockList` option — the
> `TLSSocket` constructor doesn't forward it (measured: `net.connect` to `127.0.0.1` gives
> `ERR_IP_BLOCKED`, `tls.connect` with the same `blockList` connects anyway). That's why
> the Node example connects the raw socket (with the block list) first and upgrades to TLS
> over it, rather than letting `tls.connect` dial.

## 5.7 The rest of the client

The dial-time check is the heart of it, but four more settings have to be right or they
reopen the hole.

**Disable proxies taken from the environment.** This one is subtle and important. If the
client reads `HTTP_PROXY`/`HTTPS_PROXY` from the environment, it connects to the *proxy*,
and your dial-time check now inspects the proxy's address, not the upstream's — the proxy
then resolves and dials the upstream for you, unguarded. Go's `http.DefaultTransport` sets
`Proxy: ProxyFromEnvironment` ([docs](https://pkg.go.dev/net/http#ProxyFromEnvironment)), so
the guard above sets `t.Proxy = nil`. reqwest reads the environment *and* the macOS/Windows
system proxy by default, so the Rust guard calls `.no_proxy()`. libcurl honours
`http_proxy` unless you set `CURLOPT_PROXY` to `""`, which "explicitly disables the use of a
proxy, even if there is an environment variable set for it". The clouds say the same from
their side — Azure's IMDS docs tell clients to "bypass web proxies within the VM when
querying IMDS." (If you *do* want a proxy, make it a deliberate egress proxy that does its
own filtering — §5.10 — not one inherited from ambient environment variables.)

**Cap the response size.** A step that pulls a multi-gigabyte body can exhaust the engine's
memory as surely as any injection. Go's `io.LimitReader` plus a length check (the
`ReadLimited` helper above) bounds it; libcurl has `CURLOPT_MAXFILESIZE_LARGE`; undici has
`maxResponseSize`. reqwest has no built-in cap, so you stream and count. Cap request headers
too (Go's `Transport.MaxResponseHeaderBytes`).

**Cap the time.** A whole-request deadline (`http.Client.Timeout`, reqwest `.timeout`,
libcurl `CURLOPT_TIMEOUT_MS`, undici `bodyTimeout`) plus a connect deadline, so a slow or
black-holed upstream can't pin a worker open. This is [go-services file 02](../go-services/02-the-http-client.md)'s
territory — the only addition here is that the deadline must also bound the redirect chain,
not just the first hop.

**Restrict schemes and ports.** Only `http` and `https` — not `file:`, `gopher:`, `dict:`
and the other schemes a general client may support, which OWASP calls out as SSRF vectors.
libcurl's `CURLOPT_PROTOCOLS_STR` and `CURLOPT_REDIR_PROTOCOLS_STR` do this directly;
`code.dny.dev/ssrf` even defaults to allowing only ports 80 and 443 (measured: it refused a
connection to `127.0.0.1:18080` on the *port* before the IP check ran).

## 5.8 Allow-lists vs block-lists

OWASP frames the whole cheat sheet around a split, and it's the right one. **Case 1** is an
application that only ever calls a known set of destinations — use an allow-list. **Case 2**
is one that may be asked to call *anything*, where "the list of IPs/domains is often unknown
upfront and is dynamically changing" so an allow-list isn't available; there you fall back
to a block-list, which OWASP is careful to call "the best solution in this scenario" while
"knowing that the block-list approach is not an impenetrable wall."

A general-purpose workflow engine is Case 2 — the whole point is that authors type
arbitrary URLs — so the block-list of §5.4 is the floor. But "Case 2 overall" doesn't mean
"Case 2 everywhere." Two refinements are usually available:

- **Per-workflow allow-lists.** A given workflow, or a given tenant, often *does* have a
  known set of hosts. Let it declare them, and enforce an allow-list on top of the
  block-list for that run. The block-list stops the dangerous ranges; the allow-list stops
  everything the workflow didn't ask for.
- **The block-list is a deny floor, not a safety ceiling.** It stops the engine being
  turned against your own network. It does nothing about the engine being used to hammer a
  public third party, or to scan public address space. Those need rate limits and,
  ultimately, an allow-list.

The ordering rule, from the OWASP example and the libraries: validate the host, resolve it,
check *the resolved IP* against your policy, and **build the request from what matched** —
don't carry the user's original string through to the dialler. Re-encode after validating.

## 5.9 The dev/test override must be explicit

Your own tests need to reach `127.0.0.1`. Developers run the engine against a local mock.
So there has to be a way to allow a private address — and that way is exactly where this
goes wrong, because the easy version ("allow loopback if a config flag is set") tends to
read the same config the untrusted workflow can influence, and now the override is the
vulnerability.

Make it narrow and deliberate. The `Guard.Allow` field above is a list of *exact* `ip:port`
pairs, empty by default, set only from the engine's own trusted configuration — never from
anything a workflow author supplies. Smokescreen (§5.10) takes the same stance: reaching
local addresses needs an explicit `--allow-self-connections`, and bypassing IP filtering for
a host needs a flag literally named `--unsafe-ip-filter-bypassed-domain`. The naming is the
design: an override you can't enable by accident, and can grep for in an audit.

## 5.10 Defence in depth: an egress control outside the engine

The dial-time guard lives in the engine's own process, which means a bug in the engine — a
new code path that builds a client without the guard, a dependency bump that resets a
`Transport` — can quietly remove it. A second control, outside the process, catches that.

The usual form is an **egress proxy**: all outbound traffic goes through one hardened
component that enforces the policy independently. Stripe's
[Smokescreen](https://github.com/stripe/smokescreen) (MIT) is a worked example — an HTTP
`CONNECT` proxy that, per its README, "resolves each domain name that is requested, and
ensures that it is a publicly routable IP address and not an internal IP address. This
prevents a class of attacks where … our own webhooks infrastructure is used to scan [the]
internal network." It also centralises egress, which buys a second thing: a small set of
stable outbound IPs you can give to partners, instead of every engine host dialling out
directly.

Smokescreen's own documentation carries the warning that makes the whole chapter concrete:
its hostname ACL matches "hostnames *as they appear in the request*", and if you want to
filter on "the destination IP address *after DNS resolution*" you must use its IP-filtering
config instead. That's §5.2 restated by the tool itself — the name is not the address, and
only the post-resolution IP check is sound against rebinding.

Two more layers worth naming:

- **Network segmentation.** OWASP's "network layer" advice: don't rely only on the
  application. Put the engine where the network itself can't route to your sensitive
  ranges, so a missed check in code still hits a closed route.
- **Harden the metadata endpoint.** On AWS, IMDSv2 requires a `PUT` to start a session and a
  token header on every request, which "protects against the vast majority of SSRF
  vulnerabilities"
  ([AWS](https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/)).
  GCP and Azure require a header (`Metadata-Flavor: Google`, `Metadata: true`) and reject
  any request carrying `X-Forwarded-For`.

**What none of this solves.** Be honest about the edges:

- IMDSv2's header defence works, in AWS's own words, "only when the vulnerability merely
  allows the attacker to control the URL" — and AWS "found many SSRF vulnerabilities that
  allow attackers to set arbitrary headers." An engine that lets a workflow set arbitrary
  request headers (many do, for auth) gives back exactly the capability the header defence
  assumed the attacker lacked. Header control is its own thing to limit.
- An IP guard says nothing about *public* abuse. The engine can still be pointed at any
  public host to flood it, scan it, or relay through it. That's a rate-limiting and
  allow-list problem, not an SSRF-range problem.
- A legitimate public upstream that is itself compromised, or that redirects you somewhere
  new on every call, is inside your allowed set by definition. The guard limits where the
  engine *connects*, not whether the thing it connected to is trustworthy.
- Everything here concerns IP destinations. A workflow that can read files, run templates
  that reach a filesystem, or trigger other steps has other inside-reach paths that an
  egress IP check never sees.

The guard closes the biggest, cheapest hole — the engine being turned into a window onto
your own network. It is necessary and it is not sufficient, which is why it's one layer of
several.

## Check yourself

1. A reviewer proposes blocking SSRF by parsing each URL, rejecting any host in
   `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` or `127.0.0.0/8`, and allowing the rest.
   Give two different inputs that pass this check and still reach a private address, and say
   which defeat each one uses.
2. The engine resolves a hostname, confirms the IP is public, and then makes the request
   using the original hostname. Mechanically, what can happen between the check and the
   request, and why does checking the resolved IP *at connection time* instead remove the
   problem?
3. Why must the address check run again on every redirect hop, and not just on the first
   URL? Describe the request that exploits a client that checks only the first.
4. The Rust guard validates every resolved address in a custom `Resolve` and still lets a
   request reach `127.0.0.1`. Two different inputs trigger this — one with no redirect, one
   with. Explain the mechanism in each case and the two changes that close them.
5. A teammate adds an egress allow-list keyed on the *hostname* in the request and says
   "metadata is blocked now, `169.254.169.254` isn't in the list." Why is that not enough,
   and what does Smokescreen's own warning about its hostname ACL tell you to do instead?
6. Your engine lets workflow authors set arbitrary request headers, and you run on AWS with
   IMDSv2 enabled. Someone argues the dial-time IP guard is now redundant because IMDSv2
   "protects against SSRF." Where is the gap, and which of the two controls actually stops a
   request to `169.254.169.254` in this setup?

(Answers in [09-exercises.md](09-exercises.md).)
