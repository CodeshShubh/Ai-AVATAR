# Data Architecture

See [AGENTS.md](../AGENTS.md) for the project-wide map.

## Database and relationships

`database.py` creates an async SQLAlchemy engine/session factory and converts PostgreSQL URLs to asyncpg. In debug, `main.py` may call `Base.metadata.create_all`; production uses Alembic.

```text
User
 ├─< Avatar
 │    └─< Session
 └─< Session
       ├─< Message
       └─< Conversation
```

## Models

| Model/table | Important fields and relationships |
|---|---|
| `User/users` | id, email, username, hashed password, active/superuser/guest flags, timestamps; owns avatars and sessions. |
| `Avatar/avatars` | user FK, name, image/thumbnail URL, storage key, status, voice id, JSON metadata; belongs to user, owns sessions. |
| `Session/sessions` | user/avatar FKs, status/settings, start/end timestamps; owns messages/conversations. |
| `Message/messages` | session FK, role/content/type, audio/video URLs, metadata, tokens, latency, timestamp. |
| `Conversation/conversations` | session FK, title/summary/count and timestamps. |

Foreign keys use `ondelete=CASCADE`; ORM relationships use delete-orphan cascades. Indexes cover user/email/username/guest, avatar owner/created, voice references, session owner/start, and message session/created ordering.

## Migrations

`0000_initial_schema.py` creates the initial tables. `0001_add_voice_id_to_avatars.py` adds avatar voice association. `0002_add_perf_indexes.py` adds performance indexes. `0003_fk_on_delete_cascade.py` adds cascade behavior. `0004_add_is_guest_to_users.py` adds the guest flag. `alembic/env.py` and `alembic.ini` configure migration execution.

## Lifecycle and storage

Routes create/update/delete records with ownership checks. Avatar deletion cascades sessions and their messages/conversations; session/conversation/message deletion is exposed by their routes. Celery includes guest-account and old-file cleanup. Local avatar/generated files use configured local storage and `/uploads`; S3 mode stores keys and returns provider URLs. Voice profiles are file/index based under `voice_profiles`; exact generated-media retention depends on worker/service path.

## Redis and safety

Redis is configured for cache and Celery broker/result roles; exact cache keys are implementation-specific in `services/cache.py`. It is not the source of truth for relational records. Destructive commands such as `docker compose down -v` remove named database/cache/media volumes. Before data mutations inspect route ownership helpers, cascades, storage deletion, and matching tests.

## Related References

- [ARCHITECTURE.md](ARCHITECTURE.md) · [BACKEND.md](BACKEND.md) · [FRONTEND.md](FRONTEND.md)
- [API_MAP.md](API_MAP.md) · [AI_MEDIA_PIPELINE.md](AI_MEDIA_PIPELINE.md) · [DEBUGGING.md](DEBUGGING.md)
