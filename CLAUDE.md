# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Voicebox is a **local-first AI voice studio** — voice cloning + TTS (7 engines), STT (Whisper), global dictation, and an MCP server that lets agents speak in cloned voices. Everything runs on the user's machine. It is a Tauri (Rust) desktop app over a Python/FastAPI backend, with a shared React frontend.

The backend listens on **`http://127.0.0.1:17493`** in both dev and prod. Interactive API docs at `http://127.0.0.1:17493/docs`.

## Common commands

Everything is driven by [`just`](https://github.com/casey/just) (`brew install just`). Run `just --list` for the full set. The justfile is platform-aware (bash on unix, PowerShell on Windows) — prefer `just <target>` over raw commands so you get the right one.

```bash
just setup          # one-time: create Python venv, install Python + JS deps (GPU-aware)
just dev            # backend (if not already up) + Tauri desktop app
just dev-web        # backend + web app (no Rust/Tauri build — fastest frontend loop)
just dev-backend    # backend only (uvicorn --reload on :17493)
just dev-frontend   # Tauri app only (assumes a backend is already running)
just kill           # stop all dev processes (uvicorn + vite)

just check          # ALL checks: Biome (JS) + ruff lint + ruff format-check (Python)
just fix            # auto-fix + reformat everything (Biome + ruff)
just check-js       # biome check (lint + format + organize imports)
just check-python   # ruff check + ruff format --check on backend/
just test           # pytest backend/tests
just test-models    # E2E: generate with every TTS engine against the frozen binary (slow; --only <engine> to scope)

just build          # CPU server binary + Tauri installer
just generate-api   # regenerate the TS API client from the running backend's OpenAPI schema
just db-reset       # delete + reinit the SQLite db
```

Run a single Python test: `backend/venv/bin/python -m pytest backend/tests/test_profiles.py::TestProfileCRUD::test_create -v`

**`bun run <script>`** also works from the root for JS-only tasks (`bun run typecheck`, `bun run lint`, `bun run build:web`). The root `package.json` is a Bun workspace over `app`, `tauri`, `web`, `landing`.

### What CI enforces vs. what's local-only

CI (`.github/workflows/ci.yml`) runs **only** `bun run typecheck` (app + web) and `bun run build:web`. It does **not** run Biome lint or any Python checks. So before pushing, run `just check` and `just test` locally — green CI does not mean lint/format/Python are clean.

## Architecture

### Three-shell frontend (the most important structural fact)

There is **one** React app — it lives in `app/`. It is never built directly; `tauri/` and `web/` are thin shells that each import everything from `app/` via a Vite `@` alias pointing at `../app/src`, then inject a **platform implementation**.

- `app/src/platform/types.ts` defines the `Platform` interface (`filesystem`, `updater`, `audio`, `lifecycle`, `metadata`).
- `tauri/src/platform/` implements it with Tauri APIs (native FS dialogs, sidecar server lifecycle, system-audio capture, auto-updater).
- `web/src/platform/` implements it with browser APIs (downloads, no server lifecycle, etc.).
- Components consume it via `usePlatform()` (`app/src/platform/PlatformContext.tsx`) — **never** import `@tauri-apps/*` directly in `app/` shared code, or you break the web build. Anything OS-specific goes behind the `Platform` interface.

When adding a frontend feature, write it in `app/`. Only touch `tauri/` or `web/` to wire a new platform capability.

Frontend state: **Zustand** stores in `app/src/stores/` (`generationStore`, `playerStore`, `serverStore`, etc.); server data via **React Query**; routing via TanStack Router. UI primitives are Radix + a local `components/ui/` (shadcn-style). The API client in `app/src/lib/api/` is **generated** — don't hand-edit it; change the backend and run `just generate-api`.

### Backend layering (authoritative — note CONTRIBUTING.md is stale here)

The request flow is strictly layered. `CONTRIBUTING.md` still references a flat `backend/main.py` with all routes and a `backend/tts.py` — that is **out of date**. The real structure (and `backend/README.md`) is:

```
HTTP → routes/ (thin: validate, delegate, format) → services/ (business logic, CRUD, orchestration)
     → backends/ (TTS/STT/LLM inference) → utils/ (audio, effects, cache, chunking)
```

- **`backend/app.py`** — FastAPI app factory, CORS, lifespan, AMD-GPU env setup, mounts the MCP server at `/mcp`, mounts the built frontend for the web/prod path.
- **`backend/routes/__init__.py`** — `register_routers(app)` includes every domain router. **Adding an endpoint = add the handler to the right `routes/*.py` and ensure its router is registered here.**
- **`backend/services/generation.py`** — single `run_generation()` handles generate / retry / regenerate (model load, voice-prompt creation, chunked inference, normalize, effects, version persistence).
- **`backend/services/task_queue.py`** — serial async queue so **only one GPU inference runs at a time**. Never call a backend's `generate()` directly from a route; go through the queue. Background task refs are held to avoid GC.
- **`backend/backends/__init__.py`** — `TTSBackend` / `STTBackend` `Protocol`s, the declarative `ModelConfig` registry, and factory functions. Adding an engine = implement the protocol + register a `ModelConfig`. `backends/base.py` holds shared helpers (HF cache checks, device detection, voice-prompt combining).
- **`backend/database/`** — SQLAlchemy ORM models; `__init__.py` re-exports for back-compat; migrations run automatically on startup. ORM models are imported with a `DB` prefix alias (`VoiceProfile as DBVoiceProfile`); Pydantic schemas live in `backend/models.py` with `Create`/`Response`/`Request` suffixes.

### Inference backends — engine-agnostic by design

The server picks an inference backend at startup (`backend/utils/platform_detect.py`): **MLX** on Apple Silicon, **PyTorch** otherwise (CUDA / ROCm / Intel XPU / DirectML / CPU). Both satisfy the same `TTSBackend` protocol, so the route/service layer is engine-agnostic. The 7 TTS engines (`backend/backends/*_backend.py`) and Whisper STT all flow through this. One bundled Qwen3 LLM (dictation refinement + voice personalities) shares the same runtime and model cache.

To add a TTS engine, follow `.agents/skills/add-tts-engine/SKILL.md` + `docs/content/docs/developer/tts-engines.mdx` — it's a phased guide (dependency audit → protocol impl → frontend wiring → PyInstaller bundling) optimized for agents.

### MCP server

`backend/mcp_server/` is a FastMCP server mounted at `/mcp` (Streamable HTTP) inside the FastAPI app, plus a bundled stdio shim (`backend/mcp_shim/`). Four tools: `voicebox.speak`, `voicebox.transcribe`, `voicebox.list_captures`, `voicebox.list_profiles`. Per-client voice bindings keyed by the `X-Voicebox-Client-Id` header. The repo root ships `.mcp.json` pre-wired so Claude Code in this checkout gets the Voicebox tools once the dev backend is running.

### Packaging

In **dev**, the frontend talks to a manually-started uvicorn server. In **prod**, the Python backend is frozen by **PyInstaller** (`backend/build_binary.py`, `voicebox-server.spec`, `backend/pyi_*` runtime hooks) into a `voicebox-server` binary bundled as a Tauri sidecar. CUDA is shipped as a separate downloadable binary placed under the app-data `backends/` dir at runtime. Data directory defaults to the OS app-data dir; override with `--data-dir` or `VOICEBOX_DATA_DIR`. Models download from HuggingFace on first use; override the cache with `VOICEBOX_MODELS_DIR`.

## Conventions

- **Python**: target 3.12+, ruff-formatted, 120-col, double quotes, Google-style docstrings, built-in generics (`list[str]`, `X | None`) — no `typing.List`/`Optional`. Relative imports within the `backend` package. Lazy-import heavy deps (torch/transformers/mlx) inside functions with a `# lazy: heavy import` note. Logging via `logging` with `%s` placeholders, never `print()` or f-strings in log calls. **No ASCII section dividers** — if a file needs them, split it. Full rules in `backend/STYLE_GUIDE.md`.
- **TypeScript/React**: Biome-formatted (single quotes, 2-space, trailing commas, 100-col), `useExhaustiveDependencies` enforced, named exports, functional components. `noExplicitAny` is a warning.
- Error handling: services raise plain exceptions (`ValueError`, `FileNotFoundError`); route handlers translate to `HTTPException`. Never swallow exceptions or use bare `except:`.

## Releasing

Version is bumped with **bumpversion** (`.bumpversion.cfg`) — `bumpversion patch|minor|major` updates all `package.json`s, `Cargo.toml`, `tauri.conf.json`, and `backend/__init__.py`, commits, and tags. Pushing a tag triggers `.github/workflows/release.yml`. See `.agents/skills/release-bump/SKILL.md`. Update `CHANGELOG.md` for releases.

`docs/PROJECT_STATUS.md` is the living engineering roadmap (shipped vs. in-flight, prioritized issues, candidate engines, bottlenecks) — keep it current when shipping significant features or accepting/backlogging a model integration.
