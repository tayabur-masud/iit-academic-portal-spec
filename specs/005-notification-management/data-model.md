# Data Model: Notification Management

**Feature**: [spec.md](spec.md)  
**Status**: Logical model for planning; confirm the current service schema and PostgreSQL grants before implementation.

## Modeling Decisions

- Use the existing `PortalDbContext`, PostgreSQL database, and EF Core migration chain. Do not add a separate database, broker, or package.
- Keep notification dispatch state separate from audit events. Audit history records administrative actions; notification history records delivery facts.
- Producers supply a stable source occurrence ID and recipient identity without naming a channel. The dispatcher expands enabled event/channel templates; initially only Email exists.
- Persist accepted ordinary subject/body snapshots. Never persist security-message rendered content, links, or proofs in permanent records, attempts, audit events, or ordinary logs.
- Keep immutable notification history and delivery attempts separate from mutable worker dispatch state. The history API projects current status from the dispatch state; workers never update history or attempt facts.
- Store recipient user IDs as stable values without cascading Identity foreign keys so records survive account removal. Store the actual destination used for each accepted execution.
- Do not add automatic deletion/expiry for permanent notification records, attempts, or template revisions. Protected transient recovery input is deleted after processing or expiry.
- A version-controlled migration is required. Update runtime grants for new tables/sequences; portal routes cannot update/delete history or attempts.

## Implemented Mapping

Tables: `NotificationTemplates`, `NotificationTemplateRevisions`, `NotificationRecords`, `NotificationDispatchStates`, `NotificationAttempts`, `PasswordRecoveryRequests`, `FeedbackReminderPolicies`, `FeedbackReminderSchedules` (migration `NotificationManagement`, which also seeds the seven templates and their first revisions with fixed identifiers).

- Templates, revisions, and the reminder policy derive from `AuditedEntity` (who and when come for free; a revision's creator is the Admin who saved it). Records, dispatch states, attempts, transient requests, and schedules are exempt in `AuditedEntityConventionTests`, each with its reason.
- `NotificationTemplates.CurrentRevisionId` is a plain column, not a foreign key, so a template and its first revision can be seeded together. Records reference revisions, and attempts and dispatch states reference records, all restrictively; no recipient column is a foreign key.
- The original-execution unique index is `(EventType, OccurrenceId, RecipientUserId, Channel)` where `ParentNotificationId IS NULL`; a resend is unique on `(ParentNotificationId, RequestIdempotencyKey)` where the parent is not null.
- `NotificationDispatchStates` also holds `SendStartedAtUtc` (set just before the provider call), `SentAtUtc`, `FailureCode`, and `UpdatedAtUtc`. A lease that expires while `SendStartedAtUtc` is set means acceptance is unknown, so the work becomes `Uncertain` (attempt code `WorkerInterrupted`) instead of being sent again. When retries run out the status is `Failed` with code `RetriesExhausted`; the attempts keep the underlying codes.
- `PasswordRecoveryRequests.ExpiresAtUtc` comes from `PasswordRecovery:RequestLifetime`, separate from the proof lifetime. Rows are deleted by key once processed or expired (the permanent history is never deleted).
- The runtime role cannot update, delete, or truncate `NotificationRecords`, `NotificationTemplateRevisions`, or `NotificationAttempts`; it can update the other tables.

## Logical Entities

### Notification Template

Stable event/channel definition containing event type, channel, current revision reference, and enabled state. `(EventType, Channel)` is unique. Security event templates (teacher/student invitation, email confirmation, password reset) are constrained to enabled. Ordinary academic templates may be disabled. Recipient, privacy, and event classification are not editable.

### Notification Template Revision

Immutable, monotonically numbered revision for one template. Stores subject, plain-text body, approved placeholder names, creator identity, and creation time. Each accepted notification references the revision used. Concurrent Admin edits supply the revision they observed; stale revisions are rejected.

### Notification Record

One logical delivery execution for a source event, recipient, and channel.

| Attribute | Meaning and constraints |
|---|---|
| Notification ID | Stable unique identifier. |
| Event type / occurrence ID | Business trigger and producer-owned stable occurrence identity. |
| Recipient user ID | Stable identity value; not a cascading foreign key. |
| Destination address | Actual address fixed for this accepted execution. |
| Channel | Initially `Email`. |
| Template revision ID | Required immutable revision reference. |
| Requested time | UTC acceptance time; used for history date filtering. |
| Ordinary content snapshot | Exact submitted subject/body for non-security content only; null for security content. |
| Content-withheld flag | True for security messages and exposed explicitly in Admin detail. |
| Parent notification ID | Optional failed predecessor for a manual resend; original record remains intact. |
| Request idempotency key | Unique for one resend command and stable across retries of that command. |

Current delivery status is projected from the separate one-to-one dispatch-state row; mutable retry and lease data are never stored in this immutable record.

The database enforces uniqueness of `(EventType, OccurrenceId, RecipientUserId, Channel)` for original event executions and `(ParentNotificationId, RequestIdempotencyKey)` for accepted resends. Resends are new records, never edits to predecessors.

### Notification Dispatch State

Mutable one-to-one operational state for a notification record. Stores current status (`Pending`, `Retrying`, `Sent`, `Failed`, `Suppressed`, or `Uncertain`), attempt count, next-attempt time, lease expiry, and safe failure code. The worker may insert/update this row to claim work and advance lifecycle state; it must not update/delete the notification record, template revision, or delivery-attempt facts. The Admin API reads this state to present current status and has no mutation route for it.

### Delivery Attempt

Append-only fact related to one notification. Stores attempt number, UTC start/completion time, outcome (`Accepted`, `RetryableFailure`, `PermanentFailure`, or `Uncertain`), safe failure classification, and provider-accepted time when known. It never stores proofs, rendered security subject/body, provider credentials, or raw provider exceptions. Provider acceptance does not guarantee inbox delivery.

### Transient Password-Recovery Request

Durable, short-lived queue input preserving the existing generic response for known and unknown addresses. Store the submitted address only in protected form, with request time, expiry, and lease/processing state. Resolve the account before creating a permanent notification record; unknown/ineligible addresses create no searchable history. Delete transient payload after processing/expiry. Do not store reset proofs; generate them in memory for authorized delivery attempts.

### Feedback Reminder Policy

One portal-wide singleton containing a positive whole-minute lead time, revision, and safe audit metadata. No initial duration is inferred: closing reminders remain unscheduled until an Admin saves a valid value. No per-window override exists.

### Feedback Reminder Schedule

Mutable pre-dispatch scheduling state keyed by window occurrence and eligible recipient. Stores only the occurrence ID, recipient ID, current closing time, current policy revision, due time, and `Scheduled`/`Cancelled`/`Dispatched` state; it never stores or references feedback submission data. Recompute the due time for the same still-open occurrence whenever its close time or the global policy changes. Cancel when the window closes, is cancelled, or is superseded. Create the permanent reminder notification only when the current schedule becomes due; the notification idempotency key then prevents duplicate dispatch. If a reminder was already issued, policy or deadline changes never schedule another for that occurrence/recipient.

## Relationships and Invariants

- One template has many immutable revisions; each notification references exactly one revision.
- One immutable notification record has exactly one mutable dispatch-state row and zero or more immutable delivery attempts; one resend may reference one failed predecessor.
- The event/occurrence/recipient/channel key prevents replay from creating another original notification. Resends use their own idempotency key and predecessor relationship.
- Worker claims and lifecycle changes update only dispatch state. Notification records, template revisions, and delivery attempts are append-only; history search joins dispatch state to show current status.
- Template revision and destination are fixed when work is accepted. Ordinary content is rendered and snapshotted at acceptance. Security rendering/proofs exist only in worker memory during delivery.
- Only retryable failures below the bounded limit are retried. Possibly accepted provider work becomes `Uncertain`, not `Sent` or an automatic retry.
- Successful Admin template, policy, and resend mutations stage safe audit events through the existing audit unit of work. Audit excludes message content, proofs, and feedback data.
- Feedback eligibility derives only from window occurrence/status/close time and current global lead time. Persist schedule and notification occurrence/recipient facts, never submission state or links to feedback submissions.
- Close-time or policy changes recompute a not-yet-dispatched schedule for the current open occurrence; closure/cancellation/superseding occurrence cancels it. Occurrence/recipient uniqueness prevents policy changes from repeating an issued reminder.
- Permanent history has no account foreign key that can cascade-delete it. The portal exposes no update/delete operations for records or attempts.

## State Transitions

1. **Pending**: accepted enabled-template request has durable work and no completed attempt.
2. **Retrying**: retryable failure is recorded and a bounded next attempt is scheduled.
3. **Sent**: delivery service accepted the message; safe metadata and ordinary content snapshot remain.
4. **Failed**: missing recipient, permanent rejection, or exhausted retries; eligible failure may be resent by an active Admin.
5. **Suppressed**: ordinary template was disabled when accepted; no send occurs, and re-enabling does not replay it.
6. **Uncertain**: provider acceptance cannot be determined; do not claim delivery or retry automatically. Spec permits Admin resend only for `Failed`.

## Indexes and Migration Impact

- Unique event-occurrence/recipient/channel key for original idempotency.
- History keyset index on `(RequestedAtUtc DESC, NotificationId DESC)` and supporting indexes for status, event, recipient identity, and request-time range.
- Due-work index on `(Status, NextAttemptAtUtc, LeaseExpiresAtUtc)` for worker claims.
- Unique `(TemplateId, RevisionNumber)`; restrictive references from records to revisions and attempts to records.
- Index transient request expiry/lease for bounded processing; remove its protected payload on completion or expiry.

Add schema through EF Core migration and update `database/postgresql/runtime-grants.sql`. The runtime role may update dispatch-state rows for worker claims/status transitions, but receives no update/delete privilege on notification records, template revisions, or delivery attempts. PostgreSQL tests validate migration, uniqueness, keyset filtering, lease recovery, append-only access, and these separated runtime permissions; SQLite does not prove provider-specific behavior.
