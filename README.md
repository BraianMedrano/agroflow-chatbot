# AgroFlow chatbot demo

This repository owns the local n8n and Evolution infrastructure plus workflow exports for the AgroFlow chatbot. Everything runs on one computer: WhatsApp → Evolution API → n8n AI Agent → AgroFlow API, with the dashboard reading the same API. Nothing is exposed to the Internet. AgroFlow API remains the source of truth for appointments and business rules.

- [Minimum local API integration](docs/API_CONTRACT.md)
- [Local demo runbook](docs/DEMO_RUNBOOK.md): setup on any PC, keys, WhatsApp pairing, editing the workflow
- [Optional VPS deployment](docs/VPS_DEPLOY.md)
- [Project and architecture context](AGROFLOW-CONTEXTO-N8N-DEMO-Y-ARQUITECTURA.md)

The Monday demo runs locally and supports only appointment creation, lookup by caller phone, and the `EN_CAMINO` transition. Authentication, deployment, incidents, maps, and production hardening remain out of scope.
