# System Architecture

See [AGENTS.md](../AGENTS.md) for the project-wide map.

## High-Level Architecture

```text
Browser / Next.js
  ├─ HTTP REST + credentials ─► FastAPI routers
  │                              ├─ SQLAlchemy async ► PostgreSQL
  │                              ├─ Cache ► Redis
  │                              └─ Local/S3 storage
  └─ WS /ws/session/{id} ─────► ConnectionManager
                                 STT → LLM → TTS → Animator → media events
```

Compose runs `postgres`, `redis`, `backend`, `frontend`, `celery-worker`, `flower`, and optional `nginx`. External AI providers are selected through `backend/app/config.py`; they are called by backend services, never directly by the browser.

## Service Relationships

The frontend calls REST through `frontend/lib/api.ts` and the live pipeline through `frontend/components/ChatInterface.tsx`. FastAPI routes use `get_db`, ownership dependencies, and service modules. Celery uses Redis broker/result databases and shares backend code/volumes. Nginx is a reverse proxy for browser-facing HTTP/WebSocket traffic. Local media is exposed at `/uploads`; S3 URLs are produced by `storage.py` when configured.

## End-to-End Flows

- Registration/login/logout: `AuthModal.tsx` → `lib/api.ts` → `users.py` → `tokens.py`/`models.py`; login and guest creation set an auth cookie.
- Avatar upload/selection: `AvatarUpload.tsx` → `uploadAvatar` → `avatars.py` → `avatar_processor.py`/`storage.py` → `AvatarList.tsx` and Zustand selection.
- Session/chat: `ChatInterface.tsx` → `sessions.py` REST creation → `/ws/session/{id}` → `websocket.py` → persistence and media events.
- Voice clone: `VoicePanel.tsx` → `/api/v1/voices/clone` → `voices.py` and `voice_profiles`; avatar assignment uses `avatars.py`.
- History: `HistoryPanel.tsx` → sessions/conversations/messages routes; records come from PostgreSQL.

## Runtime and CPU/GPU

Compose waits for Postgres/Redis health before backend and worker startup. Backend lifespan optionally creates tables in debug mode, initializes storage/cache, mounts local uploads, and starts WebSocket cleanup. Production expects Alembic from `entrypoint.sh`.

`docker-compose.cpu.yml` selects `Dockerfile.cpu` and has no NVIDIA reservation. `docker-compose.gpu.yml` uses GPU-capable backend/worker configuration. The default compose file also contains GPU reservations; select deliberately. Exact model availability depends on the selected image and setup scripts.

## Boundaries

Routes own HTTP validation/auth and coordinate persistence; services own provider/storage/media integrations; models own persistence shape; frontend API code owns HTTP serialization and auth headers; WebSocket client/server must evolve together. Do not bypass ownership checks or call provider SDKs from UI components.

## Related References

- [BACKEND.md](BACKEND.md)
- [FRONTEND.md](FRONTEND.md)
- [API_MAP.md](API_MAP.md)
- [DATA_AND_DATABASE.md](DATA_AND_DATABASE.md)
- [AI_MEDIA_PIPELINE.md](AI_MEDIA_PIPELINE.md)
- [DEBUGGING.md](DEBUGGING.md)
