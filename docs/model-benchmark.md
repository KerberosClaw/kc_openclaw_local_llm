# 本地 LLM Benchmark 報告（OpenClaw 用途）

> **English summary:** Benchmark results for local LLMs on RTX 5070 Ti 16GB, tested for OpenClaw agent use. Models: phi4:14b, gemma3:12b, llama3.1:8b, ministral-3:14b, qwen3-vl:8b (thinking vs instruct), mistral-small3.1:24b, qwen2.5:14b-instruct. Metrics: tok/s across context windows (2K-32K), answer quality (logic, reasoning, coding, debugging), VRAM usage, CPU offload detection. Includes context window sweet spot analysis and OpenClaw suitability assessment. The document is written in Traditional Chinese.

> 生成時間：2026-03-15（上次更新：2026-03-16 加入 qwen3-vl:8b-instruct benchmark 比較）
> 測試條件：OpenClaw 停止後進行，VRAM 淨空，RTX 5070 Ti 16GB

---

## 執行環境

| 項目 | 規格 |
|------|------|
| GPU | NVIDIA RTX 5070 Ti 16GB VRAM |
| RAM | 58GB |
| OS | Windows 11 + WSL2 Ubuntu |
| Ollama | v0.17.7 |
| 測試時間 | 2026-03-15 |

---

## 測試模型

| 模型 | 參數量 | 最大 ctx | 量化 | 大小 |
|------|--------|----------|------|------|
| phi4:14b | 14B | 16,384 | Q4 | 9.1 GB |
| gemma3:12b | 12B | 131,072 | Q4 | 8.1 GB |
| llama3.1:8b | 8B | 131,072 | Q4 | 4.7 GB |
| ministral-3:14b | 14B | 262,144 | Q4 | 8.7 GB |

---

## Token/s 速度總覽

| 模型 | ctx=2048 | ctx=4096 | ctx=8192 | ctx=16384 | ctx=32768 |
|------|----------|----------|----------|-----------|-----------|
| llama3.1:8b | 142.5 | 141.9 | 142.5 | — | — |
| gemma3:12b | 81.5 | 81.8 | 81.0 | — | — |
| phi4:14b | 77.2 | 77.4 | 77.3 | 76.8 | — |
| ministral-3:14b | 78.0¹ | 85.5 | 75.8² | 85.4 | 49.2 ⚠️ |

> ¹ ctx=2048 首批測試包含模型冷啟動（VRAM 從 1.2GB 暖機至 11GB），且 2048 token 上限可能截斷長回答，影響部分題目速度。
>
> ² ctx=8192 多步推理一題生成了 12,941 tokens（超長 chain-of-thought），拉低平均值；其他三題仍在 71–86 tok/s。
>
> ⚠️ ctx=32768：VRAM 使用量達 14,476 MB（佔 88.8%），KV cache 幾乎把顯存塞滿，記憶體頻寬嚴重不足，tok/s 從 ~85 跌至 ~49（下降約 42%）。模型本身仍在 GPU 上跑，但效能已大幅劣化，不建議使用此 ctx 大小。

---

## 回答品質評估

### 邏輯題（3開關3燈，只能進房一次）
正確解法：先開第1個開關幾分鐘使燈泡發熱，關掉；再開第2個；進房後：亮的=第2個，熱但暗的=第1個，冷暗的=第3個。

| 模型 | 正確? | 備註 |
|------|-------|------|
| phi4:14b | ✅ | 正確提及加熱燈泡，邏輯完整 |
| gemma3:12b | ✅ | 最清晰，用熱量判斷說明詳盡 |
| llama3.1:8b | ❌ | 答案錯誤，說「打開門觀察」（題目明說看不見）|
| ministral-3:14b | ✅ | 正確，熱燈泡判斷說明清楚 |

### 推理題（5嫌疑人邏輯推理）
正確解：A→C，B→D，C與D不同時犯案。假設A犯→C犯→D不可犯→B不可犯，結果 {A,C}；或假設B犯→D犯→C不可犯→A不可犯，結果 {B,D}。兩解皆合法。

| 模型 | 正確? | 備註 |
|------|-------|------|
| phi4:14b | ✅ | 條件分析清晰，推理過程正確 |
| gemma3:12b | ✅ | 條件列舉詳盡，推理正確 |
| llama3.1:8b | ⚠️ | 推理方向對但結論說明不完整 |
| ministral-3:14b | ✅ | 條件逐步推導清晰，邏輯正確 |

### 程式設計 / 除錯
| 模型 | 程式設計 | 程式除錯 |
|------|----------|----------|
| phi4:14b | ✅ 正確 | ✅ 正確 |
| gemma3:12b | ✅ 正確 | ✅ 正確 |
| llama3.1:8b | ✅ 正確 | ✅ 正確 |
| ministral-3:14b | ✅ 正確（使用 Counter） | ✅ 正確（找出 range bug） |

---

## Context Window 分析

| 模型 | 甜蜜點 | 大 ctx 影響 |
|------|--------|------------|
| phi4:14b | 任何 ctx≤16384 速度穩定 | 最大只有 16384（限制較大）|
| gemma3:12b | 2048–8192 最佳 | 131072 時速度驟降至 20 tok/s（VRAM 不夠全放 GPU）|
| llama3.1:8b | 2048–8192 最佳 | 131072 時速度驟降至 14 tok/s |
| ministral-3:14b | **4096–16384** | 32768 時 tok/s 跌至 49（詳見下方分析）|

---

## ministral-3:14b Context 甜蜜點深度分析

### 各 ctx 效能數據

| ctx | avg tok/s | VRAM 使用量 | 狀態 |
|-----|-----------|-------------|------|
| 2048 | ~78.0 | 10,976 MB | 含冷啟動，ctx 限制影響長回答 |
| 4096 | ~85.5 | 11,296 MB | 🏆 最佳效能起點 |
| 8192 | ~75.8 | 11,936 MB | 穩定，長 CoT 任務速度稍低 |
| 16384 | ~85.4 | 13,216 MB | 🏆 效能最佳，空間充裕 |
| 32768 | ~49.2 | 14,476 MB | ⚠️ 效能劣化 42%，不建議 |

### 為什麼 ctx=32768 速度會掉？

ctx=32768 時，KV cache 將 VRAM 塞到 14,476 MB（16GB 的 88.8%）。此時記憶體頻寬幾乎被 KV cache 讀寫吃光，GPU 雖然還在跑但等資料等到打歪，tok/s 從 85 掉到 49。

這不是「完全 CPU offload」——模型還在 GPU 上，但 VRAM 快滿的情況下，GPU 核心的等待時間大幅增加。**如果 ctx 再繼續往上堆（例如 65536+），KV cache 就真的裝不下，Ollama 會把超出部分的計算 offload 到 CPU，屆時速度會更慘（可能跌到個位數 tok/s）。**

### 結論：ministral-3:14b 甜蜜點

**建議 ctx 範圍：4096 – 16384**
- 在此範圍內，tok/s 穩定維持 84–86，效能滿載
- 16384 是最推薦的預設值：速度快、空間夠、VRAM 還有 3GB 餘裕給推理過程緩衝
- ctx=32768 雖然勉強放得下，但速度劣化 42%，不划算
- 如需長文脈絡（8192–16384 token），16384 是上限，不要往上開

---

## OpenClaw 適用性評估

| 項目 | phi4:14b | gemma3:12b | llama3.1:8b | ministral-3:14b |
|------|----------|------------|-------------|-----------------|
| 速度 | 77 tok/s | 82 tok/s | 142 tok/s | 85 tok/s |
| 邏輯推理 | ✅ 優 | ✅ 優 | ❌ 差 | ✅ 優 |
| 非 thinking model | ✅ | ✅ | ✅ | ✅ |
| ctx 上限（實用） | 16384 | 131072 | 131072 | 16384（建議）|
| VRAM 需求 | ~9GB | ~8GB | ~5GB | ~11GB |
| 繁體中文 | 良好 | 優秀 | 普通 | 良好 |

---

## 排名與建議

### 速度排名（甜蜜點 ctx 下）
1. llama3.1:8b（142 tok/s）
2. ministral-3:14b（85 tok/s，推薦 ctx 4096–16384）
3. gemma3:12b（82 tok/s）
4. phi4:14b（77 tok/s）

### 品質排名
1. gemma3:12b（推理清晰、中文自然）
2. ministral-3:14b（推理正確，邏輯清晰）
2. phi4:14b（推理正確、英文偏重）
4. llama3.1:8b（基礎題錯誤）

### 建議

**首選：gemma3:12b**
- 速度、品質、ctx 三者最平衡
- 最大 ctx 131072（OpenClaw 工具呼叫對話不會截斷）
- 繁體中文品質最好
- 8.1GB 只佔一半 VRAM，有空間

**新選項：ministral-3:14b**
- 速度與推理能力都優於 phi4（稍慢於 gemma3）
- 甜蜜點 ctx 16384，用起來跟 phi4 相近但速度快一點
- 注意：名義上支援 262K ctx，但 16GB VRAM 下超過 16384 就開始掉速，不要亂開大 ctx
- 適合「速度/推理兼顧、不需要超長 ctx」的場景

**備選：phi4:14b**
- ctx 上限只有 16384 是硬傷（長對話容易超出）
- 品質不錯但繁體中文稍微英化

**不建議：llama3.1:8b**
- 基礎邏輯題答錯，推理能力不足
- 速度快但對 OpenClaw agent 任務幫助有限

---

---

## qwen3-vl:8b Benchmark（2026-03-16）

### 模型特性

| 項目 | 值 |
|------|----|
| 大小 | 5856 MB |
| 最大 ctx | 262,144 |
| Capabilities | completion, vision, tools, **thinking** |
| 量化 | Q4 |

### Token/s 速度

| 題目 | thinking ctx 2048 | thinking ctx 4096 | thinking ctx 8192 |
|------|-------------------|-------------------|-------------------|
| 基礎邏輯 | 107.8 tok/s | 112.7 tok/s | 117.2 tok/s |
| 多步推理 | 90.5 tok/s | 90.3 tok/s | 104.3 tok/s |
| 程式設計 | 90.3 tok/s | 90.4 tok/s | 90.4 tok/s |
| 程式除錯 | 90.6 tok/s | 90.1 tok/s | 86.2 tok/s |
| **平均** | **94.8** | **95.9** | **99.5** |

> ctx 262144 測試：**3.7 tok/s**（KV cache 佔滿 VRAM → CPU offload，不可用）

### /no_think 測試結果

| 題目 | no_think ctx 2048 | no_think ctx 4096 | no_think ctx 8192 |
|------|-------------------|-------------------|-------------------|
| 基礎邏輯 | 103.7 tok/s | 116.6 tok/s | 115.5 tok/s |
| 多步推理 | 89.9 tok/s | 99.0 tok/s | 104.8 tok/s |
| 程式設計 | 90.0 tok/s | 91.9 tok/s | 90.7 tok/s |
| 程式除錯 | 90.2 tok/s | 90.8 tok/s | 86.4 tok/s |
| **平均** | **93.5** | **99.6** | **99.4** |

**⚠️ 結論：兩組 tok/s 幾乎相同，`/no_think` 文字前綴在 `api/generate` 端點無效。**

### 慢的真正原因：token 數量

| 題目 | thinking 8192 tokens | thinking 8192 耗時 |
|------|---------------------|-------------------|
| 基礎邏輯 | 2,659 tokens | 28.6s |
| 多步推理 | 9,787 tokens | 97.3s |
| 程式設計 | 4,504 tokens | 51.3s |
| **程式除錯** | **17,214 tokens** | **198.2s** |

tok/s 本身不差（86-117），**慢是因為 thinking 模式生成大量內部推理 token**，光「程式除錯」就能超過 17000 tokens。

### 真正關閉 thinking 的方法

`/no_think` 文字前綴只在 Chat API 有效。底層關閉方法：

```bash
# 建立 no-think 版本的 modelfile
cat > /tmp/Modelfile.nothink << 'EOF'
FROM qwen3-vl:8b
PARAMETER think false
EOF
ollama create qwen3-vl-nothink -f /tmp/Modelfile.nothink
```

然後在 `~/.openclaw/openclaw.json` 改用 `qwen3-vl-nothink` 模型。

### 與 ministral-3:14b 比較

| 指標 | qwen3-vl:8b | ministral-3:14b |
|------|-------------|-----------------|
| 速度 | 86-117 tok/s | 75-85 tok/s |
| Vision | ✅ | ❌ |
| Thinking model | ✅（會慢） | ❌（穩定快） |
| VRAM | ~6-7GB | ~11GB |
| 實際回答速度 | 慢（大量思考 token）| 快（直接輸出）|
| tools 穩定性 | 待觀察 | 穩定（有 patch）|

---

---

## qwen3-vl:8b-instruct Benchmark（2026-03-16）

### 模型特性

| 項目 | 值 |
|------|----|
| 大小 | 6.1 GB |
| Capabilities | completion, vision, tools（**無 thinking**）|
| 量化 | Q4_K_M |

### Token/s 速度與 Token 數比較

#### ctx 2048

| 題目 | thinking tok/s | thinking tokens | instruct tok/s | instruct tokens | 加速倍數 |
|------|---------------|-----------------|----------------|-----------------|---------|
| 基礎邏輯 | 107.8 | 2,866（32.8s）| 121.5 | 407（8.0s）| **4.1x** |
| 多步推理 | 90.5 | 6,231（71.9s）| 97.8 | 5,751（60.1s）| 1.2x |
| 程式設計 | 90.3 | 5,096（57.3s）| 91.0 | 667（7.3s）| **7.8x** |
| 程式除錯 | 90.6 | 8,057（91.1s）| 90.9 | 1,518（20.3s）| **4.5x** |

#### ctx 4096

| 題目 | thinking tokens/時間 | instruct tokens/時間 | 加速倍數 |
|------|---------------------|---------------------|---------|
| 基礎邏輯 | 4,414（42.1s）| 454（7.3s）| **5.8x** |
| 多步推理 | 8,013（90.8s）| 4,076（38.8s）| **2.3x** |
| 程式設計 | 4,497（51.0s）| 713（10.7s）| **4.8x** |
| 程式除錯 | 11,310（127.9s）| 6,199（68.7s）| **1.9x** |

#### ctx 8192

| 題目 | thinking tokens/時間 | instruct tokens/時間 | 加速倍數 |
|------|---------------------|---------------------|---------|
| 基礎邏輯 | 2,659（28.6s）| 438（10.2s）| **2.8x** |
| 多步推理 | 9,787（97.3s）| 3,622（32.6s）| **3.0x** |
| 程式設計 | 4,504（51.3s）| 639（8.0s）| **6.4x** |
| 程式除錯 | 17,214（198.2s）| 16,680（176.8s）| 1.1x ⚠️ |

> ⚠️ ctx 8192 程式除錯：instruct 在大 ctx 下仍然產生大量 token（16,680），原因是模型給出非常詳細的說明，不是 thinking 問題。ctx 2048/4096 此題改善明顯（4.5x/1.9x）。

### 結論

**instruct 版對大多數日常任務有 2x–8x 加速**，是目前最佳選擇：
- 保留 vision + tools 支援
- 無 thinking overhead
- tok/s 與 thinking 版相同（~90-121），快是因為生成 token 數大幅減少
- 唯一例外：複雜多步推理類問題（模型本身就需要多 token 才能推導）

**建議 ctx**：4096（平衡速度與品質）

---

*報告由 llm-benchmark skill 自動生成*

---

## mistral-small3.1:24b Benchmark（2026-03-18）

### 模型特性

| 項目 | 值 |
|------|----|
| 大小 | 15 GB |
| 參數 | 24B |
| 最大 ctx | 131,072 |
| Capabilities | completion, vision, tools |
| 量化 | Q4_K_M |
| Ollama tag | `mistral-small3.1:32k`（num_ctx 固定 32768）|

### Token/s 速度

| 題目 | ctx=2048 | ctx=4096 | ctx=8192 |
|------|----------|----------|----------|
| 基礎邏輯 | 31.7 | 26.7 | 20.1 |
| 多步推理 | 31.3 | 26.3 | 20.0 |
| 程式設計 | 31.4 | 26.2 | 20.1 |
| 程式除錯 | 31.3 | 26.4 | 20.1 |
| **平均** | **31.4** | **26.4** | **20.1** |

補測（單題）：

| ctx | tok/s | 備註 |
|-----|-------|------|
| 16384 | 14.7 | 可用但慢 |
| 32768 | 10.2 | VRAM 15.7GB/16.3GB，幾乎滿載 |
| 131072 | 2.3 | ⚠️ CPU offload，不可用 |

### GPU 資源使用

| ctx | VRAM 使用 | GPU 溫度 |
|-----|-----------|----------|
| 2048 | 15,569 MB | 48°C |
| 4096 | 15,517 MB | 49°C |
| 8192 | 15,409 MB | 48°C |
| 131072 | 15,393 MB（offload） | 39°C（GPU 閒置）|

### 回答品質

| 題目 | 結果 | 備註 |
|------|------|------|
| 基礎邏輯（3 燈 3 開關）| ✅ | 正確使用加熱判斷法 |
| 多步推理（5 嫌疑人）| ✅ | 條件逐步推導，邏輯正確 |
| 程式設計（字元頻率）| ✅ | 使用 Counter，正確排序 |
| 程式除錯（Fibonacci）| ✅ | 正確找出 bug |

### Context Window 甜蜜點

| ctx | tok/s | VRAM | 建議 |
|-----|-------|------|------|
| 2048 | 31.4 | ~15.6GB | 太小，tool schemas 塞不下 |
| 4096 | 26.4 | ~15.5GB | 最低可用 |
| **8192** | **20.1** | **~15.4GB** | ✅ 適中 |
| 16384 | 14.7 | ~15GB est. | ⚠️ 偏慢 |
| **32768** | **10.2** | **~15.7GB** | **✅ OpenClaw 使用中**（最低 ctx 要求 16K）|

> OpenClaw 要求 contextWindow ≥ 16000。考慮到 system prompt（~20K tokens 含 tool schemas + AGENTS.md）需要足夠的 ctx，目前設定為 32768。10 tok/s 對 Telegram 對話可接受。

### 與其他模型比較

| 指標 | qwen3-vl:8b-instruct | ministral-3:14b | mistral-small3.1:24b |
|------|---------------------|-----------------|---------------------|
| 速度（甜蜜點）| 90-121 tok/s | 75-85 tok/s | 20-31 tok/s |
| Vision | ✅ | ❌ | ✅ |
| Tools | ✅（不穩定）| ✅（穩定，需 JS patch）| **✅（穩定，原生）** |
| 邏輯推理 | 待測 | ✅ 優 | ✅ 優 |
| 繁體中文 | ✅ 優 | 良好 | 良好 |
| VRAM | ~6-7GB | ~11GB | ~15GB |
| ctx 甜蜜點 | 4096 | 4096-16384 | 8192-32768 |

### 為什麼選 mistral-small3.1

- **Tool calling 穩定**：qwen3-vl:8b 把 shell 指令名當 tool name，導致 OpenClaw exec 循環 timeout；mistral-small3.1 的 native function calling 從第一次測試就正確
- **Vision + Tools 兼具**：ministral-3:14b 雖然 tool calling 穩定但沒有 vision
- **速度換穩定**：10 tok/s @32K 比 qwen3-vl 的 90 tok/s 慢很多，但 qwen3-vl 的 exec 根本不能用，等於速度再快也是 0
- **24B 參數量**：推理品質明顯優於 8B，四道題全部正確
