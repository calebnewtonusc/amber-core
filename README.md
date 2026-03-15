# amber-core

The Amber platform core — memory engine, people graph, search, approvals, and identity. This repo is the shared foundation that all Amber deployments (e.g. `amber-sagar`) build on.

## Architecture

```
amber-core/
  apps/
    api/       — REST API (Express + Postgres + pgvector) → Railway
    worker/    — Background jobs (drift detection, reminders) → Railway
  packages/
    shared-types/    — TypeScript domain types (Person, Memory, ActionItem...)
    memory-engine/   — Ingestion, extraction (Claude), embedding pipeline
    people-graph/    — Hybrid search: semantic + structured + trust ranking
    storage/         — GCP Cloud Storage adapter (photos, transcripts, exports)
    prompts/         — Shared prompt templates
```

## Deployment

All services deploy to **Railway**. Each `apps/*` directory has its own `railway.toml`.

### Services

| Service | Path | Description |
|---|---|---|
| `amber-api` | `apps/api` | REST API, memory ingestion, people search, approvals |
| `amber-worker` | `apps/worker` | Background jobs, drift detection, proactive suggestions |

### Deploy to Railway

1. Connect this repo to Railway
2. Railway will detect each service from the `railway.toml` files
3. Set the environment variables from `.env.example` for each service
4. Add a Postgres plugin (Railway has managed Postgres with pgvector)
5. Run the schema: `psql $DATABASE_URL < apps/api/src/db/schema.sql`

### GCP Storage Setup

1. Create a GCP project
2. Create a Cloud Storage bucket (e.g. `amber-assets`)
3. Create a service account with `Storage Object Admin` role
4. Download the JSON key
5. Set `GCP_SERVICE_ACCOUNT_JSON` or `GCP_KEY_FILE` env var

## Storage

**Database**: Postgres with `pgvector` extension
**File storage**: GCP Cloud Storage (photos, transcripts, Loom, Fireflies, exports)
**Cache/Queue**: Redis via Upstash (optional)

## Development

```bash
pnpm install
pnpm dev          # starts all services in watch mode
pnpm build        # builds all packages
pnpm type-check   # TypeScript check
```

## Data Model

Core objects: `Person`, `Memory`, `ActionItem`, `RelationshipState`, `ApprovalTask`, `AmberIdentity`

See `packages/shared-types/src/index.ts` for the full schema.

## API

Base URL: `https://amber-api.railway.app`

| Method | Path | Description |
|---|---|---|
| GET | `/api/people` | List all people |
| GET | `/api/people/search?q=` | Hybrid people search |
| POST | `/api/people` | Create a person |
| POST | `/api/memories` | Ingest a memory |
| GET | `/api/memories` | List memories |
| GET | `/api/action-items` | Open action items |
| GET | `/api/approvals` | Pending approvals |
| PATCH | `/api/approvals/:id` | Approve / reject |

## Messaging

All Amber agents use [Loop Message](https://loopmessage.com) for iMessage delivery.

- API: `POST https://a.loopmessage.com/api/v1/message/send/`
- Auth: `Authorization: <LOOP_API_KEY>` (no Bearer)
- Body: `{ contact, text, sender }` where `sender` = sender UUID from dashboard
- Incoming: webhook `POST /webhook` — payload has `event`, `contact`, `text`, `message_id`

