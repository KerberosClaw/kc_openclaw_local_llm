# OpenClaw + Local LLM 踩坑紀錄：24 個我們替你踩過的坑

> **English summary:** A comprehensive list of pitfalls encountered while setting up OpenClaw with local Ollama LLMs on a two-machine architecture (Mac + PC with NVIDIA GPU). Covers config issues, tool calling failures, model compatibility, Docker networking, Telegram DNS, VRAM management, and the critical role of AGENTS.md. Based on real-world testing with 13+ models on RTX 5070 Ti 16GB. The document is written in Traditional Chinese.

基於 OpenClaw 2026.3.13 + Ollama 0.17.7 + RTX 5070 Ti 16GB 實測。每一條都是真實的痛。

---

## 設定類（又名「明明照著文件做，為什麼就是不行」）

### #1 `~/.openclaw/` 權限問題（EACCES）——經典中的經典
- **原因：** 哪個天才用 sudo 去編輯設定檔，owner 就變成 root 了，OpenClaw 當然讀不到
- **解法：** `sudo chown $USER ~/.openclaw/openclaw.json`
- **預防：** 用 python3 程式寫入設定檔，永遠不要碰 sudo。我們學到了。

### #2 Pairing required 死結——開不了門的鑰匙
- **解法 A（瀏覽器）：** `openclaw.json` 加 `"dangerouslyDisableDeviceAuth": true`
- **解法 B（macOS native App）：** `openclaw devices list` -> `openclaw devices approve <requestId>`
- **注意：** `dangerouslyDisableDeviceAuth` 只繞過瀏覽器 device auth，不繞過原生 App 的 operator pairing。對，它們是兩回事，我們也很驚訝。

### #3 `/new` 只顯示 heartbeat，main session 去哪了？
- **原因：** `~/.openclaw/workspace/BOOTSTRAP.md` 存在。對，就這麼簡單，但找到這個答案花了我們不少時間。
- **解法：** `rm ~/.openclaw/workspace/BOOTSTRAP.md`

### #4 openclaw.json 字串內有未跳脫的雙引號——隱形殺手
- **現象：** gateway 啟動得很開心但什麼功能都沒有，err log 才默默吐出 `JSON5: invalid character`
- **原因：** systemPrompt 裡寫了 `searxng-search "關鍵字"`，那對 `"` 沒轉義
- **解法：** 字串內 `"` 改 `\"`，用 `python3 -m json.tool` 驗證
- **預防：** 用 python3 程式寫入 openclaw.json，別跟 shell heredoc 搏鬥了，你贏不了的

### #5 `api` 與 baseUrl 格式——看似簡單卻能卡你一小時
- Ollama native API：`"api": "ollama"`，`baseUrl` **不加 `/v1`**
- 路徑是 `/api/chat`，不是 `/v1/chat/completions`

### #6 SearXNG 不是 OpenClaw 原生 web_search provider——驚不驚喜？
- OpenClaw 原生支援：`brave`、`gemini`、`grok` 等（都需要 API key）
- SearXNG 整合方式：透過 exec tool 呼叫 wrapper script，**不是** `web_search.provider`。你沒看錯，它就是一個 workaround。但它能用，而且免費。

---

## 模型與 Tool Calling 類（又名「為什麼我的 AI 突然變笨了」）

### #7 換模型前必須確認 tools capability——否則直接白搭
- 不支援 tools 的模型：gemma3、phi4、llama3.2-vision
- **確認方式：** `curl -s http://localhost:11434/api/show -d '{"name":"MODEL"}' | python3 -c "import sys,json; print(json.load(sys.stdin).get('capabilities'))"`
- 必須包含 `tools`，否則 OpenClaw 報 `400: does not support tools`，然後你會開始懷疑是不是自己設定有問題（不是，是模型不行）

### #8 Tool calling 失效（model 輸出純文字 JSON）——三重陷阱
- **原因 A：** 模型沒有 `tools` capability（最常見，也最容易被忽略）
- **原因 B：** `contextWindow` 設得比模型實際 context_length 大
- **原因 C：** AGENTS.md 沒有明確指示模型用 exec 呼叫什麼指令

### #9 AGENTS.md 是唯一可靠的指令注入點——這個很重要，畫重點
- `openclaw.json` 的 `systemPrompt` 放在 system prompt 末尾，容易被截斷
- `SKILL.md` 只有 name/description 被引用，**內容不注入**
- **只有 AGENTS.md 的內容完整出現在 LLM 的 system prompt 中**
- 所有關鍵指令（燈控、搜尋、SSH）都要寫在 AGENTS.md。我們是在反覆除錯之後才搞懂這件事的，希望你不用重蹈覆轍。

### #10 OpenClaw system prompt 太大導致小模型 tool calling 失敗——模型被壓垮了
- **根因：** OpenClaw 完整 system prompt（tool schemas + AGENTS.md + workspace files）達 **12,000+ tokens**
- **影響：** 3B-9B 模型在此壓力下無法正確決策呼叫哪個 tool，就像塞了一整本操作手冊進小學生的腦袋
- **表現：** 不呼叫 tool（回文字或無回應）、呼叫錯誤的 tool、回吐文字格式的 tool call
- **解法：** 用 24B+ 模型，或完善 AGENTS.md 讓特定 8B 模型（如 qwen3-vl:8b-instruct）也能穩定運作
- **關鍵發現：** AGENTS.md 的指令品質比模型大小更重要。寫好指令比買好顯卡便宜。

### #11 Thinking model 速度極慢——帳面數字在騙你
- **原因：** 每次生成大量內部推理 token（最多 17000+），而且部分版本無法從外部關閉這個行為
- **解法：** 用 instruct 變體（如 `qwen3-vl:8b-instruct`），速度快 2-8 倍，保留 vision + tools。tok/s 看起來差不多，但實際等待時間天差地遠。

### #12 8B model 把 shell 指令名當 tool name——一個讓人啼笑皆非的 bug
- **現象：** 模型呼叫 `tool_call: tradfri` 而非 `exec(command="tradfri ...")`，導致 `Tool tradfri not found` -> 循環 -> timeout
- **根因：** 8B 模型分不清 exec tool 和 shell command 的差異。它看到「tradfri」就覺得那是個 tool，殊不知它只是個指令名。
- **解法：** 在 AGENTS.md 寫明確的 exec 指令範例，或用 wrapper script 簡化指令。手把手教它，它就會了。

### #13 14B 模型高 context 時工具呼叫降級為文字格式——漸進式崩潰
- **現象：** cron/exec 輸出 `toolName[ARGS]{...}` 純文字，不執行
- **根因：** context 壓力導致模型降級（tool schemas ~13000 tokens 佔大半 context），模型開始「湊合著來」
- **解法：** 加大 contextWindow（如 16384 -> 32768），或減少 tool schemas

---

## VRAM 與效能類（又名「為什麼突然變這麼慢」）

### #14 contextWindow 設太大 -> CPU offload -> 速度暴跌——溫水煮青蛙
- **原因：** KV cache 大小與 contextWindow 成正比，超過 VRAM 後 Ollama 默默 offload 到 CPU，然後你就等吧
- **規則（16GB VRAM）：**
  - 8B 模型建議 ctx <= 49152
  - 14B 模型建議 ctx <= 16384
  - 24B 模型建議 ctx <= 32768

### #15 Benchmark 必須先停 OpenClaw 再跑——你的數據可能是假的
- **原因：** OpenClaw 持續佔用 VRAM，benchmark 在搶剩下的資源，數據當然不準確
- **解法：** 停 OpenClaw + `sudo systemctl restart ollama`（清 VRAM）再跑
- **注意：** 若有 Ollama watchdog，也要先停 timer，否則 watchdog 會自動重啟 Ollama，然後你又白做了

### #16 vLLM 在 WSL2 + 消費級 GPU 上效能極差——理想很豐滿，現實很骨感
- **實測：** vLLM 單請求 5-11 tok/s vs Ollama 130 tok/s（同 GPU）。不是打錯字，是真的差這麼多。
- **原因：** WSL2 的 `pin_memory=False` 限制 + GPU driver 翻譯層 overhead
- **結論：** vLLM 適合多人併發的 server 場景（continuous batching），不適合單人 Telegram bot
- **建議：** 單人使用直接用 Ollama，別折騰了

---

## Docker 與網路類（又名「為什麼在 Mac 上跟在 Linux 上不一樣」）

### #17 Docker `network_mode: host` 在 macOS 不能用——又一個平台差異
- **原因：** macOS Docker 跑在 LinuxKit VM 裡，host mode 只暴露 VM 的網路，不是你 Mac 的網路
- **解法：** 使用 bridge 網路 + `ports` 映射，bridge 可透過 VM NAT 存取 LAN IP

### #18 OpenClaw skill 不能用 symlink——懶人稅
- **現象：** `Skipping skill path that resolves outside its configured root`
- **解法：** `cp -r` 老老實實複製實際檔案，不用 symlink。對，就是這麼不方便。

### #19 SearXNG `formats` 缺少 json -> 403 Forbidden——一個欄位毀一天
- **原因：** `settings.yml` 的 `formats` 只有 `html`，JSON API 被封鎖
- **解法：** 加 `json` 到 formats 清單
- **預防：** pin SearXNG image 版本，避免 auto-update 偷偷改變預設行為，然後你又要花一下午除錯

---

## Telegram 類（又名「為什麼 bot 又斷了」）

### #20 Telegram bot polling 中斷（`Polling stall detected`）——DNS 的鍋
- **原因：** DNS 解析到不可達的 Telegram DC IP
- **解法：** `/etc/hosts` 固定 `149.154.167.220 api.telegram.org`
- **macOS 持久化：** LaunchDaemon `com.local.telegram-hosts.plist`，RunAtLoad=true
- **備用 IP：** `149.154.175.50`、`149.154.175.100`（多存幾個以防萬一）

---

## Vision 類（又名「我明明傳了圖，它為什麼在胡說」）

### #21 Vision 傳圖沒反應（model 輸出幻覺）——少一個設定差很多
- **原因：** model 定義缺少 `"input": ["text", "image"]`
- **解法：** `openclaw.json` 的 model 定義加此欄位
- **確認：** OpenClaw 用 `model.input?.includes("image")` 決定是否傳圖。沒這個欄位，它壓根不會把圖片送給模型，然後模型就開始自由發揮了。

---

## macOS 特有（又名「為什麼 Apple 要跟大家不一樣」）

### #22 macOS `/usr/local/bin` 不存在（Apple Silicon）——歡迎來到 ARM 世界
- **原因：** Apple Silicon macOS 的 Homebrew 路徑是 `/opt/homebrew/bin`，跟你看到的所有舊教程都不一樣
- **解法：** wrapper script 放 `/opt/homebrew/bin/`

### #23 macOS 開機後 /etc/hosts 不持久——系統就是要跟你作對
- **解法：** LaunchDaemon 開機時冪等寫入。對，每次開機都要重來一次。

### #24 OrbStack/Docker Desktop 需加入 Login Items——別忘了這步
- **原因：** 預設不在 Login Items，重開機後 Docker 不啟動，container 也不跑，然後你花半小時找「為什麼搜尋壞了」
- **解法：** 系統偏好設定 -> 登入項目 -> 加入 OrbStack/Docker Desktop

---

*基於 OpenClaw 2026.3.13 -- Ollama 0.17.7 -- RTX 5070 Ti 16GB -- 2026-03-19 實測*
