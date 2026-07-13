# Nyxite Support — Features

ASP.NET Core (C#) **vendor-side helpdesk server** — the ticketing service the project maintainer runs to receive in-app bug reports, triage them, and reply to reporters. A **separate component** (its own repo, `NyxiteSupport`): like [`NyxiteLicense`](https://github.com/Nyxite/NyxiteLicense), it is **central vendor infrastructure run by the maintainer** and is **not** part of the **default** self-hosted stack a customer deploys. By default there is exactly **one** helpdesk — the maintainer's — that **every** instance routes to; an **enterprise operator may opt into running their own desk** (cluster **SUP-10–SUP-13**, see [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) and the [section below](#self-hostable-helpdesk--opt-in-per-operator-desk-sup-10sup-13)), in which case their tickets stay local and only confirmed bugs route upstream. The self-hosted `NyxiteServer` acts as an **authenticating relay** for report submission, and each client (desktop/web/Android) hosts the report UI plus a "My tickets" view.

**Privacy first — but the support plane is a *consensual, non-E2EE* plane, deliberately disjoint from the content plane.** This is the **one** place in Nyxite where a user may voluntarily disclose readable information to the maintainer, and it exists only because the user **explicitly chooses** to. Three guarantees make that safe:

1. **Explicit consent + destination notice.** Before a report is sent, the user sees a clear **"unlike your files, this report is *not* end-to-end encrypted, and it goes to the Nyxite maintainer"** notice (relevant when the user is on a third-party instance) alongside a GDPR disclosure.
2. **User-driven redaction.** The user redacts the screenshot **on their own device** before it ever leaves — destructively — so disclosure is minimized to exactly what the user paints in.
3. **No content-plane leakage.** A report carries **no content keys and no content-plane ciphertext** — only the free text, the redacted screenshot, and a technical diagnostic envelope the user reviews. The content-plane zero-knowledge guarantee (the server never holds a content key) is **untouched**.

## Model — a consensual support plane (SUP-1)

- The support plane is architecturally separate from the content plane: it is entered only by a deliberate user action, never automatically, and it never touches a content key or any content-plane ciphertext.
- **Not E2EE, stored server-readable.** Reports are readable by the maintainer (that is the point of a helpdesk) and are **stored in plaintext** on the vendor-side helpdesk (SUP-8). The load-bearing protections are the **explicit non-E2EE warning** and the **user's own client-side redaction** — not encryption. The disk-theft exposure this implies is explicit and accepted; retention limits and right-to-erasure bound it.
- **Central, maintainer-run by default (SUP-5).** `NyxiteSupport` is vendor infrastructure. By default it is not in the self-hosted stack, has no base-Compose entry, and no operator deploys it — mirroring the `NyxiteLicense` isolation posture. The **one exception** is an **enterprise operator who opts into their own desk** (SUP-10–SUP-13, [section below](#self-hostable-helpdesk--opt-in-per-operator-desk-sup-10sup-13)) via a drop-in override — the base stack still never names it.

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

## Routing & availability — one central desk in v1 (SUP-9)

- There is **one** default helpdesk (the maintainer's), and **all reports route to the maintainer** unless the operator has opted into their own desk (SUP-10–SUP-13).
- **Every** instance — the maintainer's official Nyxite instance(s) **and** third-party self-hosted instances — shows the in-app report UI and routes tickets to that single central desk. Routing is **not silent**: the SUP-1 non-E2EE + "goes to the Nyxite maintainer" consent notice is shown before every send (its third-party-instance wording exists precisely because third-party users file to the maintainer).
- Per-operator, self-hosted helpdesks — tickets local, only confirmed bugs routed upstream — are the resolved **SUP-10–SUP-13** mode ([section below](#self-hostable-helpdesk--opt-in-per-operator-desk-sup-10sup-13)).

## Data, storage & operations (SUP-8)

- **PostgreSQL** for tickets, comments, tags, assignees, submitter references, and the diagnostic envelope; **blob storage** for redacted screenshots — all **plaintext**, isolated from any content-plane database.
- **Retention & right-to-erasure** — a **configurable retention window** for tickets and their screenshots, plus reporter/operator delete, since the plane holds user-disclosed PII (email, redacted images, free text).
- **Abuse controls** — per-account / per-instance rate limits, screenshot size caps, and a max-open-tickets bound; the server-side authenticating relay is the first anti-spam gate.
- **Stack** — ASP.NET Core (C#) matching the server stack; its own operator helpdesk UI; deployed as **isolated vendor infrastructure** with no network path to any content plane. `NyxiteAdmin` shows an **outbound link** to this UI only on instances where reporting is enabled.

## What this component is (and is not)

- **Is:** ASP.NET Core (C#) central helpdesk — ticket store, comprehensive REST API + webhooks, its own operator UI, PostgreSQL + blob storage for plaintext reports. Vendor-run, isolated.
- **Is not:** part of the self-hosted stack; a content or key service; something an operator deploys. It never touches a content key or content-plane ciphertext.
- **Boundary with the server:** `NyxiteServer` owns the **client-side report intake surface** (the "Report a bug" flow, redaction, the "My tickets" view) and the **authenticating relay** for submission; this component owns the **helpdesk plane** — ticket storage, lifecycle, API, webhooks, and the operator UI.

## Self-hostable helpdesk — opt-in per-operator desk (SUP-10–SUP-13)

An **enterprise** operator may opt into running their **own `NyxiteSupport` desk**, so their instance's tickets land **locally** instead of at the maintainer's central desk; only tickets **confirmed as bugs** route upstream. This reverses the SUP-5/SUP-9 central-only posture **for opted-in operators only** — the default self-hosted stack is untouched. Resolved 2026-07-12; canonical in [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md).

- **Deployment (SUP-10)** — the desk ships as a drop-in **`docker-compose.override.yml`** the operator adds to their stack (Compose auto-merges it). The base stack stays clean and never names the helpdesk — the opt-in *is* the presence of the file — and the override also patches the `server` service to relay to the **local** desk instead of the central one. Not a base-stack Compose profile (which would list a dormant desk service in every operator's file).
- **Enterprise-gated (SUP-11)** — running your own desk is an **enterprise feature** (L-3 catalog); the **central** helpdesk stays free to use. The gate is enforced at **runtime by the entitlement lease** (L-2/L-4), not by the file's presence, so the desk **refuses to function without a valid lease** and degrades/locks with the other enterprise features on a licence lapse.
- **Bug routing (SUP-12)** — a ticket is armed for upstream routing by an explicit operator **"confirm as bug"** action **or** the `bug` category tag (manual, or via a SUP-4 webhook / local-AI worker), then **auto-routes after the operator-configured age window**. Routing is **soft-mandatory** — built in and non-disable-able, but the operator controls *what* is confirmed (confirm nothing → route nothing); there is no hard "disable upstream" switch.
- **Upstream payload, auth & ownership (SUP-13)** — a routed bug sends the **full bundle** (free text + the client-redacted screenshot + the diagnostic envelope), tagged with instance fingerprint + opaque user ref via the SUP-7 relay run **desk→central**, authenticated by the operator's existing **license token + instance fingerprint** (L-5/L-7 — no new credential). The reporter is told **at submission** that confirmed bugs may be shared upstream. On routing the ticket is **handed off to the maintainer**, who owns the thread and may reply; the operator's desk **relays those replies back down** to the reporter's "My tickets" / guest link. Consent copy on any desk reads **"goes to your instance operator"** + the upstream-sharing line; guest reply links point at the operator's desk. The operator runs its own SUP-8 **plaintext** plane and inherits its retention + right-to-erasure duties (GDPR controller).

## Resolved decisions

All support/helpdesk sub-decisions are ratified — canonical in [../docs/OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (Resolved, SUP-1–SUP-13):

- **SUP-1** — consensual, **non-E2EE support plane**, disjoint from content; explicit non-E2EE + destination notice; no content keys / no content-plane ciphertext
- **SUP-2** — **manual, destructive, client-side** screenshot redaction (black-box + blur), flattened before upload
- **SUP-3** — **full two-way ticketing** (states, tags, assignee, internal notes, public replies, reporter "My tickets" view)
- **SUP-4** — **API-first**: every action via a comprehensive REST API; webhook events + write-back for n8n / local-AI categorization; all free (never gated)
- **SUP-5** — new dedicated **vendor-side component `NyxiteSupport`** (ASP.NET Core + PostgreSQL), central on the maintainer's VPS, own operator UI; **not** part of the default self-hosted stack *(amended by SUP-10–SUP-13 — an opted-in enterprise operator may instance-deploy their own desk)*
- **SUP-6** — submitters: authenticated account users (opaque ref) + guest-link users (collect email + share context); guests replied to via email + tokenized no-login status link
- **SUP-7** — `NyxiteServer` acts as an **authenticating relay** (anti-spam) tagging submissions with instance fingerprint + opaque user ref; the server never stores the ticket
- **SUP-8** — reports **stored in plaintext** vendor-side (accepted disk-theft tradeoff; redaction + configurable retention + right-to-erasure are the mitigations)
- **SUP-9** — **one central desk in v1**: one desk, and **all** instances (maintainer's + third-party) show the report UI and route to it, non-silently via the SUP-1 consent notice *(corrected 2026-07-12 — the earlier "third-party instances have no reporting surface" wording is withdrawn)*
- **SUP-10** — self-hosted desk **deployment** = drop-in `docker-compose.override.yml` (base stack untouched, not a Compose profile); the override repoints the `server` relay to the local desk
- **SUP-11** — self-hosted desk is **enterprise-gated**, enforced at runtime by the entitlement lease (not the file); the central helpdesk stays free
- **SUP-12** — upstream routing **armed** by operator "confirm as bug" **or** the `bug` tag, then **auto-routes after a configurable age**; **soft-mandatory** (no disable switch — confirm nothing, route nothing)
- **SUP-13** — routed bug sends the **full bundle** (text + redacted screenshot + envelope) via the SUP-7 relay run **desk→central**, authenticated by the operator's **license token + fingerprint**; **handed off to the maintainer** (replies relayed back down); reporter told at submission

**Resolved 2026-07-12:** the self-hostable helpdesk (opt-in) mode is **no longer a backlog item** — see cluster SUP-10–SUP-13 above and in [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md).
