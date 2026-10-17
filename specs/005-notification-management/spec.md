# Feature Specification: Notification Management

**Feature Branch**: `feature/005-notification-management`

**Created**: 2026-10-15

**Status**: Implemented, except the academic producers, which are blocked on their owners (see the producer contracts). End-to-end teacher setup-email delivery was completed by feature 006 through the common `AccountCredentialSetup` event (tasks.md T016)

**Input**: Add reusable background notifications for account invitations, email confirmation, password reset, published course results, and feedback-window opening/reminders. Initially deliver email only; use Brevo for Development email and replace it before Production without changing notification producers. Support future channels without changing feature triggers. Admins manage channel-specific templates and search permanent delivery history, with security-content exclusion and feedback confidentiality.

## Clarifications

### Session 2026-10-15

- Q: When should feedback closing reminders be sent? A: Use one Admin-configurable portal-wide interval before closing, not a fixed interval or per-window setting.
- Q: May Admins disable security-message templates? A: Invitation, email-confirmation, and password-reset templates must remain enabled. Admins may edit valid wording but cannot disable these templates.
- Q: What should trigger an email-address confirmation message? → A: Only when the owning account workflow explicitly requests address confirmation; invitations alone do not automatically send a separate confirmation message.

### Session 2026-10-16

- Q: Which email provider should be used initially, and where? A: Use Brevo for all email in Development; replace it with a different provider before Production. Keep provider choice behind a replaceable delivery adapter so producers do not change.
- Q: May a security-template preview include a sample action link? A: It may show only a clearly synthetic, nonfunctional placeholder; it must not generate, display, or persist a live link or proof.

### Session 2026-10-16 (branding)

- Q: How is IIT branding (logo, colors, footer) applied to emails? A: Every email is sent in one fixed IIT-branded HTML layout built by the system around the Admin's plain-text wording, with the plain text sent as its alternative. Admins do not edit HTML, choose images, or change the layout; the branding values (logo address, brand color, portal name, logo alt text, footer, button label) are deployment settings, changed by configuration and validated at startup, not from the portal.
- Q: Where is the logo loaded from? A: A public HTTPS address (by default the portal's own `/images/iit-logo.png`, overridable by one setting), because mail clients load it from the internet. Each notification type already has its own template (subject, wording, enabled state, revisions); the layout is shared so every message looks the same.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Receive Reliable Event Notifications (Priority: P1)

As a portal user, I need important account and academic messages delivered to me without notification problems interrupting the academic action that caused them.

**Why this priority**: Reliable notification delivery is the foundation for account access and timely academic communication.

**Independent Test**: Create a teacher account or request password recovery, simulate an unavailable email service, and verify the originating action succeeds, notification work remains recoverable, and delivery is retried after service recovery.

**Acceptance Scenarios**:

1. **Given** an Admin creates a teacher or student account, **When** account creation commits successfully, **Then** an invitation is requested for that account without exposing or handing its password to the Admin.
2. **Given** a valid email-confirmation or password-reset request, **When** the owning account workflow authorizes it, **Then** the user receives the corresponding email and existing identity-disclosure protections remain unchanged.
3. **Given** email delivery is unavailable, **When** an originating action succeeds, **Then** delivery runs separately, retries without repeating that action, and exhausted retries become visible to Admins as failed notifications.
4. **Given** an originating action is rejected or rolled back, **When** notification processing runs, **Then** no success notification for that action is delivered.
5. **Given** a future delivery channel is introduced, **When** an existing feature triggers its existing event, **Then** that feature need not change how it requests notification.
6. **Given** Development email is configured with Brevo and Production uses an approved replacement, **When** the same notification event is delivered in each environment, **Then** the event producer request is unchanged and Production rejects the Development-only Brevo configuration.

---

### User Story 2 - Maintain and Preview Message Templates (Priority: P1)

As an active Admin, I need to maintain wording for each event and channel, preview it with sample data, and control whether its template is enabled without a software release.

**Why this priority**: Institutional wording must be manageable while protecting sensitive account flows and ensuring messages remain valid.

**Independent Test**: Edit a template, preview it with synthetic values without sending an email, save it, and verify a subsequently requested notification uses the new wording while historical content remains unchanged.

**Acceptance Scenarios**:

1. **Given** a supported event/channel template, **When** an Admin changes valid subject/body wording, **Then** subsequent notifications use the saved version without a release.
2. **Given** unsupported placeholders or a missing required security-link placeholder, **When** an Admin previews or saves the template, **Then** validation identifies the problem and the last valid template remains usable.
3. **Given** sample event data, **When** an Admin previews a template, **Then** the preview uses synthetic data, sends nothing, exposes no live credential link or feedback submission, and shows only a visibly synthetic, nonfunctional link placeholder for a security template.
4. **Given** an enabled ordinary academic template, **When** an Admin disables it, **Then** later matching events are recorded as suppressed rather than silently treated as delivered; enabling it again applies to future requests and does not send a backlog automatically.
5. **Given** an invitation, email-confirmation, or password-reset template, **When** an Admin attempts to disable it, **Then** the change is rejected and the template remains enabled; valid wording edits remain permitted.

---

### User Story 3 - Search History and Recover Failed Deliveries (Priority: P1)

As an active Admin, I need a searchable permanent record of notification outcomes and a controlled way to resend failed notifications.

**Why this priority**: Delivery accountability and recovery are required even when providers or recipient addresses fail.

**Independent Test**: Search successful and failed records using combined filters, inspect an ordinary message's exact sent content, confirm security content is absent, and resend a failed record without overwriting its original attempts.

**Acceptance Scenarios**:

1. **Given** notification records exist, **When** an Admin combines recipient, event, status, and UTC date filters, **Then** only matching records appear using the shared infinite-loading pattern.
2. **Given** a delivered ordinary message, **When** an Admin opens its record after template edits or an account email change, **Then** the record still shows the actual destination and exact subject/body used for that send.
3. **Given** an invitation or password-reset message, **When** history is viewed or searched, **Then** only safe delivery facts appear and no message subject/body, security link, or proof is retrievable.
4. **Given** a failed notification eligible for resend, **When** an Admin confirms resend, **Then** a new linked delivery execution is requested in the background and prior outcomes remain intact.
5. **Given** a Student, Teacher, Coordinator, anonymous visitor, or Admin account acting in a non-Admin role, **When** notification history or template management is requested directly, **Then** access is denied without disclosing restricted data.

---

### User Story 4 - Receive Academic Publication and Feedback Notices (Priority: P2)

As a student, I need to know when course results are published and when a feedback window opens or is approaching its closing time, without any disclosure of submitted feedback.

**Why this priority**: Academic messages depend on the owning publication and feedback workflows, but must use the same notification capability.

**Independent Test**: Supply representative committed publication/window events and eligible recipient lists from their owning workflows; verify correct notification selection and reminder timing using synthetic feedback data.

**Acceptance Scenarios**:

1. **Given** results are finalized but not published, **When** finalization occurs, **Then** no published-results message is sent; a committed publication notifies only students entitled to those results.
2. **Given** a feedback window opens, **When** its owning workflow identifies eligible students, **Then** those students receive the opening message with course/window information only.
3. **Given** a window remains open and a reminder is due, **When** scheduled reminder processing occurs, **Then** each eligible student receives at most one reminder for that window occurrence, independent of whether they submitted feedback.
4. **Given** a window is closed, cancelled, or rescheduled before its reminder is sent, **When** reminder eligibility is checked, **Then** no obsolete reminder is sent.
5. **Given** an Admin sets a valid portal-wide reminder interval, **When** reminder due times are evaluated, **Then** every open window uses its closing time minus that interval, with no per-window override; already issued reminders are not repeated after a policy change.

### Edge Cases

- An event is repeated, a background worker restarts, or an Admin repeats a resend request while it is pending.
- A provider accepts a send but its acknowledgement is lost; the outcome is uncertain rather than proof of receipt.
- A multi-recipient event has a mixture of valid addresses, missing addresses, and provider failures; recipients must not be exposed to one another.
- A template is edited, disabled, or invalidated after notification work has already been accepted.
- A user changes their email, is deactivated, or is removed before delivery or manual resend.
- An invitation/reset/confirmation link expires before delivery or resend.
- A scheduled reminder's deadline changes or the feedback window closes during a delivery outage.
- Notification work cannot be retained at the time an academic action commits; delivery loss must be visible and recoverable without undoing the action.
- Message values contain markup or unexpected text; rendering must not execute content or leak internal details.
- History is searched across large date ranges; results remain bounded and permission checked throughout infinite loading.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The portal MUST provide a reusable event-notification capability with email as its only initially supported delivery channel. SMS, in-app notifications, and additional channel delivery are outside initial scope.
- **FR-002**: Features MUST request notifications in terms of their business event and intended recipients, without depending on a specific delivery channel. Adding a supported channel later MUST NOT require existing features to change how they trigger notifications.
- **FR-003**: The supported event catalog MUST distinguish teacher invitation, student invitation, email confirmation, password reset, results published, feedback window opened, and feedback closing reminder, each with its own email template.
- **FR-004**: Committed teacher/student account creation MUST request an invitation; valid email-confirmation and password-reset workflows MUST explicitly request their respective messages, and an invitation MUST NOT implicitly trigger a separate email-confirmation message. Credential activation, confirmation, proof validity, and disclosure rules remain owned by the account workflows; sending an invitation alone MUST NOT activate an account. Existing teacher pending-account behavior remains until secure setup completes.
- **FR-005**: A committed results publication MUST notify only the students authorized to see that course's published results. Finalization without publication MUST NOT trigger a publication message. Messages MUST direct the recipient to authorized portal access rather than include student marks or other students' results.
- **FR-006**: A feedback-window opening MUST notify its eligible students. Closing reminders MUST be sent only while the window is still open and current, and each student MUST receive at most one logical reminder per window occurrence. Active Admins MUST be able to view and configure one positive portal-wide interval before closing. All windows MUST use that interval, with no per-window override. Invalid values MUST be rejected without changing the last valid setting. Reminder eligibility MUST use the current closing time and current interval; changing either MUST NOT repeat a reminder already issued. An Admin-approved interval MUST be configured before closing reminders are scheduled; no duration is assumed by this specification.
- **FR-007**: Feedback-notification recipients, content, timing, and history MUST NOT depend on or disclose whether a student submitted feedback. No feedback text, ratings, submission identifiers, or submission state may enter notification content, preview data, or history.
- **FR-008**: Notification delivery MUST execute in the background. Delivery/provider/template failures MUST NOT roll back a successful originating action or require that action to wait for a send or retry. Notifications for rejected/uncommitted actions MUST NOT be delivered.
- **FR-009**: Accepted notification work MUST survive process restarts and receive bounded background retries for retryable failures. Missing addresses, permanent provider rejections, and exhausted retries MUST have explicit actionable outcomes; no endless retry loop is permitted. Retry limits and intervals MUST be documented and validated during planning.
- **FR-010**: Failure to retain notification work MUST raise an operational failure signal and support reconciliation from the committed originating event, without failing that action. A provider outage MUST NOT silently lose notification requests.
- **FR-011**: Repeated processing of the same event/recipient/channel combination MUST NOT create additional logical notifications. Each delivery attempt MUST be traceable to that combination; uncertain provider acknowledgement MUST be represented honestly rather than promising exactly-once inbox delivery.
- **FR-012**: Each supported event/channel combination MUST have a template with a subject where applicable, body, enabled state, approved placeholders, and a traceable revision. Active Admins MUST be able to list/view/edit templates and enable or disable ordinary academic templates without a release. Invitation, email-confirmation, and password-reset templates MUST always remain enabled; attempts to disable them MUST be rejected without changing their state. Admins MAY edit their wording subject to required-content validation.
- **FR-013**: Template editing MUST validate approved placeholders and required content, including required action links for security messages. Unsupported fields, executable content, invalid syntax, and missing required values MUST be rejected without replacing the previous valid revision. Recipient/channel/privacy classification MUST NOT be editable through template wording.
- **FR-014**: Admins MUST be able to preview subject/body with clearly synthetic sample data. Preview MUST NOT send a notification or generate/display a live security link or proof. A security-template preview MAY show only a visibly synthetic, nonfunctional action-link placeholder; preview output MUST NOT be persisted. Preview MUST apply the same validation and privacy restrictions as real rendering.
- **FR-015**: Each accepted notification MUST capture its template revision and rendered content for consistent delivery/retry, subject to FR-018's security-content exception. Template edits affect newly accepted notifications, not already accepted executions. Disabled templates suppress new requests with a recorded reason; re-enabling MUST NOT automatically replay suppressed events.
- **FR-016**: Permanent history MUST record the originating event and occurrence identifier, stable recipient identity, delivery channel, actual destination address, template revision, request time, each attempt time/outcome, successful send time when applicable, and safe failure reason. Pending, retrying, sent, failed, suppressed, and uncertain outcomes MUST be distinguishable. Failed/suppressed records MUST NOT be presented as sent.
- **FR-017**: For non-security messages, history MUST preserve the exact subject/body submitted for each send and actual destination, unchanged by later template or profile edits. History and attempt facts MUST NOT be editable or deletable through the portal and MUST survive recipient removal; no automatic expiry or cleanup is permitted in this feature.
- **FR-018**: Messages containing security links, including invitation, email confirmation, and password reset, MUST NOT persist rendered content, subject/body snapshots, links, or proofs in permanent history, ordinary logs, or failures. Previews MUST NOT generate, display, or persist live security links, proofs, or real security-message content; a preview MAY display only a visibly synthetic, nonfunctional action-link placeholder. History MUST show safe delivery metadata and an explicit content-withheld indication. Only protected transient delivery work may retain necessary secrets until expiry/completion, after which they MUST be removed; this exception MUST NOT create a permanent content archive.
- **FR-019**: Active Admins MUST be able to search history using recipient identity, event type, delivery status, and a UTC date range, combined predictably. Date filtering uses request time with a start-inclusive/end-exclusive range; details also show individual attempt/send times. Long results MUST follow the shared infinite-loading standard in [the design system](../design-system.md), not numbered UI pagination.
- **FR-020**: Admins MUST be able to request resend only for a failed notification, with confirmation and a reason when ineligible. Resend MUST preserve the old record, append a linked execution, and run in the background; pending/in-flight/sent/suppressed records MUST NOT be resent as failed. Repeated confirmation while the resend is pending MUST NOT create duplicate executions.
- **FR-021**: An ordinary resend MUST use the recorded destination and content, and be refused if the account is no longer eligible or that address no longer belongs to the intended recipient. Security-message resend MUST revalidate the owning workflow and generate a fresh valid link rather than replay expired secrets; history remains content-free. If fresh authorization cannot be established, refuse resend and direct the Admin to the owning workflow.
- **FR-022**: Only sessions actively using the Admin role MUST access template management, previews, history, resend, or the portal-wide reminder setting, including direct requests outside the UI. Template changes, reminder-setting changes, and resend requests MUST use the existing audit capability with safe actor/action/target/outcome details, never security content or submitted feedback.
- **FR-023**: Each notification MUST be addressed independently so one recipient cannot see other recipients' addresses or personal information. Rendering and failure responses MUST safely handle supplied values and MUST NOT expose credentials, provider secrets, or internal diagnostics.
- **FR-024**: The initial feature MUST supply the event catalog/templates and notification-side behavior for every listed trigger, wire existing account flows, and define acceptance contracts for workflows not yet implemented. Creating student-management, course-publication, or feedback-collection workflows is outside scope. Missing producers MUST be explicitly tracked by owner; their end-to-end notification integration MUST NOT be reported complete until the owning workflow triggers it and passes its acceptance tests.
- **FR-025**: Notification administration MUST provide loading, empty, validation, preview, success, disabled, failure, retry, and end-of-history states consistent with the shared IIT design system. No user-facing notification preference center, bulk broadcast composer, or provider administration console is included.
- **FR-027**: Every email MUST be sent in the standard IIT-branded layout (logo with alt text, the subject as heading, the message, and a footer) together with its plain-text version. The layout MUST be built in memory at send time from the plain-text subject and body, so it is never stored and never changes what an Admin edits or what history keeps; every supplied or configured value MUST be encoded so it can never become markup (the brand color is accepted only as a six-digit hex value dark enough for white button text, and the logo address only as an absolute https address, both checked at startup); only whole http(s) addresses may become links; and previews MUST show the same layout with sample data only (a secure message's link remains a visibly nonfunctional placeholder, never a link).
- **FR-026**: Brevo MUST be used for email delivery in Development only and MUST NOT be enabled in Production. Email delivery MUST be replaceable behind a provider-neutral boundary; changing providers MUST NOT require changes to notification producers or their event requests. Development messages MUST use synthetic/test identities and MUST NOT contain real user credentials or personal data.

### Key Entities *(include if feature involves data)*

- **Notification event**: A supported business occurrence, its stable identifier, safe context, and intended recipients; channel independent.
- **Notification template and revision**: Event/channel-specific approved subject/body wording, placeholders, enabled state, and revision identity.
- **Notification record**: Permanent recipient/channel/destination/event metadata, lifecycle status, template reference, and non-security content snapshot; security records explicitly omit content.
- **Delivery attempt**: An immutable execution fact with attempt time, outcome, safe failure classification, and send time when known.
- **Resend relationship**: Links a new authorized delivery execution to its failed predecessor while retaining both histories.
- **Feedback reminder eligibility**: Window occurrence, eligible students, closing time, reminder due time, and cancellation/rescheduling state; excludes submitted-feedback data.
- **Reminder policy**: One Admin-managed portal-wide positive interval before a feedback window closes, applied consistently across windows without per-window overrides.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Every supported event produces one logical notification per eligible recipient/channel or an explicit suppressed/failed outcome; replaying an event creates zero extra logical notifications.
- **SC-002**: Under a simulated email-service outage, 100% of tested originating actions retain their normal success/failure outcome; their 95th-percentile completion time increases by no more than 200 milliseconds compared with notifications disabled under the same workload.
- **SC-003**: With email delivery healthy, at least 95% of accepted immediate notifications are submitted for delivery within 60 seconds. Reminder submission follows the approved reminder schedule rather than event-acceptance time.
- **SC-004**: For 100% of ordinary sent-message test cases, history reproduces the destination and exact submitted subject/body after template/account edits. For 100% of security-message cases, no live security-link values or proofs appear in permanent history, previews, or ordinary logs; security previews show only visibly synthetic, nonfunctional link placeholders.
- **SC-005**: All tested non-Admin and anonymous direct administration requests are denied without revealing history or template data; 100% of successful template changes and resend requests have a safe audit record.
- **SC-006**: In outage/restart tests, all accepted notification work is recovered to a terminal or explicitly uncertain outcome; no lost work or retry beyond the approved limit occurs. A failed resend retains all predecessor attempts and produces exactly one linked execution per accepted resend request.
- **SC-007**: At least 90% of representative Admins can edit and preview a valid template or locate a specified failed notification within two minutes without developer assistance or a release.
- **SC-008**: Changing any synthetic student's feedback submission content/state causes zero changes in notification eligibility, rendered content, or exposed history. Closed/cancelled-window tests send zero obsolete reminders. All tested windows use the configured portal-wide interval, and interval changes create zero duplicate reminders for a window/recipient already notified.
- **SC-009**: A representative existing feature can trigger its unchanged event through a test additional channel without changing that feature's notification request; production delivery remains email-only until that channel is separately approved.
- **SC-010**: A representative notification event can be delivered through the Development email provider and a substitute Production provider without changing its producer request; Production startup/configuration rejects the Development-only provider.

## Assumptions

- The portal reuses existing authentication, active-role authorization, credential rules, secure account proofs, and safe business/security audit conventions.
- The shared notification capability is the follow-up email mechanism previously deferred by Teacher Management. This specification does not change existing account activation rules or grant sign-in access merely because an email was sent.
- Event-owning workflows supply authoritative event identity, recipients, and safe academic/window context. Missing student, publication, and feedback producers are future integration dependencies, not permission to invent those workflows here.
- Retry limits/intervals are approved in the plan. Brevo is the Development-only email provider; all Development delivery uses synthetic/test identities and data. A different provider MUST be selected and verified before Production. Provider choice is isolated behind the delivery adapter. Sent means accepted by the configured service, not guaranteed inbox receipt or user reading; tracking opens, clicks, and delivery webhooks are not required.
- Brevo's transactional API exposes an endpoint that can return sent message content. Development MUST NOT send real user credentials or personal data through Brevo. Before Production, the replacement provider's content retention/access behavior MUST be verified against FR-018; the production deployment MUST NOT use Brevo unless it is separately approved as compliant with that requirement.
- Template revision and destination are fixed when notification work is accepted, subject to recipient eligibility checks before security sends/resends. Disabled templates affect future requests; queued messages keep their accepted revision.
- Permanent history includes personal destination addresses as explicitly requested and is Admin-only. No retention duration or deletion policy is inferred; any later policy change requires separate approval.
- A reminder goes to every eligible student regardless of submission state, preventing notifications/history from revealing feedback participation.
- An Admin configures the shared closing-reminder interval before reminders are scheduled. This feature supplies no fixed duration and no per-window timing option. Security-message templates remain enabled regardless of ordinary academic template settings.
- The branded layout is fixed by the system; only its branding values are configurable (defaults: the IIT logo and brand color from the design system, a plain footer). Its logo is loaded from a public HTTPS address; in Development, where the portal is not reachable from the internet, the logo will not display for a real recipient and the portal name is shown as text if no usable address is configured.
- Initial wording is one institutional language per event/channel. Localization, SMS delivery, additional channels, preference/opt-out management, and new academic workflows are outside this feature.