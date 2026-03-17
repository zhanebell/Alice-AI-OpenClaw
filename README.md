# Alice — Personal AI Assistant (OpenClaw + Signal + Ollama)

A personal AI assistant that runs in Docker and responds via **Signal**.
Uses a local model (Ollama) by default, with OpenRouter as a cloud fallback.

Powered by [OpenClaw](https://github.com/openclaw/openclaw).

## Architecture

```
Signal (your phone)
       │
       ▼
┌──────────────┐     ┌──────────────┐
│  signal-cli  │◄───►│   OpenClaw   │
│  (JSON-RPC)  │     │   Gateway    │
│  :8080       │     │   :18789     │
└──────────────┘     └──────┬───────┘
                            │
                    ┌───────┴───────┐
                    │    Ollama     │
                    │ (local LLM)  │
                    │   :11434     │
                    └──────────────┘
```

## Prerequisites

- Docker & Docker Compose v2
- A phone number for the Signal bot (separate from your personal number recommended)
- (Optional) [OpenRouter API key](https://openrouter.ai/keys) for cloud model fallback

## Quick Start

### 1. Clone and configure

```bash
git clone <repo-url> OpenClaw-Alice
cd OpenClaw-Alice
cp .env.example .env
# Edit .env — add your OpenRouter key if you want cloud fallback
```

### 2. Edit Signal phone numbers

Edit `config/openclaw.json` and replace:
- `+1BOTPHONENUMBER` → your bot's Signal number (e.g. `+15551234567`)
- `+1YOURPHONENUMBER` → your personal number (e.g. `+15557654321`)

### 3. Start everything

```bash
docker compose up -d
```

This starts 3 services:
- **signal-cli** — Signal bridge on port 8080
- **OpenClaw Gateway** (Alice) on port 18789
- **Ollama** — local model server on port 11434

### 4. Pull a tiny model

```bash
docker exec alice-ollama ollama pull qwen2.5:3b
```

This downloads ~2GB. Other tiny options: `phi3:mini` (~2.3GB), `gemma2:2b` (~1.6GB).

### 5. Register the Signal bot number

Register the bot's phone number with Signal:

```bash
# Register (will prompt for captcha if needed)
docker exec alice-signal-cli signal-cli -a +1BOTPHONENUMBER register

# If captcha required: open https://signalcaptchas.org/registration/generate.html
# Complete captcha, copy the signalcaptcha:// URL, then:
docker exec alice-signal-cli signal-cli -a +1BOTPHONENUMBER register --captcha 'signalcaptcha://...'

# Verify with the SMS code you receive
docker exec alice-signal-cli signal-cli -a +1BOTPHONENUMBER verify CODE
```

### 6. Approve DM pairing

Send a message to the bot number from your phone. Then approve:

```bash
docker compose run --rm openclaw-cli pairing list signal
docker compose run --rm openclaw-cli pairing approve signal <PAIRING_CODE>
```

### 7. Open the Control UI

Visit `http://localhost:18789` and paste the gateway token from your `.env`.

## Model Switching

Switch models in chat with `/model`:

```
/model ollama/qwen2.5:3b          # tiny, runs on any laptop
/model ollama/llama3.2:3b         # alternative tiny model
/model openrouter/anthropic/claude-sonnet-4-5   # cloud fallback
```

Pull more local models:
```bash
docker exec alice-ollama ollama pull llama3.2:3b
docker exec alice-ollama ollama pull phi3:mini
```

## Deploying to Your LLM PC

Clone this repo on your home machine, then:

```bash
cp .env.example .env
# Edit .env with your keys
# Edit config/openclaw.json with your Signal numbers
# Uncomment the GPU section in docker-compose.yml if you have NVIDIA
docker compose up -d
docker exec alice-ollama ollama pull qwen2.5:3b
```

For bigger models on your GPU machine:
```bash
docker exec alice-ollama ollama pull llama3.3
docker exec alice-ollama ollama pull qwen2.5-coder:32b
```

## Chat Commands

| Command | Description |
|---------|-------------|
| `/model` | Switch model (numbered picker) |
| `/model list` | Show all available models |
| `/new` or `/reset` | Start a fresh conversation |
| `/status` | Show session info (model, tokens, cost) |
| `/compact` | Compact session context |

## Troubleshooting

```bash
# Check health
docker exec alice-gateway curl -s http://127.0.0.1:18789/healthz

# Check readiness (shows which channels are connected)
docker exec alice-gateway curl -s http://127.0.0.1:18789/readyz

# Run doctor
docker compose run --rm openclaw-cli doctor

# Check Signal channel
docker compose run --rm openclaw-cli status

# View logs
docker logs alice-gateway --tail 50
docker logs alice-signal-cli --tail 50
```

## Links

- [OpenClaw Docs](https://docs.openclaw.ai/)
- [signal-cli](https://github.com/AsamK/signal-cli)
- [Ollama](https://ollama.com/)
- [OpenRouter](https://openrouter.ai/)
