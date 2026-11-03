# AlgoTrack

**AlgoTrack** is a full-stack learning platform for organizing Data Structures and Algorithms (DSA) study, tracking topic progress, recording solved problems, and scheduling reviews.

> **Project status:** Planning / repository foundation. The architecture, database model, and MVP API contracts are being designed. Features listed below are planned unless explicitly marked as implemented.

## Goals

- Build a portfolio-quality full-stack application using a realistic engineering workflow.
- Practice DSA through a structured, personal learning roadmap.
- Track learning progress at roadmap-topic level.
- Store each user’s current `Solved` state per problem and review it on a fixed schedule; the MVP does not keep a full attempt history.
- Demonstrate clean architecture, API security, testing, CI, and professional Git practices.

## Planned technology stack

| Area | Technology |
| --- | --- |
| Frontend | React, TypeScript, Tailwind CSS |
| Backend | ASP.NET Core Web API, C# |
| Architecture | Clean Architecture, modular monolith, lightweight CQRS |
| Database | Microsoft SQL Server, Entity Framework Core |
| Authentication | Firebase Authentication (email/password and Google) |
| Testing | .NET unit and integration tests; frontend tests to be selected |
| CI | GitHub Actions |
| Version control | Git, GitHub, Conventional Commits |

The exact .NET target framework and supporting package versions will be selected before implementation and kept consistent across local development and CI.

## Planned MVP features

- **Authentication and profile:** Firebase sign-in, verified-email policy, and application-user provisioning.
- **Shared topic catalog:** browse shared topics used to construct learning roadmaps.
- **Private roadmaps:** create and manage personal roadmaps.
- **Topic progress:** track `NotStarted`, `InProgress`, and `Completed` for each topic within a roadmap.
- **Prerequisites:** define topic dependencies within a roadmap and reject self-dependencies or cycles.
- **Problem catalog:** browse problems categorized by difficulty and associated shared topics.
- **Practice state:** store the current `Solved` status once per user-problem pair; undoing `Solved` and full attempt history are out of scope for the MVP.
- **Review queue:** create at most one review schedule per user-problem pair, and only when the problem is marked `Solved`; exact fixed intervals remain to be decided.
- **Dashboard:** summarize roadmap progress and upcoming or overdue reviews.
- **Admin problem management:** seed/manage shared problems, soft-delete them, and restore them through server-authorized admin access.

## Architecture

AlgoTrack is planned as a modular monolith using Clean Architecture.

```text
React + TypeScript + Tailwind CSS
              |
              | HTTPS + Firebase ID token
              v
       AlgoTrack.Api
   Controllers / Middleware
              |
              v
    AlgoTrack.Application
 Commands / Queries / Handlers
              |
              v
       AlgoTrack.Domain
 Entities / Business Rules

AlgoTrack.Infrastructure
 EF Core / SQL Server / Firebase Admin SDK
              |
              +---- implements Application abstractions
              +---- persists data in SQL Server
```

### Backend projects

- `AlgoTrack.Domain` — core entities and business invariants; no dependency on infrastructure.
- `AlgoTrack.Application` — use cases, command/query handlers, DTOs, validation, and abstractions.
- `AlgoTrack.Infrastructure` — EF Core persistence, SQL Server integration, and Firebase token verification.
- `AlgoTrack.Api` — HTTP endpoints, middleware, authentication/authorization configuration, and dependency-injection composition root.

### Testing projects

- `AlgoTrack.Domain.Tests`
- `AlgoTrack.Application.Tests`
- `AlgoTrack.Api.IntegrationTests`

The solution and tests have not necessarily been created yet. This section describes the intended structure.

## Security principles

- The API verifies Firebase ID tokens on the server.
- The backend derives the current user from verified token claims; clients cannot choose another user's identity by sending a user ID.
- Roadmaps are private and must be checked against their owner on every protected operation.
- Admin permissions are provisioned on the server and are never trusted from client-submitted roles.
- Email verification is required for protected personal/workflow endpoints according to the API policy.
- Secrets, service-account keys, private connection strings, and production credentials must never be committed.
- CORS is configured for explicitly allowed frontend origins; CORS is not a substitute for authentication or authorization.

## Data model overview

The planned SQL Server model includes:

- `AppUser`
- `Roadmap`
- `Topic`
- `RoadmapTopic`
- `TopicPrerequisite`
- `Problem`
- `ProblemTopic`
- `PracticeRecord`
- `ReviewSchedule`

Key decisions:

- Topics are shared across users; learning progress belongs to `RoadmapTopic`.
- A roadmap has one owner and is private in the MVP.
- Prerequisites connect topics within the same roadmap and must not form cycles.
- `PracticeRecord` stores only the current state per `(UserId, ProblemId)`; no separate attempt history is kept in the MVP.
- Review schedules are unique by `(UserId, ProblemId)`, shared across that user's roadmaps, and are created only when the problem becomes `Solved`.
- Topics are archived rather than physically deleted.
- Problems use soft deletion and can be restored by an authorized administrator.

The physical schema and API contracts are documentation artifacts; field-level consistency and remaining open decisions must be reviewed before implementation. No migrations have been created yet.

## Repository layout

The planned repository layout is:

```text
AlgoTrack/
├── src/
│   ├── AlgoTrack.Api/
│   ├── AlgoTrack.Application/
│   ├── AlgoTrack.Domain/
│   └── AlgoTrack.Infrastructure/
├── tests/
│   ├── AlgoTrack.Domain.Tests/
│   ├── AlgoTrack.Application.Tests/
│   └── AlgoTrack.Api.IntegrationTests/
├── docs/
│   ├── architecture/
│   ├── decisions/
│   └── api/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   └── PULL_REQUEST_TEMPLATE.md
├── .editorconfig
├── .gitignore
├── CONTRIBUTING.md
└── README.md
```

Folders and projects will be added as implementation progresses.

## Getting started

**Prerequisites and commands will be finalized when the solution and frontend apps are created.** The intended local setup will require:

- Git
- Node.js LTS and a package manager for the frontend
- The selected .NET SDK
- SQL Server (local instance or container)
- A Firebase project configured for email/password and Google sign-in

Do not place Firebase service-account credentials or production secrets in the repository. Use local secret management or environment variables and commit only safe example configuration.

## Development workflow

1. Pick or create a GitHub Issue with clear acceptance criteria.
2. Create a short-lived branch from `main`, for example `feat/12-roadmap-creation`.
3. Make one coherent change and add/update tests.
4. Review the staged diff before committing.
5. Use Conventional Commit messages, such as:
   - `docs(readme): document project goals`
   - `feat(roadmaps): add roadmap creation`
   - `test(reviews): cover fixed interval policy`
   - `ci(github): add dotnet validation workflow`
6. Push the branch and open a Pull Request.
7. Review the diff and wait for required checks before merging.
8. Keep `main` in a working, reviewable state.

See `CONTRIBUTING.md` for the repository's detailed contribution rules once that file is completed.

## Roadmap

- [ ] Finalize project charter, MVP requirements, and decision log
- [x] Document physical database design and API contracts (remaining open decisions are tracked in those documents)
- [ ] Establish repository standards and branch protection
- [ ] Create the Clean Architecture .NET solution and test projects
- [ ] Implement Firebase token verification and `GET /api/v1/me`
- [ ] Implement SQL Server persistence and migrations
- [ ] Implement roadmap and topic workflows
- [ ] Implement problem catalog and practice history
- [ ] Implement review scheduling and dashboard
- [ ] Build the React frontend
- [ ] Add CI, security checks, and deployment documentation
- [ ] Publish a deployed demo and screenshots

## License

No license has been selected yet. Until a license is added, do not assume that the repository is available for reuse, redistribution, or modification by others.
