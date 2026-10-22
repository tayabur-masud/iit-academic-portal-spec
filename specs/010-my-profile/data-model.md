# Data Model: My Profile

**Feature**: [My Profile](spec.md)  
**Status**: Read-only projection over existing account and role-profile data. No entity or schema change is introduced.

## Shared View

One response represents the authenticated account's profile for the session's active role. The account ID and active role are taken from validated session claims; neither is chosen by the caller.

| Value | Source | Display rule |
|---|---|---|
| Account email | Portal account | Current email belonging to the signed-in account. |
| Active role | Validated session | One of Admin, Student, Teacher, or Coordinator. Determines the profile variant. |
| Profile | Existing role-specific record | Only the profile linked to the signed-in account and active role; absent profile is represented as unavailable. |

Internal account/profile IDs, role assignments not active in this session, account status, audit fields, credentials, setup proofs, and other users' data are not part of the response.

## Role Profile Mapping

### Admin Staff Profile

- Name.
- Mobile number.
- Current Designation name.

The profile is read from Staff Management's existing StaffProfile. Profile active state and audit metadata are not displayed.

### Student Profile

All populated personal details already stored on the Student profile are visible to the same signed-in Student:

- Applicant name; father and mother names.
- Date and place of birth; marital status; nationality.
- Applicant mobile number.
- Mailing and permanent addresses, including address details, cell number, and resolved locality/geographic names (Upazila or City Corporation, District, Division).
- Guardian name, relationship, address, and cell number.
- Prior academic history, ordered as entered: category, examination, school/college, board, group, department, university, subject, institution, year, marks obtained, marks percentage, CGPA, class, and division.

Do not return profile IDs, user IDs, history row IDs, catalog foreign-key IDs, audit metadata, account status, admission summaries, registration/admission identifiers, current academic results, credentials, or setup proofs. Optional values remain absent/null when not supplied; do not fabricate values.

### Teacher Profile

- Display name.
- Employee staff identifier when present.
- Phone when present.
- Current Designation name when assigned.

The profile is read from the existing TeacherProfile. Profile active state, account status, roles, and audit metadata are not displayed.

### Coordinator Role

Coordinator is an additional role associated with a teacher account, not a separate profile entity. The Coordinator view uses that account's TeacherProfile fields. Coordinator assignment/batch details are not part of My Profile.

## Relationships and Read Boundaries

- PortalUser has at most one StaffProfile, StudentProfile, and TeacherProfile through the existing account relationships.
- The active role selects exactly one profile variant: Admin → StaffProfile; Student → StudentProfile; Teacher → TeacherProfile; Coordinator → TeacherProfile.
- A multi-role account can view a different role profile after switching its active role, but the account identity is unchanged.
- All reads are scoped to the authenticated account. No caller-supplied account/profile identifier is accepted.
- A missing role profile returns an unavailable state rather than a profile for another role or account.
- Student prior-education rows are part of the Student profile and are included; Student admissions and current academic results are separate records and excluded.

## Validation and Lifecycle

- This feature performs no writes, validation of profile changes, account lifecycle transitions, or audit recording.
- Existing management workflows continue to own profile edits and role/account changes.
- No database migration, backfill, or new stored profile value is required.
