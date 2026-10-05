# 7. Glossary

Reference, not reading. Each entry names the file that teaches it.

## Building

| Term | Meaning | Where |
|---|---|---|
| **Static site** | A site whose server returns pre-built files as-is; no code of yours runs per request | 01 §1.2 |
| **Request-time rendering** | Producing HTML on the server for each request (the "dynamic" model) | 01 §1.2 |
| **Build step** | Running the generator once to turn source into output files | 01 §1.3 |
| **Static site generator (SSG)** | The build tool that renders content through templates into an output folder | 01 §1.3 |
| **Front matter** | Metadata (title, date, draft) at the top of a content file, in YAML or TOML | 01 §1.3 |
| **Template engine** | The language templates are written in — Go templates, Tera, Nunjucks, Liquid, `.astro` | 02 §2.3 |
| **Delimiters** | The character pairs a template engine treats as code: `{{ }}`, `{% %}`, `{# #}` | 02 §2.6 |
| **Raw block** | A template region the engine outputs verbatim — `{% raw %}…{% endraw %}` in Tera | 02 §2.6 |
| **Shortcode** | A template snippet callable from inside Markdown content | 02 §2.6 |
| **Islands** | Astro's model: static HTML by default, JS shipped only for components marked `client:*` | 02 §2.3 |
| **Output folder** | The built site — `public/`, `dist/`, `_site/` — the only thing deployed | 01 §1.3 |
| **Slug** | The URL segment naming a page, e.g. `first-post` | 01 §1.4 |
| **Clean URL** | A URL without a file extension, made by writing `slug/index.html` | 01 §1.4 |
| **Directory index** | The server convention of serving `index.html` when a directory is requested | 01 §1.4 |

## Hosting

| Term | Meaning | Where |
|---|---|---|
| **Host** | Stores your files and answers HTTP requests for them | 01 §1.5 |
| **CDN** | A network of edge data centres serving cached copies near visitors | 01 §1.5 |
| **Hosted blog platform** | A service that owns the editor, templates, build and hosting; you supply words | 03 §3.3 |
| **Workers static assets** | Cloudflare's current recommended way to serve static files | 03 §3.5 |
| **Soft limit** | A usage limit the provider may enforce by contacting you rather than by blocking | 03 §3.4 |
| **Lock-in** | Features that make leaving a host a migration rather than a re-upload | 03 §3.7 |
| **Automatic HTTPS** | A server or host obtaining and renewing certificates itself via ACME | 03 §3.6 |

## CI and deploying

| Term | Meaning | Where |
|---|---|---|
| **CI** | Continuous integration — automated jobs triggered by pushes | 04 §4.1 |
| **Artefact** | The packaged output of the build job, deployed unchanged | 04 §4.2 |
| **`GITHUB_TOKEN`** | The per-job token GitHub Actions issues, scoped by `permissions` | 04 §4.3 |
| **Least privilege** | Granting each job only the permissions it needs | 04 §4.3 |
| **OIDC token** | A short-lived signed statement of which repo/ref/workflow a job is, accepted by a host in place of a secret | 04 §4.4 |
| **Scoped API token** | A stored credential limited to specific actions and resources | 04 §4.4 |
| **Pinning** | Fixing a dependency to an exact version, checksum or commit | 04 §4.6 |
| **Checksum (SHA-256)** | A digest of a file's bytes; a mismatch means it's not the file you chose | 04 §4.6 |
| **Lockfile** | A file recording the exact versions of every npm dependency | 04 §4.6 |
| **SHA-pinned action** | `uses: owner/action@<40-char commit SHA>` — immutable, unlike a tag | 04 §4.7 |
| **Concurrency group** | A named group in which only one workflow run executes at a time | 04 §4.8 |
| **Environment** (GitHub) | A named deploy target with optional protection rules | 04 §4.5 |

## Domains and DNS

| Term | Meaning | Where |
|---|---|---|
| **Registry** | Operator of a TLD; keeps the authoritative registrations | 01 §1.5 |
| **Registrar** | Sells registrations and records them with the registry | 01 §1.5 |
| **TLD / gTLD / ccTLD** | Top-level domain; generic (`.com`, `.dev`) / country-code (`.uk`, `.io`) | 05 §5.2 |
| **Apex** | The bare domain, `example.com`; holds the zone's SOA and NS records | 05 §5.6 |
| **Nameservers** | The DNS servers authoritative for your domain; set at the registrar | 05 §5.6 |
| **WHOIS** | The legacy, unstructured registration lookup protocol | 05 §5.2 |
| **RDAP** | Registration Data Access Protocol — registration lookups over HTTPS/JSON | 05 §5.2 |
| **Bootstrap** (RDAP) | IANA's file mapping TLDs to their RDAP servers | 05 §5.2 |
| **Redaction** | Withholding personal fields from public registration lookups | 05 §5.4 |
| **Privacy / proxy service** | A service whose details replace yours in the registration | 05 §5.4 |
| **Renewal price** | What a registration costs each year after the first | 05 §5.3 |
| **A / AAAA** | DNS records mapping a name to an IPv4 / IPv6 address | 05 §5.6 |
| **CNAME** | A DNS record aliasing one name to another; can't coexist with other records | 05 §5.6 |
| **ALIAS / ANAME / CNAME flattening** | Provider-specific ways to alias the apex, answered as A/AAAA | 05 §5.6 |
| **HTTPS / SVCB record** | Standard record type (RFC 9460) with an AliasMode usable at the apex | 05 §5.6 |
| **Dangling CNAME** | A CNAME whose target no longer resolves | 05 §5.6 |
| **CAA** | DNS record naming the CAs allowed to issue for a domain | 05 §5.7 |

## Certificates and going live

| Term | Meaning | Where |
|---|---|---|
| **HSTS** | Header telling browsers to use only HTTPS for a host | 05 §5.5 |
| **HSTS preload list** | Browser-built-in HSTS list; `.dev`, `.app`, `.page` are on it as whole TLDs | 05 §5.5 |
| **ACME** | The protocol hosts use to obtain certificates automatically | 05 §5.7 |
| **HTTP-01 / DNS-01 / TLS-ALPN-01** | ACME challenge types proving control of a name | 05 §5.7 |
| **Certificate Transparency (CT)** | Public, append-only logs of every publicly trusted certificate | 06 §6.2 |
| **SCT** | Signed Certificate Timestamp — a CT log's receipt, required by browsers | 06 §6.2 |
| **Wildcard certificate** | Covers `*.example.com`; keeps individual subdomains out of CT logs | 06 §6.2 |
| **Request-based analytics** | Counting from server/CDN logs; includes bots | 06 §6.3 |
| **Script-based analytics** | Counting via a JS beacon; misses non-JS clients | 06 §6.3 |
| **Verified bot** | A crawler a platform has confirmed is what it claims | 06 §6.3 |
| **Feed (RSS / Atom)** | Machine-readable list of posts for feed readers | 06 §6.5 |
| **XSLT** | XML transformation language, formerly used to style feeds in browsers; being removed | 06 §6.5 |
