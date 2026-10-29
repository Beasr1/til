# Blog topics

- [ ] **The card forgot its challenge** — NFC mutual auth passed on a dip and failed on a tap from home: the card lost its challenge while the link sat idle through the backend round trip. A keep-alive every 100 ms fixed it, and then the twist: the culprit was the reader's old firmware wiping idle cards, and an update fixed it without the keep-alive.
- [ ] **Your office network is a test fixture** — a timing bug that never showed in the office, because the office's direct tunnel kept the round trip short.
- [ ] **One chip, two ATRs** — a card that read on contact was "not recognised" when tapped, because the table had its contact ATR but not the contactless one PC/SC synthesises.
- [ ] **The test vector in the appendix** — a feature-flagged secure-messaging path that no CI built had rotted, and the spec's worked example caught the padding bug left in it.
- [ ] **Adding a field to a format you can't change** — one optional field in a frozen host–guest wire format, tested old-to-new and new-to-old.
- [ ] **Audit the right copy of the source** — a port comparison concluded the predecessor lacked a feature it had, because it read the older of two copies.
