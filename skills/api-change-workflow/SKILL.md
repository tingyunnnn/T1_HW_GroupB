---
name: api-change-workflow
description: 修改或驗證校園設備報修系統的 Flask API、JSON 資料契約與驗證規則時，執行影響分析、測試及問題分類。
---

# API 修改與驗證流程

## 何時使用

用於本專案 API 路由、Request/Response、狀態指派或資料驗證的修改與異常排查。純文件排版不需啟用；檔案上傳不是現有功能，也不是此 Skill 的工作範圍。

## 輸入與前置條件

取得使用者的變更目標、預期行為與允許範圍，閱讀 [software-spec.md](../../software-spec.md) 及 [AGENTS.md](../../AGENTS.md)。檢查 Git 狀態與工作分支；不得在未確認下合併 main。以 app.py、呼叫端及 tests 的現況為依據，不假定規格中的期望已實作。測試使用虛構姓名與 Email；後台 API 與 PATCH 目前沒有權限保護。

## 執行步驟

1. **讀規格與確認差異**：找出相關 FR／AC、目前路由與重現方式，區分使用者要求與既有行為。缺少會改變 API 契約的必要需求時，先請人工確認該部分。
2. **找影響範圍**：列出 app.py、適用的 static/app.js 或 static/admin.js、templates、tests 與規格文件；比較修改前後 Endpoint、HTTP 方法、JSON 型別／欄位、錯誤碼、公開與後台欄位差異。僅修改任務授權的檔案。
3. **選擇驗證案例**：先沿用既有測試；行為變更時補足能捕捉回歸的案例，不為純文件變更新增程式測試。覆蓋成功、錯誤型別、缺值、邊界及失敗不改資料。
4. **最小範圍修改**：維持共用 FORM_OPTIONS／STATUSES 與前後端一致；先驗證再寫入，保持 public_ticket 過濾。不得順便加入登入、資料庫、通知或上傳。
5. **執行測試與檢查差異**：於專案根目錄執行 `python -m unittest discover -s tests -v`；影響前端時依 README 在獨立 5055 伺服器執行 `node tests/browser.cjs`。對照適用 AC，檢查 diff 與文件連結，記錄實際結果；測試失敗不得宣稱完成。
6. **分類並回報**：按以下界線列出阻擋與人工確認，附上可審查的分支差異。

## 本專案驗證範例：狀態與負責人

以 POST /api/tickets 建立虛構案件並取得回應 id，再 PATCH /api/tickets/<ticket_id>，不要假定已有 T-001。

| 請求／條件 | 應驗證結果 |
|---|---|
| {"status":"已派工","assignee":" 王技師 "} | 200，兩欄同時更新，負責人為王技師 |
| 非待審核搭配空白或非字串 assignee | 400，原案件所有資料保持不變 |
| 已有負責人，省略 assignee | 保留原值；status 仍須提供 |
| {"status":"待審核","assignee":""} | 200，清除負責人 |
| 五種合法狀態任意跳轉，提供有效負責人 | 可成功，沒有依序限制 |
| 非物件、缺 status、舊狀態已完成／待料中 | 400 |
| 不存在 id 且 status 合法 | 404；查找案件早於 assignee 驗證 |
| 成功 POST／PATCH 及 GET /api/tickets | 不含 contact_name、contact_email、contact |
| GET /api/admin/tickets | 含姓名與 Email，明確標示未受保護 |

依 tests/test_app.py 的 assignee、atomic failure、free transitions 與 privacy 案例驗證，另檢查介面錯誤保留輸入、按鈕及窄螢幕表格（如受影響）。

## 阻擋問題與人工確認

- **阻擋問題**：測試失敗、資料被部分更新、公開回應洩漏聯絡資訊，或缺少決定契約的必要需求。修復或釐清後才宣稱該變更完成；可先繼續不依賴該問題的工作。
- **驗證受阻**：Python、Flask、Playwright 或 Edge 不可用時，列出失敗指令、缺少條件與未驗證 AC；不得視為測試通過。
- **需人工確認**：手機閱讀與操作體驗、超出既有需求的產品決策、部署前的權限風險，以及是否合併 main。將既有無保護 API 的風險交代清楚，不擅自擴大本次修改。

## 最終輸出

依 AGENTS.md 回報：修改摘要；受影響檔案及 API 契約差異；對應 FR／AC；測試指令與通過／失敗／未執行結果；阻擋問題；風險與人工確認事項；工作分支及 PR。若只驗證而未修改，也應明確說明。
