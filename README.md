# OpenClaw + Local LLM: A Field Guide to What Actually Works (and What Absolutely Doesn't)

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[正體中文](README_zh.md)

So you want to run [OpenClaw](https://openclaw.ai) with local LLMs via [Ollama](https://ollama.com) on a consumer NVIDIA GPU. Noble goal. We spent an unreasonable amount of time so you don't have to. This repo has model comparisons, benchmark data, setup automation, and a painfully honest record of every dumb mistake we made along the way.

> Tested on RTX 5070 Ti 16GB -- OpenClaw 2026.3.13 -- Ollama 0.17.7

---

## Why Does This Repo Exist?

Because every "best local LLM" article on the internet tests models for **chat** -- you know, "write me a poem about cats" stuff. Nobody tests for **agent tool calling**, which is the part that actually matters for OpenClaw. Its system prompt is 12,000+ tokens of tool schemas and workspace files, which turns out to be a fantastic way to make most small models completely fall apart. We tested 13 models. Only 2 survived. The other 11 died in increasingly creative ways.

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

> **The thing nobody tells you:** AGENTS.md instruction quality matters more than model size. Turns out you can make an 8B model behave if you just ask nicely (and very specifically). See [setup guide](docs/setup-guide.md) for the details that took us way too long to figure out.

### Inference Engine: Ollama vs vLLM (a.k.a. "We Tried Both So You Don't Have To")

| Metric | Ollama | vLLM |
|--------|--------|------|
| Single request tok/s | **130** | 5-11 |
| Install complexity | One liner | Docker build (Blackwell needs special image) |
| WSL2 support | Native | `pin_memory` limitation, major perf hit |
| Multi-user concurrency | Sequential | PagedAttention batching (vLLM advantage) |

> vLLM is genuinely great at concurrent serving. But for a single-user Telegram bot? Ollama wins by 10-25x, and you get to keep your sanity during installation.

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

This project uses skills from [kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills):

| Skill | Description |
|-------|-------------|
| [llm-benchmark](https://github.com/KerberosClaw/kc_ai_skills/tree/main/llm-benchmark) | Automated Ollama model benchmark with CPU offload detection |
| [searxng](https://github.com/KerberosClaw/kc_ai_skills/tree/main/searxng) | OpenClaw local search integration via SearXNG |

## Quick Start (for the Impatient)

See the [Setup Guide](docs/setup-guide.md) for the full walkthrough. But if you just want the cliff notes:

1. **PC:** Install Ollama + SearXNG + pull `qwen3-vl:8b-instruct`
2. **Mac:** Install OpenClaw + configure `openclaw.json` + write `AGENTS.md` (this part is more important than you think)
3. **Test:** Search + tool calling via Telegram or Web UI, then breathe a sigh of relief when it actually works
