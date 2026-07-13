# 05 — Data model, storage, retention & operations

> Persistence (plaintext, isolated), retention + right-to-erasure, abuse controls, stack, packaging, and out-of-scope. Schemas are **[P]**.

## 5.1 Storage posture (SUP-8)

- Reports and tickets are stored **in plaintext** — free text, diagnostic envelope, redacted screenshots, comments, and (for guests) email. This service is where the maintainer *reads* reports, so there is no server-held key to hide them behind; the protections are consent + client-side redaction + retention (not encryption). The **disk-theft exposure is explicit and accepted**.
- **Isolation.** This service's PostgreSQL + blob storage are **separate infrastructure** with **no path to any content-plane database or blob store** (01 §1.2). A compromise here exposes only support tickets — never content, keys, names, or counts.

## 5.2 Relational model (PostgreSQL) [P]

- `tickets` — `id (pk)`, `state`, `priority`, `assignee_id`, `title`, `description`, `instance_fingerprint`, `user_ref` (opaque), `guest_email` (nullable), `guest_name` (nullable), `share_context` (nullable, opaque), `created_at`, `updated_at`.
- `ticket_tags` — `ticket_id (fk)`, `tag`.
- `ticket_comments` — `id (pk)`, `ticket_id (fk)`, `kind` (`internal | public`), `author_kind` (`operator | reporter | automation`), `author_id`, `body`, `created_at`.
- `ticket_diagnostics` — `ticket_id (fk)`, JSONB envelope (app version/build, platform, locale, screen id, scrubbed logs, connection state) as submitted (02 §2.4).
- `ticket_attachments` — `id (pk)`, `ticket_id (fk)`, `blob_ref`, `content_type`, `bytes`, `sha256` (integrity of the stored raster), `created_at`.
- `ticket_history` — `id (pk)`, `ticket_id (fk)`, `event`, `actor_id`, `actor_kind`, `data` (JSONB), `created_at`.
- `guest_ticket_tokens` — `token_hash (pk)`, `ticket_id (fk)`, `created_at`, `revoked` (03 §3.5).
- `operators`, `api_tokens`, `webhooks` — operator identities, scoped automation tokens, and webhook subscriptions (04).
- **No table** references or joins to any Nyxite user, project, file, or key. This database holds **only** support-plane data.

## 5.3 Blob storage [P]

- Redacted screenshots are stored as **plaintext** rasters in a content-addressed blob store behind an `IBlobStore`-style abstraction (same pattern as the server, but a **separate** store), keyed by `sha256`, with EXIF/metadata already stripped client-side (02 §2.3).
- Size-capped per attachment and per ticket (§5.4).

## 5.4 Retention & right-to-erasure (SUP-8) [P]

- **Configurable retention window** — tickets and their attachments are purged after a retention period; the period is an **operator config value**, not a hardcoded constant (e.g. purge closed tickets after a configured age; cap total age regardless of state). *(Placeholder-style thresholds are always configuration, never fixed numbers.)*
- **Right-to-erasure** — a reporter (via the client / guest link) or the maintainer can request deletion of a ticket and its attachments; deletion is honored on the support plane. Because there is no content-plane linkage, erasure here is self-contained.
- **Purge is destructive by design** on this plane (unlike the content plane's soft-delete durability rule) — support data is disposable and erasure must actually remove it. Purges are logged in `ticket_history` before the row/blob is removed.

## 5.5 Abuse controls [P]

- **Rate limits** — per `instance_fingerprint` and per `user_ref` at intake (the relaying server is the first gate, 02 §2.5), and per API token on the management API (04).
- **Caps** — max screenshot bytes per attachment, max attachments per report, and a **max open tickets** per `user_ref` to bound spam.
- **Instance recognition** — `POST /reports` accepts only from **recognized** instances (authenticated by instance fingerprint + license token, analogous to the license `/register` anti-spam check). Both the maintainer's official instance(s) and third-party self-hosted instances are recognized and route to the central desk (SUP-9, corrected 2026-07-12); an operator running their **own** opt-in desk (SUP-10–SUP-13) intakes locally and relays only confirmed bugs upstream via the same anti-spam anchor.

## 5.6 Stack & packaging [P]

- **ASP.NET Core (C#)** minimal API, matching the server stack and sharing C# domain conventions. PostgreSQL for §5.2; a separate blob store for §5.3.
- **Operator UI** — this service ships **its own** operator helpdesk UI (queue, ticket detail, triage, reply, tags/assignee, search). It is a **React + shadcn/ui** single-page app (DS-2), the same stack as `NyxiteWeb` / `NyxiteAdmin`, so it inherits the shared component library instead of re-implementing primitives by hand. It adopts the shared **`NyxiteDesign` tokens** (`nyxite-tokens.json`, the platform-agnostic source of truth) as **Layer A only** — tokens + standard components (tables, forms, badges, dialogs) — and consumes the generated CSS custom properties + Tailwind theme (DS-3); it does **not** take the editor app-shell (Layer B, rail/toolbar-densities/canvas), keeping its own helpdesk layout so it shares the Nyxite design *language* while staying contextually distinct (DS-1). Brand fonts (Manrope UI / Source Serif 4 content) are **self-hosted / bundled, never a Google Fonts CDN**. `NyxiteAdmin` only **links out** to it, and only on instances where reporting is enabled (SUP-9); the UI itself is admin-only, reachable over the maintainer's admin network (WireGuard), consistent with `NyxiteAdmin`'s exposure tier.
- **Deployed as isolated vendor infrastructure** on the maintainer's VPS — its own host/container, no network path to any customer content plane. Publicly reachable only for the **intake relay** (`POST /reports` from self-hosted servers) and the **guest status link** (03 §3.5); the management API + operator UI are admin-network-only.

## 5.7 Out of scope

- **Self-hostable / per-operator helpdesk** — **resolved (SUP-10–SUP-13, 2026-07-12)** and specified in [01 §1.6](01-support-plane.md): an enterprise operator opts in via a drop-in `docker-compose.override.yml`, tickets stay local, only confirmed bugs route upstream after a configurable age. The **service code is the same**; this is a **deployment/routing mode**, not a separate build.
- **Built-in AI categorizer** — categorization is the maintainer's automation over webhooks + API (04 §4.5); no model ships in this service.
- **Any content, key, or user-data access** — architecturally impossible here (01 §1.2).
- **Billing / payments / analytics** — not a helpdesk concern.
- **Content-plane linkage** — no join to any Nyxite user/project/file record; the submitter is only ever an opaque `user_ref`.
