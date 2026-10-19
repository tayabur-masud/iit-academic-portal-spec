# Feature Specification: Course Management

**Feature Branch**: `feature/008-course-management`

**Created**: 2026-10-19

**Status**: Implemented; manual quickstart scenarios and the SC-006 usability study are not yet run (tasks.md T016, T017)

**Input**: User description: "Manage courses. A course has a name, code, credit hour, optional theory/lab credit-hour breakdown, an optional prerequisite course, a Degree, a mandatory/elective type, and standard audit history."

## Clarifications

- Course codes are unique across the entire course catalog, not just within a Degree.
- Theory and lab credit-hour values are independently optional. When both are supplied, their sum must equal the course's total credit hours.

### Session 2026-10-19

- Q: What should happen when an Admin tries to deactivate a course that is currently used as a prerequisite? → A: Reject deactivation until all active courses stop referencing it as a prerequisite.
- Q: What should happen if changing a Course's Degree would make a prerequisite relationship cross-Degree? → A: Reject the change without modifying any affected Course.
- Q: May one supplied theory or lab credit value exceed the Course's total credit hours? → A: No; each supplied component must be between zero and the total, inclusive.
- Q: May two Courses share a name when their codes differ? → A: Yes; Course names are not unique, while codes remain globally unique.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Maintain the Course Catalog (Priority: P1)

As an active Admin, I need to maintain courses under their Degree programs so academic offerings have consistent names, codes, credit hours, and prerequisite information.

**Why this priority**: Course definitions are the shared academic catalog needed before later enrollment, teaching, and result workflows can refer to consistent courses.

**Independent Test**: As an active Admin, create, view, update, search, and deactivate courses in two Degrees. Verify required course details, type, optional credit breakdown and prerequisite, global code uniqueness, audit history, and rejection of invalid Degree/prerequisite relationships without changing stored data.

**Acceptance Scenarios**:

1. **Given** an active Admin enters a course, **When** the name, globally unique code, positive credit-hour total, Degree, and mandatory/elective type are valid, **Then** one course is created under that Degree and its creation is recorded in audit history; another course may use the same name if its code differs.
2. **Given** a course has either or both theory/lab credit-hour values, **When** the Admin saves the course, **Then** each supplied value is between zero and the total credit hours, inclusive; if both are supplied, their sum equals the total.
3. **Given** an Admin selects a prerequisite, **When** the prerequisite is valid for the course's Degree, **Then** the course references that existing prerequisite; no prerequisite is required when the field is empty.
4. **Given** a course has a prerequisite, **When** an Admin attempts to set itself or create a prerequisite cycle, **Then** the change is rejected without changing the course.
5. **Given** an Admin submits a course code already used anywhere in the catalog or invalid credit values, **When** the course is saved, **Then** a field-level validation error is returned and no data is changed.
6. **Given** an active Admin updates or deactivates a course, **When** the change commits, **Then** the course and its audit history are updated without changing enrollments, course results, or other academic records.
7. **Given** an anonymous or non-Admin user requests course-management data or changes, **When** the server authorizes the request, **Then** access is denied without disclosing protected data.
8. **Given** one or more active courses reference a course as a prerequisite, **When** an Admin attempts to deactivate that prerequisite course, **Then** deactivation is rejected without changing either the prerequisite course or its dependent courses.
9. **Given** changing a course's Degree would make its own prerequisite or an active dependent course's prerequisite cross-Degree, **When** an Admin saves the change, **Then** it is rejected without changing any affected course.

### Edge Cases

- A course has zero, negative, or over-precision total credit hours.
- Only one component of an optional theory/lab breakdown is supplied; this is valid without requiring the other component.
- Both theory and lab components are supplied but do not sum to the course's total credit hours.
- A single supplied theory or lab component is negative or exceeds the course's total credit hours.
- A prerequisite is missing, inactive, belongs to another Degree, refers to itself, or would create a cycle.
- Deactivation of a course referenced by an active course as its prerequisite is rejected without changing either course.
- Two Degrees attempt to use the same course code; the duplicate is rejected across the catalog.
- An Admin changes a course's Degree and would invalidate its own or an active dependent course's prerequisite relationship; the change is rejected without modifying affected courses.
- Audit persistence fails during a course create, update, or deactivation.
- An unauthorized user attempts to create, view, update, or deactivate a course.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The portal MUST provide active Admins with create, view, list, search, update, and deactivate operations for courses.
- **FR-002**: Each course MUST have a required name, required code, required positive credit-hour total, exactly one Degree, and a required type of Mandatory or Elective. Course names MUST NOT be unique-constrained; codes remain globally unique.
- **FR-003**: A course MAY have independently optional theory and lab credit-hour components. Each supplied component MUST be between zero and the total credit-hour value, inclusive, and have no more than two decimal places. If both components are supplied, their sum MUST equal the total credit-hour value.
- **FR-004**: A course MAY reference one prerequisite course; the prerequisite MUST be a different active course in the same Degree, and the prerequisite graph MUST NOT contain a cycle.
- **FR-005**: Course codes MUST be unique across the entire course catalog. Validation errors MUST be field-keyed and rejected requests MUST NOT change stored course data.
- **FR-006**: Every course MUST record created/modified actor and time using the portal's standard audit history. Successful mutations MUST be audited, and an audit failure MUST roll back the course change.
- **FR-007**: Only a session actively using the Admin role MUST be authorized to view or change course-management data; authorization MUST be enforced by the server on every request.
- **FR-008**: Course Management MUST maintain course definitions only; it MUST NOT create or change course registrations, enrollments, teaching assignments, grades, or result records.
- **FR-009**: A course's Degree and prerequisite relationships MUST remain valid. A course MUST NOT be assigned a prerequisite from another Degree, prerequisite cycles MUST be rejected, a Degree change that would invalidate its own or an active dependent course's prerequisite MUST be rejected without modifying affected courses, and a course MUST NOT be deactivated while any active course references it as a prerequisite.

### Key Entities *(include if feature involves data)*

- **Course**: A Degree-owned catalog item with name, code, total credit hours, optional theory/lab breakdown, optional prerequisite, Mandatory/Elective type, active status, and standard created/modified audit details.
- **Degree**: The existing academic program that owns the course.
- **Prerequisite course**: An optional relationship from one course to another course in the same Degree.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of active courses have a name, code, positive credit-hour total, Degree, and Mandatory/Elective type.
- **SC-002**: 100% of duplicate course codes across the catalog are rejected without changing an existing course.
- **SC-003**: 100% of supplied theory/lab components are non-negative and no greater than the total; when both components are supplied, they sum to the total, while a single supplied component remains valid.
- **SC-004**: 100% of missing, cross-Degree, self-referencing, cyclic, or Degree-change-invalidated prerequisites and attempts to deactivate a course referenced by an active dependent are rejected without changing affected course data.
- **SC-005**: 100% of unauthorized course-management requests are denied by the server.
- **SC-006**: At least 9 of 10 active Admin participants who have not previously used Course Management can create a valid course within three minutes without assistance.
- **SC-007**: 100% of successful course changes have audit history, and no course-management operation changes enrollment or result records.

## Assumptions

- Course Management is for active Admins and follows existing course-catalog UI, audit, authorization, validation, and inactive-record conventions.
- Each Course belongs to exactly one existing Degree. A prerequisite, when supplied, must be active and belong to the same Degree; self-prerequisites and cycles are invalid.
- Course type is the fixed two-value set Mandatory and Elective.
- Total credit hours are positive and may use up to two fractional decimal places. Theory and lab components are independently optional, and when both are supplied they sum to the total.
- Deactivation, rather than deletion, preserves references from future enrollment, teaching, and result workflows.
- Enrollment, course registration, teaching assignments, result entry, grading, and graduation calculations are outside this feature.