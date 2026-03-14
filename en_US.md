# TOP Grants — Discipline Convener Platform API Specification

**Authors:** Saw S. Lin (Zhang Zhiqi), with assistance of Gemini, and Claude.

**Scope:** Covers the Discipline Convener (DC / 學門召集人) client only. Admin-side and applicant-side endpoints are out of scope.

**Versioning:** Version names are to be included inside the AI response body. Breaking changes increment the major version. All current development targets `v1`.

**Premises:**

- There will not currently be an API Gateway that integrates `phdCandidate` and `youngScholar` backends, so there will be separate corresponding endpoints.
- Hence, the following pattern is to be adopted: `/api/phd-candidate/applications/...` and `/api/young-scholar/applications/...`.

---

## 1. Design Principles

<!-- - Every endpoint except `POST /sessions` and `POST /sessions/refresh` requires an `Authorization: Bearer <sessionToken>` header. -->

- Each `POST` to `/applications/{id}/reviewer-recommendations` **appends** a new recommendation record. It does not replace prior records.
- The backend computes and owns each application's `status`. The DC user cannot set it directly.
- `POST` creates a new sub-resource. `PUT` replaces an existing resource (idempotent). `DELETE` removes a resource.

> Note:
>
> - (2026/03/13 Update) New applications may still arrive after the review stage begins. Therefore, the commencement of application processing by the DC user does not imply a freeze on new submissions. The frontend will display any newly arrived applications upon user login or manual refresh.

---

## 2. Common Interfaces

### 2.1 Base Response Envelope

All API responses share this structure:

```json
{
  "status": "success | error",
  "code": "integer (HTTP status code)",
  "timestamp": "string (ISO 8601)",
  "apiVersion": "string (e.g., 'v1')",
  "data": "object | array | null",
  "error": "object | null"
}
```

- On success: `data` contains the payload; `error` is `null`.
- On error: `data` is `null`; `error` contains the error object.

### 2.2 Error Object

```json
{
  "type": "string (one of the ErrorType values below)",
  "message": "string (human-readable summary)",
  "details": [
    {
      "field": "string | null (dot-notation path to the offending field)",
      "message": "string"
    }
  ]
}
```

`details` may be an empty array. It is used for validation errors to enumerate all failed fields.

### 2.3 ErrorType Reference

| ErrorType               | When Used                                                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `VALIDATION_ERROR`      | Request body or query parameter failed validation                                                                   |
| `AUTHENTICATION_ERROR`  | Token missing, expired, or invalid                                                                                  |
| `CAPTCHA_ERROR`         | CAPTCHA token missing, invalid, score too low, or expired                                                           |
| `AUTHORIZATION_ERROR`   | Authenticated user lacks permission for this resource                                                               |
| `NOT_FOUND`             | Requested resource does not exist                                                                                   |
| `CONFLICT`              | Request conflicts with the current resource state (e.g., finalizing review before all initial reviews are complete) |
| `QUERY_EXECUTION_ERROR` | Database query failed                                                                                               |
| `TIMEOUT_ERROR`         | Backend operation timed out                                                                                         |
| `RATE_LIMIT_ERROR`      | Too many requests from the client                                                                                   |
| `INTERNAL_SERVER_ERROR` | Unexpected server-side failure                                                                                      |

<!-- | `AI_SERVICE_ERROR`      | Smart recommendation service error                                                                                  | -->
<!-- | `EXPORT_ERROR`          | File/PDF export generation failed                                                                                   | -->

---

## 3. HTTP Status Codes

| Code | Meaning               | When Used                                               |
| ---- | --------------------- | ------------------------------------------------------- |
| 200  | OK                    | Successful GET, PUT, DELETE                             |
| 201  | Created               | Successful POST that creates a resource                 |
| 202  | Accepted              | Request accepted for asynchronous processing            |
| 400  | Bad Request           | Validation error, malformed request, or CAPTCHA failure |
| 401  | Unauthorized          | Missing or invalid session token                        |
| 403  | Forbidden             | Authenticated but not permitted                         |
| 404  | Not Found             | Resource does not exist                                 |
| 408  | Request Timeout       | Query ~~or AI generation~~ timed out                    |
| 409  | Conflict              | State conflict (see §5 for state machine)               |
| 429  | Too Many Requests     | Rate limit exceeded                                     |
| 500  | Internal Server Error | Unexpected backend error                                |

---

## 4. Authentication

[Update after 2026/03/13 discussion] This revamped Frontend for DC users will no longer handle user authentication. The revised authentication flow is like so:

1. When User visits the deployed web app, they will be greeted with a page having only the "Log In" button.
2. User clicks "Log In", which triggers a browser window pop-up – similar to SSO flow through 3rd party services, like Google, Apple, WeChat, etc.
3. The authentication process is to be handled by the 廠商, in which User interacts with the UI built by 廠商 until success.
4. Meanwhile, the main app will wait until receving authenticated session token. Otherwise, User will be prompted to retry once timeout is triggered.
5. The main app will store the authenticated token for subsequent API calls.

**Aligning with secuirity requirements**

In light of security requirements,

1. Frontend will prompt User via an alert dialog to get authenticated, some time before the session token expires (say, 2 mins). Of course, Frontend will display a live timer on the session valid duration.
2. Should User choose to log in again, she will be redirected to the authentication flow. Upon successful authentication, Frontend stores the new session token, and she can continue as usual.
3. Should no activity is detected, Frontend will detect unsaved changes, and make an API call while the token is valid to save user changes. Then, Frontend will log User out.

---

### `POST /api/sessions` — Login

> [TODO] Details to be confirmed later.

<!-- No `Authorization` header required.

**Request Body:**


**Response (`200 OK`):**

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

**Errors:**

| Code | ErrorType              | Condition                                                             |
| ---- | ---------------------- | --------------------------------------------------------------------- |
| 400  | `VALIDATION_ERROR`     | `username` or `password` is missing                                   |
| 400  | `CAPTCHA_ERROR`        | `captchaToken` is missing, invalid, expired, or score below threshold |
| 401  | `AUTHENTICATION_ERROR` | Credentials are incorrect                                             |
| 429  | `RATE_LIMIT_ERROR`     | Too many failed login attempts from this IP/user                      | -->

---

### `DELETE /api/sessions` — Logout

**Headers:** `Authorization: Bearer <sessionToken>`

**Request Body:** None.

**Response (`200 OK`):**

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

**Errors:**

| Code | ErrorType              | Condition                               |
| ---- | ---------------------- | --------------------------------------- |
| 401  | `AUTHENTICATION_ERROR` | Token is missing or already invalidated |

---

### Authenticated Request Headers

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

---

## 5. Application Status State Machine

The backend computes and owns each application's `status`. The DC cannot set it directly; it changes in response to events from reviewers, admins, and the DC's own actions.

### 5.1 Status Values

| Value    | Display (ZhTW) | Meaning                                                                                                                                                                                       |
| -------- | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `待推薦` | 待推薦         | DC has not yet submitted any reviewer recommendations, **or** there is not yet any recommendation record.                                                                                     |
| `待初審` | 待初審         | At least one recommendation record is present. There can be no confirmed reviewers yet. Or, if there is any, not all confirmed reviewers have completed their reviews. DC has no saved draft. |
| `待複審` | 待複審         | At least `minIniitalReviewerCount` confirmed reviewers have submitted their reviews. DC has not yet saved any draft.                                                                          |
| `待送出` | 待送出         | DC has saved a draft final review (`isFinalized = false`). Initial reviews may or may not all be complete; finalization is blocked until they are.                                            |
| `已完成` | 已完成         | DC has submitted and finalized the review (`isFinalized = true`).                                                                                                                             |

### 5.2 Transition Conditions

```
待推薦
  ─► 待初審
        When: DC submits at least one reviewer recommendation.
  ─► 待送出
        When: DC saves a draft before submitting any recommendations
              [edge case — uncommon but allowed]

待初審
  ─► 待複審
        When: At least `minIniitalReviewerCount` confirmed reviewers' reviewStatus = "已完成"
              AND DC has no saved draft
  ─► 待送出
        When: DC saves a draft before the minimum required initial reviews are complete

待送出
  ─► 待初審  (regression)
        When: A completed initial review is revoked/invalidated and completed count drops below `minIniitalReviewerCount`
  ─► 待送出
        When: DC saves a draft (PUT /final-review, isFinalized = false)
  ─► 已完成
        When: DC finalizes without a prior draft (PUT /final-review, isFinalized = true)

待送出
  ─► 已完成
        When: DC submits (PUT /final-review, isFinalized = true)
              AND at least `minIniitalReviewerCount` confirmed reviewers' reviewStatus = "已完成"
              Backend must reject with 409 CONFLICT if this condition is not met.
```

### 5.3 Business Rules

1. DC **may save a draft** (`isFinalized = false`) at any time, even before all initial reviews are complete.
2. DC **cannot finalize** (`isFinalized = true`) unless at least `minIniitalReviewerCount` reviewers have completed their initial reviews. Backend must enforce this with `409 CONFLICT`.
3. Once an application reaches `已完成`, no further edits are accepted. Subsequent `PUT /final-review` calls must be rejected with `409 CONFLICT`.
4. If a reviewer who has already completed their review (`reviewStatus = "已完成"`) is later revoked at the admin level (edge case), the backend must re-evaluate the application status accordingly.

---

## 6. Endpoints

All calls to the following endpoints will have `Authorization: Bearer <sessionToken>` header. All endpoints are to be prefixed with `/api`.

---

### 6.1 `GET /phd-candidate/users/me/review-batches` and `GET /young-scholar/users/me/review-batches` — "Home" Page Data

`review batch` = `審查梯次`

**Request Body:** None.

**Response (`200 OK`):**

Must return only active review batches.

For **`phd-candidate`**:

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "reviewBatchId": "string (UUID)",
    "year": "integer",
    "deadlineDateTime": "string (ISO 8601, must include time and UTC offset)",
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

For **`young-scholar`**:

```json
{
  // Same as `phd-candidate`
}
```

**Errors:**

| Code | ErrorType               | Condition                                                            |
| ---- | ----------------------- | -------------------------------------------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                             |
| 403  | `AUTHORIZATION_ERROR`   | User is not a DC, or has no assigned disciplines in the active batch |
| 404  | `NOT_FOUND`             | No active review batch exists                                        |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                           |

---

### 6.2. `GET /phd-candidate/users/me/applications` and `GET /young-scholar/users/me/applications` - "Application List" Page per Track

When User clicks "進入" on a particular track, this API will be called.

**Request Body:**

```json
{
  "reviewBatchId": "string"
}
```

**Response (`200 OK`):**

Both `phd-candidate` and `young-scholar` share the same response format:

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "reviewBatchId": "string (UUID)",
    "fixedQuota": "integer (核定名額/正取名額上限)",
    "totalApplicationsCount": "integer", // Total no. of applications, not limited to those to be handled by current DC user.
    "overallExpectedRatio": "number (float/double)",
    "evaluationRubrics": [
      // This is included here, for the DC reviewer to consult, when she is scoring on her own, before any initial review is completed.
      {
        "title": "string",
        "fullScore": "integer"
      }
    ],
    "disciplines": [
      {
        "name": "string (must be unique within this track and cycle)",
        "totalApplicationsCount": "integer", // Total no. of applications, not limited to those to be handled by current DC user.
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

**Errors:**

| Code | ErrorType               | Condition                                                            |
| ---- | ----------------------- | -------------------------------------------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                             |
| 403  | `AUTHORIZATION_ERROR`   | User is not a DC, or has no assigned disciplines in the active batch |
| 404  | `NOT_FOUND`             | No active review batch exists                                        |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                           |

---

### 6.3. `GET /phd-candidate/applications/{applicationId}` and `GET /young-scholar/applications/{applicationId}` - "Application Details" page

**Path Parameter:**

| Parameter       | Type        | Description           |
| --------------- | ----------- | --------------------- |
| `applicationId` | UUID string | ID of the application |

**Request Body:** None.

**Response (`200 OK`):**

For **`/phd-candidate`**:

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
        "endYearAndMonth": "string (YYYY/MM) | null (null if currently enrolled)"
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
      // Can be null
      "score": "integer | null",
      "remarks": "string | null",
      "isFinalized": "boolean",
      "updatedDateTime": "string (ISO 8601) | null (null if no save has occurred)"
    }
  }
}
```

For **`/young-scholar`**:

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
      // Same as `phd-candidate`
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
      // Can be null
      "score": "integer | null",
      "remarks": "string | null",
      "isFinalized": "boolean",
      "updatedDateTime": "string (ISO 8601) | null (null if no save has occurred)"
    }
  }
}
```

**Errors:**

| Code | ErrorType               | Condition                                                    |
| ---- | ----------------------- | ------------------------------------------------------------ |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                     |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines |
| 404  | `NOT_FOUND`             | Application not found                                        |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                   |

---

### 6.4. `GET /phd-candidate/applications/{applicationId}/recommendation-records` and `GET /young-scholar/applications/{applicationId}/recommendation-records` - Recommendation Records per Application

**Path Parameter:**

| Parameter       | Type        | Description           |
| --------------- | ----------- | --------------------- |
| `applicationId` | UUID string | ID of the application |

**Request Body:** None.

**Response Body:**

Both `phd-candidate` and `young-scholar` share the same response format:

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
        "createdDateTime": "string (ISO 8601)", // Records associated with an application are uniquely identified by their `createdDateTime`
        "reviewers": [
          {
            "priority": "integer (starts from 1; lower = higher priority)",
            "name": "string",
            "email": ["string"],
            "remarks": "string | null (DC's reason for recommending this reviewer)"
          }
        ]
      }
    ]
  }
}
```

**Errors:**

| Code | ErrorType               | Condition                                                    |
| ---- | ----------------------- | ------------------------------------------------------------ |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                     |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines |
| 404  | `NOT_FOUND`             | Application not found                                        |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                   |

---

### 6.5 `GET /applications/{applicationId}/initial-reviews` — Initial Reviewer Details

Returns confirmed initial reviewers and their review details for a specific application. Call this when the DC opens an application's detail view.

**Path Parameter:**

| Parameter       | Type        | Description           |
| --------------- | ----------- | --------------------- |
| `applicationId` | UUID string | ID of the application |

**Request Body:** None.

**Response (`200 OK`):**

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

> **Implementation Notes:**
>
> - `initialReviews` contains only reviewers confirmed by 承辦人員.
> - `reviewDetails` is non-null only when `reviewStatus = '已完成'`. Return `null` for `reviewDetails` when status is `未完成`.
> - `scores` array length must match the number of rubric items in the track's `evaluationRubrics`, ordered by rubric index.
> - `totalScore` must equal the sum of `scores`. Backend should compute and return this; do not rely on frontend summation.

**Error Type:**

| Code | ErrorType               | Condition                                                    |
| ---- | ----------------------- | ------------------------------------------------------------ |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                     |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines |
| 404  | `NOT_FOUND`             | Application not found                                        |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                   |

---

### 6.6 `POST /applications/{applicationId}/reviewer-recommendations` — Submit Reviewer Recommendations

Creates a new recommendation record for the application. Each call appends to the recommendation history. The backend saves `createdDateTime` server-side; do not accept it from the client.

**Path Parameter:**

| Parameter       | Type        | Description           |
| --------------- | ----------- | --------------------- |
| `applicationId` | UUID string | ID of the application |

**Request Body:**

```json
{
  "reviewers": [
    {
      "priority": "integer (starts from 1; lower = higher priority)",
      "name": "string",
      "email": ["string"],
      "remarks": "string | null"
    }
  ]
}
```

**Validation Rules:**

- `reviewers` must be a non-empty array.
- `priority` values must be unique within the request and form a contiguous sequence starting from 1.
- Each reviewer must have at least one entry in `email`.

**Response (`201 Created`):**

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
        "priority": "integer (starts from 1; lower = higher priority)",
        "name": "string",
        "email": ["string"],
        "remarks": "string | null (DC's reason for recommending this reviewer)"
      }
    ]
  },
  "error": null
}
```

> **Implementation Note (State Management Best Practice):**
>
> To ensure optimal frontend performance and adhere to modern API standards, the backend returns the **fully constructed recommendation record**. The frontend must **not** make a subsequent `GET` request to refresh the list after a successful `POST`. Instead, the frontend should inject this returned object directly into its local state manager to instantly update the UI.

**Errors:**

| Code | ErrorType               | Condition                                                                                                   |
| ---- | ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| 400  | `VALIDATION_ERROR`      | `reviewers` is empty; `priority` values are non-unique or non-contiguous; `email` is missing for a reviewer |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                                                                    |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines                                                |
| 404  | `NOT_FOUND`             | Application not found                                                                                       |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                                                                  |

---

### 6.7 `PUT /applications/{applicationId}/final-review` — Save or Submit Final Review

Saves a draft or finalizes the DC's final review for an application. Idempotent — repeated calls with identical data produce the same result.

**Path Parameter:**

| Parameter       | Type        | Description           |
| --------------- | ----------- | --------------------- |
| `applicationId` | UUID string | ID of the application |

**Request Body:**

```json
{
  "score": "integer",
  "remarks": "string",
  "isFinalized": "boolean"
}
```

**Validation Rules:**

- **`score` and `remarks` must be non-null**, when `isFinalized = true`. However, for drafts (`isFinalized = false`), either one of them can be null, but not both.
- When `isFinalized = false` (draft): No restriction on initial review completion.
- When `isFinalized = true`:
  - At least `minIniitalReviewerCount` confirmed reviewers must have `reviewStatus = '已完成'`. Backend must enforce this; return `409 CONFLICT` if not.
- Once an application is `已完成` (previously finalized), any subsequent `PUT` must be rejected with `409 CONFLICT`.

**Response (`200 OK`):**

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

> **Implementation Note (State Management Best Practice):**
>
> Similar to reviewer recommendations, the backend returns the **fully updated final review object**. The frontend should directly update its local state using this response rather than making a redundant `GET` request.

**Errors:**

| Code | ErrorType               | Condition                                                                                                          |
| ---- | ----------------------- | ------------------------------------------------------------------------------------------------------------------ |
| 400  | `VALIDATION_ERROR`      | `score` or `remarks` is null or missing                                                                            |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                                                                           |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines                                                       |
| 404  | `NOT_FOUND`             | Application not found                                                                                              |
| 409  | `CONFLICT`              | `isFinalized = true` but the minimum required initial reviews are not complete; or application is already `已完成` |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                                                                         |

---

## 7. Reference: 預計入選名額 Calculation

The `預計入選名額` for each discipline is a "soft" recommended quota — a guidance figure for the DC when deciding how many applicants to rate highly.

**Formula:**

```markdown
整體入選期望值 (`overallExpectedRatio`) = fixedQuota / totalApplicationsCount (總申請人數)

預計入選名額 (per discipline, `projectedQuotaRaw`) = 該學門申請數 (discipline's `totalApplicationsCount`) × `overallExpectedRatio`

最終預計入選人數 (`projectedQuotaRounded`) = round(`projectedQuotaRaw`)
```

**Fairness rule:** If a discipline's raw quota (`projectedQuotaRaw`) is `< 1.0`, the DC may still recommend 1 highly-rated applicant for that discipline to ensure representational fairness.

Both `overallExpectedRatio` and per-discipline projected quotas are computed and returned by the backend. Frontend must not recompute them.
