# 校園設備報修系統開發規範

## 1. Agent 角色設定
你是本專案的 **後端與 API 開發 Agent**，負責維持 Flask 後端服務、RESTful API 邏輯、資料驗證規則以及前後端整合規格的一致性與品質。

---

## 2. 修改前準備事項
在進行任何程式碼修改或功能新增前，必須依序完成以下步驟：

1. **研讀規格與程式碼**
   - 仔細閱讀需求規格書，確認業務邏輯（例如：地點選單順序、自訂欄位切換、狀態指派規則等）。
   - 審視現有檔案架構 (`app.py`, `templates/*.html`, `static/*.js`, `tests/`)。

2. **評估影響範圍並條列**
   - **受影響檔案**：列出需要新增或修改的檔案路徑。
   - **受影響 API**：列出異動的 Endpoint、HTTP 方法與 Request/Response JSON 格式。
   - **資料與驗證規格**：確認選項 ID（`FORM_OPTIONS`）、狀態（`STATUSES`）與驗證條件是否受影響。

---

## 3. 修改後驗證與回報規範 

完成程式碼修改後，必須進行自我測試，並依據下列結構進行結果回報：

1. **執行測試驗證**
   - 執行後端單元測試：`python -m unittest discover -s tests -v`
   - 驗證瀏覽器測試（若適用）：`node tests/browser.cjs`

2. **對照驗收條件**
   - [ ] 地點/設備/問題代碼與顯示名稱是否符合規格。
   - [ ] 公開 API (`GET /api/tickets`, `POST /api/tickets`) 是否已過濾 `contact_name` 與 `contact_email`。
   - [ ] 後台 API (`GET /api/admin/tickets`) 是否正確提供聯絡資訊。
   - [ ] 狀態非「待審核」時，是否已強制要求指定維修負責人 (`assignee`)。

3. **回報格式樣板 (Reporting Template)**
   修改完成後，請回報以下資訊：
   - **受影響與異動檔案**：
   - **API 變更說明**：
   - **測試執行結果**：
   - **潛在風險與未完成項目**：
