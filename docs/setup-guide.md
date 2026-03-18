# 本地 LLM + OpenClaw 兩機架構完整架設教程

> **English summary:** Complete setup guide for running OpenClaw with local Ollama LLMs on a two-machine architecture (Mac client + Linux/WSL2 PC with NVIDIA GPU). Includes: automated Claude Code installation prompts (Prompt A for PC, Prompt B for Mac), manual step-by-step instructions, model selection guide with 13 models tested, AGENTS.md configuration (critical for tool calling stability), SearXNG local search integration, Telegram bot setup, and 16 common pitfalls. Tested on OpenClaw 2026.3.13, Ollama 0.17.7, RTX 5070 Ti 16GB. The document is written in Traditional Chinese.

> 適用對象：想在自己的 Linux 機器（PC）上跑本地 AI 推理引擎，再透過 Mac 上的 OpenClaw 統一管理 Telegram bot、網頁 UI、視覺辨識，不依賴任何外部 API key。
>
> 本教程基於 Linux（Ubuntu）+ NVIDIA GPU + macOS（Apple Silicon），2026-03-19 版本。
> 測試版本：OpenClaw 2026.3.13 · Ollama 0.17.7 · RTX 5070 Ti 16GB

---

## 架構概覽

```
Telegram / 瀏覽器
  | HTTP / WebSocket
OpenClaw（Mac，npm global）— Telegram bot、Agent UI
  | Ollama native API（http://PC_LAN_IP:11434）
Ollama（PC Ubuntu systemd service）
  |-- qwen3-vl:8b-instruct（推薦，vision + tools，無 thinking）
  | http://PC_LAN_IP:8080
SearXNG（PC Docker container）— 本地搜尋，不需 API key
```

### 硬體需求

| 項目 | 最低需求 | 說明 |
|------|---------|------|
| PC GPU VRAM | 8 GB（6 GB 勉強） | 決定能跑的模型大小 |
| PC RAM | 16 GB | 建議 32 GB+ |
| PC 儲存 | 30 GB+ | 模型檔案 |
| PC OS | Ubuntu 20.04+ | 需要 systemd + Docker |
| Mac | Apple Silicon / Intel | macOS 12+ |

### 服務清單

| 服務 | 運行位置 | 用途 |
|------|---------|------|
| Ollama | PC Linux systemd | LLM 推理引擎 |
| SearXNG | PC Docker 容器 | 本地搜尋引擎 |
| OpenClaw | Mac npm global | AI agent core、Telegram bot、Web UI |

---

## 模型選擇指南（先讀這段！）

> **這是本教程最重要的段落。** 選錯模型 = 浪費時間。我們測了 13 個模型，只有 2 個能在 OpenClaw 環境下穩定使用。

### 推薦模型

| 模型 | VRAM | 速度 | 推薦原因 |
|------|------|------|---------|
| **`qwen3-vl:8b-instruct`** | ~6 GB | 130 tok/s | **首選。** Tool calling 穩定、速度快、支援 vision、VRAM 省 |

### 為什麼大部分模型都不能用？

OpenClaw 的完整 system prompt（含 tool schemas + workspace files）達 **12,000+ tokens**。這對模型的 tool calling 能力是嚴峻的考驗：

- **小模型（3B-9B）** 的「注意力」被巨大的 system prompt 佔滿，無法正確判斷應該呼叫哪個 tool
- **部分模型不支援 Ollama tools API**（如 gemma3、phi4），直接報錯
- **部分模型格式不對**（如 llama 系列），只回吐文字而不是真正呼叫 tool

**關鍵發現：AGENTS.md 的指令品質比模型大小更重要。** `qwen3-vl:8b-instruct` 之前也曾失敗（tool name 錯誤），但在完善 AGENTS.md 的指令後成功穩定運作。詳見下方「AGENTS.md 設定」段落。

### 完整模型實測結果（RTX 5070 Ti 16GB，OpenClaw 2026.3.13）

#### OpenClaw Tool Calling 實測

所有模型在相同環境下測試：12K+ token system prompt、搜尋（exec + searxng-search）、燈控（exec + tradfri）、複合操作。

| 模型 | 大小 | 搜尋 | 單一操作 | 複合操作 | 結論 |
|------|------|------|----------|----------|------|
| **qwen3-vl:8b-instruct** | 6.1 GB | ✅ | ✅ | ✅ | **主力，全通過** |
| mistral-small3.1:24b | 15 GB | ✅ | ✅ | ✅ | 穩定但太慢（~10 tok/s @32K） |
| qwen2.5:14b-instruct | 9.0 GB | ✅ | ⚠️ | ⚠️ | 能用但囉嗦猶豫，舊世代不如 qwen3 |
| qwen3:4b | 2.5 GB | ✅ | ✅ | ❌ | 單指令 OK，複合崩 |
| qwen3:4b-instruct | 2.5 GB | ❌ | ❌ | — | 無回應 |
| frob/qwen3.5-instruct:9b | 6.6 GB | ❌ | ❌ | — | 不呼叫 exec |
| gpt-oss:20b | 13 GB | ❌ | ✅/❌ | — | 極不穩定 |
| orieg/gemma3-tools:12b-it-qat | 8.9 GB | ❌ | ❌ | — | 吐文字不呼叫 tool |
| MFDoom/deepseek-r1-tool-calling:14b | 9 GB | ❌ | — | — | 吐 JSON 文字 |
| phi4:14b | 9.1 GB | — | — | — | 不支援 tools API |
| llama3.2-vision:11b | 7.8 GB | — | — | — | 不支援 tools API |
| llama3.1:8b | 4.9 GB | ❌ | ❌ | — | 只回吐文字 |
| llama3.2:3b | 2 GB | ❌ | ❌ | — | 無回應 |

> **qwen3-vl 沒有中間尺寸：** 8B 之上直接跳到 30B-A3B（Q4_K_M 需 20GB），16GB VRAM 放不下。8B 就是 16GB 顯卡的天花板。

#### Benchmark（qwen3-vl:8b-instruct，Ollama，RTX 5070 Ti）

| Context Window | 平均 tok/s | TTFT | VRAM |
|----------------|-----------|------|------|
| 2048 | 108 | ~0.03s | ~6 GB |
| 4096 | 111 | ~0.02s | ~6 GB |
| 8192 | 111 | ~0.02s | ~6 GB |
| 16384 | 129 | ~0.03s | ~7 GB |
| **32768** | **131** | **~0.02s** | **~8 GB** |

> 8B 模型只用 6-8 GB VRAM，即使 32K ctx 速度也不掉（16GB 顯卡有大量餘裕）。

#### Benchmark（qwen2.5:14b-instruct，Ollama，RTX 5070 Ti）

| Context Window | 平均 tok/s | TTFT | VRAM |
|----------------|-----------|------|------|
| 2048 | 79 | ~0.04s | ~9 GB |
| 4096 | 79 | ~0.04s | ~9.5 GB |
| 8192 | 79 | ~0.04s | ~10 GB |
| 16384 | 79 | ~0.04s | ~11 GB |
| **32768** | **19** ⚠️ | **0.14s** | **~15 GB** |

> 14B 模型在 ctx ≤ 16384 時穩定 79 tok/s，但 32K ctx 暴跌到 19 tok/s（VRAM 接近滿載）。OpenClaw 最低要求 ctx 16000，所以只能設 16384，速度比 qwen3-vl:8b 的 131 tok/s 慢 40%。加上沒有 vision、tool calling 品質較差（囉嗦猶豫），不推薦。

### 推理引擎比較：Ollama vs vLLM

> 我們也測試了 vLLM 作為替代推理引擎的可能性。結論：**單人使用場景下 Ollama 完勝。**

| 指標 | Ollama | vLLM |
|------|--------|------|
| 單請求 tok/s | **130** | 5-11 |
| 安裝難度 | `curl | sh` 一行 | 需 Docker build（Blackwell 需特殊 image） |
| WSL2 支援 | 原生，開箱即用 | 有 `pin_memory` 限制，效能大打折扣 |
| 多人併發 | 排隊處理 | PagedAttention 批次處理（優勢場景） |

vLLM 的核心優勢是 **continuous batching**（多人併發時效率遠超 Ollama）。但單人 Telegram bot 場景永遠只有一個請求在跑，vLLM 毫無優勢，反而因為 WSL2 的 overhead 更慢。

> 測試環境：RTX 5070 Ti 16GB，WSL2 Ubuntu 22.04，vLLM 0.17.2rc1（Docker，Qwen2.5-VL-7B-Instruct-AWQ），Ollama 0.17.7（qwen3-vl:8b-instruct Q4_K_M）

---

## 前置準備（手動，約 5 分鐘）

在跑自動安裝 prompt 前，需先完成以下步驟。Claude Code 之後才能無人值守遠端設定 PC。

### 步驟一：PC 設定 sudo 免密碼

在 PC 上執行（換成你的使用者名稱）：

```bash
echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER
sudo chmod 440 /etc/sudoers.d/$USER
```

### 步驟二：Mac → PC SSH 金鑰設定

在 Mac 上執行：

```bash
# 產生 ed25519 金鑰（若已有可跳過）
ssh-keygen -t ed25519 -C "mac-to-pc"

# 把公鑰推送到 PC（換成你的 PC IP 和使用者名稱）
ssh-copy-id -i ~/.ssh/id_ed25519.pub YOUR_PC_USER@PC_LAN_IP

# 測試連線
ssh -i ~/.ssh/id_ed25519 YOUR_PC_USER@PC_LAN_IP echo "SSH OK"
```

### 步驟三：Mac 安裝 Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

> 以上三步完成後，後續所有安裝都可以從 Mac 上讓 Claude Code 代勞。

---

## 快速安裝：讓 Claude Code 代勞

### Prompt A：設定 PC（Inference 機）

> 在 Mac 開一個新的 Claude Code session，把以下 prompt 貼進去。Claude Code 會透過 SSH 遠端設定 PC。完成後確認驗證報告全部 ✅，再執行 Prompt B。

<details>
<summary>展開 Prompt A（PC 設定）</summary>

```
你是本地 LLM 基礎架構安裝助手，任務是透過 SSH 在 Linux PC 上安裝 Ollama + SearXNG。全程使用繁體中文溝通。

## 安裝前收集資訊

先問使用者（一次問清楚）：
1. PC 的 LAN IP（例如 192.168.1.100）
2. PC 的 SSH 使用者名稱
3. SSH port（預設 22，若有改請告知）

取得資訊後，先執行以下指令確認環境，顯示給使用者看：

ssh -i ~/.ssh/id_ed25519 -p SSH_PORT PC_USER@PC_IP "uname -a && nvidia-smi -L 2>/dev/null || echo 'no NVIDIA GPU detected'"

確認環境正常後開始安裝。

---

## 安裝步驟

以下所有 SSH 指令格式：ssh -i ~/.ssh/id_ed25519 -p SSH_PORT PC_USER@PC_IP "指令"
每個步驟安裝完後先測試，顯示結果再繼續。

### 1. Docker

ssh ... "if ! command -v docker &>/dev/null; then
  curl -fsSL https://get.docker.com | sh
  sudo usermod -aG docker \$USER
  echo 'Docker installed'
else
  echo \"Docker already: \$(docker --version)\"
fi"

測試：ssh ... "docker --version" → 顯示版本號

### 2. NVIDIA Container Toolkit

ssh ... "if ! dpkg -l | grep -q nvidia-container-toolkit; then
  curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
  curl -sL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
  sudo apt-get update -qq && sudo apt-get install -y nvidia-container-toolkit
  sudo nvidia-ctk runtime configure --runtime=docker
  sudo systemctl restart docker
  echo 'NVIDIA Container Toolkit installed'
else
  echo 'Already installed'
fi"

測試：ssh ... "docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu22.04 nvidia-smi | head -4 || echo 'GPU test failed'"

### 3. Ollama

ssh ... "if ! command -v ollama &>/dev/null; then curl -fsSL https://ollama.com/install.sh | sh; fi
sudo mkdir -p /etc/systemd/system/ollama.service.d
echo '[Service]
Environment=\"OLLAMA_HOST=0.0.0.0\"' | sudo tee /etc/systemd/system/ollama.service.d/override.conf
sudo systemctl daemon-reload && sudo systemctl enable ollama && sudo systemctl restart ollama
sleep 3
curl -s http://localhost:11434/api/version && echo 'Ollama OK'"

### 4. 下載模型並確認 capabilities

ssh ... "ollama pull qwen3-vl:8b-instruct"

ssh ... "curl -s http://localhost:11434/api/show -d '{\"name\":\"qwen3-vl:8b-instruct\"}' | python3 -c \"import sys,json; caps=json.load(sys.stdin).get('capabilities'); print('capabilities:', caps); print('tools OK' if 'tools' in (caps or []) else '錯誤：此模型沒有 tools capability，無法使用 OpenClaw')\""

### 5. SearXNG

ssh ... "mkdir -p ~/services/searxng
cat > ~/services/searxng/settings.yml << 'YAML'
use_default_settings: true
server:
  secret_key: \"\$(openssl rand -hex 32)\"
  limiter: false
search:
  safe_search: 0
  formats: [html, json]
engines:
  - name: brave
    engine: brave
    shortcut: br
    disabled: false
  - name: duckduckgo
    engine: duckduckgo
    shortcut: ddg
    disabled: false
  - name: mwmbl
    engine: mwmbl
    shortcut: mwm
    disabled: false
YAML

cat > ~/services/docker-compose.yml << 'YAML'
services:
  searxng:
    image: searxng/searxng:latest
    ports:
      - \"0.0.0.0:8080:8080\"
    volumes:
      - ./searxng/settings.yml:/etc/searxng/settings.yml:ro
    restart: unless-stopped
YAML

cd ~/services && docker compose up -d"

測試：ssh ... "sleep 3 && curl -s 'http://localhost:8080/search?q=test&format=json' | python3 -c \"import sys,json; d=json.load(sys.stdin); print(f'SearXNG OK: {len(d.get(\\\"results\\\",[]))} results')\""

### 6. 防火牆

ssh ... "sudo ufw allow 11434/tcp comment 'Ollama API' && sudo ufw allow 8080/tcp comment 'SearXNG' && echo 'Firewall OK'"

---

## 最終驗證報告

執行完所有步驟後，輸出以下格式的報告：

=== PC 安裝驗證報告 ===
Docker:          ✅ / ❌
NVIDIA Toolkit:  ✅ / ❌
Ollama:          ✅ 版本 X.X.X / ❌
模型 capabilities: ✅ tools 確認 / ❌
SearXNG:         ✅ 回傳 N 筆結果 / ❌
防火牆:          ✅ / ❌
Ollama URL:      http://PC_IP:11434
SearXNG URL:     http://PC_IP:8080
======================
PC 設定完成。請確認以上所有項目為 ✅ 後，再執行 Prompt B 設定 Mac。
```

</details>

---

### Prompt B：設定 Mac（Client 機）

> 確認 Prompt A 驗證報告全部 ✅ 後，在 Mac 開新的 Claude Code session，把以下 prompt 貼進去。

<details>
<summary>展開 Prompt B（Mac 設定）</summary>

```
你是 OpenClaw Mac 安裝助手，任務是在這台 Mac 上安裝並設定 OpenClaw。全程使用繁體中文溝通。

## 安裝前收集資訊

先問使用者（一次問清楚）：
1. PC 的 LAN IP（Prompt A 報告中的 Ollama URL IP）
2. PC 的 SSH 使用者名稱
3. 要串接 Telegram bot 嗎？（是/否）若是，請提供：
   - Bot Token（從 @BotFather 取得）
   - 你的 Telegram User ID（從 @userinfobot 取得）

然後在本機執行取得必要資訊：
- whoami 取得 Mac 使用者名稱
- openssl rand -hex 24 產生 GATEWAY_TOKEN（記錄備用）

---

## 安裝步驟

每個步驟完成後測試並顯示結果再繼續。

### 1. nvm + Node.js + OpenClaw

bash -c '
if [ ! -d ~/.nvm ]; then
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
fi
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"
nvm install --lts
npm install -g openclaw
openclaw --version'

### 2. uv

if ! command -v uv &>/dev/null && [ ! -f ~/.local/bin/uv ]; then
  curl -LsSf https://astral.sh/uv/install.sh | sh
fi
~/.local/bin/uv --version || uv --version

### 3. clawhub + SearXNG skill

export NVM_DIR="$HOME/.nvm" && source "$NVM_DIR/nvm.sh"
npm install -g clawhub
clawhub install abk234/searxng

### 4. searxng-search wrapper script

注意：macOS Apple Silicon 用 /opt/homebrew/bin，不是 /usr/local/bin

cat > /opt/homebrew/bin/searxng-search << 'SCRIPT'
#!/bin/bash
SEARXNG_URL=http://PC_LAN_IP:8080 ~/.local/bin/uv run ~/.openclaw/workspace/skills/searxng/scripts/searxng.py search "$@"
SCRIPT
chmod +x /opt/homebrew/bin/searxng-search

（PC_LAN_IP 替換為使用者提供的 IP）

測試：searxng-search "test" → 應有搜尋結果輸出

### 5. openclaw.json

mkdir -p ~/.openclaw

用 GATEWAY_TOKEN 和 PC_LAN_IP 變數，以 python3 寫入（避免 heredoc shell 變數展開問題）：

python3 << PYEOF
import json, os

GATEWAY_TOKEN = "從 openssl rand -hex 24 取得的值"
PC_LAN_IP = "使用者提供的 PC IP"

config = {
    "agents": {
        "defaults": {
            "model": "ollama/qwen3-vl:8b-instruct",
            "compaction": {"mode": "safeguard"},
            "timeoutSeconds": 120
        }
    },
    "tools": {
        "exec": {"host": "gateway", "security": "full", "ask": "off"},
        "web": {"search": {"enabled": False}},
        "deny": []
    },
    "commands": {
        "native": "auto",
        "nativeSkills": "auto",
        "restart": True,
        "ownerDisplay": "raw"
    },
    "gateway": {
        "mode": "local",
        "trustedProxies": [],
        "http": {"endpoints": {"chatCompletions": {"enabled": True}}},
        "controlUi": {
            "dangerouslyDisableDeviceAuth": True,
            "allowedOrigins": [
                "http://localhost:18789",
                "https://localhost:18443"
            ]
        },
        "auth": {"mode": "token", "token": GATEWAY_TOKEN}
    },
    "models": {
        "providers": {
            "ollama": {
                "baseUrl": f"http://{PC_LAN_IP}:11434",
                "apiKey": "ollama-local",
                "api": "ollama",
                "models": [
                    {
                        "id": "qwen3-vl:8b-instruct",
                        "name": "Qwen3-VL 8B Instruct",
                        "contextWindow": 32768,
                        "input": ["text", "image"]
                    }
                ]
            }
        }
    }
}

path = os.path.expanduser("~/.openclaw/openclaw.json")
json.dump(config, open(path, "w"), ensure_ascii=False, indent=2)
print("openclaw.json 建立完成")
PYEOF

若使用者要串接 Telegram，再注入：

python3 << PYEOF
import json, os

BOT_TOKEN = "使用者提供的 bot token"
USER_ID = "使用者提供的 user id"
PC_LAN_IP = "使用者提供的 PC IP"

path = os.path.expanduser("~/.openclaw/openclaw.json")
data = json.load(open(path))

data["channels"] = {
    "telegram": {
        "enabled": True,
        "botToken": BOT_TOKEN,
        "dmPolicy": "allowlist",
        "allowFrom": [USER_ID],
        "direct": {
            USER_ID: {
                "systemPrompt": (
                    f"你是運行在 Mac 的 AI 助手，模型於 PC（{PC_LAN_IP}，NVIDIA GPU）推理。時區：Asia/Taipei。\n\n"
                    "規則：\n"
                    "1. 永遠使用繁體中文回答。\n"
                    "2. 需要執行指令時，直接用 exec，不問確認。\n"
                    "3. 搜尋用 exec 執行 searxng-search \\\"關鍵字\\\"，不得捏造。\n"
                    "4. 不客套，直接做。"
                )
            }
        }
    }
}

json.dump(data, open(path, "w"), ensure_ascii=False, indent=2)
print("Telegram 設定注入完成")
PYEOF

### 6. AGENTS.md（關鍵！Tool calling 穩定的核心）

建立 ~/.openclaw/workspace/AGENTS.md，這是 OpenClaw 唯一可靠的指令注入點。
systemPrompt 和 SKILL.md 的內容不會被完整注入到 LLM context，只有 AGENTS.md 會。

cat > ~/.openclaw/workspace/AGENTS.md << 'EOF'
# AGENTS.md

## Heartbeat

收到 heartbeat / 初始化訊息 → 回覆 `HEARTBEAT_OK`，不做任何 tool call。

## Session Startup

簡短問題直接回答。

## Memory

- 日記：`memory/YYYY-MM-DD.md`
- 長期：`MEMORY.md`

## Red Lines

- 不洩漏私人資料
- 破壞性指令先問
- `trash` > `rm`
- **不改 `~/.openclaw/openclaw.json`**

## Web Search

`web_search` 沒有 API key，禁止使用。改用 exec：

```bash
searxng-search "查詢關鍵字"
```

查細節：先搜尋找 URL → `web_fetch` 抓全文 → 據實回答，不得捏造。
EOF

> **重要：** 這個檔案是 tool calling 穩定的關鍵。不寫或寫得不好，8B 模型就會亂呼叫 tool。
> 如果你有自訂的 exec 指令（如智慧家居控制、SSH 到 PC），也要寫在這裡。

### 7. Telegram DNS 修復（macOS 開機持久）

只有使用 Telegram bot 的使用者需要此步驟。

ping -c 2 api.telegram.org 先確認連線狀況，顯示結果給使用者。

若有 packet loss 或 timeout，執行以下設定：

sudo tee /Library/LaunchDaemons/com.local.telegram-hosts.plist << 'XML'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.local.telegram-hosts</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string><string>-c</string>
    <string>grep -q 'api.telegram.org' /etc/hosts || echo '149.154.167.220 api.telegram.org' >> /etc/hosts</string>
  </array>
  <key>RunAtLoad</key><true/>
</dict>
</plist>
XML

sudo launchctl load /Library/LaunchDaemons/com.local.telegram-hosts.plist
grep -q 'api.telegram.org' /etc/hosts || echo '149.154.167.220 api.telegram.org' | sudo tee -a /etc/hosts

---

## 最終驗證報告

輸出以下格式的報告：

=== Mac 安裝驗證報告 ===
OpenClaw:         ✅ 版本 X.X.X / ❌
uv:               ✅ / ❌
searxng skill:    ✅ / ❌
searxng-search:   ✅ 回傳 N 筆結果 / ❌
openclaw.json:    ✅ JSON 格式正確 / ❌
AGENTS.md:        ✅ 已建立 / ❌
Ollama 連線:      ✅ http://PC_IP:11434 可達 / ❌
Telegram DNS:     ✅ api.telegram.org 可達 / ❌（未設定）
========================

然後提示使用者：

Mac 設定完成！接下來需要你手動執行：

1. 啟動 OpenClaw（開一個 terminal 保持執行）：
   source ~/.nvm/nvm.sh && openclaw gateway --bind lan --port 18789 --allow-unconfigured

2. 若出現 "pairing required"，開新 terminal 執行：
   source ~/.nvm/nvm.sh && openclaw devices list
   source ~/.nvm/nvm.sh && openclaw devices approve <requestId>

3. 首次 onboard（若提示需要）：
   source ~/.nvm/nvm.sh && openclaw onboard

完成後開啟瀏覽器：http://localhost:18789
```

</details>

---

## 手動安裝（逐步版）

<details>
<summary>展開 Part A：PC（Inference 機）</summary>

### 1. Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 重新登入使 group 生效
```

### 2. NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -sL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 確認
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu22.04 nvidia-smi
```

### 3. Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh

# 讓其他機器可透過區網存取
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
EOF

sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl restart ollama

# 確認
curl http://localhost:11434/api/version
```

### 4. 下載模型

```bash
ollama pull qwen3-vl:8b-instruct

# 確認 capabilities（必須有 tools）
curl -s http://localhost:11434/api/show -d '{"name":"qwen3-vl:8b-instruct"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('capabilities'))"
# 期望輸出：['completion', 'vision', 'tools']
```

### 5. SearXNG

```bash
mkdir -p ~/services/searxng

cat > ~/services/searxng/settings.yml << EOF
use_default_settings: true

server:
  secret_key: "$(openssl rand -hex 32)"
  limiter: false

search:
  safe_search: 0
  formats:
    - html
    - json

engines:
  - name: brave
    engine: brave
    shortcut: br
    disabled: false
  - name: duckduckgo
    engine: duckduckgo
    shortcut: ddg
    disabled: false
  - name: mwmbl
    engine: mwmbl
    shortcut: mwm
    disabled: false
EOF

cat > ~/services/docker-compose.yml << 'EOF'
services:
  searxng:
    image: searxng/searxng:latest
    ports:
      - "0.0.0.0:8080:8080"
    volumes:
      - ./searxng/settings.yml:/etc/searxng/settings.yml:ro
    restart: unless-stopped
EOF

cd ~/services && docker compose up -d

# 確認
curl -s 'http://localhost:8080/search?q=test&format=json' \
  | python3 -c "import sys,json; print(len(json.load(sys.stdin).get('results',[])), 'results')"
```

### 6. 防火牆

```bash
sudo ufw allow 11434/tcp comment "Ollama API"
sudo ufw allow 8080/tcp comment "SearXNG"
```

</details>

<details>
<summary>展開 Part B：Mac（Client 機）</summary>

### 1. nvm + Node.js + OpenClaw

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.nvm/nvm.sh
nvm install --lts
npm install -g openclaw
openclaw --version
```

### 2. uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 3. clawhub + SearXNG skill

```bash
npm install -g clawhub
clawhub install abk234/searxng
```

### 4. searxng-search wrapper script

```bash
cat > /opt/homebrew/bin/searxng-search << 'EOF'
#!/bin/bash
SEARXNG_URL=http://PC_LAN_IP:8080 ~/.local/bin/uv run ~/.openclaw/workspace/skills/searxng/scripts/searxng.py search "$@"
EOF
chmod +x /opt/homebrew/bin/searxng-search
```

### 5. openclaw.json

見 Prompt B 步驟 5 的 python3 範例。

> **重要設定值：**
> - `api` 必須設 `"ollama"`，`baseUrl` **不加 `/v1`**
> - `"input": ["text", "image"]` 啟用 vision 的必要欄位
> - `contextWindow: 32768`（8B 模型在 16GB VRAM 下的安全值）
> - `tools.deny: []`（全開，讓模型自己決定用哪個 tool）

### 6. AGENTS.md

見 Prompt B 步驟 6。**這是 tool calling 穩定的關鍵，務必建立。**

### 7. 啟動 OpenClaw

```bash
source ~/.nvm/nvm.sh
openclaw gateway --bind lan --port 18789 --allow-unconfigured
```

</details>

---

## 快速驗證清單

架設完成後依序確認：

- [ ] PC：`curl http://localhost:11434/api/version` → 有回應
- [ ] PC：`docker ps` → searxng 容器 Up
- [ ] Mac：`searxng-search "test"` → 有搜尋結果
- [ ] Mac：OpenClaw 啟動無報錯
- [ ] 瀏覽器開 `http://localhost:18789` → 進入 OpenClaw UI
- [ ] 在 UI 問「你好」→ 有回應（確認 Ollama 連線正常）
- [ ] 在 UI 問「幫我搜尋今天的新聞」→ model 用 searxng-search 搜尋並回報結果
- [ ] （若有串 Telegram）傳圖片給 bot → model 正確描述圖片內容

---

## 踩過的坑

### #1：`~/.openclaw/` 權限問題（EACCES）
- **原因：** 以 sudo 或 root 編輯設定檔，owner 變成 root，OpenClaw 讀不到
- **解法：** `sudo chown $USER ~/.openclaw/openclaw.json`

### #2：Pairing required 死結
- **解法 A（瀏覽器）：** `openclaw.json` 的 `controlUi` 段加 `"dangerouslyDisableDeviceAuth": true`
- **解法 B（macOS native App）：** `openclaw devices list` → `openclaw devices approve <requestId>`

### #3：`/new` 只顯示 heartbeat，main session 不出現
- **原因：** `~/.openclaw/workspace/BOOTSTRAP.md` 存在
- **解法：** `rm ~/.openclaw/workspace/BOOTSTRAP.md`

### #4：macOS `/usr/local/bin` 不存在（Apple Silicon）
- **解法：** wrapper script 放 `/opt/homebrew/bin/`

### #5：Telegram bot polling 中斷
- **原因：** DNS 解析到不可達的 Telegram DC IP
- **解法：** `/etc/hosts` 固定 `149.154.167.220 api.telegram.org`，用 LaunchDaemon 開機持久化

### #6：Vision 傳圖沒反應
- **原因：** model 定義缺少 `"input": ["text", "image"]`
- **解法：** `openclaw.json` 的 model 定義加此欄位

### #7：Tool calling 失效（model 輸出純文字 JSON）
- **原因 A：** 模型沒有 `tools` capability（gemma3、phi4 等）
- **原因 B：** `contextWindow` 設得比模型實際值大
- **原因 C：** AGENTS.md 沒有明確指示模型用 exec 呼叫什麼指令
- **確認：** `ollama show MODEL --verbose | grep context_length`

### #8：contextWindow 設太大 → CPU offload → 速度暴跌
- **規則：** 8B 模型 16 GB VRAM 建議 ctx ≤ 49152；14B 模型建議 ctx ≤ 16384

### #9：Thinking model 速度極慢
- **原因：** 每次生成大量內部推理 token（最多 17000+）
- **解法：** 用 instruct 變體（如 `qwen3-vl:8b-instruct`），速度快 2-8 倍

### #10：`api` 與 baseUrl 格式
- Ollama native API：`"api": "ollama"`，`baseUrl` **不加 `/v1`**

### #11：AGENTS.md 是唯一可靠的指令注入點
- `openclaw.json` 的 `systemPrompt` 放在 system prompt 末尾，容易被截斷
- `SKILL.md` 只有 name/description 被引用，內容不注入
- **只有 AGENTS.md 的內容完整出現在 LLM 的 system prompt 中**

### #12：openclaw.json 字串內有未跳脫的雙引號
- **現象：** gateway 啟動但無任何功能，log 顯示 "Config invalid"
- **解法：** 字串內 `"` 改 `\"`，用 `python3 -m json.tool` 驗證

### #13：SearXNG 不是 OpenClaw 原生 web_search provider
- 整合方式：透過 exec tool 呼叫 wrapper script，**不是** `web_search.provider`

### #14：換模型前必須確認 tools capability
- `ollama show MODEL | grep capabilities`，必須包含 `tools`
- 不支援的模型：gemma3、phi4、llama3.2-vision

### #15：SearXNG `formats` 缺少 json → 403 Forbidden
- **原因：** `settings.yml` 的 `formats` 只有 `html`，JSON API 被封鎖
- **解法：** 加 `json` 到 formats 清單

### #16：OpenClaw system prompt 太大導致小模型 tool calling 失敗
- **根因：** OpenClaw 的 system prompt + tool schemas 達 12,000+ tokens
- **影響：** 3B-9B 模型在此壓力下無法正確決策呼叫哪個 tool
- **解法：** 用 24B+ 模型（如 mistral-small3.1），或完善 AGENTS.md 讓 8B 模型（qwen3-vl）也能穩定運作

---

## 常用指令速查

```bash
# 啟動 OpenClaw（Mac）
source ~/.nvm/nvm.sh && openclaw gateway --bind lan --port 18789 --allow-unconfigured

# 重啟 OpenClaw（Mac，LaunchAgent 方式）
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway

# PC：查看目前載入的模型與 VRAM 用量
ssh USER@PC_IP "curl -s http://localhost:11434/api/ps | python3 -c \"import sys,json; d=json.load(sys.stdin); [print(m['name'], m.get('size_vram',0)//1024//1024, 'MB') for m in d.get('models',[])]\""

# PC：確認模型 capabilities
ssh USER@PC_IP "curl -s http://localhost:11434/api/show -d '{\"name\":\"MODEL\"}' | python3 -c \"import sys,json; print(json.load(sys.stdin).get('capabilities'))\""

# PC：重啟 Ollama
ssh USER@PC_IP "sudo systemctl restart ollama"

# Mac：驗證 openclaw.json 格式
python3 -m json.tool ~/.openclaw/openclaw.json > /dev/null && echo "JSON OK"
```

---

## 附加工具

本 repo 包含兩個 Claude Code / OpenClaw skill，可直接複製使用。

### LLM Benchmark Skill

測試本地 Ollama 模型的效能（tok/s、TTFT、VRAM 用量），自動產生 Markdown 報告。

```bash
# 複製 benchmark script 到 PC
scp skills/llm-benchmark/scripts/benchmark.py USER@PC_IP:/tmp/

# 在 PC 上執行（需 pip install requests）
python3 /tmp/benchmark.py qwen3-vl:8b-instruct
```

或安裝為 Claude Code skill：
```bash
cp -r skills/llm-benchmark ~/.claude/skills/
```

### SearXNG Skill

OpenClaw 的本地搜尋 skill，透過 exec tool 呼叫 SearXNG 實例。

```bash
# 安裝 wrapper script（macOS Apple Silicon）
cp skills/searxng/scripts/searxng-search /opt/homebrew/bin/
chmod +x /opt/homebrew/bin/searxng-search
# 編輯 searxng-search，把 PC_LAN_IP 改成你的 PC IP

# 安裝 OpenClaw skill
cp -r skills/searxng ~/.openclaw/workspace/skills/
```

---

*基於 OpenClaw 2026.3.13 · Ollama 0.17.7 · RTX 5070 Ti 16GB · 2026-03-19 實測*
*13 個模型實測，完整數據見本文「模型選擇指南」段落*
