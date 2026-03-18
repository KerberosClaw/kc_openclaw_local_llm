# OpenClaw + Local LLM 踩坑紀錄

> **English summary:** A comprehensive list of pitfalls encountered while setting up OpenClaw with local Ollama LLMs on a two-machine architecture (Mac + PC with NVIDIA GPU). Covers config issues, tool calling failures, model compatibility, Docker networking, Telegram DNS, VRAM management, and the critical role of AGENTS.md. Based on real-world testing with 13+ models on RTX 5070 Ti 16GB. The document is written in Traditional Chinese.

基於 OpenClaw 2026.3.13 + Ollama 0.17.7 + RTX 5070 Ti 16GB 實測。

---

## 設定類

### #1 `~/.openclaw/` 權限問題（EACCES）
- **原因：** 以 sudo 或 root 編輯設定檔，owner 變成 root，OpenClaw 讀不到
- **解法：** `sudo chown $USER ~/.openclaw/openclaw.json`
- **預防：** 用 python3 程式寫入設定檔，避免 sudo

### #2 Pairing required 死結
- **解法 A（瀏覽器）：** `openclaw.json` 加 `"dangerouslyDisableDeviceAuth": true`
- **解法 B（macOS native App）：** `openclaw devices list` → `openclaw devices approve <requestId>`
- **注意：** `dangerouslyDisableDeviceAuth` 只繞過瀏覽器 device auth，不繞過原生 App 的 operator pairing

### #3 `/new` 只顯示 heartbeat，main session 不出現
- **原因：** `~/.openclaw/workspace/BOOTSTRAP.md` 存在
- **解法：** `rm ~/.openclaw/workspace/BOOTSTRAP.md`

### #4 openclaw.json 字串內有未跳脫的雙引號
- **現象：** gateway 啟動但無任何功能，err log 顯示 `JSON5: invalid character`
- **原因：** systemPrompt 裡寫了 `searxng-search "關鍵字"`，`"` 未轉義
- **解法：** 字串內 `"` 改 `\"`，用 `python3 -m json.tool` 驗證
- **預防：** 用 python3 程式寫入 openclaw.json，不用 shell heredoc

### #5 `api` 與 baseUrl 格式
- Ollama native API：`"api": "ollama"`，`baseUrl` **不加 `/v1`**
- 路徑是 `/api/chat`，不是 `/v1/chat/completions`

### #6 SearXNG 不是 OpenClaw 原生 web_search provider
- OpenClaw 原生支援：`brave`、`gemini`、`grok` 等（需 API key）
- SearXNG 整合方式：透過 exec tool 呼叫 wrapper script，**不是** `web_search.provider`

---

## 模型與 Tool Calling 類

### #7 換模型前必須確認 tools capability
- 不支援 tools 的模型：gemma3、phi4、llama3.2-vision
- **確認方式：** `curl -s http://localhost:11434/api/show -d '{"name":"MODEL"}' | python3 -c "import sys,json; print(json.load(sys.stdin).get('capabilities'))"`
- 必須包含 `tools`，否則 OpenClaw 報 `400: does not support tools`

### #8 Tool calling 失效（model 輸出純文字 JSON）
- **原因 A：** 模型沒有 `tools` capability
- **原因 B：** `contextWindow` 設得比模型實際 context_length 大
- **原因 C：** AGENTS.md 沒有明確指示模型用 exec 呼叫什麼指令

### #9 AGENTS.md 是唯一可靠的指令注入點
- `openclaw.json` 的 `systemPrompt` 放在 system prompt 末尾，容易被截斷
- `SKILL.md` 只有 name/description 被引用，**內容不注入**
- **只有 AGENTS.md 的內容完整出現在 LLM 的 system prompt 中**
- 所有關鍵指令（燈控、搜尋、SSH）都要寫在 AGENTS.md

### #10 OpenClaw system prompt 太大導致小模型 tool calling 失敗
- **根因：** OpenClaw 完整 system prompt（tool schemas + AGENTS.md + workspace files）達 **12,000+ tokens**
- **影響：** 3B-9B 模型在此壓力下無法正確決策呼叫哪個 tool
- **表現：** 不呼叫 tool（回文字或無回應）、呼叫錯誤的 tool、回吐文字格式的 tool call
- **解法：** 用 24B+ 模型，或完善 AGENTS.md 讓特定 8B 模型（如 qwen3-vl:8b-instruct）也能穩定運作
- **關鍵發現：** AGENTS.md 的指令品質比模型大小更重要

### #11 Thinking model 速度極慢
- **原因：** 每次生成大量內部推理 token（最多 17000+），且部分版本無法從外部關閉
- **解法：** 用 instruct 變體（如 `qwen3-vl:8b-instruct`），速度快 2-8 倍，保留 vision + tools

### #12 8B model 把 shell 指令名當 tool name
- **現象：** 模型呼叫 `tool_call: tradfri` 而非 `exec(command="tradfri ...")`，導致 `Tool tradfri not found` → 循環 → timeout
- **根因：** 8B 模型分不清 exec tool 和 shell command 的差異
- **解法：** 在 AGENTS.md 寫明確的 exec 指令範例，或用 wrapper script 簡化指令

### #13 14B 模型高 context 時工具呼叫降級為文字格式
- **現象：** cron/exec 輸出 `toolName[ARGS]{...}` 純文字，不執行
- **根因：** context 壓力導致模型降級（tool schemas ~13000 tokens 佔大半 context）
- **解法：** 加大 contextWindow（如 16384 → 32768），或減少 tool schemas

---

## VRAM 與效能類

### #14 contextWindow 設太大 → CPU offload → 速度暴跌
- **原因：** KV cache 大小與 contextWindow 成正比，超過 VRAM 後 Ollama offload 到 CPU
- **規則（16GB VRAM）：**
  - 8B 模型建議 ctx ≤ 49152
  - 14B 模型建議 ctx ≤ 16384
  - 24B 模型建議 ctx ≤ 32768

### #15 Benchmark 必須先停 OpenClaw 再跑
- **原因：** OpenClaw 持續佔用 VRAM，benchmark 數據不準確
- **解法：** 停 OpenClaw + `sudo systemctl restart ollama`（清 VRAM）再跑
- **注意：** 若有 Ollama watchdog，也要先停 timer，否則 watchdog 會自動重啟 Ollama

### #16 vLLM 在 WSL2 + 消費級 GPU 上效能極差
- **實測：** vLLM 單請求 5-11 tok/s vs Ollama 130 tok/s（同 GPU）
- **原因：** WSL2 的 `pin_memory=False` 限制 + GPU driver 翻譯層 overhead
- **結論：** vLLM 適合多人併發的 server 場景（continuous batching），不適合單人 Telegram bot
- **建議：** 單人使用直接用 Ollama

---

## Docker 與網路類

### #17 Docker `network_mode: host` 在 macOS 不能用
- **原因：** macOS Docker 跑在 LinuxKit VM 裡，host mode 只暴露 VM 的網路
- **解法：** 使用 bridge 網路 + `ports` 映射，bridge 可透過 VM NAT 存取 LAN IP

### #18 OpenClaw skill 不能用 symlink
- **現象：** `Skipping skill path that resolves outside its configured root`
- **解法：** `cp -r` 複製實際檔案，不用 symlink

### #19 SearXNG `formats` 缺少 json → 403 Forbidden
- **原因：** `settings.yml` 的 `formats` 只有 `html`，JSON API 被封鎖
- **解法：** 加 `json` 到 formats 清單
- **預防：** pin SearXNG image 版本，避免 auto-update 改變預設行為

---

## Telegram 類

### #20 Telegram bot polling 中斷（`Polling stall detected`）
- **原因：** DNS 解析到不可達的 Telegram DC IP
- **解法：** `/etc/hosts` 固定 `149.154.167.220 api.telegram.org`
- **macOS 持久化：** LaunchDaemon `com.local.telegram-hosts.plist`，RunAtLoad=true
- **備用 IP：** `149.154.175.50`、`149.154.175.100`

---

## Vision 類

### #21 Vision 傳圖沒反應（model 輸出幻覺）
- **原因：** model 定義缺少 `"input": ["text", "image"]`
- **解法：** `openclaw.json` 的 model 定義加此欄位
- **確認：** OpenClaw 用 `model.input?.includes("image")` 決定是否傳圖

---

## macOS 特有

### #22 macOS `/usr/local/bin` 不存在（Apple Silicon）
- **原因：** Apple Silicon macOS 的 Homebrew 路徑是 `/opt/homebrew/bin`
- **解法：** wrapper script 放 `/opt/homebrew/bin/`

### #23 macOS 開機後 /etc/hosts 不持久
- **解法：** LaunchDaemon 開機時冪等寫入

### #24 OrbStack/Docker Desktop 需加入 Login Items
- **原因：** 預設不在 Login Items，重開機後 Docker 不啟動，container 也不跑
- **解法：** 系統偏好設定 → 登入項目 → 加入 OrbStack/Docker Desktop

---

*基於 OpenClaw 2026.3.13 · Ollama 0.17.7 · RTX 5070 Ti 16GB · 2026-03-19 實測*
