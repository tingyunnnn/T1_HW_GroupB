# 軟體規格書：校園設備報修系統

## 目標與使用情境

本系統以 Flask 提供本機 MIS 課堂展示：報修人於 `/` 填寫設備故障並查看公開案件；管理操作人員於 `/admin` 查看聯絡資訊、自由切換案件狀態及指派維修負責人。「管理員」是操作角色，並不代表系統已驗證其身分。

本規格依 main 的 `app.py`、`templates/`、`static/` 與 `tests/` 整理，沿用 Repository 分支規格中的正式選項、API 與指派規則。安裝及啟動見 [README.md](README.md)，Agent 工作規範見 [AGENTS.md](AGENTS.md)。

## 功能需求

| 編號 | 需求 |
|---|---|
| FR-01 | 地點選單提供 21 個正式選項，順序、穩定 ID 與標籤依 `FORM_OPTIONS.buildings`；不得以 LA、IL 等代碼代替獨立 ID。完整清單保留於 README 的「正式地點」，測試基準為 `tests/expected_options.json`。 |
| FR-02 | 設備提供 projector（投影機）、computer（電腦）、air_conditioner（冷氣）、microphone（麥克風）、network_equipment（網路設備）、other（其他）；問題提供 cannot_power_on（無法開機）、malfunction（無法正常運作）、connection_issue（連線異常）、damaged（設備損壞）、other（其他），依此順序。 |
| FR-03 | 設備／問題選 other 時必填自訂文字；切回一般選項時前端隱藏、清空、停用自訂欄位，後端忽略不適用的自訂內容。 |
| FR-04 | 詳細位置、姓名、Email 必填且須為去除首尾空白後非空的字串；姓名無額外字元或身分驗證。Email 使用共享 EMAIL_PATTERN 格式規則，不限制學校網域，不驗證信箱存在。 |
| FR-05 | 建立案件後回傳編號；前台顯示成功訊息、重設表單並更新清單。緊急程度為一般、急件、緊急。 |
| FR-06 | 五種狀態依序為「待審核、已派工、處理中、等待零件、已結案」，可自由切換。新案件為待審核，assignee 為空字串。 |
| FR-07 | 待審核可無負責人；其他四種狀態必須有去除首尾空白後非空的負責人文字。負責人為自由輸入，無帳號名單驗證。狀態與負責人驗證全部成功才更新。 |
| FR-08 | 公開 GET、POST、PATCH 回應排除 contact、contact_name、contact_email；公開 API 仍含 assignee，前台表格不顯示負責人。 |
| FR-09 | 後台透過無保護的 GET /api/admin/tickets 顯示姓名、Email、地點／設備、問題、時間、狀態、負責人及儲存按鈕。 |
| FR-10 | 後台表格至少寬 1250px，窄螢幕於表格容器水平捲動；案件編號與儲存按鈕不折行。儲存失敗顯示中文錯誤並保留輸入。 |

## API 與資料契約

GET `/` 與 GET `/admin` 回傳 HTML。以下 API 回傳 JSON；POST／PATCH 使用 `Content-Type: application/json`，body 必須是 JSON 物件。未知欄位目前不會另行報錯，也不會作為新增案件欄位或更新目標。

| HTTP 方法與路徑 | 成功 | 失敗 |
|---|---|---|
| GET /api/tickets | 200，公開案件陣列，無資料為 [] | 無自訂錯誤分支 |
| GET /api/admin/tickets | 200，完整案件陣列，包含姓名與 Email | 無登入或權限檢查 |
| POST /api/tickets | 201，{message, ticket} | 400，{message} 中文驗證訊息 |
| PATCH /api/tickets/<ticket_id> | 200，{message, ticket} | 400 格式／驗證錯誤；404 案件不存在；均回傳 {message} |

### 建立案件

| 欄位 | 型別／規則 |
|---|---|
| building | 必填字串，21 個正式地點 ID 之一 |
| classroom | 必填非空字串，教室／詳細位置 |
| equipment、description | 必填字串，分別使用 FR-02 的正式 ID |
| equipment_other、description_other | 對應選項為 other 時必填非空字串，否則儲存為空字串 |
| contact_name | 必填非空字串，虛構報修人姓名 |
| contact_email | 必填字串，須完整符合 app.py 的 EMAIL_PATTERN |
| urgency | 省略時為「一般」；有提供時須為非空字串且為一般／急件／緊急之一 |

上述字串先 strip。Email 接受 `student+tag@sub.example.com`，拒絕空白、`a@localhost`、`a@-example.com`；精確正規表示式以 `app.py` 的 EMAIL_PATTERN 為準，模板使用相同規則及 HTML type=email。

```json
{"building":"liberal_arts_la","classroom":"302","equipment":"projector","description":"cannot_power_on","contact_name":"測試同學","contact_email":"student@example.com","urgency":"一般"}
```

公開 ticket 所有欄位均為字串：`id`、`building`、`classroom`、`equipment`、`description`、`urgency`、`equipment_other`、`description_other`、`equipment_type`、`description_type`、`location`、`status`、`assignee`、`created_at`。後台另含 `contact_name` 與 `contact_email`。

- id 以記憶體案件數加一產生，至少三位數，例如 T-001、T-002；不是永久或跨程序唯一編號。
- equipment／description 為中文標籤或自訂顯示文字；equipment_type／description_type 保留輸入 ID。
- location 為「大樓標籤 + 一個空白 + classroom」。
- created_at 為伺服器本地時間字串 `YYYY-MM-DD HH:MM`，不含時區。

### 更新案件

```json
{"status":"已派工","assignee":"王技師"}
```

status 必填字串，strip 後須屬 FR-06；舊狀態「待處理、已完成、待料中」不接受。assignee 若提供須為字串；省略保留原值，明確傳空字串只可在待審核清除。不能藉此路由修改其他案件欄位。

驗證順序是 JSON 物件 → status 型別／合法值 → 查找案件 → assignee 型別及必填規則 → 同時更新。因此不存在的案件搭配合法 status 回 404，即使 assignee 不合法；不合法 status 優先回 400。驗證失敗不修改任何案件資料。

## 限制與本次不實作範圍

案件只存於 Python 記憶體，重啟即清空，不提供跨程序一致性或持久化保證。系統未提供登入、授權、資料庫、Email 通知、附件／檔案上傳、刪除案件或強制依序流轉。

**現有風險：** `/admin`、後台讀取 API 及 PATCH API 均無身分或權限保護；知道網址者可讀取聯絡資訊及更改狀態。公開回應過濾聯絡資訊不等於完成存取控制。直接執行 app.py 使用 debug=True。只適合本機虛構資料展示，不應將未實作的安全機制寫成已完成，也不得為整理文件擅自新增功能。

## 驗收條件與測試

| 編號 | 可觀察條件 | 驗證來源 |
|---|---|---|
| AC-01 | 21 個地點、6 個設備、5 個問題的順序、ID、標籤一致且無重複，舊值被拒絕 | test_app.py 選項測試、browser.cjs |
| AC-02 | other 缺值回 400，有效值去空白保存；切回一般選項清除、停用且不送自訂欄位 | test_app.py 自訂欄位測試、browser.cjs |
| AC-03 | 空白／錯誤型別姓名、位置與不合法 Email 被拒絕，有效 Email 可建立 | test_app.py 必填與 Email 測試、browser.cjs |
| AC-04 | 有效建立回 201、案件編號、待審核及空負責人；前台成功後重設並更新清單 | test_create_and_public_privacy、browser.cjs |
| AC-05 | 五種狀態可自由切換；非待審核無負責人回 400 且不部分更新；省略保留／空字串清除符合規則 | test_app.py 狀態與指派測試、browser.cjs |
| AC-06 | 公開 GET／POST／PATCH 不含聯絡資訊，後台 GET 含姓名與 Email | test_app.py 隱私與後台測試 |
| AC-07 | 非物件、錯誤狀態回 400；合法 status 搭配不存在案件回 404 | test_app.py 無效資料與更新測試 |
| AC-08 | 後台表頭與欄位數一致；1440、1024、768、375、320px 無整頁橫向溢出，窄螢幕可捲動至儲存並操作 | browser.cjs；人工確認手機閱讀與操作體驗 |

後端執行 `python -m unittest discover -s tests -v`（或 README 的虛擬環境指令）。瀏覽器依 README 啟動獨立 5055 測試伺服器，再執行 `node tests/browser.cjs`，需要 Node.js、Playwright 與 Edge；會新增及更新虛構測試案件。

完成標準：適用測試通過，需求與 AC 可對照，文件路徑有效；未執行、失敗與人工確認事項必須分別回報。自動測試不取代手機視覺確認，也不代表已完成安全防護。
