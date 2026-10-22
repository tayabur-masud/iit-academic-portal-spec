# Quickstart: My Profile Validation

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Data model**: [data-model.md](data-model.md)  
**API contract**: [my-profile-api.md](contracts/my-profile-api.md)

## Prerequisites

- Node.js 24 LTS and .NET 10 SDK configured according to the project setup documentation.
- The existing API and Angular applications configured with a development database.
- Test accounts with active Student, Teacher, Admin, and Coordinator roles. The Coordinator test account must have its linked TeacherProfile.
- In the development database used here, `admin@iit.test` has a Staff profile and the other seeded accounts have none, which covers the missing-profile states; add Teacher and Student profiles through Teacher and Student Management to walk through scenarios 2 to 4.
- Each role test account should have a profile populated with representative optional values; include a Student with guardian/address/prior-education details and another Student with optional details absent.

## Automated Checks

From the workspace root:

```powershell
Push-Location iit-academic-portal-service
dotnet test tests/IitAcademicPortal.Api.Tests/IitAcademicPortal.Api.Tests.csproj --filter FullyQualifiedName~MyProfileTests
Pop-Location

Push-Location iit-academic-portal
npm test -- --watch=false
npm run build
Pop-Location
```

## End-to-End Scenarios

1. Sign in as an Admin and open My Profile from the account menu. Verify the page shows the signed-in account email, own Staff name/mobile, and current Designation name only.
2. Sign in as a Teacher. Verify the page shows the account email, own display name, optional staff identifier/phone, and Designation when present.
3. Sign in as a Student with all fields populated. Verify all personal profile fields, guardian details, resolved mailing/permanent address names, and prior academic history are shown. Confirm no account/profile/history/catalog IDs, account status, audit metadata, admission summaries, or current academic results appear.
4. Sign in as a Student whose optional details are absent. Verify missing values are omitted or clearly unavailable and no placeholder data is fabricated.
5. Sign in as a Coordinator with a linked TeacherProfile. Verify the Teacher profile is used and no separate Coordinator profile or assignment list appears.
6. Use an account assigned multiple roles. Switch its active role and reopen My Profile. Verify the displayed profile variant changes with the active role while account email remains the same.
7. Sign in with a role account lacking its corresponding profile. Verify the safe unavailable state does not fall back to another role's profile.
8. Request `/api/profile` anonymously and verify `401`. Attempt to request another account by adding an identifier or role selector; verify the API accepts no such input and cannot disclose that account.
9. Verify the page has no edit, save, deactivation, credential, or role-change action. Confirm profile updates remain available only from the role's existing management workflow.
10. Exercise keyboard navigation, a narrow viewport, loading, and recoverable API failure states; verify all content remains readable and no stale profile is presented after failure.

## Database Migration

No migration is expected. The feature reads existing account, role-profile, and Designation data and adds no persisted fields or relationships.

## Verification Record

Run on 2026-10-23 on Windows 11 with .NET 10, Node.js 24, and the development PostgreSQL database.

### Automated checks

| Command | Result |
|---|---|
| `dotnet test tests/IitAcademicPortal.Api.Tests --filter FullyQualifiedName~MyProfileTests` | 12 passed, 0 failed |
| `dotnet test tests/IitAcademicPortal.Api.Tests` (whole service suite) | 1,134 passed, 0 failed, 99 skipped (the PostgreSQL-only tests, which need `AUDIT_TEST_POSTGRES`) |
| `npm test -- --watch=false` (includes `my-profile.spec.ts`, `shell.spec.ts`, `app.routes.spec.ts`, `auth.spec.ts`) | 32 files, 378 tests passed |
| `npm run build` | Succeeds; initial bundle 446.96 kB (500 kB warning budget), `my-profile` lazy chunk 10.57 kB |

The API tests cover all four roles, Coordinator to Teacher mapping, a multi-role account switching between Staff and Teacher profiles, missing profiles, anonymous (401) and roleless (403) requests, ignored identifiers in the query, 405 for every write method, the exact Staff, Teacher, Student, address, and history field sets, absent optional Student values, current Designation names, `no-store`, and the absence of IDs, admissions, registration numbers, rolls, and audit or account fields.

To build the service while the API is running, stop it first: the running process locks the build output.

### Browser walkthrough

Headless Chrome against the running API and Angular dev server, signed in with the seeded development accounts, at 1280px (light and dark) and 375px.

| Scenario | Outcome |
|---|---|
| 1. Admin (`admin@iit.test`) | Passed: account email, active role, and the Staff profile (name, mobile number, Designation); no other fields. Reached from the account menu in two actions; the menu closes. |
| 2. Teacher with a profile | Not run in the browser: the seeded `teacher@iit.test` has no Teacher profile. Covered by the API and component tests. |
| 3. Student with every detail | Not run in the browser: the seeded `student.personal@example.test` has no Student profile. Covered by the API and component tests. |
| 4. Student with optional details absent | Not run in the browser (same reason). Covered by the API and component tests. |
| 5. Coordinator | Passed for `coordinator@iit.test`: the Teacher-profile view with the Coordinator note; this account has no Teacher profile, so the "No teacher profile yet" state shows. No Coordinator profile is implied. |
| 6. Multi-role switch | Passed for `teacher.coordinator@iit.test`: switching Teacher to Coordinator on My Profile stays on `/profile` and reloads for the new role with the same email. A switch between two roles that both have profiles was covered by the API and component tests only. |
| 7. Missing profile | Passed for the Teacher, Coordinator, and Student seeded accounts: "No … profile yet", and no other role's details. |
| 8. Anonymous and cross-account | Passed: an anonymous `GET /api/profile` returned 401 from the running API. Cross-account and role-selector attempts were covered by the API tests. |
| 9. No write action | Passed: no form, input, or save, edit, or deactivation control on any view. |
| 10. Keyboard, narrow screen, failure | Passed: Enter opens the account menu, two Tabs reach My Profile (role choice, then the link) with a visible focus outline, and Enter opens it; no horizontal scrolling at 375px; a refused request shows "Profile not shown" with no profile details, and Try again loads the profile. No console errors on any view. |

### Convergence Follow-up (2026-10-23)

| Check | Result |
|---|---|
| Focused `my-profile.spec.ts` after adding the multi-institution case | 11 passed, 0 failed |
| Full `npm test -- --watch=false` | 378 passed across 32 files |
| `npm run build` | Succeeded; initial bundle 446.96 kB and My Profile lazy chunk 10.59 kB |
| Remaining T016 browser and screen-reader walkthrough | Not completed: the browser is unauthenticated, and the seeded Teacher/Student accounts have no profiles. No credentials were supplied for signing in or creating the test profiles. A screen-reader pass also remains outstanding. |

Still to check by hand: scenarios 2 to 4 with accounts that have Teacher and Student profiles, and a screen reader pass.
