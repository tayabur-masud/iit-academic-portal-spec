# Feature Specification: Student Management

**Feature Branch**: `feature/007-student-management`

**Created**: 2026-10-18

**Status**: Implemented, except the positive completed-credit integration (blocked on the academic-results owner, tasks.md T022) and the SC-002 usability study (not run, T025)

**Input**: User description: "Use one common admission form for all student types. An Admin receives the paper form and attachments, enters the information, and selects the degree and batch. A Student-role portal user is created when student information is recorded. Students must provide an email and mobile number to complete admission. Assigned hall and registration number are entered later; both must be present before a student can complete graduation."

## Clarifications

### Session 2026-10-18

- Q: Who should be allowed to perform graduation clearance once the student has the required credits, hall, and registration number? → A: Graduation eligibility is calculated automatically; there is no separate clearance action or user role. A student is eligible only when all requirements are met.
- Q: What should be the authoritative source for each degree’s required credits and a student’s completed credits? → A: The selected degree’s settings provide its required credit total, and passed course results provide the student’s completed credits. Student Management consumes these values and does not duplicate or own course/result records.

### Session 2026-10-19

- Q: How should a returning Student enroll in another IIT degree program? → A: Maintain one Student profile and Student portal account per person, with a separate admission record for each Degree/Batch enrollment. Reuse the existing Student account and do not resend credential setup. If the email belongs to a non-Student account, reject the admission without merging accounts.
- Q: How should an earlier IIT program, such as a completed PGDIT, appear when the same person enrolls in EMIT? → A: Keep the PGDIT as its own admission under the same Student profile; do not duplicate it as a prior-education entry. If an earlier program is not recorded in Student Management, it may be entered as academic history. Completed credits are specific to an admission and are not carried into another program unless a separate approved transfer-credit rule is introduced.
- Q: Where should prior academic-history entries belong when a Student has multiple program admissions? → A: Attach academic-history entries to the Student profile so the person's prior education is shared across all program admissions.
- Q: How should mailing and permanent addresses capture Bangladesh geographic locations? → A: Keep mailing and permanent addresses separate. Each address has one free-text details field and an Upazila selection; derive and display its District and Division from the existing geographic hierarchy.
- Q: Should prior-education field values reference mutable catalogs or preserve what appears on the paper form? → A: Use the six fixed categories as the controlled category value and retain every other applicable field as an optional text snapshot, not a Configuration reference, so original source-form values remain intact.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Enter a Student Admission Record (Priority: P1)

As an active Admin, I need to maintain one Student identity and create a separate admission for each degree and batch so returning students can enroll in another program without duplicate portal accounts.

**Why this priority**: Student-specific enrollment data is the foundation for later administrative updates and graduation checks.

**Independent Test**: As an active Admin, admit a new student and then admit that same person to a second degree using the same normalized email. Verify one Student profile/account has two separate program admissions, only the initial account creation requests credential setup, and each admission has its own Degree/Batch and eligibility. Verify missing contact details or an email used by a non-Student account creates no partial data, and a non-Admin cannot create or inspect records.

**Acceptance Scenarios**:

1. **Given** an active Admin has received a completed paper form and the applicant has supplied an email and mobile number, **When** the Admin enters the information and selects a degree and batch, **Then** one Student profile and one program admission are created with exactly one linked Student-role portal account in the same transaction.
2. **Given** an Admin records prior education for a Student, **When** the Student profile is saved, **Then** each supplied SSC/O-Level, HSC/A-Level, Bachelor/Graduation, Master, PGDIT, or Other entry is retained on the profile with the applicable school/college, board, group, department, university, year, marks, percentage, CGPA, class, division, and subject values shown on the source form and is available across that person's admissions.
3. **Given** a non-Admin or anonymous user requests student admission data, **When** the request is authorized, **Then** access is denied without disclosing personal information.
4. **Given** a paper form or supporting documents are received, **When** the Admin enters the record, **Then** the portal stores the form's typed data only; scanned attachments and document images are not uploaded or retained by this feature.
5. **Given** a new Student profile and account are committed with the first admission, **When** account setup is requested, **Then** the account remains pending without a password and uses the existing secure credential-setup email flow; delivery failure does not partially roll back or activate the account. A later admission for the same Student does not create another account or request another setup email.
6. **Given** the applicant email or mobile number is missing, or the normalized email belongs to a non-Student account, **When** the Admin attempts to complete an admission, **Then** the operation is rejected without creating a partial profile, admission, or account. If the email belongs to an existing Student profile, the new admission is linked to that profile and account.
7. **Given** an Admin creates a Student profile and first admission or records a later admission for an existing profile, **When** the changes are saved, **Then** the profile/admission changes are recorded in audit history and no password or setup proof is exposed to the Admin.
8. **Given** the linked Student account is pending, **When** the student successfully follows the existing credential-setup email and sets a valid password, **Then** the account becomes Active and can sign in with the Student role. An invalid, expired, or undelivered setup link MUST NOT activate the account.
9. **Given** a Student profile already has a recorded PGDIT admission, **When** an Admin records a later EMIT admission using the same normalized email and a different Degree/Batch, **Then** a distinct EMIT admission is created under the same profile/account, the PGDIT remains visible as a separate prior admission, no setup email is requested again, and PGDIT credits are not automatically carried into EMIT eligibility.
10. **Given** an Admin enters mailing and permanent addresses, **When** they provide the address details and select an Upazila for each, **Then** the correct District and Division are shown from the existing hierarchy and cannot disagree with the selected Upazila.

---

### User Story 2 - Complete Student Identifiers Later (Priority: P1)

As an active Admin, I need to add a student's assigned hall and registration number after initial admission entry so those values can be recorded when they become available.

**Why this priority**: Hall and registration information is not available at initial data entry but is required for graduation eligibility.

**Independent Test**: Create a student record with neither value, add either value independently, then add the other. Verify each update is authorized, audited, and retained without requiring the other value to be present yet.

**Acceptance Scenarios**:

1. **Given** a new student admission record is created, **When** it is first saved, **Then** assigned hall and registration number are unset and do not block the initial save.
2. **Given** either assigned hall or registration number is available later, **When** an active Admin records that value, **Then** it is saved without requiring the other value to be assigned in the same update.
3. **Given** a registration number is already assigned to another student record, **When** an Admin tries to reuse it, **Then** the update is rejected without changing either record.
4. **Given** a non-Admin or anonymous user attempts to change the hall or registration number, **When** the request is authorized, **Then** the change is denied and no value is modified.
5. **Given** an Admin assigns or corrects a hall or registration number, **When** the change is saved, **Then** the change is added to the admission record's audit history.

---

### User Story 3 - Calculate Graduation Eligibility (Priority: P1)

As an academic administrator, I need the portal to determine graduation eligibility automatically so a student is eligible only after completing the credits required for the selected degree and receiving both an assigned hall and registration number.

**Why this priority**: Degree credits, assigned hall, and registration number are all mandatory for graduation eligibility, though the hall and registration number are assigned after admission.

**Independent Test**: Evaluate students with incomplete credits, missing hall, or missing registration number; verify each is automatically ineligible and the unmet requirement is identified. Verify a student becomes eligible only when completed credits from passed course results meet the selected degree's configured requirement and both identifiers are present.

**Acceptance Scenarios**:

1. **Given** a student's passed course results show fewer completed credits than the selected degree requires, **When** graduation eligibility is evaluated, **Then** the student is marked ineligible and the outstanding credit requirement is identified.
2. **Given** a student has completed the required credits but is missing an assigned hall or registration number, **When** graduation eligibility is evaluated, **Then** the student is marked ineligible and each missing identifier is identified.
3. **Given** a student has completed the required credits and has both an assigned hall and registration number, **When** graduation eligibility is evaluated, **Then** the student is marked eligible without a separate clearance action.
4. **Given** a student's final academic result is ready, **When** that result is calculated or published, **Then** this feature does not block result calculation or publication; the graduation-eligibility check is separate.
5. **Given** degree credit requirements or passed course results are unavailable, **When** graduation eligibility is evaluated, **Then** the student is not marked eligible and the unavailable credit information is identified.

### Edge Cases

- An Admin attempts to create a duplicate student record from the same paper form.
- A returning Student applies to another Degree/Batch using the email already linked to their Student account.
- The applicant email belongs to an account with a non-Student role; the admission must not merge or change that account.
- A returning Student's earlier IIT program is already recorded as an admission; it must not also be duplicated as an academic-history entry.
- A Student has passed credits in one program admission; those credits must not automatically count toward a different program admission.
- An applicant has no email or mobile number, or the normalized email is already used by another portal account.
- Admission record or Student account persistence fails during the combined creation transaction.
- The credential-setup notification cannot be queued or delivered after the admission and account are committed.
- A student has an incomplete or unavailable value on the paper form.
- A student has some academic-history categories but not others.
- The source form provides different academic-history columns for secondary and postsecondary examinations.
- Degree credit requirements or passed course-result data are unavailable or inconsistent when eligibility is evaluated.
- Mailing and permanent addresses have different cell numbers, or one address cell number is absent.
- Hall assignment is recorded before registration number, or vice versa.
- An Admin attempts to reuse a registration number already assigned to another student.
- A mailing or permanent address refers to a missing, inactive, or deleted Upazila; the address must not retain an unresolved or contradictory geographic path.
- An admission record's degree or batch is corrected after hall or registration details have been assigned.
- Graduation eligibility is evaluated while required credit, hall, or registration data is missing.
- An unauthorized user attempts to view or modify a record containing personal information.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The portal MUST provide one Student profile per person and one common admission record format for each degree-program enrollment. Each admission MUST reference one degree and one batch selected from the available academic offerings. A Student profile MAY have multiple admissions for different programs; each admission remains separately identifiable.
- **FR-002**: An active Admin MUST be able to enter and maintain the applicant name, father's name, mother's name, mailing and permanent addresses, date of birth, place of birth and marital status when present on the source form, nationality, and local guardian/emergency contact details when present on the Student profile. Mailing and permanent addresses MUST be separate; each supplied address MUST contain one free-text address-details value and a selected Upazila, with its District and Division resolved from the existing geographic hierarchy. Corresponding address cell numbers are retained when present. The digital profile MUST require the applicant's email and mobile number; these are not present on the supplied paper forms, and the applicant mobile number is separate from any address or guardian cell number. Each admission MUST retain its admission test serial number, admission test roll number, and optional class roll. The admission test roll number is the field labeled "Exam Roll" on the applicable paper-form variant. The admission test serial number is separate and MUST NOT be conflated with the admission test roll number, class roll, or registration number. The paper declaration and applicant signature MUST remain physical and MUST NOT be captured digitally.
- **FR-003**: An active Admin MUST be able to record zero or more academic-history entries on a Student profile for SSC/O-Level, HSC/A-Level, Bachelor/Graduation, Master, PGDIT, and Other. Category MUST use one of those six fixed values. Each other applicable source-form field, including examination label, school/college, board, group, department, university, subject, institution, year, marks obtained, marks percentage, CGPA, class, and division, MUST be retained as optional text exactly as supplied; these values MUST NOT require matching mutable Configuration catalogs. Missing categories and fields MUST remain optional. Profile academic history MUST be available across all admissions for that Student.
- **FR-004**: The initial admission record MUST NOT require assigned hall or registration number. An active Admin MUST be able to record either value later without requiring the other value in the same update.
- **FR-005**: A registration number MUST NOT be assigned to more than one student admission record.
- **FR-006**: The portal MUST determine graduation eligibility automatically for each admission. A student MUST be eligible for that program only when completed credits from passed course results associated with that admission meet the required credit total configured for its selected degree and both its assigned hall and registration number are present; otherwise the admission MUST be marked ineligible and the unmet requirement(s) identified. Credits from a different admission MUST NOT carry over unless a separate approved transfer-credit rule is introduced. If degree credit settings or passed course results are unavailable, the admission MUST NOT be marked eligible and the unavailable credit information MUST be identified. This eligibility rule MUST NOT block final result calculation or publication.
- **FR-007**: Only an active Admin MUST be allowed to view, create, or change student admission records; hiding controls in the interface MUST NOT grant access.
- **FR-008**: Student profile, admission, and linked Student-account creation or changes MUST be auditable. Initial profile/account/admission creation MUST commit together or leave none created. A later admission for an existing Student MUST commit with its profile changes and audit event without creating another account. Unauthorized or rejected updates MUST NOT partially change data.
- **FR-009**: This feature MUST capture typed form information only. Photograph scans and other supporting documents remain with the physical paper file and MUST NOT be uploaded or stored by this feature.
- **FR-010**: The first admission for a person MUST create exactly one Student portal account and one Student profile linked to that account. Every admission MUST link to exactly one Student profile, and a profile MAY have multiple admissions. When the normalized applicant email identifies an existing Student account/profile, the portal MUST reuse it and create a distinct admission without creating another account. If the email identifies a non-Student account, the portal MUST reject the admission without partial changes or automatic account merging. A Student account without a profile MAY be linked by creating its profile as part of the admission transaction. Email MUST remain unique across portal accounts.
- **FR-011**: A newly created Student account MUST start in `PendingCredentialSetup` with no password and MUST remain unable to sign in until the student successfully completes the existing, purpose-specific credential-setup flow by setting a valid password. Only successful setup MUST transition the account to `Active`; invalid, expired, replayed, or undelivered setup proofs MUST NOT activate it. After a transaction that creates a new Student account commits, the system MUST request the existing secure credential-setup email. Delivery failure MUST leave the account pending, and existing Admin reissue behavior MUST be used. Attaching a profile to an existing Student account or creating another admission MUST NOT request setup again or change the existing account status/credential. The applicant mobile number MUST be retained on the Student profile for contact but MUST NOT be treated as verified unless separately specified.
- **FR-012**: Student Management MUST consume required-credit totals from the selected degree's settings and completed-credit totals from passed course results. It MUST NOT create or maintain duplicate degree-credit requirements, course registrations, or course-result records.
- **FR-013**: A Student profile MUST have a one-to-one relationship with a Student portal account and a one-to-many relationship with program admissions. Each admission MUST represent one Degree/Batch enrollment and retain its own Hall, registration number, and graduation eligibility. A Student profile MUST have zero or more academic-history entries shared across its admissions. A repeat admission for the same Student, Degree, and Batch MUST be rejected as a duplicate; an admission for another program or Batch MUST remain separate.
- **FR-014**: A previously recorded IIT program admission, such as PGDIT, MUST remain visible as a distinct admission under the same Student profile and MUST NOT be duplicated as an academic-history entry. An earlier program not recorded in Student Management MAY be entered once as academic history on the profile, visible across the person's admissions. Completed credits from one admission MUST NOT automatically count toward another admission's eligibility.
- **FR-015**: Each supplied mailing or permanent address MUST retain one free-text address-details value and reference a valid Upazila from the existing Division → District → Upazila hierarchy. The corresponding District and Division MUST be derived from that Upazila and displayed consistently; Student Management MUST NOT create duplicate geographic records or accept a conflicting parent path.

### Key Entities *(include if feature involves data)*

- **Student profile**: The person-level record linked one-to-one to a Student-role portal account; it holds shared personal/contact details, separate mailing/permanent address details with Upazila references, and can have multiple program admissions.
- **Student admission record**: One program enrollment for a Student profile, associated with one Degree and Batch and containing admission-specific Hall, registration number, and computed eligibility. Multiple admissions do not create duplicate portal accounts.
- **StudentAcademicHistory**: One prior-education row owned by a Student profile and shared across its admissions, using one of six fixed categories and retaining applicable source-form fields as optional text snapshots rather than mutable catalog references.
- **Degree and batch**: The academic offerings selected for the student's admission record.
- **Assigned hall**: The hall recorded by an Admin after initial admission entry and required for graduation eligibility.
- **Registration number**: The student identifier recorded by an Admin after initial admission entry; it is unique across student admission records and required for graduation eligibility.
- **Graduation eligibility**: A computed state for one admission, satisfied only when passed course results associated with that admission show the required credits configured for its selected degree and that admission has both an assigned hall and registration number; credits do not carry between admissions and there is no separate clearance action.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of program admissions contain a selected degree and batch and use the same common field set regardless of degree.
- **SC-002**: At least 90% of representative Admins can enter a completed paper form into a student record in five minutes or less without developer assistance.
- **SC-003**: 100% of program admissions with incomplete applicable credits or a missing Hall or registration number are marked ineligible with the unmet requirement identified; an admission meeting all three conditions is marked eligible automatically.
- **SC-004**: 100% of attempts by anonymous or non-Admin users to view or change student admission records are denied without exposing personal data.
- **SC-005**: 100% of duplicate registration-number assignments are rejected without changing either student record.
- **SC-006**: 100% of Student profiles have exactly one linked Student-role account, every program admission links to one Student profile, and missing or conflicting applicant contact identity data never leaves a partial profile, admission, or account.
- **SC-007**: 100% of newly created Student accounts remain unable to sign in while pending and become Active only after successfully setting a password through the existing credential-setup flow; later admissions reuse the account without resetting credentials or reissuing setup.
- **SC-008**: 100% of academic-history entries retain every field supplied for their category on the source form, without requiring fields that the form leaves blank or does not include.
- **SC-009**: 100% of supplied mailing and permanent addresses resolve to the correct Upazila, District, and Division, and invalid or inconsistent geographic selections are rejected.

## Assumptions

- The Admin enters the information from a paper form; students do not submit admission information through a self-service portal in this feature.
- The same common set of personal and academic-history fields applies to every degree program; degree-specific custom fields are outside this scope.
- Degree and batch choices use the portal's available academic offerings.
- The original paper form, photograph, and supporting attachments remain in physical custody and are not digitized by this feature.
- The applicant declaration and signature remain on the physical form; the portal does not record their text, signed status, date, or image.
- Applicant email and mobile number are mandatory digital admission fields even though the paper form does not request them; no mobile verification or SMS workflow is implied.
- The first admission creates a Student account with no password and leaves it pending until the existing emailed credential-setup flow is completed. Later program admissions reuse the existing Student account/profile without another setup email. A matching email on a non-Student account is rejected rather than silently linked or merged.
- A person may have multiple program admissions under one Student profile. A repeated admission for the same Student, Degree, and Batch is a duplicate; different program or Batch admissions remain separately identifiable.
- Academic-history entries belong to the Student profile, not to an individual admission, and remain visible across all admissions for that person.
- A completed IIT program already recorded in Student Management remains a prior admission, not a duplicate academic-history entry. Credits remain admission-specific; no cross-program credit transfer is implied.
- The degree owner maintains each degree's required-credit total, and the academic-results owner provides completed credits from passed course results; Student Management consumes these sources but does not own or duplicate their records.
- Registration numbers are unique across the portal's student admission records.
- Graduation eligibility is calculated automatically and is distinct from calculating or publishing an academic result; eligibility requires the selected degree's credits from passed course results, assigned hall, and registration number.
