# Self hosting Civocal

You can use your own provider accounts without changing application code: create the services below, generate fresh credentials in those accounts, and set the corresponding variables on your host. Civocal has no required dependency on the previous hosting or database accounts. It does still call the configured Gemini, Sarvam and Vobiz services for AI, speech and phone features; it is not a fully offline deployment.

## Services to set up

### Needed for a deployed app

1. **Container host:** any host that builds the root `Dockerfile`, supports HTTP and WebSockets, supplies a `PORT`, and can connect to PostgreSQL. Use one replica. A custom domain is optional, but a stable HTTPS address is recommended.
2. **Git hosting (for automatic deploys):** use a repository you control and point the `origin` remote to its URL before connecting it to your host. Git hosting is optional if you deploy source another way.
3. **PostgreSQL database:** create a new database in an account you control. Use a connection string with SSL as required by the provider. The database role must own the application schema (or be allowed to create tables, indexes, sequences and row-level security policies) because the container applies migrations at startup. Back up the database. Do not point a new deployment at someone else's production data.
4. **Google AI Studio / Gemini API key:** needed for the Gemma clerk. The default model names are in `backend/app/settings.py`; confirm that the models are enabled for your account and region.
5. **Sarvam API key:** needed for speech recognition and speech synthesis, including phone calls.

### Needed only for selected channels or features

- **Vobiz account and phone number:** needed for inbound phone calls and phone call-backs. Configure the number to send inbound calls to `https://YOUR_HOST/api/phone/answer`. The app generates its WebSocket stream URL and call-status callback URLs. Outbound call-back answer and hang-up URLs are generated per call. Confirm the provider supports secure WebSocket audio streaming and the required audio format.
- **OpenRouter key:** optional fallback if Gemini is slow or unavailable. The app uses the same configured Gemma model names through OpenRouter.
- **Telegram bot:** optional. Set a bot token, choose a long webhook secret, set `PUBLIC_BASE_URL`, then run `python -m scripts.telegram_webhook` from `backend` after the service is reachable. The script registers `https://YOUR_HOST/webhooks/telegram`.
- **Meta app with WhatsApp Cloud API:** optional. Configure the callback URL as `https://YOUR_HOST/webhooks/whatsapp`; use the same verify token in Meta and `WHATSAPP_VERIFY_TOKEN`, and provide the app secret and phone number ID.
- **DNS and TLS:** optional for testing on a stable host URL; recommended for launch and required by most real phone and messaging webhooks.

No separate Redis, object storage, map API, or frontend hosting service is required by the current code. Complaint photos are stored in PostgreSQL. The map uses OpenStreetMap data and tiles.

## Environment variables

Copy `.env.example` to `.env` for local development. In production, set these in your hosting provider's secret/environment settings instead of uploading a `.env` file.

| Variable | Required when | Notes |
| --- | --- | --- |
| `DATABASE_URL` | Always | PostgreSQL connection string. Migrations run automatically in Docker. |
| `GEMINI_API_KEY` | Using the AI clerk | Get a key in your own Google AI account. |
| `OPENROUTER_API_KEY` | Enabling fallback | Optional; fallback is skipped when blank. |
| `SARVAM_API_KEY` | Voice and phone features | Speech and translation provider key. |
| `CALLER_HASH_KEY` | Always in production | Generate a long random value and keep it permanently. Changing it breaks matching returning callers and invalidates officer sessions. |
| `CONTACT_ENCRYPTION_KEY` | Storing contact details for follow-up | Fernet key; leave blank to disable storing reach-back contact data. Generate with the command below. |
| `OFFICER_PASSCODE` | Staff actions | Set a strong shared passcode; blank disables officer sign-in. |
| `VOBIZ_AUTH_ID`, `VOBIZ_AUTH_TOKEN`, `VOBIZ_PHONE_NUMBER` | Phone service | Credentials and E.164 phone number from your own Vobiz account. |
| `PUBLIC_BASE_URL` | Phone callbacks / webhook configuration | Public origin, e.g. `https://civocal.example.com`, without a trailing slash. |
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET` | Telegram | Both must be set to enable the channel. |
| `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_APP_SECRET` | WhatsApp | Meta Cloud API credentials. |
| `FEATURE_PHOTO_CHECK`, `FEATURE_CALLBACK` | Optional feature switches | Default to `true`; set to `false` to disable. |
| `LLM_LIVE_MODEL`, `LLM_BACKGROUND_MODEL`, `LLM_PRIMARY_TIMEOUT_S` | Optional AI tuning | Defaults are declared in `backend/app/settings.py`. |
| `SARVAM_STT_MODEL`, `SARVAM_TTS_MODEL`, `SARVAM_TRANSLATE_MODEL`, `SARVAM_TTS_SPEAKER`, `SARVAM_TTS_PACE`, `SARVAM_TTS_TEMPERATURE` | Optional voice tuning | Defaults are declared in `backend/app/settings.py`. |
| `APP_VERSION` | Optional | Version string returned by `/health`. |

Generate the security keys from `backend` after installing requirements:

```powershell
python -c "import secrets; print(secrets.token_urlsafe(32))"
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

Use the first output for `CALLER_HASH_KEY` and the second for `CONTACT_ENCRYPTION_KEY`. Keep the hash key stable across deploys; store both as secrets. Do not reuse development values.

## Deploy steps

1. Create a new PostgreSQL database and a new deployment in accounts you control.
2. Create your own Git repository if you want automatic deploys. If needed, point the existing remote at it with `git remote set-url origin YOUR_REPOSITORY_URL`, then push your code. Connect that repository to your host, or deploy the checked-out source. Set the container build context to the repository root so Docker can copy both `backend/` and `web/`.
3. Add `DATABASE_URL`, fresh security keys, and the AI/speech keys in the host's environment settings. Add the Vobiz variables for phone service and channel variables only when enabling those channels.
4. Deploy the container. Startup runs `alembic upgrade head`, then starts FastAPI on the host's `PORT`. Check `/health`; it should report `status: ok` and `db: ok`.
5. Set `PUBLIC_BASE_URL` to the final HTTPS origin and configure Vobiz/Telegram/Meta webhook URLs. Redeploy if the host does not apply environment changes immediately.
6. Test the browser intake and dashboard first, then test each configured external channel with non-production sample reports.
7. Configure automated database backups, log retention and budget alerts in the accounts you control.

## Data and release safety

- A new `CALLER_HASH_KEY` means existing callers no longer match their older reports. Back it up securely and do not rotate casually.
- Schema migrations execute on every container startup. Before production schema changes, back up the database and review the Alembic migration.
- Do not run `python -m scripts.reset_demo --yes` against a live database; it deletes complaint data.
- The dashboard's report and officer-access model is designed for a pilot. Review authentication and public data exposure before opening it to real residents.
- Provider accounts have separate quotas, billing and regional/model availability. Check those limits in your own vendor dashboards before launch.
