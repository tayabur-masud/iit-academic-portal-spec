# Feature Specification: Configuration

**Feature Branch**: `feature/003-configuration` (no Specify branch hook is configured)

**Created**: 2026-10-13

**Status**: Draft

**Input**: User description: Provide Admins with reusable configuration catalogs for academic workflows, batch setup, and Bangladesh Division/District/Upazila reference data for future portal features.

## Clarifications

### Session 2026-10-14

- Q: Should entities record who created and last changed them, and when? → A: Yes, every business entity, automatically (created by, created time, last modified by, last modified time), and the rule is written down so future entities get it too. It is delivered as a shared `AuditedEntity` base class stamped by the database context from the server clock and the signed-in account, with a test that fails when a new entity is neither derived nor explicitly exempt (FR-025; tasks T049 to T053).
- Q: Should every delete be a soft delete? → A: Yes. Deleting a configuration record marks it deleted (who and when are kept) instead of removing the row; the record disappears from every list, lookup, and selector, and the portal has no way to restore it. A record that live records still refer to cannot be deleted (it is refused and deactivation is offered, as before). A deleted record's name and code can be used again, so uniqueness applies to live records only. This replaces the earlier requirement for physical deletion (FR-016); tasks T045 to T048.
- Q: What is a Hall, and what does it hold? → A: A university residential hall, an independent reference catalog (no link to students in this feature). It has a required name, an optional code that is unique within halls, an optional address, and an active/inactive status. Added after the original scope; it is delivered as an addendum (tasks T038 to T043). The owner supplied the existing halls (23 halls and hostels of the University of Dhaka), loaded by a saved, re-runnable SQL seed with fixed identifiers and the default address "University of Dhaka, Dhaka-1000, Bangladesh" for every hall.

### Session 2026-10-13

- Q: At what scope should a Batch name be unique? → A: A Batch name is unique within a Degree; different Degrees may reuse the same name.
- Q: Which versioned official dataset should supply the initial Division, District, and Upazila hierarchy? → A: The owner provides a JSON seed file citing the approved official source and version.
- Q: What should happen when an Admin tries to deactivate a Degree, Semester, Division, or District that already has dependent records? → A: Allow deactivation, preserve existing references and their display, block new assignments to the inactive record, and do not cascade deactivation.
- Q: How should academic Sessions relate to Batches? → A: Add a Session catalog with a globally unique normalized label; every Batch must reference exactly one Session, replacing its free-text admission session.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Maintain Academic Reference Catalogs (Priority: P1)

As an authorized Admin, I need to maintain the academic reference catalogs used by future student, teacher, and academic-operation features so that those features can share consistent, validated values.

**Why this priority**: These independent catalogs establish the shared vocabulary future portal modules need and can be delivered without student-history or teacher-management workflows.

**Independent Test**: As Admin, create, view, update, search, deactivate referenced records, and delete unreferenced records for each independent catalog; verify the specified fields, uniqueness rules, validation messages, and audit records without creating a student or teacher profile.

**Acceptance Scenarios**:

1. **Given** an Admin creates or edits an educational institution, **When** the request is valid, **Then** the institution's name, fixed type (School, College, or University), optional code/address, and active status are maintained without creating student academic history.
2. **Given** an Admin maintains an education board, education group, examination type, degree, semester, designation, or hall, **When** a valid create or update is submitted, **Then** the corresponding independent catalog record is maintained without an unintended relationship to another catalog, student, or teacher.
3. **Given** an Admin submits a duplicate normalized name or optional code for the same catalog, **When** the record is validated, **Then** the duplicate is rejected with a clear field-level explanation.
4. **Given** an Admin deactivates a catalog record, **When** future features request selectable configuration values, **Then** the inactive record is not offered for new selection while remaining available for existing reference display.
5. **Given** a user without authorized Admin access calls a configuration-management operation directly, **When** the server evaluates the request, **Then** it denies the operation regardless of interface visibility.
6. **Given** an Admin opens the signed-in navigation, **When** the active role is Admin, **Then** the Configuration menu entry appears in the Admin module; it is absent for other active roles.
7. **Given** an Admin deletes an unreferenced catalog record, **When** the delete succeeds, **Then** the record is soft deleted (marked deleted, hidden from every list and selector, and kept in the database) and an audit event is committed; a record with dependents cannot be deleted and remains available for deactivation instead.

### User Story 2 - Configure Academic Batches (Priority: P1)

As an authorized Admin, I need to define student cohorts against an IIT degree, starting semester, and academic session so that future enrollment and student-management features can reference valid batches.

**Why this priority**: Batch records connect academic programs and starting semesters while leaving enrollment and progression to their own features.

**Independent Test**: Create and edit batches with valid and invalid degree/semester/session references; verify each required relationship, duplicate handling, status behavior, and unchanged independent maintenance of the related catalogs.

**Acceptance Scenarios**:

1. **Given** an Admin creates a batch, **When** the Degree, starting Semester, and Session exist and are active, **Then** the batch references exactly one of each and stores its name and active status.
2. **Given** a batch omits or references a nonexistent, inactive, or otherwise disallowed Degree, starting Semester, or Session, **When** it is saved, **Then** the request is rejected without creating or changing the batch.
3. **Given** an Admin submits a normalized Batch name already used by another Batch for the same Degree, **When** the record is validated, **Then** the duplicate is rejected without changing another batch.
4. **Given** an Admin creates, views, updates, deactivates, or deletes an unreferenced batch, **When** the operation succeeds, **Then** no student enrollment, student assignment, semester progression, or other academic workflow is performed.
5. **Given** two Batches use the same normalized name for different Degrees, **When** both records are validated, **Then** both are allowed.

### User Story 3 - Maintain Bangladesh Geographic Reference Data (Priority: P1)

As an authorized Admin, I need the approved Division, District, and Upazila hierarchy available for future forms so that geographic selections remain accurate and consistent.

**Why this priority**: The geographic hierarchy is shared reference data needed by future student and address workflows, but it must remain independent of those workflows.

**Independent Test**: Load the approved source dataset, verify every parent-child relationship and code/name uniqueness rule, then query and maintain the hierarchy while confirming the catalog is not coupled to student profiles or institution addresses.

**Acceptance Scenarios**:

1. **Given** the approved geographic dataset is loaded, **When** an Admin or future selector requests the hierarchy, **Then** it exposes Divisions, their Districts, and each District's Upazilas with stable identifiers and parent relationships.
2. **Given** an Admin creates or updates a District or Upazila, **When** its parent does not exist or the supplied parent is invalid, **Then** the change is rejected and the existing hierarchy is unchanged.
3. **Given** an Admin attempts to delete a Division or District with child records, **When** the request is processed, **Then** the operation is rejected without cascading deletion or orphaning children; an unreferenced geographic record may be deleted.
4. **Given** a user supplies geographic values from different branches of the hierarchy, **When** the parent-child relationship is validated, **Then** the invalid combination is rejected.
5. **Given** the approved geographic source is unavailable or cannot be verified, **When** initial data is prepared, **Then** the portal does not invent records or require manual entry of the entire national hierarchy.
6. **Given** an Admin submits the owner-approved geographic JSON through the protected import operation, **When** the full source, names, and parent hierarchy validate, **Then** the complete seed is imported atomically; invalid input writes no geographic records.

### User Story 4 - Protect Configuration Changes and References (Priority: P1)

As an institutional administrator, I need configuration changes authorized, audited, and protected against invalid references so that shared academic settings remain trustworthy and historical use remains understandable.

**Why this priority**: Incorrect, unauthorized, or unaudited reference-data changes can affect multiple future academic workflows.

**Independent Test**: Attempt authorized and unauthorized writes through the UI and direct API; perform valid, duplicate, and referentially invalid operations; verify audit history, data integrity, and inactive-record behavior.

**Acceptance Scenarios**:

1. **Given** an Admin successfully creates, updates, deactivates, or deletes a configuration record, **When** the change commits, **Then** the existing audit system records the actor, action, entity type, record identifier, timestamp, and approved change summary.
2. **Given** a configuration write requires a business audit event, **When** audit persistence fails, **Then** the configuration change does not commit.
3. **Given** an authenticated session is not actively using the Admin role, **When** it attempts a configuration write, **Then** the server denies the request even if the account has an Admin assignment in another inactive role.
4. **Given** a referenced Degree, Semester, Division, or District is deactivated, **When** existing batches or geographic children are viewed or new assignments are attempted, **Then** existing references remain identifiable and unchanged, new assignments to the inactive record are blocked, and deactivation does not cascade to dependents.

## Edge Cases

- A name differs only by surrounding whitespace, letter case, or repeated internal spaces; normalization prevents inappropriate duplicates within that catalog.
- Two Sessions have the same normalized label; reject the duplicate globally. Multiple Batches may reference the same Session.
- Two batches use the same normalized name for the same Degree, even if they reference different Sessions; reject the duplicate because uniqueness is scoped to Degree, not Session.
- An optional code is omitted, blank, or differs only by case from an existing code; blank is treated as absent and normalized non-empty codes follow the catalog's uniqueness constraint.
- A Degree is created or updated with a missing, zero, negative, or non-numeric "Credits to be completed" value; the change is rejected with a field-level explanation.
- Two catalog types contain the same name, such as an examination type and an IIT degree; they remain independent and do not conflict across catalog types.
- A referenced Degree or starting Semester is deactivated; existing Batch references and display remain intact, new Batch assignments to it are blocked, and no Batch is cascade-deactivated.
- A Division or District with children is deactivated; existing geographic parent-child links remain intact, new children or assignments to the inactive parent are blocked, and no descendant is cascade-deactivated.
- A Division has Districts or a District has Upazilas; deleting one cannot cascade and must not leave a live child pointing at a deleted parent.
- A geographic data load contains duplicate, missing-parent, or unknown records; loading reports the invalid entries and does not silently create partial or invented hierarchy data.
- An Admin submits an invalid parent ID or a District/Upazila whose parent is from another hierarchy branch; reject without changing persisted relationships.
- A user changes the active role away from Admin while on a configuration page; subsequent writes are denied by the server and the UI must not preserve Admin write access.
- Audit storage is unavailable during a required configuration change; the change is rolled back and the failure follows the existing audit operational policy.
- Inactive records are displayed in existing-reference views with an inactive indication but are omitted from selectors for new assignments unless an owning feature explicitly requires historical selection.

## Requirements *(mandatory)*

### Business Rules

- The feature maintains only reusable reference/configuration data; it does not create or edit student histories, student profiles, teacher profiles, enrollments, or academic-operation records.
- Only a session actively acting in the Admin role may create, update, deactivate, or delete configuration records. The server checks the current session role on every request; assigned roles, Angular navigation, or a client-supplied role do not grant permission.
- The Configuration menu entry and management screens MUST appear only within the Admin module. Hiding the entry is not an authorization boundary; the server MUST protect every management endpoint.
- Institution type is a fixed controlled value: School, College, or University. No separate institution-type management screen is part of this feature.
- Educational institutions, education boards, education groups, examination types, IIT degrees/programs, semesters, designations, and halls are independent catalogs except for the explicitly stated Batch and geographic relationships.
- SSC/O-Level, HSC/A-Level, Bachelor, Master, PGDIT, and Others are previous-education examination/qualification categories, not IIT degree records.
- Academic Sessions are an independent catalog with a globally unique normalized label; they are not authentication sessions and do not represent semester progression.
- A Batch references exactly one IIT degree/program, exactly one starting semester, and exactly one Academic Session. Degree, Semester, and Academic Session remain independently maintainable; this does not implement semester progression.
- Each Degree MUST have a required positive numeric "Credits to be completed" total. Student Management uses this degree requirement with passed course results to determine graduation eligibility; Configuration stores the required total but does not calculate a student's completed credits.
- Geographic references form Division → District → Upazila. They are independent of student profiles and institution addresses.
- Names are compared after trimming and case normalization within their own catalog. Optional non-empty codes are normalized and unique within the corresponding academic catalog. Geographic Division names are unique globally; District and Upazila names are unique within their parent. Geographic records do not have codes. Batch names are unique within their Degree; a different Degree may reuse the same name.
- When a referenced Degree, Semester, Academic Session, Division, or District is deactivated, preserve existing references and their display, block new assignments to it, and do not cascade deactivation to dependents.
- Successful configuration changes use the existing audit system and its current Admin-session authorization. Mandatory change audit is committed atomically with the configuration change; validation errors are not treated as committed business changes.
- Admins may delete an unreferenced configuration record. A delete is soft: the record is marked deleted, hidden everywhere, and kept in the database, and its name and code are free to use again. If any live record depends on it, deletion MUST be rejected without cascading and the Admin MUST be able to deactivate it instead.

### Functional Requirements

#### Catalog maintenance

- **FR-001**: The portal MUST provide authorized Admins with create, view, update, list, search, filter, and deactivate operations for Educational Institutions. Records contain a required name and type (School, College, University), optional code and address, and active/inactive status.
- **FR-002**: The portal MUST provide an independent Education Boards catalog with required board name, optional code, and active/inactive status; it MUST NOT associate a board with a student or institution in this feature.
- **FR-003**: The portal MUST provide independent Education Groups with required name and active/inactive status for values such as Science, Humanities, and Business Studies.
- **FR-004**: The portal MUST provide independent Examination Types for previous-education categories, including SSC/O-Level, HSC/A-Level, Bachelor, Master, PGDIT, and Others. These records MUST NOT be treated as IIT degrees or contain student academic-history fields.
- **FR-005**: The portal MUST provide independent Degrees for programs offered by IIT, with required name, optional code, a required positive numeric "Credits to be completed" total, and active/inactive status. Degree records MUST NOT depend on a batch or semester. Student graduation-eligibility workflows use this total as the degree's required credits; Configuration does not calculate a student's completed credits.
- **FR-006**: The portal MUST provide independent Semesters with a required name/label, active/inactive status, and an ordering value when required by the existing domain. A Semester MUST NOT depend on a degree, batch, or progression workflow.
- **FR-007**: The portal MUST provide independent Designations with required name, optional code, and active/inactive status; assignment to a Teacher remains out of scope.
- **FR-008**: For each catalog, the portal MUST validate required values and reject duplicate normalized names and duplicate normalized non-empty codes within that catalog. Identical names in different catalogs are allowed.

#### Batches and geographic hierarchy

- **FR-009**: The portal MUST maintain Batches with a required name, exactly one existing Degree, exactly one existing starting Semester, exactly one Academic Session reference, and active/inactive status. The former free-text admission session is replaced by this required reference.
- **FR-010**: Batch creation/update MUST reject missing, nonexistent, or inactive Degree, Semester, or Session references, and reject duplicate normalized names within the same Degree. The same name MAY be used by a Batch under a different Degree; Session does not change this uniqueness scope.
- **FR-011**: Degrees, Semesters, and Academic Sessions MUST remain independently maintainable when no Batch references them; batch creation MUST NOT assign students or advance a semester.
- **FR-012**: The portal MUST maintain Divisions with a required name, Districts with a required name referencing exactly one Division, and Upazilas with a required name referencing exactly one District. Geographic records do not have codes.
- **FR-013**: Geographic validation MUST enforce parent-child referential integrity and normalized-name uniqueness: Division names globally; District names within the selected Division; Upazila names within the selected District.
- **FR-014**: Geographic data MUST be loaded from an owner-provided JSON seed file that cites the approved official source and version and is accompanied by a validation report. The import MUST be initiated through an endpoint authorized for the active Admin role, validate the full hierarchy before writing, and commit all rows atomically. The portal MUST NOT invent geography records or require administrators to enter the entire hierarchy manually.
- **FR-015**: The portal MUST expose data relationships sufficient for future cascading Division → District → Upazila selection without coupling the catalog to student profiles or institution addresses.

#### Lifecycle, authorization, and audit

- **FR-016**: The Admin interface MUST support soft deletion of an unreferenced configuration record: the record is marked deleted with the time and the account that deleted it, is excluded from every list, lookup, and selector, can no longer be read or changed, and is not restorable from the portal. Normalized name and code uniqueness apply to records that are not deleted. If a record has dependent records, deletion MUST be rejected with a clear explanation and no cascade or orphaning; the interface MUST offer deactivation as the safe alternative.
- **FR-017**: Inactive records MUST be excluded from new assignment selectors and remain identifiable in existing-reference views. Deactivating a referenced Degree, Semester, Academic Session, Division, or District MUST preserve existing links, block new assignments to that record, and MUST NOT cascade deactivation to dependents.
- **FR-018**: Every configuration create, update, deactivate, and delete operation MUST be authorized server-side against the current session's active Admin role, regardless of UI visibility or client-submitted role information.
- **FR-019**: Successful configuration create, update, deactivate, and delete operations MUST use the existing audit system and record actor, action, entity type, record identifier, UTC timestamp, outcome, and allowlisted change summary. The event and configuration change MUST commit atomically where the existing audit contract requires it.
- **FR-020**: Configuration APIs MUST return safe validation errors for missing/invalid fields, duplicate academic-catalog names/codes, duplicate geographic names within their required scope, nonexistent references, and invalid parent-child combinations; rejected validation attempts MUST NOT change stored data.
- **FR-021**: Administrative list/search experiences MUST show loading, empty, error, and inactive states; support search/filter and infinite loading in bounded batches where catalog volume warrants it; and follow the shared design system and accessibility rules. They MUST NOT show numbered page controls or an items-per-page selector.
- **FR-022**: The frontend MUST provide a “Configuration” menu entry and management route under the Admin module (`/admin/configuration`). The entry MUST be visible only when the session's active role is Admin; direct navigation by other roles MUST be denied.
- **FR-023**: The portal MUST provide Admins with create, view, update, list, search, filter, deactivate, and delete-if-unreferenced operations for Academic Sessions. Each Session MUST have a required label that is unique after normalization across the Session catalog; Batches may share a Session, while new Batch assignments to an inactive Session are rejected.
- **FR-024**: The portal MUST provide independent Halls (university residential halls) with a required name, an optional code, an optional address, and active/inactive status, maintained with the same operations, normalized name and code uniqueness, deactivation, delete-if-unreferenced, authorization, and audit rules as the other independent catalogs. A Hall MUST NOT be associated with a student, teacher, or other catalog in this feature.
- **FR-025**: Every configuration record MUST record who created it and when, and who last modified it and when. The values MUST come from the server clock and the authenticated session and MUST NOT be accepted from a client; on creation the last-modified values equal the created values, and any later change, including deactivation and deletion, updates only the last-modified values. The account is the stable account identifier, and is empty when no signed-in account made the change (a background process or a seed script). The same rule MUST apply to every future business entity through the shared base class described in the service README.

#### Acceptance Criteria

- **AC-001 - Independent catalogs**: Each catalog in FR-001–FR-008 can be maintained independently; validated names/codes reject duplicates within its catalog, while similar names in another catalog remain allowed. Degree create/update requires a positive numeric "Credits to be completed" value and rejects missing, zero, negative, or non-numeric values. (FR-001–FR-008)
- **AC-002 - Batch integrity**: Every Batch has a name and references exactly one valid Degree, one valid starting Semester, and one active Session; missing or invalid references and duplicate normalized names within the same Degree are rejected, the same name is accepted for a different Degree regardless of Session, and Degree/Semester/Session remain independently maintainable. (FR-005–FR-006, FR-009–FR-011)
- **AC-003 - Geographic integrity and seed**: Division, District, and Upazila data follow the confirmed source and exact parent relationships; invalid/cross-branch relationships and duplicate normalized names at the specified global/parent scope are rejected. Geographic records have no codes. (FR-012–FR-015)
- **AC-004 - Lifecycle safety**: Deactivation of a referenced Degree, Semester, Academic Session, Division, or District preserves existing links and display, blocks new assignments to the inactive record, and does not cascade deactivation. Physical deletion never cascades away or orphans dependents. (FR-016–FR-017)
- **AC-005 - Authorization and audit**: 100% of unauthorized direct API create/update/deactivate/delete requests are denied; successful create/update/deactivate/delete operations produce the required audit event atomically, and failed validation does not change stored data. (FR-018–FR-020)
- **AC-006 - Administration UX**: Admins can find and maintain catalogs using accessible, responsive list/form states consistent with the project design system. (FR-021)
- **AC-007 - Admin module navigation**: The Configuration menu entry appears in the Admin module only; non-Admin active roles cannot use the route or management APIs directly. (FR-018, FR-022)
- **AC-008 - Academic Sessions**: Session labels are globally unique after normalization; each Batch references exactly one Session, a Session can be shared by multiple Batches, and inactive Sessions cannot be assigned to new Batches. (FR-009–FR-011, FR-023)

## Key Entities *(include if feature involves data)*

- **Educational Institution**: Independent reference for a School, College, or University; required name/type, optional code/address, status.
- **Education Board**: Independent examination authority; required name, optional code, status.
- **Education Group**: Independent school/college academic group; required name and status.
- **Examination Type**: Independent previous-education qualification category; name and status; not an IIT degree or student-history record.
- **Degree**: Independent IIT-offered degree/program; name, optional code, required positive numeric "Credits to be completed" total, and status.
- **Semester**: Independent semester label and optional domain-required order; status.
- **Designation**: Independent teacher designation catalog; name, optional code, status.
- **Hall**: Independent university residential hall catalog; required name, optional code, optional address, status.
- **Academic Session**: Independent catalog entry with a required label, normalized label, and status; globally unique label and may be referenced by multiple Batches, each of which requires one Session.
- **Batch**: Cohort with one Degree reference, one starting Semester reference, one Academic Session reference, name, and status.
- **Division**: Bangladesh administrative top-level geographic reference with name/status and no code.
- **District**: Geographic reference with one Division parent and name/status; no code.
- **Upazila**: Geographic reference with one District parent and name/status; no code.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of tested duplicate normalized names/codes within a catalog are rejected, and the same name in separate catalogs is not incorrectly rejected.
- **SC-002**: 100% of missing, invalid, or nonexistent Batch Degree, starting-Semester, or Session references and invalid geographic parent-child relationships are rejected without modifying stored data.
- **SC-003**: 100% of configuration writes by sessions not actively in the Admin role are denied at the server boundary.
- **SC-004**: 100% of successful configuration create/update/deactivate operations produce the required audit record atomically; failed validation leaves both configuration and audit history unchanged except for existing security-event rules.
- **SC-005**: 100% of loaded geographic records match the approved source/version and pass parent-reference validation before being made selectable.
- **SC-006**: At least 95% of representative Admin participants can find and update or deactivate a specified catalog record within two minutes without assistance.
- **SC-007**: 100% of non-Admin sessions cannot see or directly access the Configuration management route; unreferenced records can be deleted by Admin, while referenced-record deletion is rejected without cascading.
- **SC-008**: 100% of Degree create/update attempts without a positive numeric "Credits to be completed" value are rejected without changing the Degree record.

## Assumptions

- The existing active Admin role and server-side policy authorize configuration management; no new roles or authentication changes are introduced.
- Fixed institution types are School, College, and University; no separate type-management UI is required.
- Catalog names are trimmed and compared case-insensitively within their entity type. Optional non-empty codes are likewise normalized and unique within their academic catalog. Geographic records have no codes; Division names are unique globally, while District and Upazila names are unique within their parent.
- Academic Session labels are unique globally after normalization; every Batch references one Session. Batch names are unique per Degree, not globally and not per Session.
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
