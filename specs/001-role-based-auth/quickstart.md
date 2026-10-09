# Quickstart: Authentication and Role Access Validation

**Feature**: [Role-Based Authentication and Access Control](spec.md)
**Related artifacts**: [Data model](data-model.md), [authentication contract](contracts/authentication.openapi.json), [research decisions](research.md)

## Prerequisites

- Frontend and service source repositories checked out using Angular 21 with Node.js 24 LTS, and ASP.NET Core 10 Web API / C# 14 with ASP.NET Core Identity 10 and EF Core 10. Use a compatible Npgsql EF Core provider 10.x.
- PostgreSQL 18 development database configured with pgvector where required by the application, and the service's reviewed migrations applied, including `SessionIdleTimeout` for the persisted last-activity timestamp.
- Source lives in the `iit-academic-portal/` (Angular) and `iit-academic-portal-service/` (ASP.NET Core) submodules; each README has the full setup.
- A development email sink or test mailbox connected for password-recovery messages. In Development the service writes each message as an `.eml` file to `src/IitAcademicPortal.Api/.mail/`. Do not use real user credentials or email reset links in shared logs.
- Test accounts for Admin, Student, Teacher, and Coordinator, plus one account assigned Teacher and Coordinator. The Student test account uses a personal email address. In Development, set `DevelopmentSeed:Password` in user secrets and restart the service to create `admin@iit.test`, `student.personal@example.test`, `teacher@iit.test`, `coordinator@iit.test`, and `teacher.coordinator@iit.test`.
- Test records spanning one owned Student record, one non-owned Student record, assigned/unassigned Teacher courses, and assigned/unassigned Coordinator batches.
- HTTPS or a local secure-development equivalent for testing secure cookies and browser anti-forgery behavior.

## Build and Test

Confirm the local runtime versions first: Node.js MUST report major version 24 and the .NET SDK MUST report major version 10. These commands are verified; run them from the spec repository root:

```powershell
node --version
dotnet --version

# Service: apply migrations, run tests (xUnit), then start on HTTPS. Swagger UI opens at https://localhost:7286/swagger.
Push-Location iit-academic-portal-service
dotnet tool restore
dotnet ef database update -p src/IitAcademicPortal.Infrastructure -s src/IitAcademicPortal.Api
dotnet test
dotnet run --project src/IitAcademicPortal.Api --launch-profile https
Pop-Location

# Frontend (second terminal): run tests (Vitest), then serve on http://localhost:4200.
Push-Location iit-academic-portal
npm ci
npm test -- --watch=false
npm start
Pop-Location
```

Start the service with the `https` launch profile. The frontend dev server proxies `/api` to `https://localhost:7286`, and the plain `http` profile does not listen there.

For the API smoke checks below, use Swagger UI. It sends the session cookie automatically and adds the `X-CSRF-Token` header to every state-changing request.

## End-to-End Scenarios

1. **Single-role sign-in**: Sign in as a Student using the personal email recorded for the account. Confirm the Student experience opens, the active role is Student, and protected data is limited to that student's records.
2. **Multi-role default and switching**: Sign in as the Teacher+Coordinator account (seeded with default role Teacher). Confirm the portal opens the Teacher area directly without asking for a role, and the menu's role switcher offers only Teacher and Coordinator. Verify access only to assigned courses and assessment components. Switch to Coordinator and verify assigned-batch boundaries. Attempt an unassigned role and confirm denial.
3. **Direct unauthorized access**: While signed in as Student, request a Teacher-only or another student's protected record directly through the API. Confirm server-side denial and no protected data in the response. Repeat for unassigned courses/batches.
4. **Generic sign-in failures**: Submit a wrong password and an unknown email. Confirm the same public failure shape and that neither attempt creates a session.
5. **Password recovery enumeration**: Request a reset for a known email and an unknown email. Confirm the same HTTP status and public response body for both. For the known test mailbox, follow the one-use recovery proof.
6. **Replacement-password validation**: Try a password shorter than eight characters, one without a letter, and one without a number; each must be rejected without changing the credential. A password meeting all three requirements may complete reset.
7. **Session reset semantics**: Establish sessions for the same test account in two separate browsers. Complete reset in the first browser. Confirm only that session is revoked, the other session remains active as specified, and the reset proof cannot be reused.
8. **Three-hour inactivity timeout**: Establish a session and make an authenticated request within three hours; confirm the session and cookie lifetime are refreshed. Then leave a separate session idle for three hours; confirm the next protected request is rejected, the stale cookie is cleared, and only that session is revoked.
9. **Logout and browser navigation**: Sign out, then revisit a protected route and directly request a protected resource. Confirm both require authentication and cached protected information is not exposed.
10. **Loading, validation, and accessibility**: Submit each asynchronous form slowly or with a test delay. Confirm progress is announced, duplicate submission is prevented, field errors are associated with inputs, keyboard navigation works, and focus remains visible in supported responsive layouts.

## API Smoke Checks

Use a disposable test account and a local email sink. The endpoint paths and payload contracts are in `contracts/authentication.openapi.json`; session cookies and anti-forgery values are credentials and MUST NOT be committed or logged. At minimum, verify:

- `POST /api/auth/sessions` creates a session for valid email/password and returns a generic `401` for invalid credentials.
- `GET /api/auth/sessions/current` returns the active role and only assigned roles.
- `PUT /api/auth/sessions/current/active-role` rejects an unassigned role and changes context only for the current session.
- `DELETE /api/auth/sessions/current` revokes the current session, clears its cookie, and leaves other sessions unchanged.
- `POST /api/auth/password-reset-requests` returns the same `202` response for known and unknown email addresses.
- `POST /api/auth/password-resets` enforces the password rule, consumes valid proof once, and revokes only the session used for reset.

