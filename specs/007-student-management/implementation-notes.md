# Implementation Notes: Student Management

**Feature**: [spec.md](spec.md) | **Tasks**: [tasks.md](tasks.md) | **Validation**: [quickstart.md](quickstart.md)

Decisions the plan left open, and where the code goes beyond a first reading of the design.

## Data

- **Tables.** `StudentProfiles` (unique `UserId`, restrictive links to the account and to the mailing and permanent Upazila),
  `StudentAdmissions` (unique `(StudentProfileId, DegreeId, BatchId)`, unique non-null `NormalizedRegistrationNumber`, restrictive links
  to profile, Degree, Batch, and Hall), and `StudentAcademicHistory` (owned by its profile; every source value is text, no catalog keys).
  All three derive from `AuditedEntity`. Migration: `StudentManagement`.
- **Required fields.** The spec names email, mobile, Degree, and Batch. The implementation also requires the applicant name (a record
  without one cannot be found or shown) and the admission test serial number and exam roll (FR-002 says each admission "MUST retain"
  them, while class roll is explicitly optional). Every other personal, guardian, and address field is optional, because paper forms
  are often incomplete.
- **Registration number** is trimmed for display and compared case- and space-insensitively (`NameNormalizer.NormalizeCode`), across all
  admissions, including those of deactivated accounts.
- **Catalog choices in history** (revision of FR-003 on 2026-10-19). School/college, university, institution, board, and group are chosen
  from Educational institutions, Education boards, and Education groups, filtered by type for each field (Schools and Colleges, Universities,
  any). Each row stores the reference (`SchoolOrCollegeId`, `UniversityId`, `InstitutionId`, `BoardId`, `GroupId`; migration
  `StudentHistoryCatalogReferences`) and a name snapshot set from the catalog when the record is newly chosen; a row that keeps the same record
  keeps its saved snapshot, so renames never rewrite history. An unlisted value is typed ("Not listed: type the name") and stored without a
  reference. Those catalog records cannot be deleted while history uses them.
- **Academic history** keeps a `SortOrder` so rows are shown as entered. An update sends the complete list: rows with an `id` are kept and
  updated, rows without one are added, and rows left out are removed. Only `Category` is controlled (six enum names).
- **Configuration records in use.** Deleting a Degree, Batch, Hall, or Upazila that an admission or address uses is refused like any
  other in-use record ("Deactivate it instead"), through the new `IConfigurationStore.UsedByStudentsAsync<T>`. A deactivated record an
  admission already has stays valid on an unrelated update; a newly chosen one must be active.

## Accounts and setup

- **One account rule.** The normalized email is looked up first. No account: a pending, passwordless Student account (Student role and
  default) is created with the profile and admission in one transaction, with `account.created`, `student.profile.created`, and
  `student.admission.created`, and the common `AccountCredentialSetup` email is requested after the commit. A Student-only account: it
  is reused untouched; a profile is attached if it has none; no setup email is requested. Any other account (one with Admin, Teacher, or
  Coordinator): 409, nothing written.
- **Returning students.** `GET /api/admin/students/lookup?email=` (an addition to the planning contract) tells the form whether the email is
  new, a returning Student (with the profile, history, and admissions to prefill), or another kind of account. Submitting an admission for
  a returning Student replaces the shared profile and history with the submitted values, exactly like an update, so the form prefills them
  first.
- **Email changes** stay in User Management; Student Management never edits the account.

## Audit

Business events `student.profile.created`, `student.profile.updated`, `student.admission.created`, `student.admission.updated`. Program and
institutional identifiers keep values (`degreeId`, `batchId`, `assignedHallId`, `registrationNumber`, serial, exam roll, class roll);
personal values (names, birth data, mobile and cell numbers, addresses, guardian, academic history) are recorded as "changed" only. A new
account also records `account.created`.

## Eligibility

`GraduationEligibilityService` reads the Degree's `CreditsToBeCompleted`, the admission's credits from `ICompletedCreditsProvider` (keyed by
admission ID), and the admission's Hall and registration number. Each requirement is reported as Met, NotMet, or Unavailable; eligible
only when all are Met. Nothing is stored. The registered provider is `UnavailableCompletedCreditsProvider`, because no academic-results
owner exists yet, so nobody is eligible in the running portal until that owner registers its adapter (task T022).

## Frontend

- `/admin/students` (directory, one row per admission), `/admin/students/new`, and `/admin/students/:admissionId` (the same form, plus the
  account panel, other programs, eligibility, and record history). All lazy-loaded; the initial bundle is 483.07 kB of 500 kB.
- Addresses use cascading Division, District, and Upazila selects backed by the Configuration API; only the Upazila and details are sent.
- History rows show the fields that apply to their category; values in other fields are kept, not shown.

## Other fixes made while implementing

- `HallSeedScriptTests` and `GeographySeedScriptTests` now `TRUNCATE ... CASCADE`: PostgreSQL refuses to truncate a table that the new
  student tables reference, even when they are empty.
- `ConfigurationPostgresTests.CodedNamed` now sets a Degree's credits; the degree-credits change on `main` had left that test failing
  against PostgreSQL.
- `teacher-form.spec.ts` expected the pending-account wording from before feature 006; 006 changed the wording after its last test run,
  so that spec had been failing on `main`. It now checks the current text and the link to the account.

## Not done

- **T022** (positive completed-credit integration): blocked on the academic-results owner.
- **T025** (timed usability study with representative Admins): not run.
- The migration was applied to the Development database on request (with the pending `DegreeCredits` migration); no shared or Production database was changed.
