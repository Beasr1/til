# 3. Signing is identity

This is the crux of the whole course. If you take one chapter away, take this one: it is the
reason a static attack is contained, and the precise shape of what it *doesn't* contain.

## The problem

File 02 left us with an uncomfortable fact: an APK is a ZIP, and anyone can unzip it, change
`classes.dex`, and zip it back. If the system installed whatever ZIP you handed it, "modify
any app" would be a five-minute job and there'd be no such thing as app integrity.

So the system needs to answer one question at install and update time: **is this APK from
the same source as it claims, and the same source as last time?** Not "is it safe" — that's
undecidable. Just: *same author, consistently*. That question is answered by a digital
signature.

## 3.1 Every APK is signed, and the signature is checked

Android has required APKs to be signed since the beginning. The developer signs with the
**private key** of a keypair; the APK carries the matching **certificate** (the public key).
At install the system verifies the signature and records *which certificate* signed this
package ([source.android.com/docs/security/features/apksigning](https://source.android.com/docs/security/features/apksigning)).

The private key stays with the developer. It is **not in the APK** — only the certificate is.
That asymmetry is the entire game: an attacker who edits your APK has your certificate (it's
right there) but *not* your private key, so they cannot produce a signature that verifies
against your certificate. They can only re-sign with a key of their own.

## 3.2 The signature schemes, v1 to v4

The scheme has been revised as Android learned what the earlier versions failed to protect.
You will see all of these named by tools and in assessment reports, so know what each covers.

| Scheme | Since | What it signs | The point |
|---|---|---|---|
| **v1** (JAR signing) | the beginning | Each file's digest, listed in `META-INF/` (`MANIFEST.MF`, `CERT.SF`, `CERT.RSA`) | Inherited from signed JARs. Signs file *contents* but **not** the ZIP structure around them |
| **v2** | Android 7.0 | The **whole APK** as one blob, in an "APK Signing Block" inserted into the ZIP | Any change to any byte outside the signing block invalidates it. Faster to verify |
| **v3** | Android 9 | Same as v2, plus a **signing lineage** | Adds **key rotation**: prove the new key is endorsed by the old one, so you can change keys without becoming a new app |
| **v3.1** | Android 13 | A refinement of v3's rotation | Later tweak to rotation targeting; the detail isn't load-bearing here |
| **v4** | Android 11 | A **Merkle hash tree** over the APK (fs-verity structure), stored *beside* the APK as `<name>.apk.idsig` | Enables streaming / incremental install. **Requires a complementary v2 or v3 signature** — it can't stand alone |

Two things to notice.

**v1 signs contents, not structure.** Because v1 only lists per-file digests, it leaves ZIP
metadata and some structure unprotected — which is the historical reason v2 exists. v2 treats
the APK as a single blob and hashes all of it
([source.android.com/docs/security/features/apksigning/v2](https://source.android.com/docs/security/features/apksigning)).

**v4 is not a replacement.** It's an *additional* signature stored in a separate `.idsig`
file, used so a device can start running an app while it's still downloading (incremental
install). AOSP: "A v4 signature requires a complementary v2 or v3 signature."
([source.android.com/docs/security/features/apksigning/v4](https://source.android.com/docs/security/features/apksigning/v4)).

```
v1:  signs each entry's digest     ┌───────────────┐
     → ZIP structure unprotected   │ classes.dex ✓ │  (contents hashed, listed in META-INF)
                                    │ res/…       ✓ │
                                    └───────────────┘

v2+: signs the whole file          ┌───────────────────────────────┐
     → one byte changes anywhere    │▓▓▓▓▓▓ entire APK bytes ▓▓▓▓▓▓▓│ → one hash, in the
       outside the block → invalid  └───────────────────────────────┘   APK Signing Block
```

## 3.3 Why "signature" equals "identity"

Signing would be a weak defence if it only meant "this file is internally consistent". Its
force comes from what the system *binds* to the recorded certificate. Three bindings matter:

- **Update continuity.** An update is accepted only if it's signed by the same key as the
  installed app (or a key in its v3 rotation lineage). A differently signed APK with the same
  package name is **rejected as an update** — the system won't install it over the original
  ([developer.android.com/studio/publish/app-signing](https://developer.android.com/studio/publish/app-signing)).
- **Signature permissions.** Some permissions are granted only to apps signed with the *same*
  certificate as the app that defined them. Two apps can share data or components only if
  they share a signing identity.
- **Data ownership.** The original app's `/data/data` directory belongs to the original
  install. A re-signed copy is a different install and gets its own empty directory.

Put those together and the consequence is sharp:

> **An attacker can edit your app, but the moment they re-sign it with their own key it
> becomes — to the system — a *different app*. It cannot update over the real one. It cannot
> read the real one's data. It cannot claim the real one's signature permissions. Its
> certificate no longer matches the one your server (or Google Play) remembers.**

That is the containment on static attacks, and it's why file 02's "it's just a ZIP" doesn't
end the story.

> **Teacher's aside.** "Signed" does not mean "trustworthy", and it does not by itself mean
> "this is the Google Play build". A signature proves *continuity of authorship* — same key
> as before — not virtue. Anyone can generate a keypair and sign an app; self-signed is the
> norm, not the exception, and there's no CA deciding who may sign an Android app. The trust
> is more like trust-on-first-use: the system remembers the cert it first saw for a package
> and insists every later version match it. The question signing answers is "same author?",
> never "good author?".

## 3.4 The tools: apksigner and zipalign

Two official tools show up whenever signing does, and their *ordering* trips people up.

**`apksigner`** (ships in the Android SDK build-tools) applies and verifies v1–v4 signatures.
It can enable or disable each scheme independently, and `apksigner verify -v` prints which
schemes verified ([developer.android.com/tools/apksigner](https://developer.android.com/tools/apksigner)).

**`zipalign`** aligns the APK's uncompressed entries to 4-byte boundaries so the OS can
`mmap` them directly, which saves memory at runtime
([developer.android.com/tools/zipalign](https://developer.android.com/tools/zipalign)).

> ⚠️ `zipalign` must run **before** v2+ signing, never after. v2+ signs the whole file, so
> realigning bytes afterwards changes the file and *breaks the signature you just applied*.
> The correct order is: build → zipalign → sign. Doing it backwards produces an APK that
> fails verification, and it's a classic repackaging-pipeline mistake (file 07).

## 3.5 The limit of what signing does

Be precise about the boundary, because it's where file 10 becomes necessary.

Signing stops a modified app from **masquerading** as yours: updating over it, inheriting its
data and permissions, being recognised by a verifier that pinned your certificate. What
signing does **not** do is stop an attacker from installing and running their re-signed copy
*as a new, separate app on a device they control*. If all they want is to run a cracked
version for themselves, the signature didn't prevent that — it only made the copy a distinct
identity.

So on the *client*, signing bounds the damage; it doesn't prevent the modification. The thing
that lets a *server* refuse to deal with the re-signed copy is checking the certificate
server-side — which is exactly what Play Integrity's app-integrity verdict does (file 10):
it reports whether the running app's certificate matches the one Google Play distributes.
Signing creates the identity; attestation is what reports that identity to someone off the
device who can act on it.

## Check yourself

1. An attacker changes one method in `classes.dex` and re-zips the APK without touching
   anything else. Under v1 signing versus v2 signing, what specifically catches the change
   in each case — and why does v1's design leave more unprotected?
2. The attacker re-signs the modified APK with a freshly generated key so it verifies
   cleanly. They install it. Name three things this app now *cannot* do that the original
   could, and say which binding causes each.
3. A developer loses their signing key and signs the next release with a new one. Users can't
   update. Which scheme was meant to prevent exactly this, and what does it require the
   developer to have done in advance?
4. Your build pipeline runs `apksigner` and then `zipalign`. Installs start failing
   signature verification. What's wrong, and what's the fix?
5. "Our app is signed, so it can't be tampered with." Separate the true half of that
   sentence from the false half, in terms of *masquerading* versus *running a modified copy*.
6. A re-signed copy of your app talks to your server. The signature check on the device
   passed (it's a valid signature — just the attacker's). What could the *server* check that
   the device-side signature check structurally cannot, and why does that check have to live
   off the device?
