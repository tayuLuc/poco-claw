# Poco-Claw on Mac (Apple Silicon) via Podman

## Prerequisites

- **Podman** (`brew install podman`)
- **OpenCode Go subscription** (key from [opencode.ai/auth](https://opencode.ai/auth))

```bash
# Create Podman VM (arm64)
podman machine init --cpus 4 --memory 4096

# Start
podman machine start
```

## Quick Start

```bash
cd /path/to/poco-claw/deploy/podman

# Configure
cp .env.example .env
# edit .env — fill in OC_GO_CC_API_KEY (required)

# Build proxy and start all services
podman compose build oc-go-cc
podman compose up -d

# Initialize S3 bucket
podman compose --profile init up -d rustfs-init

# With Mem0 (smart memory)
podman compose --profile mem0 up -d
```

## Architecture

```
┌──────────────┐    Anthropic API     ┌──────────────┐    OpenAI API     ┌─────────────┐
│  poco-claw   │ ──── /v1/messages ──▶│  oc-go-cc    │ ── /chat/completions ──▶│ OpenCode Go │
│  (backend)   │◀─────────────────────│  :3456       │◀────────────────────│             │
│  (executor)  │                      │  proxy       │                     │ api.opencode│
└──────────────┘                      └──────────────┘                     └─────────────┘
```

```
┌───── User ──────────┐
│  localhost:8080       │
└────────┬──────────────┘
         │
    ┌────▼─────┐
    │  Caddy   │ ← single external port
    │  :8080   │
    └──┬───┬───┘
       │   │
  ┌────▼─┐ └──────────┐
  │front │           │
  │:3000 │     ┌─────▼────┐
  └──────┘     │  backend │
               │  :8000   │
               └──┬───┬───┘
                  │   │
            ┌─────▼┐ ┌▼────────┐
            │oc-go-cc│ │postgres│
            │ :3456  │ │ :5432  │
            └────────┘ └────────┘
```

`oc-go-cc` translates Anthropic-format requests to OpenAI-format for OpenCode Go.

## Ports

| Service | Internal Port | Host Port |
|---|---|---|
| **Caddy (reverse proxy)** | 80 | **8080** ← single entry |
| Frontend (Next.js) | 3000 | — (via Caddy) |
| Backend (FastAPI) | 8000 | — (via Caddy) |
| Executor Manager | 8001 | **8001** (callback from executors) |
| oc-go-cc (proxy) | 3456 | — (internal) |
| PostgreSQL | 5432 | — (internal) |
| RustFS (S3 API) | 9000 | — (internal) |
| RustFS (Console) | 9001 | — (internal) |

Only Caddy (`:8080`) and Executor Manager (`:8001`) are exposed to host. Everything else stays strictly inside podman compose network.

## Models

`oc-go-cc` routing (automatic):

| Scenario | Model |
|---|---|
| **default** | DeepSeek V4 Flash |
| **long_context** (>80K tokens) | MiniMax M2.5 (1M context) |
| **think/plan/reason** | GLM-5 |
| **complex** (arch/refactor) | GLM-5.1 |
| **background** (read/grep/ls) | Qwen 3.5 Plus |
| **fast** | Qwen 3.6 Plus |

DeepSeek V4 Pro available as `deepseek-v4-pro` in `config.json`.

## Podman on Mac specifics

- **rootless** — socket at `~/.local/share/containers/podman/`
- `host.containers.internal` instead of `host.docker.internal`
- Native arm64 — no x86 emulation

## Management

```bash
# Status
podman compose ps

# Logs
podman compose logs -f backend
podman compose logs -f oc-go-cc

# Stop
podman compose down

# Rebuild proxy (if oc-go-cc updated)
podman compose build oc-go-cc
podman compose up -d oc-go-cc
```

## Structure

```
poco-claw/
├── compose.yaml              # Podman Compose
├── config.json               # oc-go-cc config (models, routing)
├── .env.example              # Environment template
├── README.md                 # This file
└── docker/
    └── oc-go-cc/
        └── Dockerfile        # Proxy build from pre-built binary
```
