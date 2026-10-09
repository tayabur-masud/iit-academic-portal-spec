# Feature Specification: Audit System

**Feature Branch**: `feature/002-audit-system`

**Created**: 2026-10-09

**Status**: Draft

**Input**: User description: Create a centralized, reusable audit system for security-sensitive and business-related activity in the IIT Academic Portal, with protected review, change tracking, reliability, and configurable retention.

## Clarifications

### Session 2026-10-09

- Q: Who may review audit history? → A: Only users whose current session is acting in the Admin role. There is no separate auditor permission for now; one may be added by a later feature if needed.
- Q: Must this feature deliver audit events for every mandatory business action listed (courses, fees, results, and so on)? → A: No. This feature delivers the reusable audit capability: the event contract, the recording paths, protected review, and integrity protections. Each owning feature adds its mandatory business events and before/after allowlist as that feature is built, and its own acceptance criteria cover them.
- Q: When a mandatory business action is rejected or denied, which outcomes should be recorded as formal business audit events? → A: Record committed changes, final decisions (including approvals/rejections), and policy-based business denials; exclude validation errors and transient technical failures. Access-control denials remain security events.
- Q: Which additional security-sensitive actions, if any, should the initial audit capability record beyond authentication successes and failures, protected-operation denials, and audit-record mutation attempts? → A: Also record password-reset outcomes, session creation/revocation and logout, and active-role switch outcomes.
- Q: Which denied requests are access-denied audit events? → A: Requests from an authenticated session that the server refuses (HTTP 403). Anonymous requests refused with HTTP 401, including the portal's own check for whether a visitor is signed in, are not audit events; they are counted in an operational metric.
- Q: What does an event's source contain? → A: The client IP address derived on the server from the deployment's trusted forwarded headers. No user-agent or other client-supplied value is stored.
- Q: How are sign-ins and sessions recorded? → A: One event per sign-in attempt, with no separate session-created event. A failed attempt records no account, only the source and correlation identifier. Session revocation is its own event with a reason: logout, idle timeout, or password reset.
- Q: How is an actor shown to reviewers? → A: Events store only the stable actor identifier. The review screen resolves the current account email when it is viewed and shows "(account removed)" when the account no longer exists. No personal data is copied into the event.
- Q: Are reads of audit history recorded? → A: Yes. Each Admin search and event-detail view is recorded as a security event. Only Admins can read, so the volume is low.
- Q: Does the audit capability automatically capture entity changes? → A: Not initially. Owning features record explicit events with their allowlisted fields. Automatic capture is added when the first business feature needs it.
- Q: What counts as a release? → A: The initial audit capability is released only when event recording (US1), Admin review (US3), and record protection with reliable recording (US4) are all complete and their gates verified.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Record Important Events (Priority: P1)

As a portal operator, I need important security and business actions recorded consistently so that authorized reviewers can establish what happened, when, and in which context.

**Why this priority**: Audit records are the foundation for incident investigation, accountability, and the later review workflows.

**Independent Test**: Exercise the shared event contract with representative business-event submissions and in-scope security events; confirm the common event structure and recording behavior, and verify mandatory business events are required only for features already integrated with the audit capability. Unintegrated business workflows are not part of this feature's delivery.

**Acceptance Scenarios**:

1. **Given** a mandatory audited business action succeeds, **When** it is committed, **Then** one or more corresponding business audit events identify the actor, action, affected entity, outcome, UTC time, and correlation context where available.
2. **Given** an unauthenticated or system-generated event, **When** it is recorded, **Then** the actor is null and the event identifies its action, outcome, source, and affected entity when known.
3. **Given** a future feature introduces an important administrative action, **When** that feature integrates with the audit system, **Then** it declares which actions are mandatory audit events and which changed fields are approved for before/after tracking, and records them using the common structure.
4. **Given** concurrent actions produce audit events, **When** they are persisted, **Then** every committed event has a unique identifier and remains independently reviewable.
5. **Given** a routine low-value diagnostic event occurs, **When** the application handles it, **Then** it is not required to be retained as formal audit history and its recording does not change the application outcome.
6. **Given** an integrated feature reaches a final policy-based business denial for a mandatory action, **When** the decision is made, **Then** it records a denied business audit event; input-validation errors and transient technical failures do not create formal business audit events.
7. **Given** a password reset, session creation or revocation, logout, or active-role switch is processed, **When** the action succeeds or fails, **Then** a security audit event records its action and outcome without recording credentials, reset proofs, cookies, or session handles.

---

### User Story 2 - Understand Relevant Changes (Priority: P1)

As an authorized reviewer, I need to see which approved fields changed and their safe previous and new values so that I can understand the effect of an audited modification without exposing secrets or collecting unnecessary personal information.

**Why this priority**: Knowing only that a record changed is insufficient to investigate important academic and administrative decisions.

**Independent Test**: Update records with allowlisted and non-allowlisted fields; verify safe before/after values are captured only for approved fields, while sensitive fields are omitted or represented as changed without disclosing their values.

**Acceptance Scenarios**:

1. **Given** an audited update changes allowlisted fields, **When** the update succeeds, **Then** its event identifies each changed field and its previous and new safe values.
2. **Given** an update changes a sensitive or non-allowlisted field, **When** the event is recorded, **Then** the field's secret or personal value is not stored; a safe change indicator may be recorded where needed.
3. **Given** an update changes no auditable values, **When** the business action is otherwise required to be audited, **Then** the event still records the action and outcome without inventing a change summary.

---

### User Story 3 - Search and Review Audit History (Priority: P1)

As an administrator, I need to find and inspect relevant audit history so that I can review activity without access to unrelated protected application data.

**Why this priority**: Records must be usable during routine oversight and incident investigations, not merely retained.

**Independent Test**: Sign in as an Admin and as a user with another role; search by each supported filter, traverse pages, inspect an event, and repeat as the non-Admin user.

**Acceptance Scenarios**:

1. **Given** an administrator acting in the Admin role, **When** they search by a date range, actor, action type, entity type, entity identifier, outcome, or correlation/request identifier, **Then** the results include only events matching all supplied filters.
2. **Given** search results span multiple pages, **When** the reviewer moves between pages, **Then** each event appears at most once in the result sequence, pages are bounded, and results are ordered newest first with a stable tie-breaker.
3. **Given** an authorized reviewer opens an event, **When** its details are displayed, **Then** the safe event data and change summary are visible, timestamps are clearly labeled in the viewer's display time zone, and excluded secrets remain unavailable.
4. **Given** any session requests audit search or detail data, **When** the server evaluates the request, **Then** access is allowed only if that session is actively acting in the Admin role; every session with a non-Admin active role is denied regardless of its other assigned roles, the server reevaluates the active role on every request, and this feature introduces no separate audit-review permission.
5. **Given** a user has Teacher and Coordinator roles but not Admin, **When** the user switches between those active roles, **Then** neither role grants audit access.
6. **Given** a user has the Admin role and another role, **When** the user switches the active role away from Admin, **Then** the next audit request is denied; switching back to Admin restores access. The server reevaluates the current session's active role on every request, and access from a previous active role is not retained implicitly.

---

### User Story 4 - Protect Records and Ensure Reliable Recording (Priority: P1)

As an institutional security owner, I need audit history to resist unauthorized alteration and to remain reliable when recording infrastructure is unhealthy so that important actions cannot bypass accountability silently.

**Why this priority**: Alterable or silently incomplete audit history undermines trust in every other audit capability.

**Independent Test**: Attempt unauthorized reads and all application-level mutations; simulate audit persistence failures during business and security events; verify rollback, independent retry/alert behavior, and the original authorization result.

**Acceptance Scenarios**:

1. **Given** a mandatory business audit event is required, **When** the business change and event are committed, **Then** they use one transaction with the existing PostgreSQL database; if either write fails, neither change commits.
2. **Given** a security audit event cannot be persisted immediately, **When** authentication or authorization is decided, **Then** that outcome is unchanged solely because of the audit failure, and a durable monitored retry retains the event until it is recorded or explicitly handled by an authorized recovery process.
3. **Given** a retry reaches its limit or an event-loss risk occurs, **When** monitoring evaluates the audit path, **Then** it raises an actionable alert and does not silently discard the event.
4. **Given** a user or ordinary application service attempts to update or delete an audit event, **When** the request is processed, **Then** the operation is denied; the application database role also cannot update or delete persisted audit events.
5. **Given** an unauthorized audit-record mutation is attempted, **When** the attempt is denied, **Then** a safe security event is recorded without recursively requiring another audit event to record its own failure.
6. **Given** no approved retention policy is configured, **When** retention or archival processing is considered, **Then** no audit event is automatically deleted or archived.
7. **Given** records are archived under a later approved policy, **When** an authorized reviewer searches historical activity, **Then** archived records remain retrievable under the same access controls and integrity protections as active records.

## Edge Cases

- An unsuccessful sign-in has no verified actor identity; the actor identifier is null and the submitted password, token, cookie, and unneeded account-identifying input are not retained.
- Security-event retry, alerting, and recovery must not recursively require audit events to record their own failures; recovery is explicitly authorized and all retry exhaustion or event-loss risks are monitored.
- Authentication and access-denied events that cannot be persisted immediately remain on a durable retry path until successfully recorded or explicitly handled by an authorized recovery process; exhausting automatic retries is not permission to discard them.
- Ordinary diagnostic events are not formal audit records unless an owning feature explicitly classifies the event as mandatory business or security audit history.
- An account with multiple academic roles, including Teacher and Coordinator, receives no audit access from those roles alone. Switching roles causes the server to reevaluate access in the current session.
- A mandatory business operation is denied by an owning feature's policy; record the final denial as a business audit event. Do not record input-validation errors or transient technical failures as formal business audit events; access-control denials remain security audit events.
- An attempted audit-record update or deletion is denied and recorded as a security event through the non-recursive security-event path.
- Application-level controls and the application's restricted database role do not establish protection against a privileged database administrator or database superuser; the specification does not claim complete tamper-proofing against those principals.
- A user account, role assignment, or entity is later deactivated, removed, or unavailable; historical events remain identifiable and reviewable by stable actor and entity identifiers and are not rewritten to match current state.
- A requested date range has an end before its start, an invalid timestamp, or an unsupported time-zone value; reject the filter with a safe validation response and return no misleading partial results.
- An event lacks a correlation/request identifier; assign a server-generated correlation identifier where available and otherwise allow the field to remain null.
- A bulk operation affects multiple entities or has mixed outcomes; record sufficient per-entity outcomes to identify which changes succeeded, failed, or were denied.
- Audit event writes arrive concurrently or share the same timestamp; unique IDs and a deterministic ordering prevent loss and unstable pagination.
- A field is not on the owning feature's approved audit allowlist; do not store its previous or new value.
- The durable operational/security retry mechanism or its storage is unavailable, retries are exhausted, or event loss is suspected; raise a monitored critical alert and retain the event for authorized recovery rather than silently discarding it.
- Retention configuration is absent or invalid; do not delete or automatically archive records, and surface the configuration problem to operators.
- Archival is introduced under an approved policy; archived records remain searchable and retrievable by authorized reviewers, with access and integrity protections equivalent to active records.
- A database administrator or superuser has privileges beyond the application's database role; application-level append-only behavior must not be represented as protection against that elevated access.

## Requirements *(mandatory)*

### Business Rules

- The audit system is a reusable capability of the existing IIT Academic Portal and its layered-monolithic application; it is not a separate application or microservice.
- Formal audit history distinguishes mandatory business audit events, security audit events, and optional diagnostic events. These classes have separate recording and failure rules below; not every HTTP request or database operation is a formal audit event.
- Mandatory business audit events cover user creation, profile changes, activation/deactivation, and role assignment/removal; course creation and material changes; course registration and enrollment decisions; elective selection decisions and administrative overrides; fee-payment status changes and payment overrides; result creation, modification, approval, rejection, and publication; batch and coordinator assignment changes; final business decisions; policy-based business denials; and other business actions explicitly declared mandatory by future features. This list is the coverage target. The feature that owns each action implements its event when that feature is built; this feature does not. Input-validation errors and transient technical failures are not formal business audit events; access-control denials are security audit events.
- Security audit events cover authentication successes/failures; password-reset outcomes; session creation/revocation and logout; active-role switch outcomes; access-denied events for protected operations (authenticated requests refused with HTTP 403; anonymous 401 responses are not audit events); attempts to alter or delete audit records; Admin reads of audit history; and other explicitly classified security-sensitive actions. Session creation is part of the sign-in event; revocation is a separate event with its reason.
- Optional diagnostic events are low-value routine or technical events that do not need formal audit retention. They MUST NOT be promoted to mandatory audit history by default, and their recording failure MUST NOT change the primary operation outcome.
- Each future feature MUST declare which of its actions require mandatory business audit events and identify the specific changed fields approved for before/after tracking.
- Mandatory business audit events MUST be persisted atomically with their business change using one transaction with the existing PostgreSQL database and the existing application architecture. If either write fails, neither the business change nor its required event commits. This requirement does not assume a separate database or distributed transaction.
- Security audit events MUST NOT change the primary authentication or authorization outcome solely because audit recording is unavailable. Failed writes MUST remain in a durable, monitored retry mechanism until successfully recorded or explicitly handled by an authorized recovery process; retry exhaustion and event-loss risks MUST alert and MUST NOT result in silent discard.
- A reviewer is authorized only when the current session is acting in the Admin role. Student, Teacher, or Coordinator role membership grants no audit access, and this feature introduces no new role or permission. A separate auditor permission may be introduced by a later feature.
- Audit authorization MUST be reevaluated server-side on every search, detail, or other audit-data request against the current session context. Switching active roles MUST NOT carry access from a prior role implicitly.
- Portal APIs and ordinary application services MUST provide append-only audit behavior. The application's database role MUST be able to append and read audit records but MUST NOT update or delete existing records; the enforcement mechanism is evaluated during planning.
- Application-level and application-database-role protections do not make records tamper-proof against privileged database administrators or superusers. Cryptographic hash chains, external tamper-evident storage, and immutable archival infrastructure are optional enhancements to evaluate, not requirements to introduce without approval.
- Audit values MUST be selected by an explicit allowlist owned by the feature responsible for the data; fields not approved for audit MUST NOT have their values collected.
- Audit records MUST NOT be automatically deleted or archived unless an institutionally approved policy is configured through an authorized configuration process. No retention period may be inferred from technical defaults.
- Archival is distinct from deletion. If introduced under an approved policy, archives MUST preserve authorized retrieval, access controls, and integrity protections. Any later deletion MUST follow the approved policy and applicable authorization and audit requirements.
- Timestamps are stored in UTC. Review screens display them in a clearly identified viewer time zone.

### Functional Requirements

#### Event coverage and structure

- **FR-001**: The system MUST classify formal audit events as: (a) mandatory business events for user creation, profile changes, activation/deactivation, role assignment/removal, course creation/material changes, registration/enrollment decisions, elective selections/administrative overrides, fee-payment status changes/payment overrides, result creation/modification/approval/rejection/publication, batch/coordinator assignment changes, final business decisions, policy-based business denials, and future business actions declared mandatory by their owning feature; (b) security events for authentication successes/failures, password-reset outcomes, session creation/revocation and logout, active-role switch outcomes, access-control denials, audit-record mutation attempts, and other explicitly classified security-sensitive actions; or (c) optional low-value diagnostic events that are not formal audit history. Input-validation errors and transient technical failures MUST NOT be formal business audit events. The system MUST NOT require every HTTP request or database operation to be audited.
- **FR-002**: Each event MUST include a unique event identifier, UTC timestamp, action/event type, outcome (`success`, `failure`, or `denied`), and affected entity type and identifier when one is known.
- **FR-003**: Each event MUST include the actor's stable user identifier when known; the actor MUST be null for unauthenticated or system-generated activity without a user actor.
- **FR-004**: Each event MUST include a correlation or request identifier when available and identify the server-derived source when it materially helps review. The source is the client IP address derived on the server from trusted forwarded headers; no user-agent or client-supplied origin is stored or trusted.
- **FR-005**: The event structure MUST support safe metadata and a change summary without requiring feature-specific changes to the audit-review interface.
- **FR-006**: Audit event types MUST be extensible for future important actions while preserving common fields and outcome meanings. Each future feature MUST declare its mandatory business event types, any security event types it introduces, and the fields explicitly approved for before/after tracking.

#### Change tracking and privacy

- **FR-007**: For audited modifications, the system MUST record changed field names and previous/new values only for fields explicitly allowlisted by the owning feature.
- **FR-008**: For sensitive or non-allowlisted fields, the system MUST omit previous/new values; where the fact of a change is necessary, it MAY record only a safe changed indicator.
- **FR-009**: Audit records MUST NOT contain passwords, authentication tokens, session cookies or handles, password-reset proofs, cryptographic secrets, or other credentials. They MUST minimize personal information and metadata to what is needed to identify and review the action.
- **FR-010**: Audit details MUST NOT expose sensitive metadata in public application responses or ordinary application logs. Operational logs MUST use appropriately protected access and MUST NOT duplicate excluded secrets.

#### Recording, authorization, and integrity

- **FR-011**: Audit recording MUST occur on the trusted server side and MUST NOT depend on frontend logging or client-submitted event claims.
- **FR-012**: The audit capability MUST be reusable by portal features through a documented event contract, supporting explicit business-event recording with allowlisted change fields. Automatic capture of entity changes is not part of the initial capability and is added when the first business feature needs it.
- **FR-013**: A mandatory business audit event and its business change MUST be committed atomically within one transaction against the existing PostgreSQL database and existing application architecture. If either write fails, neither MUST commit. This MUST NOT require a separate database or distributed transaction.
- **FR-014**: Security audit events, including authentication successes/failures and access-denied events, MUST NOT change the primary authentication or authorization outcome solely because audit recording is unavailable. A failed write MUST be retained by a durable, monitored retry mechanism until successfully recorded or explicitly handled by an authorized recovery process. The mechanism MUST NOT silently discard events; no specific queue technology is prescribed.
- **FR-015**: Audit access MUST be checked server-side on every search, detail, or other audit-data request using the current session context. Access is permitted only when the current session's active role is Admin. Student, Teacher, or Coordinator roles MUST NOT grant access. Angular navigation visibility and client-submitted role information MUST NOT grant authorization.
- **FR-016**: Review access MUST be read-only. Portal APIs and ordinary application services MUST deny attempts to update or delete persisted audit events.
- **FR-017**: Monitoring MUST alert on audit persistence failure, retry exhaustion, invalid retention configuration, and risks of event loss. Audit failures MUST NOT be recursively recorded by requiring another audit event; such failures use the same monitored recovery path.

#### Search, retention, and operations

- **FR-018**: Authorized reviewers MUST be able to filter events by UTC date/time range, actor, action type, entity type, entity identifier, outcome, and correlation/request identifier; supplied filters MUST combine predictably. A supplied start time is inclusive and a supplied end time is exclusive.
- **FR-019**: Search results MUST support bounded pagination, default to newest-first order, and use a stable tie-breaker so concurrent writes do not cause avoidable omissions or duplicates across pages.
- **FR-020**: Authorized reviewers MUST be able to open a detailed event view containing the event's safe metadata and change summary without exposing excluded values. The actor is shown by its stable identifier together with the account email resolved when the event is viewed, or "(account removed)" when the account no longer exists.
- **FR-021**: Search MUST validate date-range inputs and return understandable empty, invalid-filter, loading, and error states consistent with [the project design system](../design-system.md).
- **FR-022**: The system MUST store timestamps in UTC and display them with an explicit time-zone indication in the user's configured display time zone or, if none is configured, UTC.
- **FR-023**: Audit storage and search MUST support suitable indexes for the required filter fields and time ordering; exact index definitions are determined during planning.
- **FR-024**: A retention duration MUST NOT be invented or inferred from technical defaults. Retention settings may be applied only through an authorized configuration process based on an approved institutional policy. Until such a policy is configured, no automatic deletion may occur.
- **FR-025**: Archival is distinct from deletion and MUST NOT occur automatically as part of the initial feature unless an approved policy explicitly requires it. If archival is later introduced, archived events MUST remain retrievable by authorized reviewers and protected by appropriate access controls and integrity measures. Any future deletion MUST follow the approved retention policy through a separately authorized process, remain unavailable to the ordinary application database role, and meet applicable audit requirements.
- **FR-026**: The audit capability MUST support concurrent requests without losing committed events or allowing one event to overwrite another.
- **FR-027**: For audit records, the database role used by the application MUST be limited to the permissions needed to append and read records and MUST NOT be able to update or delete existing records. The implementation plan MUST evaluate suitable database-level enforcement; this specification does not prescribe a particular mechanism.
- **FR-028**: Unauthorized audit-record update or deletion attempts MUST be denied and recorded as security audit events through the durable security-event path without recursively requiring another audit event for failure handling.
- **FR-029**: Each Admin search and event-detail view of audit history MUST be recorded as a security audit event naming the actor, the kind of access, and the filters used, without copying the returned event contents.

### Acceptance Criteria

- **AC-001 - Classified event recording**: Every tested in-scope security event and every mandatory business event from a feature already integrated with the audit capability produces a correctly classified event with its required identity, UTC time, type, outcome, affected entity when known, actor when known, and correlation/source context where available. Integrated features record committed changes, final decisions, and policy-based business denials; input-validation errors and transient technical failures are not formal business events, and access-control denials are security events. Representative business-event submissions verify the shared contract without requiring unintegrated business workflows to be delivered by this feature. Stable actor and entity identifiers remain reviewable after the associated account or business entity is deactivated or removed. Future features declare mandatory event types and before/after allowlists. Low-value diagnostic events are not required as formal audit history, and not every request or database operation creates a formal audit event. (FR-001–FR-006, FR-011–FR-012, FR-026)
- **AC-002 - Safe change tracking**: Tests confirm allowlisted changes include field names and approved before/after values; sensitive and non-allowlisted values, credentials, and unnecessary personal metadata are absent from persisted events, API responses, and ordinary logs. (FR-007–FR-010)
- **AC-003 - Session-scoped authorized review**: Audit reads are authorized server-side on every request. Access requires the session's active role to be Admin; no other role grants audit access. Switching the active role reevaluates authorization and does not carry access from the prior role. Authorized reviewers can filter, paginate, and inspect safe details; other reads are denied without disclosure. Each Admin search and detail view is itself recorded as a security event. (FR-015, FR-018–FR-022, FR-029)
- **AC-004 - Transactional and operational reliability**: Mandatory business changes and their events use one transaction in the existing PostgreSQL database; either both commit or neither does. Security-event failures do not change authentication/authorization outcomes, remain durably retryable until recorded or explicitly handled through authorized recovery, and alert on failure, exhaustion, or loss risk without recursive audit logging or silent discard. (FR-013–FR-014, FR-017, FR-028)
- **AC-005 - Retention and archival**: No retention period is inferred and no automatic deletion or archival occurs without an approved policy applied through authorized configuration. Archival, if later introduced, is distinct from deletion and preserves authorized historical retrieval, access control, and integrity. Any deletion uses a separately authorized process and follows the approved policy and applicable audit requirements. Searchable fields have suitable indexes. (FR-023–FR-025)
- **AC-006 - Append-only integrity boundary**: Application interfaces/services and the application's database role cannot update or delete existing events; unauthorized mutation attempts are denied and recorded through the non-recursive security-event path. The specification does not claim protection against privileged database administrators or superusers. Database-level enforcement is evaluated during planning. (FR-016–FR-017, FR-027–FR-028)

### Non-Functional Requirements

- The audit system MUST follow security-by-default, least-privilege, data-minimization, and layered-architecture principles in the project constitution.
- Audit history MUST be available only to authorized reviewers and MUST not be included in ordinary user-facing responses or unprotected logs.
- Search and review MUST remain usable with keyboard navigation and assistive technology and follow the shared design system.
- Event recording and search MUST be observable through protected operational metrics and alerts that identify failures without containing secrets.
- The design MUST preserve accurate UTC ordering and complete committed event records under concurrent activity.
- Mandatory business-event atomicity MUST use the existing application architecture and PostgreSQL database; it does not assume another database or a distributed transaction.
- The application's database role MUST have append/read but not update/delete permission for audit records. This does not claim resistance to privileged database administrators; hash chains, external tamper-evident storage, and immutable archival are optional planning evaluations, not mandatory infrastructure.

### Key Entities *(include if feature involves data)*

- **Audit event**: A classified account of a mandatory business action or security action, including unique identity, UTC time, optional actor, action type, affected entity, outcome, correlation/request context, source, and safe metadata. Optional diagnostics are not formal audit history by default.
- **Change summary**: The allowlisted field names and safe previous/new values for an audited modification, or a safe indication that a protected value changed.
- **Audit reviewer**: A user whose current session is acting in the Admin role.
- **Retention policy**: An institutionally approved configuration applied through an authorized process that defines when records may be archived or deleted; archival remains retrievable and is distinct from deletion. Without such approval, neither automatic archival nor deletion occurs.
- **Correlation context**: A request or correlation identifier used to relate events from one application operation or workflow.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of committed mandatory business changes in integrated features have their correctly classified audit event committed atomically with them; 100% of final business decisions and policy-based business denials in integrated features have a corresponding business audit event; 100% of security events are eventually recorded or explicitly handled by authorized recovery, with the correct actor, action, affected entity when known, outcome, and UTC timestamp.
- **SC-002**: 100% of tested attempts by unauthorized users to read audit history are denied without disclosing event contents; 100% of attempts by the application's database role or ordinary application services to update or delete audit history are denied without changing stored events.
- **SC-003**: 100% of test cases involving passwords, authentication tokens, cookies, reset proofs, cryptographic secrets, and non-allowlisted values find none of those values in persisted events or ordinary application responses/logs.
- **SC-004**: 100% of simulated mandatory business-audit persistence failures prevent their associated business mutations from committing; 100% of simulated security-event path failures leave authentication/denial outcomes unchanged, retain the event for retry or authorized recovery, and raise an observable alert without silent loss or recursive logging.
- **SC-005**: At least 95% of representative authorized reviewers complete a task locating a specified event using the available filters within two minutes without assistance.
- **SC-006**: At least 95% of first-page searches over a representative volume of 1,000,000 events (until the operations owner confirms another volume) return within three seconds.
- **SC-007**: 100% of retention tests confirm no automatic deletion or archival occurs unless an approved retention policy is configured through an authorized process.
- **SC-008**: Once archival is introduced under an approved policy, 100% of tests involving archived events confirm authorized reviewers can retrieve them and unauthorized users remain denied.

## Assumptions

- Existing portal authentication supplies the actor identity and current authorization context. The audit system does not create accounts or role assignments.
- The existing Admin role may review audit history only while it is the session's active role. No auditor permission exists in this feature.
- Session-scoped active-role authorization is supplied by the existing authentication system. Each audit request reevaluates the current session; no access is inferred from a previously active role or client-supplied role.
- Mandatory business audit events use a single transaction with the existing PostgreSQL database and layered-monolithic application. No separate database or distributed transaction is assumed.
- Each owning feature is responsible for approving its allowlist of fields whose previous and new values are safe to retain. Until a value is allowlisted, store no before/after value for that field.
- No institutional retention duration has been approved. Records remain until an approved policy is configured; the system must not infer or invent a duration.
- Security events use a durable, monitored retry/recovery mechanism independent of the primary authentication or authorization decision; its technology and operational thresholds are selected during planning, not presumed in this specification.
- Pagination uses a default page size of 25 and a maximum page size of 100; exact user-interface presentation follows the design system.
- If a user has no configured display time zone, audit timestamps are shown in UTC with that zone identified.
- Database-level enforcement for the application's append/read-only audit access is evaluated during planning. This does not guarantee protection from privileged database administrators or superusers.
- Hash chains, external tamper-evident storage, and immutable archival infrastructure are optional enhancements to evaluate against project needs; they are not mandatory without approval.

## Out of Scope

- Defining an institutional retention duration or legal/regulatory schedule.
- Creating a new audit microservice, repository, or application structure.
- Defining feature-specific course, fee, enrollment, elective, result, batch, or account business rules beyond recording their important actions.
- Recording mandatory business events for features that are not yet built. Each owning feature integrates with the audit system and declares its event types and allowlisted fields when it is delivered.
- Defining the detailed implementation classes, API paths and payload schemas, database table layout, or exact index definitions.
- Selecting a specific durable retry or database-level permission-enforcement technology before the implementation plan evaluates the existing architecture and operational requirements.
- Creating or modifying portal accounts or role assignments.
- A separate auditor role or permission, or any workflow for granting one. Audit review is Admin-only for now.
- User-facing editing, deletion, or alteration of audit history.
- Replacing security/operational telemetry, application monitoring, or general-purpose analytics systems.
- Mandating cryptographic hash chains, external tamper-evident storage, or immutable archival infrastructure without separate approval.