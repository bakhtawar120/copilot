# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Meet** — a full-stack AI chat assistant. Three independently runnable services, each with its own toolchain:

- `api/` — FastAPI, Python 3.13, uv, Ruff, pytest. Owns all model access and secrets.
- `web/` — Next.js 16, React 19, TypeScript 7, Tailwind 4, shadcn/ui.
- `agent/` — LiveKit Agents voice worker (Python 3.13, uv). Optional; only for voice mode.

`AGENTS.md` holds the house rules; they apply here too. The ones most easily broken: **local AI (Ollama) first, cloud only as fallback**; keep every AI provider behind the common `Provider` interface; never expose API keys to `web/`; don't add dependencies without need; add/update tests for changed behaviour and run Ruff on Python changes.

## Commands

```bash
# api  (http://localhost:8000/docs)
cd api && uv sync && uv run fastapi dev app/main.py
cd api && uv run ruff check . && uv run ruff format --check . && uv run pytest
cd api && uv run pytest tests/test_chat.py -k fallback    # one file / matching tests

# web  (http://localhost:3000)
cd web && npm install && npm run dev
cd web && npm run typecheck && npm run build              # no test suite on the web side

# agent (API must be running)
cd agent && uv sync && uv run python voice_agent.py dev
cd agent && uv run ruff check . && uv run ruff format --check . && uv run pytest

# everything
docker compose up --build                     # api + web
docker compose --profile voice up --build     # + voice agent
docker compose --profile ollama up            # Ollama in Docker (set OLLAMA_HOST=http://ollama:11434)
```

Env files: `api/.env`, `agent/.env`, `web/.env.local` (copy from each `.env.example`). Local model: `ollama run gemma3:1b`; for photos pull a vision model (`gemma3`, `llava`, `qwen2.5vl`).

## Architecture

**Browser → web → api.** The browser only ever calls `/api/*` on the Next.js app; `web/next.config.ts` rewrites those to `API_URL`. No CORS in dev, and keys never reach the client. `compress: false` is deliberate — compression would buffer the SSE chat stream.

**Chat pipeline (api).** `app/main.py:create_app()` is an app factory taking optional `settings`, `registry`, `voice_settings`, `library_settings`, `library_engine` — tests inject fakes through it. `POST /api/chat` (`routes/chat.py`) streams server-sent events: `meta` (which provider/model actually answered, `fallback`, `notice`), then `{"delta": …}` chunks, then `done` or `error`. `services/chat.py:ChatService` drives fallback using `providers/registry.py:ProviderRegistry.candidates()`: the chosen model first, then the first installed Ollama model, then cloud providers in `CLOUD_PRIORITY` order. Fallback only happens **before the first token** — a mid-answer failure is an error, so replies never mix models. `ALLOW_CLOUD_FALLBACK=false` disables it.

**Providers.** `providers/base.py:Provider` (`configured`, `list_models`, `stream_chat`). `ollama.py` (local), `anthropic.py`, and `openai_compat.py` (OpenAI, Gemini, Grok, Meta share one OpenAI-compatible implementation). Model lists are fetched live and cached in the registry. Vision capability: Ollama reports it; cloud models are matched by name in `app/vision.py` (+ `VISION_MODELS`). When any message has images, only vision models are candidates. Images go to Ollama as raw **bytes** (a string that looks like a path is read from disk by the SDK).

**Library / files (RAG).** `app/library/` is a self-contained package mounted by `mount_library()`, adding `/api/files/*` and `/api/library/*` and putting a `LibraryService` on `app.state.library`. The search engine sits behind the `RagEngine` protocol (`engine.py`); the real one is Gemini File Search (`gemini.py`), and `RAG_ENGINE=offline` swaps in a local keyword stand-in (`offline.py`) for dev/tests. When a chat request carries `rag` (collections or chat files), `routes/chat.py` routes it through `app/library_chat.py` to the Library engine instead of the model picker, emitting a final `sources` event with `[n]` citations. Library failures must never take down chat/voice — `mount_library` turns the Library off with a reason instead of raising. Data (originals + SQLite) lives in `api/data/library/` (gitignored). Owner pages at `/library` require `LIBRARY_TOKEN` (Bearer) in production.

**Voice mode.** `app/voice/` is a drop-in package (relative imports only) mounted by `mount_voice(app, brain=…)`; it adds `/api/voice/config`, `/api/voice/session`, `/api/voice/chat`. The *brain* is `app/voice_brain.py:chat_brain`, which reuses `ChatService` so voice gets the same model choice and fallback as text. Flow: web calls `/api/voice/session` → API creates a LiveKit room + token that dispatches the agent and remembers model/history → `agent/voice_agent.py` joins, does STT/turn detection/TTS on LiveKit Inference, and `agent/app_llm.py:AppLLM` posts each turn to `/api/voice/chat`. The agent must never go silent on failure (speaks an apology, reports the reason via participant attributes). `VOICE_AGENT_TOKEN` must match in `api/.env` and `agent/.env`.

**Request-size limits.** `core/body_limit.py:BodySizeLimit` middleware is added per path (`/api/chat`, `/api/files`, `/api/library/documents`) with budgets derived from settings; keep it in sync when adding upload-bearing routes.

**web.** `components/chat/chat-app.tsx` is the shell; feature folders `components/{chat,photos,files,library,voice}` pair with hooks in `hooks/` (`use-chat.ts` owns conversations, streaming and persistence). `lib/api.ts` has the SSE parser; `lib/types.ts` mirrors the API's Pydantic schemas in `api/app/schemas.py` — change both together. Chat history is in localStorage and photos in IndexedDB (`lib/image-store.ts`); there is no server-side database for chats. App name and placeholder user live in `lib/config.ts`.

## Testing notes

- API tests need no network: `api/tests/conftest.py` provides `FakeProvider`, a `make_client(*providers, **settings_overrides)` factory, and `parse_sse()`. The Library is off by default in tests (`library_settings` fixture uses a tmp dir); Gemini paths are tested against `tests/fake_gemini_api.py`.
- pytest runs with `asyncio_mode = "auto"`. Ruff: line length 100, rule sets `E,F,W,I,B,UP,N,SIM,ASYNC,RUF,PL` (same config in `api/` and `agent/`).

## Project skills

`.claude/skills/` contains the skills these features were built from (`fullstack-ai-assistant`, `add-photos`, `add-files-library`, `livekit-voice-mode`, `uv-*`). Use them when extending or debugging the corresponding feature.

## Commits

Gitmoji-prefixed, short: `:sparkles: Add voice mode`, `:hammer: Upda