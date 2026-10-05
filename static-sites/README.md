# Static Sites, from Build to Your Own Domain — A Course

A course on building and shipping a static website as an engineer: what "static"
actually means, choosing a generator, choosing a host, deploying from CI without handing
out more power than you need, putting it on a domain you own, and what happens in the
first hours after it goes public.

**This is reference learning material.** Everything here is general — no particular
site, host account or project. Plans, prices and version numbers were checked against
each provider's own pages on **5 October 2026** and are dated in the text; this field
changes monthly, so recheck anything time-sensitive before you quote it. General HTTP
and TLS knowledge is assumed.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- The problem comes before the mechanism, and the mechanism before the config.
- Every file ends with **Check yourself** questions. Answers are in
  [08-exercises.md](08-exercises.md).
- Where a source couldn't be checked, or sources disagree, the text says so.

## The one thing to understand first

> **A static site moves all the work to before the request.** Nothing of yours runs
> when a visitor arrives, which is why it's cheap, fast and hard to break — and why the
> interesting risks are in the build pipeline and in the name, not on the server.

And the second thing, which surprises everyone on launch day:

> **Issuing a certificate publishes your hostname.** Certificate Transparency logs are
> public by design, so a new site is found by scanners within minutes, whether or not
> you've told anyone.

## Primary sources

| Source | What it settles |
|---|---|
| [MDN — What is a web server?](https://developer.mozilla.org/en-US/docs/Learn_web_development/Howto/Web_mechanics/What_is_a_web_server) | The static vs dynamic definition |
| [Hugo docs](https://gohugo.io/documentation/) · [Zola docs](https://www.getzola.org/documentation/) · [Eleventy docs](https://www.11ty.dev/docs/) · [Astro docs](https://docs.astro.build/) | Each generator's install, templating, feeds and assets |
| [Tera docs](https://keats.github.io/tera/) | Delimiters and `raw` blocks |
| [GitHub Pages docs](https://docs.github.com/en/pages) | Plan availability, limits, HTTPS, publishing sources |
| [Cloudflare Workers static assets](https://developers.cloudflare.com/workers/static-assets/) · [Pages](https://developers.cloudflare.com/pages/) | Limits, custom domains, the Pages → Workers guidance |
| [Netlify credit-based plans](https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/credit-based-pricing-plans/) | Free-plan limits and pausing |
| [Caddy — Automatic HTTPS](https://caddyserver.com/docs/automatic-https) | Self-hosting with automatic certificates |
| [GitHub — Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use) | Least privilege, SHA pinning |
| [GitHub — OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect) | Short-lived tokens instead of stored secrets |
| [actions/deploy-pages](https://github.com/actions/deploy-pages) · [actions/configure-pages](https://github.com/actions/configure-pages) | The Pages deploy, its permissions and its error messages |
| [cloudflare/wrangler-action](https://github.com/cloudflare/wrangler-action) | Deploying to Cloudflare from CI |
| [RFC 9082](https://www.rfc-editor.org/rfc/rfc9082.html) / [9083](https://www.rfc-editor.org/rfc/rfc9083.html) / [9224](https://www.rfc-editor.org/rfc/rfc9224.html) | RDAP queries, responses and bootstrap |
| [RFC 1034 §3.6.2](https://www.rfc-editor.org/rfc/rfc1034.html), [RFC 2181 §10.1](https://www.rfc-editor.org/rfc/rfc2181.html), [RFC 9460](https://www.rfc-editor.org/rfc/rfc9460.html) | Why no CNAME at the apex, and the standard alternative |
| [Let's Encrypt — challenge types](https://letsencrypt.org/docs/challenge-types/) · [RFC 8659 (CAA)](https://www.rfc-editor.org/rfc/rfc8659.html) | How a host proves control and gets a certificate |
| [RFC 6962](https://www.rfc-editor.org/rfc/rfc6962.html) / [RFC 9162](https://www.rfc-editor.org/rfc/rfc9162.html) · [Chrome CT policy](https://googlechrome.github.io/CertificateTransparency/ct_policy.html) | Certificate Transparency, and why it's mandatory in practice |
| [Chrome — Removing XSLT](https://developer.chrome.com/docs/web-platform/deprecating-xslt) | The XSLT removal timeline, relevant to styled feeds |

## Reading order

### Part 1 — Foundations (prerequisite for everything else)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations](01-foundations.md) | Say what "static" really means, read a generator's output folder, and name who does what among registrar, DNS, host and CDN |

### Part 2 — Building and shipping

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Choosing a generator](02-choosing-a-generator.md) | Pick between Hugo, Zola, Eleventy and Astro for reasons that will still matter in a year, and avoid the template-delimiter trap |
| 3 | [Where to host it](03-where-to-host.md) | Compare blog platforms, GitHub Pages, Cloudflare, Netlify and a VPS on the constraints that bite: private repos, limits, injected scripts, lock-in |
| 4 | [**Deploying from CI, safely**](04-deploying-from-ci.md) | ⭐ Write a deploy workflow with least privilege, OIDC where possible, pinned tools with checksums and SHA-pinned actions |

### Part 3 — Going public

| # | File | After this you can… |
|---|------|---------------------|
| 5 | [Your own domain](05-your-own-domain.md) | Check availability properly, buy on renewal price, point the apex and `www`, and understand why `.dev` forces HTTPS |
| 6 | [Going live](06-going-live.md) | Explain launch-day "visitors", read analytics honestly, and decide deliberately about tracking and feeds |

### Reference

| # | File | |
|---|------|--|
| 7 | [Glossary](07-glossary.md) | Look things up |
| 8 | [Exercises & answers](08-exercises.md) | Worked answers to every Check yourself, plus things to try |

Files 02 and 03 can be read in either order. File 04 assumes you've picked a host (03).
File 05 can be read any time after 01 — if you're buying a domain this week, read it
first.

## If you're short on time

- **20 minutes:** file 01 §1.5–1.6, then file 06 §6.2.
- **Setting up a deploy pipeline:** files 01, 04, then file 03's table.
- **Buying a domain:** file 05, then file 06 §6.2 (what happens once the certificate exists).
- **Picking tools for a new site:** files 01, 02, 03.

## The one-paragraph summary of everything

A static site is a folder of files produced ahead of time by a generator from content
and templates, and served unchanged to every visitor — "static" means no code of yours
runs per request, not that the page can't move. Clean URLs come from writing
`slug/index.html` and letting the server's directory index do the rest. Generators split
into single binaries (Hugo, Zola: one file to pin and checksum) and npm tools (Eleventy,
Astro: more extensible, larger dependency tree), and each template engine reads your
hand-written HTML first, so its delimiters (`{{`, `{%`, `{#`) must not appear
unescaped. Hosts differ in the margins that matter: GitHub Pages needs a public repo on
the free plan, Netlify's free plan pauses at its limit, Cloudflare now steers new static
sites to Workers and ties custom domains to its DNS, and blog platforms reserve your
domain, scripts and footer for paying users. Because nothing runs at request time, the
risk lives in CI: build once with read-only permissions, deploy the artefact from a
separate job, prefer short-lived OIDC tokens to stored secrets, and pin tools by version
*and* checksum and actions by commit SHA. Your domain is the one thing others link to,
so own it: check availability knowing an RDAP 404 can mean "unknown TLD", compare
renewal not first-year prices, use ALIAS/flattening or `www` because the apex can't hold
a CNAME, and on HSTS-preloaded TLDs like `.dev` expect nothing to load until the
certificate exists. And the certificate announces you: Certificate Transparency
publishes every hostname, so scanners arrive within minutes and your first analytics are
mostly bots — which is also why an unguessable hostname is never protection.

## How to use me

Ask things like:

- "Which generator would you pick for this site, given these constraints: …?"
- "Audit my deploy workflow for permissions and pinning."
- "I got this error from a deploy — which layer is it from?"
- "Explain my first week of access logs: what's human and what isn't?"
- "Walk me through moving this site from host X to host Y without breaking links."
- "My domain shows a certificate error — DNS, ACME, CAA, or HSTS?"
- "Is my mental model right? Here's what I think happens: …"
