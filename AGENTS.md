# AI Avatar System — Agent Reference

## 1. Scope and project identity

This document is a source-based reference for this repository. It describes the implementation found under `backend/`, `frontend/`, compose files, scripts, migrations, and tests. Claims in `README.md` that are not backed by inspected source are marked `UNKNOWN / NOT VERIFIED`.

The project is a web application for authenticated users to upload an avatar image, optionally clone a voice, create avatar sessions, exchange text/voice turns, persist history, and stream synthesized avatar media. The backend is FastAPI/Python; the client is Next.js/React/TypeScript. PostgreSQL stores application records, Redis supports caching and Celery, and local filesystem or S3 stores media.

## 2. Repository map

| Path | Responsibility |
|---|---|
| `backend/main.py` | FastAPI app, lifespan, middleware, routers, health/root endpoints. |
| `backend/app/config.py` | Pydantic settings, environment variables, production/secret validation. |
| `backend/app/api/v1/` | REST route modules: users, avatars, sessions, conversations, messages, voices. |
| `backend/app/models.py` | SQLAlchemy models and indexes. |
| `backend/app/schemas.py` | Pydantic request/response models. |
| `backend/app/database.py` | Async SQLAlchemy engine, session factory, `get_db`. |
| `backend/app/services/` | Storage, cache, avatar processing/animation, STT, TTS, LLM. |
| `backend/app/websocket.py` | WebSocket connection manager and live turn pipeline. |
| `backend/app/celery_app.py` | Celery app and background tasks. |
| `backend/app/middleware/` | Security headers, request logging, rate limiting. |
| `backend/alembic/` | Schema migration environment and revisions `0000`–`0004`. |
| `backend/musetalk_worker.py` | MuseTalk worker process entrypoint. |
| `backend/tests/` | Pytest unit, integration, WebSocket, hardening, and regression tests. |
| `frontend/app/` | Next App Router root layout/page and metadata routes. |
| `frontend/components/` | Auth, avatar, chat, history, settings, voice, and UI components. |
| `frontend/lib/api.ts` | Axios REST client and auth token handling. |
| `frontend/lib/types.ts` | Client types. |
| `frontend/lib/vad.ts` | Voice activity detection support. |
| `frontend/store/useStore.ts` | Persisted Zustand auth/session/connection state. |
| `docker-compose.yml` | Main local compose stack. |
| `docker-compose.cpu.yml` | CPU Dockerfile variant. |
| `docker-compose.gpu.yml` | GPU reservations/default Dockerfile variant. |
| `docker-compose.prod.yml` | Production compose configuration. |
| `nginx/nginx.conf` | Reverse proxy configuration. |
| `infrastructure/*.tf` | Terraform AWS/ECS resources. |
| `scripts/` | MuseTalk, face restoration, LiveAvatar setup, demo seed, AWS deployment. |

## 3. Stack and execution model

- Backend: Python, FastAPI, SQLAlchemy async, asyncpg, Pydantic Settings, Alembic, JWT (`python-jose`/implementation in `tokens.py`), bcrypt/passlib-compatible password hashing as used by `users.py`.
- Frontend: Next.js 16, React 18, TypeScript, Axios, Zustand, TanStack React Query, Tailwind CSS, `react-hot-toast`, Lucide icons.
- Data: PostgreSQL 15 in compose; Redis 7 with append-only persistence and LRU max-memory policy.
- AI/media: Anthropic, OpenAI-compatible providers or Ollama through `app/services/llm.py`; Whisper/faster-whisper path in `stt.py`; Chatterbox with Edge TTS and gTTS fallbacks in `tts.py`; MuseTalk, optional GFPGAN, LiveAvatar and simple/FFmpeg engine paths in animator/setup/config code.
- Background work: Celery broker/result Redis databases 1 and 2; Flower is included in the compose stack.
- Storage: local `uploads` by default, or S3/CloudFront through `storage.py` and settings.
- Monitoring: Prometheus instrumentation, optional Sentry, optional OpenTelemetry.

## 4. Architecture and data flow

```text
Browser (Next.js)
  ├─ Axios REST ───────────────► FastAPI routers ─► services ─► PostgreSQL/Redis/storage
  └─ WebSocket /ws/{session_id} ► ConnectionManager
                                  ├─ optional audio ► STT
                                  ├─ streamed text ► LLM provider
                                  ├─ sentence chunks ► TTS
                                  ├─ audio + avatar ► animator/MuseTalk/simple
                                  └─ media/text events ► browser
```

Persisted ownership chain: `User` owns `Avatar` and `Session`; `Session` owns `Message` and `Conversation`. Foreign keys use cascade deletes in both models and migrations. The WebSocket reloads session/avatar/history data, keeps an in-memory bounded context, persists messages, and can roll evicted context into a conversation summary when memory is enabled.

## 5. Docker and infrastructure

The compose services are `postgres`, `redis`, `backend`, `frontend`, `celery-worker`, `flower`, and optional `nginx`. Postgres is published only on `127.0.0.1:5432`; Redis on `127.0.0.1:6379`; backend on `8000`; frontend on `3000`; Flower on loopback `5555`; nginx publishes `80`/`443`. Postgres and Redis have healthchecks. Backend/worker mount the source plus named media/voice volumes. The normal compose file requests an NVIDIA GPU for backend and worker; `docker-compose.cpu.yml` selects `Dockerfile.cpu` and removes that reservation. Compare the exact selected compose file before changing ports, volumes, or service names.

`docker-compose.prod.yml` is the production compose variant; `backend/entrypoint.sh` runs production startup/migrations according to its script. `backend/Dockerfile`, `Dockerfile.cpu`, and `Dockerfile.gpu` define image differences. `nginx/nginx.conf` proxies browser traffic and WebSockets. AWS alternatives are represented by `infrastructure/main.tf`, `ecs.tf`, `variables.tf`, `deploy.sh`, and `scripts/deploy-aws.sh`; the exact deployed topology should be rechecked before relying on README wording.

## 6. Backend boot, middleware, and health

`backend/main.py` creates the app and includes routers under `/api/v1/{users,avatars,sessions,conversations,messages,voices}`. Lifespan creates tables only in `DEBUG`; production expects `alembic upgrade head` before app startup. It initializes storage/cache, mounts local files at `/uploads` when local storage is enabled, and starts WebSocket cleanup.

Middleware includes GZip, rate limiting, CORS with credentials and configured origins, request logging, and security headers. Prometheus is exposed when enabled; Sentry and OpenTelemetry are optional. `/health` checks Postgres and Redis as hard dependencies and reports GPU/LLM readiness; unhealthy dependencies return HTTP 503. `/` identifies API version/status. Interactive docs are disabled when `DEBUG` is false.

## 7. REST API reference

All route implementations are in `backend/app/api/v1/`; response details must be taken from `backend/app/schemas.py` rather than inferred here.

### Users/auth (`users.py`)

- `POST /api/v1/users/register` creates a normal user.
- `POST /api/v1/users/login` accepts OAuth2 form fields (`username` is the email), creates a JWT, and sets the configured httpOnly auth cookie.
- `POST /api/v1/users/guest` creates an isolated disposable guest user and JWT when enabled.
- `POST /api/v1/users/logout` clears the auth cookie.
- `GET /api/v1/users/me`, `PUT /api/v1/users/me` read/update the current user.
- `GET /api/v1/users/`, `GET /api/v1/users/{user_id}` are user listing/detail routes with authorization logic in the module.

### Avatars (`avatars.py`)

- `POST /api/v1/avatars/upload` streams and validates an image upload, stores it, creates processing metadata, and returns an avatar.
- `GET /api/v1/avatars/`, `GET /api/v1/avatars/{avatar_id}` list/read owned avatars.
- `PUT /api/v1/avatars/{avatar_id}/voice` assigns/unassigns a voice.
- `PATCH /api/v1/avatars/{avatar_id}/metadata`, `PATCH /api/v1/avatars/{avatar_id}/name` update metadata/name.
- `DELETE /api/v1/avatars/{avatar_id}` deletes an owned avatar and related session records through cascades.

### Sessions/conversations/messages

- Sessions: `POST /api/v1/sessions/create`, `GET /api/v1/sessions/`, `GET /api/v1/sessions/{id}`, `POST /api/v1/sessions/{id}/end`, `GET /api/v1/sessions/{id}/export`, `DELETE /api/v1/sessions/{id}`.
- Conversations: list/detail by id or session, create, rename, summarize, delete (`backend/app/api/v1/conversations.py`).
- Messages: `POST /api/v1/messages/send`, list by session, get/edit/delete by id (`messages.py`). The REST send path is separate from the richer WebSocket streaming path.

### Voices (`voices.py`)

- `POST /api/v1/voices/clone` accepts an audio sample and records a voice profile/index entry.
- `GET /api/v1/voices/`, `GET /api/v1/voices/{voice_id}` list/read owned voices.
- `POST /api/v1/voices/{voice_id}/synthesize` creates a preview; `GET /api/v1/voices/{voice_id}/preview` serves it.
- `DELETE /api/v1/voices/{voice_id}` deletes a voice and clears avatar references as implemented.

## 8. Authentication and authorization

`users.py` hashes passwords, truncating input to bcrypt's 72-byte limit before hashing/verification. `tokens.py` creates/decodes JWTs with required `sub` and `exp` claims, configured algorithm/secret, and expiration. Login/guest responses contain a token while the server also sets `AUTH_COOKIE_NAME` (default `access_token`) as httpOnly; production requires secure cookies through settings validation. API dependencies accept the cookie/bearer flow and enforce current-user ownership. Guest accounts are real users marked `is_guest`, retained according to `GUEST_RETENTION_HOURS`, and cleaned by a Celery task.

The frontend persists token/user state in Zustand localStorage (`avatar-system-storage`), sends bearer auth with Axios, and enables `withCredentials` for the cookie. A global 401 interceptor clears persisted auth and emits `auth:logout`. Logout calls the server endpoint and clears client state through the UI flow. WebSocket code uses `api.getToken()` for authentication; inspect the client construction in `ChatInterface.tsx` when changing WS auth.

## 9. Database

`backend/app/models.py` defines `users`, `avatars`, `sessions`, `messages`, and `conversations`. Important fields include user identity/password/guest flag; avatar owner/name/image and voice/storage metadata; session owner/avatar/status/settings; message role/content/type/audio/video/tokens/latency; conversation title/summary/count. Useful indexes cover owner lookups, recency ordering, voice references, session message ordering, and active/status queries.

`backend/app/database.py` uses an async engine and converts `postgresql://` to `postgresql+asyncpg://`; SQLite-specific pooling is accommodated in code, but SQLite is UNKNOWN / NOT VERIFIED as a supported project deployment. Migrations `0000_initial_schema.py`, `0001_add_voice_id_to_avatars.py`, `0002_add_perf_indexes.py`, `0003_fk_on_delete_cascade.py`, and `0004_add_is_guest_to_users.py` are the authoritative production schema history.

## 10. Redis, Celery, and background work

`app/services/cache.py` uses Redis at `REDIS_URL`. Celery uses broker DB 1 and result backend DB 2. `celery_app.py` defines avatar processing, video generation, old-file cleanup, and guest-account cleanup tasks. The compose worker runs `celery -A app.celery_app worker --loglevel=info --concurrency=2`; Flower runs on port 5555. Task behavior and whether each path is used by a current API route must be checked in `celery_app.py` before modifying it.

## 11. Avatar/media system

Avatar upload validation is in `avatars.py`; it validates UUIDs/extensions, caps the streamed body using configured limits, stores through `storage_service`, and records URLs/keys/status. `storage.py` provides local filesystem and S3 implementations; local files are mounted/served by `/uploads` in `main.py`. `avatar_processor.py` handles image processing/metadata. `animator.py` selects the configured engine (`musetalk`, `liveavatar`, or `simple`) and produces media for the WebSocket/Celery paths. Temporary per-session files in `websocket.py` are owner-only (`0700` directories/`0600` files).

MuseTalk is represented by `backend/musetalk_worker.py`, `services/animator.py`, `MUSETALK_PATH`, and `scripts/setup_musetalk.sh`. The worker is intended to keep models loaded. `FACE_RESTORE=gfpgan` is optional and configured through `scripts/setup_face_restore.sh`. LiveAvatar is an alternative offline/slow engine configured by `LIVEAVATAR_*`; its real-time suitability is explicitly limited in `config.py`. The `simple` engine is the CPU/static-image fallback. Exact generated-file codecs and browser playback details are implementation-specific and should be inspected in `animator.py` and the media handling portions of `ChatInterface.tsx` before changes.

## 12. Voice and AI services

### STT

`services/stt.py` exposes `STTService`; settings select `whisper`, `google`, or `azure`, with `WHISPER_MODEL` defaulting to `large-v3-turbo`. The WebSocket receives audio and routes transcription through this service. Provider availability and model download requirements are environment-dependent.

### TTS and voice cloning

`services/tts.py` uses Chatterbox when configured/available, optionally with a speaker WAV from `voice_profiles`. On failure it falls back to Edge TTS, then gTTS; fallback loses voice cloning. `voices.py` enforces 10–60 second samples, language allowlist, ownership, preview generation, and an index file at `voice_profiles/index.json`. `requirements-tts.txt` is optional and may require a separate environment as documented by repository comments.

### LLM

`services/llm.py` abstracts Anthropic, OpenAI-compatible, and Ollama providers. `LLM_PROVIDER`, `LLM_MODEL`, temperature/token limits, API keys, `OPENAI_BASE_URL`, and optional reasoning effort control requests. Errors are mapped to rate-limit/auth/unavailable/general classes. `websocket.py` streams model output, chunks text at clause/sentence boundaries, and starts TTS/media work per chunk. Conversation memory is a bounded recent context (`MAX_CONTEXT_MESSAGES`); overflow can be summarized and carried into the system prompt. Tool/function calling is UNKNOWN / NOT VERIFIED; no such route is evident in the inspected symbols.

## 13. WebSocket and conversation pipeline

The WebSocket endpoint is registered in `main.py`/handled by `ConnectionManager` in `backend/app/websocket.py`; verify the exact URL and event names there before client changes. Connection state includes session/user/avatar/voice/system prompt/memory. It authenticates the session owner, loads persisted context, limits total connections, keeps alive/pings, reaps stale sessions after two hours, and serializes sends.

The intended verified live sequence is: browser text or recorded audio → optional STT → LLM streaming → sentence/clause chunking → TTS WAV → avatar animator → WebSocket text/audio/video events → browser playback → persistence of user/assistant messages. A new turn cancels an in-flight turn (barge-in). Errors are mapped to user-actionable provider messages. REST session/message routes provide history, edit/delete, export, and non-streaming operations.

## 14. Frontend

The app uses the Next App Router with a root page (`frontend/app/page.tsx`) and layout. The main UI is assembled from `AuthModal`, `AvatarList`, `AvatarUpload`, `ChatInterface`, `HistoryPanel`, `SettingsPanel`, `VoicePanel`, `WaveformVisualizer`, and connection/error UI components. `components/providers/QueryProvider.tsx` supplies React Query. `lib/api.ts` is the single REST client; `useStore.ts` holds persisted auth, selected avatar/session, hands-free preference, and WS status.

`ChatInterface.tsx` is the highest-value frontend file for the end-to-end turn: inspect it with `backend/app/websocket.py` when changing WS protocol, media URLs, auth, reconnect behavior, or cancellation. `AvatarUpload.tsx`/`AvatarList.tsx` couple to avatar routes; `VoicePanel.tsx` couples to voice clone/preview routes; `HistoryPanel.tsx` couples to session/conversation/message routes. Browser microphone/audio and VAD behavior is split between `ChatInterface.tsx`, `VoicePanel.tsx`, and `lib/vad.ts`. Frontend tests are not present in the repository; `package.json` provides lint, typecheck, build, dev, and start scripts.

## 15. Configuration and security

Use `.env.example`, `.env.prod.example`, and `backend/app/config.py` as the configuration sources. Never copy values from `.env` into documentation; secrets are configured through environment variables. Important groups are database/Redis, storage/AWS, LLM, avatar engine, STT/TTS, JWT/cookies, CORS, guest retention, upload limits, WebSocket limits, Celery, URLs, monitoring, and OpenTelemetry.

Production settings reject `DEBUG=true`, insecure cookie/CORS settings, and weak `SECRET_KEY`/`JWT_SECRET_KEY`. CORS credentials require explicit origins. Upload sizes/extensions are capped. Rate limiting is middleware-based. Security headers are in `middleware/security.py`; exact header set must be read there. Local static uploads are intentionally served by `/uploads`; review access implications before changing storage exposure.

## 16. Development and safe commands

From the repository root, verified documented commands include:

```bash
cp .env.example .env
docker compose -f docker-compose.cpu.yml up -d
docker compose ps
docker compose logs -f backend
docker compose exec backend alembic upgrade head
docker compose exec backend pytest
cd frontend && npm run lint && npm run typecheck && npm run build
```

`docker compose up -d` uses the default compose file and may request NVIDIA resources; select CPU/GPU intentionally. `docker compose down` stops/removes containers but normally preserves named volumes. `docker compose down -v` removes named volumes including Postgres/Redis/media data and is destructive—do not run casually. `scripts/seed_demo.py` mutates application data; setup scripts download large models and change local model state. Do not run deployment scripts or Terraform without reviewing target account/region/state and environment first.

## 17. Testing

Backend tests use pytest and are configured by `backend/pytest.ini` and `backend/tests/conftest.py`. Coverage includes health, tokens/auth hardening, users, avatars, sessions, conversations/memory, WebSockets/WS auth/e2e, fallback behavior, animator/LiveAvatar, telemetry, face restoration, regressions, and infrastructure hardening. Run `pytest` from `backend` (or the compose exec equivalent) with required services/config available. Frontend has no test runner script or test directory verified; use lint/typecheck/build as available checks. Coverage percentage is UNKNOWN / NOT VERIFIED.

## 18. Deployment

There are two visible deployment families: compose production (`docker-compose.prod.yml`, nginx, production env) and AWS/Terraform (`deploy.sh`, `scripts/deploy-aws.sh`, `infrastructure/`). GPU-specific model requirements are substantial; inspect selected Dockerfile and setup script before deployment. `backend/entrypoint.sh` and Alembic govern production startup schema. AWS resource details must be read from Terraform rather than inferred from README claims.

## 19. Known issues, risk areas, and incomplete verification

- README and source comments describe several claimed performance numbers, model versions, AWS instance behavior, and “production-ready” capabilities. These are claims, not independently measured here: UNKNOWN / NOT VERIFIED.
- `docker-compose.yml` comments explain that environment variables are passed broadly via `env_file`, with internal service-name overrides; changing `.env` or compose precedence can silently alter runtime behavior.
- Multiple avatar engines and optional model stacks create configuration-dependent behavior. MuseTalk, LiveAvatar, GFPGAN, Chatterbox, cloud LLMs, and optional providers cannot all be assumed installed in every image.
- Frontend protocol/media details, exact WebSocket event schema, and all response bodies must be verified in `ChatInterface.tsx`, `schemas.py`, and the remainder of `websocket.py` before API changes.
- Frontend automated tests and numeric coverage are UNKNOWN / NOT VERIFIED.

Actual `TODO`/`FIXME` inventory should be regenerated with `rg -n 'TODO|FIXME|XXX|HACK'` before relying on this section; no unverified issue is asserted here.

## 20. Coupling and modification rules

1. Read the relevant route, schema, service, model, migration, and frontend client together before changing a contract.
2. For persisted fields, update `models.py`, add an Alembic revision, and update schemas/tests; do not rely on `DEBUG` `create_all` for production.
3. For WebSocket behavior, inspect `websocket.py` and `ChatInterface.tsx` together, including auth, event names, cancellation, media cleanup, and reconnects.
4. For avatar/voice/media work, inspect storage keys/URLs, local `/uploads`, ownership checks, worker paths, and browser playback as one system.
5. Preserve existing provider fallbacks and explicit error classes unless intentionally changing user-visible behavior.
6. Keep secrets out of source, logs, docs, and client bundles; check `.env*` and compose build/runtime precedence.
7. Check both CPU and GPU compose/Dockerfile paths when changing dependencies or commands.
8. Prefer existing services and ownership helpers over duplicate implementations; maintain cascade and index assumptions in the models/migrations.
9. Run focused backend tests plus lint/typecheck/build for frontend changes; use health/log checks after compose changes.

## 21. Recommended reading order for a new AI agent

1. `README.md`, `SETUP_GUIDE.md`, `CONTRIBUTING.md`
2. `docker-compose.yml`, selected CPU/GPU/production compose file, and `.env.example`
3. `backend/main.py`, `backend/app/config.py`, `backend/app/models.py`, `backend/app/schemas.py`, `backend/app/database.py`
4. The relevant router in `backend/app/api/v1/`
5. Its service(s) under `backend/app/services/`
6. `backend/app/websocket.py` and `frontend/components/ChatInterface.tsx` for live conversations
7. `frontend/lib/api.ts`, `frontend/store/useStore.ts`, then the relevant UI component
8. `backend/alembic/versions/` and matching tests under `backend/tests/`
9. `backend/Dockerfile*`, `backend/entrypoint.sh`, `nginx/nginx.conf`, and deployment/Terraform files for operational work

