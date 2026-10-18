# notebook Refined Specification

## Scope

Build a TypeScript monorepo for personal notes. Use Fastify 5, React 18, Vite, PostgreSQL 16, `pg.Pool`, bcryptjs, JWT, and WebSockets.

Do not use SQLite, MySQL, production in-memory stores, or the legacy `docker-compose` command. Use the Docker CLI command `docker compose`.

Use strict TypeScript in `backend/` and `frontend/`. Keep every TypeScript source file below 500 lines.

## Public Health Contract

Expose both paths as compatibility aliases:

- `GET /healthz`
- `GET /api/v1/health`

Return status `200` and JSON `{"status":"ok"}`. Return a JSON content type. Do not require authentication.

## Authentication Contract

### POST /api/v1/auth/register

Accept this JSON body:

```json
{"email":"user@example.com","password":"secret123"}
```

Normalize the email with trim and lowercase. Require a valid email and a password with at least eight characters. Create one user with a bcrypt hash.

Return status `201`:

```json
{"token":"<jwt>","user":{"id":1,"email":"user@example.com","created_at":"2026-08-03T12:00:00.000Z"}}
```

Return status `400` for invalid JSON or fields. Return status `409` for a duplicate email. Return status `500` for an unexpected persistence failure. Return JSON errors with this shape:

```json
{"error":"<stable error code>","message":"<human-readable message>"}
```

Use error codes `INVALID_REQUEST`, `EMAIL_EXISTS`, and `INTERNAL_ERROR`.

### POST /api/v1/auth/login

Accept the same email and password fields. Return status `200` with the same token and user shape. Return status `401` with error code `INVALID_CREDENTIALS` when the user does not exist or the password does not match. Do not reveal which credential failed.

### GET /api/v1/auth/me

Require `Authorization: Bearer <jwt>`. Return status `200` with the user object. Return status `401` with error code `UNAUTHORIZED` when the header is missing, malformed, expired, or invalid.

Sign tokens with `JWT_SECRET`. Include the user identifier in the token subject. Reject expired tokens.

## Notes Contract

The source specification references seven notes operations but does not provide the referenced command matrix. This refined contract defines the five unambiguous operations below. Do not invent additional public operations until the missing matrix receives product approval.

### GET /api/v1/notes

Require authentication. Return status `200` and a JSON array of notes owned by the user. Sort by `updated_at DESC`.

Accept an optional `query` query parameter. Trim it and perform a case-insensitive search across `title` and `content`. An empty query returns all notes.

### GET /api/v1/notes/:id

Require authentication. Return status `200` and one owned note. Return status `404` with error code `NOT_FOUND` when the note does not exist or belongs to another user.

### POST /api/v1/notes

Require authentication. Accept:

```json
{"title":"Project Meeting Notes","content":"Discussed architecture."}
```

Require a string title after trimming. Permit an omitted content field and use an empty string. Return status `201` with the created note. Return status `400` with error code `INVALID_REQUEST` for invalid fields.

### PUT /api/v1/notes/:id

Require authentication. Accept one or both fields `title` and `content`. Require at least one field. Update `updated_at` with PostgreSQL `now()`. Return status `200` with the updated note. Return status `400` for invalid input and status `404` for an absent or foreign note.

### DELETE /api/v1/notes/:id

Require authentication. Delete only an owned note. Return status `204` with an empty body. Return status `404` for an absent or foreign note.

## Data Representation

Use PostgreSQL 16 only. Create these tables with migrations:

- `users(id BIGSERIAL PRIMARY KEY, email TEXT UNIQUE NOT NULL, password_hash TEXT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now())`
- `notes(id BIGSERIAL PRIMARY KEY, user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE, title TEXT NOT NULL, content TEXT NOT NULL DEFAULT '', created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now())`
- `schema_migrations(version TEXT PRIMARY KEY, applied_at TIMESTAMPTZ NOT NULL DEFAULT now())`

Create index `idx_notes_user_updated` on `(user_id, updated_at DESC)`. Apply each migration once at startup.

Convert every PostgreSQL BIGINT identifier to a JavaScript number. Return identifiers as JSON numbers. Return timestamps as UTC ISO-8601 strings with millisecond precision.

## WebSocket Contract

Expose `GET /ws?token=<jwt>` through the WebSocket upgrade path. Verify the token before accepting application messages. Close missing, invalid, or expired tokens with code `4001`, or reject the upgrade with HTTP `401`.

Broadcast events to every active connection for the changed user's identifier:

```json
{"type":"NOTE_CREATED","payload":{"id":12,"user_id":1,"title":"New Idea","content":"Draft content...","created_at":"2026-08-03T12:15:00.000Z","updated_at":"2026-08-03T12:15:00.000Z"}}
```

```json
{"type":"NOTE_UPDATED","payload":{"id":12,"user_id":1,"title":"Updated Idea Title","content":"Revised draft content...","created_at":"2026-08-03T12:15:00.000Z","updated_at":"2026-08-03T12:20:00.000Z"}}
```

```json
{"type":"NOTE_DELETED","payload":{"id":12}}
```

Do not broadcast events between different users. Broadcast only after the database mutation succeeds.

## Frontend Contract

Create a React SPA with these observable views:

- Unauthenticated login and registration form.
- Authenticated header with logo, email, WebSocket status, and logout.
- Sidebar with new-note action, live search, note titles, snippets, relative timestamps, and active selection.
- Editor with title input, content textarea, save action, delete action, and timestamps.

Store the JWT in `localStorage`. Connect to WebSocket after login. Reconnect with exponential backoff. Apply create, update, and delete events without page refresh. Clear the editor after deletion.

Use `VITE_API_URL`, default `http://localhost:3000`, and `VITE_WS_URL`, default `ws://localhost:3000`.

## Required Files

Create the pinned modular layout under `backend/` and `frontend/`. Use separate route, repository, service, context, component, and test files. Keep entrypoints thin and delegate immediately to testable application factories.

The backend must include `src/index.ts`, `src/app.ts`, `src/db.ts`, `src/migrate.ts`, `src/types.ts`, separate authentication and notes repositories, separate route files, `src/services/auth-service.ts`, and `src/services/ws-hub.ts`.

The frontend must include separate authentication, notes context, WebSocket client, header, note list, note editor, and authentication form modules.

## Tooling and Commands

Create one root `Makefile` with these recipes:

- `install`: run `npm install` in both workspaces.
- `start`: start the backend and frontend.
- `build`: build both workspaces.
- `test`: run both test suites and return non-zero on any failure.
- `lint`: run ESLint and TypeScript checks in both workspaces.
- `format`: run Prettier and remain idempotent.
- `e2e`: run `docker compose up --build --exit-code-from e2e`.

Use PostgreSQL integration tests with a dedicated operating-system user. Use socket readiness polling. Set a two-second maximum timeout for every HTTP, WebSocket, and TCP test operation.

## Startup and Shutdown

Use `PORT`, default `3000`; `DATABASE_URL`, default `postgres://postgres:postgres@localhost:5432/notebook`; and `JWT_SECRET`, default `notebook-super-secret-jwt-key-change-in-production`.

Log `notebook: <reason>` to stderr and exit `1` for invalid ports, database failures, and migration failures. Exit `0` after SIGINT or SIGTERM.

## E2E Acceptance

The Docker Compose E2E suite must verify health, registration, login, authenticated note creation, WebSocket authentication, note update broadcast, note deletion, and frontend HTML containing `<div id="root">`.

Test every public operation with a dedicated black-box scenario. Assert response status, JSON shape, numeric identifiers, timestamp format, error code, and body rules.

## Engineering Constraints

Use dependency injection for repositories, pools, configuration, and the WebSocket hub. Use in-memory doubles for intermediate unit tests. Use real PostgreSQL and Docker services only in integration and final E2E acceptance tests.

Do not use placeholders, empty handlers, `pass`, ellipsis bodies, `NotImplementedError`, tautological assertions, shell error masking, or skipped tests. Ask whether a script can generate structured SQL, JSON, YAML, or test fixtures more reliably than manual editing before authoring them.

Every implementation task must include its co-located tests. Final acceptance requires zero TypeScript errors, zero lint findings, and a 100 percent test pass rate.