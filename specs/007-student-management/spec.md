# Feature Specification: Student Management

**Feature Branch**: `feature/007-student-management`

**Created**: 2026-10-18

**Status**: Draft

**Input**: User description: "Use one common admission form for all student types. An Admin receives the paper form and attachments, enters the information, and selects the degree and batch. A Student-role portal user is created when student information is recorded. Students must provide an email and mobile number to complete admission. Assigned hall and registration number are entered later; both must be present before a student can complete graduation."

## Clarifications

### Session 2026-10-18

- Q: Who should be allowed to perform graduation clearance once the student has the required credits, hall, and registration number? → A: Graduation eligibility is calculated automatically; there is no separate clearance action or user role. A student is eligible only when all requirements are met.
- Q: What should be the authoritative source for each degree’s required credits and a student’s completed credits? → A: The selected degree’s settings provide its required credit total, and passed course results provide the student’s completed credits. Student Management consumes these values and does not duplicate or own course/result records.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Enter a Student Admission Record (Priority: P1)

As an active Admin, I need to enter a student's paper admission form and required contact details into one common record so the portal has consistent enrollment information and a linked Student account for every degree and batch.

**Why this priority**: Student-specific enrollment data is the foundation for later administrative updates and graduation checks.

**Independent Test**: As an active Admin, enter sample records for two degree programs, provide each student's email and mobile number, and select a batch. Verify each admission record is linked to exactly one Student-role portal account in pending credential setup, with one setup email requested after commit. Verify missing contact details or an email already used by another account creates no partial admission/account, and a non-Admin cannot create or inspect records.

**Acceptance Scenarios**:

1. **Given** an active Admin has received a completed paper form and the applicant has supplied an email and mobile number, **When** the Admin enters the information and selects a degree and batch, **Then** one common student admission record and exactly one linked portal account with the Student role are created together.
2. **Given** an Admin records prior education, **When** the record is saved, **Then** each supplied SSC/O-Level, HSC/A-Level, Bachelor/Graduation, Master, PGDIT, or Other entry retains the applicable school/college, board, group, department, university, year, marks, percentage, CGPA, class, division, and subject values shown on the source form.
3. **Given** a non-Admin or anonymous user requests student admission data, **When** the request is authorized, **Then** access is denied without disclosing personal information.
4. **Given** a paper form or supporting documents are received, **When** the Admin enters the record, **Then** the portal stores the form's typed data only; scanned attachments and document images are not uploaded or retained by this feature.
5. **Given** the admission record and Student account are committed, **When** account setup is requested, **Then** the linked account remains pending without a password and uses the existing secure credential-setup email flow; email delivery failure does not partially roll back or activate the account.
6. **Given** the applicant email or mobile number is missing or the normalized email already identifies another portal account, **When** the Admin attempts to complete admission, **Then** the operation is rejected without creating a partial admission record or account.
7. **Given** an Admin creates an admission record and linked account, **When** they are saved, **Then** the changes are recorded in audit history and no password or setup proof is exposed to the Admin.
8. **Given** the linked Student account is pending, **When** the student successfully follows the existing credential-setup email and sets a valid password, **Then** the account becomes Active and can sign in with the Student role. An invalid, expired, or undelivered setup link MUST NOT activate the account.

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
- An admission record's degree or batch is corrected after hall or registration details have been assigned.
- Graduation eligibility is evaluated while required credit, hall, or registration data is missing.
- An unauthorized user attempts to view or modify a record containing personal information.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The portal MUST provide one common student admission record format across all degree programs. Each record MUST reference one degree and one batch selected from the available academic offerings.
- **FR-002**: An active Admin MUST be able to enter the applicant name, father's name, mother's name, mailing and permanent addresses and their cell numbers when present, date of birth, place of birth and marital status when present on the source form, nationality, admission test serial number, admission test roll number, class roll when present, and local guardian/emergency contact details when present. The admission test roll number is the field labeled "Exam Roll" on the applicable paper-form variant. The admission test serial number is a separate field and MUST NOT be conflated with the admission test roll number, class roll, or later registration number. The digital admission record MUST also require the applicant's email and mobile number; these are not present on the supplied paper forms, and the applicant mobile number is separate from any address or guardian cell number. The paper declaration and applicant signature MUST remain physical and MUST NOT be captured digitally.
- **FR-003**: An active Admin MUST be able to record zero or more academic-history entries for SSC/O-Level, HSC/A-Level, Bachelor/Graduation, Master, PGDIT, and Other. Each entry MUST retain the fields supplied by its source form, including examination, school/college, board, group, department, university, subject, institution, year, marks obtained, marks percentage, CGPA, class, and division as applicable. Missing categories MUST remain optional.
- **FR-004**: The initial admission record MUST NOT require assigned hall or registration number. An active Admin MUST be able to record either value later without requiring the other value in the same update.
- **FR-005**: A registration number MUST NOT be assigned to more than one student admission record.
- **FR-006**: The portal MUST determine graduation eligibility automatically. A student MUST be eligible only when completed credits from passed course results meet the required credit total configured for the selected degree and both assigned hall and registration number are present; otherwise the student MUST be marked ineligible and the unmet requirement(s) identified. If degree credit settings or passed course results are unavailable, the student MUST NOT be marked eligible and the unavailable credit information MUST be identified. This eligibility rule MUST NOT block final result calculation or publication.
- **FR-007**: Only an active Admin MUST be allowed to view, create, or change student admission records; hiding controls in the interface MUST NOT grant access.
- **FR-008**: Student admission data and linked Student-account creation or changes MUST be auditable, and admission/account creation MUST commit together or leave neither record created. Unauthorized or rejected updates MUST NOT partially change a record.
- **FR-009**: This feature MUST capture typed form information only. Photograph scans and other supporting documents remain with the physical paper file and MUST NOT be uploaded or stored by this feature.
- **FR-010**: Completing admission MUST create exactly one portal account linked to the student admission record with the Student role. The normalized applicant email MUST be unique across portal accounts; if it already belongs to another account, admission completion MUST be rejected without partial changes.
- **FR-011**: The linked Student account MUST start in `PendingCredentialSetup` with no password and MUST remain unable to sign in until the student successfully completes the existing, purpose-specific credential-setup flow by setting a valid password. Only successful setup MUST transition the account to `Active`; invalid, expired, replayed, or undelivered setup proofs MUST NOT activate it. After the admission/account transaction commits, the system MUST request the existing secure credential-setup email for the applicant's email address. Delivery failure MUST leave the account pending, and existing Admin reissue behavior MUST be used rather than creating a second account. The applicant mobile number MUST be retained for contact but MUST NOT be treated as verified unless separately specified.
- **FR-012**: Student Management MUST consume required-credit totals from the selected degree's settings and completed-credit totals from passed course results. It MUST NOT create or maintain duplicate degree-credit requirements, course registrations, or course-result records.

### Key Entities *(include if feature involves data)*

- **Student admission record**: The common enrollment record for a student, associated with a degree and batch and linked to exactly one Student-role portal account; it contains personal, contact, admission, and deferred identifier information entered from the paper form.
- **Academic-history entry**: One prior-education row categorized as SSC/O-Level, HSC/A-Level, Bachelor/Graduation, Master, PGDIT, or Other, retaining the applicable school/college, board, group, department, university, subject, year, marks, percentage, CGPA, class, and division fields from its source form.
- **Degree and batch**: The academic offerings selected for the student's admission record.
- **Assigned hall**: The hall recorded by an Admin after initial admission entry and required for graduation eligibility.
- **Registration number**: The student identifier recorded by an Admin after initial admission entry; it is unique across student admission records and required for graduation eligibility.
- **Graduation eligibility**: A computed state that is satisfied only when passed course results show the required credits configured for the selected degree and the student has both an assigned hall and registration number; there is no separate clearance action.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of records created through this feature contain a selected degree and batch and use the same common field set regardless of degree.
- **SC-002**: At least 90% of representative Admins can enter a completed paper form into a student record in five minutes or less without developer assistance.
- **SC-003**: 100% of students with incomplete degree credits or a missing hall or registration number are marked ineligible with the unmet requirement identified; students meeting all three conditions are marked eligible automatically.
- **SC-004**: 100% of attempts by anonymous or non-Admin users to view or change student admission records are denied without exposing personal data.
- **SC-005**: 100% of duplicate registration-number assignments are rejected without changing either student record.
- **SC-006**: 100% of completed admissions have exactly one linked Student-role account, and missing or conflicting applicant contact identity data never leaves a partial admission or account record.
- **SC-007**: 100% of newly linked Student accounts remain unable to sign in while pending and become Active only after successfully setting a password through the existing credential-setup flow.
- **SC-008**: 100% of academic-history entries retain every field supplied for their category on the source form, without requiring fields that the form leaves blank or does not include.

## Assumptions

- The Admin enters the information from a paper form; students do not submit admission information through a self-service portal in this feature.
- The same common set of personal and academic-history fields applies to every degree program; degree-specific custom fields are outside this scope.
- Degree and batch choices use the portal's available academic offerings.
- The original paper form, photograph, and supporting attachments remain in physical custody and are not digitized by this feature.
- The applicant declaration and signature remain on the physical form; the portal does not record their text, signed status, date, or image.
- Applicant email and mobile number are mandatory digital admission fields even though the paper form does not request them; no mobile verification or SMS workflow is implied.
- A new Student account is created with no password and remains pending until the existing emailed credential-setup flow is completed. Email conflicts are rejected rather than silently linking an unrelated account.
- The degree owner maintains each degree's required-credit total, and the academic-results owner provides completed credits from passed course results; Student Management consumes these sources but does not own or duplicate their records.
- Registration numbers are unique across the portal's student admission records.
- Graduation eligibility is calculated automatically and is distinct from calculating or publishing an academic result; eligibility requires the selected degree's credits from passed course results, assigned hall, and registration number.
