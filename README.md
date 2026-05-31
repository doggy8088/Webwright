# Webwright

<p align="center">
  <img src="assets/webwright_logo.svg" alt="Webwright logo" width="320">
</p>

<p align="center"><b>讓你的程式設計模型成為最先進的瀏覽器代理</b></p>

<p align="center">
  <img src="https://img.shields.io/badge/python-%E2%89%A53.10-blue?logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/playwright-chromium-green" alt="Playwright">
  <img src="https://img.shields.io/badge/backends-OpenAI%20%7C%20Anthropic%20%7C%20OpenRouter-orange" alt="Backends">
  <img src="https://img.shields.io/badge/footprint-%E2%89%A4~1.5k%20LoC-brightgreen" alt="Footprint">
</p>

- 📝 **部落格：** [Webwright: A Terminal Is All You Need For Web Agents](https://www.microsoft.com/en-us/research/articles/webwright-a-terminal-is-all-you-need-for-web-agents/)
- 🌐 **專案頁面：** [microsoft.github.io/Webwright](https://microsoft.github.io/Webwright/)

Webwright 會為 LLM 提供一個終端機，讓它能啟動多個瀏覽器工作階段來檢查頁面並完成網頁任務。它只在需要時擷取並檢視頁面截圖／狀態。它要求每個網頁任務都必須以可重複執行的 Python 腳本端到端完成，也就是說，你的網頁代理瀏覽歷程就是單一程式碼檔案。沒有多代理系統、沒有圖形引擎、沒有外掛層、沒有隱藏的編排機制——只有終端機、瀏覽器與模型。

已經有你偏好的代理，並且想讓 Claude Code、Codex、Hermes、OpenClaw 在瀏覽器任務上更強大嗎？可以考慮加入 [Webwright 外掛／技能](#-use-as-a-plugin)！

---

## 📰 最新消息

- **2026-05-11** — 支援 Task2UI 模式：Webwright 完成任務後，會將任務結果渲染成以 HTML 為基礎的 Web 應用程式，方便你檢視與重複使用。  
- **2026-05-06** — 已新增 Codex 與 Claude Code 的外掛資訊清單；可透過 `/plugin install webwright@webwright` 安裝。也已推出 OpenClaw 與 Hermes Agent 整合；同一個 `skills/webwright/` 資料夾現在可在 Claude Code、Codex、OpenClaw 與 Hermes 之間共用載入。
- **2026-05-04** — 首次公開釋出：~1.5k LoC、OpenAI / Anthropic / OpenRouter 後端，以及 Playwright 執行環境。

---

<details>
<summary><strong>💡 動機：突破在具狀態瀏覽器中逐步互動的限制</strong></summary>

目前多數網頁代理都把瀏覽器工作階段本身視為工作區：在每一步中，模型會收到目前頁面狀態，並預測下一個單一步驟操作——例如點擊、輸入、DOM selector，或簡短的工具呼叫。不論格式為何，代理都被限制在預先定義的互動迴圈中，一次只預測一個網頁操作。當 LLM 還比較弱時，這樣的框架很有用；但隨著模型越來越擅長撰寫與除錯程式碼，這種框架反而變成瓶頸。

Webwright 採取不同立場：**將代理與瀏覽器分離**，把瀏覽器視為代理在開發程式時可以啟動、檢查並丟棄的環境。持久保存的產物不是瀏覽器工作階段，而是 **本機工作區中的程式碼與日誌**。

- 🧱 **對網頁環境提供穩健、可重用的互動方式** — 不再依賴脆弱的像素層級操作，具備終端機的程式設計代理可以查詢元素、等待條件成立，並處理 lazy loading 或重新渲染等動態行為。產生出的腳本可以重新執行、調整與分享給其他任務使用，而不是每次都從零重新探索。
- ⚡ **高效率組合複雜工作流程** — 像選日期或填表單這類多步驟互動，可以濃縮成簡潔的程式。透過迴圈、函式與抽象化，代理能在相似任務間泛化（例如不同日期），不必反覆預測相同的底層操作序列。互動輪次更少、執行更快、長流程中的錯誤累積也更少。
- 🧪 **以工作區為狀態，而非以瀏覽器為狀態** — 代理可以撰寫探索性腳本、啟動全新的瀏覽器工作階段，並自行決定何時擷取截圖與檢查失敗原因，就像人類工程師反覆調整 RPA 腳本一樣。
- 🪄 **雖然極簡，效果卻出乎意料地好** — 這種精簡配置其實很能處理複雜、尤其是長時程的網頁任務（見 [效能](#-performance)）。

</details>

---

<details>
<summary><strong>🌟 為什麼選擇 Webwright</strong></summary>

多數網頁代理框架都把實際的代理迴圈埋在多層抽象之下。Webwright 則採取相反做法：

- 🪶 **設計上追求輕量** — 核心代理迴圈只有單一個約 450 行的檔案，Playwright 環境約 570 行，CLI 約 150 行。
- 🧩 **可插拔的模型後端** — OpenAI、Anthropic 與 OpenRouter 各自約 150–200 行。
- 🔍 **零隱藏框架** — 只使用 `httpx`、`pydantic`、`playwright` 與 `typer`。
- 🔁 **扁平化的 prompt → observe → execute script 迴圈** — 端到端可讀、易於除錯，也容易 fork。
- 🧪 **以執行產物為優先** — 每次執行都會將 trajectory 與截圖寫入磁碟，方便檢查。

如果你想要的是一個極簡、容易除錯、可作為瀏覽器代理起點的專案，而不是另一個龐大平台，那就是它。

</details>

---

<details>
<summary><strong>🆚 Webwright 與其他瀏覽器代理儲存庫有何不同</strong></summary>

以下是架構層級上的差異：

|                     | **Stagehand (Browserbase)**                                  | **agent-browser (Vercel)**                                                | **browser-use**                                       | **Webwright**                                                       |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------- |
| **範式**            | 混合式：程式碼 + 自然語言原語（`act` / `extract` / `agent`） | 由其他代理（Claude Code、Codex 等）呼叫的 CLI 工具                        | 以 DOM/AX snapshot 為基礎的自主 LLM 代理迴圈          | **具備終端機的程式設計代理**；瀏覽器只是它啟動的環境 |
| **動作空間**        | Playwright 程式碼，或由自然語言 → LLM 轉成 Playwright        | 離散式子命令（`open`、`click @e2`、`snapshot`、`eval`）                   | 由 LLM 選擇索引化的 click/type 操作                   | **自由形式的 Python（自行撰寫 Playwright 腳本）** |
| **什麼是「狀態」？**| 瀏覽器工作階段                                               | 瀏覽器工作階段（由 daemon 跨 CLI 呼叫維持）                               | 瀏覽器工作階段                                        | **本機工作區——程式碼、截圖與日誌。** 瀏覽器可隨時丟棄。 |
| **迴圈形狀**        | 命令式；`agent()` 會在需要時處理多步驟                       | 每個微步驟對應一次 CLI 呼叫                                               | observe → predict next action → execute → repeat      | write code → execute → inspect screenshots → repair（code-as-action） |
</details>


---

## 🎥 示範
https://github.com/user-attachments/assets/4ed94cd5-11be-4daa-b2d7-1260a803baca

---

## 📊 效能

在兩個真實網站基準測試中，以 100 步預算達到最先進成果——完整細節請參閱[部落格文章](https://www.microsoft.com/en-us/research/articles/webwright-a-terminal-is-all-you-need-for-web-agents/)。

- 🏆 **Online-Mind2Web（300 個任務）：** 使用 GPT-5.4 達到 **86.7%** —— 在 AutoEval 類別中名列所有開源框架之首。Claude Opus 4.7 達到 **84.7%**，並且在困難切分上表現更強（N=100 時 **80.5%**，GPT-5.4 為 76.6%）。
- 🚀 **Odysseys（200 個長時程任務）：** 使用 GPT-5.4 達到 **60.1%**（平均 76.1 步）—— 相比先前 SOTA（採用 vision-based approach 與持久瀏覽器的 Opus 4.6，44.5%）高出 **15.6 個百分點**，相比基礎 GPT-5.4（使用 xy-coordinate prediction 與持久瀏覽器，33.5%）高出 **26.6 個百分點**。
- 🧠 **以程式碼作為動作勝過座標預測：** Webwright 在所有難度切分中，都明顯優於重現的 GPT-5.4 screenshot+xy-coordinate 基線。
- 🧰 **小模型 + 可重用工具：** 產生出的腳本可以封裝為參數化 CLI 工具——即使是 **Qwen-3.5-9B**，在有 5 個以上工具可用時，也能在 Online-Mind2Web 網站上良好完成任務。

<p align="center">
  <img src="assets/odysseys_eval_step100.png" alt="Odysseys long-horizon eval @ 100 steps" width="49%">
  <img src="assets/om2w_autoeval_step100.png" alt="Online-Mind2Web AutoEval @ 100 steps" width="49%">
</p>

---

## 🗺️ 專案地圖

```text
webwright/
├── pyproject.toml           # 套件：webwright
├── src/webwright/
│   ├── run/cli.py           # CLI 入口點（`webwright`）
│   ├── agents/default.py    # 核心代理迴圈
│   ├── environments/        # Playwright 瀏覽器工作區
│   ├── tools/               # image_qa、self_reflection
│   ├── models/              # openai_model、anthropic_model、base
│   ├── config/              # base.yaml、model_openai.yaml、model_claude.yaml
│   └── utils/
├── assets/
│   └── task_showcase/       # 用於可重複執行 run 的小型 Flask 儀表板
│       ├── app.py
│       ├── templates/       # dashboard.html、task.html
│       └── tasks/<short_id>/ # 每個任務各有 task.json + report.json
├── tests/
└── outputs/                 # 執行產物（trajectories、screenshots）
```

---

## 📰 Task Showcase（以儀表板呈現可重複執行的 runs）

位於 [`assets/task_showcase/`](assets/task_showcase/README.md) 下方的小型 Flask 應用程式，會把 **可重複執行** 的 odyssey 任務（優惠、庫存、列表、求職看板、天氣等）整合到同一個儀表板中。每個任務只需要兩個檔案——`task.json`（中繼資料）與 `report.json`（人工整理的結構化輸出：來源 + 結果區塊，例如表格、清單、摘要）——而模板會以通用方式渲染它們，因此要新增任務，只需把新的資料夾放進 `assets/task_showcase/tasks/`。

```bash
pip install flask
python assets/task_showcase/app.py    # http://127.0.0.1:5005
```

若要讓 Webwright 在執行時產生可直接供 renderer 使用的任務資料夾，可疊加 Task Showcase overlay：

```bash
python -m webwright.run.cli \
    -c base.yaml -c model_openai.yaml -c task_showcase.yaml \
    -t "<可重複執行的網頁任務>" \
    --task-id my_repeatable_task \
    -o outputs/default
```

> **注意：** 只有在包含 `-c task_showcase.yaml` 時才會產生 `report.json`。若只使用 `base.yaml` 執行，會產生 `trajectory.json` 與除錯產物，但不會有 `report.json`。

執行時會在輸出工作區內寫入 `task_showcase/tasks/<short_id>/task.json` 與 `report.json`。直接渲染這些產生出的檔案即可，不必再把它們複製回儲存庫：

```bash
python assets/task_showcase/app.py \
    --tasks-dir outputs/default/<run>/task_showcase/tasks
```

---

## 🚀 快速開始

### 先決條件

- Python 3.10+
- 透過 Playwright 安裝的 Chromium
- 你所選後端（OpenAI、Anthropic 或 OpenRouter）的 API 金鑰

### 安裝

```bash
pip install -e .
playwright install chromium
```

### 執行

匯出所設定後端的憑證（例如搭配 `model_openai.yaml` 時使用 `OPENAI_API_KEY`，或搭配 `model_claude.yaml` 時使用 `ANTHROPIC_API_KEY`）。`image_qa` 與 `self_reflection` 工具預設會使用相同的已配置模型，因此 Anthropic 執行流程不需要額外的 OpenAI 金鑰。接著：

```bash
python -m webwright.run.cli \
    -c base.yaml -c model_openai.yaml \
    -t "Search for flights from SEA to JFK on 2026-08-15 to 2026-08-20" \
    --start-url https://www.google.com/flights \
    --task-id demo_openai \
    -o outputs/default
```

### 🚩 旗標

| 旗標 | 說明 |
|------|-------------|
| `-c` | 來自 `src/webwright/config/` 的設定檔（可堆疊）。 |
| `-t` | 任務指令。 |
| `--start-url` | 初始頁面。 |
| `--task-id` | 輸出子資料夾名稱。 |
| `-o` | 輸出目錄。 |

---

## 🔌 作為外掛使用

Webwright 針對 [Claude Code](https://docs.claude.com/en/docs/claude-code/plugins)（[`.claude-plugin/plugin.json`](.claude-plugin/plugin.json)）與 [OpenAI Codex](https://developers.openai.com/codex/plugins)（[`.codex-plugin/plugin.json`](.codex-plugin/plugin.json)）都提供了外掛資訊清單，共用技能位於 [`skills/webwright/`](skills/webwright/)，slash commands 位於 [`skills/webwright/commands/`](skills/webwright/commands/)。宿主代理會原生驅動 Webwright 迴圈——除了你的宿主訂閱之外，不需要額外的 LLM API 金鑰或成本。若宿主可原生讀取 PNG 截圖，便會略過 `image_qa` / `self_reflection` 工具。

通用的執行期相依套件（任一路徑安裝後只需安裝一次）：

```bash
pip install -e .
playwright install chromium
```

<details>
<summary><b>Claude Code</b></summary>

### 安裝

透過 Claude Code 內建的 marketplace 安裝：

```text
# 1. 將此儲存庫加入 Claude Code 的 plugin marketplace
/plugin marketplace add microsoft/Webwright

# 2. 從該 marketplace 安裝外掛
/plugin install webwright@webwright
```

偏好使用本機 checkout？也可以把 marketplace 指令指向已 clone 的儲存庫：

```text
/plugin marketplace add /absolute/path/to/Webwright
/plugin install webwright@webwright
```

### 使用

安裝後請**啟動新的 Claude Code 工作階段**——外掛會在工作階段啟動時載入，重新啟動前不會出現。

你可以直接用自然語言要求 Claude Code 執行（技能會依描述自動啟用），也可以使用其中一個 slash command：

```
/webwright:run search Google Flights for flights from SEA to JFK on 2026-08-15 to 2026-08-20
/webwright:craft search a ticket on Google Flights from LAX to SFO depart June 7 return June 14
```

- `/webwright:run`（或任何自然語言提示）會為當前任務的具體值產生 **one-shot** `final_script.py`。
- `/webwright:craft` 會產生 **可重用的 CLI 工具**：`final_script.py` 會變成一個參數化函式，附帶 Google 風格的 `Args:` docstring 與 `argparse` 包裝器，旗標預設值就是這次任務的具體內容，因此你之後可以用不同參數重新執行——例如 `python final_script.py --origin JFK --destination LAX --depart-date 2026-07-01`。

在這兩種模式下，Claude Code 都會建立含有 `plan.md` 的工作區，在 `final_runs/run_<id>/` 下執行加上 instrumentation 的 Playwright 腳本，並根據已儲存的截圖，對每個關鍵節點進行視覺化自我驗證。

</details>

<details>
<summary><b>OpenAI Codex</b></summary>

### 安裝

Codex 可讀取 Claude 風格的 marketplace，因此同一個儲存庫也可作為 Codex 的 plugin marketplace。在 Codex CLI 中：

```bash
# 1. 將此儲存庫加入 Codex 的 plugin marketplace
codex plugin marketplace add microsoft/Webwright

# 2. 開啟外掛瀏覽器並安裝 Webwright
codex
/plugins
```

偏好使用本機 checkout？

```bash
codex plugin marketplace add /absolute/path/to/Webwright
```

之後重新啟動 Codex，讓新的 marketplace 與外掛被載入。

### 使用

在新的 Codex 對話中，你可以直接用自然語言要求（技能會依描述自動啟用），或以 `@webwright` 明確呼叫內附技能：

```
@webwright search Google Flights for flights from SEA to JFK on 2026-08-15 to 2026-08-20
```

Codex 會建立含有 `plan.md` 的工作區，在 `final_runs/run_<id>/` 下執行加上 instrumentation 的 Playwright 腳本，並根據已儲存的截圖，對每個關鍵節點進行視覺化自我驗證。

若想在不解除安裝的情況下停用外掛，請將 `~/.codex/config.toml` 中對應條目設為 `enabled = false`，再重新啟動 Codex。

</details>

<details>
<summary><b>🦞 OpenClaw</b></summary>

### 安裝

可直接從本機 checkout 安裝（路徑、封存檔、npm 規格、git repo 或 `clawhub:` 規格都可）：

```bash
openclaw plugins install /absolute/path/to/Webwright
openclaw gateway restart   # 重新載入，讓外掛與技能生效
```

驗證：

```bash
openclaw plugins list | grep webwright
openclaw skills  list | grep webwright   # 應顯示 "✓ ready"
```

### 使用

`webwright` 技能現在可供任何 OpenClaw 代理介面（CLI、Telegram 等）使用——你可以用自然語言請代理執行，或使用位於 [`skills/webwright/commands/`](skills/webwright/commands/) 的 slash commands，例如 `/webwright run <task>`。

若要解除安裝：`openclaw plugins uninstall webwright`。

</details>

<details>
<summary><b>Hermes Agent</b></summary>

### 安裝

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 是相容於 [skills](https://agentskills.io) 的用戶端，因此同一個 `skills/webwright/` 資料夾也能作為 Hermes 技能載入。請將它建立符號連結到你的 Hermes user-skills 目錄：

```bash
mkdir -p ~/.hermes/skills
ln -sfn /absolute/path/to/Webwright/skills/webwright ~/.hermes/skills/webwright
```

不需要 Hermes 專用的 manifest；只會載入 `SKILL.md`。

### 使用

啟動 Hermes（`hermes`）後，以自然語言要求它處理網頁任務——技能會依描述自動啟用。你也可以用 `/webwright` 明確呼叫它。

注意：位於 [`skills/webwright/commands/`](skills/webwright/commands/) 的具名子命令（`/webwright:run`、`/webwright:craft`）是 Claude Code / Codex 的慣例，在 Hermes 中不會生效；但技能本身仍可端到端運作。

</details>

## 📃 Trajectory 比較與檢視器

你可以使用 Webwright harness 與它的 Codex / GitHub Copilot skill 版本來執行同樣的任務，並比較不同 harness 之間的 token 使用量與 trajectories。trajectory viewer 支援 Codex、GitHub Copilot 與 Webwright harness traces。

![Trajectory comparison](assets/trajectory-compare.png)

### 使用方式

```bash
cd assets/compare_trajectory/
python3 -m http.server
```

在瀏覽器中開啟網頁，上傳 Webwright 的 `raw_responses.jsonl` 並附上 `trajectory.json` 以檢視內容。接著在另一側上傳你的 Codex 或 GitHub Copilot trace。

### 取得 Codex traces：

```
ls ~/.codex/sessions/2026/MONTH/DAY/SESSION_ID.jsonl
```

### 取得 GitHub Copilot traces：

```
/export file session
-> session.md is the uploadable trace
```

### 快速比較

#### 「找出 2005-2015 年之間出廠、售價 25,000 到 50,000 美元、里程低於 50,000 英里的最便宜二手 8 缸 BMW。」

| Tokens | Webwright Harness（本機瀏覽器模式） | Codex Webwright Skill |
| --- | ---: | ---: |
| Input | 420,433 | 3,271,143 |
| Output | 3,593 | 20,040 |
| Reasoning | 0 | 4,410 |
| Cached | 217,216 | 3,081,3440 |
| Total | 424,026 | 3,291,183 |

個別執行與結果可能有所差異。

---

## 致謝

- [SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent/tree/main) —— 極簡代理迴圈的設計靈感。
- [Playwright](https://playwright.dev/) —— 瀏覽器自動化。

## 引用

如果你在研究中使用 Webwright，或以此為基礎進行開發，請引用此儲存庫：

```bibtex
@misc{webwright2026,
  title        = {Webwright: A terminal is all you need for web agents},
  author       = {Lu, Yadong and Xu, Lingrui and Huang, Chao and Awadallah, Ahmed},
  year         = {2026},
  howpublished = {\url{https://github.com/microsoft/Webwright}},
  note         = {GitHub repository}
}
```
