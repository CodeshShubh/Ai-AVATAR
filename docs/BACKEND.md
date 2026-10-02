# Backend Architecture

See [AGENTS.md](../AGENTS.md) for the project-wide map.

## Entry Point

`backend/main.py` creates the FastAPI app, registers lifespan initialization, middleware, Prometheus/OpenTelemetry hooks, REST routers, `/`, `/health`, and `/ws/session/{session_id}`. Lifespan handles debug `create_all` versus production Alembic expectations, storage/cache initialization, local `/uploads`, and WebSocket cleanup.

Middleware is GZip, `RateLimitMiddleware`, CORS with credentials, request logging, and security headers. The health endpoint checks PostgreSQL and Redis and reports GPU/LLM status.

## Configuration

`backend/app/config.py` groups application, database, Redis, local/S3 storage, LLM, avatar engine, STT/TTS, JWT/cookie, conversation memory, guest retention, rate limits, uploads, monitoring, tracing, and URLs. Secret values are supplied through `SECRET_KEY`, `JWT_SECRET_KEY`, provider keys, and AWS variables; values are intentionally not repeated here. Production validation rejects debug mode, insecure cookies, wildcard/HTTP CORS, and weak secrets.

## API Layer

| Router | Main routes | Main dependencies |
|---|---|---|
| `users.py` | register, login, guest, logout, `/me`, user list/detail | password hashing, `tokens.py`, `get_db`, current-user dependencies |
| `avatars.py` | upload, list/detail, voice assignment, metadata/name update, delete | `storage_service`, `AvatarProcessor`, DB ownership checks |
| `sessions.py` | create/list/detail/end/export/delete | DB ownership, `websocket_manager` on ending |
| `conversations.py` | list/detail/session list/create/rename/summarize/delete | DB ownership, LLM summary path |
| `messages.py` | REST send, session list, detail/edit/delete | DB ownership; REST fallback |
| `voices.py` | clone/list/detail/synthesize/preview/delete | local `voice_profiles`, TTS, avatar references |

Request/response models are in `backend/app/schemas.py`; do not infer fields from route names.

## Services

| Service | File | Responsibility | Called by |
|---|---|---|---|
| Storage | `services/storage.py` | local/S3 put/get/delete/URLs | avatar/voice/media routes and WS |
| Cache | `services/cache.py` | Redis lifecycle/cache operations | app health and service paths |
| Avatar processor | `services/avatar_processor.py` | image processing/metadata | avatar upload |
| Animator | `services/animator.py` | engine selection and media generation | WebSocket/Celery |
| STT | `services/stt.py` | configured speech recognition | WebSocket |
| TTS | `services/tts.py` | Chatterbox and fallbacks, WAV output | WebSocket/voice preview |
| LLM | `services/llm.py` | Anthropic/OpenAI-compatible/Ollama calls | WebSocket/conversation summary |

## WebSocket

`main.py` authenticates `/ws/session/{session_id}` from bearer query/header handling and the configured auth cookie, validates session ownership, enforces capacity, then delegates to `websocket_manager`. Confirm exact auth extraction in `main.py` before changing it.

Client/server input types verified in `main.py`/`ChatInterface.tsx`: `audio` with audio data, `text` with text, `stop`, `interrupt`, `set_voice` with `voice_id`, `set_language` with language, and `ping`. Confirmed server response type `error` and heartbeat `pong`; additional response types are produced by `websocket.py` and typed in the frontend, so inspect both before protocol changes. The manager loads session state, streams LLM output, chunks speech, runs TTS/animation, persists messages, supports barge-in cancellation, per-session send locks, stale cleanup, and disconnect cleanup. Reconnect behavior is implemented in `ChatInterface.tsx`; server does not queue disconnected turns.

## Workers

`app/celery_app.py` defines avatar processing, video generation, old-file cleanup, and guest cleanup tasks. Compose starts `celery -A app.celery_app worker --loglevel=info --concurrency=2`; Flower monitors it on 5555. `backend/musetalk_worker.py` is the persistent MuseTalk process entrypoint used by the animator path.

## Dependencies and modification map

FastAPI handles HTTP/WS; SQLAlchemy/asyncpg handles async PostgreSQL; Pydantic validates settings/schemas; Redis/Celery handle cache/background work; provider libraries implement AI/media; Axios/Next are not backend dependencies. For a route change inspect its router, `schemas.py`, models/migrations, `frontend/lib/api.ts`, caller component, and relevant tests. For WS/media changes inspect `main.py`, `websocket.py`, animator/STT/TTS/LLM services, and `ChatInterface.tsx` together.

## Related References

- [ARCHITECTURE.md](ARCHITECTURE.md) · [FRONTEND.md](FRONTEND.md) · [API_MAP.md](API_MAP.md)
- [DATA_AND_DATABASE.md](DATA_AND_DATABASE.md) · [AI_MEDIA_PIPELINE.md](AI_MEDIA_PIPELINE.md) · [DEBUGGING.md](DEBUGGING.md)
