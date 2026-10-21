# Data Model: Student Management

**Feature**: [Student Management](spec.md)  
**Status**: Implemented by the `StudentManagement` migration and, for City Corporation address references, the `StudentAddressCityCorporations` migration (adds `MailingCityCorporationId` and `PermanentCityCorporationId`, their indexes and restrictive keys, and the checks `CK_StudentProfiles_MailingLocality_AtMostOne` and `CK_StudentProfiles_PermanentLocality_AtMostOne`). Published CGPA remains a read-only computed input, not a Student Management column.

## StudentProfile

The person-level Student record. It is linked one-to-one to a Student portal account and may have multiple program admissions.

| Field | Type | Required | Rules |
|---|---|---:|---|
| Id | UUID | Yes | Stable admission identifier and integration key for the results owner. |
| UserId | Portal user identifier | Yes | Unique foreign key to the linked Student-role account; uniqueness is on the profile, not on an admission. |
| ApplicantMobile | String | Yes | Shared required contact number; not verified by this feature. |
| Personal and contact fields | Typed values | Per source form | Applicant/family names, birth and nationality data, and optional guardian/emergency contacts. |
| MailingAddressDetails | String | No | One free-text house/road/locality detail value; required when a mailing address is supplied. |
| MailingUpazilaId | UUID | No | References an existing Upazila. Mutually exclusive with MailingCityCorporationId; exactly one locality reference is required when MailingAddressDetails is supplied. |
| MailingCityCorporationId | UUID | No | Restrictive reference to an existing City Corporation. Mutually exclusive with MailingUpazilaId; District and Division are resolved from the selected locality. |
| PermanentAddressDetails | String | No | One free-text house/road/locality detail value; required when a permanent address is supplied. |
| PermanentUpazilaId | UUID | No | References an existing Upazila. Mutually exclusive with PermanentCityCorporationId; exactly one locality reference is required when PermanentAddressDetails is supplied. |
| PermanentCityCorporationId | UUID | No | Restrictive reference to an existing City Corporation. Mutually exclusive with PermanentUpazilaId; District and Division are resolved from the selected locality. |
| Address cell numbers | Strings | No | Mailing and permanent address cell numbers remain separate and are retained when present on the form. |
| Audit fields | Existing audited-entity fields | Yes | Server-generated created/modified actor and UTC timestamps. |

The linked `PortalUser` remains the canonical source for applicant email, login name, Student role, and credential-setup status. Email uniqueness is enforced across all portal accounts. Student Management does not copy email into a second writable profile field.

For each supplied address, details, `districtId`, and exactly one locality (Upazila or City Corporation) are required. The Admin selects Division and District, and both locality catalogs load filtered to that District. `districtId` is request validation context only: the service verifies that the chosen locality belongs to it, but StudentProfile stores only the locality foreign key and derives District and Division from Configuration. The API does not accept a Division ID. Keep existing Upazila reference columns and values; a new migration adds nullable City Corporation columns, restrictive foreign keys, indexes, and at-most-one-locality check constraints. The application validates that an address with details has one locality and that an address without details has no locality.

## StudentAdmissionRecord

One program enrollment for a Student profile. A profile may have separate admissions for PGDIT, EMIT, or other programs; each admission references its own academic offering and has its own later identifiers and graduation eligibility.

| Field | Type | Required | Rules |
|---|---|---:|---|
| Id | UUID | Yes | Stable identifier and key for published academic-results lookup for this specific program admission. |
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
| SchoolOrCollege | String snapshot | No | The chosen institution's name when chosen, otherwise the value typed from the form. |
| SchoolOrCollegeId | UUID | No | Restrictive reference to an EducationalInstitution of type School or College, when chosen from the catalog. |
| Board | String snapshot | No | The chosen board's name when chosen, otherwise the value typed from the form. |
| BoardId | UUID | No | Restrictive reference to an EducationBoard, when chosen. |
| Group | String snapshot | No | The chosen group's name when chosen, otherwise the value typed from the form. |
| GroupId | UUID | No | Restrictive reference to an EducationGroup, when chosen. |
| Department | String snapshot | No | Optional value exactly as supplied on the source form. |
| University | String snapshot | No | The chosen university's name when chosen, otherwise the value typed from the form. |
| UniversityId | UUID | No | Restrictive reference to an EducationalInstitution of type University, when chosen. |
| Subject | String snapshot | No | Optional value exactly as supplied on the source form. |
| Institution | String snapshot | No | The chosen institution's name when chosen, otherwise the value typed from the form. |
| InstitutionId | UUID | No | Restrictive reference to an EducationalInstitution of any type, when chosen. |
| Year | String snapshot | No | Optional value exactly as supplied, preserving the form's representation. |
| MarksObtained | String snapshot | No | Optional value exactly as supplied, preserving precision/format. |
| MarksPercentage | String snapshot | No | Optional value exactly as supplied, preserving precision/format. |
| CGPA | String snapshot | No | Optional value exactly as supplied, preserving precision/format. |
| Class | String snapshot | No | Optional value exactly as supplied on the source form. |
| Division | String snapshot | No | Optional value exactly as supplied on the source form. |

Category is a controlled value. School/college, university, institution, board, and group reference their Configuration catalog record when one is chosen and always keep a name snapshot: the catalog name when the record was first chosen for the row, which later renames or deactivations never rewrite (a row that keeps the same record keeps its snapshot). A newly chosen record must be active and, for school/college and university, of the fitting type. A value not in a catalog is typed and kept without a reference. Every other prior-education field is an optional text snapshot. A catalog record used by any history row cannot be deleted (deactivate it instead). A previously recorded IIT program admission (for example, PGDIT) remains a separate admission under the profile and is not duplicated as an academic-history entry. A program not previously recorded in Student Management may be entered once as profile-level academic history.

## GraduationEligibility

Computed read model; not persisted as a student-owned balance or status snapshot.

| Value | Source | Rules |
|---|---|---|
| RequiredCredits | Selected Degree | Read the configured positive credit total. |
| MinimumCgpa | Selected Degree | Read the configured minimum CGPA, greater than 0 and at most 4.00. |
| CompletedCredits | Published results provider | Read passed-course credits through an admission-keyed, read-only boundary; do not persist a copy. Credits from another admission do not carry over without an approved transfer rule. |
| CurrentCumulativeCgpa | Published results provider | Read the current cumulative CGPA for this admission, on the configured 4.00 Grade numeric scale; it is recalculated by the academic-results owner as results publish and is not stored or calculated in Student Management. |
| AssignedHallPresent | StudentAdmissionRecord.AssignedHallId | Required for this admission's eligibility. |
| RegistrationNumberPresent | StudentAdmissionRecord.RegistrationNumber | Required for this admission's eligibility. |
| Eligibility and unmet requirements | Derived | Eligible only when credits meet RequiredCredits, CurrentCumulativeCgpa meets MinimumCgpa, and Hall and registration number are present. Missing academic data is unavailable and never yields eligibility. |

## Relationships and Integrity

- One `StudentProfile` belongs to exactly one `PortalUser`; one Student account may be linked to at most one profile.
- One `StudentProfile` has zero or more `StudentAdmissionRecord` entries. Each admission references one Degree and one Batch; the Batch must belong to the selected Degree.
- A profile may have at most one admission for the same Degree and Batch; a different Degree or Batch is represented by a separate admission.
- An admission may reference one Hall after it is assigned. Hall and registration number may be updated independently.
- Each supplied mailing or permanent address references exactly one Upazila or City Corporation. Both are District children; parent names are derived, and neither locality can be deleted while referenced.
- One StudentProfile has zero or more academic-history entries shared across its admissions; existing IIT program admissions are not duplicated as history entries.
- Registration number is unique across admissions when set. Applicant email is unique across all `PortalUser` accounts. A normalized email matching a Student account reuses its profile; a non-Student account is never merged.
- Initial Student account, profile, first admission, and their audit events commit atomically. Additional admissions and profile updates commit together; setup email is requested only when a new account is created.
- Student Management stores no attachments, password/setup proof, course registration, course result, completed-credit total, or CGPA. Published results remain the sole source for admission-specific credits and current cumulative CGPA.