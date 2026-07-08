# 03 — Ticketing: lifecycle, replies & the reporter view

> The two-way ticketing model: the state machine, categorization, the operator/reporter conversation, and how a reporter (account or guest) follows their ticket. Schemas are **[P]**.

## 3.1 Ticket lifecycle (SUP-3) [P]

A ticket is created on report intake (02) in state `new` and moves through:

```
new → triaged → in-progress → waiting-on-reporter → resolved → closed
                     ↑                                   |
                     └────────────── reopen ─────────────┘
```

- **`new`** — just received, untriaged.
- **`triaged`** — categorized (tags/priority set), acknowledged.
- **`in-progress`** — actively being worked.
- **`waiting-on-reporter`** — a public reply asked the reporter for more; the ball is in their court.
- **`resolved`** — a fix/answer is provided; reporter can confirm or reopen.
- **`closed`** — terminal; `reopen` returns it to `in-progress` (or `triaged`) and is always allowed within retention.

Transitions are validated server-side (this service); illegal transitions are rejected. Every transition is recorded with actor + timestamp for the ticket history.

## 3.2 Attributes (SUP-3) [P]

- **Category tags** — free or from a maintained vocabulary; **set manually or by automation** (04). A ticket may carry several.
- **Priority** — e.g. `low | normal | high | urgent`.
- **Assignee** — an operator/agent identity on this service (not a Nyxite content-plane user).
- **Submitter reference** — the opaque `user_ref` + `instance_fingerprint` from intake (02); for guests, the collected email + share context.
- **Diagnostic envelope + screenshots** — as submitted (02), immutable once received.

## 3.3 Comments: internal notes vs public replies (SUP-3) [P]

Two comment kinds on a ticket:

- **Internal note** — visible only to operators/automation; never shown to the reporter. Used for triage discussion, automation annotations, and cross-references.
- **Public reply** — visible to the reporter (via "My tickets" or the guest link). Posting a public reply optionally moves the ticket to `waiting-on-reporter`.

A **reporter reply** (from the client or guest link) always lands as a public comment authored by the reporter and can reopen a `resolved`/`closed` ticket (within retention).

## 3.4 Reporter "My tickets" view — account users (SUP-3, SUP-7) [P]

- Each client renders a **"My tickets"** view listing the account's own reports with current **state**, **public replies**, and unread indicators.
- Data is fetched through the **self-hosted server**, which proxies this service's **reporter API** keyed by the opaque `user_ref` (02 §2.7) — a client never talks to this service directly, and a reporter can only ever see **their own** tickets (enforced by `user_ref` scoping on this service).
- **Notifications** — a new public reply raises an in-app notification on next sync/poll (reuses the client's existing notification surface; no third-party push, consistent with the no-FCM stance).

## 3.5 Reporter view — guests (SUP-6) [P]

- A guest has no account, so on ticket creation this service issues a **capability token** bound to the ticket and emails the guest a **no-login status/thread link** (`https://<support-host>/t/<opaque-ticket-token>`).
- The link opens a minimal, read-and-reply status page: current state, public replies, and a reply box. The token is **unguessable, per-ticket, and revocable**, scoped to exactly one ticket; it carries no account or content access.
- Email delivery (reply notifications, the initial link) is a support-plane concern only; the guest email is stored plaintext under consent + retention (05).

## 3.6 History & audit [P]

- Every state transition, assignment, tag change, and comment is appended to an immutable per-ticket **history** with actor + timestamp, for operator accountability.
- This is **operator-side** history on the support plane; it is unrelated to (and never joined with) the content-plane audit log, which remains content-free.
