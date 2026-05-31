# Webwright 開發者快速入門指南 (Quick Start Guide)

歡迎來到 **Webwright**！本文件旨在幫助新加入的開發者快速熟悉此專案的設計理念、開發環境架構，並引導你成功執行第一個任務與開發自訂工具。

在閱讀本指南前，建議先閱讀 [summary.md](file:///Users/will/projects/webwright/docs/summary.md) 以瞭解專案的整體架構與核心模組。

---

## 1. 核心理念：程式碼即行動 (Code-as-Action)

傳統的網頁代理（Web Agent）通常是在每一步由 LLM 決定單一的網頁操作（如 click, type），並依賴持久化的瀏覽器狀態。

Webwright 採取了不同的思維方式：
- **瀏覽器只是執行環境：** 代理可以在執行過程中隨時啟動新的瀏覽器執行探索，並在執行結束後丟棄它。
- **工作區才是真正狀態：** 真正的持久化狀態是你在工作區（Workspace）中寫入的程式碼、產生的螢幕截圖、軌跡（trajectory）與執行日誌。
- **反覆測試與修正：** 代理的任務不是「一次做對」，而是編寫一個 Python 腳本（`final_script.py`），執行它，檢查截圖與日誌，並持續修正程式碼，直到滿足所有限制條件。

---

## 2. 環境建置與安裝

本專案需要 Python 3.10+。請在專案根目錄下依序執行以下步驟：

### 2.1 安裝專案依賴

使用可編輯模式（editable mode）安裝套件：

```bash
pip install -e .
```

這會安裝核心依賴，如 `playwright`、`httpx`、`pydantic`、`typer` 等。

### 2.2 安裝 Playwright 瀏覽器

Webwright 預設使用 Chromium 與 Firefox 進行網頁操作：

```bash
# 安裝 Chromium (用於標準 CLI harness)
playwright install chromium

# 安裝 Firefox (用於外掛/技能模式下避開 Akamai 等 TLS/H2 指紋辨識)
playwright install firefox
```

---

## 3. 執行你的第一個網頁任務

執行 Webwright 需要設定對應的 LLM API 金鑰。我們以 OpenAI 後端為例。

### 3.1 設定 API 金鑰

```bash
export OPENAI_API_KEY="your-api-key-here"
```

如果使用 Anthropic 後端，請匯出 `ANTHROPIC_API_KEY`；若使用 OpenRouter，請匯出 `OPENROUTER_API_KEY`。

### 3.2 執行預設任務

Webwright 的進入點是 [cli.py](file:///Users/will/projects/webwright/src/webwright/run/cli.py)。我們可以使用以下指令啟動：

```bash
python -m webwright.run.cli \
    -c base.yaml -c model_openai.yaml \
    -t "查詢從西雅圖(SEA)到紐約(JFK)，在 2026-08-15 出發、2026-08-20 回程的航班" \
    --start-url https://www.google.com/flights \
    --task-id flight_search_demo \
    -o outputs/default
```

### 參數說明：
- `-c` / `--config`：載入來自 `src/webwright/config/` 的設定檔。可以堆疊多個 YAML 檔（後者會覆蓋前者）。
- `-t` / `--task`：要執行的任務描述。
- `--start-url`：瀏覽器初始載入的網頁 URL。
- `--task-id`：此任務在輸出目錄下的子目錄名稱。
- `-o` / `--output-dir`：所有產物輸出的根目錄。

---

## 4. 輸出產物結構與除錯

執行後，你會在指定的工作區（例如 `outputs/default/flight_search_demo/`）中看到以下結構：

```text
flight_search_demo/
├── plan.md                       # 任務關鍵點 (Critical Points) 清單與勾選狀態
├── config_snapshot/
│   └── merged_config.yaml        # 實際生效的完整 YAML 設定快照
├── trajectory.json               # 主代理的對話軌跡日誌
├── screenshots/                  # 探索階段所擷取的螢幕截圖
├── logs/                         # 探索階段指令的 stdout / stderr 輸出
├── command_history.sh            # 探索階段執行的歷史指令
├── debug/                        # 每一步的詳細 Thought/Action/Observation JSON 與 Markdown
└── final_runs/                   # 乾淨執行 final_script.py 的各次產物
    └── run_1/
        ├── final_script.py       # 代理最終撰寫出、可重複執行的 Playwright Python 腳本
        ├── final_script_log.txt  # 最終腳本執行的結構化步驟日誌與最終數據 (FINAL_RESPONSE)
        └── screenshots/          # 對應 Critical Points 的驗證螢幕截圖
```

### 除錯技巧：
1. **模型格式錯誤：** 檢查 [base.py](file:///Users/will/projects/webwright/src/webwright/models/base.py) 裡的 JSON Schema 與格式錯誤重試邏輯。
2. **工具設定不一致：** 若輔助工具（例如 `image_qa`）使用的模型與主代理不同，檢查 [_model_config.py](file:///Users/will/projects/webwright/src/webwright/tools/_model_config.py)。
3. **確認設定覆蓋：** 行為不符合預期時，優先檢查 `config_snapshot/merged_config.yaml` 確認最終合併後的設定。

---

## 5. 新增自訂的開發功能

若你需要擴充 Webwright，通常會涉及以下三個部分之一：

1. **調整 Agent 流程：** 修改 [default.py](file:///Users/will/projects/webwright/src/webwright/agents/default.py) 中的 `DefaultAgent`，調整其對話格式、重試限制與 done 條件。
2. **調整 Environment 行為：** 修改 [local_workspace.py](file:///Users/will/projects/webwright/src/webwright/environments/local_workspace.py) (針對 workspace 模式) 或 [local_browser.py](file:///Users/will/projects/webwright/src/webwright/environments/local_browser.py) (針對 live browser 模式)。
3. **新增執行模式 (Overlay Configs)：** 在 `src/webwright/config/` 下新增 YAML 設定檔。

### 範例：新增模型設定
如果你想新增一個模型支援，可以在 `src/webwright/models/` 新增一個 subclass 繼承自 `BaseModel`（定義於 [base.py](file:///Users/will/projects/webwright/src/webwright/models/base.py)），並在 [cli.py](file:///Users/will/projects/webwright/src/webwright/run/cli.py) 進行註冊。

---

## 6. 單元測試

修改程式碼後，請務必執行單元測試以確保沒有破壞現有的整合契約：

```bash
pytest
```

主要的測試案例（如 `tests/unit/test_tool_model_routing.py`）會驗證輔助工具是否正確地與主 agent 共享相同的模型配置。
