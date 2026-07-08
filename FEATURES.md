# Nyxite Support — Features

ASP.NET Core (C#) **vendor-side helpdesk server** — the ticketing service the project maintainer runs to receive in-app bug reports, triage them, and reply to reporters. A **separate component** (its own repo, `NyxiteSupport`): like [`NyxiteLicense`](license.md), it is **central vendor infrastructure run by the maintainer** and is **not** part of the self-hosted stack a customer deploys. There is exactly **one** helpdesk — the maintainer's; no operator ever runs their own (a future opt-in self-hostable mode is tracked as a backlog item — see [OPEN-DECISIONS.md](../docs/OPEN-DECISIONS.md)). The self-hosted `NyxiteServer` acts only as an **authenticating relay** for report submission, and each client (desktop/web/Android) hosts the report UI plus a "My tickets" view.

**Privacy first — but the support plane is a *consensual, non-E2EE* plane, deliberately disjoint from the content plane.** This is the **one** place in Nyxite where a user may voluntarily disclose readable information to the maintainer, and it exists only because the user **explicitly chooses** to. Three guarantees make that safe:

1. **Explicit consent + destination notice.** Before a report is sent, the user sees a clear **"unlike your files, this report is *not* end-to-end encrypted, and it goes to the Nyxite maintainer"** notice (relevant when the user is on a third-party instance) alongside a GDPR disclosure.
2. **User-driven redaction.** The user redacts the screenshot **on their own device** before it ever leaves — destructively — so disclosure is minimized to exactly what the user paints in.
3. **No content-plane leakage.** A report carries **no content keys and no content-plane ciphertext** — only the free text, the redacted screenshot, and a technical diagnostic envelope the user reviews. The content-plane zero-knowledge guarantee (the server never holds a content key) is **untouched**.

## Model — a consensual support plane (SUP-1)

- The support plane is architecturally separate from the content plane: it is entered only by a deliberate user action, never automatically, and it never touches a content key or any content-plane ciphertext.
- **Not E2EE, stored server-readable.** Reports are readable by the maintainer (that is the point of a helpdesk) and are **stored in plaintext** on the vendor-side helpdesk (SUP-8). The load-bearing protections are the **explicit non-E2EE warning** and the **user's own client-side redaction** — not encryption. The disk-theft exposure this implies is explicit and accepted; retention limits and right-to-erasure bound it.
- **Central, maintainer-run only (SUP-5).** `NyxiteSupport` is vendor infrastructure. It is never in the self-hosted stack, has no Compose entry, and no operator deploys it — mirroring the `NyxiteLicense` isolation posture.

## In-app bug reporting (clients: desktop / web / Android) (SUP-2)

- A **"Report a bug"** entry point inside each client.
- Free-text **title + description**.
- **Non-E2EE consent + destination notice** shown before send (SUP-1).
- **Diagnostic envelope** — a technical, non-content context bundle the user can **review and edit** before sending: app version, build, platform/OS, locale, the current screen/route id (**not** content), scrubbed client-side error logs / stack traces, and relay/connection state.

## Screenshot capture & redaction (SUP-2)

- Optional screenshot of the current app view, captured client-side (native window grab on desktop, canvas/display capture in the browser, `PixelCopy`/`MediaProjection` on Android).
- **Redaction editor** with **black-box** and **blur** tools.
- **Destructive, client-side flattening** — redacted regions are burned into the pixels **before upload**; the original image and any redaction mask are **never** sent. There is no "original + overlay" that could be peeled back.

## Ticketing — full two-way lifecycle (SUP-3)

- States: `new → triaged → in-progress → waiting-on-reporter → resolved → closed`, with reopen.
- **Category tags** (set manually or by automation), priority, and assignee.
- **Internal notes** (operator-only) vs **public replies** (visible to the reporter).
- **Reporter sees status + replies** — a genuine back-and-forth conversation, not fire-and-forget intake.

## Submitters — accounts and guests (SUP-6)

- **Authenticated account users** — the ticket is tied to an **opaque account reference** on the originating instance; replies surface in the client's "My tickets" view.
- **Guest-link users** (people on a share/guest session, no account) — may also file; the client **collects an email + optional name + the share context**. Because there is no account to notify, replies reach a guest via **email + a no-login tokenized status/thread link**.

## Reporter-facing "My tickets" view (SUP-3/SUP-7)

- Each client (desktop/web/Android) provides a **"My tickets"** view where an account user sees their open reports, current status, and support replies, with in-app notifications.
- Guests use the emailed tokenized link instead of an in-app view.

## API-first + automation (SUP-4)

- A **comprehensive REST API** is the primary surface: **every** ticket action is doable via the API, not only the operator UI — CRUD, state transitions, tag/priority/assignee changes, comments/replies, attachments, merge/close/reopen, and listing/filtering/search.
- **Webhook events** (`ticket.created`, `ticket.updated`, `ticket.commented`, `ticket.state_changed`) let the maintainer wire an external workflow engine (**n8n**) or a **local AI** worker that reads a ticket and **writes categories / priority / assignee back** through the API. Automation authenticates as a trusted **API-token** client.
- **All free (community).** The helpdesk and its API/webhooks are not an enterprise-gated feature.

## Submission path — server as authenticating relay (SUP-7)

- A client submits its report to its own `NyxiteServer`, which **authenticates** the submitter (account session or guest-share session), then **forwards** the report to the central `NyxiteSupport` service tagged with the **instance fingerprint** (anti-spam trust anchor, as with `NyxiteLicense /register`), an **opaque user reference**, and the user-supplied contact. The server is a relay only — it does not store the ticket.

## Routing & availability — central-only in v1 (SUP-9)

- There is **one** helpdesk (the maintainer's), and **all reports route to the maintainer**.
- In v1 the in-app reporting surface is enabled on the **maintainer's official Nyxite instance(s)**; a third-party self-hosted instance has **no** reporting surface (its users' reports are **not** silently routed upstream). Per-operator, self-hosted helpdesks are a **deferred backlog item** (below).
- Every participating client shows the report UI + "My tickets" view only where reporting is enabled; where it is not, the surface is simply absent (no disabled hint).

## Data, storage & operations (SUP-8)

- **PostgreSQL** for tickets, comments, tags, assignees, submitter references, and the diagnostic envelope; **blob storage** for redacted screenshots — all **plaintext**, isolated from any content-plane database.
- **Retention & right-to-erasure** — a **configurable retention window** for tickets and their screenshots, plus reporter/operator delete, since the plane holds user-disclosed PII (email, redacted images, free text).
- **Abuse controls** — per-account / per-instance rate limits, screenshot size caps, and a max-open-tickets bound; the server-side authenticating relay is the first anti-spam gate.
- **Stack** — ASP.NET Core (C#) matching the server stack; its own operator helpdesk UI; deployed as **isolated vendor infrastructure** with no network path to any content plane. `NyxiteAdmin` shows an **outbound link** to this UI only on instances where reporting is enabled.

## What this component is (and is not)

- **Is:** ASP.NET Core (C#) central helpdesk — ticket store, comprehensive REST API + webhooks, its own operator UI, PostgreSQL + blob storage for plaintext reports. Vendor-run, isolated.
- **Is not:** part of the self-hosted stack; a content or key service; something an operator deploys. It never touches a content key or content-plane ciphertext.
- **Boundary with the server:** `NyxiteServer` owns the **client-side report intake surface** (the "Report a bug" flow, redaction, the "My tickets" view) and the **authenticating relay** for submission; this component owns the **helpdesk plane** — ticket storage, lifecycle, API, webhooks, and the operator UI.

## Resolved decisions

All support/helpdesk sub-decisions are ratified — canonical in [../docs/OPEN-DECISIONS.md](../docs/OPEN-DECISIONS.md) (Resolved, SUP-1–SUP-9):

- **SUP-1** — consensual, **non-E2EE support plane**, disjoint from content; explicit non-E2EE + destination notice; no content keys / no content-plane ciphertext
- **SUP-2** — **manual, destructive, client-side** screenshot redaction (black-box + blur), flattened before upload
- **SUP-3** — **full two-way ticketing** (states, tags, assignee, internal notes, public replies, reporter "My tickets" view)
- **SUP-4** — **API-first**: every action via a comprehensive REST API; webhook events + write-back for n8n / local-AI categorization; all free (never gated)
- **SUP-5** — new dedicated **vendor-side component `NyxiteSupport`** (ASP.NET Core + PostgreSQL), central on the maintainer's VPS, own operator UI; **not** part of the self-hosted stack, never instance-deployed
- **SUP-6** — submitters: authenticated account users (opaque ref) + guest-link users (collect email + share context); guests replied to via email + tokenized no-login status link
- **SUP-7** — `NyxiteServer` acts as an **authenticating relay** (anti-spam) tagging submissions with instance fingerprint + opaque user ref; the server never stores the ticket
- **SUP-8** — reports **stored in plaintext** vendor-side (accepted disk-theft tradeoff; redaction + configurable retention + right-to-erasure are the mitigations)
- **SUP-9** — **central-only in v1**: one desk, all reports route to the maintainer; reporting surface enabled on the maintainer's official instance(s) only; per-operator self-hosted desks deferred (backlog)

**Backlog (deferred):** a **self-hostable helpdesk (opt-in)** mode where a self-hoster runs their own desk, tickets stay local to them, and only confirmed **bugs** route upstream to the maintainer (auto-routing after a configurable age) — see [OPEN-DECISIONS.md](../docs/OPEN-DECISIONS.md).
