# TOP Grants — 學門召集人平台 API 規格書

**作者:** 張智奇 (Saw S. Lin)，由 Gemini 和 Claude 輔助。

**適用範圍:** 僅涵蓋學門召集人 (Discipline Convener – 簡稱“DC”) 客戶端。後台管理端 (Admin) 和申請人端 端點不在範圍內。

**Base URL:** `/api/v1`

**版本控制:** URL 路徑版本控制 (`/v1`, `/v2`, …)。破壞性變更會增加主版本號。目前所有開發都針對 `v1`。

**假設:**

- 將會有一個 API Gateway 負責整合 `phdCandidate` 和 `youngScholar` 後端，並能透過 `applicationId` 識別申請案屬於哪個組別。
- 如果上述前提不成立，我們將需要採用以下模式：`/api/v1/phdCandidate/applications/...` 和 `/api/v1/youngScholar/applications/...`。

---

## 1. 設計原則

- 除了 `POST /sessions` 和 `POST /sessions/refresh` 外，每個端點都需要 `Authorization: Bearer <sessionToken>` 標頭。
- 只有在所有申請作業關閉後，本系統才可使用。在此窗口開啟前，針對學門召集人事務的“寫入”操作將被拒絕。
- 首頁/歡迎頁資料在單一請求中載入 (`GET /users/me/grant-cycle`) 以減少來回次數。每個申請案的初審委員詳細資訊 (分數、評論) 在需要時獲取。
- 每個對 `/applications/{id}/reviewer-recommendations` 的 `POST` 請求都會 **附加** 一個新的推薦紀錄。它不會替換先前的紀錄。
- 後端計算並擁有每個申請案的 `status` (狀態)。DC 使用者無法直接設定它。
- `POST` 建立新的子資源。`PUT` 替換現有資源 (冪等)。`DELETE` 刪除資源。

---

## 2. 通用介面

### 2.1 基礎回應封包

所有 API 回應共用此結構：

```json
{
  "status": "success | error",
  "code": "integer (HTTP 狀態碼)",
  "timestamp": "string (ISO 8601)",
  "apiVersion": "string (例如 'v1')",
  "data": "object | array | null",
  "error": "object | null"
}
```

- 成功時：`data` 包含負載；`error` 為 `null`。
- 錯誤時：`data` 為 `null`；`error` 包含錯誤物件。

### 2.2 錯誤物件

```json
{
  "type": "string (下方 ErrorType 的其中一個值)",
  "message": "string (易讀的摘要)",
  "details": [
    {
      "field": "string | null (失效欄位的點記法路徑)",
      "message": "string"
    }
  ]
}
```

`details` 可以是空陣列。用於驗證錯誤以列舉所有失敗的欄位。

### 2.3 ErrorType 參考

| ErrorType               | 使用時機/易讀摘要                                         |
| ----------------------- | --------------------------------------------------------- |
| `VALIDATION_ERROR`      | 請求主體或查詢參數未通過驗證                              |
| `AUTHENTICATION_ERROR`  | 憑證遺失、過期或無效                                      |
| `CAPTCHA_ERROR`         | CAPTCHA 憑證遺失、無效、分數過低或過期                    |
| `AUTHORIZATION_ERROR`   | 經驗證的使用者缺乏此資源的權限                            |
| `NOT_FOUND`             | 請求的資源不存在                                          |
| `CONFLICT`              | 請求與目前的資源狀態衝突 (例如：在所有初審完成前進行決審) |
| `QUERY_EXECUTION_ERROR` | 資料庫查詢失敗                                            |
| `TIMEOUT_ERROR`         | 後端操作超時                                              |
| `RATE_LIMIT_ERROR`      | 來自客戶端的請求過多                                      |
| `INTERNAL_SERVER_ERROR` | 意外的伺服器端錯誤                                        |

---

## 3. HTTP 狀態碼

| 代碼 | 意義                  | 使用時機                                |
| ---- | --------------------- | --------------------------------------- |
| 200  | OK                    | 成功的 GET, PUT, DELETE                 |
| 201  | Created               | 成功建立資源的 POST                     |
| 202  | Accepted              | 請求已被接受以進行非同步處理            |
| 400  | Bad Request           | 驗證錯誤、格式錯誤的請求或 CAPTCHA 失敗 |
| 401  | Unauthorized          | 缺少或無效的 session 憑證               |
| 403  | Forbidden             | 已驗證但不允許訪問                      |
| 404  | Not Found             | 資源不存在                              |
| 408  | Request Timeout       | 查詢超時                                |
| 409  | Conflict              | 狀態衝突 (請參閱 §5 狀態機)             |
| 429  | Too Many Requests     | 超出頻率限制                            |
| 500  | Internal Server Error | 意外的後端錯誤                          |

---

## 4. 身分驗證

### 4.1. CAPTCHA

**由誰處理 CAPTCHA:** 這是一項分攤的責任。

- **前端** 載入 CAPTCHA 供應商的腳本，在登入表單送出時呼叫它以取得一次性的 `captchaToken`，並將該憑證與登入資訊一起發送。
- **後端** 從不產生或渲染 CAPTCHA 挑戰。它接收 `captchaToken`，以伺服器對伺服器的方式進行驗證 (對 CAPTCHA 供應商的驗證 API 進行私人 POST)，並據此接受或拒絕登入。

**供應商 (AI 推薦):** [Google reCAPTCHA v3](https://developers.google.com/recaptcha/docs/v3) (隱形、基於分數 — 無需使用者互動) 或 [hCaptcha](https://www.hcaptcha.com) (隱私優先且 API 行為等效的替代方案)。

> ⚠️ **未決決定:** 在實作前確認 CAPTCHA 供應商。前端和後端必須使用來自同一供應商的相符 Site key (公開、前端) 和 Secret key (私人、後端)。

**reCAPTCHA v3 憑證流程:**

1. 前端載入 `<script src="https://www.google.com/recaptcha/api.js?render=SITE_KEY"></script>`。
2. 在登入送出時：呼叫 `grecaptcha.execute(SITE_KEY, { action: 'login' })` → 解析為 `captchaToken`。
3. 將 `captchaToken` 包含在 `POST /sessions` 請求主體中。
4. 後端呼叫 `POST https://www.google.com/recaptcha/api/siteverify`，帶上憑證和 secret key。
5. 後端檢查 `score >= 0.5` (閾值可配置)。如果不是，則回傳 `400 CAPTCHA_ERROR`。

---

### `POST /sessions` — 登入

不需要 `Authorization` 標頭。

**請求主體:**

```json
{
  "username": "string",
  "password": "string",
  "captchaToken": "string"
}
```

**回應 (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "sessionToken": "string (JWT)",
    "refreshToken": "string",
    "userId": "string (UUID)",
    "expiresAt": "string (ISO 8601)"
  },
  "error": null
}
```

**錯誤:**

| 代碼 | ErrorType              | 條件                                          |
| ---- | ---------------------- | --------------------------------------------- |
| 400  | `VALIDATION_ERROR`     | `username` 或 `password` 遺失                 |
| 400  | `CAPTCHA_ERROR`        | `captchaToken` 遺失、無效、過期或分數低於閾值 |
| 401  | `AUTHENTICATION_ERROR` | 登入資訊不正確                                |
| 429  | `RATE_LIMIT_ERROR`     | 來自此 IP/使用者的失敗登入嘗試過多            |

---

### `DELETE /sessions` — 登出

**標頭:** `Authorization: Bearer <sessionToken>`

**請求主體:** 無。

**回應 (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": null,
  "error": null
}
```

**錯誤:**

| 代碼 | ErrorType              | 條件               |
| ---- | ---------------------- | ------------------ |
| 401  | `AUTHENTICATION_ERROR` | 憑證已遺失或已失效 |

---

### `POST /sessions/refresh` — 刷新 Session 憑證

延長 session 而無需重新登入。前端應主動在 `expiresAt` 前呼叫此項 (例如：過期前 5 分鐘)。不需要 `Authorization` 標頭；使用登入時核發的 `refreshToken`。

**請求主體:**

```json
{
  "refreshToken": "string"
}
```

**回應 (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "sessionToken": "string (JWT)",
    "refreshToken": "string (已輪換 — 先前的憑證立即失效)",
    "expiresAt": "string (ISO 8601)"
  },
  "error": null
}
```

> 刷新憑證每次使用時都會輪換，以減輕重放攻擊 (replay attacks)。

**錯誤:**

| 代碼 | ErrorType              | 條件                            |
| ---- | ---------------------- | ------------------------------- |
| 401  | `AUTHENTICATION_ERROR` | `refreshToken` 遺失、無效或過期 |

---

### 經過身分驗證的請求標頭

所有端點皆需要，除了 `POST /sessions` 和 `POST /sessions/refresh` 外：

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

---

## 5. 申請案狀態機 (State Machine)

後端計算並擁有每個申請案的 `status` (狀態)。DC 無法直接設定；它會根據審查委員、管理員的事件以及 DC 自身的操作而改變。

### 5.1. 狀態值

| 值                       | 顯示文字 (ZhTW) | 意義                                                                                                                  |
| ------------------------ | --------------- | --------------------------------------------------------------------------------------------------------------------- |
| `PENDING_RECOMMENDATION` | 待推薦          | DC 尚未提交任何推薦委員，**或** 尚未有任何推薦紀錄。                                                                  |
| `PENDING_INITIAL_REVIEW` | 待初審          | 至少存在一筆推薦紀錄。可能尚無任何已確認的委員。或者如果有的話，並非所有已確認的委員都已完成審查。DC 無已儲存的草稿。 |
| `PENDING_FINAL_REVIEW`   | 待複審          | 至少有 `minNumInitialReviewers` 個已確認的委員提交了他們的審查結果。DC 尚未儲存任何草稿。                             |
| `DRAFT_FINAL_REVIEW`     | 待送出          | DC 已儲存決審草稿 (`isFinalized = false`)。初審可能完成也可能未完全完成；直到完成前將被阻擋送出。                     |
| `COMPLETED`              | 已完成          | DC 已提交並完成決審 (`isFinalized = true`)。                                                                          |

### 5.2. 狀態轉換條件

```
PENDING_RECOMMENDATION
  ─► PENDING_INITIAL_REVIEW
        時機：DC 提交至少一名推薦委員。
  ─► DRAFT_FINAL_REVIEW
        時機：DC 在提交任何推薦前先儲存了一份草稿。
              [邊緣情況 — 不常見但允許]

PENDING_INITIAL_REVIEW
  ─► PENDING_FINAL_REVIEW
        時機：至少 `minNumInitialReviewers` 名已確認的委員的 reviewStatus = "已完成"
              且 DC 無已儲存草稿
  ─► DRAFT_FINAL_REVIEW
        時機：DC 在最低要求的初審數量完成之前先儲存了一份草稿。

PENDING_FINAL_REVIEW
  ─► PENDING_INITIAL_REVIEW  (退回)
        時機：已完成的初審被撤銷/作廢，且已完成的數量降至 `minNumInitialReviewers` 以下。
  ─► DRAFT_FINAL_REVIEW
        時機：DC 儲存草稿 (PUT /final-review, isFinalized = false)
  ─► COMPLETED
        時機：DC 在沒有先前草稿的情況下完成 (PUT /final-review, isFinalized = true)

DRAFT_FINAL_REVIEW
  ─► COMPLETED
        時機：DC 提交 (PUT /final-review, isFinalized = true)
              並且至少 `minNumInitialReviewers` 名已確認的委員的 reviewStatus = "已完成"
              如果不符合此條件，後端必須以 409 CONFLICT 拒絕。
```

### 5.3. 業務規則

1. DC **可以儲存草稿** (`isFinalized = false`) 在任何時候，即使在所有初審完之前。
2. DC **無法決定送出 (finalize)** (`isFinalized = true`) 除非至少 `minNumInitialReviewers` 名初審委員已完成。後端必須使用 `409 CONFLICT` 強制執行。
3. 一旦申請案達到 `COMPLETED`，就不接受進一步的編輯。隨後的 `PUT /final-review` 呼叫必須被拒絕，回傳 `409 CONFLICT`。
4. 如果一位已經完成審查 (`reviewStatus = "已完成"`) 的委員後來在管理層面被撤銷 (極端情況)，後端必須相應地重新評估申請案的狀態。

---

## 6. 端點

### 6.1. `GET /users/me/grant-cycle` — 首頁和歡迎頁面資料

回傳渲染歡迎和主頁以及啟用所有 DC 使用者互動所需要的所有資料。單一回應中包含這兩個組別以及所有被分配的學門。每個申請案的初審委員詳細資訊 (分數、評論) 透過 §6.2 分開獲取。

**請求主體:** 無。

**回應 (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "grantCycleId": "string (UUID)",
    "year": "integer",
    "deadlineDateTime": "string (ISO 8601, 必須包含時間和 UTC 偏移)",
    "applicationTracks": {
      "phdCandidate": {
        "fixedQuota": "integer (核定名額)",
        "evaluationRubrics": [
          {
            "title": "string",
            "fullScore": "integer"
          }
        ],
        "disciplines": [
          {
            "name": "string (必須在這個組別和週期中唯一)",
            "minNumInitialReviewers": "integer"
          }
        ],
        "applications": [
          {
            "id": "string (UUID)",
            "disciplineAppliedFor": "string (與上面 disciplines 列表中的學門名稱相符)",
            "submittedDateTime": "string (ISO 8601)",
            "status": "PENDING_RECOMMENDATION | PENDING_INITIAL_REVIEW | PENDING_FINAL_REVIEW | DRAFT_FINAL_REVIEW | COMPLETED",
            "nameZhTW": "string",
            "nameEn": "string",
            "gender": "string",
            "email": "string",
            "currentInstitutionAndDepartmentZhTw": "string",
            "currentInstitutionAndDepartmentEn": "string",
            "doctoralThesisTitleZhTw": "string",
            "doctoralThesisTitleEn": "string",
            "thesisAbstractZhTw": "string",
            "thesisAbstractEn": "string",
            "expectedImpact": "string",
            "applicantEduBackground": [
              {
                "institution": "string",
                "department": "string",
                "country": "string",
                "degree": "string",
                "graduationStatus": "string",
                "startYearAndMonth": "string (YYYY/MM)",
                "endYearAndMonth": "string (YYYY/MM) | null (如果目前在學則為 null)"
              }
            ],
            "attachments": {
              "researchProposal": { "fileName": "string", "fileUrl": "string" },
              "publicationList": { "fileName": "string", "fileUrl": "string" },
              "phdCandidacyCertificate": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "phdTranscripts": { "fileName": "string", "fileUrl": "string" },
              "publishedWorks": [
                {
                  "displayName": "string",
                  "fileName": "string",
                  "fileUrl": "string"
                }
              ]
            },
            "advisors": [
              {
                "name": "string",
                "jobTitle": "string",
                "institutionAndDepartment": "string",
                "email": "string",
                "eduBackground": "string",
                "workExperience": "string",
                "evaluationOfApplicant": {
                  "researchPotential": "string",
                  "thesisContent": "string",
                  "thesisAdvisingMethod": "string",
                  "thesisProgressInPercentages": "integer (0–100)"
                }
              }
            ],
            "recommendationRecords": [
              {
                "id": "string (UUID)",
                "createdDateTime": "string (ISO 8601)",
                "reviewers": [
                  {
                    "priority": "integer (從 1 開始；較小 = 較高優先級)",
                    "name": "string",
                    "email": ["string"],
                    "remarks": "string | null (DC 推薦此委員的理由)"
                  }
                ]
              }
            ],
            "finalReview": {
              "score": "integer | null",
              "remarks": "string | null",
              "isFinalized": "boolean",
              "updatedDateTime": "string (ISO 8601) | null (如果未發生存檔則為 null)"
            }
          }
        ]
      },
      "youngScholar": {
        "fixedQuota": "integer",
        "evaluationRubrics": [
          {
            "title": "string",
            "fullScore": "integer"
          }
        ],
        "disciplines": [
          {
            "name": "string",
            "minNumInitialReviewers": "integer"
          }
        ],
        "applications": [
          {
            "id": "string (UUID)",
            "disciplineAppliedFor": "string",
            "submittedDateTime": "string (ISO 8601)",
            "status": "PENDING_RECOMMENDATION | PENDING_INITIAL_REVIEW | PENDING_FINAL_REVIEW | DRAFT_FINAL_REVIEW | COMPLETED",
            "nameZhTW": "string",
            "nameEn": "string",
            "gender": "string",
            "email": "string",
            "currentInstitutionAndDepartmentZhTw": "string",
            "currentInstitutionAndDepartmentEn": "string",
            "publicationTitleEn": "string",
            "publicationAbstractZhTw": "string",
            "publicationAbstractEn": "string",
            "expectedImpact": "string",
            "applicantEduBackground": [
              {
                "institution": "string",
                "department": "string",
                "country": "string",
                "degree": "string",
                "graduationStatus": "string",
                "startYearAndMonth": "string (YYYY/MM)",
                "endYearAndMonth": "string (YYYY/MM) | null"
              }
            ],
            "applicantWorkExperience": [
              {
                "institution": "string",
                "department": "string",
                "jobTitle": "string",
                "startYearAndMonth": "string (YYYY/MM)",
                "endYearAndMonth": "string (YYYY/MM) | null"
              }
            ],
            "attachments": {
              "researchProposal": { "fileName": "string", "fileUrl": "string" },
              "publicationList": { "fileName": "string", "fileUrl": "string" },
              "doctoralDegreeCertificate": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "employmentProof": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "supervisorRecommendationLetter": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "publishedWorks": [
                {
                  "displayName": "string",
                  "fileName": "string",
                  "fileUrl": "string"
                }
              ]
            },
            "recommendationRecords": [
              {
                "id": "string (UUID)",
                "createdDateTime": "string (ISO 8601)",
                "reviewers": [
                  {
                    "priority": "integer",
                    "name": "string",
                    "email": ["string"],
                    "remarks": "string | null"
                  }
                ]
              }
            ],
            "finalReview": {
              "score": "integer | null",
              "remarks": "string | null",
              "isFinalized": "boolean",
              "updatedDateTime": "string (ISO 8601) | null"
            }
          }
        ]
      }
    }
  },
  "error": null
}
```

> **實作備註:**
>
> - 如果 DC 僅被分配到一個組別，則另一個組別的 `applications` 陣列將為空 (而非完全沒有)。
> - `recommendationRecords` 按 `createdDateTime` 降序排序 (最近的記錄排在最前面)。
> - `disciplines` — 後端必須強制要求在每個組別和週期中名稱唯一。
> - `youngScholar` 申請案沒有 `advisors`；對於此組別，請完全省略該欄位。

**錯誤:**

| 代碼 | ErrorType               | 條件                                          |
| ---- | ----------------------- | --------------------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | 無效或過期的憑證                              |
| 403  | `AUTHORIZATION_ERROR`   | 使用者不是 DC，或在有效的週期中沒有分配到學門 |
| 404  | `NOT_FOUND`             | 目前沒有進行中的補助週期                      |
| 500  | `INTERNAL_SERVER_ERROR` | 意外的後端失敗                                |

---

### 6.2. `GET /applications/{applicationId}/initial-reviewers` — 初審委員詳情

傳回特定申請案且經過確認的初審委員和他們評論的細節。當 DC 開啟申請摘要會呼叫此 API。

**路徑參數:**

| 參數            | 類型        | 說明        |
| --------------- | ----------- | ----------- |
| `applicationId` | UUID string | 申請案的 ID |

**請求主體:** 無。

**回應 (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "applicationId": "string (UUID)",
    "initialReviews": [
      {
        "reviewerName": "string",
        "reviewerEmail": ["string"],
        "reviewStatus": "未完成 | 已完成",
        "reviewDetails": {
          "statusUpdatedDateTime": "string (ISO 8601) | null",
          "scores": ["integer | null"],
          "remarks": "string | null"
        }
      }
    ]
  },
  "error": null
}
```

> **實作備註:**
>
> - `initialReviews` 僅包含 由承辦人員確認 的初審委員。
> - 僅當 `reviewStatus = '已完成'` 時，`reviewDetails` 為非 null。若狀態為 `未完成` 則 `reviewDetails` 會傳回 `null`。
> - `scores` 陣列長度和該組的 `evaluationRubrics` 評分標準項目一樣，依據評分順序排序。

**錯誤:**

| 代碼 | ErrorType               | 條件                             |
| ---- | ----------------------- | -------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | 無效或過期的憑證                 |
| 403  | `AUTHORIZATION_ERROR`   | 該申請案不隸屬所分配 DC 的學門下 |
| 404  | `NOT_FOUND`             | 找不到該申請案                   |
| 500  | `INTERNAL_SERVER_ERROR` | 意外的後端錯誤                   |

---

### 6.3. `POST /applications/{applicationId}/reviewer-recommendations` — 提交推薦委員

在此申請案下創建新的推薦紀錄，每個呼叫都會往推薦的歷史附加新的紀錄。後端在伺服器端自動存下 `createdDateTime`，不會採用從客戶端帶來的資訊。

**路徑參數:**

| 參數            | 類型        | 說明        |
| --------------- | ----------- | ----------- |
| `applicationId` | UUID string | 申請案的 ID |

**請求主體:**

```json
{
  "reviewers": [
    {
      "priority": "integer (從1開始，數字越小優先順序越高)",
      "name": "string",
      "email": ["string"],
      "remarks": "string | null"
    }
  ]
}
```

**驗證條件:**

- `reviewers` 陣列不可為空。
- `priority` 的值於請求時需為唯一，並且從 1 開始為連續順序。
- 每個委員至少需要一項 `email` 欄位的值。

**回應 (`201 Created`):**

```json
{
  "status": "success",
  "code": 201,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "recommendationRecordId": "string (UUID)",
    "createdDateTime": "string (ISO 8601)"
  },
  "error": null
}
```

**錯誤:**

| 代碼 | ErrorType               | 條件                                                                         |
| ---- | ----------------------- | ---------------------------------------------------------------------------- |
| 400  | `VALIDATION_ERROR`      | `reviewers` 為空陣列；`priority` 不唯一或者不連貫；某位委員缺少 `email` 值。 |
| 401  | `AUTHENTICATION_ERROR`  | 無效或過期的憑證                                                             |
| 403  | `AUTHORIZATION_ERROR`   | 該申請案不隸屬分配於 DC 的學門下。                                           |
| 404  | `NOT_FOUND`             | 找不到申請案                                                                 |
| 500  | `INTERNAL_SERVER_ERROR` | 意外的後端失敗。                                                             |

---

### 6.4. `PUT /applications/{applicationId}/final-review` — 儲存或送出決審

儲存 DC 決審的草稿或進行最終提交。重複呼叫且帶有相同的資料會產生相同的結果 (冪等性)。

**路徑參數:**

| 參數            | 類型        | 說明        |
| --------------- | ----------- | ----------- |
| `applicationId` | UUID string | 申請案的 ID |

**請求主體:**

```json
{
  "score": "integer",
  "remarks": "string",
  "isFinalized": "boolean"
}
```

**驗證條件:**

- 即使為草稿 (`isFinalized = false`) **`score` 和 `remarks` 也必填且不能為 null**。
- 當 `isFinalized = false` (草稿) 時：不需要檢查初審完成的情況。
- 當 `isFinalized = true`：
  - 至少 `minNumInitialReviewers` 個確認的初審委員狀態需為 `reviewStatus = '已完成'`。後端必須檢查，如果不符合時回傳 `409 CONFLICT`。
- 一旦申請案的狀態進入到 `COMPLETED`（先前已決審），此後所有後續的 `PUT` 請求必須被拒絕，回傳 `409 CONFLICT`。

**回應 (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "updatedDateTime": "string (ISO 8601)"
  },
  "error": null
}
```

**錯誤:**

| 代碼 | ErrorType               | 條件                                                                                                       |
| ---- | ----------------------- | ---------------------------------------------------------------------------------------------------------- |
| 400  | `VALIDATION_ERROR`      | `score` 或 `remarks` 帶有 null 值或是未出現。                                                              |
| 401  | `AUTHENTICATION_ERROR`  | 無效或過期的憑證                                                                                           |
| 403  | `AUTHORIZATION_ERROR`   | 此申請案不隸屬於被分配至此 DC 的學門                                                                       |
| 404  | `NOT_FOUND`             | 找不到申請案。                                                                                             |
| 409  | `CONFLICT`              | 為 `isFinalized = true` 的同時並沒有滿足最低要求數量的委員提交初審，或者是申請案本身已經處在 `COMPLETED`。 |
| 500  | `INTERNAL_SERVER_ERROR` | 意外的後端錯誤。                                                                                           |

---

## 7. 參考資料: 期望值 (Expected Value) 計算

> 注意：此資訊僅為背景知識而分享。前端可自行處理計算，同時也需要來自後端的 `expectedValue`。

每個學門的 `expectedValue` 是一個「彈性」的推薦配額 — 供 DC 在決定要給多少申請人高分時作為參考。

**預設公式如下:**

```
overallSelectionRatio = fixedQuota / totalApplicationsInTrack

expectedValue (對每個學門) = disciplineApplicantCount × overallSelectionRatio
```

**保權性設計:** 若某個學門算出來的 `expectedValue < 1.0`，基於代表性公平原則，DC 仍然可以為該學門推薦 1 位評價較高的候選人。

`overallSelectionRatio` 和各學門的 `expectedValue` 皆由後端計算並回傳，前端不需要重新計算。
