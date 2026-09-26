# AgroFlow API consumer guide for the local chatbot demo

## Authority

The canonical HTTP contract is `docs/contracts/openapi.yaml` in the
[`agustinvallante/AgroFlow`](https://github.com/agustinvallante/AgroFlow)
repository. This document explains how the n8n workflow consumes that contract;
it does not define business rules, routes, request fields, statuses, or errors.

The Monday profile is a local demo with fictitious seeded data. It does not include
user login, registration, production authentication, deployment, incidents, maps,
or production WhatsApp guarantees.

## Ownership boundary

| Concern | Owner |
|---|---|
| Understand free text and ask for missing fields | n8n agent |
| Normalize the incoming phone and extracted text | n8n workflow |
| Validate seeded carrier, truck, farm, capacity, and lifecycle | AgroFlow API |
| Assign a window and persist the appointment | AgroFlow API |
| Produce conversational wording from confirmed API data | n8n agent |
| Display the same persisted appointment | AgroFlow Dashboard |

**n8n owns the conversation. The API owns the record.**

The agent must never invent an appointment window, priority, state transition, or
successful cancellation. It communicates a result only after receiving a successful
API response.

## Local configuration

n8n runs inside Docker, so `localhost` points to the n8n container rather than the
host machine. Configure:

```env
AGROFLOW_API_URL=http://host.docker.internal:5000
```

The canonical routes already include `/api/v1`; workflow expressions append them to
this base URL.

Docker Desktop resolves `host.docker.internal`. On Linux, add the equivalent
`host.docker.internal:host-gateway` mapping or use the host gateway address. If API
and n8n later share one Compose network, use the API service name instead.

The Monday profile has no user authentication or API key. Do not infer a production
security decision from this local-only exception.

## Workflow inputs

The existing normalization step provides:

```text
chatId
phone
name
messageId
instance
text
textNormalized
receivedAt
```

The workflow uses the normalized incoming `phone` as the carrier identifier. It
must not hardcode internal UUIDs in prompts, tools, or mapping nodes.

## Supported intents

The minimum workflow supports exactly three business intents:

1. request an appointment;
2. look up the caller's appointment;
3. report that the truck is `EN_CAMINO`.

Other intents should receive a clear demo-scope response. They must not invoke
unrelated or speculative API operations.

## 1. Request an appointment

The agent gathers and confirms:

- caller phone, taken from the normalized webhook;
- truck plate;
- seeded farm code;
- cane cut timestamp with explicit offset;
- estimated load in tons.

Request:

```http
POST ${AGROFLOW_API_URL}/api/v1/appointments
Content-Type: application/json
```

```json
{
  "carrierPhone": "5493815550123",
  "truckPlate": "AF123BC",
  "farmCode": "F-01",
  "cutAt": "2026-09-28T05:30:00-03:00",
  "estimatedLoadTons": 28.5
}
```

A successful `201` response contains the persisted appointment, assigned window,
and state `ASIGNADO`. Only then may the agent confirm the appointment.

The chatbot does not ask for a preferred window. Window selection belongs to the
backend.

## 2. Look up the caller's appointment

Request:

```http
GET ${AGROFLOW_API_URL}/api/v1/appointments?phone=5493815550123
```

The API returns appointments associated with the normalized phone. The workflow
must use the current persisted state and window; it must not reconstruct either
from conversational memory.

If more than one result is returned, the agent identifies the relevant appointment
using the confirmed plate or asks the caller to clarify.

## 3. Report `EN_CAMINO`

The workflow first identifies the appointment through the phone lookup and confirmed
plate. It then requests the transition:

```http
POST ${AGROFLOW_API_URL}/api/v1/appointments/{id}/transitions
Content-Type: application/json
```

```json
{
  "newStatus": "EN_CAMINO"
}
```

The agent confirms departure only after a successful `200` response reports
`EN_CAMINO`.

The chatbot does not perform later operator transitions and does not cancel
appointments in the Monday profile.

## Error handling

The API returns `application/problem+json` with stable `code` and `traceId` fields.
The workflow handles them as follows:

| HTTP | Meaning for the conversation |
|---|---|
| `400` | Ask again for the invalid or missing field. |
| `404` | Explain that a seeded carrier, truck, farm, or appointment was not found; request confirmation. |
| `409` | Explain the current business conflict without claiming success. |
| `500` | Apologize and ask the user to retry later; log the `traceId`. |

Relevant conflict codes include:

```text
ACTIVE_APPOINTMENT_EXISTS
NO_CAPACITY
INVALID_TRANSITION
```

The model must not translate an error into a successful appointment or transition.

## Minimal workflow shape

```text
Evolution webhook or local fixture
  → Respond 200
  → Filter own/group/unsupported messages
  → Normalize phone and text
  → AI Agent with short-term memory
      → create appointment tool
      → lookup appointment tool
      → EN_CAMINO transition tool
  → Send or capture reply
```

Short-term n8n memory can help the dialogue but is never an appointment store. A
workflow restart may lose conversational context; it must not lose API records.

## Implementation tasks

1. Replace the canned stub call with environment-based AgroFlow API calls.
2. Define the three tools using the canonical request and response schemas.
3. Require explicit confirmation of extracted plate, farm, cut time, and load before creation.
4. Use the webhook phone automatically; never ask the model to invent it.
5. Handle `400`, `404`, `409`, and `500` paths explicitly.
6. Keep assignment, capacity, priority, and transition validation out of n8n.
7. Export the updated workflow without credentials or personal data.
8. Run the complete local path twice against a reset fictitious seed.

## Acceptance checklist

- [ ] `AGROFLOW_API_URL` is configurable and works from the n8n container.
- [ ] A valid conversation creates one persisted appointment.
- [ ] Repeating or retrying the conversation does not produce an invented confirmation.
- [ ] A phone lookup returns the same appointment displayed by the dashboard.
- [ ] `EN_CAMINO` is confirmed only after the API persists it.
- [ ] Invalid seeded references and business conflicts remain failures.
- [ ] The workflow contains no hardcoded internal UUIDs, credentials, or real personal data.
- [ ] Unsupported intents stay outside the Monday demo.
