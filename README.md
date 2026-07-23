# Suelta Agent Skills

[Agent Skills](https://agentskills.io) that let AI coding agents — Claude
Code, Cursor, Codex, and any other skills-compatible agent — operate
[Suelta](https://getsuelta.com) in natural language: build, test, and publish
WhatsApp AI assistants on a business's real WhatsApp line.

## Install

```bash
npx skills add suelta/agent-skills
```

## Requirements

- A Suelta account with its WhatsApp line connected (or the skill will walk
  the user through connecting it).
- A Suelta API key, created by the account owner in the web app under
  **Settings → Llaves de API**. Recommended scopes: `flows:read`,
  `flows:write`, `flows:publish`, `settings:read`, `settings:write`
  (add `messages:send` only if the agent should send outbound messages —
  sends spend real Meta quota).

```bash
export SUELTA_API_URL=https://api.getsuelta.com
export SUELTA_API_KEY=suelta_sk_...
```

## Skills

| Skill | What it does |
|---|---|
| [`build-suelta-flow`](skills/build-suelta-flow/SKILL.md) | Full assistant lifecycle: preflight checks (WhatsApp line, Google Calendar), flow creation and configuration, tools, sandbox test conversations, publish, version revert, gradual rollout (canary), audience targeting, and outbound sends with flow enrollment. |

## Try it

After installing, ask your agent things like:

> "Créame un asistente de citas para mi barbería en Suelta: atiende de martes
> a sábado, cita de 30 minutos, y pruébalo antes de publicar."

> "Cambia la despedida del flow 'Atención Patitas', pruébalo en sandbox y
> publícalo al 10% de los contactos."

The skill makes the agent verify prerequisites first (WhatsApp connected,
calendar linked), test every change in Suelta's sandbox before going live,
and ask for your confirmation before anything that touches real traffic or
spends money.

## Safety model

The API key can never: create or revoke API keys, connect or disconnect the
WhatsApp line, or reach admin surfaces — those stay in the web app, with the
account owner. Revoking a key takes effect on the very next request.

## License

MIT
