# Text-to-Speech — Remove Required Auth + Repo Setup — Design

**Date:** 2026-06-12
**Location:** `~/Code/text-to-speech`
**Status:** Approved for planning

## Goal

Make the existing self-hosted **text-to-speech** stack (Kokoro-FastAPI +
OpenReader) usable **without a login**, and bring the project up to the same
repository standard as its sibling `~/Code/speech-to-text`: a committed
`README.md`, a git-ignored `.env` with a committed `.env.example`, a
`.gitignore`, and a public GitHub repo with an opening PR.

## Starting state

The project is a single hand-written `docker-compose.yml` (Kokoro-TTS on `8880`,
OpenReader on `3003`/`8333`) plus a stray `.DS_Store`. It is **not** a git repo
and has no README, `.env`, or `.gitignore`. OpenReader is configured in **full
sign-in mode**: an `AUTH_SECRET` and a personal `ADMIN_EMAILS` are committed
inline, and the app forces a login at `/signin`.

## Key constraint: auth cannot be fully disabled

OpenReader v4+ **requires** `BASE_URL` and `AUTH_SECRET` at startup; there is no
flag to turn auth off entirely. The supported way to remove the *login
requirement* is anonymous sessions:

> `USE_ANONYMOUS_AUTH_SESSIONS=true` — users reach `/app` and use the reader
> without signing in or creating an account. (Docs:
> https://docs.openreader.richardr.dev/configure/auth)

This is the chosen approach. `AUTH_SECRET` and `BASE_URL` remain set (they must),
but they move into `.env` so no real secret is committed.

## Decisions (from brainstorming)

- **Auth mode:** anonymous sessions (`USE_ANONYMOUS_AUTH_SESSIONS=true`). No
  login/signup required.
- **Admin:** keep `ADMIN_EMAILS`. OpenReader admins own the **shared TTS
  provider** that anonymous users rely on, so an admin is retained — moved to
  `.env`, with a placeholder in `.env.example`.
- **Repo visibility:** public, matching the `speech-to-text` sibling. Safe
  because the only secret (`AUTH_SECRET`) and the only personal datum
  (`ADMIN_EMAILS`) live exclusively in the git-ignored `.env`.

## Design

### `docker-compose.yml` (edit in place)

In the `openreader` service `environment:` block:

- Add `USE_ANONYMOUS_AUTH_SESSIONS=true`.
- Replace the inline literals with `.env`-backed references, keeping working
  defaults where there is no secret:
  - `BASE_URL=${BASE_URL:-http://localhost:3003}`
  - `AUTH_SECRET=${AUTH_SECRET:?set AUTH_SECRET in .env}`
  - `ADMIN_EMAILS=${ADMIN_EMAILS}`
- Host port mappings become `${OPENREADER_PORT:-3003}` and
  `${KOKORO_PORT:-8880}` so ports are configurable but unchanged by default.

The Kokoro service holds no secrets and is otherwise unchanged. The hand-written
comment style (including the auth comment, updated to describe anonymous-session
mode) is preserved.

### `.env` (git-ignored)

Carries the real, currently-working values so the running stack keeps its
sessions/data valid:

```
OPENREADER_PORT=3003
KOKORO_PORT=8880
BASE_URL=http://localhost:3003
AUTH_SECRET=<the existing secret already in the running stack>
ADMIN_EMAILS=<your admin email>   # real value lives only in .env
```

### `.env.example` (committed)

Same keys, safe placeholders, and guidance:

- `AUTH_SECRET=` with a note to generate one via `openssl rand -hex 32`.
- `ADMIN_EMAILS=you@example.com` with a note that the first matching user is admin.
- Header instructing `cp .env.example .env`.

### `.gitignore` (committed)

Ignore `.env` and `.DS_Store`.

### `README.md` (committed)

Same tone/shape as the `speech-to-text` README: one-line intro, stack table
(Kokoro-FastAPI + OpenReader), quick start (`cp .env.example .env` →
`docker compose up -d` → open `http://localhost:3003`), an explicit
**"no sign-in required"** note explaining anonymous sessions (and that auth
cannot be fully disabled, only the login requirement), a configuration table,
data/volumes section (`openreader_docstore`), and brief troubleshooting.

### Git / PR

`git init` → commit baseline on `main` → branch `feat/no-auth-and-docs` → create
public GitHub repo `text-to-speech` → push → open PR.

## Verification

This is infrastructure/config, so "tests" are concrete checks:

1. `docker compose config` resolves with `.env` present (no unset-variable
   errors, `USE_ANONYMOUS_AUTH_SESSIONS=true` shows in the rendered output).
2. `git status` shows `.env` ignored and `.env.example`, `README.md`,
   `.gitignore`, `docker-compose.yml`, and `docs/` tracked.
3. A grep confirms no real `AUTH_SECRET` value and no personal email in any
   committed file.

## Out of scope

- Changing the TTS engine, models, or ports' defaults.
- A live re-bring-up of the stack (the user already runs it); compose config
  validation is the behavioral check here.
