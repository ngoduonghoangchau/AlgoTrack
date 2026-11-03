# AlgoTrack Decision Log

**Purpose:** Record approved product and technical decisions. Accepted decisions are binding unless superseded by a later entry.

## ADR-001 — Review Scheduling Policy

**Status:** Accepted

- Review intervals are `1, 3, 7, 14, 30, 60` days.
- After the sixth interval, subsequent intervals remain 60 days.
- Create one review schedule for a user-problem pair only when the problem becomes `Solved`.
- Early completion is allowed.
- Calculate the next review date from the actual review-completion timestamp, not the previous due date.
- Overdue reviews are not penalized. Completing an overdue review advances the schedule from the actual completion timestamp.
- A completion request must not advance the same review occurrence more than once. The API must define a concurrency-safe/idempotent mechanism before implementation.
- The MVP stores current practice state only; it does not store a full practice-attempt or review-event history.

## ADR-002 — Archived Shared Topics

**Status:** Accepted

- Topics are shared catalog records and are archived rather than physically deleted.
- Archiving a topic does not archive or delete existing roadmap-topic links.
- Existing links and progress remain intact and are shown with an archived-topic indicator.
- An archived topic cannot be newly added to a roadmap.
- A user may independently archive an existing roadmap-topic link.
- Restoring a topic does not automatically restore links previously archived by roadmap owners.

## ADR-003 — Seed and Admin Ownership

**Status:** Accepted

- Seed inserts missing catalog records only; it must not overwrite fields maintained by an administrator.
- Seed must not automatically restore a soft-deleted problem.
- Seed-managed problems use a stable, unique `SeedKey`. `SeedKey` may be nullable for admin-created problems that do not originate from seed data.
- A seed run encountering a conflicting key or invalid data must report the conflict rather than silently overwrite data.
- Server-authorized admin APIs are the authoritative mechanism for catalog edits, problem soft deletion, and restoration.
- Use database uniqueness constraints and a controlled seeding process to make repeated seed runs safe.

## ADR-004 — Account Deletion

**Status:** Accepted — Out of MVP scope

- Self-service account deletion is excluded from the MVP; no public self-service deletion endpoint is defined.
- A documented operational process must handle verified account-deletion requests.
- Firebase identity deletion and AlgoTrack application-data deletion are separate operations.
- The process must track partial failures and support safe retries; success in one system must not be treated as proof of completion in the other.
- A future self-service workflow requires a separate lifecycle, retention, and failure-recovery design.

## ADR-005 — Public API Identifier Representation

**Status:** Accepted

- SQL Server identifiers use `BIGINT` where defined by the physical schema.
- C# persistence and application code use `long`.
- Public JSON request and response contracts represent these identifiers as decimal strings.
- Route identifiers are validated and parsed to `long` by the API.
- React/TypeScript clients keep IDs as strings and must not convert them to JavaScript `Number`.

## Previously Approved Decisions

- Authentication uses Firebase Authentication with email/password and Google providers.
- Email verification is required for protected application operations.
- `GET /api/v1/me` lazily provisions the application profile using the verified Firebase UID.
- Private roadmaps are owner-only; return `404 Not Found` both for missing roadmaps and roadmaps not owned by the caller.
- Practice state is current-state only, with `Solved` as the sole MVP status; undoing `Solved` is unsupported.
- Roadmap deletion means archive, not physical deletion.
- Roadmap-topic progress is `NotStarted`, `InProgress`, or `Completed`.
- A problem is soft-deleted and may be restored through a server-authorized admin API.
- Review schedule scope is one user-problem pair, with at most one schedule per pair.
