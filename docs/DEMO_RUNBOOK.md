# Local demo runbook

Run the whole AgroFlow demo on one computer: API, dashboard, n8n, Evolution API and a real
WhatsApp test number. Any teammate can follow this on their own PC (macOS, Windows or Linux).

## How the pieces connect

```text
Carrier phone ──WhatsApp──▶ Bot phone (linked device)
                                 │  outbound connection, nothing is exposed
                                 ▼
                     Evolution API (Docker, 127.0.0.1:8080)
                                 │  webhook over the Docker network
                                 ▼
                     n8n "Chatbot" workflow (Docker, 127.0.0.1:5678)
                                 │  AI Agent + tools, via host.docker.internal
                                 ▼
                     AgroFlow API (host, localhost:5000)  ◀── Dashboard (localhost:5173)
```

- **Nothing is published to the Internet.** Evolution keeps an outbound connection to
  WhatsApp, like WhatsApp Web. Never expose port `8080`: it controls the WhatsApp account.
- The API is the source of truth. n8n only talks; the dashboard only reads and operates turns.

## 1. Prerequisites (once per PC)

| Tool | Why |
|---|---|
| Docker Desktop (or Docker Engine + Compose on Linux) | n8n, Evolution, Postgres, Redis |
| .NET 8 SDK | AgroFlow API |
| Node.js 20+ | Dashboard |
| Git | The three repositories |
| An OpenAI API key | The chatbot's language model |
| Two WhatsApp numbers | One for the bot, one acting as the carrier (see step 6) |

Clone the three repositories side by side:

```bash
git clone https://github.com/agustinvallante/AgroFlow.git
git clone https://github.com/GabrielBurieque/AgroFlow-Dashboard.git
git clone https://github.com/BraianMedrano/agroflow-chatbot.git
```

## 2. Where every key lives

| Secret | Where | How to get it |
|---|---|---|
| `N8N_ENCRYPTION_KEY` | `infra/.env` | `openssl rand -hex 32` |
| `EVOLUTION_API_KEY` | `infra/.env` | `openssl rand -hex 32` |
| `EVOLUTION_DB_PASSWORD` | `infra/.env` | `openssl rand -hex 24` |
| OpenAI API key | n8n credential **OpenAI account** | OpenAI dashboard |
| Evolution API key (same as `EVOLUTION_API_KEY`) | n8n credential **Header Auth account** | Copy from `infra/.env` |
| AgroFlow API | none | The local demo profile has no authentication |

Rules: never commit `infra/.env`, never paste keys in chats or issues, and never put a key
inside a workflow node. n8n stores credentials encrypted with `N8N_ENCRYPTION_KEY`; if you
change that value later, existing credentials stop working and must be created again.

On Windows without `openssl`, generate the values in PowerShell:

```powershell
-join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
```

## 3. Configure the chatbot stack (once per PC)

From `agroflow-chatbot/`:

```bash
cp infra/env.example infra/.env
chmod 600 infra/.env        # macOS/Linux only
```

Edit `infra/.env` and replace the three `replace-with-...` placeholders. The defaults for
`AGROFLOW_API_URL=http://host.docker.internal:5000` and `EVOLUTION_INSTANCE=agroflow-demo`
work as they are.

## 4. Start everything (every session)

Open three terminals.

**API** (`AgroFlow/`):

```bash
cd backend/Dsw2025Tpi.Api
dotnet run --launch-profile http --urls http://localhost:5000
```

Check <http://localhost:5000/health> returns `{"status":"Healthy"}`. On **Linux**, use
`--urls http://0.0.0.0:5000` instead: Docker reaches the host through its gateway address,
which cannot see a loopback-only listener.

**Dashboard** (`AgroFlow-Dashboard/`):

```bash
cp .env.example .env        # first time only; set VITE_DATA_SOURCE=http and VITE_API_URL=http://localhost:5000
npm install                 # first time only
npm run dev
```

Open <http://localhost:5173>.

**n8n + Evolution** (`agroflow-chatbot/`):

```bash
docker compose --env-file infra/.env -f infra/compose.yaml up -d
docker compose --env-file infra/.env -f infra/compose.yaml ps
```

Wait until all four containers are `healthy`. n8n is at <http://localhost:5678>.

## 5. Load the workflow in n8n (once per PC)

1. Open <http://localhost:5678> and create the local owner account (it only exists on your PC).
2. **Credentials → Create credential**:
   - **OpenAI** → name it `OpenAI account` → paste your OpenAI API key.
   - **Header Auth** → name it `Header Auth account` → Name: `apikey`, Value: your `EVOLUTION_API_KEY`.
3. **Workflows → Import from file** → `flujos-n8n/Chatbot.json`.
4. Open the nodes that show a credential warning and select the credential you just created:
   `OpenAI Chat Model` and `Send WhatsApp reply`. Credential ids from another PC never carry over.
5. Save and switch the workflow to **Active**. The production webhook is
   `http://n8n:5678/webhook/d01e7ba4-e412-47e5-a841-dafc71257beb` inside Docker.

`flujos-n8n/stub-agroflow-api.json` is a legacy offline stub. The chatbot no longer uses it;
do not import it.

## 6. Connect WhatsApp (once per PC and bot number)

Understand the two roles first:

- **Bot phone**: the number you link to Evolution. Its WhatsApp *becomes* the bot. Messages you
  type from this phone are ignored (`fromMe`), so you cannot test by writing to yourself.
- **Carrier phone**: a second WhatsApp that writes to the bot. The API identifies the carrier by
  this number, so it must exist in the API seed (step 7).

Use a spare number for the bot: Evolution uses Baileys, an unofficial WhatsApp client, and the
number can be banned.

1. Open <http://localhost:8080/manager> and log in with `EVOLUTION_API_KEY`.
2. Create an instance named exactly like `EVOLUTION_INSTANCE` (`agroflow-demo`), channel
   **Baileys**.
3. Open the instance, show the QR code, and on the bot phone go to
   **WhatsApp → Linked devices → Link a device** and scan it. The status must turn **open**.
4. In the instance's **Webhook** settings: enabled, URL
   `http://n8n:5678/webhook/d01e7ba4-e412-47e5-a841-dafc71257beb`, "webhook by events" off,
   base64 off, and only the `MESSAGES_UPSERT` event. Save.

The URL uses `n8n`, not `localhost`: Evolution calls n8n from inside the Docker network.

A linked session belongs to one Evolution instance. If several teammates run the demo, each one
links their own bot number, or only one PC keeps the shared bot number linked at a time.

## 7. Register the carrier phone in the API

The seed only knows two fictitious carriers: `+5493815550101` (truck `AF123BC`) and
`+5493815550102` (truck `AF456DE`); farms are `FINCA-NORTE` and `FINCA-SUR`. A real phone that is
not in the seed gets `REFERENCE_NOT_FOUND`.

To test with your real carrier phone, locally and without committing:

1. Stop the API.
2. In `AgroFlow/backend/Dsw2025Tpi.Data/Appointments/AppointmentsSeeder.cs`, replace
   `+5493815550101` with your carrier number in E.164 format. Argentine mobiles look like
   `+549` + area code + number, for example `+5493811234567`, which matches what WhatsApp sends.
3. Delete `backend/Dsw2025Tpi.Api/agroflow-demo.db` and its `-wal`/`-shm` files so the seed runs
   again from scratch.
4. Start the API again. Discard the seeder change before committing anything in `AgroFlow`.

## 8. Smoke test

From the carrier phone, write to the bot:

1. `Hola, quiero un turno` → the bot asks for plate, farm, cut time and load.
2. `AF123BC, FINCA-NORTE, cortamos hoy a las 6, 28 toneladas` → the bot repeats the data and
   asks for confirmation.
3. `Sí` → the bot confirms with the window assigned by the API. The turn appears in the
   dashboard queue within a few seconds.
4. `¿Qué turno tengo?` → the bot reads the same turn back from the API.
5. `Ya salí` → the bot reports `EN_CAMINO`; the dashboard shows the turn on its way.

Each window has capacity for two trucks. Delete the database (step 7.3) to reset between rehearsals.

## 9. Change the workflow

1. Edit in the n8n editor. The main places are:
   - **AI Agent → System message**: rules, tone, supported intents.
   - **create_appointment / lookup_appointments / report_en_camino**: the API calls. They read
     `$env.AGROFLOW_API_URL`; never hardcode hosts, phones or ids.
   - **Send WhatsApp reply**: uses `$env.EVOLUTION_INTERNAL_URL` and `$env.EVOLUTION_INSTANCE`.
2. Test by writing from the carrier phone and check **Executions** in n8n.
3. Export: **⋯ → Download**, replace `flujos-n8n/Chatbot.json` with the downloaded file, and
   review the diff. The export contains credential names and ids but never the secret values.
   Keep the webhook path unchanged so every teammate's Evolution configuration keeps working.
4. Commit on a branch and open a pull request. Teammates re-import the file (step 5.3) and
   re-select their own credentials.

## 10. Troubleshooting

| Symptom | Check |
|---|---|
| The bot never answers | n8n **Executions**: is there a run? If not, check the webhook URL and the `MESSAGES_UPSERT` event in Evolution, and that the workflow is **Active**. |
| Execution stops at the Filter | You wrote from the bot phone (`fromMe`), from a group, or sent a non-text message. |
| Tools fail with a connection error | The API is not running on port 5000, or on Linux it listens on `localhost` instead of `0.0.0.0`. |
| `$env` access denied | Recreate the n8n container: `docker compose ... up -d --force-recreate n8n`. |
| `REFERENCE_NOT_FOUND` | The carrier phone, plate or farm is not in the seed (step 7). |
| `NO_CAPACITY` | The window is full; reset the database or use a later cut time. |
| Reply node returns `401` | The `Header Auth account` value differs from `EVOLUTION_API_KEY`. |
| Instance keeps disconnecting | Your PC went to sleep or the phone unlinked the device. Relink by QR. |

## Stop and reset

```bash
docker compose --env-file infra/.env -f infra/compose.yaml down            # keeps data and WhatsApp session
docker compose --env-file infra/.env -f infra/compose.yaml down --volumes  # deletes n8n, Evolution and the session
```

Stop the API and dashboard with `Ctrl+C`.

## Optional: share the n8n editor or move to a server

- To show your n8n editor to someone remotely for a while, use a temporary tunnel only to n8n,
  for example `cloudflared tunnel --url http://localhost:5678`, and close it afterwards. Add
  header authentication to the webhook before sharing. Never tunnel Evolution.
- The VPS profile is documented in [VPS_DEPLOY.md](VPS_DEPLOY.md).
