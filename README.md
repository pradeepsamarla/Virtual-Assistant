# pipecat-quickstart

A virtual interactive voice assistant built with [Pipecat](https://docs.pipecat.ai/).
It listens to you, answers your queries, and reasons through problems in real
time over a cascade voice pipeline (STT → LLM → TTS).

By default it runs **fully local and free** — no API keys, no quotas. Each stage
is configurable via env vars:

The STT provider is **configurable** via the `STT_PROVIDER` env var:
- `whisper` (default) — free, fully local, no quota; runs in-process on CPU
- `cartesia` — Cartesia's cloud STT (subject to a daily quota)

The LLM provider is **configurable** via the `LLM_PROVIDER` env var:
- `ollama` (default) — a free, fully local model (no API key, runs on your machine)
- `deepseek` — DeepSeek's cloud API (requires a funded `DEEPSEEK_API_KEY`)

The TTS provider is **configurable** via the `TTS_PROVIDER` env var:
- `piper` (default) — free, fully local, no quota; runs in-process on CPU
- `cartesia` — Cartesia's cloud TTS (higher quality, but subject to a daily quota)

## Configuration

- **Bot Type**: Web
- **Transport(s)**: SmallWebRTC, Daily (WebRTC)
- **Pipeline**: Cascade
  - **STT**: Whisper (`base`, local, default) or Cartesia (cloud)
  - **LLM**: Ollama (`qwen2.5:1.5b`, local, default) or DeepSeek (`deepseek-chat`)
  - **TTS**: Piper (`en_US-amy-medium`, local, default) or Cartesia (cloud)

Both LLM services are OpenAI-compatible, so switching providers only changes the
LLM service in `server/bot.py` — the rest of the pipeline stays the same.

### Using the local (Ollama) LLM

1. Install Ollama: https://ollama.com/download
2. Pull the model: `ollama pull qwen2.5:1.5b`
3. Ensure `LLM_PROVIDER=ollama` in your `.env` (this is the default).

No API key or balance is required — the model runs locally.

### Using the local (Whisper) STT

STT defaults to local **Whisper** (`STT_PROVIDER=whisper`) — no API key, no quota.
The model (`WHISPER_MODEL`, default `base`) downloads automatically on first use
and runs on CPU. Use a smaller model (`tiny`) for speed or a larger one (`small`,
`medium`) for accuracy. Set `STT_PROVIDER=cartesia` to use Cartesia's cloud STT instead.

### Using the local (Piper) TTS

TTS defaults to local **Piper** (`TTS_PROVIDER=piper`) — no API key, no quota. The
voice model (`PIPER_VOICE`, default `en_US-amy-medium`) downloads automatically on
first use and runs on CPU. Set `TTS_PROVIDER=cartesia` to use Cartesia's cloud TTS
instead.

### Latency notes

On CPU the **LLM** dominates response time. To keep replies snappy: the default
model is the small/fast `qwen2.5:1.5b`, and the Docker setup sets
`OLLAMA_KEEP_ALIVE=-1` so the model stays resident in RAM (no per-request cold
start). Pipecat also streams the LLM output to TTS sentence-by-sentence, so the
avatar starts speaking after the first sentence rather than the whole reply.

## Setup

### Server

1. **Navigate to server directory**:

   ```bash
   cd server
   ```

2. **Install dependencies**:

   ```bash
   uv sync
   ```

3. **Configure environment variables**:

   ```bash
   cp .env.example .env
   # Edit .env and add your API keys
   ```

   No keys are required by default — STT (Whisper), LLM (Ollama), and TTS (Piper)
   all run locally. Keys are only needed if you opt into a cloud provider:
   `CARTESIA_API_KEY` (for `STT_PROVIDER=cartesia` / `TTS_PROVIDER=cartesia`),
   `DEEPSEEK_API_KEY` (for `LLM_PROVIDER=deepseek`), or `DAILY_API_KEY` (Daily transport).

4. **Run the bot**:

   - SmallWebRTC: `uv run bot.py`
   - Daily: `uv run bot.py --transport daily`

## 3D talking avatar (local lip-sync)

A custom web client renders a 3D avatar whose mouth lip-syncs to the bot's
speech in real time. It runs entirely in the browser with **no GPU**: the bot's
audio is analyzed client-side ([HeadAudio](https://github.com/met4citizen/TalkingHead)
MFCC + viseme classifier) and mapped onto the avatar's morph targets via
[TalkingHead.js](https://github.com/met4citizen/TalkingHead) + Three.js.

- Start the bot, then open **http://localhost:7860/avatar/**
- Click **Start conversation**. The avatar greets you; talk (if you have a mic)
  or type in the box. The mouth animates while it speaks.
- The default avatar is a [Ready Player Me](https://readyplayer.me) model. Swap
  in your own by appending `?avatar=<RPM_glb_url>` (optionally `&body=M`), e.g.
  `http://localhost:7860/avatar/?avatar=https://models.readyplayer.me/<id>.glb`

The client is served same-origin by the bot (FastAPI `StaticFiles`) so the
WebRTC offer to `/api/offer` has no CORS issues. The stock Pipecat Playground
remains available at `/client/` for audio-only testing.

## Run with Docker (recommended — one command, no local installs)

The whole stack (local LLM + bot + 3D-avatar client) runs from Docker Compose.
You don't install Python, uv, or Ollama — only Docker. The model is pulled
automatically into a named volume on first run and reused afterwards.

**Prerequisites:** [Docker](https://docs.docker.com/get-docker/) (with Compose v2).
No API keys are required — the whole pipeline runs locally by default.

```bash
# 1. Clone
git clone https://github.com/<owner>/<repo>.git
cd <repo>

# 2. Bring everything up (first run builds the image + pulls the model)
docker compose up --build
```

Then open **http://localhost:7860/avatar/** and click **Start conversation**.

What it starts:
- `ollama` — local LLM runtime; a one-shot `ollama-pull` service auto-pulls
  `qwen2.5:1.5b` into a persistent volume.
- `bot` — the Pipecat bot + avatar client on port 7860, wired to the Ollama
  container automatically (`OLLAMA_BASE_URL`, model name, etc.).

Optional overrides go in `.env` (e.g. `WHISPER_MODEL=small`,
`OLLAMA_MODEL=qwen2.5:7b`, or `STT_PROVIDER=cartesia` + `CARTESIA_API_KEY=...`
to use the cloud provider). To stop: `docker compose down` (add `-v` to also
delete the cached model).

> **Networking note:** the `bot` service uses `network_mode: host` so WebRTC
> media can reach the browser without publishing an ephemeral UDP port range.
> This works on Linux and WSL2. On Docker Desktop for macOS/Windows, host
> networking is limited — use the "Run on your own machine" path below instead,
> or run inside a Linux/WSL2 host.

## Run on your own machine (quick install)

Works on macOS, Linux, or Windows (WSL recommended). Everything runs locally and
free by default — no API keys.

**Prerequisites**
- [Python 3.11+](https://www.python.org/downloads/)
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (Python package manager)
- [Ollama](https://ollama.com/download) (local LLM runtime)

**Steps**
```bash
# 1. Clone
git clone https://github.com/<owner>/<repo>.git
cd <repo>

# 2. Local LLM: install Ollama, then pull the model (one-time, ~2 GB)
ollama pull qwen2.5:1.5b
#   make sure the Ollama service is running (the installer starts it; or run `ollama serve`)

# 3. Install the bot's Python deps
cd server
uv sync

# 4. (Optional) configure overrides — no keys needed for the local default
cp .env.example .env
#   defaults: STT=Whisper, LLM=Ollama, TTS=Piper (all local, no key required)

# 5. Run the bot
uv run bot.py
```

Then open **http://localhost:7860/avatar/** in Chrome and click **Start
conversation**. Talk (if you have a mic) or type — the 3D avatar speaks and
lip-syncs the replies. First reply may take a few seconds while Ollama loads the
model into memory.

> No GPU required. The first run downloads the Ollama model, the Whisper STT
> model, and the Piper voice; later runs are fully offline.

### Troubleshooting

- **`Error during completion: Connection error` (Ollama):** the bot can't reach
  Ollama. Make sure `ollama serve` is running, then on WSL/Linux prefer
  `127.0.0.1` over `localhost` — `localhost` may resolve to IPv6 (`::1`) while
  Ollama listens on IPv4. Set `OLLAMA_BASE_URL=http://127.0.0.1:11434/v1`.
- **`Error code: 400 ... invalid model name`:** the model name is empty. This
  happens if `.env` has `OLLAMA_MODEL=` (blank). Set `OLLAMA_MODEL=qwen2.5:1.5b`
  (and run `ollama pull qwen2.5:1.5b`). Blank values now fall back to defaults.
- **No `sudo` password (e.g. WSL):** install Ollama without root by extracting
  the release tarball into your home dir and adding it to `PATH`:
  ```bash
  curl -fL https://ollama.com/download/ollama-linux-amd64.tgz -o ollama.tgz
  mkdir -p ~/.ollama-install && tar -xzf ollama.tgz -C ~/.ollama-install
  echo 'export PATH="$HOME/.ollama-install/bin:$PATH"' >> ~/.bashrc
  export PATH="$HOME/.ollama-install/bin:$PATH"
  ```

## Project Structure

```
pipecat-quickstart/
├── server/              # Python bot server
│   ├── bot.py           # Main bot implementation
│   ├── client/          # Custom 3D-avatar web client (served at /avatar/)
│   │   ├── index.html   # Avatar UI + Pipecat JS SDK + lip-sync wiring
│   │   ├── avatars/     # Ready Player Me GLB model(s)
│   │   └── vendor/      # HeadAudio worklet + viseme model (vendored)
│   ├── pyproject.toml   # Python dependencies
│   ├── .env.example     # Environment variables template
│   ├── .env             # Your API keys (git-ignored)
│   ├── Dockerfile       # Container image for Pipecat Cloud (arm64 base)
│   ├── Dockerfile.local # Self-hosted image used by docker compose
│   └── pcc-deploy.toml  # Pipecat Cloud deployment config
├── docker-compose.yml   # One-command local deploy (Ollama + bot)
├── .gitignore           # Git ignore patterns
└── README.md            # This file
```

## Deploying to Pipecat Cloud

This project is configured for deployment to Pipecat Cloud. You can learn how to deploy to Pipecat Cloud in the [Pipecat Quickstart Guide](https://docs.pipecat.ai/getting-started/quickstart#step-2-deploy-to-production).

Refer to the [Pipecat Cloud Documentation](https://docs.pipecat.ai/deployment/pipecat-cloud/introduction) to learn more about configuring, deploying, and managing your agents in Pipecat Cloud.

## Learn More

- [Pipecat Documentation](https://docs.pipecat.ai/)
- [Pipecat GitHub](https://github.com/pipecat-ai/pipecat)
- [Pipecat Examples](https://github.com/pipecat-ai/pipecat-examples)
- [Discord Community](https://discord.gg/pipecat)