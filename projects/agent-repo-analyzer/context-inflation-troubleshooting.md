# Context 膨脹問題排解

- 背景來源：[GitHub Issue #10](https://github.com/Aiii-Developers/aiii-repo-analyzer/issues/10)
- 狀態：OPEN
- 作者：TaroAiii

本檔整理我實際遇到的 context 膨脹問題、排解過程與驗證結果；GitHub issue #10 是本案例的背景來源，供 workflow.md 的 RAG parent/child 流程引用。

## 原始案例紀錄（GitHub Issue #10）

## Hermes webhook context 重複壓縮：實際案例與四類問題

### 案例

- Session ID：`20260903_114352_30b7bd1d`
- Delivery ID：`f2af020b-d8e4-4a6f-a798-421a783a5325`
- Source：`webhook`
- Model：`gpt-5.6-sol`（provider：`openai-codex`）
- 執行時間：1,594.9 秒（約 26.6 分鐘）
- API calls：51
- Session 累積 input tokens：1,088,352
- Context compression：18 次

這不是跨 webhook 共用 conversation context。Webhook adapter 會把 Delivery ID 放進 session key：

```text
webhook:{route}:{delivery_id}
```

每個 delivery 是獨立 one-shot session，完成後以 `webhook_complete` 關閉。因此本案例的 18 次壓縮全部發生在同一筆新 webhook session 內。

「新 session」只表示不繼承舊對話，不表示 context 用量從 0 開始。`hermes prompt-size --platform webhook --json` 顯示 fresh session 固定載入約：

| 項目 | 大小 |
|---|---:|
| System prompt | 22.9 KB |
| Skills index | 9.6 KB |
| Memory | 2.4 KB |
| User profile | 1.8 KB |
| 11 個 tool schemas | 21.5 KB |
| 本次 webhook / skill payload | 13.1K chars |

## 問題一：設定把 context window 鎖在 65,536，低於 Codex runtime 可用窗口

目前設定：

```text
model.context_length = 65536
compression.threshold = 0.5
```

`get_model_context_length()` 把明確的 `model.context_length` 設定列為最高優先級，因此 compressor 先依 65,536 初始化。小窗口保護邏輯又會把實際 threshold 拉到約 85%，即約 55K tokens。

另一方面，Hermes UI 又顯示 Codex `gpt-5.6-sol` 可使用 272K context。使用者因此看到 272K，但壓縮仍大約在 65K request pressure 附近發生，行為與提示不一致。

## 問題二：Codex runtime 更新 context window 時沒有重算 compression threshold

`agent/codex_runtime.py` 在 response 回報 `model_context_window` 後，只執行：

```python
compressor.context_length = context_window
```

但沒有同步重算：

- `threshold_percent`
- `threshold_tokens`
- `tail_token_budget`
- `max_summary_tokens`

因此可能形成：

```text
compressor.context_length 已更新為 272K
threshold_tokens 仍是依 65K 計算的舊值
```

這會造成模型明明仍有大量可用 context，Hermes 卻持續提早壓縮。

## 問題三：retrieval 找到候選後，工作流沒有收斂，工具結果持續灌入 context

本次完整 session DB 歷史中可辨識出大量工具呼叫，包括：

- `terminal`：19 次
- `read_file`：12 次
- `get_code_snippet`：10 次
- `search_graph`：6 次
- `trace_path`：4 次
- `search_code`：2 次

部分單筆結果大小達 33K、22K、21K chars。`get_code_snippet` 結果合計超過 100K chars，另有大量 terminal/read_file 輸出。

Retrieval 只回傳候選，不會自動限制後續 routing，也不會從 context 移除結果。理想流程應是 retrieval 命中後只驗證最高相關的 3–5 個候選，取得足夠的 symbol、位置與 call path 後立即停止搜尋。本次 agent 仍持續擴大 search/read/trace，抵銷 retrieval 原本應帶來的 context 節省。

## 問題四：壓縮摘要本身遞迴膨脹，且缺少合理的 session 級停止條件

第 18 次壓縮後，最後保留的 active prior-context summary 為：

```text
summary chars: 145,312
[PRIOR CONTEXT] occurrences: 16
Historical Task Snapshot occurrences: 47
Completed Actions occurrences: 17
```

摘要明顯反覆攜帶前一次 summary/header/historical sections。結果不是「壓縮後降到安全水位」，而是壓縮後仍接近 threshold；新增少量 tool result 就再次觸發壓縮。

目前有單一 conversation-loop 內的嘗試上限，但沒有有效的 per-session compression 次數上限或「壓縮後尺寸沒有改善」的 anti-thrash breaker，所以本案例一路重複到 18 次，並伴隨狀態品質下降與流程重做。

## 預期行為

```text
新 webhook session
→ retrieval 找到少量候選
→ 限定 3–5 個 MCP snippet/trace 驗證
→ 根因證據足夠即停止 routing
→ 回寫結果
```

一般 webhook 分析不應在單一 one-shot session 中壓縮 18 次。

## 建議修正與驗收條件

1. 移除不必要的 `model.context_length=65536` override，或讓設定與 Codex runtime 回報窗口一致。
2. Codex runtime 收到 `model_context_window` 時，應走可完整重算 threshold/budgets 的更新方法，不能只指定 `context_length`。
3. 為 retrieval 命中後的 MCP routing 設定候選數、結果大小與停止條件；避免已取得根因證據後繼續廣搜。
4. 修正 summary carry-forward normalization，避免 prior context/historical sections 遞迴嵌套；增加：
   - 壓縮後 token/size 必須實際下降的 assertion/metric。
   - 連續壓縮無改善時的 anti-thrash breaker。
   - 合理的 per-session compression 上限與明確錯誤狀態。
5. Regression test 至少涵蓋：
   - 65K 初始化後 runtime 更新至 272K，`threshold_tokens` 同步重算。
   - 對同一 summary 連續壓縮多次，`[PRIOR CONTEXT]` 與 historical headings 不會線性或指數累積。
   - retrieval 命中後，工具呼叫與結果 token budget 有界。

### 驗收目標

- 同等 webhook 測試查詢不再出現連續壓縮循環。
- summary 每次壓縮後均低於目標 budget，且不包含重複 prior-context wrappers。
- 顯示的 context window、實際 threshold 與 runtime model window 一致。
- retrieval 命中後能在有限的 MCP 驗證呼叫內完成分析與回寫。

## 排解後的實作結果

### Comment 1（TaroAiii，2026-09-03T06:46:50Z）

## Hermes webhook context 優化實作與效果

已完成 Hermes context/runtime、摘要正規化及 `action: aiii-repo-analyzer` skill 工作流優化，並以新的真實 webhook 完成端到端驗證。

### 已實作項目

1. **恢復正確的 context window**
   - 移除錯誤的 `model.context_length=65536` override。
   - `openai-codex / gpt-5.6-sol` 現在由 provider metadata 解析為 **272,000 tokens**。
   - 使用既有 Codex 專屬 **85%** 壓縮門檻，實際 threshold 為 **231,200 tokens**。

2. **修正 Codex runtime 與 compressor 預算不同步**
   - runtime 回報 context window 時，不再只更新 `context_length`。
   - 同步重算 threshold、tail token budget、summary budget。
   - explicit config override 仍維持最高優先權。
   - 相同 runtime/window 回報不會清除 usage、fallback 或 anti-thrash 狀態。

3. **阻止 context summary 遞迴膨脹**
   - 遞迴移除舊的 `PRIOR CONTEXT` wrapper。
   - 重複 Markdown summary sections 只保留最新一份。
   - summary 產生後實際執行 token budget 限制，不再只靠 prompt 提示。
   - 以原 18 次壓縮 session 的真實摘要測試：
     - 字元：**145,312 → 6,570**
     - wrapper：**16 → 0**
     - Historical Task sections：**34 → 1**
     - 正規化後 rough tokens：**1,674**

4. **直接優化 `action: aiii-repo-analyzer` skill**
   - 9-step workflow 精簡為 4 個 phase。
   - MCP discovery/allowlist 檢查改為工具失敗時才執行，不再每筆 webhook 重跑。
   - 移除兩次額外的 todo checkpoint。
   - retrieval 命中後直接進入 `get_code_snippet`／`trace_path` 驗證。
   - 找到根因、具體位置及兩組證據後立即停止探索。
   - 禁止 MCP snippet 與 `read_file` 重複讀取相同範圍。
   - 設定 investigation MCP call、候選檔案及證據鏈預算。
   - skill 已建立本機上游 working copy並同步至 Hermes 安裝副本。

5. **修正 writeback Python runtime**
   - `write-results.py` 與 retrieval/writeback 指令固定使用 Hermes venv Python 3.11。
   - `write-results.py` 加入延後解析 annotations；即使誤用系統 Python 3.9，也不再於函式宣告階段因 `dict[...] | None` 崩潰。

6. **Gateway 重啟**
   - 重啟前 PID：`98504`
   - 重啟後 PID：`3797`
   - 新 process 使用 Hermes venv。
   - 此 macOS 的 launchd 回傳 exit 5，Hermes 成功改用 detached fallback；Gateway 已正常監聽 webhook，無 startup exception。

### 測試結果

- Context compressor／anti-thrash：**202 passed**
- Codex runtime／event bridge：**76 passed**
- Skill/writeback regression：**3 passed**
- `pip check`：成功，無 broken requirements
- Python 3.9 與 Python 3.11 均可載入 `write-results.py`
- MCP dependency：
  - `mcp 1.26.0`
  - `starlette 1.0.1`
  - `CallToolResult` 具備 `isError`、`structuredContent`
- `hermes mcp test codebase_memory`：成功，目前發現 **14 tools**
- 真正 Hermes `call_tool`：
  - `get_architecture`：成功
  - `search_graph`：成功
  - 無 `CallToolResult/isError` 錯誤

### 新 webhook 端到端結果

- Doc ID：`6a991238cdcbedd4dff43774`
- Delivery ID：`2df066a6-bcd3-465c-a6e6-f783dd6b0ec3`
- Session ID：`20260903_142249_d1cdc5a3`
- Source / end reason：`webhook / webhook_complete`
- Studio PATCH：**HTTP 200**
- Local embedding POST：**HTTP 200，success=true**
- 最終 webhook response：已成功送出

| 指標 | 優化前 | 優化後 |
|---|---:|---:|
| 執行時間 | 1,594.9 秒 | **241.2 秒** |
| API calls | 51 | **18** |
| Tool calls | 45 | **34** |
| Input tokens | 1,088,352 | **77,072** |
| Output tokens | 15,450 | **6,201** |
| Context 壓縮次數 | 18 | **0** |

執行時間降低約 **84.9%**，input tokens 降低約 **92.9%**，context 壓縮由 18 次降為 0 次。

### 尚未完成／觀察項目

- Skill 文字規則設定 investigation MCP 上限 12 次，但實測使用 14 次；若要保證硬上限，需要在 Hermes tool dispatcher 實作 per-action 程式化 budget。
- Gateway 目前是 detached fallback process，launchd 尚不能提供登入自啟與 crash auto-restart。
- `aiii-skills` 上游內容依本次限制只保留在本機 working copy，尚未 commit/push。

本次未輸出或修改任何 credential，所有敏感值均維持 `[REDACTED]`。

留言連結：[GitHub comment](https://github.com/Aiii-Developers/aiii-repo-analyzer/issues/10#issuecomment-5521714527)

## 與 workflow.md 的關聯

workflow.md 將本 issue 作為 context routing 與 RAG retrieval 的回歸背景：child 先做細粒度召回，再依 parent_id 去重與聚合，只將少量 parent 交給 MCP 驗證。這只能處理 retrieval／工具輸出邊界；context window、summary normalization 與 anti-thrash 仍需由 runtime／compressor 一併保護。

