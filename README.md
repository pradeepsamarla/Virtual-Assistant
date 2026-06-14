# pipecat-quickstart

A virtual interactive voice assistant built with [Pipecat](https://docs.pipecat.ai/).
It listens to you, answers your queries, and reasons through problems in real
time over a cascade voice pipeline (STT → LLM → TTS).

The LLM provider is **configurable** via the `LLM_PROVIDER` env var:
- `ollama` (default) — a free, fully local model (no API key, runs on your machine)
- `deepseek` — DeepSeek's cloud API (requires a funded `DEEPSEEK_API_KEY`)

## Configuration

- **Bot Type**: Web
- **Transport(s)**: SmallWebRTC, Daily (WebRTC)
- **Pipeline**: Cascade
  - **STT**: Cartesia
  - **LLM**: Ollama (`qwen2.5:3b`, local, default) or DeepSeek (`deepseek-chat`)
  - **TTS**: Cartesia

Both LLM services are OpenAI-compatible, so switching providers only changes the
LLM service in `server/bot.py` — the rest of the pipeline stays the same.

### Using the local (Ollama) LLM

1. Install Ollama: https://ollama.com/download
2. Pull the model: `ollama pull qwen2.5:3b`
3. Ensure `LLM_PROVIDER=ollama` in your `.env` (this is the default).

No API key or balance is required — the model runs locally.

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

   Required key: `CARTESIA_API_KEY` (used for both STT and TTS). The LLM is
   local Ollama by default (no key needed); set `LLM_PROVIDER=deepseek` and
   `DEEPSEEK_API_KEY` to use DeepSeek instead. `DAILY_API_KEY` is only needed
   for the Daily transport.

4. **Run the bot**:

   - SmallWebRTC: `uv run bot.py`
   - Daily: `uv run bot.py --transport daily`

## Project Structure

```
pipecat-quickstart/
├── server/              # Python bot server
│   ├── bot.py           # Main bot implementation
│   ├── pyproject.toml   # Python dependencies
│   ├── .env.example     # Environment variables template
│   ├── .env             # Your API keys (git-ignored)
│   ├── Dockerfile       # Container image for Pipecat Cloud
│   └── pcc-deploy.toml  # Pipecat Cloud deployment config
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