# Nyxite Support — Specification (v1.0.0)

This folder is the detailed, implementation-level specification for the **Nyxite Support** server — the **ASP.NET Core (C#)** vendor-side helpdesk that receives in-app bug reports, stores tickets, runs the two-way ticketing lifecycle, exposes an API-first management surface with webhooks, and hosts the operator UI. It is a **separate component**, run **centrally by the project maintainer**, and is **not** part of the self-hosted stack.

It expands the architectural planning documents in the central [`Nyxite`](https://github.com/Nyxite/Nyxite) repository (`features/support.md`) into a concrete build specification.

## Guiding principle: a consensual support plane, disjoint from the content plane

Nyxite is zero-knowledge everywhere on the **content plane** — the server stores only ciphertext and never holds a content key. The **support plane is the one deliberate, consensual exception**: a place a user may *voluntarily* disclose readable information to the maintainer, entered only by an explicit user action, and **architecturally separate** from content. A report carries **no content key and no content-plane ciphertext**; the content-plane guarantee is **untouched**. Reports are **not** end-to-end encrypted — the maintainer reads them (that is the point of a helpdesk) — and the user is told so, plainly, before sending. The load-bearing protections are the **explicit non-E2EE + destination notice** and the **user's own client-side, destructive redaction**, not encryption.

Like [`NyxiteLicense`](https://github.com/Nyxite/NyxiteLicense), this service is **isolated vendor infrastructure**: it has no network path to any customer content plane, is **central by default** (one helpdesk — the maintainer's; an enterprise operator may opt into their own desk per SUP-10–SUP-13, [01 §1.6](01-support-plane.md)), and plays no part in the E2EE guarantee between a user and their self-hosted server.

## Boundary with the server and clients

- **Clients** (`NyxiteDesktop` / `NyxiteWeb` / `NyxiteAndroid`) own the **report intake surface**: the "Report a bug" flow, the screenshot capture + **destructive redaction editor**, the diagnostic envelope, the consent notice, and the reporter-facing **"My tickets"** view.
- The self-hosted **`NyxiteServer`** owns the **authenticating relay**: it authenticates the submitter (account or guest-share session), then forwards the report to this service tagged with the instance fingerprint + an opaque user reference. It stores no ticket.
- **This service (`NyxiteSupport`)** owns the **helpdesk plane**: ticket storage, the lifecycle state machine, the comprehensive REST API, webhook dispatch, the operator UI, and data retention. Where the two disagree on the wire contract (submission payload, ticket API shapes), **this spec wins** for the vendor side and the server/client specs reference it.

## Source of truth

The central `Nyxite` repo is authoritative. This spec links to `docs/OPEN-DECISIONS.md` rather than restating decisions.

## Proposal convention

- **[P]** — *Proposed.* A concrete decision filled in by this spec, subject to confirmation; not yet ratified in the master docs.
- **[SUP-n]** — references a support decision (SUP-1…SUP-13) in `docs/OPEN-DECISIONS.md`.
- **[DS-n]** — references the design-system decision **DS** (and sub-decisions DS-1…DS-3) in `docs/OPEN-DECISIONS.md`, which pins the shared [`NyxiteDesign`](https://github.com/Nyxite/NyxiteDesign) visual language. Per DS-2 the operator UI is a **React + shadcn/ui** SPA that adopts the `NyxiteDesign` tokens (Layer A only — no editor app-shell; see 05 §5.6).

## Documents

| # | Document | Covers |
|---|----------|--------|
| 01 | [support-plane.md](01-support-plane.md) | The consensual non-E2EE plane, boundaries, central-only model, disjointness proof obligations, GDPR/consent posture |
| 02 | [report-intake-and-redaction.md](02-report-intake-and-redaction.md) | Client report UI, consent notice, diagnostic envelope, screenshot capture + destructive redaction, server authenticating relay, submission wire format, guest flow |
| 03 | [ticketing-and-lifecycle.md](03-ticketing-and-lifecycle.md) | Ticket state machine, tags/priority/assignee, internal notes vs public replies, reporter "My tickets" + notifications, guest tokenized status link |
| 04 | [api-and-automation.md](04-api-and-automation.md) | REST API surface, auth/API tokens, webhook events + write-back, n8n / local-AI integration, rate limiting |
| 05 | [data-model-and-operations.md](05-data-model-and-operations.md) | PostgreSQL schema + blob storage (plaintext), retention + right-to-erasure, abuse controls, stack, packaging, out of scope |

## Status

Specification for a greenfield build. No helpdesk code exists yet; the `support` repo currently holds `FEATURES.md`, `LICENSE.md`, and this `specification/` set. This document set defines what to build.

## License

PolyForm Noncommercial License 1.0.0 — see the repo `LICENSE.md`.
