# OpenClaw + 本地 LLM：哪些真的能用

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[English](README.md)

使用消費級 NVIDIA GPU 在本地跑 [OpenClaw](https://openclaw.ai) + [Ollama](https://ollama.com) LLM 的完整指南與工具。包含模型比較、Benchmark 數據、自動化安裝腳本，以及實戰踩坑紀錄。

> 測試環境：RTX 5070 Ti 16GB · OpenClaw 2026.3.13 · Ollama 0.17.7

---

## 為什麼需要這個 Repo？

網路上大部分「最佳本地 LLM」文章測的是**聊天**效果，不是 **agent tool calling**。OpenClaw 的 system prompt 達 12,000+ tokens（tool schemas + workspace files），大部分小模型扛不住。我們測了 13 個模型，只有 2 個能穩定使用。

## 模型比較表（RTX 5070 Ti 16GB）

| 模型 | 大小 | 搜尋 | 單一操作 | 複合操作 | 結論 |
|------|------|------|----------|----------|------|
| **qwen3-vl:8b-instruct** | 6.1 GB | ✅ | ✅ | ✅ | **推薦首選** |
| mistral-small3.1:24b | 15 GB | ✅ | ✅ | ✅ | 穩定但太慢（~10 tok/s） |
| qwen2.5:14b-instruct | 9.0 GB | ✅ | ⚠️ | ⚠️ | 能用但囉嗦猶豫 |
| qwen3:4b | 2.5 GB | ✅ | ✅ | ❌ | 複合指令崩潰 |
| gpt-oss:20b | 13 GB | ❌ | ✅/❌ | — | 極不穩定 |
| phi4:14b | 9.1 GB | — | — | — | 不支援 tools API |
| llama3.2-vision:11b | 7.8 GB | — | — | — | 不支援 tools API |
| llama3.1:8b | 4.9 GB | ❌ | ❌ | — | 只回吐文字 |
| *另外 5 個模型* | | | | | *全部失敗* |

> **關鍵發現：** AGENTS.md 的指令品質比模型大小更重要。詳見[架設教程](docs/setup-guide.md)。

### 推理引擎：Ollama vs vLLM

| 指標 | Ollama | vLLM |
|------|--------|------|
| 單請求 tok/s | **130** | 5-11 |
| 安裝難度 | 一行指令 | 需 Docker build（Blackwell 需特殊 image） |
| WSL2 支援 | 原生 | `pin_memory` 限制，效能大打折扣 |
| 多人併發 | 排隊處理 | PagedAttention 批次處理（vLLM 優勢場景） |

> vLLM 擅長多人併發。單人 Telegram bot 場景下 Ollama 快 10-25 倍。

## 架構

```
Telegram / 瀏覽器
  | HTTP / WebSocket
OpenClaw（Mac，npm global）— Telegram bot、Agent UI
  | Ollama native API（http://PC_LAN_IP:11434）
Ollama（PC Linux systemd）
  |-- qwen3-vl:8b-instruct（推薦，vision + tools，無 thinking）
  | http://PC_LAN_IP:8080
SearXNG（PC Docker）— 本地搜尋，不需 API key
```

## 文件

| 文件 | 說明 |
|------|------|
| [架設教程](docs/setup-guide.md) | 完整安裝教程，含 Claude Code 自動化 prompt |
| [模型 Benchmark](docs/model-benchmark.md) | 詳細 tok/s、TTFT、VRAM 數據 |
| [踩坑紀錄](docs/pitfalls.md) | 24 個實戰踩坑，含根因與修法 |

## Skills

| Skill | 說明 |
|-------|------|
| [llm-benchmark](skills/llm-benchmark/) | Ollama 模型自動化 benchmark，含 CPU offload 偵測 |
| [searxng](skills/searxng/) | OpenClaw 本地搜尋整合（SearXNG） |

## 快速開始

完整步驟見[架設教程](docs/setup-guide.md)。簡要流程：

1. **PC：** 安裝 Ollama + SearXNG + 拉 `qwen3-vl:8b-instruct`
2. **Mac：** 安裝 OpenClaw + 設定 `openclaw.json` + 撰寫 `AGENTS.md`
3. **測試：** 透過 Telegram 或 Web UI 測試搜尋 + tool calling
