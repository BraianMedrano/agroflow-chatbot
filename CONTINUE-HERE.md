# Continue here

Last updated: 2026-09-14. Demo date is not fixed; tentatively a couple of weeks out.

> **Update 2026-09-28.** The demo now runs fully local and the chatbot is wired to the real
> AgroFlow API: `flujos-n8n/Chatbot.json` uses an AI Agent with three API tools instead of the
> stub, and the setup lives in [docs/DEMO_RUNBOOK.md](docs/DEMO_RUNBOOK.md). Sections below that
> describe the stub, the VPS, or "next steps" are historical context.

## What changed

This file previously described an urgent, single-session push to get a chatbot
working before running out of budget. That framing is gone.

The demo has no fixed date (tentatively a couple of weeks out), and one new requirement settles the
architecture: **the ingenio dashboard consumes the same calculations the
chatbot produces.** Two consumers, one truth. The truth cannot live inside n8n.

**Revised again 2026-09-05.** The conversation lives in n8n, driven by an AI
Agent node; the API owns only the record.

The word "conversation" bundles three things, and they split cleanly:
understanding free text and conducting the dialogue belong in n8n, where a
language model handles "che la patente es AE123BC y salgo de San José" as one
message. Appointments and calculations belong in the API, because the ingenio
dashboard reads them.

The rule that survives every revision: **anything the dashboard must see is
written to the API at the moment it happens.** Losing the thread of a chat to an
n8n restart is acceptable; losing an appointment already on the ingenio's screen
is not.

The agent never invents a window, priority or departure time. It calls the API,
receives the numbers, and only puts them into words.

This is a **demo** — a simulation for a presentation, not functional software.
The API is scoped accordingly: three endpoints, a static finca table, no queue
theory. OpenAI key already provisioned; cost is not a constraint.

Contract: `docs/API_CONTRACT.md`. Read it before building either side.

Demo plan, approved scope, proposed contract v2, simulation spec and screenshots:
`docs/demo-plan/DEMO_PLAN.md`. It supersedes the scope notes in this file where
they differ (live map, incidents, hybrid dashboard).

## Verified state

All of this was confirmed live on `vps-lab`, read-only, on 2026-09-04.

- Four containers healthy, `restarts=0`: `agroflow-n8n`,
  `agroflow-evolution`, `agroflow-evolution-postgres`,
  `agroflow-evolution-redis`. The earlier n8n restart loop is fixed — the
  896 MiB cgroup plus `NODE_OPTIONS=--max-old-space-size=512` was transferred
  and applied.
- n8n owner account created. Reached at `http://localhost:5678` over the tunnel.
- Evolution API v2.3.7 reachable at `http://localhost:8080`, manager at
  `/manager/` (the trailing slash matters; `/manager` 301s to it).
- WhatsApp instance `agroflow-demo` exists and is connected:
  `connectionStatus = open`, `integration = WHATSAPP-BAILEYS`, on a disposable
  number.
- Evolution reaches n8n over the Docker bridge:
  `http://agroflow-n8n:5678/healthz` returns `{"status":"ok"}` from inside the
  Evolution container.
- Webhook configured on the instance for `MESSAGES_UPSERT` only, with
  "Webhook by Events" **off**.
- One real `MESSAGES_UPSERT` payload captured and mapped. Redacted sample:
  `docs/samples/messages-upsert.conversation.json`.

### n8n workflow as it stands

Round trip closed and verified with a real WhatsApp message on 2026-09-05: a
message in produces a reply back on the phone.

```
Webhook (POST)
  -> Respond to Webhook (200)
  -> Filter
  -> Edit Fields (Normalize)
  -> HTTP Request  -> POST http://localhost:5678/webhook/stub-chat-inbound
  -> HTTP Request  -> POST http://agroflow-evolution:8080/message/sendText/agroflow-demo
```

`Respond to Webhook` sits **second, not last**, on purpose. Evolution must be
acknowledged before the workflow does its slow work, otherwise its HTTP call
waits through the API round trip and the send, and a timeout makes it retry —
delivering the same message twice. Acknowledge first, work after.

The two HTTP Request nodes read from different places:

```
$json                  -> previous node's output      -> reply.text (from the stub)
$('Edit Fields').item  -> the normalized message      -> chatId
```

The Evolution credential is an n8n **Header Auth** credential named `apikey`.
It is never inline in a node, so the workflow can be exported safely.

Filter conditions, all AND:

```
{{ $json.body.data.key.fromMe }}        is false
{{ $json.body.data.messageType }}       is equal to    conversation
{{ $json.body.data.key.remoteJid }}     does not contain    @g.us
```

Normalize output — eight flat fields, "Include Other Input Fields" off:

```
chatId          {{ $json.body.data.key.remoteJid }}
phone           {{ $json.body.data.key.remoteJid.split('@')[0] }}
name            {{ $json.body.data.pushName || '' }}
messageId       {{ $json.body.data.key.id }}
instance        {{ $json.body.instance }}
text            {{ $json.body.data.message.conversation.trim() }}
textNormalized  {{ $json.body.data.message.conversation.trim().toLowerCase().normalize('NFD').replace(/[\u0300-\u036f]/g, '') }}
receivedAt      {{ new Date($json.body.data.messageTimestamp * 1000).toISOString() }}
```

This normalized shape is deliberately the request body of
`POST /api/v1/chat/inbound`, minus `channel` and `ingenioId`, which are
constants added at the HTTP Request node.

## Payload facts that must not be re-derived

- The message text is at `body.data.message.conversation`. There is no
  `message.text`. The key changes with `messageType`
  (`extendedTextMessage.text`, `imageMessage.caption`, and so on), which is why
  the filter pins `messageType` before anything reads the text.
- `body.sender` is **not** the sender. It is the instance's own JID. With
  `fromMe: false`, the counterparty is `body.data.key.remoteJid`.
- `body.date_time` is wrong: it emits local time with a `Z` suffix, three hours
  off for Argentina. Always derive time from `body.data.messageTimestamp`
  (Unix **seconds**, multiply by 1000).
- `body.apikey` arrives `null`. The inbound webhook has no authentication; the
  unguessable path UUID is its only protection. Add header auth before the
  endpoint is ever reachable from the public internet.
- `data.message.messageContextInfo` is end-to-end encryption metadata. It is
  roughly half the payload and is never used.

## Next steps, in order

- [x] **Stub the contract.** Second n8n workflow `stub-agroflow-api`, published,
      answering `POST /webhook/stub-chat-inbound` with the documented response
      shape. It needed no new service, dependency or stack decision.
- [x] **Close the WhatsApp round trip.** Verified with a real message on
      2026-09-05. The transport path — receive, filter, normalize, call out,
      reply — is proven and does not need revisiting.

Remaining, in order:

1. **Export the workflows** and commit both JSON files to this repository. They
   currently exist only inside the n8n instance, which has no backup. Do this
   before touching anything else.
2. **Replace the single HTTP Request with an AI Agent node.** OpenAI credential,
   a system prompt carrying the conversational script from section 6 of the
   context document, and three HTTP tools pointing at stubbed endpoints. The
   surrounding nodes do not change.
3. **Decide the API stack and repository.** Team decision — see below.
4. **Build the API.** Three endpoints, static finca table, simulation behind one
   interface, `simulated: true` on everything it returns. Deliberately small.
5. **Point the agent's tools at the real API.** URLs only.
6. **Dashboard** over `GET /appointments` and `GET /status` — the same rows the
   agent writes.
7. **Public routing**, only once there is something worth showing. See below.
8. **Rehearse** against the script in section 7 of the context document,
   including the fallback video and screenshots. A live model in front of an
   audience needs a rehearsed path and a recorded backup.

## API stack — provisional

**Current direction: .NET.** Not final. The team will decide together later, so
nothing here should assume it.

This is deliberately a low-stakes open question. `docs/API_CONTRACT.md`
specifies HTTP, JSON and semantics only — no language, framework, ORM or
runtime appears in it. Whatever the group picks has to satisfy the same
contract, and the n8n side cannot tell the difference.

Layout expectation, in .NET terms, mapping to the seams the contract assumes:

```
AgroFlow.Domain           entities, state machine, priority rules
AgroFlow.Application      use cases; IAppointmentScheduler lives here
AgroFlow.Infrastructure   EF Core + Npgsql, repositories
AgroFlow.Api              endpoints, DTOs, auth
```

The fictitious calculation is an `IAppointmentScheduler` implementation
(`SimulatedAppointmentScheduler`). The real queue-theory algorithm is a second
implementation of the same interface, swapped by registration. Nothing outside
`Application` learns which one is running — that substitution is the entire
reason the calculation is behind an interface rather than inline in an endpoint.

No .NET SDK is installed on the development Mac as of 2026-09-04. Whoever builds
the API needs one, or builds it elsewhere. This does not block the n8n side.

## Public routing — unblocked, deferred

Previously recorded as blocked on certificate coverage. That blocker is
resolved by evidence: the VPS origin, queried on `127.0.0.1:443` with SNI
`n8n.casteltech.ar`, serves a Cloudflare Origin CA certificate whose SAN is
`*.casteltech.ar, casteltech.ar`, valid to 2041. The existing `castel_proxy`
vhost mounts it at:

```
/etc/nginx/certs/casteltech-origin.pem
/etc/nginx/certs/casteltech-origin.key
```

What actually remains:

1. `dig +short n8n.casteltech.ar` returns nothing. The proxied Cloudflare DNS
   record does not exist yet.
2. `infra/nginx/n8n.casteltech.ar.conf` still carries its BLOCKED header and no
   certificate directives. Fill in the two paths above and remove the header.
3. Apply `infra/compose.vps.yaml` so n8n joins `castel_network` and the proxy
   can resolve `agroflow-n8n`.

Steps 2 and 3 need operator sudo. Deferred until the chatbot works end to end —
publishing an empty demo buys nothing.

## Safety constraints

- Never read, print, or transfer the remote `infra/.env`. Variable names may be
  listed with `cut -d= -f1`; values may not.
- The `.env` key is `EVOLUTION_API_KEY`. Inside the container Compose maps it to
  `AUTHENTICATION_API_KEY` — a different name for the same secret.
- Do not restart or reconfigure Evolution, PostgreSQL, Redis, `castel_proxy`, or
  any unrelated container or stack.
- Disposable test data only. No real transportistas, plates, or ingenios.
- Baileys is unofficial. Never use a personal or commercial number, and never
  send bulk messages.

## Rollback, preserving volumes

```bash
ssh vps-lab 'cd /opt/agroflow-demo && docker compose --env-file infra/.env -f infra/compose.yaml down'
```

`--volumes` is intentionally absent, so named volumes survive.

## Deferred

- **Align the API shape with the dashboard that already exists.** The three
  endpoints in `docs/API_CONTRACT.md` were designed from the conversational side
  alone. The existing dashboard design will likely require changes — different
  fields, extra filters, possibly another read endpoint. Review it against the
  real screens before the API is built, not after.
- **Confirm where travel distances come from.** The static finca table in
  `docs/API_CONTRACT.md` §4 is an assumption, not a decision.

- Restore `/opt/agroflow-demo` to mode `750`; it currently reads `755`.
- Header authentication on the n8n webhook before any public exposure.
- Message types beyond plain text — audio, image, and replies each use a
  different key under `data.message`.
- Migration to WhatsApp Cloud API for anything past the demo.
