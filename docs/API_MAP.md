# API Map

See [AGENTS.md](../AGENTS.md) for the project-wide map.

All REST paths below are prefixed exactly as registered in `backend/main.py`. Authenticated routes use `require_current_user` unless noted. Request/response schema names are included only where verified from route decorators/source.

## Authentication and users — `backend/app/api/v1/users.py`

| Method/path | Purpose | Auth/request/response | Frontend |
|---|---|---|---|
| POST `/api/v1/users/register` | Create user | public; `UserCreate` → `UserResponse` | `api.register`, `AuthModal` |
| POST `/api/v1/users/login` | Login/JWT cookie | public; OAuth2 form → `Token` | `api.login` |
| POST `/api/v1/users/guest` | Disposable user/JWT | public → `Token` | `api.createGuest` |
| POST `/api/v1/users/logout` | Clear cookie | public/cookie → implementation response | `api.logout` |
| GET `/api/v1/users/me` | Current profile | current user → `UserResponse` | `api.getProfile` |
| PUT `/api/v1/users/me` | Update profile | current user; update payload → `UserResponse` | `api.updateProfile` |
| GET `/api/v1/users/` | List users | current user; → `List[UserResponse]` | UNKNOWN / NOT VERIFIED |
| GET `/api/v1/users/{user_id}` | User detail | current user; → `UserResponse` | UNKNOWN / NOT VERIFIED |

## Avatars — `avatars.py`

`POST /upload` (multipart image, `AvatarResponse`), `GET /` (`List[AvatarResponse]`), `GET /{avatar_id}` (`AvatarResponse`), `PUT /{avatar_id}/voice` (query `voice_id`, `AvatarResponse`), `PATCH /{avatar_id}/metadata` (metadata payload, `AvatarResponse`), `PATCH /{avatar_id}/name` (rename payload, `AvatarResponse`), and `DELETE /{avatar_id}` (204). These use `storage_service`, `AvatarProcessor`, PostgreSQL, and `AvatarList`/`AvatarUpload` callers.

## Sessions — `sessions.py`

`POST /create` (`SessionResponse`, avatar request), `GET /` (`List[SessionResponse]`), `GET /{id}` (`SessionResponse`), `POST /{id}/end` (`SessionResponse` and disconnect), `GET /{id}/export` (export response/blob; max 5000 messages), `DELETE /{id}` (204). Frontend callers are in `lib/api.ts`, `ChatInterface.tsx`, and `HistoryPanel.tsx`.

## Conversations — `conversations.py`

`GET /` (`List[ConversationResponse]`), `GET /{id}` (`ConversationResponse`), `GET /session/{session_id}` (`List[ConversationResponse]`), `POST /` (`ConversationResponse`), `PATCH /{id}/rename` (`ConversationResponse`), `POST /{id}/summarize` (`ConversationResponse`), and `DELETE /{id}` (204). They use owned sessions/DB and the LLM summary path where applicable.

## Messages — `messages.py`

`POST /send` (`MessageResponse`, REST fallback), `GET /session/{session_id}` (`List[MessageResponse]`), `GET /{message_id}` (`MessageResponse`), `PATCH /{message_id}` (`MessageResponse`), `DELETE /{message_id}` (204). Frontend methods are in `lib/api.ts`; real-time turns use the WebSocket instead.

## Voices — `voices.py`

`POST /clone` (multipart audio/name/language; response shape is implementation-defined), `GET /` list, `GET /{voice_id}` detail, `POST /{voice_id}/synthesize` WAV preview, `GET /{voice_id}/preview` WAV, and `DELETE /{voice_id}`. All require current user and use voice profile storage/TTS. Frontend caller: `VoicePanel.tsx` via `lib/api.ts`.

## Health/monitoring and WebSocket

`GET /` is API status; `GET /health` checks DB/Redis and reports service state; Prometheus exposure is enabled when configured. WebSocket endpoint: `WS /ws/session/{session_id}`. Verified incoming types are `audio`, `text`, `stop`, `interrupt`, `set_voice`, `set_language`, `ping`; server explicitly emits `error` and `pong`, with other typed events defined in `websocket.py`/`ChatInterface.tsx`. Do not change the protocol without inspecting both.

## Related References

- [ARCHITECTURE.md](ARCHITECTURE.md) · [BACKEND.md](BACKEND.md) · [FRONTEND.md](FRONTEND.md)
- [DATA_AND_DATABASE.md](DATA_AND_DATABASE.md) · [AI_MEDIA_PIPELINE.md](AI_MEDIA_PIPELINE.md) · [DEBUGGING.md](DEBUGGING.md)
