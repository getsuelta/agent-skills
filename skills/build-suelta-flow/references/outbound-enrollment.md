# Outbound sends + enrolling a contact onto a flow

Two things in one endpoint: send an approved WhatsApp template to a customer,
and (optionally) leave a flow "listening" for that customer's reply.

**Every successful send spends the tenant's billable Meta quota. Never call
this without explicit user confirmation of recipient + template.**

## POST /api/me/messages/template

Scope: `messages:send`. Requires an active plan. Synchronous.

```json
{
  "to": "+573001234567",
  "template_name": "recordatorio_cita",
  "language": "es",
  "parameters": ["Ana", "martes 3pm"]
}
```

| Field | Required | Notes |
|---|---|---|
| `to` | yes | E.164 with `+`. Spaces/dashes/parens tolerated. |
| `template_name` | yes | `^[a-z0-9_]{1,512}$`. Must be an APPROVED template on the tenant's WABA. Template CRUD is not exposed to keys — ask the user which templates exist. |
| `language` | no | Meta code (e.g. `es`). Falls back to the stored template's language. |
| `parameters` | no | Positional: index 0 fills `{{1}}`. Max 30; none may be empty, contain newlines/tabs, 5+ consecutive spaces, or exceed 1024 chars. Count must match the template. |
| `flow_id` | no | Enroll the recipient onto this flow after the send. |
| `listen_hours` | no | Enrollment window. Default 24, range 6–72. Only with `flow_id`. |

Response 200:

```json
{"message_id":"wamid.HBgM...","to":"+573001234567","enrolled":true}
```

`enrolled` is `true` only when `flow_id` was given AND enrollment succeeded.
If the send worked but enrollment failed, you get `"enrolled":false` plus an
`enroll_error` string — the message was already sent and billed; fix the flow
instead of resending. A 200 means Meta ACCEPTED the message, not that it was
delivered.

## How enrollment works

- The target flow must: exist, belong to the tenant, have
  `audience:"enrolled"`, be `enabled:true`, and be published. This is
  validated BEFORE the (billed) send, so a bad flow costs nothing.
- One enrollment per (tenant, phone) — enrolling again replaces the previous
  one, including onto a different flow.
- While enrolled, the contact's replies are PINNED to that flow: it bypasses
  the router, operating hours, and canary. The window closes hard at
  `listen_hours` (mid-conversation too).
- There is no list/cancel-enrollment endpoint. An enrollment ends by expiring,
  by being replaced, or by the flow itself calling its `end_enrollment` tool
  (give outbound flows one, so the model can close the loop gracefully —
  "listo, ¡nos vemos el martes!").

## Recipe: outbound campaign flow

1. Build the flow with golden path A, but set
   `PATCH /flows/{id}/audience` `{"audience":"enrolled"}` after publishing —
   it will never answer organic inbound traffic.
2. Give it an `end_enrollment` tool and instructions for the conversation goal
   (confirm, reschedule, upsell...).
3. For each recipient the user approves:
   `POST /messages/template` with `flow_id` + `listen_hours`.
4. Watch replies via `GET /agent-events?flow_id=...`.
