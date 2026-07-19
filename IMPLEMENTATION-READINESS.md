# NyxiteSupport — Implementation Readiness Assessment

**Question:** Can NyxiteSupport be **implemented as a faithful build today** using only (a) this component's `specification/` (01–05) + `FEATURES.md` and (b) the shared `Nyxite` info repo (`docs/OPEN-DECISIONS.md`, `features/support.md`, `implementation/phase-4.6-support-helpdesk.md`)?

**Verdict: NO — not as a faithful build today.**

The support specification is architecturally complete and internally consistent. The *what* and *why* are **fully ratified**: decisions **SUP-1…SUP-13** (the consensual non-E2EE plane, disjointness invariants, central-by-default routing, the authenticating relay, guest flow, API-first + webhooks, plaintext-with-retention posture, and the opt-in self-hosted desk) plus the design-system decisions **DS-1…DS-3** are all resolved in `docs/OPEN-DECISIONS.md`. The five specification chapters express those decisions cleanly and the invariants (01 §1.2) are review- and test-enforceable.

But this is the **inverse** of the server's situation. The server repo ships a genuinely complete build spec (00–15) whose blockers are a handful of unwritten subsystems; here the **architecture is settled and almost none of the concrete implementation artifacts are**. Every chapter is stamped **"Schemas are [P]"** (Proposed), and that is not decoration — the *how* is genuinely unwritten. There is **no code**: the `support` repo currently holds only `FEATURES.md`, `LICENSE.md`, and the `specification/` set (per README §Status). Critically, the **three distinct authentication mechanisms** the plane depends on, the **`user_ref` derivation**, the **diagnostic-log scrubber** (the load-bearing content-leak guarantee), the **pinned wire/DB/API schemas**, and the **config defaults + email/blob backend choices** are all either undescribed or flagged `[P]` by the spec itself. A developer told to "build NyxiteSupport" would stall at the first endpoint — there is no pinned submission payload to deserialize, no credential scheme to authenticate the relay, and no rule set for the scrubber that makes the whole plane safe.

So: **6 hard blockers** (three of them the missing auth mechanisms; one security-critical; one the missing shared wire fixture), **7 moderate gaps** (standard patterns, but every threshold, token, and backend is unspecified), and **2 minor/soft items**. None of these is a research track — unlike the server's Phase-6 items, nothing here is "undesigned by admission as go/no-go." They are all **pin-the-design** work that must be done **before the first line of `NyxiteSupport` or its relay is written**.

Phase reference: this component is **Phase 4.6** (`P4.6-*`); its build-gate steps are `P4.6-SUP-1…5` (this service), `P4.6-SRV-1/2` (the server relay), `P4.6-DSK/WEB/AND-1` (client composers), and the two cross-cutting fixtures `P4.6-CORE-1` (submission + ticket DTOs) and `P4.6-CORE-2` (redaction-destructiveness vector).

---

## HARD BLOCKERS

*(Prioritized. HB-1…HB-3 are the three separate auth mechanisms the plane needs — none is specified, and they must not be conflated. HB-4 is the security-critical content-leak guarantee. HB-6 is the shared wire fixture every other component compiles against.)*

### HB-1 — Intake auth / instance recognition for `POST /reports` is unspecified (the single biggest gap)
**Blocks:** `P4.6-SUP-1` (intake + instance recognition), `P4.6-SRV-1` (the relay's forward path), and abuse control (`P4.6-TC-13`, which asserts junk from an *unrecognized* instance is rejected).

The whole anti-spam and attribution model rests on this service recognizing *which instance* relayed a report, but **how** is only gestured at:
- `05 §5.5` and `02 §2.5` say intake is *"analogous to the license `/register` signature verification"* and authenticated by *"instance fingerprint + license token"* — but the **actual credential, the signature scheme, and the verification procedure** are never pinned. `04 §4.2` restates this as a third auth mode ("Instance-relay auth … recognized instance credentials/fingerprint") without defining it.
- The **enrollment/recognition of a *free* third-party instance** is the unresolved core. SUP-9 (corrected 2026-07-12, `01 §1.5`) requires that **third-party self-hosted instances — which have no enterprise license — are nonetheless "recognized" and route to the central desk.** A "license token" cannot be the credential for an instance that has no license. How a free instance is enrolled, what anchor proves it is a genuine Nyxite instance (vs. a spammer forging a fingerprint), and how that anchor is minted/rotated are all undefined.
- No signature format, no challenge/nonce, no key-distribution story, no replay protection.

**To resolve:** pin the intake credential and verification for **both** an enterprise-licensed instance (reuse L-5/L-7 license token + fingerprint) **and** a free instance (a new recognition anchor — e.g. an instance-registration handshake analogous to but distinct from `NyxiteLicense /register`), the signature suite, and the exact server-side accept/reject procedure. This is `P4.6-SUP-1`'s "instance recognition" and gates every downstream intake test.

### HB-2 — `user_ref` derivation algorithm is undefined
**Blocks:** `P4.6-SRV-1` (the server mints `user_ref`), `P4.6-SRV-2` / `P4.6-SUP-2` (reporter API scoped by `user_ref`), and `P4.6-TC-7` (a reporter sees **only** their own tickets).

`02 §2.5` requires `user_ref` to be simultaneously **(a)** stable per account (so "My tickets" correlates a reporter's submissions across time), **(b)** non-reversible (this service learns only an opaque handle, never account identity — invariant 01 §1.2.4), and **(c)** scoped so no instance can enumerate or collide with another instance's refs. Nothing specifies the **derivation**:
- The construction (e.g. `HMAC(server_secret, account_id)` vs. a random per-account handle persisted server-side) is unstated.
- **Cross-instance uniqueness/namespacing** — whether `user_ref` is globally unique or only unique within an `instance_fingerprint` — is undefined, and it is load-bearing: if two instances can mint the same `user_ref`, the reporter-API scoping (HB-3) is broken.
- The **guest-session variant** — how `user_ref` (or its equivalent) is derived for a share/guest session with no account (`02 §2.6`) — is not specified.

**To resolve:** pin the derivation (function, secret/keying, and whether it is deterministic or stored), the `(instance_fingerprint, user_ref)` composite-key rule that guarantees cross-instance non-collision, and the guest variant.

### HB-3 — Reporter-API auth (server → support proxy) is unspecified
**Blocks:** `P4.6-SRV-2` (server proxies `GET/POST /support/tickets…`), `P4.6-SUP-2` (the `user_ref`-scoped reporter API), and `P4.6-TC-7`.

This is a **third, distinct** auth surface — not the intake credential (HB-1) and not operator/API-token auth (HB-5). When a client opens "My tickets," the self-hosted server proxies this service's reporter API keyed by `user_ref` (`02 §2.7`, `03 §3.4`). The spec never says **how the relaying server authenticates to that reporter API**, nor — critically — **what prevents instance A from requesting instance B's `user_ref` tickets.** `03 §3.4` asserts scoping "enforced by `user_ref` scoping on this service" but gives no mechanism binding a caller to the `(instance_fingerprint, user_ref)` pairs it is entitled to read.

**To resolve:** pin the reporter-API credential (likely the same recognized-instance anchor as HB-1, but stated), and the server-side authorization rule that a recognized instance may only read/write `user_ref`s it minted — i.e. the reporter API must scope by `(authenticated instance_fingerprint, user_ref)`, not `user_ref` alone.

### HB-4 — Client-side diagnostic-log / envelope scrubber rules are undefined (security-critical)
**Blocks:** `P4.6-DSK/WEB/AND-1` (each client's envelope), `P4.6-CORE-1` (the envelope schema), and `P4.6-TC-5` (the envelope is "content-free").

`02 §2.4` (SUP-1/SUP-2) requires the client to strip **"file/project names, paths, and any decrypted strings"** from error logs/stack traces before they can be attached, and to attach only a **valid screen/route identifier** (`editor`, `settings/devices`) — never a file or project name. This is a **load-bearing guarantee**: the scrubber is the only thing standing between a diagnostic bundle and a content leak on a plane that is *deliberately not encrypted*. Yet there is:
- **no scrubbing algorithm** — no allow/deny pattern set, no path/identifier redaction rules, no handling of user strings embedded in exception messages;
- **no canonical list of valid screen/route ids** against which the attached location is validated (the spec gives two examples, not the vocabulary);
- **no conformance vector.** (`P4.6-CORE-2` pins destructiveness for *screenshots*, but there is **no equivalent scrubber vector for the log/envelope path** — a gap in the fixture set, not just the prose.)

**To resolve:** specify the scrubber as an explicit rule set (deny-patterns for paths/filenames/known content markers, an allow-list validator for screen/route ids, and a decrypted-string guard), pin the canonical screen/route-id vocabulary, and add a failing-by-default conformance vector (companion to `P4.6-CORE-2`) asserting a planted content string never survives scrubbing on all three clients.

### HB-5 — Operator authentication, provisioning & RBAC are unspecified
**Blocks:** `P4.6-SUP-4` (the operator UI) and the whole management surface's access model (`P4.6-SUP-3`).

`04 §4.2` says only *"human operators authenticate to the operator UI (this service's own auth; admin-only)."* Nothing pins:
- the **mechanism** — password, SSO/OIDC, WebAuthn? (`05 §5.6` places the UI behind the WireGuard admin network, but network-tier isolation is not operator authentication);
- **operator onboarding/provisioning** — how the first operator and subsequent ones are created (there is an `operators` table in `05 §5.2`, but no enrollment flow);
- the **role/RBAC model** — the schema has only an `assignee_id`; whether operators have differentiated permissions (triage-only vs. admin vs. webhook/token management) is undefined. Contrast the server spec, which has a full RBAC table set.

**To resolve:** choose the operator auth mechanism, define the provisioning flow (bootstrap + invite), and decide whether operator RBAC exists in v1 or all operators are equal (and if the latter, say so explicitly).

### HB-6 — The concrete wire / API / DB contracts do not exist; the shared `P4.6-CORE-1` fixture is unbuilt
**Blocks:** `P4.6-CORE-1` (the cross-component submission + ticket DTO fixture) and, through it, **every** other 4.6 step — clients, the server relay, and this service all "compile against it" (per the phase-4.6 build gate).

This is the cross-cutting counterpart to the server's OpenAPI gap, but more severe because **nothing** downstream exists yet. Concretely missing:
- **No JSON schemas** for the report-submission payload, the ticket/comment read DTOs, or the webhook event bodies. Chapters describe fields in prose only.
- **No field-level contract** — types, required/optional, lengths, enum value sets (the lifecycle states and `priority` values are named in prose but not pinned as an enum contract).
- **No PostgreSQL DDL.** `05 §5.2` lists columns as prose (e.g. "`tickets` — `id (pk)`, `state`, …"); there are no types, constraints, indexes, FKs, or migrations.
- **No OpenAPI.** The endpoint paths in `04 §4.3` are labelled illustrative ("Schemas/paths are [P]"), and `P4.6-DOC-2` expects the API reference to be *generated from the service's OpenAPI* — which does not exist.
- **The `P4.6-CORE-1` fixture itself is unbuilt** — the shared, content-free wire object that asserts (by construction) no content-key / no content-plane ciphertext / no plaintext file-or-project-name field, and that clients + server + support all validate against.

**To resolve:** author `P4.6-CORE-1` (the frozen shared DTOs) first — it is the highest-leverage single deliverable — then derive the DDL migrations and the service OpenAPI from it. No new design is needed to *shape* these (the inputs exist in 02–05), but until they are pinned on disk nothing cross-component can compile.

---

## MODERATE GAPS (implementable with standard patterns, but unspecified — resolve before coding)

### MG-1 — Every "configurable" threshold lacks a default, a unit, and a config-key name
**Affects:** `P4.6-SUP-5` (retention + abuse controls), `P4.6-SUP-1` (intake caps).

Consistent with the project's "placeholders mean configurable" rule, `05 §5.4`/`§5.5` correctly make these operator config — but a build needs **default values, units, and config keys**, none of which exist for: the **retention window** (and any separate cap-total-age-regardless-of-state), the **max screenshot bytes per attachment**, **max attachments per report**, **max-open-tickets per `user_ref`**, and the **per-`instance_fingerprint` / per-`user_ref` / per-API-token rate limits**. `P4.6-TC-12`/`-13` assert these are "read from config, not a constant" — so the config surface (key names + shipped defaults) must be defined. (This mirrors the server spec's `NYXITE__…` config-key convention, which has no counterpart here yet.)

### MG-2 — Guest capability-token scheme is undefined
**Affects:** `P4.6-SUP-2` (guest token issuance + emailed link), `03 §3.5`, `P4.6-TC-6`.

`05 §5.2` stores `guest_ticket_tokens.token_hash`, and `03 §3.5` requires the token be *"unguessable, per-ticket, and revocable."* Unspecified: **token length/entropy**, the **hash function** producing `token_hash`, **expiry/TTL** (and whether the emailed status link expires separately), and the **revocation semantics** (the `revoked` column exists but its lifecycle is undefined).

### MG-3 — Email delivery for guests is unspecified
**Affects:** `P4.6-SUP-2`, `03 §3.5`, `02 §2.6`, `P4.6-TC-6`.

Guest reply notifications and the initial tokenized status link require outbound email, but there is no **provider/SMTP choice**, **message templates**, **localization**, **bounce/deliverability handling**, or **link TTL**. This is the only outbound-messaging dependency on the plane and it is entirely unpinned.

### MG-4 — Webhook mechanics are named but not defined
**Affects:** `P4.6-SUP-3`, `04 §4.4`, `P4.6-TC-10`.

`04 §4.4` names "HMAC over the body," "retried with backoff," and "idempotency-keyed," but pins none of: the **HMAC algorithm + signature header format**, the **retry/backoff schedule** (attempts, intervals, give-up), the **idempotency-key format**, or **per-subscription secret management** (generation, storage, rotation).

### MG-5 — Blob-store backend is not chosen
**Affects:** `P4.6-SUP-1` (the `IBlobStore`-style store), `05 §5.3`.

The spec pins the *pattern* ("`IBlobStore`-style, `sha256`-keyed, a **separate** store from the content plane") but not the **concrete backend** (filesystem vs. S3 vs. MinIO) or a **content-type policy** beyond "PNG or platform equivalent." A deploy manifest (`P4.6-DEP-1`) cannot be finalized without the backend choice.

### MG-6 — Self-hosted-desk (SUP-10…13) reply relay-down transport, handoff model & lease check are only at decision level
**Affects:** the opt-in self-hosted desk (`01 §1.6`); no dedicated step in 4.6 (the phase targets the central desk), so this is design owed **before** a self-hosted-desk build.

SUP-13 states that after a bug is confirmed and handed off upstream, "the operator's desk **relays replies back down** to the reporter." Described only at the decision level, undefined at build level:
- the **downward transport** — does the operator's desk **poll** the central desk, does central **webhook** the desk, and **how is that channel authenticated** (the SUP-7 relay is defined desk→central for the upstream payload, but the reply path back down is not);
- the **handoff / thread-ownership data model** — how a locally-owned ticket becomes maintainer-owned, and how replies are threaded across the two desks;
- the **enterprise-lease runtime check** (SUP-11, L-2/L-4) — the desk must "refuse to function without a valid lease," but the integration point with the entitlement lease is not specified.

### MG-7 — Ticket identity & merge semantics are undefined
**Affects:** `P4.6-SUP-3` (`POST /tickets/{id}/merge`), `04 §4.3`, `05 §5.2`.

The `tickets.id` **scheme** (UUID vs. a human-friendly ticket number for operator/reporter reference) is unstated. And `POST /tickets/{id}/merge` is listed with no semantics: what happens to the merged ticket's **comments, history, tags, and reporter/`user_ref` linkage** (does the reporter of the absorbed ticket keep visibility via "My tickets"?) is undefined — non-trivial because merge crosses reporter boundaries that HB-3's scoping otherwise isolates.

---

## MINOR / SOFT-BLOCKER ITEMS (a competent dev resolves these in-flight, but they should be named)

- **MA-1 — NyxiteDesign token pipeline for the operator SPA (soft blocker for the UI).** DS-3 (`05 §5.6`) says the operator SPA consumes **CSS custom properties + a Tailwind theme generated from `nyxite-tokens.json`**, but the **concrete build tool** (an in-repo generator vs. Style Dictionary) is still an open implementation choice in `NyxiteDesign`. `P4.6-SUP-4` can proceed against Layer-A tokens once the generator is chosen, so this soft-blocks the operator UI, not the service core. Brand fonts must be **self-hosted/bundled, never a Google Fonts CDN** (a fixed constraint, not a gap).
- **MA-2 — Client↔reporter-API notification/polling contract.** `03 §3.4` says a new public reply "raises an in-app notification on next sync/poll," reusing the client's existing notification surface. The **unread-tracking / read-cursor / ETag contract** (how the client detects new public replies without content push, consistent with the no-FCM stance) is not defined. Straightforward, but part of the reporter-API contract that `P4.6-SRV-2`/`P4.6-SUP-2` must agree on.

---

## What IS ratified and stable (for the record)

To keep the "NO" honest — the following are settled and need no further decision, they are simply not yet turned into artifacts:

- **The decision layer is complete.** SUP-1…SUP-13 and DS-1…DS-3 are all resolved in `docs/OPEN-DECISIONS.md`; there are no live-open support decisions. The 2026-07-12 corrections (SUP-9 routing; SUP-10…13 self-hosted desk) are reconciled across `01 §1.5`/`§1.6` and `05 §5.5`/`§5.7`.
- **The disjointness invariants** (`01 §1.2`) are precise and test-enforceable, and the phase encodes them as `P4.6-TC-11` (plaintext, isolated, no content-plane linkage) and `P4.6-CORE-1`'s by-construction schema assertion.
- **The lifecycle** (`03 §3.1`: `new → triaged → in-progress → waiting-on-reporter → resolved → closed`, + `reopen`) and the **internal-note vs public-reply** model (`03 §3.3`) are fully described in prose — they need an enum/DTO pinning (HB-6), not a design.
- **The relational shape** (`05 §5.2`) names every table and column and correctly forbids any content-plane join — it needs DDL (HB-6), not new modelling.
- **The API-first principle + endpoint list** (`04 §4.1`/`§4.3`) and the **redaction destructiveness rule** (`02 §2.3`, with the `P4.6-CORE-2` vector already scoped) are settled — the redaction vector is the *one* concrete conformance fixture already specified.
- **The deployment posture** (`05 §5.6`, `P4.6-DEP-1/2`): isolated vendor infra, no Compose entry in the customer stack, management/UI on the admin network, only intake + guest link public — this is clear and buildable once the service exists.

---

## Gaps grouped by area

| Area | Gap | Severity |
|------|-----|----------|
| Intake auth (`P4.6-SUP-1`, `SRV-1`) | Instance recognition / relay credential undefined, esp. for **free** third-party instances with no license (HB-1) | **Hard blocker** |
| Opaque identity (`SRV-1`, `SUP-2`) | `user_ref` derivation, cross-instance uniqueness, guest variant undefined (HB-2) | **Hard blocker** |
| Reporter-API auth (`SRV-2`, `SUP-2`) | Server→support proxy auth + cross-instance `user_ref` isolation unspecified (HB-3) | **Hard blocker** |
| Diagnostics (`DSK/WEB/AND-1`, `CORE-1`) | Log/envelope scrubber rules + screen-id vocabulary + conformance vector missing — content-leak guarantee (HB-4) | **Hard blocker (security-critical)** |
| Operator access (`SUP-4`, `SUP-3`) | Operator auth mechanism, provisioning, and RBAC undefined (HB-5) | **Hard blocker** |
| Wire/DB/API (`CORE-1`, all `SUP-*`) | No JSON schemas, no DDL, no OpenAPI; shared `P4.6-CORE-1` fixture unbuilt (HB-6) | **Hard blocker (highest-leverage)** |
| Config (`SUP-5`, `SUP-1`) | Retention/caps/rate-limit defaults, units, key names absent (MG-1) | Moderate |
| Guest tokens (`SUP-2`) | Length/entropy, hash fn, TTL, revocation undefined (MG-2) | Moderate |
| Email (`SUP-2`) | Provider, templates, localization, bounces, link TTL unspecified (MG-3) | Moderate |
| Webhooks (`SUP-3`) | HMAC alg/header, retry schedule, idempotency-key, secret mgmt undefined (MG-4) | Moderate |
| Blob store (`SUP-1`) | Backend (fs/S3/MinIO) + content-type policy not chosen (MG-5) | Moderate |
| Self-hosted desk (`01 §1.6`) | Reply relay-down transport, handoff model, lease runtime check at decision level only (MG-6) | Moderate |
| Ticket identity (`SUP-3`) | Id scheme + merge semantics (comments/history/tags/reporter linkage) undefined (MG-7) | Moderate |
| Operator SPA (`SUP-4`) | NyxiteDesign token build tool still an open choice (MA-1) | Minor (soft) |
| Reporter notifications (`SRV-2`, `SUP-2`) | Unread/read-cursor/ETag polling contract undefined (MA-2) | Minor |

---

## Top 6 most critical gaps

1. **Intake instance-recognition auth (HB-1)** — the biggest gap; no credential/signature scheme, and no enrollment story for **free** third-party instances that must still be recognized (SUP-9). Gates all intake.
2. **The three auth mechanisms are all missing and must not be conflated (HB-1/HB-2/HB-3)** — intake instance-recognition, `user_ref` derivation + cross-instance scoping, and the server→reporter-API proxy credential are three separate unspecified surfaces; the reporter-isolation guarantee (`P4.6-TC-7`) depends on all three.
3. **Pinned wire/DB/API contracts + the `P4.6-CORE-1` fixture (HB-6)** — no JSON schemas, no PostgreSQL DDL, no OpenAPI, and the shared content-free DTO fixture that clients + server + support all compile against does not exist; the single highest-leverage deliverable.
4. **Diagnostic scrubber rules (HB-4)** — the load-bearing content-leak guarantee on a deliberately-unencrypted plane has no algorithm, no screen-id vocabulary, and (unlike the screenshot path) no conformance vector.
5. **Operator auth, provisioning & RBAC (HB-5)** — the entire management surface has no login mechanism, onboarding flow, or permission model beyond a bare `assignee_id`.
6. **Config defaults, guest tokens, email, webhooks & blob backend (MG-1…MG-5)** — all standard patterns, but every threshold, token, provider, and backend is left as "[P] / operator config" with no default, unit, key name, or chosen implementation.

**Net:** NyxiteSupport cannot be started as a faithful build today. The decisions are ratified, but a developer would first need the **pinned wire/DB/API schemas** (HB-6), the **three auth mechanisms** — intake instance-recognition, `user_ref` derivation, and the server→reporter-API credential (HB-1/2/3), the **diagnostic scrubber rules** (HB-4), **operator login + RBAC** (HB-5), and the **config defaults + email + blob backend** choices (MG-1…MG-5) — most of which the spec itself flags as **[P] Proposed**. This is pin-the-design work, not research: none of it is undesigned-by-admission, and all of it is achievable from the existing chapters, but it must precede the first line of code.
