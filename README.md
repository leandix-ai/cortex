# Cortex Gateway

> OpenAI-compatible AI gateway with automatic complexity-based model routing, multi-provider support, client key management, and a built-in Web UI.

## Features

- **Auto-Routing** — Classifies each request as `simple` or `complex` using a 2-stage strategy: fast regex heuristics → LLM fallback (only when uncertain)
- **Multi-Provider** — Supports both **Anthropic** (`claude-*`) and **OpenAI**-compatible backends per routing tier
- **Context Compression** — Prompt caching via Anthropic's `cache_control` (ephemeral) on system prompts
- **OpenAI-Compatible** — Drop-in proxy for `/v1/chat/completions` and `/v1/models`
- **SSE Streaming** — Full streaming with `<think>` tag parsing → `reasoning_content` field
- **Multi-Language** — Automatic language detection and response localisation (Vietnamese, Chinese, Japanese, Korean, Arabic, Thai, Russian)
- **Tool Use** — Full support for OpenAI-format `tool_calls` / `tool` messages, converted transparently to Anthropic format
- **Client Key Management** — Issue, list, and revoke `cgw-*` API keys per client
- **SQLite Persistence** — Model config, client keys, and request logs stored in a local SQLite database
- **Web UI** — Built-in home page, login page, and admin dashboard served at `/`

## Requirements

- Python 3.10+
- Anthropic or OpenAI API key (depending on provider choice)

## Installation

### From PyPI

```bash
pip install cortex-gateway
```

### From source

```bash
git clone https://github.com/leandix/cortex-gateway
cd cortex-gateway
pip install -e .
```

## Configuration

Copy `.env.example` to `.env` and fill in your API keys:

```bash
cp .env.example .env
```

```ini
# Simple task model (fast, cheap — e.g. Haiku)
SIMPLE_MODEL_NAME=claude-haiku-4-5
SIMPLE_MODEL_API_KEY=sk-ant-...
SIMPLE_MODEL_BASE_URL=https://api.anthropic.com

# Complex task model (powerful — e.g. Sonnet)
COMPLEX_MODEL_NAME=claude-sonnet-4-5
COMPLEX_MODEL_API_KEY=sk-ant-...
COMPLEX_MODEL_BASE_URL=https://api.anthropic.com

# Gateway
GATEWAY_HOST=0.0.0.0
GATEWAY_PORT=8000

# Optional: custom SQLite path (default: cortex_gateway.db in current dir)
# CORTEX_DB_PATH=/var/data/cortex.db

# Optional: custom JWT secret (auto-generated per instance if not set)
# JWT_SECRET=your-secret-here
```

> **Note**: Model configuration is seeded from `.env` on first startup into SQLite and can be updated live via the Dashboard.

## Running

```bash
# Start with defaults (reads GATEWAY_HOST / GATEWAY_PORT from .env)
cortex-gateway

# Custom host and port
cortex-gateway --host 127.0.0.1 --port 9000

# Dev mode with hot reload
cortex-gateway --reload

# Multiple worker processes
cortex-gateway --workers 4

# Show version
cortex-gateway --version

# Run as Python module
python -m cortex_gateway
```

After startup, the following URLs are available:

| URL | Description |
|-----|-------------|
| `http://localhost:8000/` | Public home page |
| `http://localhost:8000/login` | Admin login |
| `http://localhost:8000/dashboard` | Admin dashboard |

## Client Configuration

To use Cortex Gateway as a drop-in proxy, point your AI client at `http://localhost:8000/v1` and provide a **client API key** (created from the Dashboard).

### Continue (config.yaml)

```yaml
models:
  - name: Cortex
    provider: openai
    model: cortex-auto           # auto-routes between simple / complex model
    apiBase: http://localhost:8000/v1
    apiKey: cgw-...              # your client API key from the Dashboard
```

### Override routing per request

Force a specific model by name:

```yaml
models:
  - name: Cortex Sonnet
    provider: openai
    model: claude-sonnet-4-5    # bypasses auto-routing, goes directly to complex tier
    apiBase: http://localhost:8000/v1
    apiKey: cgw-...
```

Or use custom headers:

```
X-Cortex-Model-Simple: claude-haiku-4-5
X-Cortex-Model-Complex: claude-sonnet-4-5
```

## API Endpoints

### Core (requires client API key)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Chat completions (streaming & non-streaming) |
| `/v1/models` | GET | List available models |

### Admin (requires JWT — obtained from `POST /api/login`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/login` | POST | Login with admin password → returns JWT |
| `/api/change-password` | POST | Change admin password |
| `/api/status` | GET | Public gateway status (uptime, model names, total requests) |
| `/api/keys` | GET | List all client API keys |
| `/api/keys` | POST | Create a new client API key |
| `/api/keys/{key}` | DELETE | Revoke a client API key |
| `/api/models` | GET | Get model configuration (masked API keys) |
| `/api/models` | PUT | Update model configuration for simple/complex tier |
| `/api/stats` | GET | Request stats grouped by client key and model |
| `/health` | GET | Health check |

> Default admin credentials: `admin123` — **change this immediately after first login**.

## Routing Engine

Routing uses a **2-stage pipeline**:

1. **Heuristic Scoring** — Fast, zero-cost regex pattern matching. Each pattern contributes a positive (complex) or negative (simple) score. If the score exceeds a threshold (≥ 2 → COMPLEX, ≤ −2 → SIMPLE), the result is returned immediately.
2. **LLM Classification** — Called only when the heuristic score is in the "grey zone". Uses the simple model with a minimal prompt (`max_tokens=5`) to output `SIMPLE` or `COMPLEX`.

Special overrides force `SIMPLE` for meta-requests like "give this chat a title".

## Database

Cortex Gateway uses a local SQLite database (`cortex_gateway.db` by default) with the following tables:

| Table | Purpose |
|-------|---------|
| `admin` | Hashed admin password (SHA-256 + salt) |
| `model_config` | Active model settings per tier (`simple` / `complex`) |
| `client_keys` | Client API keys with name, status, and creation time |
| `request_logs` | Per-request log (client key, model, complexity, token counts) |

The database path can be overridden with `CORTEX_DB_PATH`.

## Project Structure

```
cortex_gateway/
├── app.py        # FastAPI routes, JWT auth, streaming, message format conversion
├── cli.py        # CLI entry point (argparse + uvicorn)
├── config.py     # Env vars, system prompts, routing patterns, language data
├── db.py         # SQLite schema, admin/key/stats operations
├── router.py     # Complexity classifier, language detector
└── static/       # Built-in Web UI (index.html, login.html, dashboard.html)
```

## License

MIT
