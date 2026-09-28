# Feature: local-api-chatbot

## Objective

Run the whole demo locally (AgroFlow API on the host, n8n + Evolution API in Docker) and
connect the WhatsApp chatbot to the real AgroFlow appointments API instead of the stub.

## Problem

- `flujos-n8n/Chatbot.json` still calls the canned stub workflow.
- The n8n container never receives `AGROFLOW_API_URL`, and Linux hosts cannot resolve
  `host.docker.internal` without a `host-gateway` mapping.
- The Evolution host and instance name are hardcoded in the reply node.
- The runbook is VPS-first; the demo is now local-first.

## Why

AgroFlow API (`agustinvallante/AgroFlow` PR #61) and the dashboard (`fab4cf5`) are merged on
`main`. The chatbot is the remaining piece of the local end-to-end path.

## Scope

- Infra: pass API and Evolution settings into n8n; add the host-gateway mapping.
- Workflow: AI Agent (OpenAI) with short-term memory and three HTTP tools:
  create appointment, look up by phone, report `EN_CAMINO`.
- Docs: local-first runbook, WhatsApp pairing with a test phone, seed data notes.

## Constraints

- Evolution API stays bound to `127.0.0.1`; it is never exposed publicly. Baileys keeps an
  outbound connection to WhatsApp, and the webhook to n8n travels over the Docker network.
- n8n owns the conversation; the API owns the record (`docs/API_CONTRACT.md`).
- No credentials, UUIDs, or real personal data in the exported workflow.
- VPS override (`compose.vps.yaml`) is kept as an optional profile.

## Configuration

- TDD: on (global Strict TDD). No automated test runner exists for n8n workflow JSON or
  Compose files; checks are static (`jq`, `docker compose config`) plus the manual local run.
- Delivery strategy: `single-pr` (forecast ~450 authored lines, mostly workflow JSON).
- RDD: on (global).

## Tasks

- [x] T1 — Infra: pass `AGROFLOW_API_URL`, `EVOLUTION_INTERNAL_URL`, `EVOLUTION_INSTANCE`
  into n8n; add `host.docker.internal:host-gateway`; update `env.example`.
  Route: inline (2 mechanical files).
- [x] T2 — Workflow: replace stub call with AI Agent + memory + 3 API tools; env-driven reply.
  Route: delegated writer failed (no write permission; reported false success), redone inline.
- [x] T3 — Docs: local-first runbook usable from any PC (setup, start, API keys/credentials,
  WhatsApp pairing, editing and re-exporting the flow), seed phone requirement, README.
  Scope extended by the user on 2026-09-28. Route: inline (delegated writer lacked permissions).

## Acceptance criteria

- `docker compose --env-file infra/env.example -f infra/compose.yaml config` resolves the new vars.
- `Chatbot.json` is valid JSON, has no stub URL, no hardcoded instance/host, no UUIDs or secrets.
- A message from a seeded carrier phone creates an appointment visible in the dashboard
  (manual check, requires Docker + .NET 8).

## Progress

- Pulled `AgroFlow` to `f97daef` and `AgroFlow-Dashboard` to `fab4cf5`.
- T1: `docker compose config` (local and local+vps) exit 0; resolves AGROFLOW_API_URL,
  EVOLUTION_INSTANCE, EVOLUTION_INTERNAL_URL, N8N_BLOCK_ENV_ACCESS_IN_NODE, host-gateway.
  n8n 2.0 release notes say env access is blocked by default, so it is set explicitly.
  Commit ac59ce4. Review: assessed medium; user declined review for this candidate.

- T2: `jq empty` ok; 11 nodes, all connection endpoints exist; no stub URL, Evolution host or
  instance hardcoded. API requires E.164 with `+` (AppointmentsController PhonePattern), so the
  tools prefix `+` and send `phone` as a query parameter (encoded). Reply body built with
  JSON.stringify to survive quotes/newlines. Not import-tested (no Docker daemon).

- T3: DEMO_RUNBOOK.md rewritten local-first (any PC, keys table, n8n credentials/import,
  WhatsApp pairing with bot vs carrier phone, seed override, smoke test, editing/exporting the
  workflow, troubleshooting). Old VPS runbook moved unchanged to VPS_DEPLOY.md with a banner.
  API_CONTRACT examples fixed (+E.164, FINCA-NORTE). Repo URLs verified from git remotes.
  Structural readback only; not executed end to end.
  Commit d469250. Review: slice_budget_reached, medium; user declined review for the branch slice.

## Pending checks

- Manual end-to-end run (needs Docker daemon + .NET 8): import workflow into n8n 2.37.10,
  confirm node typeVersions load, tools return problem+json to the agent (neverError), and a
  seeded carrier creates an appointment visible in the dashboard.

## Next step

Manual end-to-end run, then push and PR (user decision).

## Post-merge fix (fix/chatbot-reply-and-prompt)

- The merged `Chatbot.json` (PR #5) is the late delegated writer's version, which overwrote the
  inline version before commit. Fixed after merge: system message lacked the `=` expression
  prefix (the agent never saw the current date), the reply body interpolated the model output
  into raw JSON (breaks on quotes/newlines), and runbook node names did not match.
