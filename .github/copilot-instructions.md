# Copilot Instructions — NVIDIA NIM AutoRouter

This repo is a headless local Python + FastAPI gateway that routes coding-agent requests to one configured NVIDIA NIM model. Backend is Python/FastAPI; database is PostgreSQL; provider is NVIDIA NIM; primary protocol is OpenAI-compatible Chat Completions. Client targets: OpenCode, Claude Code, Codex CLI, GitHub Copilot (BYOK surfaces), and other compatible agents. It is **not** an autonomous terminal agent — no shell execution, patch application, or tool loops.

Version: 0.2.0 (architecture baseline + starter scaffold). Most adapter endpoints are scaffolded placeholders returning 501; only `/health` and `/v1/models` are implemented.

## Build, test, lint, and run commands

Requirements: Python 3.12+, PostgreSQL, NVIDIA API key for `https://integrate.api.nvidia.com/v1`.

```bash
# 1. Copy env template and set secrets
cp .env.example .env
# Set NVIDIA_API_KEY, ROUTER_API_KEY, DATABASE_URL in .env

# 2. Start PostgreSQL (loopback only)
docker compose up -d db

# 3. Install backend (editable, with dev extras)
pip install -e '.[dev]'

# 4. Run the service
uvicorn app.main:app --host 127.0.0.1 --port 8000

# 5. Verify
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/v1/models -H "Authorization: Bearer change-me-local-router-key"
```

Dev extras include `pytest`, `pytest-asyncio`, `ruff`, `mypy`.

```bash
# Full test suite
pytest

# Run a single test file
pytest tests/test_health.py

# Run a single test by node id
pytest tests/test_health.py::test_health

# Lint
ruff check .

# Format check
ruff format --check .

# Type check
mypy app
```

Docker:
```bash
docker build -t nim-autorouter:0.2.0 .
docker run -p 127.0.0.1:8000:8000 --env-file .env nim-autorouter:0.2.0
```

TypeScript SDK (consumer, not backend):
```bash
cd sdk/typescript
npm install
npm run build   # tsc -p tsconfig.json
```

## High-level architecture

- **Entry point:** `app.main:app` — FastAPI app with auth dependency (`check_auth`) on every non-health route.
- **Request path:** Client → protocol adapter → normalized internal request → policy/classifier → model capability filter → model selection → NVIDIA NIM provider → normalized result → protocol serializer.
- **Endpoints (all require `Authorization: Bearer <ROUTER_API_KEY>` except `/health`):**
  - `GET /health` — status ok (no auth).
  - `GET /v1/models` — returns empty model list; TODO: read enabled models from database-backed registry.
  - `POST /v1/chat/completions` — scaffolded, returns 501 (Chat Completions adapter not yet implemented).
  - `POST /v1/messages` — Anthropic Messages adapter placeholder, returns 501.
  - `POST /v1/responses` — OpenAI Responses API adapter placeholder (for Codex CLI), returns 501.
- **Configuration:** `app/config.py` uses `pydantic-settings` (`BaseSettings`) loading from `.env`. Key settings: `router_api_key`, `nvidia_api_key`, `nvidia_base_url`, `database_url`, `http_timeout_seconds`, `log_level`.
- **Database:** `app/db.py` — SQLAlchemy 2.0 engine + `SessionLocal` + `DeclarativeBase`, `get_db()` dependency. No Alembic migrations or schema tables yet.
- **Model registry:** `config/models.yaml` — six candidate models, all `enabled: false` with `verification: pending`. Keys: `laguna_xs_21`, `deepseek_v41_flash`, `minimax_m31_flash`, `nemotron_super_vl`, `nemotron_nano_omni`, `nemotron_bolt_e4b`.
- **TypeScript SDK:** `sdk/typescript/src/client.ts` — wraps the official `openai` npm package pointed at `http://127.0.0.1:8000/v1`; requires `NIM_ROUTER_API_KEY`. Not the backend runtime.
- **Protocol boundaries:** OpenCode → Chat Completions (first-class); Claude Code → Anthropic Messages adapter; Codex CLI → Responses API adapter (Chat Completions alone is insufficient); GitHub Copilot → BYOK/custom, surface-specific, requires targeted validation. Adapters must preserve tool calls, streaming events, usage, finish/stop reasons, and error semantics.

## Key conventions

- **Typed Pydantic schemas** for request/response bodies; no untyped dicts in route handlers.
- **Async I/O** for provider calls (`httpx`).
- **Inject DB sessions** via `get_db()` dependency; avoid business logic in route handlers.
- **Centralize errors**; secure every non-health route with `check_auth` (Bearer `ROUTER_API_KEY`).
- **Bind to loopback by default** (`127.0.0.1`); never expose publicly without TLS/auth/rate limiting/network restrictions.
- **Never log secrets or full private prompts.**
- **Versioned adapters** — one adapter per client wire protocol; do not pretend every client speaks the same protocol.
- **Testing:** Mock external NIM in unit tests; live NIM tests are opt-in only. Do not label live model tests passed unless they ran against the user's authenticated endpoint. Test error paths, auth, rate limits, and streaming.
- **Model safety:** A candidate model is not active until authenticated `/v1/models` discovery and per-model inference/capability tests pass. All six models are disabled pending verification — do not enable without runtime verification.
- **Skills:** 12 project-specific `SKILL.md` files under `skills/` (8 core + 4 optional) define per-area mission, workflow, constraints, and verification checklists. Read the relevant skill file before working in that area.
- **Docs:** `docs/ARCHITECTURE.md`, `docs/CLIENT_COMPATIBILITY.md`, `docs/ROADMAP.md`, `docs/STATUS.md` — read these for architecture decisions, client compat matrix, roadmap phases (P0–P5), and current implementation status.
