
# AlgoTrack — API Contracts

**Status:** Draft v1 — pending review  
**API version:** v1  
**Base path:** `/api/v1`  
**Content type:** `application/json`  
**Authentication:** Firebase Authentication  
**Database:** SQL Server  
**Related issue:** #3 — API contracts

## 1. API Conventions

### 1.1 Naming and data formats

- Use REST-style resource paths.
- Use camelCase for JSON properties.
- The physical schema currently uses SQL Server `BIGINT` identifiers. Finalize the JSON/path representation before implementation; do not silently treat these IDs as UUIDs. Because the frontend uses JavaScript, prefer decimal strings at the API boundary if numeric precision could be a concern.
- Represent timestamps as ISO 8601 UTC strings.
- Use UTF-8 JSON.
- Never accept an owner user ID from the client as proof of ownership.
- Never return internal exception messages, SQL details, stack traces, or secrets.

### 1.2 Authentication

Protected requests must include:

`Authorization: Bearer <firebase-id-token>`

The API must validate the Firebase ID token using Firebase Admin SDK or an equivalent trusted server-side verification mechanism.

The API must:
- Validate token signature, issuer, audience, expiry, and other required claims.
- Derive the Firebase UID from the verified token.
- Resolve the application profile using that UID.
- Require verified email for protected application operations.
- Enforce resource ownership independently of authentication.

A Firebase ID token is not the same as a refresh token. Never accept a client-supplied UID or email as an authentication credential.

### 1.3 Standard error response

Use the Problem Details format for errors.

Example:

{
  "type": "about:blank",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more fields are invalid.",
  "instance": "/api/v1/roadmaps",
  "errors": {
    "name": [
      "Name is required."
    ]
  }
}

The `errors` extension is included when field-level validation details are available.

Do not expose sensitive implementation details in `detail`.

### 1.4 HTTP status conventions

| Status | Meaning |
| --- | --- |
| 200 | Successful request |
| 201 | Resource created |
| 204 | Successful operation with no response body |
| 400 | Invalid request or validation failure |
| 401 | Missing or invalid authentication |
| 403 | Authenticated caller lacks a required permission; proposed for a valid token with unverified email, pending final policy confirmation |
| 404 | Resource does not exist or private resource is not owned by caller |
| 409 | State conflict or duplicate resource |
| 500 | Unexpected server failure |

For a valid token with an unverified email, this draft proposes `403 Forbidden` with a stable error code. This choice remains subject to review.

## 2. Identity and Profile

### GET /api/v1/me

Returns the current user's application profile. Creates the profile on first access if it does not exist.

Authentication: required  
Email verification: required

Response: `200 OK`

{
  "id": "8b42d1a0-0000-4000-8000-000000000001",
  "displayName": "Developer",
  "email": "developer@example.com",
  "createdAt": "2026-11-03T08:00:00Z"
}

Rules:
- Derive identity from the verified Firebase UID.
- Provision the profile idempotently.
- Handle concurrent first requests without creating duplicate profiles.
- Do not return Firebase credentials or tokens in the profile response.

Open decision:
- Which profile fields can users edit, and which fields are synchronized from Firebase?

## 3. Roadmaps

### GET /api/v1/roadmaps

Lists the current user's active roadmaps.

Query parameters:
- `page`: proposed default `1`.
- `pageSize`: proposed default `20`, maximum `100`.

Response: `200 OK`

{
  "items": [
    {
      "id": "roadmap-id",
      "name": "DSA Mastery",
      "description": "Practice core algorithms",
      "createdAt": "2026-11-03T08:00:00Z",
      "updatedAt": "2026-11-03T08:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1
}

### POST /api/v1/roadmaps

Creates a roadmap owned by the current user.

Request:

{
  "name": "DSA Mastery",
  "description": "Practice core algorithms"
}

Rules:
- `name` is required and must satisfy the agreed length limit.
- `description` is optional and nullable.
- Owner identity comes from the authenticated application profile.
- Validate the request before persistence.

Response: `201 Created`, including the created resource and its location.

### GET /api/v1/roadmaps/{roadmapId}

Returns a roadmap and its summary.

Response: `200 OK`

Rules:
- Return `404` if the roadmap does not exist or belongs to another user.
- Do not reveal whether another user's roadmap exists.

### PUT /api/v1/roadmaps/{roadmapId}

Updates the roadmap's editable fields.

Request:

{
  "name": "DSA Mastery — Updated",
  "description": "Revised learning plan"
}

Response: `200 OK`

### POST /api/v1/roadmaps/{roadmapId}/archive

Archives the roadmap instead of hard-deleting it.

Response: `204 No Content`

### POST /api/v1/roadmaps/{roadmapId}/restore

Restores an archived roadmap.

Response: `204 No Content`

Open questions:
- Should repeated archive/restore requests return success or `409 Conflict`?

## 4. Roadmap Topics

### GET /api/v1/roadmaps/{roadmapId}/topics

Returns active topic links in roadmap order.

Response: `200 OK`

{
  "items": [
    {
      "roadmapTopicId": "roadmap-topic-id",
      "topicId": "topic-id",
      "title": "Arrays",
      "progress": "InProgress",
      "sortOrder": 1
    }
  ]
}

Allowed progress values:
- `NotStarted`
- `InProgress`
- `Completed`

These values are approved for the MVP.

### POST /api/v1/roadmaps/{roadmapId}/topics

Adds an existing shared topic to a roadmap.

Request:

{
  "topicId": "topic-id"
}

Response: `201 Created`

Rules:
- The topic must exist and satisfy the approved availability policy.
- The link belongs to the specified roadmap.
- Duplicate active links must not be created.
- Progress initially defaults to `NotStarted`.

### PATCH /api/v1/roadmaps/{roadmapId}/topics/{roadmapTopicId}/progress

Request:

{
  "progress": "Completed"
}

Response: `200 OK`

Rules:
- Accept only the three defined progress values.
- Verify roadmap ownership and link membership.
- Do not update a link belonging to another roadmap.

### PATCH /api/v1/roadmaps/{roadmapId}/topics/reorder

Proposed request:

{
  "orderedRoadmapTopicIds": [
    "roadmap-topic-a",
    "roadmap-topic-b",
    "roadmap-topic-c"
  ]
}

Response: `204 No Content`

The server must verify that the supplied IDs form a valid ordering of the intended active links. Exact partial-reorder semantics remain open.

### POST /api/v1/roadmaps/{roadmapId}/topics/{roadmapTopicId}/archive

Archives the roadmap-topic link without deleting the shared topic.

Response: `204 No Content`

### POST /api/v1/roadmaps/{roadmapId}/topics/{roadmapTopicId}/restore

Restores the archived link if restoration does not violate uniqueness constraints.

Response: `204 No Content`

Open questions:
- Can progress be changed while the link is archived?
- What happens when the shared topic itself is archived?
- How should restoration handle an existing active link for the same topic?

## 5. Shared Topics

### GET /api/v1/topics

Lists available shared topics.

Proposed query parameters:
- `search`
- `page`
- `pageSize`

Response: `200 OK`

{
  "items": [
    {
      "id": "topic-id",
      "name": "Arrays",
      "description": "Array data structures",
      "prerequisiteTopicIds": []
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1
}

### GET /api/v1/topics/{topicId}

Returns topic details and prerequisite relationships.

### GET /api/v1/topics/{topicId}/problems

Lists active problems associated with the topic.

Response schemas and filters will follow the agreed Problem representation.

Shared topic writes are not exposed to ordinary users.

Open question:
- Will topic and prerequisite administration remain seed-only, or be included in the admin API?

## 6. Problems

### GET /api/v1/problems

Lists active problems with pagination and filtering.

Proposed query parameters:
- `search`
- `difficulty`
- `topicId`
- `page`
- `pageSize`

Exact difficulty values, supported sort fields, and pagination defaults require approval.

Response: `200 OK`

{
  "items": [
    {
      "id": "problem-id",
      "title": "Two Sum",
      "difficulty": "Easy",
      "description": "Find two numbers that add up to a target.",
      "topicIds": [
        "topic-id"
      ]
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1
}

### GET /api/v1/problems/{problemId}

Returns an active problem.

Rules:
- Exclude soft-deleted problems from ordinary user queries.
- Return `404` for missing or soft-deleted problems.
- Do not expose internal administration metadata.

## 7. Problem Administration

All routes in this section require server-configured admin authorization in addition to authentication and email verification.

### POST /api/v1/admin/problems

Creates a shared problem.

### PUT /api/v1/admin/problems/{problemId}

Updates problem metadata.

### POST /api/v1/admin/problems/{problemId}/soft-delete

Soft-deletes a problem.

### POST /api/v1/admin/problems/{problemId}/restore

Restores a soft-deleted problem.

### PUT /api/v1/admin/problems/{problemId}/topics

Replaces topic associations.

Exact request and response DTOs are pending decisions on required problem fields, audit metadata, and seed reconciliation.

Security rules:
- Never trust a client-provided `isAdmin` flag or role.
- Enforce admin authorization on the server.
- Record administrative changes where audit requirements are approved.
- Prevent unauthorized modification of shared catalog data.

## 8. Practice

### GET /api/v1/me/practice

Lists the current user's practice records.

Proposed query parameters:
- `page`
- `pageSize`

Response: `200 OK`

{
  "items": [
    {
      "problemId": "problem-id",
      "problemTitle": "Two Sum",
      "status": "Solved",
      "solvedAt": "2026-11-03T08:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1
}

### PUT /api/v1/me/practice/{problemId}

Marks a problem as solved.

Request:

{
  "status": "Solved"
}

Response: `200 OK`

Rules:
- `Solved` is the only supported practice status in the MVP.
- The database stores only the current state per user-problem pair; it does not keep a full attempt history.
- Repeated requests must be idempotent.
- Create at most one practice record per user-problem pair.
- Create the initial review schedule only when the problem becomes solved.
- Repeating the request must not reset the review schedule.
- Undoing `Solved` is not supported in the MVP.
- Verify that the problem is active and available for practice.

Open question:
- What should happen if the problem is soft-deleted after the user has already solved it?

## 9. Review Schedules

### GET /api/v1/me/reviews

Lists the current user's review schedules.

Proposed query parameters:
- `status`: proposed values `Upcoming`, `Due`, `Overdue`.
- `page`
- `pageSize`

The exact status calculation and filtering rules require approval.

### POST /api/v1/me/reviews/{problemId}/complete

Completes a review and updates the existing review schedule.

Response: `200 OK`

Proposed response:

{
  "problemId": "problem-id",
  "nextReviewAt": "2026-11-10T08:00:00Z"
}

Rules:
- A schedule must already exist for the current user-problem pair.
- Review completion must not create a second schedule.
- The next review date must follow the approved fixed-interval policy.
- The server, not the client, calculates the next review date.
- Validate repeated or concurrent completion requests.

Open questions:
- What are the exact review intervals?
- Can a review be completed before its due date?
- What happens when a review is completed multiple times?
- How should overdue reviews be handled?

## 10. Authorization Matrix

| Resource/operation | Required access |
| --- | --- |
| Get current profile | Authenticated, verified email |
| Roadmap CRUD and archive/restore | Owner |
| Roadmap-topic operations | Roadmap owner |
| Read shared topics and problems | Authenticated, verified email |
| Update current user's practice | Authenticated, verified email |
| Read/complete current user's reviews | Authenticated, verified email |
| Admin problem operations | Server-configured admin plus authentication requirements |

Every ownership check must be performed on the server.

## 11. Open Decisions Before Approval

1. Repeated archive/restore behavior and archived-roadmap listing semantics.
2. Archived shared-topic behavior and roadmap-topic restoration conflicts.
3. Exact reorder semantics.
4. Problem filters, sorting, difficulty enum, and pagination limits.
5. Problem create/update DTOs and admin audit requirements.
6. Seed reconciliation when admins modify seeded problems.
7. Account deletion and dependent-data lifecycle.
8. Editable profile fields and Firebase profile synchronization.
9. Exact error contract for unverified email.
10. Admin identity configuration and change auditing.
11. Topic/prerequisite admin operations.
12. Fixed review intervals, early completion, overdue behavior, and repeated completion.
13. Handling solved records when a problem is soft-deleted.
14. API representation of SQL Server `BIGINT` IDs and final field constraints against the physical schema.

## 12. Definition of Done

- [ ] Request and response examples are consistent across endpoint groups.
- [ ] Authentication and authorization rules are explicit.
- [ ] Error format and HTTP status conventions are documented.
- [ ] Validation and edge cases are specified.
- [ ] Open decisions are approved or clearly marked as unresolved.
- [ ] Contracts are checked against the physical database design.
- [ ] No application code is included in this documentation step.
