<div align="center">
  <img alt="OpenJarvis" src="assets/OpenJarvis_Horizontal_Logo.png" width="400">

  **Personal AI, On Personal Devices.**

 [![License](https://img.shields.io/badge/license-Apache%202.0-green.svg)](LICENSE)
  [![Python](https://img.shields.io/badge/python-%3E%3D3.10-blue.svg)](pyproject.toml)
  [![Discord](https://img.shields.io/badge/disjoin-7289da?logo=discord&logoColor=white)](https://discord.gg/wfXEkpPX)

  ---
</div>

## What is OpenJarvis?

OpenJarvis is an open-source framework for building **local-first personal AI agents** that run on your own device—no cloud dependencies, no per-token costs, full privacy.

```bash
uv run jarvis ask "Summarize my emails"
```

## Features

- **Multi-Engine Support**: Ollama, vLLM, SGLang, llama.cpp, OpenAI, Anthropic, Gemini
- **Built-in Agents**: Orchestrator, ReAct, OpenHands (code execution), Deep Research
- **Tool System**: File I/O, shell, web search, memory, calculator, git
- **Memory Backends**: SQLite, FAISS, BM25, ColBERT, hybrid
- **Telemetry**: Energy monitoring (NVIDIA/AMD/Apple Silicon), FLOPs, latency, cost
- **Desktop App**: React + Tauri (or use as API server)

## Quick Start

```bash
# Clone & install
git clone https://github.com/open-jarvis/OpenJarvis.git
cd OpenJarvis
uv sync --extra server

# Build Rust extension
uv run maturin develop -m rust/crates/openjarvis-python/Cargo.toml

# Start Ollama
ollama serve &
ollama pull qwen3:8b

# Run
uv run jarvis ask "Hello"
```

Or use cloud APIs:

```bash
export OPENAI_API_KEY=sk-...
uv run jarvis ask "Hello"
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     OpenJarvis                          │
├─────────────┬─────────────┬─────────────┬───────────────┤
│   Agents    │   Engine    │   Tools     │   Memory      │
│  (Python)   │  (Ollama)   │  (file/io)  │  (FAISS)      │
├─────────────┴─────────────┴─────────────┴───────────────┤
│                   Rust Core (PyO3)                      │
│         Telemetry | Traces | Learning                    │
└─────────────────────────────────────────────────────────┘
```

## Documentation

- [Installation](docs/getting-started/installation.md)
- [Quick Start](docs/getting-started/quickstart.md)
- [Architecture](docs/architecture/overview.md)
- [Python SDK](docs/user-guide/python-sdk.md)
- [CLI Reference](docs/user-guide/cli.md)

## API Server

```bash
uv run jarvis serve --port 8000
```

Then open http://localhost:8000

## Testing

```bash
uv run pytest tests/ -v --tb=short
```

## Tech Stack

| Layer | Technology |
|-------|------------|
| Core | Python 3.10+, Rust |
| Bindings | PyO3 (maturin) |
| Server | FastAPI, Uvicorn |
| Frontend | React 19, TypeScript, Tauri |
| Telemetry | SQLite, energy monitors |

## License

[Apache 2.0](LICENSE) — OpenJarvis Contributors

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup and guidelines.
