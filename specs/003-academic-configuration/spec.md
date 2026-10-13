# Feature Specification: Academic Configuration

**Feature Branch**: `main` (no Specify branch hook is configured)

**Created**: 2026-10-13

**Status**: Draft

**Input**: User description: Provide Admins with reusable academic configuration catalogs, batch setup, and Bangladesh Division/District/Upazila reference data for future portal features.

## Clarifications

### Session 2026-10-13

- Q: At what scope should a Batch identifier be unique? → A: A Batch identifier is unique within a Degree; different Degrees may reuse the same identifier.
- Q: Which versioned official dataset should supply the initial Division, District, and Upazila hierarchy? → A: The owner provides a JSON seed file citing the approved official source and version.
- Q: What should happen when an Admin tries to deactivate a Degree, Semester, Division, or District that already has dependent records? → A: Allow deactivation, preserve existing references and their display, block new assignments to the inactive record, and do not cascade deactivation.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Maintain Academic Reference Catalogs (Priority: P1)

As an authorized Admin, I need to maintain the academic reference catalogs used by future student, teacher, and academic-operation features so that those features can share consistent, validated values.

**Why this priority**: These independent catalogs establish the shared vocabulary future portal modules need and can be delivered without student-history or teacher-management workflows.

**Independent Test**: As Admin, create, view, update, search, deactivate, and list active/inactive records for each independent catalog; verify the specified fields, uniqueness rules, validation messages, and audit records without creating a student or teacher profile.

**Acceptance Scenarios**:

1. **Given** an Admin creates or edits an educational institution, **When** the request is valid, **Then** the institution's name, fixed type (School, College, or University), optional code/address, and active status are maintained without creating student academic history.
2. **Given** an Admin maintains an education board, education group, examination type, degree, semester, or designation, **When** a valid create or update is submitted, **Then** the corresponding independent catalog record is maintained without an unintended relationship to another catalog, student, or teacher.
3. **Given** an Admin submits a duplicate normalized name or optional code for the same catalog, **When** the record is validated, **Then** the duplicate is rejected with a clear field-level explanation.
4. **Given** an Admin deactivates a catalog record, **When** future features request selectable configuration values, **Then** the inactive record is not offered for new selection while remaining available for existing reference display.
5. **Given** a user without authorized Admin access calls a configuration-management operation directly, **When** the server evaluates the request, **Then** it denies the operation regardless of interface visibility.

### User Story 2 - Configure Academic Batches (Priority: P1)

As an authorized Admin, I need to define student cohorts against an IIT degree and starting semester so that future enrollment and student-management features can reference valid batches.

**Why this priority**: Batch records connect academic programs and starting semesters while leaving enrollment and progression to their own features.

**Independent Test**: Create and edit batches with valid and invalid degree/semester references; verify exact-one relationships, duplicate handling, status behavior, and unchanged independent maintenance of degree and semester catalogs.

**Acceptance Scenarios**:

1. **Given** an Admin creates a batch, **When** the supplied degree and starting semester are active and exist, **Then** the batch references exactly one of each and stores its name/identifier, optional admission session, and active status.
2. **Given** a batch references a nonexistent, inactive, or otherwise disallowed degree or starting semester, **When** the batch is saved, **Then** the request is rejected without creating or changing the batch.
3. **Given** an Admin submits a batch identifier already used by another Batch for the same Degree, **When** the record is validated, **Then** the duplicate is rejected without changing another batch.
4. **Given** an Admin creates, views, updates, or deactivates a batch, **When** the operation succeeds, **Then** no student enrollment, student assignment, semester progression, or other academic workflow is performed.
5. **Given** two Batches use the same identifier for different Degrees, **When** both records are validated, **Then** both are allowed.

### User Story 3 - Maintain Bangladesh Geographic Reference Data (Priority: P1)

As an authorized Admin, I need the approved Division, District, and Upazila hierarchy available for future forms so that geographic selections remain accurate and consistent.

**Why this priority**: The geographic hierarchy is shared reference data needed by future student and address workflows, but it must remain independent of those workflows.

**Independent Test**: Load the approved source dataset, verify every parent-child relationship and code/name uniqueness rule, then query and maintain the hierarchy while confirming the catalog is not coupled to student profiles or institution addresses.

**Acceptance Scenarios**:

1. **Given** the approved geographic dataset is loaded, **When** an Admin or future selector requests the hierarchy, **Then** it exposes Divisions, their Districts, and each District's Upazilas with stable identifiers and parent relationships.
2. **Given** an Admin creates or updates a District or Upazila, **When** its parent does not exist or the supplied parent is invalid, **Then** the change is rejected and the existing hierarchy is unchanged.
3. **Given** an Admin attempts to physically remove a Division or District with child records, **When** the request is processed, **Then** the operation is rejected without cascading deletion or orphaning children.
4. **Given** a user supplies geographic values from different branches of the hierarchy, **When** the parent-child relationship is validated, **Then** the invalid combination is rejected.
5. **Given** the approved geographic source is unavailable or cannot be verified, **When** initial data is prepared, **Then** the portal does not invent records or require manual entry of the entire national hierarchy.

### User Story 4 - Protect Configuration Changes and References (Priority: P1)

As an institutional administrator, I need configuration changes authorized, audited, and protected against invalid references so that shared academic settings remain trustworthy and historical use remains understandable.

**Why this priority**: Incorrect, unauthorized, or unaudited reference-data changes can affect multiple future academic workflows.

**Independent Test**: Attempt authorized and unauthorized writes through the UI and direct API; perform valid, duplicate, and referentially invalid operations; verify audit history, data integrity, and inactive-record behavior.

**Acceptance Scenarios**:

1. **Given** an Admin successfully creates, updates, or deactivates a configuration record, **When** the change commits, **Then** the existing audit system records the actor, action, entity type, record identifier, timestamp, and approved change summary.
2. **Given** a configuration write requires a business audit event, **When** audit persistence fails, **Then** the configuration change does not commit.
3. **Given** an authenticated session is not actively using the Admin role, **When** it attempts a configuration write, **Then** the server denies the request even if the account has an Admin assignment in another inactive role.
4. **Given** a referenced Degree, Semester, Division, or District is deactivated, **When** existing batches or geographic children are viewed or new assignments are attempted, **Then** existing references remain identifiable and unchanged, new assignments to the inactive record are blocked, and deactivation does not cascade to dependents.

## Edge Cases

- A name differs only by surrounding whitespace, letter case, or repeated internal spaces; normalization prevents inappropriate duplicates within that catalog.
- Two batches use the same identifier for the same Degree, even if their admission sessions differ; reject the duplicate because uniqueness is scoped to Degree, not admission session.
- An optional code is omitted, blank, or differs only by case from an existing code; blank is treated as absent and normalized non-empty codes follow the catalog's uniqueness constraint.
- Two catalog types contain the same name, such as an examination type and an IIT degree; they remain independent and do not conflict across catalog types.
- A referenced Degree or starting Semester is deactivated; existing Batch references and display remain intact, new Batch assignments to it are blocked, and no Batch is cascade-deactivated.
- A Division or District with children is deactivated; existing geographic parent-child links remain intact, new children or assignments to the inactive parent are blocked, and no descendant is cascade-deactivated.
- A Division has Districts or a District has Upazilas; physical deletion cannot cascade and must not create orphan rows.
- A geographic data load contains duplicate, missing-parent, or unknown records; loading reports the invalid entries and does not silently create partial or invented hierarchy data.
- An Admin submits an invalid parent ID or a District/Upazila whose parent is from another hierarchy branch; reject without changing persisted relationships.
- A user changes the active role away from Admin while on a configuration page; subsequent writes are denied by the server and the UI must not preserve Admin write access.
- Audit storage is unavailable during a required configuration change; the change is rolled back and the failure follows the existing audit operational policy.
- Inactive records are displayed in existing-reference views with an inactive indication but are omitted from selectors for new assignments unless an owning feature explicitly requires historical selection.

## Requirements *(mandatory)*

### Business Rules

- The feature maintains only reusable reference/configuration data; it does not create or edit student histories, student profiles, teacher profiles, enrollments, or academic-operation records.
- Only a session actively acting in the Admin role may create, update, or deactivate configuration records. The server checks the current session role on every request; assigned roles, Angular navigation, or a client-supplied role do not grant permission.
- Institution type is a fixed controlled value: School, College, or University. No separate institution-type management screen is part of this feature.
- Educational institutions, education boards, education groups, examination types, IIT degrees/programs, semesters, and designations are independent catalogs except for the explicitly stated Batch and geographic relationships.
- SSC/O-Level, HSC/A-Level, Bachelor, Master, PGDIT, and Others are previous-education examination/qualification categories, not IIT degree records.
- A Batch references exactly one IIT degree/program and exactly one starting semester. This does not make Degree or Semester depend on a Batch and does not implement semester progression.
- Geographic references form Division → District → Upazila. They are independent of student profiles and institution addresses.
- Names are compared after trimming and case normalization within their own catalog. Optional non-empty codes are normalized and unique within the corresponding catalog. Geographic names are unique within their parent; geographic codes are unique within their administrative level. A Batch identifier is unique within its Degree; a different Degree may reuse it.
- When a referenced Degree, Semester, Division, or District is deactivated, preserve existing references and their display, block new assignments to it, and do not cascade deactivation to dependents.
- Successful configuration changes use the existing audit system and its current Admin-session authorization. Mandatory change audit is committed atomically with the configuration change; validation errors are not treated as committed business changes.
- Use deactivation instead of physical deletion where referenced data exists. Physical deletion must never cascade or violate references.

### Functional Requirements

#### Catalog maintenance

- **FR-001**: The portal MUST provide authorized Admins with create, view, update, list, search, filter, and deactivate operations for Educational Institutions. Records contain a required name and type (School, College, University), optional code and address, and active/inactive status.
- **FR-002**: The portal MUST provide an independent Education Boards catalog with required board name, optional code, and active/inactive status; it MUST NOT associate a board with a student or institution in this feature.
- **FR-003**: The portal MUST provide independent Education Groups with required name and active/inactive status for values such as Science, Humanities, and Business Studies.
- **FR-004**: The portal MUST provide independent Examination Types for previous-education categories, including SSC/O-Level, HSC/A-Level, Bachelor, Master, PGDIT, and Others. These records MUST NOT be treated as IIT degrees or contain student academic-history fields.
- **FR-005**: The portal MUST provide independent Degrees for programs offered by IIT, with required name, optional code, and active/inactive status. Degree records MUST NOT depend on a batch or semester.
- **FR-006**: The portal MUST provide independent Semesters with a required name/label, active/inactive status, and an ordering value when required by the existing domain. A Semester MUST NOT depend on a degree, batch, or progression workflow.
- **FR-007**: The portal MUST provide independent Designations with required name, optional code, and active/inactive status; assignment to a Teacher remains out of scope.
- **FR-008**: For each catalog, the portal MUST validate required values and reject duplicate normalized names and duplicate normalized non-empty codes within that catalog. Identical names in different catalogs are allowed.

#### Batches and geographic hierarchy

- **FR-009**: The portal MUST maintain Batches with a required name/identifier, exactly one existing Degree, exactly one existing starting Semester, optional admission session, and active/inactive status.
- **FR-010**: Batch creation/update MUST reject nonexistent or disallowed Degree/Semester references and duplicate identifiers within the same Degree. The same identifier MAY be used by a Batch under a different Degree; admission session does not change this uniqueness scope.
- **FR-011**: Degrees and Semesters MUST remain independently maintainable when no Batch references them; batch creation MUST NOT assign students or advance a semester.
- **FR-012**: The portal MUST maintain Divisions with required name and code, Districts with required name and code referencing exactly one Division, and Upazilas with required name and code referencing exactly one District.
- **FR-013**: Geographic validation MUST enforce parent-child referential integrity and uniqueness: Division codes within Divisions; District names/codes within the selected Division; Upazila names/codes within the selected District.
- **FR-014**: Geographic data MUST be loaded from an owner-provided JSON seed file that cites the approved official source and version and is accompanied by a validation report. The portal MUST NOT invent geography records or require administrators to enter the entire hierarchy manually.
- **FR-015**: The portal MUST expose data relationships sufficient for future cascading Division → District → Upazila selection without coupling the catalog to student profiles or institution addresses.

#### Lifecycle, authorization, and audit

- **FR-016**: Referenced configuration records MUST NOT be physically deleted in a way that violates foreign keys, cascades away historical references, or leaves an orphan. The Admin interface MUST provide deactivation; any deletion of an unreferenced record must follow existing data-integrity conventions.
- **FR-017**: Inactive records MUST be excluded from new assignment selectors and remain identifiable in existing-reference views. Deactivating a referenced Degree, Semester, Division, or District MUST preserve existing links, block new assignments to that record, and MUST NOT cascade deactivation to dependents.
- **FR-018**: Every configuration write MUST be authorized server-side against the current session's active Admin role, regardless of UI visibility or client-submitted role information.
- **FR-019**: Successful configuration create, update, and deactivate operations MUST use the existing audit system and record actor, action, entity type, record identifier, UTC timestamp, outcome, and allowlisted change summary. The event and configuration change MUST commit atomically where the existing audit contract requires it.
- **FR-020**: Configuration APIs MUST return safe validation errors for missing/invalid fields, duplicate names/codes, nonexistent references, and invalid parent-child combinations; rejected validation attempts MUST NOT change stored data.
- **FR-021**: Administrative list/search experiences MUST show loading, empty, error, and inactive states; support search/filter/pagination where the catalog volume warrants it; and follow the shared design system and accessibility rules.

#### Acceptance Criteria

- **AC-001 - Independent catalogs**: Each catalog in FR-001–FR-008 can be maintained independently; validated names/codes reject duplicates within its catalog, while similar names in another catalog remain allowed. (FR-001–FR-008)
- **AC-002 - Batch integrity**: Every Batch references exactly one valid Degree and one valid starting Semester; invalid references and duplicate identifiers within the same Degree are rejected, the same identifier is accepted for a different Degree regardless of admission session, and Degree/Semester remain independently maintainable. (FR-005–FR-006, FR-009–FR-011)
- **AC-003 - Geographic integrity and seed**: Division, District, and Upazila data follow the confirmed source and exact parent relationships; invalid/cross-branch relationships and duplicate geographic values at the specified parent/level are rejected. (FR-012–FR-015)
- **AC-004 - Lifecycle safety**: Deactivation of a referenced Degree, Semester, Division, or District preserves existing links and display, blocks new assignments to the inactive record, and does not cascade deactivation. Physical deletion never cascades away or orphans dependents. (FR-016–FR-017)
- **AC-005 - Authorization and audit**: 100% of unauthorized direct API writes are denied; successful create/update/deactivate operations produce the required audit event atomically, and failed validation does not change stored data. (FR-018–FR-020)
- **AC-006 - Administration UX**: Admins can find and maintain catalogs using accessible, responsive list/form states consistent with the project design system. (FR-021)

## Key Entities *(include if feature involves data)*

- **Educational Institution**: Independent reference for a School, College, or University; required name/type, optional code/address, status.
- **Education Board**: Independent examination authority; required name, optional code, status.
- **Education Group**: Independent school/college academic group; required name and status.
- **Examination Type**: Independent previous-education qualification category; name and status; not an IIT degree or student-history record.
- **Degree**: Independent IIT-offered degree/program; name, optional code, status.
- **Semester**: Independent semester label and optional domain-required order; status.
- **Designation**: Independent teacher designation catalog; name, optional code, status.
- **Batch**: Cohort with one Degree reference, one starting Semester reference, name/identifier, optional admission session, status.
- **Division**: Bangladesh administrative top-level geographic reference with name/code/status.
- **District**: Geographic reference with one Division parent and name/code/status.
- **Upazila**: Geographic reference with one District parent and name/code/status.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of tested duplicate normalized names/codes within a catalog are rejected, and the same name in separate catalogs is not incorrectly rejected.
- **SC-002**: 100% of invalid/nonexistent Batch Degree or starting-Semester references and invalid geographic parent-child relationships are rejected without modifying stored data.
- **SC-003**: 100% of configuration writes by sessions not actively in the Admin role are denied at the server boundary.
- **SC-004**: 100% of successful configuration create/update/deactivate operations produce the required audit record atomically; failed validation leaves both configuration and audit history unchanged except for existing security-event rules.
- **SC-005**: 100% of loaded geographic records match the approved source/version and pass parent-reference validation before being made selectable.
- **SC-006**: At least 95% of representative Admin participants can find and update or deactivate a specified catalog record within two minutes without assistance.

## Assumptions

- The existing active Admin role and server-side policy authorize configuration management; no new roles or authentication changes are introduced.
- Fixed institution types are School, College, and University; no separate type-management UI is required.
- Catalog names are trimmed and compared case-insensitively within their entity type. Optional non-empty codes are likewise normalized and unique within their catalog. Geographic names/codes are unique within the stated parent/level scope.
- Batch identifiers are unique per Degree, not globally and not per admission session.
- Existing references use stable identifiers and remain understandable when records are deactivated. New selectors omit inactive values; deactivation never cascades to existing dependents.
- The audit system already defined in `specs/002-audit-system/spec.md` is the sole audit implementation. Config changes requiring audit use its business-event contract and same-transaction behavior.
- No existing configuration entities/models were found during repository inspection; this feature introduces the first catalogs in the existing layered monolith.
- The designated owner provides a JSON seed file that identifies the approved official geographic source and version and includes validation results.

## Out of Scope

- Student academic-history records, student profile changes, admissions, enrollment, or batch assignment.
- Teacher profiles, designation assignment, or teacher-management workflows.
- Student education-form submission workflows.
- Course offerings, course registration, semester progression, fee payments, examination results, or grading.
- Geographic units below Upazila (unions, wards, villages, or similar).
- A separate institution-type administration module unless future approved requirements require dynamic types.
- A separate backend service, frontend framework, authentication model, or competing audit implementation.

## Clarifications
