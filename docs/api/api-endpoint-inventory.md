
# AlgoTrack — API Endpoint Inventory

**Status:** Draft for Issue #3 review  
**API style:** REST, JSON over HTTPS  
**Base path:** `/api/v1`  
**Authentication:** Firebase ID token in `Authorization: Bearer <token>`  
**Related issue:** #3 — API contracts

> This inventory is a design proposal, not an implemented API. It lists endpoint intent and access rules before defining complete DTOs and OpenAPI schemas.

## 1. API-wide conventions

- Use `/api/v1` as the versioned base path.
- Use plural resource names and JSON request/response bodies.
- Firebase Authentication handles sign-in in the frontend. The API does not accept passwords or Google credentials as a login endpoint.
- Protected endpoints require a valid Firebase ID token.
- Protected application endpoints also require a verified email.
- Derive the caller's application user ID from the verified Firebase UID. Never trust a caller-supplied `UserId` or `OwnerUserId` for authorization.
- Roadmap endpoints must enforce ownership on every request.
- Admin/catalog maintenance requires server-configured admin authorization.
- Timestamps use ISO 8601 and UTC-normalized values.
- Pagination, filtering, error format, and exact DTOs will be specified in the detailed API contracts.

## 1.1 Confirmed API decisions

- **Profile provisioning:** lazy creation through `GET /api/v1/me`; creation is idempotent and keyed by verified Firebase UID.
- **Private roadmap authorization:** return `404 Not Found` for missing and non-owned private roadmaps to avoid resource-existence disclosure.
- **Practice reversal:** undoing `Solved` is out of scope for MVP. Do not expose an unsolve/delete practice endpoint.
- **Problem administration:** provide a separate `/api/v1/admin/...` API protected by server-configured admin authorization.

## 2. Identity and profile

Firebase performs sign-in directly in the React application. The client sends the Firebase ID token to the API; the API validates it. No API endpoint accepts raw passwords.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/me` | Get the current application profile; create it idempotently on first authenticated access if no profile exists | Authenticated, verified email |

**Confirmed decision — lazy provisioning:** `GET /api/v1/me` creates the application profile on first authenticated access if one does not exist, then returns it. Retries must not create duplicate profiles. The profile is keyed by the verified Firebase UID. Do not accept a client-supplied Firebase UID or use a public registration endpoint that duplicates Firebase Authentication.

**Implementation note:** handle concurrent first requests using a unique constraint on `FirebaseUid` plus safe conflict/re-read behavior.

## 3. Roadmaps

Roadmaps are private and owned by the authenticated user. Archived roadmaps are hidden from normal active listings but retained for history.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/roadmaps` | List the current user's active roadmaps | Owner |
| POST | `/api/v1/roadmaps` | Create a roadmap | Owner |
| GET | `/api/v1/roadmaps/{roadmapId}` | Get roadmap details | Owner |
| PUT | `/api/v1/roadmaps/{roadmapId}` | Update roadmap name/description | Owner |
| POST | `/api/v1/roadmaps/{roadmapId}/archive` | Archive a roadmap | Owner |
| POST | `/api/v1/roadmaps/{roadmapId}/restore` | Restore an archived roadmap | Owner |

The API must not allow a user to access or mutate another user's roadmap by guessing its ID.

**Confirmed decision:** return `404 Not Found` for a roadmap that does not exist, is not visible to the caller, or belongs to another user. Use a consistent response shape to avoid revealing another user's private resource.

## 4. Roadmap topics and progress

Progress belongs to a topic's placement in a specific roadmap. Removing a topic archives the link; it does not delete the shared topic.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/roadmaps/{roadmapId}/topics` | List active topic links in roadmap order | Roadmap owner |
| POST | `/api/v1/roadmaps/{roadmapId}/topics` | Add a shared topic to a roadmap | Roadmap owner |
| PATCH | `/api/v1/roadmaps/{roadmapId}/topics/{roadmapTopicId}/progress` | Update progress to `NotStarted`, `InProgress`, or `Completed` | Roadmap owner |
| PATCH | `/api/v1/roadmaps/{roadmapId}/topics/reorder` | Update the order of roadmap topics | Roadmap owner |
| POST | `/api/v1/roadmaps/{roadmapId}/topics/{roadmapTopicId}/archive` | Archive the roadmap-topic link | Roadmap owner |
| POST | `/api/v1/roadmaps/{roadmapId}/topics/{roadmapTopicId}/restore` | Restore an archived link | Roadmap owner |

**Open detail:** Decide whether reordering uses `PATCH` with a list of IDs and sort orders or a dedicated reorder command DTO. Also decide how archived shared topics affect existing roadmap links.

## 5. Shared topics

Topics are shared catalog data, not owned by an individual user.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/topics` | List active topics; support search/filter if needed | Authenticated, verified email |
| GET | `/api/v1/topics/{topicId}` | Get topic details, including prerequisite relationships | Authenticated, verified email |
| GET | `/api/v1/topics/{topicId}/problems` | List active problems tagged with this topic | Authenticated, verified email |

No topic create/update/delete endpoint is proposed for the public MVP. Topics are maintained through controlled seed/admin processes and archived rather than hard-deleted.

## 6. Problems and admin catalog management

Problems are shared catalog data. Soft-deleted problems are excluded from ordinary user catalog queries.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/problems` | Browse active problems with filtering and pagination | Authenticated, verified email |
| GET | `/api/v1/problems/{problemId}` | Get active problem details | Authenticated, verified email |

Problem catalog writes are available only through a separate server-authorized admin API. These routes must not be exposed as ordinary user CRUD and must require a server-configured admin policy.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| POST | `/api/v1/admin/problems` | Create a catalog problem | Server-configured admin |
| PUT | `/api/v1/admin/problems/{problemId}` | Update problem metadata | Server-configured admin |
| POST | `/api/v1/admin/problems/{problemId}/soft-delete` | Soft-delete a problem | Server-configured admin |
| POST | `/api/v1/admin/problems/{problemId}/restore` | Restore a soft-deleted problem | Server-configured admin |
| PUT | `/api/v1/admin/problems/{problemId}/topics` | Replace the problem's topic associations | Server-configured admin; behavior after practice history remains open |

Admin endpoints are a confirmed requirement, but exact DTOs, audit requirements, and whether seed reconciliation and admin edits may overlap must be finalized in the detailed contracts. Never trust a client-provided role or `isAdmin` field.

## 7. Practice

The MVP stores only the current practice state per user-problem pair. The approved status is `Solved`.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/me/practice` | List the current user's solved problems, with pagination/filtering | Current user |
| PUT | `/api/v1/me/practice/{problemId}` | Mark a problem as solved; idempotently create/update the current state and create the initial review schedule | Current user |

**Confirmed decision — no undo in MVP:** users cannot remove or reverse the `Solved` state through the MVP API. Repeated requests to mark the same problem solved must be idempotent and must not create duplicate practice records or review schedules. Do not expose a delete/unsolve endpoint in the MVP.

## 8. Review schedule

Each user-problem pair has at most one review schedule. A schedule is created only when a problem is marked solved.

| Method | Path | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/api/v1/me/reviews` | List the current user's upcoming/due review schedules | Current user |
| POST | `/api/v1/me/reviews/{problemId}/complete` | Record completion of the due review and update the existing schedule using the fixed interval policy | Current user |

The fixed review intervals and overdue behavior must be decided before the completion endpoint's behavior can be finalized.

## 9. Cross-cutting behavior to specify next

For every operation, the detailed API-contract artifact should define:

- Request path/query/body schema and required/optional fields.
- Response DTO and whether the response has a body.
- Validation rules and field-level error format.
- HTTP status codes, including not found, validation failure, unauthenticated, and forbidden cases.
- Authorization requirements and ownership checks.
- Idempotency and concurrency behavior where applicable.
- Pagination defaults and maximum page size for list endpoints.
- Standard error response, preferably aligned with Problem Details.
- OpenAPI operation ID, summary, tags, and security metadata.

ASP.NET Core supports generating OpenAPI documents from endpoint metadata. Configure the OpenAPI document and interactive UI deliberately for development and deployment environments.

## 10. Proposed HTTP status conventions

| Situation | Proposed status |
| --- | --- |
| Successful read/update | `200 OK` |
| Resource created | `201 Created` |
| Successful command with no response body | `204 No Content` |
| Invalid request/validation | `400 Bad Request` |
| Missing/invalid Firebase token | `401 Unauthorized` |
| Authenticated but not permitted | `403 Forbidden`, except private-roadmap concealment |
| Resource does not exist or private resource is not owned by caller | `404 Not Found` |
| Duplicate/conflicting state | `409 Conflict` where appropriate |
| Unexpected server failure | `500 Internal Server Error` with no sensitive internals |

Never leak whether another user's private roadmap exists through error details.

## 11. Open decisions before finalizing contracts

1. **Roadmap list:** Should archived roadmaps be available through a query parameter or a separate endpoint?
2. **Archived shared topic:** Can an existing roadmap link still have progress edited?
3. **Reorder contract:** Full ordered ID list or per-topic sort-order updates?
4. **Review policy:** Exact fixed intervals and behavior for overdue reviews.
5. **Problem browsing:** Exact filter fields, sort fields, and pagination defaults.
6. **Admin problem contract:** Required create/update fields and audit metadata for admin changes.
7. **Seed/admin conflict:** How should seed reconciliation behave if an admin has edited or soft-deleted a seeded problem?
8. **Account lifecycle:** Is account deletion in MVP scope, and what happens to roadmaps, practice records, and review schedules?
9. **Admin authorization source:** Which server-side configuration identifies admins (for example, Firebase UID allowlist or trusted custom claims), and how are changes audited?
10. **Admin topic operations:** Does the separate admin API also need endpoints to create/update/archive shared topics and manage prerequisites, or will those remain seed-only?
11. **Profile field policy:** Which profile fields may be updated by the user, and which are snapshots synchronized from Firebase?
12. **Email verification enforcement:** Define the exact response code and error contract when a token is valid but the email is not verified.
13. **Profile bootstrap concurrency:** Define safe behavior if multiple first-time `GET /me` requests arrive concurrently.

## 12. Definition of Done — API Endpoint Inventory

- [ ] Endpoint groups cover the approved MVP capabilities.
- [ ] Every endpoint has a method, path, purpose, and access rule.
- [ ] Firebase authentication is correctly separated from application API calls.
- [ ] Private roadmap ownership checks are explicit.
- [ ] Archive, restore, practice, and review operations match the database decisions.
- [ ] Unresolved decisions are listed instead of silently guessed.
- [ ] No controllers, DTO classes, or runtime endpoints are implemented in this documentation-only step.
