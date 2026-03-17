# Alice AI — Project Handoff & Migration Guide

> **Purpose**: Context file for a new Copilot instance to quickly understand the project
> state and migrate Alice to a local LLM server (mini PC with GPU).

---

## What Is This Project?

Alice is a personal AI assistant that runs via **Signal** messaging, powered by
**OpenClaw** (an open-source AI gateway) and **Ollama** (local LLM runtime).
Everything runs in Docker containers.

**Owner**: Zhane Bell  
**Personal Signal number**: +18084515342  
**Bot Signal number**: +15012371791 (registered via Google Voice)  
**Timezone**: Pacific/Honolulu

---

## Current State (as of March 17, 2026)

### What's Working
- **Signal messaging**: Alice receives and replies to DMs on Signal
- **DM pairing**: Approved for Zhane's number (+18084515342)
- **Model fallback**: Primary model fails → falls back to free OpenRouter model
- **Docker containers**: Running with `restart: unless-stopped` (survive reboots)
- **GitHub repo**: https://github.com/zhanebell/Alice-AI-OpenClaw.git

### What's NOT Working
- **Ollama local model**: `ollama/qwen2.5:3b` is configured as primary but OpenClaw
  can't use it on the laptop. Discovery works at startup, but actual inference calls
  fail because the laptop is too slow / resource-constrained. Alice falls back to
  `openrouter/openrouter/free` (a free cloud model) for every reply.
- **OpenRouter paid model**: `openrouter/openai/gpt-oss-120b` (Groq Cloud) ran out of
  credits. Not currently usable until Zhane tops up the balance on OpenRouter.

### Running Containers (on Zhane's laptop)
| Container | Image | Purpose |
|-----------|-------|---------|
| `alice-gateway` | `ghcr.io/openclaw/openclaw:latest` | OpenClaw gateway (v2026.3.13) |
| `alice-signal-cli` | `bbernhard/signal-cli-rest-api:latest` | Signal bridge (native daemon mode) |
| `alice-ollama` | `ollama/ollama:latest` | Local Ollama model server |

---

## Architecture

```
Signal App (Zhane's phone)
    ↕ Signal Protocol
signal-cli container (:8080, native JSON-RPC + SSE daemon)
    ↕ HTTP (SSE for events, JSON-RPC for sending)
OpenClaw Gateway (:18789)
    ↕ Ollama API or OpenRouter API
Ollama container (:11434) / OpenRouter cloud
```

---

## Key Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | All 4 services (gateway, signal-cli, cli, ollama) |
| `config/openclaw.json` | Agent config, model settings, Signal channel config — **gitignored** |
| `config/openclaw.json.example` | Template for `openclaw.json` (copy and fill in your values) |
| `.env` | Secrets (API keys, gateway token) — **gitignored** |
| `.env.example` | Template for `.env` |
| `workspace/SOUL.md` | Alice's personality definition |
| `.gitignore` | Excludes secrets, credentials, device identity, memory DB |

### Files NOT in Git (local-only, regenerated per instance)
- `config/identity/` — device identity (unique per installation)
- `config/devices/` — paired device list
- `config/credentials/signal-pairing.json` — Signal pairing approvals
- `config/memory/main.sqlite` — conversation memory
- `signal-data` Docker volume — signal-cli registration data

---

## Critical Technical Details

### signal-cli Setup
The `bbernhard/signal-cli-rest-api` image is used BUT with a **custom entrypoint**
that bypasses the Go REST wrapper and runs the **native signal-cli daemon** directly:

```yaml
entrypoint: ["signal-cli", "--config", "/home/.local/share/signal-cli",
  "daemon", "--http", "0.0.0.0:8080",
  "--receive-mode", "on-connection", "--send-read-receipts"]
user: "1000:1000"
```

This is required because OpenClaw expects `/api/v1/events` (SSE) and `/api/v1/rpc`
(JSON-RPC) endpoints. The bbernhard Go wrapper serves different paths (`/v1/receive/`,
`/v1/send/`) that are incompatible.

### Ollama Provider Config (explicit, not auto-discovery)
OpenClaw's auto-discovery tries `http://127.0.0.1:11434` which doesn't work in Docker
(Ollama is on a separate container). The config uses **explicit provider definition**:

```json
"models": {
  "providers": {
    "ollama": {
      "baseUrl": "http://ollama:11434",
      "apiKey": "ollama-local",
      "api": "ollama",
      "models": [{ "id": "qwen2.5:3b", ... }]
    }
  }
}
```

When adding new models, add them to this `models` array in `config/openclaw.json`.

### Permissions Gotcha
The `config/credentials/` directory must be owned by `node:node` (UID 1000) inside the
gateway container. If pairing fails with EACCES, run:
```bash
docker exec -u root alice-gateway chown -R node:node /home/node/.openclaw/credentials
```

---

## Migration to LLM Server (Mini PC)

### Prerequisites
- Docker + Docker Compose installed
- NVIDIA GPU drivers + NVIDIA Container Toolkit (for GPU inference)
- Git

### Step-by-Step

#### 1. Clone and configure
```bash
git clone https://github.com/zhanebell/Alice-AI-OpenClaw.git
cd Alice-AI-OpenClaw
cp .env.example .env
```

Edit `.env`:
```env
OPENROUTER_API_KEY=sk-or-v1-...    # Same key or new one (optional for cloud fallback)
OLLAMA_API_KEY=ollama-local
OPENCLAW_GATEWAY_TOKEN=             # Generate new or reuse: openssl rand -hex 24
TZ=Pacific/Honolulu
```

#### 2. Enable GPU in docker-compose.yml
Uncomment the GPU section in the `ollama` service:
```yaml
ollama:
  image: ollama/ollama:latest
  ...
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: all
            capabilities: [gpu]
```

#### 3. Upgrade the model
Replace `qwen2.5:3b` with a larger model now that you have GPU:
```bash
docker compose up -d ollama
docker exec alice-ollama ollama pull llama3.3        # 70B, needs ~40GB VRAM
# or
docker exec alice-ollama ollama pull qwen2.5:14b     # 14B, ~9GB VRAM
# or
docker exec alice-ollama ollama pull qwen2.5:32b     # 32B, ~20GB VRAM
```

Then update `config/openclaw.json`:
- Change `agents.defaults.model.primary` to `"ollama/<model-name>"`
- Update the `models.providers.ollama.models` array with the new model ID,
  contextWindow, and maxTokens

Example for qwen2.5:32b:
```json
"models": {
  "providers": {
    "ollama": {
      "baseUrl": "http://ollama:11434",
      "apiKey": "ollama-local",
      "api": "ollama",
      "models": [
        {
          "id": "qwen2.5:32b",
          "name": "Qwen 2.5 32B",
          "reasoning": false,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 32768,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

And set the primary model:
```json
"agents": {
  "defaults": {
    "model": {
      "primary": "ollama/qwen2.5:32b",
      "fallbacks": ["openrouter/openrouter/free"]
    }
  }
}
```

#### 4. Signal registration (new instance needs its own)
The Signal identity is in a Docker volume (`signal-data`) that won't transfer.
You need to re-register the bot number OR migrate the volume.

**Option A — Re-register** (simplest):
```bash
docker compose up -d signal-cli
# Register (will need SMS verification code from Google Voice):
docker exec alice-signal-cli signal-cli -a +15012371791 register --voice
# Verify with the code:
docker exec alice-signal-cli signal-cli -a +15012371791 verify CODE
```

**Option B — Migrate the volume** from the laptop:
```bash
# On laptop: export the volume
docker run --rm -v openclaw-alice_signal-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/signal-data.tar.gz -C /data .

# Transfer signal-data.tar.gz to the server

# On server: import the volume
docker volume create alice-ai-openclaw_signal-data
docker run --rm -v alice-ai-openclaw_signal-data:/data -v $(pwd):/backup \
  alpine tar xzf /backup/signal-data.tar.gz -C /data
```

#### 5. Start everything
```bash
docker compose up -d
```

#### 6. Approve DM pairing (only if you re-registered Signal)
Send a message from your phone to the bot number, then:
```bash
docker exec alice-gateway cat /home/node/.openclaw/credentials/signal-pairing.json
# Get the code from the output, then:
docker exec alice-gateway node dist/index.js pairing approve signal <CODE>
```

If you get a permissions error on the credentials directory:
```bash
docker exec -u root alice-gateway chown -R node:node /home/node/.openclaw/credentials
```
Then send a message again and retry.

#### 7. Verify
```bash
# Check health
docker exec alice-gateway curl -s http://127.0.0.1:18789/readyz
# Should return: {"ready":true,"failing":[],...}

# Check logs for model being used
docker logs --tail 20 alice-gateway
# Should show: agent model: ollama/<your-model>
# And: [signal] delivered reply to ...
```

#### 8. Shut down laptop instance
Once the server is working, stop the laptop containers:
```bash
# On laptop:
cd C:\Users\zhane\OpenClaw-Alice
docker compose down
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| SSE 404 errors in gateway logs | signal-cli must run in native daemon mode (check entrypoint in docker-compose.yml) |
| "Unknown model: ollama/..." | Model must be defined in `models.providers.ollama.models` array in openclaw.json |
| "Failed to discover Ollama models" | Normal at startup (race condition). Only a problem if actual messages fail too |
| EACCES on credentials dir | `docker exec -u root alice-gateway chown -R node:node /home/node/.openclaw/credentials` |
| Signal messages not received | Check `docker logs alice-signal-cli` and `docker logs alice-gateway` for SSE errors |
| OpenRouter billing error | Top up credits at https://openrouter.ai or rely on `openrouter/openrouter/free` fallback |
