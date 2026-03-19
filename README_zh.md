# OpenClaw + 本地 LLM：到底哪些能用（以及哪些會讓你懷疑人生）

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[English](README.md)

想在自己的消費級 NVIDIA GPU 上跑 [OpenClaw](https://openclaw.ai) + [Ollama](https://ollama.com)？好志向。我們花了不合理的時間踩坑，就是為了讓你不用再踩一次。這個 repo 包含模型比較、Benchmark 數據、自動化安裝腳本，以及一份血淚交織的實戰踩坑紀錄。

> 測試環境：RTX 5070 Ti 16GB -- OpenClaw 2026.3.13 -- Ollama 0.17.7

---

## 為什麼會有這個 Repo？

因為網路上大部分「最佳本地 LLM」文章測的都是**聊天**效果——就是「幫我寫一首關於貓的詩」那種。沒人測 **agent tool calling**，而這偏偏是 OpenClaw 的命脈。OpenClaw 的 system prompt 達 12,000+ tokens（tool schemas + workspace files），這個長度堪稱小模型殺手。我們測了 13 個模型，只有 2 個活了下來。其他 11 個以各種令人嘆為觀止的方式陣亡。

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

> **沒人告訴你的真相：** AGENTS.md 的指令品質比模型大小更重要。好好寫指令，8B 模型也能聽話；指令寫爛，24B 模型也會給你亂搞。詳見[架設教程](docs/setup-guide.md)，那些我們花太久才搞懂的細節。

### 推理引擎：Ollama vs vLLM（又名「我們兩個都試了，你不用再試」）

| 指標 | Ollama | vLLM |
|------|--------|------|
| 單請求 tok/s | **130** | 5-11 |
| 安裝難度 | 一行指令 | 需 Docker build（Blackwell 需特殊 image） |
| WSL2 支援 | 原生 | `pin_memory` 限制，效能大打折扣 |
| 多人併發 | 排隊處理 | PagedAttention 批次處理（vLLM 優勢場景） |

> vLLM 多人併發確實厲害。但單人 Telegram bot 場景？Ollama 快 10-25 倍，而且安裝過程不會讓你想砸鍵盤。

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
| [踩坑紀錄](docs/pitfalls.md) | 24 個用血和時間換來的實戰踩坑 |

## Skills

本專案使用 [kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills) 的 skills：

| Skill | 說明 |
|-------|------|
| [llm-benchmark](https://github.com/KerberosClaw/kc_ai_skills/tree/main/llm-benchmark) | Ollama 模型自動化 benchmark，含 CPU offload 偵測 |
| [searxng](https://github.com/KerberosClaw/kc_ai_skills/tree/main/searxng) | OpenClaw 本地搜尋整合（SearXNG） |

## 快速開始（給沒耐心的你）

完整步驟見[架設教程](docs/setup-guide.md)。懶人版：

1. **PC：** 安裝 Ollama + SearXNG + 拉 `qwen3-vl:8b-instruct`
2. **Mac：** 安裝 OpenClaw + 設定 `openclaw.json` + 撰寫 `AGENTS.md`（這步比你想的重要得多）
3. **測試：** 透過 Telegram 或 Web UI 測試搜尋 + tool calling，然後在它真的動起來的時候鬆一口氣
