# 01 — The support plane (consensual, non-E2EE, disjoint from content)

> **Support plane, disjoint from content.** This service receives in-app bug reports and runs the central helpdesk. It is the **one deliberate, consensual exception** to Nyxite's zero-knowledge model: a place a user *voluntarily* discloses readable information to the maintainer. A report carries **no content key and no content-plane ciphertext**; the content-plane guarantee (the server never holds a content key) is **untouched**. Reports are **not** E2EE — the maintainer reads them — and the user is told so before sending. It is **vendor infrastructure**, central and maintainer-run, **not** part of a self-hosted deployment. Concrete wire schemas are **[P]**.

## 1.1 Model (SUP-1, SUP-5)

- Nyxite's **content plane** is zero-knowledge everywhere: the server stores only ciphertext, holds no content key, and cannot read notes, names, ink, or collaboration traffic. **Nothing in this document changes that.**
- The **support plane** is a *separate* plane, entered **only** by an explicit user action (tapping "Report a bug" and confirming the non-E2EE notice). It is never automatic and never reads or references content.
- **Not E2EE, stored server-readable (SUP-8).** Reports are readable by the maintainer and stored **in plaintext** on this service's isolated database + blob storage. The protections are **consent + client-side redaction + retention limits**, not encryption. The disk-theft exposure is explicit and accepted.
- **Central, maintainer-run by default (SUP-5).** By default there is exactly **one** helpdesk — the maintainer's — never in the base self-hosted stack, no base-Compose entry, no operator deploys it (mirroring `NyxiteLicense`). The **one exception** is a per-operator **opt-in self-hosted desk** for enterprise operators — **resolved as cluster SUP-10–SUP-13** (2026-07-12), specified in §1.6.

## 1.2 Disjointness — the invariants this plane must satisfy [P]

The support plane is safe only if it is provably separate from content. These are hard invariants, enforceable by review and test:

1. **No content key** ever enters a report, the submission payload, this service, or its database.
2. **No content-plane ciphertext** (file blobs, CRDT updates, wrapped keys) is attached to or referenced by a report. A screenshot is a **user-produced, redacted raster** — not a content blob — and is the only image this plane accepts.
3. **No content-plane database or blob store** is reachable from this service. Its PostgreSQL + blob storage are isolated vendor infra (05).
4. **Opaque identity only.** A report references its submitter by an **opaque account reference** minted by the originating server (02), never by decrypting or resolving user identity on the content plane.
5. **Explicit-action gate.** No code path creates or transmits a report without a user having confirmed the non-E2EE + destination notice (§1.3).

## 1.3 Consent, destination notice & GDPR (SUP-1, SUP-6)

- **Before** a report is transmitted, the client shows a clear notice: *"Unlike your files, this report is **not** end-to-end encrypted. It is sent to the Nyxite maintainer to help fix the problem. Only include information you are comfortable sharing."* — plus a link to what is collected (the diagnostic envelope, 02).
- The **destination** is named so a user on a **third-party self-hosted instance** understands where the report goes: by default the **Nyxite maintainer**, not their own operator — or, on an instance whose operator opted into their **own desk** (§1.6, SUP-13), the **operator** (with an upstream-sharing line for confirmed bugs). The consent copy reflects whichever applies.
- **GDPR.** This plane processes user-disclosed PII (free text, redacted screenshots, and — for guests — an email). A disclosure covers what is collected, why (bug diagnosis/support), the recipient (the maintainer), retention (05), and the right to erasure. Lawful basis is the user's explicit consent captured at submit time.
- **No silent collection.** The diagnostic envelope is **user-reviewable and editable** before send (02); nothing is attached that the user cannot see.

## 1.4 Boundary with the server & clients

| Owner | Responsibility |
|---|---|
| **Clients** (desktop/web/android) | "Report a bug" UI, screenshot capture + destructive redaction, diagnostic envelope, consent notice, "My tickets" view (02, 03) |
| **`NyxiteServer`** (self-hosted) | **Authenticating relay** only — authenticates the submitter, mints the opaque user ref + instance fingerprint, forwards to this service; **stores no ticket** (02) |
| **`NyxiteSupport`** (this service) | Ticket storage + lifecycle, REST API + webhooks, operator UI, retention (03, 04, 05) |

## 1.5 Availability & routing (SUP-9, corrected 2026-07-12)

- **One central desk in v1.** By default one desk; all reports route to the maintainer (unless the operator opted into their own desk, §1.6).
- **Every** instance — the maintainer's official instance(s) **and** third-party self-hosted instances — surfaces the in-app "Report a bug" + "My tickets" UI and routes tickets to the central desk. Routing is **not silent**: the SUP-1 non-E2EE + "goes to the Nyxite maintainer" consent notice (§1.3) is shown before every send. *(This **corrects** the earlier "third-party instances have no reporting surface" wording, which contradicted SUP-1 and is withdrawn.)* Servers still advertise a `support.enabled` capability so an operator may switch the surface off entirely.

## 1.6 Self-hostable helpdesk — opt-in per-operator desk (SUP-10–SUP-13, resolved 2026-07-12)

An **enterprise** operator may opt into running their **own `NyxiteSupport` desk**, so their instance's tickets land **locally** instead of at the maintainer's central desk; only tickets **confirmed as bugs** route upstream. This reverses the SUP-5/SUP-9 central-only posture **for opted-in operators only** — the default self-hosted stack is untouched. Canonical in `docs/OPEN-DECISIONS.md`.

- **Deployment (SUP-10).** Ships as a drop-in **`docker-compose.override.yml`** the operator adds to their stack (Compose auto-merges it). The base stack stays clean and never names the helpdesk — the opt-in *is* the presence of the file — and the override patches the `server` service to relay to the **local** desk. Not a base-stack Compose profile.
- **Enterprise-gated (SUP-11).** Running your own desk is an **enterprise feature** (L-3 catalog); the central helpdesk stays free. Enforced at **runtime by the entitlement lease** (L-2/L-4), not the file's presence — the desk **refuses to function without a valid lease** and degrades/locks with the other enterprise features on lapse.
- **Bug routing (SUP-12).** A ticket is armed for upstream routing by an explicit operator **"confirm as bug"** action **or** the `bug` tag (manual or via a SUP-4 webhook / local-AI worker), then **auto-routes after the operator-configured age window**. **Soft-mandatory** — non-disable-able, but the operator controls what is confirmed (confirm nothing → route nothing); no hard "disable upstream" switch.
- **Upstream payload, auth & ownership (SUP-13).** A routed bug sends the **full bundle** (text + client-redacted screenshot + envelope), tagged with instance fingerprint + opaque user ref via the SUP-7 relay run **desk→central**, authenticated by the operator's existing **license token + instance fingerprint** (L-5/L-7 — no new credential). The reporter is told at submission that confirmed bugs may be shared upstream. On routing the ticket is **handed off to the maintainer** (who owns the thread and may reply); the operator's desk **relays replies back down** to the reporter's "My tickets" / guest link. Consent copy on a self-hosted desk reads **"goes to your instance operator"** + the upstream-sharing line. The operator runs its own SUP-8 plaintext plane and is the GDPR controller for it (retention + right-to-erasure).
