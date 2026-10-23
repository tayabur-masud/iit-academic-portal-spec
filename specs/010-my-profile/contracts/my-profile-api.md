# My Profile API Contract

**Status**: Implemented (`MyProfileController`, `MyProfileContracts.cs` in the service). A read-only self-profile view across all supported active portal roles.

## Security and Common Behavior

- Every request requires the existing authenticated portal session.
- The server derives the account ID and active role from validated session claims. The caller cannot supply a target account ID, profile ID, or role selector.
- Only the signed-in account's profile for the active role is returned. An account with multiple roles sees the profile for the role currently active in the session.
- The API returns a purpose-built allowlist, not a Staff, Teacher, or Student management response. It never returns credentials, credential-setup proofs, internal IDs, audit metadata, account status, unrelated role profiles, admissions, or current academic results.
- Authenticated responses use the existing `no-store` cache policy.
- No write route is defined. Profile changes remain in the owning role-management workflow.

## Route

| Method and route | Behavior |
|---|---|
| `GET /api/profile` | Return the authenticated account email, active role, and the matching role-profile view. No query or body is accepted. |

## Response Shape

The response has `email`, `activeRole`, and `profile`. `profile` is a role-discriminated object whose `kind` names the variant, and is `null` when the signed-in account has no profile linked to the active role. Optional values are `null` when not recorded; nothing is invented.

| Active role | `kind` | Fields |
|---|---|---|
| Admin | `staff` | `name`, `mobileNumber`, `designationName` |
| Student | `student` | `applicantName`, `fatherName`, `motherName`, `dateOfBirth` (`yyyy-MM-dd`), `placeOfBirth`, `maritalStatus`, `nationality`, `applicantMobile`, `mailingAddress`, `permanentAddress`, `guardianName`, `guardianRelationship`, `guardianAddress`, `guardianCellNumber`, `academicHistory` |
| Teacher | `teacher` | `displayName`, `employeeStaffId`, `phone`, `designationName` |
| Coordinator | `teacher` | The same account's Teacher profile; no separate Coordinator profile or assignment list. |

- An address (`mailingAddress`, `permanentAddress`) is `details`, `cellNumber`, `upazilaName`, `cityCorporationName`, `districtName`, and `divisionName`; one of the Upazila and City Corporation names is `null`. An address with nothing recorded is `null`.
- `academicHistory` is in the order entered. Each row is `category` (`SscOLevel`, `HscALevel`, `BachelorGraduation`, `Master`, `Pgdit`, or `Other`), `examination`, `schoolOrCollege`, `board`, `group`, `department`, `university`, `subject`, `institution`, `year`, `marksObtained`, `marksPercentage`, `cgpa`, `class`, and `division`, with the stored names (catalog choices resolved to names, not IDs).
- `designationName` is the Designation's current name, also when the Designation was deleted after it was assigned.
- Student responses omit all IDs and catalog foreign keys. Admissions (degree, batch, rolls, hall, registration number) and current academic results are not returned.

Example (Admin):

```json
{ "email": "admin@iit.test", "activeRole": "Admin",
  "profile": { "kind": "staff", "name": "Mohammad Selim Mia", "mobileNumber": "01716770714", "designationName": "Administrative Officer" } }
```

When `profile` is `null`, `email` and `activeRole` still identify the signed-in user's view context; no other account's details are used as fallback.

## Account name in the session response

Every session response (`POST /api/auth/sessions`, `GET /api/auth/sessions/current`, and `PUT /api/auth/sessions/current/active-role`) also carries `displayName` (see `specs/001-role-based-auth/contracts/authentication.openapi.json`). It is the signed-in account's own name for the account menu:

1. the name on the active role's profile: Staff `name` for Admin, Student `applicantName` for Student, Teacher `displayName` for Teacher and Coordinator (the name My Profile shows);
2. otherwise the name on another of the account's profiles, in the order Staff, Teacher, Student;
3. otherwise the account email.

It is resolved on the server in one read on each of those responses (`MyProfileService.GetDisplayNameAsync`), so it follows a role switch and a profile change, and is never stored separately. It is `null` only if the account no longer exists.

## Errors

- `401`: no authenticated session.
- `403`: the session has no supported active role (problem detail "Choose a portal role to see your profile.").
- `404`: the authenticated account no longer resolves (normally the session is rejected first). Any path below `/api/profile`, such as an account ID, is also `404`.
- `405`: any method other than `GET`; there is no write.
- `5xx`: safe generic problem response for unexpected failures; no query, account, or infrastructure details.
- Query parameters (for example `userId` or `role`) are ignored and cannot select another account or role.

A missing role profile is not an authorization failure: return the safe unavailable profile state for the signed-in account.

## Privacy and Role Mapping

- Account ID is the `NameIdentifier` from the validated session; active role is the session's role claim.
- Admin maps to StaffProfile; Student maps to StudentProfile; Teacher and Coordinator map to TeacherProfile.
- My Profile is a view of the active role only. The account menu may switch the active role; a subsequent profile read uses the newly active role.
- Do not expose StudentAdmissionSummary or audit/account-management fields from existing management DTOs.
