# Optional: VPS deployment

> The demo now runs locally. Follow [DEMO_RUNBOOK.md](DEMO_RUNBOOK.md) first; this document is kept only for a future VPS profile. The "Capture the first real webhook" section is historical: the workflow is already built in `flujos-n8n/Chatbot.json`.

This profile runs an isolated n8n and Evolution API demo stack on a shared VPS. The selected public n8n hostname is `n8n.casteltech.ar`. Capture one real Evolution API event before mapping any WhatsApp message fields.

## Quick validation

From the repository root, render both supported configurations without starting or changing any containers:

```bash
docker compose --env-file infra/env.example -f infra/compose.yaml config
docker compose --env-file infra/env.example -f infra/compose.yaml -f infra/compose.vps.yaml config
```

Both commands must exit with status `0`. The second command only renders the external `castel_network` reference; it does not require that network to exist until deployment.

The local rendering must set `N8N_PROXY_HOPS=0`. The VPS rendering must set `N8N_PROXY_HOPS=2` for the Cloudflare -> `castel_proxy` -> n8n chain.

## Run locally

### 1. Create local configuration

If `infra/.env` already exists, keep it and do not overwrite its secrets.

```bash
cp infra/env.example infra/.env
chmod 600 infra/.env
openssl rand -hex 32
openssl rand -hex 32
openssl rand -hex 24
```

Paste the generated values into `N8N_ENCRYPTION_KEY`, `EVOLUTION_API_KEY`, and `EVOLUTION_DB_PASSWORD`, respectively. Do not commit `infra/.env` or paste its contents into logs, issues, or chat.

### 2. Start the stack

```bash
docker compose --env-file infra/.env -f infra/compose.yaml config --quiet
docker compose --env-file infra/.env -f infra/compose.yaml pull
docker compose --env-file infra/.env -f infra/compose.yaml up -d
```

If an image registry is temporarily unavailable, stop after configuration rendering and retry `pull` later. Do not start a partially pulled stack.

Open n8n at <http://127.0.0.1:5678>. Open the Evolution API root at <http://127.0.0.1:8080> and follow its manager link when needed. If either bind port changes in `infra/.env`, use the new port.

### 3. Verify services

```bash
docker compose --env-file infra/.env -f infra/compose.yaml ps
curl --fail --silent --show-error http://127.0.0.1:5678/healthz/readiness
curl --fail --silent --show-error http://127.0.0.1:8080/
docker compose --env-file infra/.env -f infra/compose.yaml exec -T evolution-postgres sh -c 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
docker compose --env-file infra/.env -f infra/compose.yaml exec -T evolution-redis redis-cli ping
```

Expected results: all four containers are running, n8n and Evolution return successful HTTP responses, PostgreSQL reports `accepting connections`, and Redis reports `PONG`.

## Deploy to the VPS

The VPS override attaches only `agroflow-n8n` to the existing external `castel_network`. Evolution, PostgreSQL, and Redis remain on `agroflow_demo_backend`; PostgreSQL and Redis have no host ports. Both management ports bind only to `127.0.0.1`.

### 1. Confirm the public hostname

The tracked safe default is `N8N_PUBLIC_HOST=n8n.casteltech.ar`. A real VPS `infra/.env` remains authoritative and can override that value without changing the tracked sample. Keep the value as a hostname without a scheme or path.

### 2. Prepare the root-owned destination

These are **operator actions requiring sudo**:

```bash
sudo install -d -m 0750 /opt/agroflow-demo
sudo chown "$USER":"$(id -gn)" /opt/agroflow-demo
```

From the local repository, transfer the source without deleting remote runtime configuration:

```bash
rsync -av --exclude '.git/' --exclude '.atl/' --exclude '.codegraph/' --exclude '.DS_Store' --exclude 'infra/.env' ./ <vps-host>:/opt/agroflow-demo/
```

### 3. Configure and start AgroFlow only

On the VPS:

If `infra/.env` already exists from an earlier deployment, keep it and skip the copy command.

```bash
cd /opt/agroflow-demo
cp infra/env.example infra/.env
chmod 600 infra/.env
openssl rand -hex 32
openssl rand -hex 32
openssl rand -hex 24
```

Edit `infra/.env`: replace the three secret placeholders and confirm `N8N_PUBLIC_HOST=n8n.casteltech.ar`. Then run:

```bash
docker network inspect castel_network >/dev/null
docker compose --env-file infra/.env -f infra/compose.yaml -f infra/compose.vps.yaml config --quiet
docker compose --env-file infra/.env -f infra/compose.yaml -f infra/compose.vps.yaml pull
docker compose --env-file infra/.env -f infra/compose.yaml -f infra/compose.vps.yaml up -d
docker compose --env-file infra/.env -f infra/compose.yaml -f infra/compose.vps.yaml ps
```

These commands target only the `agroflow-demo` Compose project. They do not recreate or restart existing Castel, Palestra, or Engram services.

### 4. Satisfy the public-route prerequisites

Create a proxied Cloudflare DNS record for `n8n.casteltech.ar` that targets the same origin as the current Castel site. Keep the existing strict origin-validation policy; do not lower TLS verification to work around certificate errors. Do not create a public Evolution API record.

The local artifact at `infra/nginx/n8n.casteltech.ar.conf` is intentionally blocked. Read-only metadata from the certificate currently referenced by the `casteltech.ar` vhost did not show an exact `n8n.casteltech.ar` SAN or a `*.casteltech.ar` wildcard SAN. Before installation, the operator must provide all of this evidence:

- certificate metadata with `DNS:n8n.casteltech.ar` or `DNS:*.casteltech.ar` in the Subject Alternative Name extension;
- the verified in-container certificate and matching private-key paths under the existing read-only `/etc/nginx/certs` mount;
- an updated local artifact containing those exact paths as `ssl_certificate` and `ssl_certificate_key` directives.

Inspect certificate metadata only; do not print, read, or copy private-key contents. Do not reuse the current Castel certificate paths without the missing SAN evidence.

### 5. Install and enable the reviewed Nginx route

Proceed only after the certificate blocker above is resolved and the two certificate directives have been added to the artifact. The exact host installation destination is `/opt/castel-stack/nginx/conf.d/n8n.casteltech.ar.conf`; the container sees it as `/etc/nginx/conf.d/n8n.casteltech.ar.conf`.

Before changing Nginx, run the documented HTTPS smoke checks for every existing Castel, Palestra, and Engram public service and record their status. Then perform these **operator actions requiring sudo**:

```bash
sudo install -m 0644 /opt/agroflow-demo/infra/nginx/n8n.casteltech.ar.conf /opt/castel-stack/nginx/conf.d/n8n.casteltech.ar.conf && sudo docker exec castel_proxy nginx -t && sudo docker exec castel_proxy nginx -s reload
```

Run the reload command only when `nginx -t` succeeds. Then verify the new route and repeat every pre-existing service smoke check; their status and response must remain unchanged:

```bash
curl --fail --silent --show-error https://n8n.casteltech.ar/healthz/readiness
```

The vhost routes only `n8n.casteltech.ar` to `http://agroflow-n8n:5678` over `castel_network`. Do not restart `castel_proxy`, edit an unrelated vhost, or publish port `5678` to the Internet.

## Use SSH tunnels

For loopback-only management from a workstation:

```bash
ssh -N -L 5678:127.0.0.1:5678 -L 8080:127.0.0.1:8080 <vps-host>
```

Use the manager link returned by <http://127.0.0.1:8080> for Evolution management. Use the public HTTPS hostname for n8n owner setup once Nginx is ready; the direct n8n tunnel remains useful for health checks.

## Capture the first real webhook

Keep this sequence manual and observable:

1. Deploy and verify all four services.
2. Open the public n8n editor and create the initial owner account.
3. Create a workflow containing only a generic `POST` Webhook node. Give it a clear capture-only path and start listening for a test event. Do not add field extraction yet.
4. Create and connect an Evolution instance using `WHATSAPP-BAILEYS` and a secondary, disposable WhatsApp number.
5. Configure that instance—not the disabled global webhook—to send only `MESSAGES_UPSERT` to the n8n test webhook. From Evolution's private network, use the n8n service origin `http://n8n:5678` with the test webhook path shown by n8n.
6. Send one real text message to the connected number while n8n is listening.
7. Inspect the captured payload and record the field names actually emitted by Evolution API `2.3.7`. Redact phone numbers, message content, API keys, and identifiers before sharing any evidence.
8. Only then implement own-message, group, type, sender, and text extraction followed by the conversation state machine.

The n8n test webhook is temporary and works only while listening. When the capture flow is ready for repeatable use, activate it and change Evolution to the corresponding production `/webhook/` URL. Export future workflows to `automation/n8n/workflows/`; create that directory with the first real export rather than keeping empty scaffolding.

## Stop or roll back

Stop and remove only AgroFlow containers and its dedicated bridge while preserving demo data:

```bash
docker compose --env-file infra/.env -f infra/compose.yaml -f infra/compose.vps.yaml down
```

For local-only use, omit `-f infra/compose.vps.yaml`.

Deleting demo data is a separate, destructive decision:

```bash
docker compose --env-file infra/.env -f infra/compose.yaml -f infra/compose.vps.yaml down --volumes
```

If the Nginx vhost was installed and `nginx -t` failed, remove only the new file and rerun validation; the failed candidate was never loaded:

```bash
sudo rm /opt/castel-stack/nginx/conf.d/n8n.casteltech.ar.conf
sudo docker exec castel_proxy nginx -t
```

If the vhost was already reloaded, remove only the new file, validate the complete configuration, and reload it:

```bash
sudo rm /opt/castel-stack/nginx/conf.d/n8n.casteltech.ar.conf
sudo docker exec castel_proxy nginx -t && sudo docker exec castel_proxy nginx -s reload
```

Repeat every pre-existing public-service smoke check after rollback. Remove or disable only the `n8n.casteltech.ar` Cloudflare DNS record when public rollback is required. Never run a global Docker prune. The local vhost artifact, external `castel_network`, unrelated Nginx files, and every unrelated container or volume are outside the runtime rollback boundary.

## Known limits and risks

- Evolution's Baileys integration is unofficial WhatsApp automation. Use only a disposable test number and no real operational data.
- Evolution's GitHub release is `2.3.7`, while its verified Docker Hub tag is `v2.3.7`; the Compose file adds that required `v` prefix.
- n8n uses SQLite in its dedicated volume for this demo; it is not the production AgroFlow business database.
- Resource limits reserve headroom for existing VPS services but must be observed during the real message test.
- Redis has no password because it has no host port and only joins the dedicated backend network. Do not attach unrelated containers to that network.
- Public TLS installation remains blocked until certificate SAN coverage for `n8n.casteltech.ar` and the matching mounted key path are verified.
- `N8N_PROXY_HOPS=2` assumes every public request follows Cloudflare -> `castel_proxy` -> n8n. If direct origin access is possible, restrict it before relying on proxy-derived client addresses.
- No workflow or WhatsApp payload schema is included in this work unit by design.
