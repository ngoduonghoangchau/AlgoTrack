# AlgoTrack documentation synchronization

This bundle updates documentation only. No application code, migrations, or tests were added.

## Files included
- README.md
- docs/api/api-contracts.md
- docs/api/api-endpoint-inventory.md
- docs/architecture/system-architecture.md
- docs/architecture/system-context.md
- docs/database/physical-database-design.md

## Confirmed decisions synchronized
- Practice stores only the current state per user-problem pair; `Solved` is the only MVP status and undo is unsupported.
- Review schedule scope is one user-problem pair and the schedule is created only when the problem becomes solved.
- Roadmap deletion means archive, not physical deletion, in the MVP.
- Roadmap-topic progress values are `NotStarted`, `InProgress`, and `Completed`.
- Inaccessible private roadmaps return `404 Not Found` to avoid disclosing existence.
- The database schema uses SQL Server `BIGINT`; API ID representation is explicitly left for final confirmation instead of incorrectly claiming UUIDs.
- `PracticeAttempt` references were removed from the planned data model; `PracticeRecord` is the current-state entity.

## Decisions intentionally still open
- Exact fixed review intervals, early/overdue review behavior, and repeated completion semantics.
- Behavior of roadmap-topic links when their shared topic is archived.
- Problem-topic association changes after practice exists.
- Account deletion/data retention, seed reconciliation, admin identity rotation, profile field ownership, and unverified-email error contract.
- Final wire representation for SQL Server BIGINT IDs and exact DTO field constraints.

## Apply safely
Review the changed files and copy them into the repository, or apply `AlgoTrack-documentation-sync.patch` from the repository root. Inspect `git diff --check` and `git diff` before committing.
