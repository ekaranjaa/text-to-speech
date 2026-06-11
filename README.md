# text-to-speech

Self-hosted **text-to-speech**: paste or upload text and listen to it (or export
audio) from a browser reader UI. Runs entirely locally.

Powered by [**Kokoro-FastAPI**](https://github.com/remsky/Kokoro-FastAPI) (the
TTS engine, OpenAI-compatible API) and
[**OpenReader**](https://github.com/richardr1126/openreader) (the reader UI +
orchestrator). This is the text-to-speech sibling of the `speech-to-text`
(Whishper + faster-whisper) setup.

## Stack

Two containers (see `docker-compose.yml`):

| Service | Role | Host-exposed? |
|---|---|---|
| `openreader` | reader web UI + TTS orchestrator | yes, **http://localhost:3003** (+ `8333` SeaweedFS S3) |
| `kokoro-tts` | Kokoro-FastAPI TTS engine, OpenAI-compatible API | yes, **http://localhost:8880** |

CPU profile — no GPU required.

## No sign-in required

OpenReader runs in **anonymous-session** mode
(`USE_ANONYMOUS_AUTH_SESSIONS=true`), so you go straight to the reader without
creating an account or logging in. Note: OpenReader v4+ cannot turn auth off
entirely — `AUTH_SECRET` and `BASE_URL` are still required at startup — so this
removes the *login requirement*, not the auth subsystem. An admin
(`ADMIN_EMAILS`) is still configured because the admin owns the shared TTS
provider that anonymous sessions use. See
https://docs.openreader.richardr.dev/configure/auth.

## Quick start

```bash
cd ~/Code/text-to-speech
cp .env.example .env      # create your local config
# then edit .env: set AUTH_SECRET (openssl rand -hex 32) and ADMIN_EMAILS
docker compose up -d      # pulls images, starts the stack
```

Then open **http://localhost:3003** — no login needed.

```bash
docker compose logs -f openreader   # follow app logs
docker compose ps                   # service status
docker compose down                 # stop (data is kept in the volume)
```

## Configuration (`.env`)

| Variable | Meaning | Default |
|---|---|---|
| `OPENREADER_PORT` | host port for the reader UI | `3003` |
| `KOKORO_PORT` | host port for the Kokoro engine | `8880` |
| `BASE_URL` | public URL the browser uses — must match `OPENREADER_PORT` | `http://localhost:3003` |
| `AUTH_SECRET` | session-signing secret (mandatory). Generate: `openssl rand -hex 32` | — |
| `ADMIN_EMAILS` | admin user(s), comma-separated; owns the shared TTS provider | — |

To change the UI port, set `OPENREADER_PORT` **and** `BASE_URL` in `.env`, then
`docker compose up -d`.

## Data

OpenReader state (SeaweedFS blobs, SQLite metadata, migrations, temp state)
persists in the `openreader_docstore` Docker volume. Remove it
(`docker compose down -v`) for a clean slate.

## Notes & troubleshooting

- **Changed `AUTH_SECRET`?** Existing anonymous sessions are invalidated; the app
  issues fresh ones on next visit.
- **First boot pulls images** (Kokoro + OpenReader), so the first start is slow.
- **GPU:** not used — this stack is CPU-only by design.
