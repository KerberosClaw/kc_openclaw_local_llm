# OpenClaw + Local LLM: What Actually Works

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[正體中文](README_zh.md)

Guide and tools for running [OpenClaw](https://openclaw.ai) with local LLMs via [Ollama](https://ollama.com) on consumer NVIDIA GPUs. Includes model comparison, benchmark data, setup automation, and battle-tested pitfalls.

> Tested on RTX 5070 Ti 16GB · OpenClaw 2026.3.13 · Ollama 0.17.7

---

## Why This Repo?

Most "best local LLM" articles test models for **chat** — not for **agent tool calling**. OpenClaw's system prompt is 12,000+ tokens (tool schemas + workspace files), which breaks most small models. We tested 13 models and only 2 work reliably.

## Model Comparison (RTX 5070 Ti 16GB)

| Model | Size | Search | Single Op | Compound Op | Verdict |
|-------|------|--------|-----------|-------------|---------|
| **qwen3-vl:8b-instruct** | 6.1 GB | ✅ | ✅ | ✅ | **Recommended** |
| mistral-small3.1:24b | 15 GB | ✅ | ✅ | ✅ | Stable but slow (~10 tok/s) |
| qwen2.5:14b-instruct | 9.0 GB | ✅ | ⚠️ | ⚠️ | Works but verbose/hesitant |
| qwen3:4b | 2.5 GB | ✅ | ✅ | ❌ | Compound ops fail |
| gpt-oss:20b | 13 GB | ❌ | ✅/❌ | — | Extremely unstable |
| phi4:14b | 9.1 GB | — | — | — | No tools API support |
| llama3.2-vision:11b | 7.8 GB | — | — | — | No tools API support |
| llama3.1:8b | 4.9 GB | ❌ | ❌ | — | Echoes text, no tool calls |
| *+ 5 more models* | | | | | *All failed* |

> **Key finding:** AGENTS.md instruction quality matters more than model size. See [setup guide](docs/setup-guide.md) for details.

### Inference Engine: Ollama vs vLLM

| Metric | Ollama | vLLM |
|--------|--------|------|
| Single request tok/s | **130** | 5-11 |
| Install complexity | One liner | Docker build (Blackwell needs special image) |
| WSL2 support | Native | `pin_memory` limitation, major perf hit |
| Multi-user concurrency | Sequential | PagedAttention batching (vLLM advantage) |

> vLLM excels at concurrent serving. For single-user Telegram bots, Ollama wins by 10-25x.

## Architecture

```
Telegram / Browser
  | HTTP / WebSocket
OpenClaw (Mac, npm global) — Telegram bot, Agent UI
  | Ollama native API (http://PC_LAN_IP:11434)
Ollama (PC Linux systemd)
  |-- qwen3-vl:8b-instruct (recommended, vision + tools, no thinking)
  | http://PC_LAN_IP:8080
SearXNG (PC Docker) — Local search, no API key needed
```

## Docs

| Document | Description |
|----------|-------------|
| [Setup Guide](docs/setup-guide.md) | Full installation tutorial with Claude Code automation prompts |
| [Model Benchmark](docs/model-benchmark.md) | Detailed tok/s, TTFT, VRAM data across context windows |
| [Pitfalls](docs/pitfalls.md) | 24 battle-tested pitfalls with root causes and fixes |

## Skills

| Skill | Description |
|-------|-------------|
| [llm-benchmark](skills/llm-benchmark/) | Automated Ollama model benchmark with CPU offload detection |
| [searxng](skills/searxng/) | OpenClaw local search integration via SearXNG |

## Quick Start

See the [Setup Guide](docs/setup-guide.md) for full instructions. TL;DR:

1. **PC:** Install Ollama + SearXNG + pull `qwen3-vl:8b-instruct`
2. **Mac:** Install OpenClaw + configure `openclaw.json` + write `AGENTS.md`
3. **Test:** Search + tool calling via Telegram or Web UI
