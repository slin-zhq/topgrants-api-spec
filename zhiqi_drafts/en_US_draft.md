# TOP Grants 學門召集人作業平台 API Spec

Base URL: `/api/v1`. _(zhiqi: Should we do versioning? What's the best naming convention?)_

In line with REST best practices, the API is organized around key resources that map to user flows in the app.

> **Tip:** According to REST best pracitices, PUT is used to replace an entire resource or create it if it doesn't exist, while PATCH is for partially updating a resource. POST is used to submit data to create a new resource or add to an existing one. _Souce: [GeeksForGeeks](https://www.geeksforgeeks.org/java/what-is-the-difference-between-put-post-and-patch-in-restful-api/)_.

# Resources

- `users` (or Discipline Convener)
- `grant-cycles`: Only one grant cycle can be active at a time. Each grant cycle is unique to individual users, since a user may need to review:
  - Either or both of `phdCandidate` and `youngScholar` applications.
  - One or more academic disciplines.
- `disciplines`
- `applications`: Types of applications – `phdCandidate` and `youngScholar`.
- `recommendation-records`
- `initial-reviews`
- `final-reviews`

# Entity Relationships

- **User (1) - (N) Grant Cycles**: A user can participate in many grant cycles; however, only one cycle can be active at a time.
- **Grant Cycle (1) - (N) Applications**: A grant cycle can have multiple applications.
- **Grant Cycle (1) - (N) Disciplines - (1) User**: A grant cycle can have multiple disciplines corresponding to the user.
- **Application (1) - (N) Recommendation Records**: An application can have many recommendation records, created by the user assigned as final evaluator on this application.
- **Application (1) - (1) Discipline**: An application can belong to one discipline.
- **Discipline (1) - (N) Applications**: A discipline can have many applications.
- **Application (1) - (N) Initial Reviews - (1) Final Review**: An application can review many initial reviews, but only one final review.

# Endpoints, Requests, and Responses

_IMPORTANT:_ ~~Frontend assumes that there would be an API gateway, which acts a single point of entry for the frontend, with proper differentiation of requests, and identifying the applicationt track (`phdCandidate` or `yougnScholar`) and responding with data from one of the backend systems.~~ No, simpler for frontend to call respective endpoints – easier workload for backend. Add a note that this current design is to reduce backend workload as much as possible.

_Note_: Every API call will be made with authenticated token in the request header. So, in the discussion that follows, preassume that the token is present.

Presume that the system is open only after all applications have been submitted.

- Only refreshing on the home page will fetch data from backend.

## 1. Common Interfaces

### Base Response Interface

```json
{
  "status": "string", // "success" | "error"
  "code": "number",
  "timestamp": "string", // ISO 8601 format
  "apiVersion": "string",
  "data": {}, // Can be null, if there's error.
  "error": {} // Cannot be null, if there's error.
}
```

### Error Interface

```typescript
interface APIError {
  type: ErrorType;
  message: string;
  details?: ValidationErrorDetail[];
}

interface ValidationErrorDetail {
  field?: string;
  message: string;
  //   position?: number;
}

type ErrorType =
  | "VALIDATION_ERROR"
  | "AUTHENTICATION_ERROR"
  | "AUTHORIZATION_ERROR"
  | "NOT_FOUND"
  | "QUERY_EXECUTION_ERROR"
  | "DATABASE_ERROR"
  | "AI_SERVICE_ERROR"
  | "EXPORT_ERROR"
  | "TIMEOUT_ERROR"
  | "RATE_LIMIT_ERROR"
  | "INTERNAL_SERVER_ERROR";
```

## 2. Authentication

**`POST /sessions`: User login.**

Request:

```json
{
    "username": string,
    "password": string
}
```

Response:

```json
{
    ...APIResponse,
    data: {
        "sessionToken": "string",
        "userId": "string",
        "expiresInHours": "number" // Frontend can use this to set token expiry time, and auto-logout accordingly
    }
}
```

**`DELETE /sessions`: User logout.**

Request:

```json
{
  // No body needed, just Auth header with session token
}
```

Response:

```json
{
    ...APIResponse
}
```

## Authentication Headers

All authenticated endpoints require:

```
Authorization: Bearer <sessionToken>
Content-Type: application/json
```

Unauthenticated endpoints:

- `POST /sessions`

---

## Error Handling

| Code | Meaning               | Common Use Projects                   |
| ---- | --------------------- | ------------------------------------- |
| 200  | OK                    | Successful GET/PUT/PATCH/DELETE       |
| 201  | Created               | Successful POST for resource creation |
| 202  | Accepted              | Request accepted for async processing |
| 400  | Bad Request           | Invalid parameters, malformed request |
| 401  | Unauthorized          | Missing/invalid session token         |
| 403  | Forbidden             | Insufficient permissions              |
| 404  | Not Found             | Resource not found                    |
| 408  | Request Timeout       | Query/AI generation timeout           |
| 429  | Too Many Requests     | Rate limit exceeded                   |
| 500  | Internal Server Error | Unexpected backend error              |

## 3. "Welcome", "Home", and part of "Application" pages

**`GET /users/me/grant-cycle`**

This API call is made to get data sufficient enough to display "Welcome" and "Home" pages, and to enable users to interact.

It's expected that in serving this endpoint, the API Gateway consolidates data from both `phdCandidate` and `youngScholar` systems.

[INSERT WELCOME PAGE UI] [INSERT HOME PAGE UI]

_Note:_ Application data are requested partially to respect the design decision of getting all data (not coupled with backend operation admin modifications) in one go.

**Request body:**

```json
{
  // No body needed, just Auth header with session token
}
```

**Response:**

```json
{
  // Extending on the base interface...
  "data": {
    // Note: Flattened "Home" page data, frontend will compute stats
    "id": "string", // Grant cycle's id; Please use UUID.
    "year": "integer", // Ask AI: Or, should this be string?
    "deadlineDateTime": "string", // ISO 8601 format
    "applicationTracks": {
      "phdCandidate": {
        "fixedQuota": "integer", // "核定名額"
        "evaluationRubrics": [
          // Same rubrics for all disciplines in this track
          {
            "title": "string",
            "fullScore": "integer"
          }
        ],
        "disciplines": [
          {
            "name": "string", // Do ensure in the backend that names are unique, i.e., no two disciplines share the same name. If name different, separate disciplines.
            "minNumInitialReviewers": "integer"
          }
        ],
        "applications": [
          {
            "id": "string", // Please use UUID.
            "disciplineAppliedFor": "string",
            "submittedDateTime": "string", // ISO 8601 format
            "status": "待推薦 | 待初審 | 待複審 | 待送出 | 已完成",
            "nameZhTW": "string",
            "nameEn": "string",
            "gender": "string",
            "email": "string",
            "institutionAndDepartmentZhTw": "string",
            "institutionAndDepartmentEn": "string",
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
                "startYearAndMonth": "string (YYYY/MM format)",
                "endYearAndMonth": "string (YYYY/MM format)"
              }
            ],
            "attachments": {
              "論文計畫書": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "著作目錄": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "博士候選人資格證明": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "博士班歷年成績單": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "已出版著作或相關文件": [
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
                "advisorEvaluation": {
                  "researchPotential": "string",
                  "thesisContent": "string",
                  "thesisAdivisingMethod": "string",
                  "thesisProgressInPercentages": "integer" // integer from 0 to 100
                }
              }
            ],
            "recommendationRecords": [
              // Please sort by `createdDateTime` descending, i.e., most recent at the top
              {
                "createdDateTime": "string", // (ISO 8601 format)"
                "reviewers": [
                  {
                    "priority": "integer", // starts from 1, the smaller the number, the higher the priority
                    "name": "string",
                    "email": ["string"],
                    "remarks": "string?" // the reason for recommending this reviewer; can be empty
                  }
                ]
              }
            ],
            "finalReview": {
              "score": "integer?",
              "remarks": "string?",
              "isFinalized": "boolean",
              "updatedDateTime": "string?" //  (ISO 8601 format)"; Can be empty if no changes/saves yet.
            }
          }
        ]
      },
      "youngScholar": {
        "fixedQuota": "integer", // "核定名額"
        "evaluationRubrics": [
          // Same rubrics for all disciplines in this track
          {
            "title": "string",
            "fullScore": "integer"
          }
        ],
        "disciplines": [
          {
            "name": "string", // Do ensure in the backend that names are unique, i.e., no two disciplines share the same name. If name different, separate disciplines.
            "minNumInitialReviewers": "integer"
          }
        ],
        "applications": [
          {
            "id": "string", // Please use UUID.
            "disciplineAppliedFor": "string",
            "submittedDateTime": "string", // ISO 8601 format
            "status": "(same as `phdCandidate`)",
            "nameZhTW": "string",
            "nameEn": "string",
            "gender": "string",
            "email": "string",
            "institutionAndDepartmentZhTw": "string",
            "institutionAndDepartmentEn": "string",
            "publicationAbstractZhTw": "string",
            "publicationTitleEn": "string",
            "publicationAbstractZhTW": "string",
            "publicationAbstractEn": "string",
            "expectedImpact": "string",
            "applicantEduBackground": [
              // Same as `phdCandidate`
            ],
            "applicantWorkExperience": [
              // UI Label: "現職與專長相關之經歷"
              {
                "institution": "string",
                "department": "string",
                "jobTitle": "string",
                "startYearAndMonth": "string (YYYY/MM format)",
                "endYearAndMonth": "string (YYYY/MM format)"
              }
            ],
            "attachments": {
              "計畫書": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "著作目錄": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "博士學位證書": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "在職證明/合約影本": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "任職機構主管推薦函": {
                "fileName": "string",
                "fileUrl": "string"
              },
              "已出版著作或相關文件": [
                // Same as `phdCandidate`
              ]
            },
            "recommendationRecords": [
              // Same as `phdCandidate`
            ],
            "finalReview": {
              // Same as `phdCandidate`
            }
          }
        ]
      }
    }
  }
}
```

### Application Status Changes

[TODO] along with logic

## 4. Getting Initial Reviewers (Confirmed by Backend Operation Admin) and Review Data

**`GET /applications/{applicationId}/initial-reviewers`**

**Request body:**

```json
{
  // No body needed, just Auth header with session token
}
```

**Response:**

```json
{
  "data": {
    "initialReviews": [
      // This list can be empty/null, if there's no confirmed reviewer yet.
      {
        "reviewerName": "string",
        "reviewerEmail": ["string"],
        "status": "未完成｜已完成",
        "reviewDetails": {
          // This can be null, if reviewer has agreed, but hasn't finished reviewing
          "statusUpdateDateTime": "string", // (ISO 8601 format)
          "scores": ["integer"], // List is as long as the "evaluationRubrics"; must match by index
          "remarks": "string"
        }
      }
    ]
  }
}
```

## 5. Creating New Reviewer Recommendation Records

**`POST /application/{applicationId}/reviewer-recommendations`**

**Request body:**

```json
// In addition to AUTH header with session token
{
  "reviewers": [
    // Backend must save `createdDateTime`
    {
      "priority": "integer", // starts from 1, the smaller the number, the higher the priority
      "name": "string",
      "email": ["string"],
      "remarks": "string" // the reason for recommending this reviewer; can be empty
    }
  ]
}
```

**Response:**

```json
{
  // Base API Response; upon error, Frontend will prompt user to retry.
}
```

## 6. Saving Final Review Comment

**`PUT /applications/{applicationID}/final-review`**

**Request body:**

```json
// In addition to AUTH header with session token
{
  "score": "integer?",
  "remarks": "string?",
  "isFinalized": "boolean"
}
```

**Response:**

```json
{
  // Base API Response; upon error, Frontend will prompt user to retry.
}
```
