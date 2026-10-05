# 8. Exercises and answers

Try to answer before reading. Getting one wrong is data, not failure.

---

## File 01 — Foundations

**1. Local time via JS, exchange rate from an API. Static?**

Yes. The property that decides it is whether *the server* produces different bytes per
request, and here it doesn't: every visitor receives the same HTML and JS file. The
clock and the API call run in the visitor's browser, after the bytes arrive. "Static"
is about where and when HTML is made, not about whether the page changes on screen. If
you moved the exchange-rate fetch to the server, so the HTML arrived with the rate baked
in per request, it would stop being static.

**2. Rename `about.md` to `about-me.md`.**

The generator derives the slug from the filename, so the output moves from
`about/index.html` to `about-me/index.html`. The old file simply isn't written any more.
Every existing link to `/about/` — your own nav if it's hard-coded, other people's sites,
search results — now gets a 404, because a static host serves only files that exist and
nothing tells it the page moved. Fixes: set the slug in front matter so the filename no
longer controls the URL, or add a redirect from the old path using whatever your host
supports. The general lesson: in a static site, **the URL is a file path**, so renaming a
file is changing a URL.

**3. `/notes/` becomes `/notes` without a redirect.**

The relative `diagram.png` is resolved against the page URL's *directory*. On
`/notes/`, the directory is `/notes/`, so the image loads from `/notes/diagram.png`. On
`/notes`, the last segment is treated as a file name, the directory is `/`, and the
browser requests `/diagram.png` — which doesn't exist. The HTML is identical; only the
URL it was served at changed. Fixes: make the host redirect to the slash form, or use
root-relative links (`/notes/diagram.png`), which most generators can produce for you.

**4. Host C goes out of business.**

Only C and your DNS records. The registration is at A and is untouched; the DNS zone is
at B and still answers. You deploy the same output folder to a new host D, then at B
change the records that pointed at C (the CNAME for `www`, the A/AAAA or flattened
record for the apex) to D's targets. Registrar A isn't involved, because the nameservers
— the only thing A holds for you — haven't changed. This is the payoff of keeping the
roles separate in your head.

**5. "Static sites are more secure because there's no server."**

There is a server — something must answer HTTP. What's removed is *your code running
on it per request*: no application, no database, no request-time input reaching your
logic, so whole classes of attack (injection, auth bugs, unpatched app dependencies)
have nothing to hit. What remains is the file server itself, run by your host. And the
risk moved to the build and deploy pipeline: whoever can push to the repo, whatever the
CI job downloads and executes, and whatever credential it uses to upload. A compromised
pipeline publishes whatever it likes, with your name on it. File 04 is that half.

---

## File 02 — Choosing a generator

**1. When "single binary" decides it, and when "on npm" does.**

Single binary: you need to rebuild the site years from now, or on a locked-down CI
runner with no Node, or you want the whole toolchain verifiable by one checksum. What
you're buying is **reproducibility and a small surface** — one artefact, no transitive
dependencies resolved at install time. On npm: you need a specific Markdown extension,
an image pipeline, or an interactive React component that exists only in that ecosystem.
What you're buying is **extensibility** — and you pay for it with a dependency tree you
must lock (file 04 §4.6).

**2. `generate_feed = true` on Zola 0.23.**

The build fails before rendering anything, with a config parse error. Zola renamed the
key to `generate_feeds` in 0.19.0, and its `Config` struct uses `deny_unknown_fields`, so
an unknown key is an error rather than being ignored. You'd have predicted it from the
changelog — and more generally from Zola being 0.x, where minor releases may break
config. It's a good argument for pinning the generator version in CI.

**3. `<style>@media print{#comments{display:none}}</style>` and a missing footer.**

The template engine reads the template before the browser ever does. `{#` opens a Tera
comment, so everything from there until the next `#}` is treated as a comment and
dropped from the output. If a later `#}` exists (or the engine tolerates an unclosed
comment), the footer simply vanishes with no error. Fixes: move the CSS to a stylesheet
file, which the engine never parses (best: it also gets cached); wrap the block in
`{% raw %}…{% endraw %}`; or insert a space (`{ #comments`). Choose the separate file —
the space is fragile because a minifier may remove it, and raw blocks need remembering
every time someone edits.

**4. A Jinja2 snippet in Markdown works on one generator, breaks on another.**

The difference is whether Markdown is passed through a template engine before
conversion. On a generator that converts Markdown straight to HTML and inserts it, the
`{{ }}` in your code block is just text. Eleventy, by default, runs Markdown through
Liquid first, so `{{ user.name }}` in your post is evaluated as a Liquid expression —
typically rendering as empty — and `{% for %}` may throw. Zola looks for shortcode
syntax in Markdown. Same file, different pipeline order.

**5. "Eleventy is owned by a company and was renamed, so it's risky."**

Partly right about the question, wrong to stop there. Ownership and incentive changes
are real risk factors — but Hugo and Zola are 0.x projects that can also break you, and
any project can lose maintainers. What actually determines exposure is how much of your
site is written in the generator's dialect. Check: the licence file (can it be forked if
direction changes?); your templates directory (how many lines of engine-specific
syntax?); config (generator-specific features you depend on); and content (is it plain
Markdown, or full of shortcodes that only this generator understands?). Content in plain
Markdown with a small template set is cheap to move; heavy shortcode use is the real
lock-in.

---

## File 03 — Where to host it

**1. Private source, pay nothing.**

GitHub Pages: no — on GitHub Free, Pages requires a public repository. Cloudflare: yes
— keep the private repo wherever you like (including GitHub), build in CI, deploy with a
scoped API token; the host never needs to read the repo. Caveat: the token is a
long-lived secret (file 04 §4.4), and on Workers your domain's DNS must be on
Cloudflare. Netlify: possibly — its docs make organisation-owned private repos a Pro
feature and don't state clearly what a personal private repo can do on Free; and the
Free plan's hard credit limit pauses your sites if exceeded. Test before relying on it.
In every case the published site is public; "private" is about the source.

**2. Two years on a free-tier platform subdomain.**

You can take the content — posts can be exported, and they're your words. The design
has to be rebuilt elsewhere, which is work but not loss. What's lost permanently is
every inbound link: other sites, search results, bookmarks and shares all point at the
platform's domain, which you don't control and can't redirect from unless the platform
lets you. That's why owning the domain first (file 05) matters more than any other
hosting choice.

**3. A viral post on each host.**

GitHub Pages: the bandwidth limit is *soft* — the site keeps serving, and GitHub may
contact you if usage is excessive. Netlify Free: bandwidth and requests consume credits;
when the 300 run out, every project on the team pauses until the next cycle and
visitors see "Site not available". Cloudflare Workers static assets: static requests are
free and unlimited on Free, so it keeps serving. The preference part: a hard stop
guarantees you never pay; a host that keeps serving (or a paid plan that bills overage)
keeps you online at the risk of a bill or a polite email. Which failure is worse depends
on whether you value the post being readable or the cost being bounded.

**4. One Worker function for redirects.**

Requests that the Worker code handles now count against the Worker request limit
(100,000/day on Free), rather than being static asset requests, which are free and
unlimited. Depending on how routing is set up, that can include requests that previously
never touched code. The fully static alternative: a `_redirects` file, which Workers
static assets supports and applies "to static asset responses"
([Cloudflare — redirects](https://developers.cloudflare.com/workers/static-assets/redirects/)),
or, host-independently, tiny HTML pages at the old URLs with a meta refresh and a
canonical link.

**5. "Caddy makes self-hosting as easy as a static host."**

Right: certificates used to be the hardest ongoing chore of self-hosting, and Caddy
genuinely removes it — obtained and renewed automatically, from a three-line config.
Left out: OS and Caddy updates, the firewall, SSH access hygiene, backups, monitoring
and being the person who notices when the machine is down; capacity under a traffic
spike; and geography — one server is one location, whereas a static host serves from a
CDN near each visitor. HTTPS was one item on a list; the rest is still yours.

---

## File 04 — Deploying from CI

**1. One job, malicious post-install script.**

In a single job the script runs in the same environment as the deploy step, so it can
read `CLOUDFLARE_API_TOKEN` and send it anywhere. The attacker then holds a credential
that works from anywhere until you notice and revoke it — they can deploy whatever they
like, whenever they like. With the two-job shape, the build job holds no deploy
credential, so there's nothing to steal. Be precise about what that does *not* stop: the
malicious script can still tamper with the output folder, and the deploy job will
faithfully ship the poisoned artefact. Separation turns "attacker owns your account" into
"attacker altered one build" — a large improvement, but not immunity. Locking
dependencies and reviewing updates is what protects the artefact itself.

**2. `id-token: write` versus a secret; what a copied credential is worth later.**

GitHub Pages trusts GitHub's OIDC: the deploy job asks GitHub for a signed token stating
which repository, ref and workflow it is, and the Pages API validates those claims.
Nothing needs storing, so no secret — but the job needs permission to *request* the
token, which is `id-token: write`. Cloudflare doesn't accept GitHub OIDC for deploys (as
of October 2026), so the only proof available is a stored API token, and there's no
OIDC token to request. An hour later: the copied OIDC token has expired and was bound to
that job's claims — close to worthless. The copied API token works exactly as well as
it did in CI, from any machine, until revoked.

**3. `status: 404` and `permissions: write-all`.**

The `deploy-pages` source builds that message for a 404 specifically, adding "Ensure
GitHub Pages has been enabled". A permissions problem returns 403, with a different
hint about `pages: write`. 404 means the Pages site the action is trying to deploy to
doesn't exist for this repository — Pages isn't enabled, or isn't set to build from
GitHub Actions. More permissions can't create it, and `write-all` would grant every
scope to every job, undoing least privilege for nothing. Fix: Settings → Pages → Source:
GitHub Actions, then re-run.

**4. A version pin without a hash.**

One sequence: an attacker gains write access to the project's releases (a stolen
maintainer token, say) and replaces the `linux-amd64` asset of an existing release with
a trojaned build under the same name. Your workflow still asks for `0.167.0`, gets the
replaced file, and runs it with whatever the build job can reach. The version pin
doesn't help — the version number never changed. A recorded checksum would: the new
file's SHA-256 wouldn't match, and the build would stop. A SHA-pinned *action* doesn't
help here either, unless the download happened inside an action — it protects which
action code runs, not what that code downloads. (A project publishing immutable releases
closes the asset-swap route at source, but only if it has opted in.)

**5. A, B, C with `cancel-in-progress: false`, then `true`.**

With `false`: A starts. B queues as pending. C arrives while A is still running, and
replaces B — the pending B is cancelled. A finishes, then C runs. Deployed: A, then C;
B never deploys. Because C's commit includes B's changes on a linear `main`, the final
site is correct. With `true`: when B queues, A is cancelled mid-run and B starts; when C
queues, B is cancelled and C runs. Only C completes. The end state is the same; the
difference is that A and B were killed partway, and whether that's harmless depends on
whether your host swaps deployments atomically or copies files in place.

**6. SHA pins everywhere — except a reusable workflow by tag and a nested action.**

The guarantee only covers the references you pinned. The reusable workflow referenced
by tag is mutable: whoever controls that repository can change what it does, and GitHub's
own SHA-pinning policy explicitly lets reusable workflows be referenced by tag. The
third-party action pinned by SHA is frozen *as written* — but if its code says `uses:
other/action@v2`, that inner reference resolves at run time and can change. Same for a
Docker-based action using a tag, or a script that downloads "latest". SHA pinning
protects exactly one layer of indirection. Past that, you're trusting each dependency's
own pinning, which is why the fewer third-party actions a deploy uses, the better.

---

## File 05 — Your own domain

**1. 404 from rdap.org for a `.io` name.**

Two different meanings share the status code. Either rdap.org redirected to the `.io`
registry's RDAP server and *that* server said "not found" — or rdap.org itself has no
RDAP server for `.io` and returned 404 without asking anyone. As of the September 2026
IANA bootstrap file, `.io` had no entry, so it's the second: the 404 says nothing about
the name. Tell them apart by looking at the response without following redirects (`curl
-sI` — a 302 means it was forwarded; a direct 404 means it wasn't) or by checking the
bootstrap file for the TLD. The definitive check is a registrar's availability search,
which asks the registry at the point of sale.

**2. Promotional versus at-cost.**

Compare the total for the period you expect to keep the name: first year + renewal ×
(years − 1), using each registrar's *renewal* price, plus transfer costs if you'd move.
A cheap first year with a high renewal usually loses over any horizon longer than a year
or two. The at-cost option's condition, in Cloudflare's case: the domain must use
Cloudflare's nameservers, so your registrar and DNS provider become the same company.
That may be fine — but it's a coupling, not just a price.

**3. `.dev`, immediate certificate error, no way past.**

`.dev` is on the HSTS preload list as a whole TLD, so the browser refuses plain HTTP
before sending anything and won't let you click through a certificate error. The host
can only obtain a certificate once it can prove control of the name — typically HTTP-01,
which needs DNS to already point at it. DNS changes take time to be seen (resolvers cache
old answers for their TTL), and the ACME validation and issuance follow after. In that
window the host serves a default or missing certificate, which the browser rejects with
no override. Do: wait, watch the host's domain status, and check what public resolvers
return (`dig +short example.dev @1.1.1.1`). Don't: churn the DNS records, which restarts
propagation and the validation.

**4. Pointing both names at `site-1234.host.example` without ALIAS.**

`www.example.com CNAME site-1234.host.example.` is fine. `example.com CNAME …` is
illegal: the apex already holds SOA and NS records, and a CNAME can't coexist with other
data (RFC 1034 §3.6.2, RFC 2181 §10.1). Ways around it: (a) if the host publishes stable
IP addresses for apex use, create A/AAAA records for them; (b) redirect the apex to `www`
with a redirect service or the DNS provider's forwarding feature, so only `www` needs the
CNAME; (c) move DNS to a provider that offers ALIAS or flattening. An HTTPS-record
AliasMode (RFC 9460) at the apex is standard, but only helps clients that query HTTPS
records, so it can't be your only route.

**5. CAA allows one CA; the new host's certificate never arrives.**

The new host requests a certificate from its CA. Before issuing, publicly trusted CAs
check CAA (mandatory since 8 September 2017, under
[CA/Browser Forum Ballot 187](https://cabforum.org/2017/03/08/ballot-187-make-caa-checking-mandatory/)): your record names only
the old CA, so this one must refuse. The ACME challenge itself may well succeed — HTTP-01
can pass because DNS points at the new host — which is what makes it confusing: control
is proved, and issuance is still refused, and the host retries indefinitely. CAA is a
separate check from the challenge. Fix: add the new host's CA to the CAA record (or
remove CAA), then let the host retry.

---

## File 06 — Going live

**1. From certificate request to `GET /.env`.**

The host runs ACME against a CA for your name (file 05 §5.7). Before issuing, the CA
submits a pre-certificate to CT logs and receives SCTs, because browsers reject
certificates without them. The log entry — containing your hostname — is now public.
Scanners continuously consume new log entries, extract hostnames, resolve them in DNS
and send probes; studies measure first DNS queries within minutes, and probes within
seconds, of a log entry. `/.env` is a generic probe for accidentally deployed secrets,
sent to every new name regardless of what it hosts.

**2. 400 requests versus 3 visitors.**

They count different things. The CDN counts every HTTP request: each page view is
several requests (HTML, CSS, fonts, images, favicon), and scanners and crawlers that
never run JavaScript are all included. Script-based analytics counts only clients that
executed its beacon — real browsers, minus those with blockers. The 3 is much closer to
"humans who read the page", and probably an undercount.

**3. Traffic attributed to a data centre in a country with no readers.**

(a) Scanners and crawlers running on cloud servers in or near that region — the traffic
genuinely originates there, but it's not people. (b) The CDN routed requests from
elsewhere through that data centre, so it reflects network entry point, not visitor
location. Tell them apart: look at the requests themselves — user agents, paths (probes
for files you don't have), regularity — and compare with a dimension that claims visitor
country, such as script-based analytics. If the script-based view shows almost nothing
from there, it's automated traffic.

**4. `staging-8f3a2.example.com` as protection.**

The moment HTTPS is enabled, the host obtains a certificate naming
`staging-8f3a2.example.com`, it goes into public CT logs, and scanners will be requesting
it within minutes — the unguessable name is published by the act of securing it. A
wildcard certificate (`*.example.com`) keeps the specific name out of CT, which is a
real improvement, but the name still leaks through DNS queries (passive DNS datasets),
referrer headers, shared links and browser history syncing. Not protection. Put the
staging site behind authentication.

**5. Atom feed with XSLT after November 2026.**

(a) Feed readers parse the XML directly and never applied the stylesheet, so subscribers
are unaffected. (b) In Chrome from version 158 (17 November 2026, Stable), the XSLT
transform stops running for ordinary users, so someone clicking the feed link no longer
gets the styled page — they get untransformed XML, presented however Chrome chooses.
Change: replace the XSLT with a CSS stylesheet (still supported) or link visitors to an
HTML page explaining the feed. Leave alone: the feed's URL and its contents, because
changing those is what actually breaks subscribers.

---

## Things to try

1. **Build the same three-page site in two generators** (one binary, one npm). Include a
   list page and a post containing `{{` and `{%` in a code block. Note where each broke.
2. **Look yourself up in CT.** Search a domain you control on a CT log search engine and
   list every hostname that has ever had a certificate. Anything you'd forgotten about?
3. **Watch a new hostname get found.** Add a fresh subdomain to a site you own, let the
   host issue a certificate, and tail the access logs. Note the time to first request and
   the first ten paths requested.
4. **Run RDAP by hand.** `curl -sI https://rdap.org/domain/<name>` for a `.com`, a `.uk`
   and a `.io` name. Which redirect and which don't? Then check the IANA bootstrap file
   for each TLD.
5. **Audit a workflow.** Take any public deploy workflow and list: every `uses:` and
   whether it's SHA-pinned; the effective permissions of each job; which job holds which
   credential; and every download without a checksum.
6. **Break a Pages deploy on purpose.** In a scratch repository, run the Pages workflow
   with Pages disabled and read the exact error; then enable it and re-run.
7. **Read your own HTML.** View source on a published page from each host you've used.
   List every script you didn't write.

---

## Questions worth asking me

- "Walk me through what happens, packet by packet, when someone first visits my new domain."
- "Should this site be static at all? Here's what it needs to do: …"
- "Port this Hugo template to Zola (or vice versa) and tell me what doesn't translate."
- "Audit this GitHub Actions workflow for least privilege and pinning."
- "How would I deploy to a host that supports OIDC, end to end, with no stored secret?"
- "Show me how to set up redirects at each of the hosts in file 03 so moving between them is painless."
- "What does a DNS migration between providers look like without downtime?"
- "How do I add a contact form or comments without giving up the static model?"
- "What's the minimum I'd need to self-host this on a VPS, including updates and monitoring?"
- "Is my mental model right? Here's what I think happens: …"
