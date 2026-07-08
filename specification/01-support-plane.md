# 01 — The support plane (consensual, non-E2EE, disjoint from content)

> **Support plane, disjoint from content.** This service receives in-app bug reports and runs the central helpdesk. It is the **one deliberate, consensual exception** to Nyxite's zero-knowledge model: a place a user *voluntarily* discloses readable information to the maintainer. A report carries **no content key and no content-plane ciphertext**; the content-plane guarantee (the server never holds a content key) is **untouched**. Reports are **not** E2EE — the maintainer reads them — and the user is told so before sending. It is **vendor infrastructure**, central and maintainer-run, **not** part of a self-hosted deployment. Concrete wire schemas are **[P]**.

## 1.1 Model (SUP-1, SUP-5)

- Nyxite's **content plane** is zero-knowledge everywhere: the server stores only ciphertext, holds no content key, and cannot read notes, names, ink, or collaboration traffic. **Nothing in this document changes that.**
- The **support plane** is a *separate* plane, entered **only** by an explicit user action (tapping "Report a bug" and confirming the non-E2EE notice). It is never automatic and never reads or references content.
- **Not E2EE, stored server-readable (SUP-8).** Reports are readable by the maintainer and stored **in plaintext** on this service's isolated database + blob storage. The protections are **consent + client-side redaction + retention limits**, not encryption. The disk-theft exposure is explicit and accepted.
- **Central, maintainer-run only (SUP-5).** There is exactly **one** helpdesk — the maintainer's. This service is never in the self-hosted stack, has no Compose entry, and no operator deploys it (mirroring `NyxiteLicense`). A future per-operator opt-in mode is a backlog item, out of scope here.

## 1.2 Disjointness — the invariants this plane must satisfy [P]

The support plane is safe only if it is provably separate from content. These are hard invariants, enforceable by review and test:

1. **No content key** ever enters a report, the submission payload, this service, or its database.
2. **No content-plane ciphertext** (file blobs, CRDT updates, wrapped keys) is attached to or referenced by a report. A screenshot is a **user-produced, redacted raster** — not a content blob — and is the only image this plane accepts.
3. **No content-plane database or blob store** is reachable from this service. Its PostgreSQL + blob storage are isolated vendor infra (05).
4. **Opaque identity only.** A report references its submitter by an **opaque account reference** minted by the originating server (02), never by decrypting or resolving user identity on the content plane.
5. **Explicit-action gate.** No code path creates or transmits a report without a user having confirmed the non-E2EE + destination notice (§1.3).

## 1.3 Consent, destination notice & GDPR (SUP-1, SUP-6)

- **Before** a report is transmitted, the client shows a clear notice: *"Unlike your files, this report is **not** end-to-end encrypted. It is sent to the Nyxite maintainer to help fix the problem. Only include information you are comfortable sharing."* — plus a link to what is collected (the diagnostic envelope, 02).
- The **destination** is named because a user on a **third-party self-hosted instance** must understand the report goes to the **Nyxite maintainer**, not their own operator.
- **GDPR.** This plane processes user-disclosed PII (free text, redacted screenshots, and — for guests — an email). A disclosure covers what is collected, why (bug diagnosis/support), the recipient (the maintainer), retention (05), and the right to erasure. Lawful basis is the user's explicit consent captured at submit time.
- **No silent collection.** The diagnostic envelope is **user-reviewable and editable** before send (02); nothing is attached that the user cannot see.

## 1.4 Boundary with the server & clients

| Owner | Responsibility |
|---|---|
| **Clients** (desktop/web/android) | "Report a bug" UI, screenshot capture + destructive redaction, diagnostic envelope, consent notice, "My tickets" view (02, 03) |
| **`NyxiteServer`** (self-hosted) | **Authenticating relay** only — authenticates the submitter, mints the opaque user ref + instance fingerprint, forwards to this service; **stores no ticket** (02) |
| **`NyxiteSupport`** (this service) | Ticket storage + lifecycle, REST API + webhooks, operator UI, retention (03, 04, 05) |

## 1.5 Availability & routing (SUP-9)

- **Central-only in v1.** One desk; all reports route to the maintainer.
- The in-app reporting surface is enabled on the **maintainer's official instance(s)**; a third-party self-hosted instance has **no** reporting surface, so its users' reports are **not** silently routed upstream. Servers signal whether reporting is enabled to their clients (02); where disabled, the "Report a bug" and "My tickets" surfaces are simply absent.
- A per-operator **self-hostable opt-in helpdesk** (tickets local, only confirmed bugs routed upstream after a configurable age) is a **deferred backlog item** — see `docs/OPEN-DECISIONS.md`.
