# Civocal design and workflow

## System design

Civocal is a single-process web application rather than a collection of independently deployed services. A FastAPI application owns the API, webhook endpoints, phone call handling, and the compiled React dashboard. The browser talks only to this server. In development, Vite serves the dashboard and proxies API requests to FastAPI; in production the Docker image bundles the built dashboard and Python application together.

PostgreSQL is the persistent store. Alembic applies the schema migrations at container startup. It stores complaints, caller hashes, calls, turns, event history, and uploaded photos. Complaint photos are stored in database columns, so a separate object storage service is not currently required. Database access is through SQLAlchemy and psycopg; the schema uses PostgreSQL JSONB, sequences, and row-level security.

OpenStreetMap data is checked into `backend/data` as a prebuilt local gazetteer, roads, and pincodes. The application uses it for place resolution and map coordinates. It does not query a hosted places database at runtime. The web map uses OpenStreetMap tiles.

The AI and communication providers are adapters called by the backend:

| Provider | Role | Required for |
| --- | --- | --- |
| Google Gemini API (Gemma models) | Conversation reasoning, classification and photo assessment | AI clerk and AI features |
| OpenRouter | Fallback route for the same Gemma models | Resilience when enabled; optional |
| Sarvam AI | Speech-to-text, text-to-speech and note translation | Phone and voice features |
| Vobiz | Phone number, inbound audio stream and outbound call-back | Phone intake and phone follow-up |
| Telegram Bot API | Telegram message intake and updates | Optional Telegram channel |
| Meta WhatsApp Cloud API | WhatsApp message intake and updates | Optional WhatsApp channel |

Provider calls are made server-side. Their keys must stay in server environment variables and must never be placed in the web build.

## Complaint workflow

1. A resident starts a conversation by phone, browser, Telegram, or WhatsApp.
2. The backend identifies the channel. Voice is transcribed by Sarvam; typed messages are passed through directly. Photos, voice notes and location pins are handled by their channel adapters.
3. The clerk uses Gemini/Gemma to ask for missing details and choose from backend tools. Place results and coordinates come from the local OpenStreetMap data. The clerk cannot write directly to the database.
4. Backend checks enforce the rules: validate the place, check nearby reports, and require a confirmation of the read-back before filing. A successful report is saved in PostgreSQL and assigned a tracking number.
5. Phone and voice conversations use Sarvam to speak the reply. Messaging channels return a message in the originating channel. The dashboard receives live events from the backend.
6. Staff sign into the dashboard to review, prioritize, route, and update reports. Marking a report fixed can save a photo, request an AI assessment, and start a citizen follow-up when the related options are configured.

## Operational limits to plan around

- Live event subscribers, active phone streams, sessions, rate limits, pending callbacks and retry timers are held in process memory. Run a single application replica; a restart drops in-progress work and pending timers.
- Browser intake rate limits use `X-Real-IP` when present. Configure the public reverse proxy to overwrite that header; if the host does not, visitors may share one rate-limit bucket. Do not accept a client-supplied value as trusted.
- There is no background job queue or durable scheduler. Add one before running multiple replicas or relying on callbacks across restarts.
- PostgreSQL is the only supported database engine in the current schema. Use any managed PostgreSQL service that supports the migration features documented above.
- The application uses one shared officer passcode, not individual staff accounts. Use a strong secret for a small pilot; add user-level identity and access controls before a wider public deployment.
