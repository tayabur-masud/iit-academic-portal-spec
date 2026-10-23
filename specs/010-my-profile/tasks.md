---

description: "Actionable tasks for My Profile"
---

# Tasks: My Profile

**Input**: Design documents from `/specs/010-my-profile/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: API and Angular tests are required by the plan and project constitution for this authenticated, privacy-sensitive feature. No database migration tests are needed because the schema is unchanged.

**Organization**: All implementation work belongs to the single P1 user story. The feature is read-only and reuses the existing authenticated account and role-profile records.

## Phase 1: Setup

**Purpose**: Use the existing application and test projects.

No project initialization, new package, or framework setup is required.

---

## Phase 2: Foundational

**Purpose**: Prepare shared infrastructure before the user story.

No additional foundational work is required. Session identity/active-role claims, secure-by-default authentication, profile stores, the authenticated Angular shell, and `no-store` API middleware already exist.

---

## Phase 3: User Story 1 - View My Profile (Priority: P1) - MVP

**Goal**: Let a signed-in user view their own account email and profile for the active Admin, Student, Teacher, or Coordinator role without editing or exposing internal or unrelated data.

**Independent Test**: Sign in as Student, Teacher, Admin, and Coordinator. Verify each view shows only the current account's active-role profile; Coordinator uses its Teacher profile. For Students, verify all personal details, guardian details, addresses, and prior education appear without internal IDs, audit/security data, admissions, or current results. Verify multiple-role switching, missing-profile states, anonymous/cross-account denial, loading/errors, accessibility, and absence of write actions.

### Tests for User Story 1

> Add these focused tests before implementation and verify the expected failures.

- [X] T001 [P] [US1] Add API integration tests for active-role mapping (Admin, Student, Teacher, Coordinator), account scoping from session claims, unsupported/missing-role denial, missing-profile response, multi-role behavior, anonymous and cross-account denial, Student personal/guardian/address/prior-education allowlisting, exclusion of IDs/audit/admissions/results/secrets, and authenticated `no-store` response in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Profiles/MyProfileTests.cs`.
- [X] T002 [P] [US1] Add My Profile component tests for all response variants, optional Student values, Coordinator-to-Teacher mapping, loading/unavailable/recoverable-error states, view-only rendering, and keyboard/small-screen semantics in `iit-academic-portal/src/app/features/profile/my-profile.spec.ts`.
- [X] T003 [P] [US1] Update shell and route tests to verify the account-menu My Profile action is enabled for every active role, opens the shared profile route, survives role changes by loading the new active-role view, and leaves existing role navigation intact in `iit-academic-portal/src/app/layout/shell/shell.spec.ts` and `iit-academic-portal/src/app/app.routes.spec.ts`.

### Implementation for User Story 1

- [X] T004 [US1] Define role-discriminated, read-only application view models in `iit-academic-portal-service/src/IitAcademicPortal.Application/MyProfile/MyProfileModels.cs` for the shared email/active-role context and Staff, Student, and Teacher profile variants; omit account/profile/history/catalog IDs, audit metadata, account status, admissions, current results, credentials, and setup proofs.
- [X] T005 [P] [US1] Add a Student self-view query to `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStudentStore.cs` and implement it in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/StudentStore.cs`; return the Student profile's personal/contact/address/guardian fields, resolved locality names, and prior academic history ordered as entered, without admission summaries or internal IDs. Preserve the data-model rule: "Optional values remain absent/null when not supplied; do not fabricate values."
- [X] T006 [P] [US1] Add account-scoped profile detail reads to `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStaffStore.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/StaffStore.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/ITeacherStore.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/TeacherStore.cs`; return only the signed-in account's role profile and current Designation name where assigned.
- [X] T007 [US1] Implement the role-aware `MyProfileService` in `iit-academic-portal-service/src/IitAcademicPortal.Application/MyProfile/MyProfileService.cs` and register it in `iit-academic-portal-service/src/IitAcademicPortal.Application/DependencyInjection.cs`; derive profile selection from the supplied validated user ID and active role, map Admin to Staff, Student to Student, Teacher/Coordinator to Teacher, and return an unavailable profile without falling back to another role.
- [X] T008 [US1] Add the allowlisted role-discriminated response DTO in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/MyProfileContracts.cs` and authenticated `GET /api/profile` endpoint in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Profile/MyProfileController.cs`; obtain user ID and active role only from `PortalClaims`, accept no target identity/role selector, and map missing profile and safe errors as specified in `specs/010-my-profile/contracts/my-profile-api.md`.
- [X] T009 [US1] Define typed role-discriminated response models in `iit-academic-portal/src/app/features/profile/my-profile.models.ts` and add the read-only `/api/profile` request in `iit-academic-portal/src/app/features/profile/my-profile-api.ts`; do not send an account ID or role selector.
- [X] T010 [US1] Build the lazy-loaded My Profile view in `iit-academic-portal/src/app/features/profile/my-profile.ts` and `iit-academic-portal/src/app/features/profile/my-profile.html`; render the active role's approved fields, including all Student profile personal/guardian/address/prior-education values, and provide accessible loading, unavailable, and recoverable error states with no edit or save controls.
- [X] T011 [US1] Add the shared authenticated `/profile` route in `iit-academic-portal/src/app/app.routes.ts` and enable the account-menu My Profile navigation in `iit-academic-portal/src/app/layout/shell/shell.html`; keep the route available under all supported roles, close the menu after navigation, and request the current active-role profile after role switching.
- [X] T012 [US1] Update `specs/010-my-profile/contracts/my-profile-api.md` and `specs/010-my-profile/quickstart.md` to match the implemented response fields, error states, and route behavior; preserve the Student exclusions and record that no database migration is required.

**Checkpoint**: Each supported role can view only its own active-role profile, including the approved Student personal details; the view remains read-only and does not expose internal, audit, credential, admission, or other-account data.

---

## Phase 4: Polish & Cross-Cutting Validation

**Purpose**: Run automated checks and the role-by-role walkthrough; record actual outcomes.

- [X] T013 Run the focused API tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Profiles/MyProfileTests.cs`, the Angular tests in `iit-academic-portal/src/app/features/profile/my-profile.spec.ts`, `iit-academic-portal/src/app/layout/shell/shell.spec.ts`, and `iit-academic-portal/src/app/app.routes.spec.ts`, then run the service test project and `npm test -- --watch=false` plus `npm run build` from `iit-academic-portal`; record actual results in `specs/010-my-profile/quickstart.md`.
- [X] T014 Walk through the end-to-end scenarios for Student, Teacher, Admin, and Coordinator accounts, including a multi-role role switch, a missing-profile account, Student sensitive-field exclusions, keyboard navigation, and recoverable request failure; record outcomes in `specs/010-my-profile/quickstart.md` without claiming unexecuted checks passed.

## Phase 5: Addendum - Account name in the menu (FR-009)

**Purpose**: Show the signed-in user's name, not "Account", on the account-menu trigger.

- [X] T015 [US1] Add `displayName` to the session response (sign-in, current session, role switch) in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/AuthContracts.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/SessionsController.cs`, resolved by `MyProfileService.GetDisplayNameAsync` from one `IUserAccountStore.GetNamesAsync` read (active role's profile name, then Staff, Teacher, Student, then email).
- [X] T016 [P] [US1] Add API tests for the name on sign-in, current session, and role switch, the fallbacks, and a profile added later in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Profiles/MyProfileTests.cs`.
- [X] T017 [US1] Show the name (shortened with an ellipsis, full name as a tooltip) and initials on the account-menu trigger in `iit-academic-portal/src/app/layout/shell/`, read through `AuthSession.displayName`; fall back to "Account" and the role letter if no name is sent.
- [X] T018 [P] [US1] Add shell tests for the name, initials, tooltip, fallback, and role switch, and `initialsOf` unit tests, in `iit-academic-portal/src/app/layout/shell/shell.spec.ts`; update the 001 authentication contract, this feature's contract and quickstart, design-system Section 9, and both READMEs.

- [X] T019 [US1] Swap the account menu and the theme choice: the menu moves to the right of the header (opening downward; avatar only on phones, with the greeting in the open menu) and reads "Welcome, <name>"; the theme choice becomes a labelled Light / Dark / System switch at the bottom of the sidebar, in `iit-academic-portal/src/app/layout/shell/` and `iit-academic-portal/src/styles.css`. Update the shell tests, design-system Sections 5.1, 9, and 18, and the frontend README.

- [X] T020 [US1] Show the active role under the greeting only for multi-role accounts (single-role: screen-reader text only) in `iit-academic-portal/src/app/layout/shell/shell.html`, with shell tests for both cases; update design-system Section 9, the spec, and the frontend README.

- [X] T021 [US1] Remove the role badge (and its divider) from the account menu of a single-role account, which now holds only My Profile and Sign out; multi-role accounts keep the "Active role" choice. Update the shell test, design-system Section 9, and the spec.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Existing applications and test projects satisfy setup; no new packages or project structure are required.
- **Foundational (Phase 2)**: Existing session claims, active-role policies, stores, API cache middleware, and shell satisfy shared prerequisites.
- **User Story 1 (Phase 3)**: The sole P1 story depends on the existing infrastructure and approved role/profile contract; no other story blocks it.
- **Polish (Phase 4)**: T013 follows implementation and T014 follows the automated checks and runnable environment setup.

### User Story Dependencies

- **User Story 1 (P1)**: Independent and MVP; delivers the complete read-only profile flow for every supported active role.

### Within User Story 1

- T001-T003 are separate test files and can be authored in parallel before implementation.
- T004 defines shared application projections before T005-T007.
- T005 and T006 use separate store interfaces/implementations and can proceed in parallel after T004.
- T007 depends on the role-specific store methods; T008 follows the service contract.
- T009 can proceed after the response contract in T008 is agreed; T010 follows T009; T011 wires the route/menu after the page exists.
- T012 follows contract stabilization; T013-T014 follow all feature work.

## Parallel Opportunities

- **Tests**: T001 API, T002 profile component, and T003 shell/routes tests can proceed in parallel because they use separate test files.
- **Persistence reads**: T005 Student projection and T006 Staff/Teacher lookups can proceed in parallel after T004 establishes the application read models.
- **Frontend and backend**: Once T008 fixes the response shape, T009-T010 frontend API/view work can proceed alongside backend store/service implementation, with T011 after the view is available.

## Parallel Example: User Story 1

```text
Before implementation:
- Worker A: T001 API privacy and role tests
- Worker B: T002 My Profile component tests
- Worker C: T003 shared shell and route tests

After T004 defines the profile view models:
- Backend A: T005 Student self-view projection
- Backend B: T006 Staff/Teacher user-scoped reads

After T008 defines the API response:
- Frontend: T009, then T010, then T011
- Documentation: T012 after contract behavior is stable
```

## Implementation Strategy

### MVP First

1. Complete T001-T003 as focused regression tests.
2. Implement safe role-specific reads and the self-scoped endpoint in T004-T008.
3. Deliver the shared account-menu page for all roles in T009-T011.
4. Validate the US1 independent test before considering the feature complete.

### Incremental Delivery

1. Deliver the API projection for all roles with account-scoping and Student privacy tests.
2. Add the shared UI route and role-specific display, including loading, missing-profile, and error states.
3. Run T013, complete T014, and record actual results in the quickstart.

## Format Validation

All 14 task entries use `- [ ] T###`; story tasks carry `[US1]`, parallel markers are limited to independent test/store tasks, and every task description includes exact repository file paths. Setup, Foundational, and Polish tasks have no story label.

## Phase 5: Convergence

- [X] T015 Render every populated school/college, university, and institution value from each Student prior-education row instead of selecting only the first in `iit-academic-portal/src/app/features/profile/my-profile.ts` and `iit-academic-portal/src/app/features/profile/my-profile.html`; extend `iit-academic-portal/src/app/features/profile/my-profile.spec.ts` with a row containing multiple populated institution fields and verify all are visible, per FR-002 and US1/AC1 (partial).
- [ ] T016 Complete the remaining browser walkthrough for a Teacher with a profile, a Student with all profile details, and a Student with optional details absent; create the needed test profiles through the existing management screens, perform a screen-reader pass against FR-008, and record actual outcomes in `specs/010-my-profile/quickstart.md`, per US1/AC1, US1/AC9, and T014 (partial).
