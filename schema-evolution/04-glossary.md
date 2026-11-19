# 4. Glossary

Reference, not reading. Each term points at the file that teaches it.

| Term | Meaning | Where |
|---|---|---|
| **Absent / null / zero** | Three different states of a field in a payload: the key wasn't sent; the key was sent with the value `null`; the key was sent with a real value that happens to be zero. Most plain types collapse them into one | [02](02-consuming-payloads-you-dont-control.md) |
| **Backward compatibility** | New readers can read data written by old writers. Confluent's BACKWARD mode: "consumers using the new schema can read data produced with the last schema" | [03](03-expand-and-contract.md) |
| **BACKWARD / FORWARD / FULL** | Confluent Schema Registry's compatibility modes. BACKWARD: upgrade consumers first. FORWARD: upgrade producers first. FULL: either order | [03](03-expand-and-contract.md) |
| **Contract (phase)** | The last phase of expand and contract: remove the old field once no reader needs it, and keep its number or name retired | [03](03-expand-and-contract.md) |
| **Dual-write** | Writers fill both the old and the new field during a migration, from the same source of truth | [03](03-expand-and-contract.md) |
| **Expand and contract** | Also called **parallel change**. Make an incompatible change as add-new, migrate readers, remove-old, so no single release breaks a reader | [03](03-expand-and-contract.md) |
| **Explicit presence** | A field whose generated API records whether it was set, so zero and "not set" differ. In proto3, scalar fields marked `optional` | [02](02-consuming-payloads-you-dont-control.md) |
| **Facts, not verdicts** | An event carries what happened (terminal state, error codes) and lets the consumer that owns a policy apply it, rather than carrying a decision computed by the producer | [03](03-expand-and-contract.md) |
| **Fail closed** | When a reader meets a value it doesn't recognise, it takes the outcome that does nothing irreversible — for anything gating an action, "not success" | [02](02-consuming-payloads-you-dont-control.md) |
| **Fallback counter** | A metric incremented whenever a reader takes the old field because the new one is absent. Zero for long enough means the old field can stop being written | [03](03-expand-and-contract.md) |
| **Field number** | The integer that identifies a field on the wire in Protocol Buffers or Cap'n Proto. Permanent once the message is in use | [01](01-the-installed-client-problem.md) |
| **Forward compatibility** | Old readers can read data written by new writers, which requires them to skip what they don't know. Confluent's FORWARD mode: "data produced with a new schema can be read by consumers using the last schema" | [03](03-expand-and-contract.md) |
| **Implicit presence** | A field whose generated API stores only the value. In proto3, a plain scalar: "not serialized if they contain their default value", so zero and absent are the same bytes | [02](02-consuming-payloads-you-dont-control.md) |
| **Installed client** | Software running on machines you don't control and can't upgrade on your schedule — the reason old readers live for years | [01](01-the-installed-client-problem.md) |
| **Open enum** | An enum whose decoder keeps values it doesn't recognise (proto3 stores them as the underlying integer) rather than rejecting them | [02](02-consuming-payloads-you-dont-control.md) |
| **Reader audit** | Finding every reader of a field — rules, templates, frontends, stored columns, allowlists, dashboards — before it stops being written | [03](03-expand-and-contract.md) |
| **Required field** | A field a reader insists on. Adding the requirement breaks old writers; removing it is safe once every reader that enforces it has relaxed | [01](01-the-installed-client-problem.md) |
| **Reserved** | Protocol Buffers' way of retiring a field number or name so it can't be reused | [01](01-the-installed-client-problem.md) |
| **Robustness principle** | "Be conservative in what you send, and liberal in what you accept." RFC 9413 argues its tolerant half causes long-term harm when applied to faulty input | [02](02-consuming-payloads-you-dont-control.md) |
| **Round-trip test (on the bytes)** | Serialise a value and assert on the encoded keys and types, rather than decoding with the same struct that encoded it | [01](01-the-installed-client-problem.md) |
| **Tolerant reader** | Fowler's pattern: read only the fields you need and ignore the rest, so the provider can change what you don't use | [02](02-consuming-payloads-you-dont-control.md) |
| **Tombstone** | A retired field left in the schema, marked reserved or commented, so its number and name are never reused | [01](01-the-installed-client-problem.md) |
| **Virtuous intolerance** | RFC 9413's name for generating fatal errors on unspecified conditions instead of attempting recovery | [02](02-consuming-payloads-you-dont-control.md) |
| **Zero value** | The value a type has when nothing has been assigned: `0`, `""`, `false`, `nil`. In Go, what a struct field keeps when its JSON key is missing | [02](02-consuming-payloads-you-dont-control.md) |
