# AgroFlow API contract

The boundary between n8n and the AgroFlow API.

**This describes a demo.** The API simulates; it does not compute anything real.
The contract is designed so the simulation can later be replaced by real logic
without any consumer noticing, but nothing here should be built as if the real
system were being written today.

Revised 2026-09-05: the conversational agent now lives in n8n. See §1.

## 1. Who owns what

The word "conversation" bundles three separate things. They do not live in the
same place.

| | Lives in | Why |
|---|---|---|
| **Understanding** — turning free text into fields | **n8n** (AI Agent) | This is what a language model is good at. A trucker writing "che la patente es AE123BC y salgo de San José" is one message, not three prompts. |
| **Conducting** — what to ask next, wording, tone | **n8n** (AI Agent) | Demo copy changes constantly. Iterating on a prompt beats redeploying a service. |
| **Truth** — appointments, calculations, records | **API** | The ingenio dashboard reads it. Two consumers cannot share state that lives inside the conversational engine. |

n8n owns the conversation. The API owns the record.

### The one hard rule

**Anything the dashboard must see is written to the API at the moment it
happens** — not at the end of the conversation, not held in agent memory.

Losing the thread of a chat because n8n restarted is acceptable. Losing an
appointment the ingenio already had on screen is not.

### The agent must not invent the appointment

The agent never composes a window, a priority or a departure time. It calls
`POST /appointments`, receives the numbers, and only puts them into words. Every
figure a trucker sees came out of an endpoint.

This is a demo constraint, not an architectural preference: section 7 of the
context document requires that the assigned slot is never presented as the
product of a real algorithm.

## 2. Conventions

- Base path `/api/v1`. JSON, UTF-8.
- Timestamps are RFC 3339 in UTC. Never local time with a `Z` suffix — the
  Evolution webhook does exactly that, which is why n8n derives time from
  `messageTimestamp`.
- Auth: `X-Api-Key` header. n8n stores it as a credential.
- `ingenioId` is present from the start. One tenant exists; the field costs
  nothing now and avoids a migration later.

## 3. Endpoints

Three. That is the whole API.

### `POST /api/v1/appointments`

The agent's main tool. Creates the request and returns the simulated
assignment in one call — there is no separate "calculate" step to orchestrate.

```json
{
  "ingenioId": "demo",
  "phone": "5493812502185",
  "name": "Braian",
  "plate": "AE 123 BC",
  "origin": "Finca San José",
  "cutTime": "08:30"
}
```

Response `201`:

```json
{
  "id": "apt_01J8XYZ",
  "ingenioId": "demo",
  "phone": "5493812502185",
  "plate": "AE 123 BC",
  "origin": "Finca San José",
  "cutTime": "08:30",
  "windowStart": "2026-09-05T17:00:00Z",
  "windowEnd": "2026-09-05T17:30:00Z",
  "travelMinutes": 50,
  "recommendedDeparture": "2026-09-05T16:10:00Z",
  "priority": "high",
  "priorityReason": "antigüedad de la carga",
  "status": "assigned",
  "simulated": true,
  "createdAt": "2026-09-05T14:49:27Z"
}
```

`simulated: true` is not decoration. The dashboard renders simulated rows
differently, and the flag is what stops a demo figure from ever being read as a
real assignment. It flips to `false` the day a real algorithm produces the slot.

`422` when a field is missing or malformed, with a message the agent can turn
into a question. A `422` is normal conversational flow, not an error.

### `GET /api/v1/appointments`

One endpoint, two consumers.

```
GET /api/v1/appointments?phone=5493812502185      -> the agent, "¿cuál era mi turno?"
GET /api/v1/appointments?ingenioId=demo&date=...  -> the dashboard queue
```

Returns an array of the object above. Same shape both ways — the dashboard and
the agent are looking at the same rows, which is the entire point.

### `GET /api/v1/status?ingenioId=demo`

Operational snapshot. Feeds the dashboard, and lets the agent answer "¿hay
demora?" without guessing.

```json
{
  "ingenioId": "demo",
  "ingenioName": "Ingenio AgroFlow Demo",
  "operational": true,
  "currentDelayMinutes": 25,
  "trucksWaiting": 12,
  "windowCapacity": 4,
  "nextFreeWindow": "2026-09-05T18:00:00Z",
  "simulated": true
}
```

## 4. The simulation

Deliberately simple, deliberately deterministic. It has to be explainable in one
sentence on stage and produce the same answer twice in a rehearsal.

- Windows are fixed 30-minute slots.
- Each window holds up to `windowCapacity` trucks. Full window, take the next.
- Travel time comes from a **static table of fincas**, hardcoded. No maps API,
  no coordinates, no geocoding. For a demo, a lookup table is not a shortcut —
  it is the correct amount of engineering.
- Priority is by cut age: earlier `cutTime` means older cane means higher
  priority. `priorityReason` is a plain-Spanish string the agent reads aloud.
- `recommendedDeparture = windowStart - travelMinutes`.

> **Assumption, not yet confirmed.** The static finca table is my proposal for
> where distances come from. If there is a real source — coordinates, a maps
> API, a spreadsheet — this section changes. Nothing else does.

The whole thing goes behind one interface (`IAppointmentScheduler` in .NET
terms). The fake and the real are two implementations of it. That substitution
is the only reason the calculation is not written inline in the endpoint.

## 5. What the API deliberately does not do

- **No conversation state.** No steps, no "awaiting plate". That lives in the
  agent now.
- **No reply text.** The API returns data; the agent writes the words.
- **No queue theory.** Not for the demo. The interface is there for it later.
- **No auth beyond one API key.** It is a simulation behind a private URL.

## 6. The n8n side

```
Webhook -> Respond 200 -> Filter -> Normalize -> AI Agent -> Send via Evolution
                                                    |
                                                    | tools
                                                    v
                                          POST /appointments
                                          GET  /appointments?phone=
                                          GET  /status
```

The agent holds conversational memory. That memory is allowed to be ephemeral —
a restart costs a trucker their thread, not the ingenio its data, because every
committed fact already went to the API.

Model: OpenAI, key already provisioned.

## 7. Building against it

The published n8n stub workflow (`stub-agroflow-api`) already answers a canned
response. Point the agent's tools at stubs of the three endpoints above, build
and rehearse the full WhatsApp round trip, then swap in the real service.

If swapping requires editing anything in n8n other than URLs, the boundary
leaked.
