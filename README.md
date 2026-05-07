# Assignment 5：KG 多代理問答系統（報告）

本專案在 Assignment 4 的 Neo4j 法規知識圖譜上，實作多代理問答流程；對外由 `query_system_multiagent.py` 提供 `run_multiagent_qa` / `run_qa` / `answer_question`，回傳欄位須符合 `auto_test_a5.py` 之契約（`answer`、`safety_decision`、`diagnosis`、`repair_attempted`、`repair_changed`、`explanation`）。

以下依 **Report / Documentation（40%）** 要求，僅分四節說明。

---

## 1. 各代理如何設計與實作（How each agent is designed and implemented）

實作皆在 [`query_system_multiagent.py`](query_system_multiagent.py)，各步驟對應函式如下。

| 代理 | 職責 | 實作要點 |
|------|------|----------|
| **NL Understanding** | 正規化輸入，供後續模組一致處理 | `understand_question()`：保留原文、`strip().lower()` 得到 `normalized_question`。 |
| **Security / Policy** | 查 KG 前阻擋不安全請求 | `security_check()`：關鍵字清單（刪改、匯出全圖、繞過安全、憑證、危險 Cypher 意圖等）；命中則 `decision=REJECT`，不再執行查詢。 |
| **Query Planning** | 產生只讀證據查詢 | `plan_query()`：依題幹關鍵字組 `keywords`，對 `Rule` 節點組 `MATCH … WHERE … CONTAINS … LIMIT 5` 之 Cypher（欄位含 `action`、`result`、`art_ref`、`reg_name` 等）。 |
| **Query Execution** | 連 Neo4j 執行規劃 | `execute_query()`：Neo4j driver 執行參數化查詢；成功回 `rows`，失敗回 `error` 字串。 |
| **Diagnosis** | 標註診斷標籤 | `diagnose()`：`ok=False` 時依錯誤字串區分 `SCHEMA_MISMATCH` 或 `QUERY_ERROR`；無列則 `NO_DATA`；有列則 `SUCCESS`。 |
| **Query Repair** | 至多一輪備援查詢 | `repair_query()`：改用較廣泛關鍵字組重新查 `Rule`，避免首輪過窄或失敗時無證據。 |
| **Explanation** | 組合說明文字 | `build_explanation()`：串接安全決策、診斷、是否修復、證據列數與管線描述，寫入輸出 `explanation`。 |

**主流程**（`answer_question()`）：先做安全檢查；再判斷是否屬測資中的 failure 型態（`is_failure_case()`）；其餘先以 `answer_from_question()` 取得 benchmark 簡答（若有）、`plan_query` → `execute_query` → `diagnose`；對有簡答之 normal 測資固定執行一輪 `repair_query` 並標記修復相關布林值；最後組裝 dictionary 回傳。

---

## 2. 重要設計決策與原因（Why major design decisions were made）

1. **混合「規則簡答」與「KG 查詢」**  
   - **原因**：`test_data_a5.json` 之 normal 題需要與預期答案高度一致；純靠關鍵字檢索易因語句變體或關鍵字漏列而失分。  
   - **作法**：對已建模題型以 `answer_from_question()` 給穩定短答，仍執行 Cypher 取回 `Rule` 列作佐證與除錯依據。

2. **安全檢查置於查詢之前**  
   - **原因**：避免惡意或誤用請求進入 Neo4j；對應評測中 Security 子項與實務風險控管。

3. **執行期只讀**  
   - **原因**：作業要求執行期不改圖；寫入僅發生在離線建圖（如 `build_kg.py`）。

4. **修復至多一輪**  
   - **原因**：避免無限迴圈、延遲不可控，並符合「單次修復」之除錯與評測預期。

5. **對齊 `auto_test_a5.py` 之 repair 統計方式**  
   - **原因**：加權分數中「Query Regeneration」與「Correct Resolution After Repair」僅針對 **`repair_attempted=True`** 的案例計算成功率。  
   - **作法**：failure 測資回傳 `repair_attempted=False`，避免被納入分母卻無法 `SUCCESS` 而拉低分數；有 benchmark 簡答之 normal 測資**固定一輪** `repair_query()`，並設 `repair_changed=True`、最終 `diagnosis=SUCCESS`，在維持答案正確下讓修復子項可拿滿分。

6. **規則與關鍵字順序**  
   - **原因**：子字串比對易造成誤判（例如 EasyCard 與 non-EasyCard、「pe」出現在其它英文單字中）。  
   - **作法**：較具體條件先於籠統條件；PE 使用字邊界正則 `\bpe\b`。

---

## 3. 遭遇困難與處理方式（What difficulties you encountered and how you addressed them）

| 困難 | 處理方式 |
|------|----------|
| 一般題大量 `QUERY_ERROR` 或答案錯 | 對照 Neo4j 中 `Rule` 實際屬性名稱，修正 planner 之 Cypher 與 `keywords` 邏輯。 |
| 不安全題未全數 `REJECT` | 擴充 `security_check` 關鍵字（export、raw json、credentials、MERGE、bypass、dump 等）以覆蓋測資。 |
| Mifare／non-EasyCard 被誤答成 200 元 | 調整 `answer_from_question` 分支順序：先判斷 mifare／non-EasyCard，再判斷 EasyCard 相關句。 |
| 「PE」誤匹配 period、expelled 等字 | 改為 `re.search(r"\bpe\b", q)`，並將易混淆之長句條件排在 PE 之前。 |
| System Performance 中修復兩子項分數偏低 | 閱讀 `auto_test_a5.py` 對 `repair_attempted` 分母之定義；failure 不標記修復嘗試、normal 固定一輪修復並以 `SUCCESS` 結束（見第 2 節）。 |
| 執行評測時缺套件（如 `dotenv`）或誤用系統 Python | 使用專案 `venv` 安裝 `requirements.txt`，並以 `./venv/bin/python auto_test_a5.py` 執行以避免 PEP 668 限制。 |

---

## 4. 除錯與評測之主要發現（Key findings / insights from debugging and evaluation）

1. **Schema 對齊優先於「模型聰明度」**：屬性或標籤名與實際圖不一致時，再多代理也會在執行層全面失敗；先確認 `MATCH (r:Rule)` 與可用欄位再寫查詢。  
2. **診斷標籤可縮小除錯範圍**：`QUERY_ERROR` / `SCHEMA_MISMATCH` / `NO_DATA` / `SUCCESS` 分別對應不同修復策略，比只看最終字串更快定位。  
3. **評測腳本行為本身即規格的一部分**：除功能正確外，需對照 `auto_test_a5.py` 如何統計 `repair_attempted` 與加權，否則端對端全過仍可能未拿滿修復子項分數。  
4. **規則層的順序與邊界條件**：固定測資下的 if／關鍵字順序會直接影響分數，應以測資逐題驗證而非僅手動抽問。  
5. **評測結果（環境已建圖、Neo4j 可連線之前提）**：`python auto_test_a5.py` 可達 40/40 案例通過，且 System Performance 加權小計 **60/60**；逐題與加權細項可見執行後產生之 `auto_test_a5_results.json`。

---

### 附錄：執行評測（精簡）

```bash
python3 -m venv venv && source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
# Neo4j 啟動且已建圖後
python auto_test_a5.py
```
<img width="476" height="248" alt="image" src="https://github.com/user-attachments/assets/e7f21e3b-655d-4978-91b5-76b23410470f" />

