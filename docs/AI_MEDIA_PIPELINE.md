# AI and Media Pipeline

See [AGENTS.md](../AGENTS.md) for the project-wide map.

## Text and voice flows

```text
Text: ChatInterface → WS text → websocket.py → LLM streaming → TTS chunks
      → animator → WS media events → browser

Voice: microphone/WebM → ChatInterface/VAD → WS audio → STT transcript
       → same LLM/TTS/animator path → browser
```

The exact event payloads are defined in `backend/app/websocket.py` and `frontend/components/ChatInterface.tsx`; do not infer them from this diagram.

## STT

`services/stt.py` selects configured `whisper`, `google`, or `azure`; `WHISPER_MODEL` defaults to `large-v3-turbo`. WebSocket audio is decoded/transcribed there. CPU/GPU performance and provider installation are configuration-dependent; exact fallback behavior should be verified in the service implementation.

## LLM

`services/llm.py` supports Anthropic, OpenAI-compatible, and Ollama using `LLM_PROVIDER`, `LLM_MODEL`, temperature/token settings, API keys, and optional `OPENAI_BASE_URL`. WebSocket code streams output, keeps a bounded recent context, and optionally rolls evicted turns into a summary. Provider failures map to rate-limit/auth/unavailable/general errors for the client.

## TTS and voice cloning

`services/tts.py` uses Chatterbox with optional speaker WAV, then Edge TTS, then gTTS. Chatterbox output is saved as WAV using torchaudio; Edge/gTTS paths convert fallback audio to WAV. `voices.py` stores cloned voice profiles and previews in `voice_profiles`; a fallback does not preserve cloned voice behavior.

## Avatar animation

`services/animator.py` selects `musetalk`, `liveavatar`, or `simple` from `AVATAR_ENGINE`. MuseTalk uses the worker/model path configured by `MUSETALK_PATH`; GFPGAN is optional through `FACE_RESTORE`. LiveAvatar settings are under `LIVEAVATAR_*`; `simple` is the CPU/static-image fallback. Celery and WebSocket paths can invoke generation. Exact codecs, filenames, and event payloads are UNKNOWN / NOT VERIFIED here and must be read from `animator.py`/`websocket.py` when changing media.

## Media lifecycle and requirements

```text
image upload → AvatarProcessor → local/S3 storage → URL → browser
audio input → STT → LLM → TTS WAV → animator → generated media → WebSocket → browser
```

| Feature | Configuration | CPU | GPU | Fallback |
|---|---|---|---|---|
| STT | `STT_PROVIDER`, `WHISPER_MODEL` | provider/model dependent | Whisper can use CUDA | provider selection only; exact fallback UNKNOWN |
| LLM | `LLM_PROVIDER`/model | OpenAI-compatible/Ollama possible | provider dependent | configured provider only |
| TTS | `TTS_PROVIDER=chatterbox` | may load CPU, slower | supported by Chatterbox | Edge TTS then gTTS |
| Animation | `AVATAR_ENGINE` | `simple` | MuseTalk/LiveAvatar intended for GPU | simple engine |
| Restoration | `FACE_RESTORE=gfpgan` | UNKNOWN / NOT VERIFIED | optional GPU path | off |

## Related References

- [ARCHITECTURE.md](ARCHITECTURE.md) · [BACKEND.md](BACKEND.md) · [FRONTEND.md](FRONTEND.md)
- [API_MAP.md](API_MAP.md) · [DATA_AND_DATABASE.md](DATA_AND_DATABASE.md) · [DEBUGGING.md](DEBUGGING.md)
