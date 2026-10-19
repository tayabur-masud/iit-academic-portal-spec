# Student Management API Contract

**Status**: Implemented contract for [spec.md](../spec.md). See [Implemented details](#implemented-details).

## Security and Common Behavior

- All student-admission management routes require an authenticated session actively using the Admin role. The server checks this on every request; state-changing requests also use the existing anti-forgery protection.
- Request and response DTOs are explicit. No password, credential-setup proof/link, or scanned document is accepted or returned.
- Validation failures return field-keyed problem details and do not partially create or change the Student profile, admission, linked account, or profile-level academic-history entries.
- Admission and linked account creation, including required audit events, commit together. The existing credential-setup notification is requested only after commit. Delivery failure leaves the account pending and is handled by existing Admin reissue behavior.

## Routes

| Method and route | Behavior |
|---|---|
| `GET /api/admin/students` | Return a bounded, searchable, paginated directory of program admissions with safe Student profile summaries. A returning Student may appear once per admission. |
| `POST /api/admin/students` | Create one program admission. If normalized email has no account, create one Student account, profile, and admission atomically, then request setup email after commit. If it identifies an existing Student account/profile, reuse them and create only a distinct admission; return whether the account was created and whether setup email was requested. Never return setup proof. |
| `GET /api/admin/students/{admissionId}` | Return one program admission, its shared Student profile, existing Student account status, profile-level academic-history entries, and the profile's other admissions. |
| `PUT /api/admin/students/{admissionId}` | Update shared profile fields, profile-level academic-history entries, and this admission's writable fields. Shared profile/history changes are visible through every admission. Hall and registration number belong to this admission and may be changed independently by retaining the other current value. |
| `GET /api/admin/students/{admissionId}/graduation-eligibility` | Return a computed eligibility result for this program admission and its unmet or unavailable requirements. This read does not block academic-result calculation or publication. |

No delete route is defined by this feature.

## Writable Admission Fields

- Required on admission submission: applicant email, applicant mobile, Degree identifier, Batch identifier, and the source-form fields required by the specification. Email is canonical on `PortalUser`; mobile and shared personal/contact fields belong to StudentProfile.
- Optional at initial admission: assigned Hall, registration number, class roll, guardian data, and source-form values not supplied.
- Academic history: zero or more StudentProfile-level entries shared across admissions. `category` is one of SSC/O-Level, HSC/A-Level, Bachelor/Graduation, Master, PGDIT, or Other. `schoolOrCollegeId`, `universityId`, `institutionId`, `boardId`, and `groupId` optionally name a Configuration record; when one is sent, the service stores it and keeps the record's name as the snapshot in `schoolOrCollege`, `university`, `institution`, `board`, or `group` (a row that keeps the same record keeps its saved snapshot). Without an ID the typed name is kept as entered. The other fields (`examination`, department, subject, year, marks, percentage, CGPA, class, division) are optional strings preserved as entered.
- Mailing and permanent addresses are separate profile-level objects. Each request object contains `details` (one free-text address-details value) and `upazilaId`. Each response object also includes the resolved Upazila, District, and Division identifiers/names from Configuration. When an address is supplied, both details and Upazila are required; clients do not submit separate District/Division IDs.
- If normalized email identifies an existing Student account/profile, reuse it without changing credentials/status or requesting setup again. If it belongs to a non-Student account, reject with 409 and do not merge accounts. A Student account with no profile may receive a profile in the admission transaction.
- Degree and Batch must be active and the Batch must belong to the selected Degree. A newly assigned Hall must identify an active Hall.
- Each supplied address Upazila must exist in the configured geographic hierarchy and resolve to its District and Division; the server returns field-keyed errors for missing or unavailable Upazilas.
- A profile may have only one admission for the same Degree and Batch. Registration number is unique across all program admissions when present.
- Each previously recorded IIT program appears as its own admission and is not duplicated as academic history. Unrecorded prior education is entered once on StudentProfile and appears across admissions. No credits transfer across admissions unless separately approved.

## Graduation Eligibility Response

The response identifies whether the student is eligible for the requested admission and the unmet requirements. It includes that admission's Degree required-credit total and completed-credit total only when the passed-course results provider supplies it. Missing provider data is explicitly identified and never interpreted as eligibility. Credits from another admission are not carried over. The concrete result-provider transport is outside this API contract; the Application layer consumes a read-only provider boundary keyed by admission ID.

## Error Behavior

- `400`: missing/invalid fields, inconsistent Degree/Batch relationship, or field validation errors.
- `401` / `403`: anonymous or non-Admin access.
- `404`: the requested student admission does not exist.
- `400`: a submitted Degree, Batch, or Hall identifier is missing, inactive, nonexistent, or inconsistent with its required relationship; errors are keyed to the corresponding field.
- `409`: email belongs to a non-Student account, duplicate Student/Degree/Batch admission, duplicate registration number, or a concurrent uniqueness conflict.

## Implemented details

- **Lookup route (added).** `GET /api/admin/students/lookup?email=` returns `status` (`New`, `ExistingStudent`, or `NotStudentAccount`)
  and, for an existing Student with a profile, `profile` (shared fields, history with ids, and every admission) so the form can prefill a
  returning Student. An invalid email is a field-keyed 400. Admin only, like every route here.
- **Create response.** `201` with `admission`, `accountCreated`, `profileCreated`, and `setupEmailRequested` (true only when a new account
  was created and its setup email was durably requested). Never a link or proof.
- **Admission response.** `id`, `profile` (`id`, `userId`, `email`, `accountStatus`, personal fields, `mailingAddress` and
  `permanentAddress` with `details`, `cellNumber`, `upazilaId`/`upazilaName`, `districtId`/`districtName`, `divisionId`/`divisionName`,
  guardian fields, `academicHistory`, `admissions`), `otherAdmissions`, `degreeId`/`degreeName`, `batchId`/`batchName`,
  `admissionSerialNumber`, `admissionExamRoll`, `classRoll`, `assignedHallId`/`assignedHallName`, `registrationNumber`, and the admission's
  created/last-modified facts. Absent optional values are omitted.
- **Request body.** The same fields for create and update; create also takes `email`. `dateOfBirth` is `yyyy-MM-dd`. `academicHistory`
  is the complete list (entries with `id` are kept, others added, missing ones removed); `category` is `SscOLevel`, `HscALevel`,
  `BachelorGraduation`, `Master`, `Pgdit`, or `Other`. Unknown members (for example a photo, attachment, signature, or password) are a 400.
- **Required fields.** `email` (create), `applicantName`, `applicantMobile`, `degreeId`, `batchId`, `admissionSerialNumber`, and
  `admissionExamRoll`. Each supplied address needs `details` and `upazilaId`.
- **Error keys.** Field-keyed 400s use the camelCase path, for example `mailingAddress.upazilaId` or `academicHistory[2].category`.
  A non-Student email, a duplicate Student/Degree/Batch admission, and a duplicate registration number (compared case- and
  space-insensitively, across every admission) are 409 with a `detail` message.
- **Directory.** `search` (applicant name, email, mobile, exam roll, class roll, or registration number), `degreeId`, `batchId`, `page`,
  `pageSize` (1-100, default 25); ordered by applicant name then admission ID; response `items` (each with `admissionExamRoll` and, when
  recorded, `classRoll`), `page`, `totalCount`, `totalPages`.
- **Eligibility.** `admissionId`, `eligible`, `requiredCredits` and `completedCredits` (omitted when unavailable), and `requirements`:
  `Credits`, `AssignedHall`, `RegistrationNumber`, each `Met`, `NotMet`, or `Unavailable` with a message.
- **Catalog choices in history.** `schoolOrCollegeId` must be a School or College, `universityId` a University, and `institutionId` any institution; a newly chosen record (including a board or group) must exist and be active, otherwise a field-keyed 400 such as `academicHistory[0].universityId`. Responses return each ID with its name snapshot.
- **Configuration.** Deleting a Degree, Batch, Hall, Upazila, educational institution, education board, or education group used by a student is a 409 ("Deactivate it instead").

