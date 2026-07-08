# 02 — Report intake, redaction & the authenticating relay

> How a bug report is composed on the client, redacted destructively before it leaves the device, authenticated + relayed by the self-hosted server, and accepted by this service. Wire schemas are **[P]**.

## 2.1 Reporting-enabled signal (SUP-9) [P]

- A client shows the **"Report a bug"** entry point and the **"My tickets"** view **only** when its server advertises reporting as enabled. The server exposes a capability flag (e.g. `support.enabled` in its client-config / capabilities response) that is true on the maintainer's official instance(s) and false elsewhere in v1.
- Where the flag is false the surfaces are **absent** — not disabled with a hint.

## 2.2 Report composition (client) (SUP-1, SUP-2)

A report is composed entirely on the client and consists of:

- **Title + description** — free text.
- **Screenshot(s)** — optional, captured and redacted client-side (§2.3).
- **Diagnostic envelope** — a technical, non-content context bundle (§2.4), shown to the user for **review + edit** before send.
- **Contact** — for account users, none is entered (the opaque account ref carries reply routing); for **guests**, an **email + optional name** (§2.6).

The **consent + destination notice** (01 §1.3) is shown and must be confirmed before the report can be transmitted.

## 2.3 Screenshot capture & destructive redaction (SUP-2) [P]

- **Capture** is per-platform: desktop grabs the app window natively; web renders the current view to a canvas (or uses a display-capture the user grants); Android uses `PixelCopy` / `MediaProjection`.
- **Redaction editor** offers two tools: **black-box** (opaque rectangle) and **blur** (irreversible pixel blur, not a reversible filter).
- **Destructive flatten — the load-bearing rule.** Redaction is applied by **compositing the black boxes / blur into the raster and re-encoding a new image**. The **original image and any redaction geometry/mask are discarded on the device and never uploaded.** There is no layered format, no separate mask, and no metadata from which redacted pixels could be recovered.
  - The upload is a plain flattened **PNG** (or platform equivalent), **EXIF/metadata stripped**, size-capped (05 §5.4).
  - **Conformance obligation [P]:** a cross-client test proves that for a redacted region, the uploaded bytes contain no trace of the original pixels (e.g. redact a known pattern, assert it is absent), on all three clients.
- Redaction is **manual** — the user is responsible for what they hide. (Auto-masking of content regions was considered and rejected in favor of user control; see SUP-2.)

## 2.4 Diagnostic envelope (SUP-1) [P]

A non-content technical bundle, **user-reviewable and editable** before send:

- App version + build, platform / OS + version, locale.
- The current **screen / route identifier** (a UI location like `editor` or `settings/devices`) — **never** a file name, project name, or any content.
- **Scrubbed** client-side error logs / recent stack traces — passed through a client scrubber that strips file/project names, paths, and any decrypted strings before they can be attached.
- Relay / connection state (online/offline, last-sync age bucket) — coarse, non-identifying.

Nothing is attached that is not represented in this reviewable envelope (01 §1.3).

## 2.5 Submission — server as authenticating relay (SUP-7) [P]

The client does **not** call this service directly. It submits to its **own `NyxiteServer`**, which relays:

1. **Client → server.** `POST /support/reports` on the self-hosted server with the composed report (multipart: JSON body + flattened screenshot blobs). The server **authenticates** the submitter — an account session **or** a valid guest-share session (§2.6).
2. **Server-side tagging.** The server attaches, and the client cannot forge:
   - an **`instance_fingerprint`** — the same opaque per-instance anchor used for `NyxiteLicense /register`, so this service can attribute and rate-limit by instance and reject junk;
   - an **opaque `user_ref`** — a stable, non-reversible reference to the account (or guest session), used only for reply routing (03); it does **not** reveal account identity to this service beyond an opaque handle;
   - the server's software version.
3. **Server → support.** The server forwards to `POST /reports` on this service over TLS. This service **verifies the instance is a recognized/enabled instance** (anti-spam, analogous to license `/register` signature verification) before accepting.
4. **The server stores no ticket** — it is a stateless relay for the report. The reporter's ticket state is read back on demand for "My tickets" (03) via the server proxying this service's reporter API, keyed by `user_ref`.

- **Failure handling.** Submission is best-effort and off the critical path; a relay/transport failure surfaces a retryable error to the user, never blocks the app, and never persists a partial report on the server.

## 2.6 Guest submitters (SUP-6) [P]

- A **guest-link user** (on a share/guest session, no account) may file a report. The server authenticates them by their **guest-share session** and includes the **share context** (which share/relay session — an opaque id, never content) so a maintainer can reproduce.
- Because there is no account to notify, the client **collects an email + optional name**. Replies reach the guest via **email + a no-login tokenized status/thread link** (03 §3.5).
- The guest email lives only on the support plane (plaintext, 05) under the same consent + retention terms; it is never joined to any content-plane user record.

## 2.7 What the server exposes for intake [P]

New self-hosted-server endpoints (server `specification/04` REST API; enabled only when `support.enabled`):

- `POST /support/reports` — authenticated report intake + relay (§2.5).
- `GET /support/tickets` / `GET /support/tickets/{id}` — reporter view of *their own* tickets, proxied to this service by `user_ref` (03).
- `POST /support/tickets/{id}/replies` — reporter reply to their own ticket (03).

These are the **only** support endpoints on the self-hosted server; all operator/management endpoints live on this service (04) and are never exposed by a self-hosted instance.
