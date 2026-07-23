# Suelta API reference (flow lifecycle surface)

Base URL: `$SUELTA_API_URL` (production: `https://api.suelta.co`). All routes
below are under `/api/me`. Auth: `Authorization: Bearer $SUELTA_API_KEY`.
Content type: `application/json` both ways.

An API key is valid over `/api/me/*` only. `full_access` is a wildcard over
the scope catalog; it never grants key management, WhatsApp channel
connect/disconnect, `tools/http-test`, or any `/api/dev|admin` route.

## Bootstrap

### GET /profile — scope: none

Who am I. Returns the tenant profile (plan, status). Use as a smoke test.

### POST /activation-intent — scope: none

Fires a "this tenant wants to activate" notification to the Suelta team. Use
when the user hits `plan_required` on go-live routes and wants to proceed.
No body. Fire at most once per session.

## Preflight

### GET /whatsapp/status — scope: settings:read

```json
{"connected": true, "phone_number": "+573001234567"}
```

`connected:false` → the line must be connected by the user in the web app
Dashboard (`/app`, button **Conectar WhatsApp**). `POST /whatsapp/connect` and
`DELETE /whatsapp/disconnect` reject API keys (403) by design.

### GET /integrations — scope: settings:read

```json
[{"id":"gcal","name":"Google Calendar","description":"...","connected":true,"email":"owner@gmail.com"}]
```

### GET /integrations/gcal/verify — scope: settings:read

Live check against Google (refreshes the token):
`{"connected":true,"email":"...","events_this_week":[...]}`.

### POST /integrations/gcal/connect — scope: settings:write

No body. Returns `{"auth_url":"https://accounts.google.com/o/oauth2/..."}`.
Hand the URL to the user; the OAuth callback completes server-side with no
session. Poll `/integrations/gcal/verify` afterwards.

### GET /integrations/gcal/calendars — scope: settings:read

`{"calendars":[{"id":"c_abc@group.calendar.google.com","name":"Citas","primary":false}]}`

### DELETE /integrations/gcal — scope: settings:write

Revokes and deletes the stored token. Destructive; confirm with the user.

## Flows

The flow JSON object (fields you send on create; server assigns `id`,
`user_id`, timestamps; `enabled` is forced to `false` on create):

```jsonc
{
  "name": "Citas Barbería",             // required
  "description": "Agenda citas",
  "instructions": "system prompt...",    // required
  "model": "gemini-3.6-flash",           // required; prefix gpt-|o1-|o3-|o4-|gemini-
  "min_confidence": 85,                  // 0-100; defaults to 85 if omitted
  "operating_hours": {                   // optional; omit = always active
    "days": ["mon","tue","wed","thu","fri"],
    "from": "08:00", "to": "18:00"
  },
  "tools": [ /* see references/tools.md */ ]
}
```

Read-only response fields: `enabled`, `is_default`, `audience`,
`canary_config`, `published_at` (absent/zero = never published),
`publish_note`, `created_at`, `updated_at`.

| Method | Path | Scope | Notes |
|---|---|---|---|
| GET | `/flows` | flows:read | List all flows for the tenant |
| POST | `/flows` | flows:write | Create (born disabled) |
| GET | `/flows/{id}` | flows:read | Active version |
| DELETE | `/flows/{id}` | flows:publish | Cascade: flow + draft + all versions. Works on live flows. Confirm first. |

Plan gate: create/read-one/draft/tools/test-chat need an active plan OR
(`onboarding` + WhatsApp connected). List is always allowed.

## Draft lifecycle

Draft body = the versioned fields ONLY (whole-object upsert):

```json
{
  "name": "...", "description": "...", "instructions": "...",
  "model": "...", "min_confidence": 85,
  "operating_hours": {"days":["mon"],"from":"08:00","to":"18:00"},
  "tools": []
}
```

| Method | Path | Scope | Notes |
|---|---|---|---|
| GET | `/flows/{id}/draft` | flows:read | 404 `{"error":"no hay borrador"}`-style when none exists — that's normal |
| PUT | `/flows/{id}/draft` | flows:write | Validates like a full flow save |
| DELETE | `/flows/{id}/draft` | flows:write | Discard pending edits; idempotent |
| POST | `/flows/{id}/draft/tools` | flows:write | Body = one tool object; auto-seeds draft from active if absent |
| PUT | `/flows/{id}/draft/tools/{toolID}` | flows:write | Whole-tool replace |
| DELETE | `/flows/{id}/draft/tools/{toolID}` | flows:write | |
| GET | `/flows/{id}/versions` | flows:read | `{"current":{...},"history":[...]}`; history newest-first, max 5 |
| POST | `/flows/{id}/publish` | flows:publish + active plan | Body `{"note":"..."}` optional. Promotes draft; or self-publishes a never-published flow (sets `enabled:true`) |
| POST | `/flows/{id}/versions/{publishedAt}/revert` | flows:publish + active plan | `publishedAt` = RFC3339 from `history[].published_at`, URL-encoded |

## Test-chat (sandbox)

### POST /flows/{id}/test-chat — scope: flows:write

```json
{
  "messages": [{"role":"contact","content":"hola"},{"role":"agent","content":"¡Hola!"}],
  "user_message": "quiero una cita",
  "contact_phone": "+573001112233"
}
```

`role` is `"contact"` (the customer) or `"agent"` (the assistant). `messages`
may be `[]` for the first turn. `contact_phone` required (400 without it).

```json
{"response":"¡Claro! ¿Para qué día...","return_direct":false,
 "media":[{"type":"image","url":"https://...presigned...","caption":"..."}]}
```

Uses the draft if one exists, else the active config. Ephemeral: nothing
persisted, no WhatsApp side-effects (image sends are previewed via 5-minute
presigned URLs). CAUTION: `http_request` tools fire real HTTP calls and gcal
tools touch the real calendar.

## Going live

| Method | Path | Scope | Body |
|---|---|---|---|
| PATCH | `/flows/{id}/toggle` | flows:publish + active plan | `{"enabled":true\|false}`. Enabling a never-published flow → 400; use publish. |
| PATCH | `/flows/{id}/canary` | flows:publish + active plan | `{"mode":"percent","percent_quota":1-100}` or `{"mode":"allowlist","allowlist_phones":["+57..."]}`. Empty body clears (full traffic). Percent assignment is sticky per contact per day. |
| PATCH | `/flows/{id}/audience` | flows:publish + active plan | `{"audience":"all"\|"saved"\|"not_saved"\|"enrolled"}`. `saved`/`not_saved` = contact is/isn't in the tenant's contact list. `enrolled` = served ONLY via outbound enrollment (see references/outbound-enrollment.md). |

## Tool utilities

| Method | Path | Scope | Notes |
|---|---|---|---|
| GET | `/tool-types` | flows:read | Catalog of tool types with per-field config metadata. gcal types appear only when gcal is connected. |
| POST | `/tool-transforms/evaluate` | flows:write | `{"expression":"...","response":{...},"args":{...}}` — dry-run an expr-lang transform |
| POST | `/tools/image-upload` | flows:write | Upload a `send_image` asset. Consumes billable Meta media quota — confirm with the user. |
| POST | `/tools/http-test` | — | NOT available to keys (SSRF boundary). Test HTTP tools via test-chat instead. |

## Flow templates

| Method | Path | Scope |
|---|---|---|
| GET | `/flow-templates` | none |
| POST | `/flow-templates/{templateID}/instantiate` | flows:write |

Instantiate persists a new (disabled) flow from a catalog template — same
effect as `POST /flows`.

## LLM keys (BYOK tenants)

Self-service tenants bring their own LLM key; flow create/publish returns 422
if the model's provider has no key stored.

| Method | Path | Scope | Body |
|---|---|---|---|
| GET | `/llm-keys` | settings:read | → `{"openai_api_key":"sk-...masked"\|null,"gemini_api_key":...}` |
| PUT | `/llm-keys` | settings:write | `{"openai_api_key":"..."}` and/or `{"gemini_api_key":"..."}`; verified against the provider (422 if invalid); 204 on success |

## Outbound messages

`POST /messages/template` — scope `messages:send` + active plan. Spends real
money. Full contract in
[outbound-enrollment.md](outbound-enrollment.md).

## Observability (optional, for diagnosing a live flow)

| Method | Path | Scope |
|---|---|---|
| GET | `/agent-events?from=YYYY-MM-DD&to=YYYY-MM-DD[&flow_id=][&event_type=]` | events:read (`contact_phone` redacted without contacts:read) |
| GET | `/conversations/{id}` | events:read |
| GET | `/metrics/summary`, `/metrics/{name}` | metrics:read |

Max 20-day range on agent-events. Event types: `tool_success`, `tool_error`,
`llm_error`, `loop_max_iterations`, `tool_not_found`,
`repeated_response_suppressed`.
