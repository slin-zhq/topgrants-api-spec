# TOP Grants — 學門召集人平台 API 規格書

**作者:** 張智奇 (Saw S. Lin)，由 Gemini 和 Claude 輔助。

**適用範圍:** 僅涵蓋學門召集人 (Discipline Convener – 簡稱“DC”) 客戶端。後台管理端 (Admin) 和申請人端 端點不在範圍內。

**版本控制:** API 回應主體內將會包含版本名稱。破壞性變更會增加主版本號。目前所有開發都針對 `v1`。

**前提:**

- 目前將不會有一個整合 `phdCandidate` 和 `youngScholar` 後端的 API Gateway，因此將會有各自對應的端點。
- 因此，將採用以下模式：`/api/phd-candidate/applications/...` 以及 `/api/young-scholar/applications/...`。

---

## 1. 設計原則

- 每個對 `/applications/{id}/reviewer-recommendations` 的 `POST` 請求都會 **附加** 一個新的推薦紀錄。它不會替換先前的紀錄。
- 後端計算並擁有每個申請案的 `status` (狀態)。DC 使用者無法直接設定它。
- `POST` 建立新的子資源。`PUT` 替換現有資源 (冪等)。`DELETE` 刪除資源。

> 注意：
>
> - (2026/03/13 更新) 在審查階段開始後，仍可能有新的申請案進入。因此，DC 開始處理申請案並不代表不再會有新的申請案。前端將在使用者登入或手動重新整理頁面時顯示新進的申請案。

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

## 4. 身分驗證 (Authentication)

[2026/03/13 討論更新] 這次為 DC 使用者改版的前端將不再處理使用者身分驗證。修改後的身分驗證流程如下：

1. 當使用者造訪部署好的網頁應用程式時，會看到一個只有「登入」按鈕的頁面。
2. 使用者點擊「登入」，這將觸發一個瀏覽器彈出視窗 — 類似透過第三方服務（如 Google、Apple、微信 等）的 SSO 流程。
3. 身分驗證過程將由「廠商」處理，使用者會在廠商建立的 UI 中進行操作，直到成功為止。
4. 同時，主應用程式會等待接收經過驗證的 session 憑證 (session token)。若觸發超時，則會提示使用者重試。
5. 主應用程式將儲存該經過驗證的憑證，供後續 API 呼叫使用。

**為了符合資安需求**

基於安全性考量：

1. 前端會在 session 憑證過期前一段時間（例如 2 分鐘），透過對話框提示使用者重新驗證身分。當然，前端也會顯示 session 有效時間的倒數計時器。
2. 若使用者選擇重新登入，她會被導向身分驗證流程。成功驗證後，前端會儲存新的 session 憑證，並允許她像平常一樣繼續操作。
3. 若偵測到無任何活動，前端會檢查是否有未儲存的變更，並在憑證仍有效時發送 API 呼叫以儲存使用者的變更。接著，前端會將使用者登出。

---

### `POST /api/sessions` — 登入

> [TODO] 詳細資訊待後續確認。

---

### `DELETE /api/sessions` — 登出

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

| 代碼 | ErrorType              | 條件             |
| ---- | ---------------------- | ---------------- |
| 401  | `AUTHENTICATION_ERROR` | 憑證已遺失或無效 |

---

### 經過身分驗證的請求標頭

所有端點皆需要：

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

---

## 5. 申請案狀態機

後端計算並擁有每個申請案的 `status` (狀態)。DC 無法直接設定；它會根據審查委員、管理員的事件以及 DC 自身的操作而改變。

### 5.1 狀態值

| 值       | 顯示文字 (ZhTW) | 意義                                                                                                                |
| -------- | --------------- | ------------------------------------------------------------------------------------------------------------------- |
| `待推薦` | 待推薦          | DC 尚未提交任何推薦委員，**或** 尚未有任何推薦紀錄。                                                                |
| `待初審` | 待初審          | 至少存在一筆推薦紀錄。可能尚無任何已確認的委員。或如果有的話，並非所有已確認的委員都已完成審查。DC 無已儲存的草稿。 |
| `待複審` | 待複審          | 至少有 `minIniitalReviewerCount` 個已確認的委員提交了他們的審查結果。DC 尚未儲存任何草稿。                          |
| `待送出` | 待送出          | DC 已儲存決審草稿 (`isFinalized = false`)。初審可能完成也可能未完全完成；直到完成前將被阻擋送出 (finalize)。        |
| `已完成` | 已完成          | DC 已提交並完成決審 (`isFinalized = true`)。                                                                        |

### 5.2 狀態轉換條件

```
待推薦
  ─► 待初審
        時機：DC 提交至少一名推薦委員。
  ─► 待送出
        時機：DC 在提交任何推薦前先儲存了一份草稿。
              [邊緣情況 — 不常見但允許]

待初審
  ─► 待複審
        時機：至少 `minIniitalReviewerCount` 名已確認的委員的 reviewStatus = "已完成"
              且 DC 無已儲存草稿
  ─► 待送出
        時機：DC 在最低要求的初審數量完成之前先儲存了一份草稿。

待送出
  ─► 待初審  (退回/回歸)
        時機：已完成的初審被撤銷/作廢，且已完成的數量降至 `minIniitalReviewerCount` 以下。
  ─► 待送出
        時機：DC 儲存草稿 (PUT /final-review, isFinalized = false)
  ─► 已完成
        時機：DC 在沒有先前草稿的情況下完成 (PUT /final-review, isFinalized = true)

待送出
  ─► 已完成
        時機：DC 提交 (PUT /final-review, isFinalized = true)
              並且至少 `minIniitalReviewerCount` 名已確認的委員的 reviewStatus = "已完成"
              如果不符合此條件，後端必須以 409 CONFLICT 拒絕。
```

### 5.3 業務規則

1. DC **可以儲存草稿** (`isFinalized = false`) 在任何時候，即使在所有初審完之前。
2. DC **無法決定送出 (finalize)** (`isFinalized = true`) 除非至少 `minIniitalReviewerCount` 名初審委員已完成。後端必須使用 `409 CONFLICT` 強制執行。
3. 一旦申請案達到 `已完成`，就不接受進一步的編輯。隨後的 `PUT /final-review` 呼叫必須被拒絕，回傳 `409 CONFLICT`。
4. 如果一位已經完成審查 (`reviewStatus = "已完成"`) 的委員後來在管理層面被撤銷 (極端情況)，後端必須相應地重新評估申請案的狀態。

---

## 6. 端點 (Endpoints)

呼叫以下所有端點都必須包含 `Authorization: Bearer <sessionToken>` 標頭。所有端點都應以 `/api` 作為前綴。

---

### 6.1 `GET /phd-candidate/users/me/review-batches` 與 `GET /young-scholar/users/me/review-batches` — 「首頁」資料

`review batch` = `審查梯次`

**請求主體:** 無。

**回應 (`200 OK`):**

僅回傳進行中的（active）審查梯次。

針對 **`phd-candidate`**:

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "reviewBatchId": "string (UUID)",
    "year": "integer",
    "deadlineDateTime": "string (ISO 8601, 必須包含時間和 UTC 偏移量)",
    "applicationSummary": [
      {
        "discipline": "string",
        "applicationCountByStatus": {
          "待推薦": "integer",
          "待初審": "integer",
          "待複審": "integer",
          "待送出": "integer",
          "已完成": "integer"
        }
      }
    ]
  }
}
```

針對 **`young-scholar`**:

```json
{
  // 結構與 `phd-candidate` 相同
}
```

**錯誤:**

| 代碼 | ErrorType               | 條件                                                      |
| ---- | ----------------------- | --------------------------------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | 憑證無效或已過期                                          |
| 403  | `AUTHORIZATION_ERROR`   | 使用者並非 DC，或在進行中的審查梯次中並未被分配到任何學門 |
| 404  | `NOT_FOUND`             | 找不到進行中的審查梯次                                    |
| 500  | `INTERNAL_SERVER_ERROR` | 預期外的後端失敗                                          |

---

### 6.2. `GET /phd-candidate/users/me/applications` 與 `GET /young-scholar/users/me/applications` - 各組別「申請案列表」頁面

當使用者在特定組別點擊「進入」時，會呼叫此 API。

**請求主體:**

```json
{
  "reviewBatchId": "string"
}
```

**回應 (`200 OK`):**

`phd-candidate` 與 `young-scholar` 共用相同的回應格式：

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "reviewBatchId": "string (UUID)",
    "fixedQuota": "integer (核定名額/正取名額上限)",
    "totalApplicationsCount": "integer", // 總申請案件數，不限於目前 DC 所負責的部分。
    "overallExpectedRatio": "number (float/double)",
    "evaluationRubrics": [
      // 包含於此，供 DC 在尚未有任何初審結果、需要自行評分時做為評量基準參考。
      {
        "title": "string",
        "fullScore": "integer"
      }
    ],
    "disciplines": [
      {
        "name": "string (於此組別與週期內必須唯一)",
        "totalApplicationsCount": "integer", // 該學門的總申請數，不限於目前 DC 所負責的部分。
        "projectedQuotaRaw": "number (float/double)",
        "projectedQuotaRounded": "integer",
        "applications": [
          {
            "id": "string (UUID)",
            "status": "待推薦 | 待初審 | 待複審 | 待送出 | 已完成",
            "applicantNameZhTW": "string",
            "currentInstitutionAndDepartmentZhTw": "string",
            "doctoralThesisTitleZhTw": "string",
            "summaryStats": {
              "mostRecentlyRecommendedInitialReviewerCount": "integer",
              "confirmedInitialReviewerCount": "integer",
              "completedInitialReviewCount": "integer"
            }
          }
        ]
      }
    ]
  }
}
```

**錯誤:**

| 代碼 | ErrorType               | 條件                                                      |
| ---- | ----------------------- | --------------------------------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | 憑證無效或已過期                                          |
| 403  | `AUTHORIZATION_ERROR`   | 使用者並非 DC，或在進行中的審查梯次中並未被分配到任何學門 |
| 404  | `NOT_FOUND`             | 找不到進行中的審查梯次                                    |
| 500  | `INTERNAL_SERVER_ERROR` | 預期外的後端失敗                                          |

---

### 6.3. `GET /phd-candidate/applications/{applicationId}` 與 `GET /young-scholar/applications/{applicationId}` - 「申請案件」頁面

**路徑參數:**

| 參數            | 類型        | 說明        |
| --------------- | ----------- | ----------- |
| `applicationId` | UUID string | 申請案的 ID |

**請求主體:** 無。

**回應 (`200 OK`):**

針對 **`/phd-candidate`**:

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "applicationId": "string (UUID)",
    "submittedDateTime": "string (ISO 8601)",
    "minIniitalReviewerCount": "integer",
    "applicantNameZhTW": "string",
    "applicantNameEn": "string",
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
      "researchProposal": { "fileName": "string", "fileUrl": "string" }, // 論文計畫書
      "publications": {
        "tableOfContents": { "fileName": "string", "fileUrl": "string" }, // 著作目錄
        "details": [
          // 著作與學術成果
          {
            "displayName": "string",
            "fileName": "string",
            "fileUrl": "string"
          }
        ]
      },
      "phdCandidacyCertificate": {
        // 博士候選人資格證明
        "fileName": "string",
        "fileUrl": "string"
      },
      "supplementaries": [
        // 補充附件
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
    "finalReview": {
      // 可為 null
      "score": "integer | null",
      "remarks": "string | null",
      "isFinalized": "boolean",
      "updatedDateTime": "string (ISO 8601) | null (如果尚未存檔則為 null)"
    }
  }
}
```

針對 **`/young-scholar`**:

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "applicationId": "string (UUID)",
    "submittedDateTime": "string (ISO 8601)",
    "minIniitalReviewerCount": "integer",
    "applicantNameZhTW": "string",
    "applicantNameEn": "string",
    "email": "string",
    "currentInstitutionAndDepartmentZhTw": "string",
    "currentInstitutionAndDepartmentEn": "string",
    "monographTitleZhTw": "string", // 專書
    "monographTitle": "string",
    "monographAbstractZhTw": "string",
    "monographAbstractEn": "string",
    "expectedImpact": "string",
    "applicantEduBackground": [
      // 結構與 `phd-candidate` 相同
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
      "researchProposal": { "fileName": "string", "fileUrl": "string" }, // 計畫書
      "publications": {
        "tableOfContents": { "fileName": "string", "fileUrl": "string" }, // 著作目錄
        "details": [
          // 著作與學術成果
          {
            "displayName": "string",
            "fileName": "string",
            "fileUrl": "string"
          }
        ]
      },
      "doctoralDegreeCertificate": {
        // 博士學位證書
        "fileName": "string",
        "fileUrl": "string"
      },
      "employmentProof": {
        // 在職證明/合約影本
        "fileName": "string",
        "fileUrl": "string"
      },
      "supervisorRecommendationLetter": {
        // 任職機構主管推薦函
        "fileName": "string",
        "fileUrl": "string"
      },
      "supplementaries": [
        // 補充附件
        {
          "displayName": "string",
          "fileName": "string",
          "fileUrl": "string"
        }
      ]
    },
    "finalReview": {
      // 可為 null
      "score": "integer | null",
      "remarks": "string | null",
      "isFinalized": "boolean",
      "updatedDateTime": "string (ISO 8601) | null (如果尚未存檔則為 null)"
    }
  }
}
```

**錯誤:**

| 代碼 | ErrorType               | 條件                             |
| ---- | ----------------------- | -------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | 憑證無效或已過期                 |
| 403  | `AUTHORIZATION_ERROR`   | 該申請案不隸屬所分配 DC 的學門下 |
| 404  | `NOT_FOUND`             | 找不到該申請案                   |
| 500  | `INTERNAL_SERVER_ERROR` | 預期外的後端失敗                 |

---

### 6.4 `GET /phd-candidate/applications/{applicationId}/recommendation-records` 與 `GET /young-scholar/applications/{applicationId}/recommendation-records` - 單一申請案推薦紀錄

**路徑參數:**

| 參數            | 類型        | 說明        |
| --------------- | ----------- | ----------- |
| `applicationId` | UUID string | 申請案的 ID |

**請求主體:** 無。

**回應主體:**

`phd-candidate` 與 `young-scholar` 共用相同的回應格式：

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "applicationId": "string (UUID)",
    "recommendationRecords": [
      {
        "createdDateTime": "string (ISO 8601)", // 與申請案關聯的紀錄可由它們的 `createdDateTime` 唯一識別
        "reviewers": [
          {
            "priority": "integer (從 1 開始；較小 = 較高優先順序)",
            "name": "string",
            "email": ["string"],
            "remarks": "string | null (DC 推薦此委員的理由)"
          }
        ]
      }
    ]
  }
}
```

**錯誤:**

| 代碼 | ErrorType               | 條件                             |
| ---- | ----------------------- | -------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | 憑證無效或已過期                 |
| 403  | `AUTHORIZATION_ERROR`   | 該申請案不隸屬所分配 DC 的學門下 |
| 404  | `NOT_FOUND`             | 找不到該申請案                   |
| 500  | `INTERNAL_SERVER_ERROR` | 預期外的後端失敗                 |

---

### 6.5 `GET /applications/{applicationId}/initial-reviews` — 初審委員詳情

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
          "totalScore": "integer | null",
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
> - `scores` 陣列長度和該組在 `evaluationRubrics` 中評分標準項目數量需一致，並依據評分標準順序排序。
> - `totalScore` 必須等於 `scores` 的總和。後端應計算並傳回此值；不要依賴前端進行加總。

**錯誤類型:**

| 代碼 | ErrorType               | 條件                             |
| ---- | ----------------------- | -------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | 憑證無效或已過期                 |
| 403  | `AUTHORIZATION_ERROR`   | 該申請案不隸屬所分配 DC 的學門下 |
| 404  | `NOT_FOUND`             | 找不到該申請案                   |
| 500  | `INTERNAL_SERVER_ERROR` | 預期外的後端失敗                 |

---

### 6.6 `POST /applications/{applicationId}/reviewer-recommendations` — 提交推薦委員

為該申請案創建新的推薦紀錄。每次呼叫都會在推薦歷史中新增一筆。後端在伺服器端儲存 `createdDateTime`；請勿接受來自客戶端的這個參數。

**路徑參數:**

| 參數            | 類型        | 說明        |
| --------------- | ----------- | ----------- |
| `applicationId` | UUID string | 申請案的 ID |

**請求主體:**

```json
{
  "reviewers": [
    {
      "priority": "integer (從 1 開始；較小 = 較高優先順序)",
      "name": "string",
      "email": ["string"],
      "remarks": "string | null"
    }
  ]
}
```

**驗證條件:**

- `reviewers` 必須是一個非空陣列。
- `priority` 數值在請求內必須唯一，且構成從 1 開始的連續數列。
- 每個審查委員在 `email` 中至少必須有一個條目。

**回應 (`201 Created`):**

```json
{
  "status": "success",
  "code": 201,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "applicationId": "string (UUID)",
    "createdDateTime": "string (ISO 8601)",
    "reviewers": [
      {
        "priority": "integer (從 1 開始；較小 = 較高優先順序)",
        "name": "string",
        "email": ["string"],
        "remarks": "string | null (DC 推薦此委員的理由)"
      }
    ]
  },
  "error": null
}
```

> **實作備註 (狀態管理最佳實務 - State Management Best Practice):**
>
> 為確保最佳的前端效能並遵循現代 API 標準，後端將傳回**完整建構的推薦紀錄**。在成功的 `POST` 請求之後，前端**絕不可**再次發送 `GET` 請求去刷新列表。相反地，前端應直接將這個回傳的物件注入其本地狀態管理中，以立即更新 UI。

**錯誤:**

| 代碼 | ErrorType               | 條件                                                                             |
| ---- | ----------------------- | -------------------------------------------------------------------------------- |
| 400  | `VALIDATION_ERROR`      | `reviewers` 為空陣列；`priority` 數值不唯一或者不連續；某位委員缺少 `email` 值。 |
| 401  | `AUTHENTICATION_ERROR`  | 憑證無效或已過期                                                                 |
| 403  | `AUTHORIZATION_ERROR`   | 該申請案不隸屬分配於 DC 的學門下                                                 |
| 404  | `NOT_FOUND`             | 找不到申請案                                                                     |
| 500  | `INTERNAL_SERVER_ERROR` | 預期外的後端失敗                                                                 |

---

### 6.7 `PUT /applications/{applicationId}/final-review` — 儲存或送出決審

儲存草稿或為一個申請案將 DC 的決審標定為完成。冪等性實作 — 攜帶相同資料進行多次呼叫應產生相同的結果。

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

- 當 `isFinalized = true` 時，**`score` 和 `remarks` 必須是非 null 的**。然而，對於草稿 (`isFinalized = false`)，兩者可以擇一為 null，但不可以兩者皆為 null。
- 當 `isFinalized = false` (草稿) 時：對初審是否完成無限制。
- 當 `isFinalized = true`：
  - 至少有 `minIniitalReviewerCount` 個已確認委員的 `reviewStatus = '已完成'`。後端必須強制執行此檢查；如果未滿足，則傳回 `409 CONFLICT`。
- 一旦申請案的狀態達到 `已完成`（先前已決審），任何後續的 `PUT` 請求都必須被拒絕，並回傳 `409 CONFLICT`。

**回應 (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "applicationId": "string (UUID)",
    "score": "integer | null",
    "remarks": "string | null",
    "isFinalized": "boolean",
    "updatedDateTime": "string (ISO 8601)"
  },
  "error": null
}
```

> **實作備註 (狀態管理最佳實務 - State Management Best Practice):**
>
> 與推薦委員的端點類似，後端將傳回**完整更新後的決審物件**。前端應直接使用此回傳資料更新其本地狀態，而不是發起多餘的 `GET` 請求。

**錯誤:**

| 代碼 | ErrorType               | 條件                                                                                                   |
| ---- | ----------------------- | ------------------------------------------------------------------------------------------------------ |
| 400  | `VALIDATION_ERROR`      | `score` 或是 `remarks` 帶有 null 值或是遺失                                                            |
| 401  | `AUTHENTICATION_ERROR`  | 憑證無效或已過期                                                                                       |
| 403  | `AUTHORIZATION_ERROR`   | 該申請案不隸屬所分配 DC 的學門下                                                                       |
| 404  | `NOT_FOUND`             | 找不到該申請案                                                                                         |
| 409  | `CONFLICT`              | 為 `isFinalized = true` 但沒有滿足最低要求數量的委員提交了初審；或者是申請案本身已經處在 `已完成` 狀態 |
| 500  | `INTERNAL_SERVER_ERROR` | 預期外的後端失敗                                                                                       |

---

## 7. 參考資料: 預計入選名額 計算

每個學門的 `預計入選名額` 是一個「彈性」的推薦配額 — 供 DC 在決定要給多少申請人高分時作為參考。

**預設公式如下:**

```markdown
整體入選期望值 (`overallExpectedRatio`) = fixedQuota / totalApplicationsCount (總申請人數)

預計入選名額 (per discipline, `projectedQuotaRaw`) = 該學門申請數 (discipline's `totalApplicationsCount`) × `overallExpectedRatio`

最終預計入選人數 (`projectedQuotaRounded`) = round(`projectedQuotaRaw`)
```

**保權性設計:** 若某個學門算出來的 `projectedQuotaRaw < 1.0`，基於代表性公平原則，DC 仍然可以為該學門推薦 1 位評價較高的候選人。

`overallExpectedRatio` 和各學門的預計入選名額皆由後端計算並回傳，前端不應重新計算。
