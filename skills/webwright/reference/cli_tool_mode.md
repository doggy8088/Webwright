# CLI 工具模式

預設的 Webwright 執行（`/webwright:run`，一般提示詞）會產生一個單次執行的 `final_script.py`，解決使用者提供之具體數值的任務。而 **CLI 工具模式**（`/webwright:craft`）則會產生一個**可重複使用、參數化的 CLI 工具**：同一個腳本可以在稍後使用不同的參數值重新執行，以執行同類型的任務。

此模式改編自 `webwright/src/webwright/config/crafted_cli.yaml` 中的「Final-Script Shape (CLI Tool, MANDATORY)」契約。原本基於 OpenAI 的 `self_reflection` 關卡會被你對照 `plan.md` 的自我驗證所取代。

## 何時使用

在下列情況下觸發 CLI 工具模式：

- 使用者呼叫了 `/webwright:craft …`，或者
- 使用者表示「使其可重複使用」、「參數化」、「做成 CLI」、「我想要用不同的 X 再次呼叫此工具」或類似的說法。

否則，請保持在預設的單次執行模式。

## `plan.md` — 新增 `# Parameters` 區塊

在撰寫腳本之前，除了常見的 `# Critical Points` 檢查清單之外，請識別使用者可能會合理變更的每項需求，並將它們列在 `plan.md` 中：

```markdown
# Task
<原封不動的任務描述>

# Parameters
| name    | type | source phrase from task | default     | allowed / format        |
|---------|------|-------------------------|-------------|-------------------------|
| <arg_a> | str  | "..."                   | "<value>"   | <format / allowed set>  |
| <arg_b> | int  | "..."                   | <value>     | <range or units>        |
| <arg_c> | str  | "..."                   | "<value>"   | <format>                |

# Critical Points
- [ ] CP1: ...
- [ ] CP2: ...
```

規則：

- `# Parameters` 中的每一個項目都必須 (a) 成為函式的參數，並且 (b) 成為 `argparse --flag`，且帶有列出的預設值。
- 對於網站而言真正固定的項目（起始 URL、網站名稱、定位器策略）則**不是**參數——請保持寫死。
- 預設值必須能完全重現原始任務。在無參數的情況下執行 `python final_script.py` 必須能重現該任務。
- 關鍵點（Critical Points）仍是必須的；它們是驗證的契約。

## `final_script.py` — 必要的結構形狀

1. **一個以任務領域命名的可重複使用函式**。例如：
   - `def search_<domain>(arg_a, arg_b, ...): ...`
   - `def lookup_<entity>(query, filters): ...`

2. **Google 風格的文件字串**，包含摘要、完整的 `Args:` 區塊和 `Returns:`。每個 `Args:` 欄位文件需記錄：
   - 參數名稱與型態、
   - 它在任務領域中代表什麼、
   - 接受的格式／單位／允許的值、
   - 預設值（對照 `# Parameters` 表格）。

   ```python
   def search_<domain>(arg_a: str, arg_b: int, arg_c: str) -> dict:
       """<此工具在目標網站上執行之操作的一行摘要>.

       Args:
           arg_a: <代表什麼>; <格式 / 允許的值>.
               Default: "<value>".
           arg_b: <代表什麼>; <範圍 / 單位>.
               Default: <value>.
           arg_c: <代表什麼>; <格式>.
               Default: "<value>".

       Returns:
           dict with keys ``<key1>`` (<type>), ``<key2>`` (<type>),
       """
   ```

3. **`argparse` CLI** 實作於 `if __name__ == "__main__":` 下。每個函式參數都有對應的 `--<arg>` flag，並包含 `type=`、`help=`（複製自文件字串）和 `default=`（等於具體的任務數值）：

   ```python
   if __name__ == "__main__":
       import argparse
       parser = argparse.ArgumentParser(
           description=search_<domain>.__doc__.splitlines()[0])
       parser.add_argument("--arg-a", dest="arg_a", type=str,
                           default="<value>",
                           help="<複製自文件字串>")
       parser.add_argument("--arg-b", dest="arg_b", type=int,
                           default=<value>,
                           help="<複製自文件字串>")
       parser.add_argument("--arg-c", dest="arg_c", type=str,
                           default="<value>",
                           help="<複製自文件字串>")
       args = parser.parse_args()
       result = asyncio.run(_run(**vars(args)))
       print(result)
   ```

4. **匯入時不得產生副作用 (Side-effect-free)**。在模組頂層不得啟動瀏覽器、不得進行網路呼叫、亦不得寫入檔案。該可重複使用的函式必須能夠從另一個 Python 行程匯入，而不會觸發執行。

5. **行動日誌參數回顯。** 在重設後寫入 `final_script_log.txt` 的第一行**必須**是 `step 0 params: ...` 行，將每個解析後的參數列為 `name=value` 對，例如：

   ```
   step 0 params: arg_a=<value> arg_b=<value> arg_c=<value>
   ```

   這樣在任何驗證步驟中，解析後的輸入都是可見的。

6. 使用與預設模式相同的插樁：檢視區 1280×1800、無頭本地 Firefox、無 `full_page=True`、螢幕截圖儲存為 `final_runs/run_<id>/screenshots/final_execution_<step>_<action>.png`、最終數據附加至 `final_script_log.txt`。

## 驗證 (取代 `self_reflection`)

除了預設的自我驗證（`plan.md` 中的每個 CP 都已勾選並附上螢幕截圖／日誌證據）之外，CLI 模式還要求：

1. **在無參數下重現任務。** 在全新的 `final_runs/run_<id>/` 目錄下執行：

   ```bash
   cd final_runs/run_<id> && python final_script.py
   ```

   該執行必須端到端成功，並產生預期的螢幕截圖與 `step 0 params: ...` 日誌行。

2. **匯入安全性冒煙測試。** 從任何其他目錄執行：

   ```bash
   python -c "import importlib.util, pathlib; \
     spec = importlib.util.spec_from_file_location('fs', 'final_runs/run_<id>/final_script.py'); \
     m = importlib.util.module_from_spec(spec); spec.loader.exec_module(m); \
     print([n for n in dir(m) if not n.startswith('_')])"
   ```

   這必須在瞬間完成，且不會啟動瀏覽器，並列印出該可重複使用函式的名稱。

3. **選用：使用不同參數值進行第二次執行。** 證明參數化確實有效。在 `final_runs/run_<id>_alt/` 內執行（或者直接將其日誌／螢幕截圖資料夾儲存於該處）。僅在替代值顯然會失敗時才略過（例如：目標網站不支援的數值）。

4. **列印 `--help`。** 最後向使用者展示：

   ```bash
   python final_runs/run_<id>/final_script.py --help
   ```

## 完成關卡 (CLI 模式)

僅在滿足以下**所有**條件時，才判定任務完成：

1. `plan.md` 同時包含 `# Parameters` 表格（包含名稱、型態、來源字句、預設值、允許值／格式）和 `# Critical Points` 檢查清單。
2. `final_script.py` 定義了剛好一個可重複使用函式，並帶有涵蓋每個參數的 Google 風格 `Args:` 文件字串。
3. 每個 `# Parameters` 項目與函式參數及 argparse `--flag` 呈 1 對 1 對應，且預設值等於具體任務的數值。
4. 腳本具備匯入安全性（冒煙測試通過）。
5. 在 `final_runs/run_<id>/` 內執行 `python final_script.py`（不帶參數）能重現任務；所有 CP 對照儲存的螢幕截圖與行動日誌均已驗證。
6. `final_script_log.txt` 中存在 `step 0 params: ...` 行。
7. 使用者已看到最終數據**以及** `--help` 輸出，從而得知如何以不同的參數再次呼叫此工具。

若有任何一項不符合，請勿宣告完成——請診斷、修正腳本（保留 CLI 形狀），在下一個 `run_<id+1>/` 中重新執行並重新驗證。
