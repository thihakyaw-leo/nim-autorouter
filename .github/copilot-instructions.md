# Copilot Instructions — NVIDIA NIM AutoRouter

This repo is a Python/FastAPI gateway that classifies requests, selects one configured NVIDIA NIM model, and forwards OpenAI-compatible chat completions. It is currently a **policy router** (not yet an autonomous terminal agent — no shell execution, patch application, or tool loops).

> The repository currently contains only `README.md` (single initial commit); the application source has not been committed yet. Treat `README.md` as the source of truth for setup and behavior. Update this file when source is added.

## Build, test, and run commands

Requirements: Python 3.11+, NVIDIA API key for `https://integrate.api.nvidia.com/v1`, and six model IDs confirmed in `config/models.yaml`.

```bash
# Create and activate venv
python -m venv .venv
# Windows: .venv\Scripts\Activate.ps1
# Linux/macOS: source .venv/bin/activate

# Install the package (editable)
pip install -e .

# Configure environment
cp .env.example .env
# Set NVIDIA_API_KEY and a strong ROUTER_API_KEY in .env

# Run the server
uvicorn app.main:app --host 127.0.0.1 --port 8080 --reload
```

Tests (dev extras required):
```bash
pip install -e ".[dev]"
pytest -q

# Run a single test file
pytest tests/test_router.py -q

# Run a single test by node id
pytest tests/test_router.py::test_model_selection -q
```

Runtime verification (after starting the server):
```bash
curl -X POST http://127.0.0.1:8080/v1/router/verify -H "X-Router-Key: change-this-local-gateway-key"
```
Calls NVIDIA's authenticated `GET /v1/models` and checks exact IDs.

OpenAI-compatible request example:
```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Authorization: Bearer change-this-local-gateway-key" \
  -H "X-Router-Key: change-this-local-gateway-key" \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Review this Python function for bugs"}]}'
```

## High-level architecture

- **Entry point:** `app.main:app` (FastAPI application).
- **Router mode:** receives a chat completion request, classifies it, selects one configured NIM model, and forwards the request. It does not execute shell commands, apply patches, or run tool loops.
- **Configuration:** `config/models.yaml` holds six model IDs that must be confirmed against your authenticated NVIDIA catalog. NVIDIA's live catalog and the user's API-key entitlement are authoritative; public model cards do not guarantee inference support for a given key/account/region.
- **Endpoints:**
  - `POST /v1/chat/completions` — OpenAI-compatible chat completions. Clients point their OpenAI-compatible base URL at `http://127.0.0.1:8080/v1`; use the gateway key as the API key (and `X-Router-Key` header if the client requires it).
  - `POST /v1/router/verify` — catalog presence check against NVIDIA `GET /v1/models` (not a live inference probe).
  - `POST /v1/messages` — minimal non-streaming Anthropic Messages adapter (set `ANTHROPIC_BASE_URL=http://127.0.0.1:8080` and `ANTHROPIC_API_KEY` to the gateway key).
- **Known limitations (documented, do not re-implement without discussion):**
  - `stream=true` is not implemented as true SSE passthrough yet; use `stream: false`.
  - The router does not forward tool definitions from client requests yet.
  - No model fallback after inference failure, usage metering, persistence, adaptive policy, or dashboard yet.
  - Not a verified OpenAI Responses API / Codex provider yet.
- **Security:** Do not expose this gateway publicly without TLS, authentication, rate limiting, and network restrictions.

## Key conventions

- Environment: copy `.env.example` to `.env` and set `NVIDIA_API_KEY` and a strong `ROUTER_API_KEY`. No credentials are bundled.
- Model IDs were normalized from the supplied selection guide and NVIDIA public documentation; verify against your live catalog before relying on them.
- Base URL for clients: `http://127.0.0.1:8080/v1`.
- `stream=true` → not yet true SSE passthrough; prefer `stream: false`.
