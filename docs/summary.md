# Webwright 專案摘要

## 專案用途
Webwright 是一個以 Python 實作的輕量瀏覽器代理（browser agent）框架，核心理念是把「瀏覽器」當成代理可啟動、觀察、丟棄的執行環境，而把「工作區中的程式、截圖、軌跡與日誌」當成真正狀態。它同時提供：

- 可直接執行的 CLI harness（`python -m webwright.run.cli` / `webwright`）
- 可安裝到 Claude Code、Codex、OpenClaw、Hermes 的共享 skill / plugin
- 補助工具（`image_qa`、`self_reflection`）與可重播結果展示頁（`assets/task_showcase/`）

對開發者來說，這個 repo 的重點不是封裝大量 agent framework，而是把「prompt → 執行一步 → 觀察 → 修正」這條路徑維持得很短、很可追蹤。

## 高階架構
整體流程大致如下：

1. `src/webwright/run/cli.py`
   - 讀入並合併多個 YAML 設定（例如 `base.yaml` + `model_openai.yaml`）
   - 決定輸出目錄，並把合併後設定快照寫到 `config_snapshot/`
   - 建立 model、environment、agent
2. `src/webwright/agents/default.py`
   - 維護對話訊息、套用 prompt template、呼叫模型
   - 把模型輸出轉成單步 action（`bash_command` 或 `python_code`）
   - 寫出 `trajectory.json` 與 `debug/steps.*` 等除錯產物
   - 在 workspace 模式下可強制要求 `self_reflection` 成功後才能完成
3. `src/webwright/environments/`
   - `local_workspace.py`：預設模式。代理透過 shell 指令在工作區內產生/執行 Playwright 腳本，並收集 logs、screenshots、recent files 等觀察
   - `local_browser.py`：即時瀏覽器模式。代理每一步直接送出 async Python 片段操作 live Playwright page
4. `src/webwright/models/`
   - `base.py` 定義共用請求/重試/格式驗證邏輯
   - `openai_model.py`、`anthropic_model.py`、`openrouter_model.py` 實作不同 provider
5. `src/webwright/tools/`
   - `image_qa.py`：對單張或多張截圖做視覺問答
   - `self_reflection.py`：兩階段截圖評分與最終 verdict
   - `_model_config.py`：讓工具重用 CLI 輸出的 `merged_config.yaml`，與主 agent 使用相同模型設定

## Skill / Plugin 結構
這個 repo 同時是一個可安裝 skill 的來源庫。

- `.claude-plugin/plugin.json`：Claude Code plugin manifest
- `.codex-plugin/plugin.json`：Codex plugin manifest
- `skills/webwright/SKILL.md`：共享 skill 主契約
- `skills/webwright/commands/run.md`：one-shot 任務模式
- `skills/webwright/commands/craft.md`：可重用 CLI tool 模式
- `skills/webwright/reference/`
  - `workflow.md`：plan → explore → final → verify 流程
  - `cli_tool_mode.md`：參數化 CLI 腳本契約
  - `playwright_patterns.md`：標準 Playwright 寫法與規則

要注意：

- **核心 Python harness** 內建 `image_qa` / `self_reflection` 工具與 workspace artifact gate。
- **skill 版本** 則把宿主代理（Claude/Codex 等）視為主控者，重用同一套 workspace contract，但在 `SKILL.md` 中把部分驗證工作改成由宿主代理原生能力完成。

也就是說，修改 repo 時要分清楚自己是在調整：

- 核心 CLI/harness 行為
- 還是 skill 文件與宿主代理契約

## 重要路徑
### 原始碼
- `src/webwright/run/cli.py`：CLI 入口與組裝流程
- `src/webwright/agents/default.py`：主 agent loop、debug artifact、completion gate
- `src/webwright/environments/local_workspace.py`：workspace/shell 模式
- `src/webwright/environments/local_browser.py`：live browser 模式
- `src/webwright/models/base.py`：模型抽象、JSON action schema、重試邏輯
- `src/webwright/models/*.py`：provider 實作
- `src/webwright/tools/*.py`：截圖問答與自我驗證工具
- `src/webwright/config/*.yaml`：執行模式與 model overlays

### Skill / 文件
- `README.md`：整體定位、快速開始、plugin 安裝方式、專案地圖
- `skills/webwright/SKILL.md`：skill 主說明
- `skills/webwright/commands/*.md`：命令入口
- `skills/webwright/reference/*.md`：操作契約
- `assets/task_showcase/README.md`：展示頁與 `report.json` 結構

### 測試
- `tests/unit/test_tool_model_routing.py`：驗證工具會正確讀取 top-level `model:` 設定與 config snapshot
- `tests/conftest.py`：把 `src/` 加入測試 import path

## 修改時建議怎麼切入
1. **先判斷你改的是哪一層**
   - CLI / agent loop：從 `run/cli.py`、`agents/default.py` 看起
   - 執行環境：從 `environments/` 看起
   - 模型 provider：從 `models/base.py` 與對應 provider 檔案看起
   - skill 契約：從 `skills/webwright/` 看起
2. **先確認設定疊加結果**
   - 很多行為不是寫死在 Python，而是由 `base.yaml` + overlay 決定
   - 若工具或 agent 行為看起來不一致，優先檢查 `config_snapshot/merged_config.yaml`
3. **維持 contract 一致**
   - 若改 `action_field`、輸出 schema、workspace artifact 名稱、self-reflection gate，需同步確認 `agent`、`environment`、`tools`、`skill docs` 是否仍一致
4. **小改優先、避免破壞 artifact 形狀**
   - 這個 repo 很依賴固定檔名/資料夾（如 `final_script.py`、`final_runs/run_<id>/`、`final_script_log.txt`、`self_reflect_result.json`）
   - 改名或改資料流前，先確認 README、skill docs、tool loader、測試是否也要一起更新

## 除錯建議
### Workspace 模式（預設）
優先看輸出目錄中的這些檔案：

- `trajectory.json`：整段 agent 軌跡
- `debug/steps.md`、`debug/steps/*.json`：每一步 thought / action / observation
- `steps/`、`logs/`、`command_history.sh`：實際執行過的 shell 指令與輸出
- `screenshots/`：探索或步驟截圖
- `final_runs/run_<id>/`：最終腳本、最終執行截圖、log、judge 結果
- `config_snapshot/merged_config.yaml`：實際生效設定

常見問題切法：

- **模型格式錯誤**：看 `models/base.py` 的 schema 與 format error template
- **工具拿到錯的模型設定**：看 `tools/_model_config.py` 與對應單元測試
- **代理提早結束/無法結束**：看 `DefaultAgent` 的 done gate 與 self-reflection 檢查
- **shell 指令跑出 workspace 外**：看 `LocalWorkspaceEnvironment._resolve_cwd()`

### Live browser 模式
優先看：

- `src/webwright/config/local_browser.yaml`
- `src/webwright/environments/local_browser.py`

這個模式的重點是：

- 狀態存在 live page，而不是 workspace artifact
- `python_code` 每步直接操作已存在的 `page/context/browser`
- `browser_mode` 可能是 `local_launch`、`local_persistent`、`local_cdp`
- 若是 CDP 問題，先檢查本機瀏覽器啟動、CDP URL、user data dir 與是否有 page target

## 對維護者最重要的幾點
- Webwright 的價值在於**簡單、可讀、可重播**；修改時盡量不要把流程藏進過多抽象。
- config overlay 是核心設計，不要只看單一 YAML。
- skill 文件其實是產品介面的一部分；改 workflow/contract 時，`SKILL.md`、commands、reference 文件通常要一起檢查。
- `image_qa` / `self_reflection` 與主 agent 必須共用同一份 model 設定，這是目前少數已有測試明確保護的契約之一。
