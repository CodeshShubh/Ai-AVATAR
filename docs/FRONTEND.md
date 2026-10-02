# Frontend Architecture

See [AGENTS.md](../AGENTS.md) for the project-wide map.

## Framework and structure

The frontend is Next.js App Router with React/TypeScript. The root page is `frontend/app/page.tsx`; `layout.tsx` supplies the document shell. There is one verified application page route (`/`); `robots.ts` and `sitemap.ts` are metadata routes. Rendering uses client components for interactive panels; exact server/client boundaries are declared in each file.

| Component | File | Purpose | Backend dependency |
|---|---|---|---|
| AuthModal | `components/AuthModal.tsx` | registration/login/guest UI | users routes |
| AvatarList | `components/AvatarList.tsx` | list/select/delete avatars | avatars routes |
| AvatarUpload | `components/AvatarUpload.tsx` | image upload | avatar upload route |
| ChatInterface | `components/ChatInterface.tsx` | session, WS, text/audio/chat/media | sessions + WebSocket |
| HistoryPanel | `components/HistoryPanel.tsx` | history/session operations | sessions/conversations/messages |
| SettingsPanel | `components/SettingsPanel.tsx` | user/settings UI | users/profile routes |
| VoicePanel | `components/VoicePanel.tsx` | recording/cloning/preview | voices routes |
| WaveformVisualizer | `components/WaveformVisualizer.tsx` | audio visualization | browser audio state |
| QueryProvider | `components/providers/QueryProvider.tsx` | React Query provider | client state/query layer |

## State

`store/useStore.ts` persists auth token/user, hands-free preference, selected avatar/session, and selected session state under `avatar-system-storage`; WebSocket connection state is held in the store but excluded from persisted fields. Actions include `setAuth`, `clearAuth`, `setActiveSession`, `setSelectedAvatar`, and `setWsConnected`. Hydration is explicitly delayed to avoid SSR mismatch.

## REST client

`lib/api.ts` builds Axios from `NEXT_PUBLIC_API_URL`, defaults to `http://localhost:8000`, uses `withCredentials`, adds a bearer token from persisted Zustand state, and clears auth on HTTP 401. It exposes auth, avatar, session, message, voice, conversation, and preview methods. `api.getToken()` is used by WebSocket code.

## WebSocket and media

`ChatInterface.tsx` builds the session URL through `buildSessionWsUrl`, opens `/ws/session/{id}`, sends `set_voice`, text/audio, stop/interrupt, language, and handles incoming typed messages, reconnect timers, audio chunks, video/media queues, errors, and connection state. Exact incoming event union is defined in that file; inspect it with `backend/app/websocket.py`. The browser records audio as `audio/webm`, uses microphone/browser media APIs and VAD support from `lib/vad.ts`, and plays received media through component-managed queues. Unknown details: browser codec support varies and no frontend automated tests are present.

## Modification map

- Auth: `AuthModal.tsx`, `lib/api.ts`, `useStore.ts`, backend `users.py`.
- Avatars: `AvatarUpload.tsx`, `AvatarList.tsx`, `lib/api.ts`, backend `avatars.py`.
- Voice: `VoicePanel.tsx`, `lib/api.ts`, backend `voices.py`, `tts.py`.
- Chat/WS: `ChatInterface.tsx`, `lib/api.ts`, `useStore.ts`, backend `main.py`/`websocket.py`.
- Media: `ChatInterface.tsx`, `WaveformVisualizer.tsx`, `lib/vad.ts`, animator/TTS/STT services.

## Related References

- [ARCHITECTURE.md](ARCHITECTURE.md) · [BACKEND.md](BACKEND.md) · [API_MAP.md](API_MAP.md)
- [DATA_AND_DATABASE.md](DATA_AND_DATABASE.md) · [AI_MEDIA_PIPELINE.md](AI_MEDIA_PIPELINE.md) · [DEBUGGING.md](DEBUGGING.md)
