# 本地 LLM Benchmark 與模型選擇報告

> **English summary:** Comprehensive benchmark and model selection report for running OpenClaw with local Ollama LLMs on RTX 5070 Ti 16GB. 13 models tested across two dimensions: raw performance (tok/s, TTFT, VRAM across context windows 2K-32K) and OpenClaw tool calling reliability (search, single/compound light control). Also includes Ollama vs vLLM inference engine comparison. Final recommendation: qwen3-vl:8b-instruct. The document is written in Traditional Chinese.

> 測試環境：RTX 5070 Ti 16GB · Ollama 0.17.7 · OpenClaw 2026.3.13
> 最後更新：2026-03-19

---

## 結論（先看這裡）

**推薦模型：`qwen3-vl:8b-instruct`**

- 130 tok/s @32K ctx，VRAM 僅 6-8 GB
- OpenClaw tool calling 全通過（搜尋、單燈、複合操作、色溫、顏色）
- 支援 vision
- 16GB VRAM 下 qwen3-vl 最大只有 8B（下一個是 30B，需 20GB）

**推薦推理引擎：Ollama**（單人使用場景）

---

## 測試環境

| 項目 | 規格 |
|------|------|
| GPU | NVIDIA RTX 5070 Ti 16GB VRAM |
| RAM | 58GB |
| OS | Windows 11 + WSL2 Ubuntu 22.04 |
| Ollama | v0.17.7 |

---

## Part 1：Benchmark（純效能測試）

測試方法：OpenClaw 停止、VRAM 淨空後，用 4 道題（邏輯推理、多步推理、程式設計、程式除錯）測試各 context window 下的 tok/s。

### 速度總覽

| 模型 | 大小 | ctx=2K | ctx=4K | ctx=8K | ctx=16K | ctx=32K |
|------|------|--------|--------|--------|---------|---------|
| **qwen3-vl:8b-instruct** | 6.1 GB | 108 | 111 | 111 | 129 | **131** |
| qwen3-vl:8b（thinking） | 5.9 GB | 95 | 96 | 100 | — | — |
| llama3.1:8b | 4.7 GB | 143 | 142 | 143 | — | — |
| qwen2.5:14b-instruct | 9.0 GB | 79 | 79 | 79 | 79 | **19** ⚠️ |
| gemma3:12b | 8.1 GB | 82 | 82 | 81 | — | — |
| ministral-3:14b | 8.7 GB | 78 | 86 | 76 | 85 | **49** ⚠️ |
| phi4:14b | 9.1 GB | 77 | 77 | 77 | 77 | — |
| mistral-small3.1:24b | 15 GB | 31 | 26 | 20 | 15 | **10** |

> ⚠️ qwen2.5:14b @32K：VRAM 接近滿載，速度暴跌到 19 tok/s
> ⚠️ ministral-3:14b @32K：VRAM 14.5GB（88.8%），速度劣化 42%
> qwen3-vl:8b-instruct @32K：速度不掉，VRAM 僅 ~8GB，大量餘裕

### TTFT（首 token 延遲）

| 模型 | TTFT |
|------|------|
| qwen3-vl:8b-instruct | ~0.02-0.03s |
| qwen2.5:14b-instruct @16K | ~0.04s |
| qwen2.5:14b-instruct @32K | ~0.14s |
| mistral-small3.1:24b | ~0.04-0.14s |

### 回答品質

| 模型 | 邏輯題 | 推理題 | 程式設計 | 程式除錯 |
|------|--------|--------|----------|----------|
| qwen3-vl:8b-instruct | ✅ | ✅ | ✅ | ✅ |
| qwen2.5:14b-instruct | ✅ | ✅ | ✅ | ✅ |
| mistral-small3.1:24b | ✅ | ✅ | ✅ | ✅ |
| ministral-3:14b | ✅ | ✅ | ✅ | ✅ |
| phi4:14b | ✅ | ✅ | ✅ | ✅ |
| gemma3:12b | ✅ | ✅ | ✅ | ✅ |
| llama3.1:8b | ❌ | ⚠️ | ✅ | ✅ |

### Context Window 甜蜜點（16GB VRAM）

| 模型大小 | 建議 ctx 上限 | 理由 |
|---------|-------------|------|
| 8B（~6GB） | 32768-49152 | VRAM 充裕，速度不掉 |
| 14B（~9GB） | 16384 | 32K 時 VRAM 接近滿載，速度暴跌 |
| 24B（~15GB） | 32768 | 已是 VRAM 極限，10 tok/s 勉強可用 |

### Thinking vs Instruct（qwen3-vl:8b）

| 指標 | thinking | instruct |
|------|----------|----------|
| tok/s | 90-117 | 90-133 |
| 實際回應時間 | 慢（生成 2000-17000 thinking tokens） | 快（只生成回答） |
| 加速倍數 | — | **2x-8x**（日常任務） |

> thinking 版的 tok/s 看起來差不多，但會生成大量隱藏推理 token，實際等待時間長很多。instruct 版是唯一正確選擇。

---

## Part 2：OpenClaw Tool Calling 實測

Benchmark 只測速度和品質，但 OpenClaw 的核心需求是 **tool calling**。以下是在完整 OpenClaw 環境（12K+ token system prompt）下的實測結果。

### 測試項目

- **搜尋：** 透過 exec tool 呼叫 `searxng-search "關鍵字"`
- **單燈操作：** 透過 exec tool 呼叫 `tradfri 小夜燈 開`
- **複合操作：** 同時操作多盞燈（關餐廳 + 開客廳 + 調亮度 + 改顏色）

### 結果

| 模型 | 大小 | 搜尋 | 單燈 | 複合 | 結論 |
|------|------|------|------|------|------|
| **qwen3-vl:8b-instruct** | 6.1 GB | ✅ | ✅ | ✅ | **推薦，全通過** |
| mistral-small3.1:24b | 15 GB | ✅ | ✅ | ✅ | 穩定但太慢（~10 tok/s @32K） |
| qwen2.5:14b-instruct | 9.0 GB | ✅ | ⚠️ | ⚠️ | 能用但囉嗦猶豫，舊世代不如 qwen3 |
| qwen3:4b | 2.5 GB | ✅ | ✅ | ❌ | 單指令 OK，複合崩 |
| qwen3:4b-instruct | 2.5 GB | ❌ | ❌ | — | 完全無回應 |
| frob/qwen3.5-instruct:9b | 6.6 GB | ❌ | ❌ | — | 不呼叫 exec |
| gpt-oss:20b | 13 GB | ❌ | ✅/❌ | — | 極不穩定 |
| orieg/gemma3-tools:12b-it-qat | 8.9 GB | ❌ | ❌ | — | 吐文字不呼叫 tool |
| MFDoom/deepseek-r1-tool-calling:14b | 9.0 GB | ❌ | — | — | 吐 JSON 文字 |
| phi4:14b | 9.1 GB | — | — | — | Ollama 不支援 tools API |
| llama3.2-vision:11b | 7.8 GB | — | — | — | Ollama 不支援 tools API |
| llama3.1:8b | 4.9 GB | ❌ | ❌ | — | 只回吐文字不呼叫 tool |
| llama3.2:3b | 2.0 GB | ❌ | ❌ | — | 完全無回應 |

### 為什麼大部分模型都失敗？

OpenClaw 的完整 system prompt 達 **12,000+ tokens**（tool schemas + AGENTS.md + workspace files）。小模型的「注意力」被佔滿，無法正確決策呼叫哪個 tool。

**關鍵發現：** `qwen3-vl:8b-instruct` 之前也曾失敗（把 shell 指令名當 tool name），但在完善 AGENTS.md 指令後成功穩定運作。**AGENTS.md 的指令品質比模型大小更重要。**

---

## Part 3：推理引擎比較（Ollama vs vLLM）

| 指標 | Ollama | vLLM |
|------|--------|------|
| 單請求 tok/s | **130** | 5-11 |
| 安裝難度 | `curl \| sh` 一行 | Docker build（Blackwell 需特殊 image） |
| WSL2 支援 | 原生 | `pin_memory=False` 限制，效能大打折扣 |
| 多人併發 | 排隊處理 | PagedAttention 批次處理（優勢場景） |

> 測試環境：Ollama + qwen3-vl:8b-instruct Q4_K_M vs vLLM + Qwen2.5-VL-7B-Instruct-AWQ，同 GPU

**結論：** vLLM 的核心優勢是 continuous batching（多人併發）。單人 Telegram bot 場景下 Ollama 快 10-25 倍。vLLM 在 WSL2 + 消費級 GPU 上水土不服，不建議。

---

## 最終推薦

### RTX 5070 Ti 16GB VRAM（實測環境）

| 場景 | 推薦模型 | ctx | 速度 |
|------|---------|-----|------|
| **OpenClaw agent（首選）** | `qwen3-vl:8b-instruct` | 32768 | 131 tok/s |
| 不需 vision 的備選 | `mistral-small3.1:24b` | 32768 | 10 tok/s |
| 純聊天（不用 tool calling） | `llama3.1:8b` | 32768 | 143 tok/s |

### 不推薦

- **gemma3 / phi4** — 不支援 Ollama tools API，無法用於 OpenClaw
- **llama3.2-vision** — 不支援 Ollama tools API
- **qwen3:4b** — 複合操作崩潰
- **gpt-oss:20b** — tool calling 極不穩定

---

*基於 RTX 5070 Ti 16GB · Ollama 0.17.7 · OpenClaw 2026.3.13 · 2026-03-19 實測*
