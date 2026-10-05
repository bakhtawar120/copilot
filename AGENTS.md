# Panda — AGENTS.md

Full-stack AI chat app. Read this before coding. See `README.md` for features
and setup, and each folder's README for detail.

## Stack

* web: Next.js 16, React 19, TypeScript 7, Tailwind 4, shadcn/ui
* api: FastAPI, Python 3.13, uv, Ruff, pytest
* agent: LiveKit Agents voice worker, Python 3.13, uv (optional, voice mode only)
* Local AI: Ollama (`localhost:11434`) — `gemma3:1b`; `gemma3` for photos
* Cloud fallback: Anthropic, OpenAI, Gemini, Grok, Meta
* Files and Library (RAG): Gemini File Search; `RAG_ENGINE=offline` without a key

## Commands

```bash
# web  → :3000
cd web && npm install && npm run dev
cd web && npm run typecheck && npm run build

# api  → :8000/docs
cd api && uv sync && uv run fastapi dev app/main.py
cd api && uv run ruff check . && uv run ruff format --check . && uv run pytest

# agent (API must be running)
cd agent && uv sync && uv run python voice_agent.py dev
cd agent && uv run ruff check . && uv run ruff format --check . && uv run pytest
```

Always run Python tools through `uv run`. The web has no test suite;
`typecheck` and `build` are its checks.

## Where things live

* `api/app/providers/` — one class per AI backend behind `base.py:Provider`;
  `registry.py` lists models and picks fallback candidates.
* `api/app/services/chat.py` — streaming and fallback for `POST /api/chat` (SSE).
* `api/app/library/` — files and Library (RAG), behind the `RagEngine` protocol.
* `api/app/voice/` — voice routes; `voice_brain.py` reuses the chat service.
* `web/lib/types.ts` mirrors `api/app/schemas.py`. Change both together.
* `web/hooks/` — client state (chat, models, attachments, voice, library).

## Rules

* Follow existing patterns; keep changes small.
* TypeScript web; typed Python api and agent.
* Run the checks above after every change; add or update tests for changed behaviour.
* API tests must not touch the network: use `FakeProvider` and `make_client`
  from `api/tests/conftest.py`.
* Run Ruff on api and agent changes.
* Never commit `.env`, API keys, secrets, or `api/data/`.
* Never expose AI/API keys to web; the browser only calls `/api/*` through
  the Next.js rewrite.
* Local AI first; cloud AI only as fallback.
* Keep AI providers behind the common `Provider` interface.
* Fall back only before the first token; never mix two models in one reply.
* The Library and voice must never stop chat from working: turn the feature
  off with a reason instead of raising at startup.
* Don't add dependencies without need.
* Don't modify unrelated files.
* Commits: gitmoji plus a short message, e.g. `:sparkles: Add voice mode`.
