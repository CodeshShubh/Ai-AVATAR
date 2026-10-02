# Debugging Guide

See [AGENTS.md](../AGENTS.md) for the project-wide map.

## Health and Docker

```bash
docker compose ps
docker compose logs -f backend
docker compose logs -f celery-worker
curl -i http://localhost:8000/health
curl -i http://localhost:3000
docker compose restart backend
docker compose build backend frontend
```

`/health` reports PostgreSQL/Redis, GPU, LLM, and active WebSocket state; HTTP 503 means a hard dependency is unhealthy. `docker compose down` is cautionary; `docker compose down -v` is destructive because it removes named data volumes. Inspect the selected compose file before rebuild/restart.

## Backend and authentication

Start at `backend/main.py`, then inspect `config.py`, `users.py`, `tokens.py`, middleware, and logs. For 401s check the Zustand-persisted token, Axios `withCredentials`/Authorization interceptor, cookie flags, CORS origins, `get_current_user`/`require_current_user`, and WebSocket handshake logic in `main.py`. Docs are available only when `DEBUG` is enabled.

## Avatar and voice traces

Avatar: `AvatarUpload.tsx` → `lib/api.ts` → `avatars.py` → `avatar_processor.py`/`storage.py` → DB `Avatar` URL/key → `AvatarList.tsx`/`ChatInterface.tsx` image load. Check extension/size validation, storage path, URL mode, and `/uploads` mounting.

Voice: `VoicePanel.tsx` recording (`audio/webm`) → `lib/api.ts` or ChatInterface WS → `voices.py`/`stt.py` → `llm.py` → `tts.py`; check `voice_profiles/index.json`, language/duration limits, provider initialization, and fallback logs.

## Animation/WebSocket traces

For no media, inspect `websocket.py` chunk/task errors, TTS output, `animator.py` engine selection, model/setup paths, and `ChatInterface.tsx` media queue/playback. In browser DevTools inspect Network → WS for `/ws/session/{id}`, handshake/close code, outgoing types (`audio`, `text`, `stop`, `interrupt`, `set_voice`, `set_language`, `ping`), and incoming typed messages. Server auth rejection uses close code 4401; capacity uses 1013 in `main.py`.

## Error-to-file map

| Symptom | First file | Second file | Layer |
|---|---|---|---|
| API 401 | `frontend/lib/api.ts` | `backend/app/api/v1/users.py` | auth/cookie/token |
| Avatar missing | `AvatarUpload.tsx` | `avatars.py`/`storage.py` | upload/storage |
| DB connection failure | `main.py` health | `database.py`/compose | infrastructure |
| Redis/Celery failure | `services/cache.py` | `celery_app.py`/compose | cache/worker |
| WS closes | `ChatInterface.tsx` | `main.py`/`websocket.py` | realtime/auth |
| No transcript | `ChatInterface.tsx` | `services/stt.py` | microphone/STT |
| No speech | `websocket.py` | `services/tts.py` | TTS/provider |
| No avatar video | `websocket.py` | `services/animator.py`/worker | animation/media |

Coverage percentages, browser codec compatibility, and provider-specific root causes are UNKNOWN / NOT VERIFIED unless demonstrated by the relevant test/log.

## Related References

- [ARCHITECTURE.md](ARCHITECTURE.md) · [BACKEND.md](BACKEND.md) · [FRONTEND.md](FRONTEND.md)
- [API_MAP.md](API_MAP.md) · [DATA_AND_DATABASE.md](DATA_AND_DATABASE.md) · [AI_MEDIA_PIPELINE.md](AI_MEDIA_PIPELINE.md)
