# Research: Student Management

**Date**: 2026-10-19  
**Spec**: [spec.md](spec.md)

## Decisions

### Student identity and program admissions

- **Decision**: Model one StudentProfile per Student portal account and one StudentAdmissionRecord per Degree/Batch enrollment. The first admission creates the account, profile, and admission in one application-owned transaction and requests credential setup after commit. A returning Student's normalized email reuses the Student account/profile and creates a distinct admission; it does not change account status, credentials, or request another setup email.
- **Rationale**: The same person may complete PGDIT and later enroll in EMIT. A one-account-per-admission model would either reject the existing email or create duplicate login identities. `UserProvisioningService.CreateAsync` commits an account-only transaction, so it cannot be called separately from admission creation. `TeacherProvisioningService` demonstrates creating a pending account and linked profile atomically, then requesting setup after commit.
- **Alternatives considered**: Requiring a new email/account for every program was rejected because it duplicates identity and credential records. Automatically merging an account with a non-Student role was rejected because it risks cross-role account takeover. Reusing a generic account-only transaction was rejected because it could leave partial data.

### Account identity and contact data

- **Decision**: Keep email, username, role, and account status on the linked `PortalUser`; link the one-to-one StudentProfile to that account, then link multiple admissions to the profile. Store applicant mobile and person-level fields on StudentProfile, without treating mobile as verified.
- **Rationale**: The existing Identity account is the canonical email and sign-in identity. A StudentProfile supplies the stable person identity shared by program enrollments; it avoids copying mobile and personal details into every admission. The specification explicitly separates applicant mobile from address and guardian phone numbers and does not require mobile verification.
- **Alternatives considered**: A second profile-owned email was rejected as duplicate identity data; phone verification or SMS was rejected as out of scope.

### Structured mailing and permanent addresses

- **Decision**: Store mailing and permanent address details separately on StudentProfile. Each address has one free-text details value and one Upazila reference; derive its District and Division from the existing geography hierarchy.
- **Rationale**: The existing Division → District → Upazila Configuration records provide canonical geographic identities. Storing only the selected Upazila ID avoids inconsistent parent IDs, while the existing cascading selector pattern lets the Admin choose Division, then District, then Upazila. The free-text value retains house, road, or locality details not modeled by the national hierarchy.
- **Alternatives considered**: Free-texting all geographic names was rejected because it prevents validated cascading selection. Storing separate editable Division and District IDs was rejected because they could disagree with the selected Upazila.

### Degree, Batch, Hall, and registration relationships

- **Decision**: Reference existing Degree and Batch records, require their relationship to agree, and reference the existing Hall catalog when a Hall is assigned. Enforce registration-number uniqueness across student admissions in application validation and PostgreSQL.
- **Rationale**: Configuration already owns these catalogs. Reusing their stable identifiers prevents copied labels and keeps student admission data aligned with Admin-managed offerings. The later Hall and registration updates remain independent.
- **Alternatives considered**: Copying Degree, Batch, or Hall names into student rows was rejected because it duplicates configuration state. Requiring Hall or registration number at initial admission was rejected by the specification.

### Profile-level prior academic history

- **Decision**: Store zero or more `StudentAcademicHistory` entries as children of StudentProfile, shared across the person's program admissions. `Category` is a fixed six-value enum; every other applicable source-form value (examination label, institution, board, group, department, university, subject, year, marks, percentage, CGPA, class, and division) is an optional text snapshot, not a Configuration foreign key.
- **Rationale**: Prior qualifications describe the person rather than one enrollment. The source forms vary by education category and leave some values blank. Snapshot strings preserve the entered values even when they are absent from or later differ from mutable catalogs, and avoid requiring fields the source form omits.
- **Alternatives considered**: Mutable Configuration references were rejected for historical values because they could change or prevent entry of unlisted source-form values. One wide mandatory row was rejected because it would require irrelevant fields. Scans or photos were rejected by the specification.

### Completed-credit source and eligibility

- **Decision**: Keep eligibility computed and read-only. Read the required credit total from the selected Degree and the completed-credit total through an application-level read boundary to the academic-results owner. If the source is absent or unavailable, return ineligible with the unavailable credit information identified. Do not persist a copied completed-credit total or course results in Student Management.
- **Rationale**: The specification assigns degree requirements to Configuration and passed-course results to the academic-results owner, while explicitly defining an unavailable-source outcome. Repository inspection found no CourseResult or passed-course credit provider in the current service, so a concrete adapter cannot be selected in this phase.
- **Alternatives considered**: A Student Management `CompletedCredits` field or duplicate course/result tables were rejected by FR-012. Treating missing source data as zero or eligible was rejected by FR-006.

### Returning Student and previous programs

- **Decision**: Keep every IIT program enrollment as its own StudentAdmissionRecord under the StudentProfile. A previously recorded PGDIT admission remains visible as a separate prior program and is not duplicated as StudentAcademicHistory. Completed credits are keyed to the current admission and do not carry from PGDIT into EMIT without a separately approved transfer-credit rule.
- **Rationale**: Degree, Batch, Hall, registration number, and graduation eligibility are program-specific. Distinct admissions preserve the old program's identity and outcomes while allowing the same account to enter another program.
- **Alternatives considered**: Overwriting the PGDIT admission with EMIT, duplicating the PGDIT admission as a `StudentAcademicHistory` row, or copying PGDIT credits into EMIT was rejected because it loses program history, duplicates history, or invents a credit-transfer rule.

### API, authorization, and UI conventions

- **Decision**: Use resource-oriented Admin student routes, explicit request/response DTOs, active-Admin authorization on the server, and Angular forms/lists matching existing User and Teacher Management features. Use the existing audit and notification services and bounded pagination for the directory.
- **Rationale**: This follows the repository's REST, authorization, validation, and UI conventions and keeps the feature in the current layered monolith.
- **Alternatives considered**: A separate service, a new identity flow, a custom audit system, or new UI framework were rejected because equivalent project services and patterns already exist.

## Repository Findings

- `UserProvisioningService` creates generic Admin or Student accounts in `PendingCredentialSetup`, sets no password, records account creation, commits, then requests the shared setup email.
- `AccountCredentialSetupService` activates pending accounts only after a valid, single-purpose setup proof and password; Admin reissue is already supported.
- `TeacherProvisioningService` creates a linked account and profile in one transaction and requests setup notification after commit; this is the closest atomic provisioning pattern.
- Degree, Batch, and Hall are existing Configuration records. Degree owns the required-credit total; Batch already refers to Degree.
- Configuration entities use audited base classes and migrations are generated through the existing EF Core setup.
- No Student profile/admission entity, academic-history entity, CourseResult entity, or completed-credit provider was found in the current source tree.
- Existing frontend features under `src/app/features/users` and `src/app/features/teachers` provide the list/form and typed API patterns to follow.

## Open Integration Dependency

The academic-results owner must expose a read-only source for the completed-credit total keyed to the Student admission identifier. Until that source is available and bound, eligibility must report the credit source as unavailable and remain ineligible. This does not authorize storing a manually maintained total in the Student record.