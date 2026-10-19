# 5. Glossary

Reference, not reading. The number in parentheses is the chapter where the term is motivated.

## The objects

| Term | Meaning |
|---|---|
| **Certificate** | a signed claim binding a public key to a name, made by an issuer, checkable offline by anyone (1) |
| **X.509** | the certificate format everything here uses; profiled for the internet by RFC 5280 (1) |
| **End-entity certificate**, **leaf** | a certificate whose key may *not* sign certificates. The bottom of a chain (1) |
| **CA certificate** | one whose `basicConstraints` says `cA: TRUE` — its key may sign certificates (1) |
| **Intermediate**, **subordinate CA** | a CA certificate itself issued by another CA (1) |
| **Root** | a self-signed CA certificate; by convention the top of a hierarchy (1) |
| **Cross-certificate** | a certificate one CA issues for *another CA's existing public key*, giving that key a second path upward (2) |
| **CRL** | Certificate Revocation List — a signed list of revoked serials you download (2) |

## The roles

| Term | Meaning |
|---|---|
| **Subject** | who or what the certificate is *about*; holds the matching private key (1) |
| **Issuer** | whoever signed the certificate. A **role in one certificate**, not a kind of company (1) |
| **CA** — certificate authority | mechanically, **a keypair** that signs certificates, plus a certificate permitting it to. The organisation around it is why anyone trusted the key, and participates in no verification step (1) |
| **Relying party** | the verifier. Was not present at issuance, and is the only party with a decision to make (1) |
| **Trust anchor** | the key a verifier *starts from*, because it was configured. Its own signature is never checked (1) |

## The fields

| Term | Meaning |
|---|---|
| **DN** — Distinguished Name | the structured name in `subject` / `issuer`. A label the issuer chose; forgeable (1) |
| **SAN** — `subjectAltName` | where machine names (DNS, IP, email) live. Replaced `commonName` for that job (1, 3) |
| **AKI** — `authorityKeyIdentifier` | identifies the issuer's **key**, for when one issuer name has had several. A hint (1) |
| **SKI** — `subjectKeyIdentifier` | this certificate's own key identifier — what a child's AKI points at (1) |
| **`basicConstraints`** | `cA` — may this key sign certificates? — and `pathLenConstraint`, how many CAs may sit below (1) |
| **`keyUsage`** | what the key may be used for; `keyCertSign` is the bit needed to issue certificates (1) |
| **`pathLenConstraint`** | maximum number of further CA certificates permitted below this one (1, 2) |
| **Serial number** | unique **per issuer**, not globally. Two issuers can both have serial `1` (1) |
| **AIA** — `authorityInformationAccess` | where to *fetch* the issuer's certificate, or an OCSP responder. Helps path building (2) |

## The two jobs

| Term | Meaning |
|---|---|
| **Certification path**, **chain** | an ordered list of certificates, each issued by the previous, the first issued by the anchor (1) |
| **Path building** | *finding* such a list, from a leaf and a bag of candidates. A **search**. Informational only — RFC 4158 — so implementations differ (2) |
| **Path validation** | *checking* a given list. An **algorithm** — RFC 5280 §6.1 — so implementations agree (2) |
| **Backtracking** | retrying an alternative candidate issuer when one branch dead-ends. Not guaranteed; §2.4 shows a real verifier declining to (2) |
| **Revocation** | whether a still-unexpired certificate has been withdrawn. A separate pass, with its **own** trust anchor (2) |

## Encoding

| Term | Meaning |
|---|---|
| **ASN.1** | the type language certificates are defined in (1) |
| **BER** | encoding rules permitting several encodings per value (1) |
| **DER** | the subset permitting exactly **one**. Required, because a signature covers bytes (1) |
| **`tbsCertificate`** | "to be signed" — the part of a certificate the signature covers (1) |
| **PEM** | base64-armoured DER with `-----BEGIN …-----` lines. A container, not a format (2) |

## Operating a store (file 4)

| Term | Meaning |
|---|---|
| **Trust store** | the set of anchors a verifier starts from. A file (or keychain) an application goes looking for — **not** a protocol artefact, which is why three programs on one machine can hold three different sets (4) |
| **Anchor set** | what you actually mount. Intermediates are *not* in it — they arrive in the chain or via AIA (4) |
| **`ca-certificates`** | the OS package that makes the web PKI trust store self-updating. Absent from minimal base images, which therefore trust **nothing** (4) |
| **Root programme** | the body deciding which roots go in a bundle (Mozilla's is the one most bundles derive from) (4) |
| **Signed bundle** | a trust-anchor distribution that is itself signed — ICAO master lists, the eIDAS LOTL. Makes re-fetching verifiable rather than an act of faith (4) |
| **`NextUpdate`** | an ETSI TS 119 612 trusted list's promise of when it will next be refreshed; bounded at 6 months (§5.3.15) (4) |
| **Fail closed / fail open** | on a failed anchor fetch, keep the last good set and alert (closed); on a failed revocation fetch, a deliberate policy choice either way (4) |
| **Anchor expiry policy** | whether a verifier checks an anchor's `notAfter`. **Implementations disagree** — OpenSSL does, Android deliberately doesn't (1, 4) |
| **Issuer allow-list** | a post-validation assertion that the intermediate in the **built** path is one you approved. Keeps full validation to the root *and* adds explicit issuer approval (4) |
| **SPKI pin** | `sha256` of the DER `subjectPublicKeyInfo` — an identifier of the **key**, so it survives certificate renewal and cross-signing. The right thing to pin (4) |
| **Churn rate** | how often an allow-list's entries change. Decides *where the list lives* — an image config file at one extreme, a datastore with an admin path at the other (4) |
| **Version floor** | a minimum-version check (`candidate >= expected`) standing in for an exact version list. Absorbs the fastest-churning dimension of an app identity allow-list, and doubles as an anti-downgrade control (4) |
| **Subject hash collision** | two CA certificates with the same subject DN and different keys. Real, and why OpenSSL's hashed CA directories append `.0`, `.1` … (1, 4) |
| **Built path** | the chain the verifier actually constructed and returned — `X509_STORE_CTX_get0_chain`, or Go's `Verify` return. The thing to assert on; the peer's supplied bundle is a hint (2, 4) |

## Per-ecosystem (file 3)

| Term | Meaning |
|---|---|
| **CT** — Certificate Transparency | public append-only logs of issued certificates. Makes misissuance *findable*, not impossible |
| **CSCA** | Country Signing Certification Authority — the trust anchor for a state's travel documents |
| **Master list** | a signed bundle of CSCA certificates, distributed so verifiers can obtain anchors out of band |
| **LOTL** | the EU's List of Trusted Lists — a signed meta-list pointing at each member state's trusted list |
| **Attestation extension** | OID `1.3.6.1.4.1.11129.2.1.17`, where Android attestation puts its entire payload |
| **Signing lineage** | Android v3's proof that a new app signing key was endorsed by the old one. A rotation proof, **not** a certification path |
| **Pinning** | storing a key or its digest and comparing directly, with no chain. The anchor mechanism for APK signing |
