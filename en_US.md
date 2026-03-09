# TOP Grants — Discipline Convener Platform API Specification

**Scope:** Covers the Discipline Convener (DC / 學門召集人) client only. Admin-side and applicant-side endpoints are out of scope.

**Base URL:** `/api/v1`

**Versioning:** URL-path versioning (`/v1`, `/v2`, …). Breaking changes increment the major version. All current development targets `v1`.

---

## 1. Design Principles

- Every endpoint except `POST /sessions` and `POST /sessions/refresh` requires an `Authorization: Bearer <sessionToken>` header.
- The system is available only after all applicant submissions are closed. Write operations regarding discipline convener affairs are rejected before this window opens.
- Home/Welcome page data is loaded in a single bulk request (`GET /users/me/grant-cycle`) to minimize round-trips. Initial reviewer detail (scores, comments) is fetched on demand per application.
- Each `POST` to `/applications/{id}/reviewer-recommendations` **appends** a new recommendation record. It does not replace prior records.
- The backend computes and owns each application's `status`. The DC user cannot set it directly.
- `POST` creates a new sub-resource. `PUT` replaces an existing resource (idempotent). `DELETE` removes a resource.

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

| ErrorType               | When Used                                                                                                                              |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `VALIDATION_ERROR`      | Request body or query parameter failed validation                                                                                      |
| `AUTHENTICATION_ERROR`  | Token missing, expired, or invalid                                                                                                     |
| `CAPTCHA_ERROR`         | CAPTCHA token missing, invalid, score too low, or expired                                                                              |
| `AUTHORIZATION_ERROR`   | Authenticated user lacks permission for this resource                                                                                  |
| `NOT_FOUND`             | Requested resource does not exist                                                                                                      |
| `CONFLICT`              | Request conflicts with the current resource state (e.g., finalizing review before all initial reviews are complete) [TODO: to discuss] |
| `QUERY_EXECUTION_ERROR` | Database query failed                                                                                                                  |
| `TIMEOUT_ERROR`         | Backend operation timed out                                                                                                            |
| `RATE_LIMIT_ERROR`      | Too many requests from the client                                                                                                      |
| `INTERNAL_SERVER_ERROR` | Unexpected server-side failure                                                                                                         |

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

### 4.1 CAPTCHA

**Who handles CAPTCHA:** It is a split responsibility.

- **Frontend** loads the CAPTCHA provider's script, invokes it on login form submit to obtain a one-time `captchaToken`, and sends that token alongside credentials.
- **Backend** never generates or renders CAPTCHA challenges. It receives `captchaToken`, verifies it server-to-server (a private POST to the CAPTCHA provider's verification API), and accepts or rejects the login accordingly.

**Recommended provider:** [Google reCAPTCHA v3](https://developers.google.com/recaptcha/docs/v3) (invisible, score-based — no user interaction needed) or [hCaptcha](https://www.hcaptcha.com) (privacy-first alternative with equivalent API behavior).

> ⚠️ **Open Decision:** Confirm the CAPTCHA provider before implementation. Frontend and backend must use the matching site key (public, frontend) and secret key (private, backend) from the same provider.

**reCAPTCHA v3 token flow:**

1. Frontend loads `<script src="https://www.google.com/recaptcha/api.js?render=SITE_KEY"></script>`.
2. On login submit: call `grecaptcha.execute(SITE_KEY, { action: 'login' })` → resolves to `captchaToken`.
3. Include `captchaToken` in `POST /sessions` request body.
4. Backend calls `POST https://www.google.com/recaptcha/api/siteverify` with the token and secret key.
5. Backend checks that `score >= 0.5` (threshold configurable). If not, return `400 CAPTCHA_ERROR`.

---

### `POST /sessions` — Login

No `Authorization` header required.

**Request Body:**

```json
{
  "username": "string",
  "password": "string",
  "captchaToken": "string"
}
```

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
| 429  | `RATE_LIMIT_ERROR`     | Too many failed login attempts from this IP/user                      |

---

### `DELETE /sessions` — Logout

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

### `POST /sessions/refresh` — Refresh Session Token

Extends the session without re-login. Frontend should call this proactively before `expiresAt` (e.g., 5 minutes before expiry). No `Authorization` header required; uses the `refreshToken` issued at login.

**Request Body:**

```json
{
  "refreshToken": "string"
}
```

**Response (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "sessionToken": "string (JWT)",
    "refreshToken": "string (rotated — previous token is immediately invalidated)",
    "expiresAt": "string (ISO 8601)"
  },
  "error": null
}
```

> Refresh tokens are rotated on each use to mitigate replay attacks.

**Errors:**

| Code | ErrorType              | Condition                                      |
| ---- | ---------------------- | ---------------------------------------------- |
| 401  | `AUTHENTICATION_ERROR` | `refreshToken` is missing, invalid, or expired |

---

### Authenticated Request Headers

Required on all endpoints except `POST /sessions` and `POST /sessions/refresh`:

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

---

## 5. Application Status State Machine

The backend computes and owns each application's `status`. The DC cannot set it directly; it changes in response to events from reviewers, admins, and the DC's own actions.

### 5.1 Status Values

| Value                    | Display (ZhTW) | Meaning                                                                                                                                                                                       |
| ------------------------ | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PENDING_RECOMMENDATION` | 待推薦         | DC has not yet submitted any reviewer recommendations, **or** there is not yet any recommendation record.                                                                                     |
| `PENDING_INITIAL_REVIEW` | 待初審         | At least one recommendation record is present. There can be no confirmed reviewers yet. Or, if there is any, not all confirmed reviewers have completed their reviews. DC has no saved draft. |
| `PENDING_FINAL_REVIEW`   | 待複審         | At least `minNumInitialReviewers` confirmed reviewers have submitted their reviews. DC has not yet saved any draft.                                                                           |
| `DRAFT_FINAL_REVIEW`     | 待送出         | DC has saved a draft final review (`isFinalized = false`). Initial reviews may or may not all be complete; finalization is blocked until they are.                                            |
| `COMPLETED`              | 已完成         | DC has submitted and finalized the review (`isFinalized = true`).                                                                                                                             |

### 5.2 Transition Conditions

```
PENDING_RECOMMENDATION
  ─► PENDING_INITIAL_REVIEW
        When: DC submits at least one reviewer recommendation.
  ─► DRAFT_FINAL_REVIEW
        When: DC saves a draft before submitting any recommendations
              [edge case — uncommon but allowed]

PENDING_INITIAL_REVIEW
  ─► PENDING_FINAL_REVIEW
        When: At least `minNumInitialReviewers` confirmed reviewers' reviewStatus = "已完成"
              AND DC has no saved draft
  ─► DRAFT_FINAL_REVIEW
        When: DC saves a draft before the minimum required initial reviews are complete

PENDING_FINAL_REVIEW
  ─► PENDING_INITIAL_REVIEW  (regression)
        When: A completed initial review is revoked/invalidated and completed count drops below `minNumInitialReviewers`
  ─► DRAFT_FINAL_REVIEW
        When: DC saves a draft (PUT /final-review, isFinalized = false)
  ─► COMPLETED
        When: DC finalizes without a prior draft (PUT /final-review, isFinalized = true)

DRAFT_FINAL_REVIEW
  ─► COMPLETED
        When: DC submits (PUT /final-review, isFinalized = true)
              AND at least `minNumInitialReviewers` confirmed reviewers' reviewStatus = "已完成"
              Backend must reject with 409 CONFLICT if this condition is not met.
```

### 5.3 Business Rules

1. DC **may save a draft** (`isFinalized = false`) at any time, even before all initial reviews are complete.
2. DC **cannot finalize** (`isFinalized = true`) unless at least `minNumInitialReviewers` reviewers have completed their initial reviews. Backend must enforce this with `409 CONFLICT`.
3. Once an application reaches `COMPLETED`, no further edits are accepted. Subsequent `PUT /final-review` calls must be rejected with `409 CONFLICT`.
4. If a reviewer who has already completed their review (`reviewStatus = "已完成"`) is later revoked at the admin level (edge case), the backend must re-evaluate the application status accordingly.

---

## 6. Endpoints

### 6.1 `GET /users/me/grant-cycle` — Home & Welcome Page Data

Returns all data needed to render the Welcome and Home pages and enable all DC user interactions. Both tracks and all assigned disciplines are included in one response. Per-application initial reviewer detail (scores, comments) is fetched separately via §6.2.

**Request Body:** None.

**Response (`200 OK`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {
    "grantCycleId": "string (UUID)",
    "year": "integer",
    "deadlineDateTime": "string (ISO 8601, must include time and UTC offset)",
    // "grantTitle": "國科會補助人文及社會科學研究海外人才培育計畫",
    "applicationTracks": {
      "phdCandidate": {
        "fixedQuota": "integer (核定名額)",
        // "totalApplicationsCount": "integer",
        // "overallSelectionRatio": "number (float; = fixedQuota / totalApplicationsCount, e.g., 0.42)",
        "evaluationRubrics": [
          {
            "title": "string",
            "fullScore": "integer"
          }
        ],
        "disciplines": [
          {
            "name": "string (must be unique within this track and cycle)",
            "minNumInitialReviewers": "integer"
            // "expectedValue": "number (float, 2 decimal places; = discipline applicant count × overallSelectionRatio)"
          }
        ],
        "applications": [
          {
            "id": "string (UUID)",
            "disciplineAppliedFor": "string (matches a discipline name in the disciplines list above)",
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
                "endYearAndMonth": "string (YYYY/MM) | null (null if currently enrolled)"
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
                    "priority": "integer (starts from 1; lower = higher priority)",
                    "name": "string",
                    "email": ["string"],
                    "remarks": "string | null (DC's reason for recommending this reviewer)"
                  }
                ]
              }
            ],
            "finalReview": {
              "score": "integer | null",
              "remarks": "string | null",
              "isFinalized": "boolean",
              "updatedDateTime": "string (ISO 8601) | null (null if no save has occurred)"
            }
          }
        ]
      },
      "youngScholar": {
        "fixedQuota": "integer",
        // "totalApplicationsCount": "integer",
        // "overallSelectionRatio": "number (float)",
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
            // "expectedValue": "number (float, 2 decimal places)"
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

> **Implementation Notes:**
>
> - If the DC is assigned to only one track, the other track's `applications` array will be empty (not absent).
> - `recommendationRecords` is sorted by `createdDateTime` descending (most recent record first).
> - `disciplines` — backend must enforce unique names within each track and cycle.
> - `youngScholar` applications do not have `advisors`; omit the field entirely for this track.

<!-- > - `invitationStatus` within each recommendation record's reviewer entries is maintained by the backend/admin side and will reflect the latest status at the time the DC refreshes. -->
<!-- > - `overallSelectionRatio = fixedQuota / totalApplicationsCount` (computed by backend, not frontend).
> - `expectedValue` per discipline `= disciplineApplicantCount × overallSelectionRatio`, rounded to 2 decimal places. If `expectedValue < 1.0`, the DC may still recommend 1 applicant per fairness rules (§3 of the requirements). -->

**Errors:**

| Code | ErrorType               | Condition                                                            |
| ---- | ----------------------- | -------------------------------------------------------------------- |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                             |
| 403  | `AUTHORIZATION_ERROR`   | User is not a DC, or has no assigned disciplines in the active cycle |
| 404  | `NOT_FOUND`             | No active grant cycle exists                                         |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                           |

---

### 6.2 `GET /applications/{applicationId}/initial-reviewers` — Initial Reviewer Details

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
          //   "totalScore": "integer | null",
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
> - `scores` array length must match the number of rubric items in the track's `evaluationRubrics` (from §6.1), ordered by rubric index.

<!-- > - `totalScore` must equal the sum of `scores`. Backend should compute and return this; do not rely on frontend summation. -->

**Errors:**

| Code | ErrorType               | Condition                                                    |
| ---- | ----------------------- | ------------------------------------------------------------ |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                     |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines |
| 404  | `NOT_FOUND`             | Application not found                                        |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                   |

---

### 6.3 `POST /applications/{applicationId}/reviewer-recommendations` — Submit Reviewer Recommendations

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
  <!-- - If a reviewer in the new list was previously in an earlier record with `invitationStatus` of `邀請中` or `已邀請`, the backend should flag their status as `取消中` — their prior invitation is being superseded. -->
  <!-- - If a reviewer previously confirmed (`invitationStatus = '已同意'`) is **omitted** from the new list, the backend must reject the request with `409 CONFLICT`. A confirmed reviewer cannot be removed from the active cycle. -->

**Response (`201 Created`):**

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

**Errors:**

| Code | ErrorType               | Condition                                                                                                   |
| ---- | ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| 400  | `VALIDATION_ERROR`      | `reviewers` is empty; `priority` values are non-unique or non-contiguous; `email` is missing for a reviewer |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                                                                    |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines                                                |
| 404  | `NOT_FOUND`             | Application not found                                                                                       |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                                                                  |

<!-- | 409  | `CONFLICT`              | The new list omits a reviewer who has already confirmed (`invitationStatus = '已同意'`)                     | -->

---

### 6.4 `PUT /applications/{applicationId}/final-review` — Save or Submit Final Review

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

- **`score` and `remarks` must be non-null** even for drafts (`isFinalized = false`).
- When `isFinalized = false` (draft): No restriction on initial review completion.
- When `isFinalized = true`:
  - At least `minNumInitialReviewers` confirmed reviewers must have `reviewStatus = '已完成'`. Backend must enforce this; return `409 CONFLICT` if not.
- Once an application is `COMPLETED` (previously finalized), any subsequent `PUT` must be rejected with `409 CONFLICT`.

**Response (`200 OK`):**

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

**Errors:**

| Code | ErrorType               | Condition                                                                                                             |
| ---- | ----------------------- | --------------------------------------------------------------------------------------------------------------------- |
| 400  | `VALIDATION_ERROR`      | `score` or `remarks` is null or missing                                                                               |
| 401  | `AUTHENTICATION_ERROR`  | Invalid or expired token                                                                                              |
| 403  | `AUTHORIZATION_ERROR`   | Application does not belong to the DC's assigned disciplines                                                          |
| 404  | `NOT_FOUND`             | Application not found                                                                                                 |
| 409  | `CONFLICT`              | `isFinalized = true` but the minimum required initial reviews are not complete; or application is already `COMPLETED` |
| 500  | `INTERNAL_SERVER_ERROR` | Unexpected backend failure                                                                                            |

<!-- ---

(zhiqi: 2026/03/10) The following is kept for spec formatting reference, only. Invitation status tracking is invalid now.

## 7. Reference: Reviewer Invitation Statuses

These statuses are set by backend/admin. The DC sees them read-only via `recommendationRecords`.

| Status   | Meaning                                                                                                    |
| -------- | ---------------------------------------------------------------------------------------------------------- |
| `邀請中` | DC submitted the recommendation; admin is processing it.                                                   |
| `已邀請` | Admin has sent the invitation to the reviewer.                                                             |
| `已同意` | Reviewer accepted. They will review the application.                                                       |
| `已拒絕` | Reviewer declined. `invitationStatusRemarks` contains their reason.                                        |
| `已回避` | Reviewer recused themselves. `invitationStatusRemarks` contains their reason.                              |
| `取消中` | The DC submitted a new recommendation list that omits this reviewer; admin is processing the cancellation. |
| `已取消` | Cancellation confirmed. Reviewer will not review this application.                                         | -->

<!-- ---

(zhiqi: 2026/03/10) Frontend will take care.

## 7. Reference: Initial Review Statuses

These statuses are computed by the backend based on reviewer activity.

| Status   | Meaning                                                                                                                                |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `未開始` | Reviewer confirmed (`已同意`) but has not yet started.                                                                                 |
| `進行中` | Reviewer has saved at least one draft.                                                                                                 |
| `已完成` | Reviewer submitted their review. Scores and remarks are visible to DC. Once submitted, this status is final and cannot be rolled back. | -->

---

## 7. Reference: 期望值 (Expected Value) Calculation

> Note: This is shared for knowledge, only. Frontend can handle the calculation, while requiring `expectedValue` from backend.

The `expectedValue` for each discipline is a "soft" recommended quota — a guidance figure for the DC when deciding how many applicants to rate highly.

**Formula:**

```
overallSelectionRatio = fixedQuota / totalApplicationsInTrack

expectedValue (per discipline) = disciplineApplicantCount × overallSelectionRatio
```

**Fairness rule:** If a discipline's `expectedValue < 1.0`, the DC may still recommend 1 highly-rated applicant for that discipline to ensure representational fairness.

Both `overallSelectionRatio` and per-discipline `expectedValue` are computed and returned by the backend. Frontend must not recompute them.

<!-- ---

## 9. Open Decisions

| #   | Question                                                                               | Recommendation                                                                                                                                                                                                                                                                         |
| --- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Which CAPTCHA provider?                                                                | Google reCAPTCHA v3 (seamless, invisible). Must be confirmed before implementation; frontend and backend must use the same provider.                                                                                                                                                   |
| 2   | Does the DC need to view or update their own profile?                                  | If yes, add `GET /users/me` and `PATCH /users/me`. Not in current scope.                                                                                                                                                                                                               |
| 3   | Is there a smart recommendation endpoint needed?                                       | Requirements §2.3 defines this as `{}` (TBD). Clarify scope and add here when ready.                                                                                                                                                                                                   |
| 4   | `youngScholar` advisors                                                                | Requirements data model does not include `advisors` for `youngScholar`. This spec omits them. Confirm.                                                                                                                                                                                 |
| 5   | Can a DC explicitly cancel a single recommendation without resubmitting the full list? | Currently, cancellation is implicit — any reviewer absent from the new list who was previously `邀請中` or `已邀請` is marked `取消中`. If explicit per-reviewer cancellation is needed, a `DELETE /applications/{id}/reviewer-recommendations/{reviewerId}` endpoint should be added. | -->
