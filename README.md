# Civocal

Civocal helps residents report local civic problems by phone or messaging. Its AI clerk gathers the details, confirms the location and complaint, and gives the resident a tracking number. Staff use the dashboard to review reports and manage follow-up.

## Design and workflow

The application has one Python/FastAPI service and a React/TypeScript dashboard. FastAPI serves both the API and the dashboard build. PostgreSQL stores complaints, calls, conversation turns, events and photos. OpenStreetMap data supplies place matches and map display. More detail is in [the architecture guide](docs/architecture.md).

```text
Phone → Vobiz ─┐
Browser ────────┤  FastAPI → Gemini/Gemma clerk → validated tools → PostgreSQL
Telegram ──────┤     ↕             ↕                    ↘ OpenStreetMap place data
WhatsApp ──────┘  Sarvam speech   Dashboard ← API + live events
```

## Run locally

Requirements: Python 3.11 and Node.js 22 or newer. A PostgreSQL database and provider API keys are needed for the corresponding live features. Copy `.env.example` to `.env` and fill in your own values; see [self hosting](docs/self-hosting.md) for setup and the complete service checklist.

```powershell
Copy-Item .env.example .env
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
alembic upgrade head
uvicorn app.main:app --reload
```

In a second terminal:

```powershell
cd web
npm ci
npm run dev
```

The dashboard is at `http://localhost:5173`; FastAPI is at `http://localhost:8000`. In development Vite forwards `/api` and `/health` to FastAPI. For a production container, the root `Dockerfile` builds the dashboard and packages it with the API. The container runs database migrations on startup and listens on the hosting platform's `PORT`.

## Main features

- Phone, browser, Telegram and WhatsApp complaint intake
- Kannada, Hindi and English conversation, speech recognition and speech output
- Location matching, duplicate reports, urgency and emergency handling
- Live complaint dashboard, map, status updates, office routing and CSV export
- Optional officer photo checks and citizen follow-up

## Project layout

- `backend/app`: API, channels, clerk, providers, database, and business logic
- `backend/migrations`: PostgreSQL schema migrations
- `backend/data`: OpenStreetMap-derived places, roads and pincodes used for matching
- `backend/tests`: backend tests
- `web/src`: dashboard and browser intake
- `docs`: architecture and self-hosting guides

## License

Apache-2.0; see [LICENSE](LICENSE).
