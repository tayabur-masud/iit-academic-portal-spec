# Data Model: Student Management

**Feature**: [Student Management](spec.md)

## StudentProfile

The person-level Student record. It is linked one-to-one to a Student portal account and may have multiple program admissions.

| Field | Type | Required | Rules |
|---|---|---:|---|
| Id | UUID | Yes | Stable admission identifier and integration key for the results owner. |
| UserId | Portal user identifier | Yes | Unique foreign key to the linked Student-role account; uniqueness is on the profile, not on an admission. |
| ApplicantMobile | String | Yes | Shared required contact number; not verified by this feature. |
| Personal and contact fields | Typed values | Per source form | Applicant/family names, birth and nationality data, and optional guardian/emergency contacts. |
| MailingAddressDetails | String | No | One free-text house/road/locality detail value; required when a mailing address is supplied. |
| MailingUpazilaId | UUID | No | References an existing Upazila; required with MailingAddressDetails. District and Division are resolved from the selected Upazila. |
| PermanentAddressDetails | String | No | One free-text house/road/locality detail value; required when a permanent address is supplied. |
| PermanentUpazilaId | UUID | No | References an existing Upazila; required with PermanentAddressDetails. District and Division are resolved from the selected Upazila. |
| Address cell numbers | Strings | No | Mailing and permanent address cell numbers remain separate and are retained when present on the form. |
| Audit fields | Existing audited-entity fields | Yes | Server-generated created/modified actor and UTC timestamps. |

The linked `PortalUser` remains the canonical source for applicant email, login name, Student role, and credential-setup status. Email uniqueness is enforced across all portal accounts. Student Management does not copy email into a second writable profile field.

For each supplied address, both its details and Upazila are required. The API/UI displays the selected Upazila together with its parent District and Division from the existing Configuration hierarchy; clients do not submit independent parent IDs that could contradict the Upazila.

## StudentAdmissionRecord

One program enrollment for a Student profile. A profile may have separate admissions for PGDIT, EMIT, or other programs; each admission references its own academic offering and has its own later identifiers and graduation eligibility.

| Field | Type | Required | Rules |
|---|---|---:|---|
| Id | UUID | Yes | Stable identifier and key for passed-course credit lookup for this specific program admission. |
| StudentProfileId | UUID | Yes | Foreign key to a StudentProfile; multiple admissions may reference the same profile. |
| DegreeId | UUID | Yes | References an existing Degree. |
| BatchId | UUID | Yes | References an existing Batch whose DegreeId matches DegreeId. |
| AssignedHallId | UUID | No | References an existing Hall; may be assigned or changed independently later for this admission. |
| RegistrationNumber | String | No | May be assigned later; unique across all admissions when present. |
| AdmissionSerialNumber, AdmissionExamRoll, ClassRoll | Typed values | As on source form | Admission-specific; serial, exam roll, class roll, and registration number remain distinct. |
| Audit fields | Existing audited-entity fields | Yes | Server-generated created/modified actor and UTC timestamps. |

An additional admission for a StudentProfile is a distinct program enrollment. Repeating the same Student/Degree/Batch admission is a duplicate. An existing Student account is reused without changing its credential or status and without another setup email.

## StudentAcademicHistory

Zero or more child entries belong to a StudentProfile and preserve the person's prior-education values supplied on paper forms. They are shared across all program admissions for that profile.

| Field | Type | Required | Rules |
|---|---|---:|---|
| Id | UUID | Yes | Stable child identifier. |
| StudentProfileId | UUID | Yes | Required parent relationship; history survives changes to any one program admission. |
| Category | Fixed enum | Yes | SSC/O-Level, HSC/A-Level, Bachelor/Graduation, Master, PGDIT, or Other. |
| Examination | String snapshot | No | Optional source-form label when distinct from Category. |
| SchoolOrCollege | String snapshot | No | Optional value exactly as supplied on the source form. |
| Board | String snapshot | No | Optional value exactly as supplied; not a foreign key to the mutable EducationBoard catalog. |
| Group | String snapshot | No | Optional value exactly as supplied; not a foreign key to the mutable EducationGroup catalog. |
| Department | String snapshot | No | Optional value exactly as supplied on the source form. |
| University | String snapshot | No | Optional value exactly as supplied on the source form. |
| Subject | String snapshot | No | Optional value exactly as supplied on the source form. |
| Institution | String snapshot | No | Optional value exactly as supplied on the source form. |
| Year | String snapshot | No | Optional value exactly as supplied, preserving the form's representation. |
| MarksObtained | String snapshot | No | Optional value exactly as supplied, preserving precision/format. |
| MarksPercentage | String snapshot | No | Optional value exactly as supplied, preserving precision/format. |
| CGPA | String snapshot | No | Optional value exactly as supplied, preserving precision/format. |
| Class | String snapshot | No | Optional value exactly as supplied on the source form. |
| Division | String snapshot | No | Optional value exactly as supplied on the source form. |

Only Category is a controlled value. Every other prior-education field is an optional text snapshot, not a foreign key to a mutable Configuration catalog, so values absent from catalogs and their original paper-form representations are retained. A previously recorded IIT program admission (for example, PGDIT) remains a separate admission under the profile and is not duplicated as an academic-history entry. A program not previously recorded in Student Management may be entered once as profile-level academic history.

## GraduationEligibility

Computed read model; not persisted as a student-owned balance or status snapshot.

| Value | Source | Rules |
|---|---|---|
| RequiredCredits | Selected Degree | Read the configured positive credit total. |
| CompletedCredits | Passed-course results provider | Read-only integration boundary keyed to StudentAdmissionRecord; no duplicate profile field, admission total, or result data. Credits from another admission do not carry over without a separately approved transfer rule. |
| AssignedHallPresent | StudentAdmissionRecord.AssignedHallId | Required for this admission's eligibility. |
| RegistrationNumberPresent | StudentAdmissionRecord.RegistrationNumber | Required for this admission's eligibility. |
| Eligibility and unmet requirements | Derived | Eligible only when all three conditions are met; missing credit data never yields eligibility and is identified in the response. |

## Relationships and Integrity

- One `StudentProfile` belongs to exactly one `PortalUser`; one Student account may be linked to at most one profile.
- One `StudentProfile` has zero or more `StudentAdmissionRecord` entries. Each admission references one Degree and one Batch; the Batch must belong to the selected Degree.
- A profile may have at most one admission for the same Degree and Batch; a different Degree or Batch is represented by a separate admission.
- An admission may reference one Hall after it is assigned. Hall and registration number may be updated independently.
- One StudentProfile has zero or more academic-history entries shared across its admissions; existing IIT program admissions are not duplicated as history entries.
- Registration number is unique across admissions when set. Applicant email is unique across all `PortalUser` accounts. A normalized email matching a Student account reuses its profile; a non-Student account is never merged.
- Initial Student account, profile, first admission, and their audit events commit atomically. Additional admissions and profile updates commit together; setup email is requested only when a new account is created.
- Student Management stores no attachments, password/setup proof, course registration, course result, or duplicate completed-credit total.