# 04 — API surface, webhooks & automation

> This service is **API-first (SUP-4)**: every operator action is available through the REST API, not only the operator UI, and **webhooks** let the maintainer wire external automation (n8n) or a local-AI worker that categorizes tickets and writes back. The helpdesk and its automation are **free**, never enterprise-gated. Schemas/paths are **[P]**.

## 4.1 API-first principle (SUP-4)

- The operator UI is **one** client of the API — it holds no privileged capability the API lacks. Anything a human can do in the UI (triage, tag, prioritize, assign, comment, transition, merge, close, reopen, search) is doable via the API with an API token.
- This is what lets the maintainer run the whole desk (or parts of it) from n8n / scripts / a local AI, per the requirement that all actions be doable via the API and not only the UI.

## 4.2 Authentication [P]

- **Operator sessions** — human operators authenticate to the operator UI (this service's own auth; admin-only, reachable over the maintainer's admin network — see 05).
- **API tokens** — automation authenticates with **scoped API tokens** (e.g. `tickets:read`, `tickets:write`, `comments:write`, `webhooks:manage`). Tokens are revocable and rate-limited (05 §5.5).
- **Instance-relay auth** — the `POST /reports` intake path (02 §2.5) authenticates the **relaying self-hosted server** by recognized instance credentials/fingerprint, separate from operator/API-token auth.

## 4.3 REST endpoints (management) [P]

All under the support host, operator/automation-scoped (never exposed by a self-hosted instance):

| Method + path | Purpose |
|---|---|
| `POST /reports` | **Intake** from a relaying server (02 §2.5) — creates a ticket |
| `GET /tickets` | List/filter/search (state, tag, priority, assignee, instance, text, date) |
| `GET /tickets/{id}` | Full ticket incl. envelope, screenshots, comments, history |
| `PATCH /tickets/{id}` | Update state, tags, priority, assignee |
| `POST /tickets/{id}/comments` | Add a comment (`kind: internal | public`) |
| `POST /tickets/{id}/transition` | Explicit lifecycle transition (validated, 03 §3.1) |
| `POST /tickets/{id}/merge` | Merge into another ticket (dedup) |
| `POST /tickets/{id}/close` · `/reopen` | Convenience transitions |
| `GET /tickets/{id}/attachments/{aid}` | Fetch a redacted screenshot blob |
| `GET/POST/DELETE /webhooks` | Manage webhook subscriptions (§4.4) |
| `GET/POST/DELETE /api-tokens` | Manage scoped automation tokens |

- **Reporter API (separate scope)** — the endpoints the self-hosted server proxies for "My tickets" (02 §2.7 / 03 §3.4): list/read/reply, **scoped to a single `user_ref`**, never able to see another reporter's tickets.

## 4.4 Webhooks (SUP-4) [P]

- **Events:** `ticket.created`, `ticket.updated`, `ticket.commented`, `ticket.state_changed` (extensible). Each delivers a JSON payload with the ticket id + changed fields.
- **Delivery:** signed (HMAC over the body with a per-subscription secret), retried with backoff, idempotency-keyed so a consumer can dedupe.
- **Write-back loop:** a consumer (n8n flow or local-AI worker) receives `ticket.created`, reads the ticket (`GET /tickets/{id}`) including the free text + diagnostic envelope + (optionally) the screenshot, decides **category tags / priority / assignee**, and writes them via `PATCH /tickets/{id}` — closing the automation loop entirely over the public API.
- Because the plane is **not** E2EE, an automation worker can read full ticket content directly (no key handling); a **local AI** running on the maintainer's infra keeps that content on-premises.

## 4.5 Automation integration notes [P]

- **n8n** — subscribe a webhook node to `ticket.created`; branch on content; call back `PATCH`/`POST /comments`. No bespoke plugin required — plain HTTP + the signed webhook.
- **Local AI** — the same shape with a self-hosted model doing classification/summarization; recommended for privacy since ticket content stays on the maintainer's VPS.
- **No built-in categorizer.** This service ships **no** bundled model (SUP-4 rejected the batteries-included option); categorization is the maintainer's automation, wired via webhooks + API.
