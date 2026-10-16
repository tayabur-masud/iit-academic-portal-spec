# Notification Producer Contracts

**Status**: Implemented internal application contracts and acceptance obligations for [spec.md](../spec.md). These are not public HTTP endpoints.

## Request Contract

Each owner requests a channel-independent event type, stable source occurrence ID, stable recipient identity (or identity-owned address lookup), and allowlisted safe context. Requests contain no channel choice, feedback submission data, credentials, raw proofs, or provider configuration. The dispatcher selects enabled channel/template; the initial catalog supports Email only.

Occurrence and recipient identity remain stable across retries and reconciliation. The dispatcher enforces one original logical notification per occurrence, recipient, and channel. Each producer supplies authoritative replay/reconciliation for a committed source occurrence. Store failure raises a secret-free operational signal and does not roll back the owning action.

## Event Catalog

| Event | Producer responsibility | Safe context and recipient rule | Acceptance contract |
|---|---|---|---|
| Teacher invitation | Existing teacher provisioning workflow after account commit | Stable user ID and eligible address; no password handoff | Account creation succeeds during provider failure; retry/replay creates no duplicate. End-to-end delivery is gated on the account owner supplying a secure initial credential-setup proof/link contract. |
| Student invitation | Future Student Management owner | Stable user ID; owner establishes eligibility and address | Contract test now; end-to-end completion waits for the owner and its credential-setup contract. |
| Email confirmation | Owning account workflow explicitly requests it; invitation alone does not trigger it | Stable user ID and confirmation purpose; owner supplies proof generation for delivery | Preserve current disclosure/proof rules; no proof in permanent history. Current source has no confirmation producer, so end-to-end completion waits for its owner. |
| Password reset | Existing password-recovery workflow | Protect accepted address transiently until worker lookup; unknown account creates no permanent notification record | Same public response for known/unknown addresses; durable work, bounded retry, safe proof generation, no proof/content in history/logs. |
| Results published | Future course/results owner after committed publication | Authorized student IDs and safe course/term labels; no marks or other students' results | Finalization without publication emits nothing; only eligible students receive requests. End-to-end completion waits for owner. |
| Feedback window opened | Future feedback-window owner after committed opening | Window occurrence, course/window labels, eligible student IDs; no submission data | Opening message does not depend on submission state. End-to-end completion waits for owner. |
| Feedback closing reminder | Scheduler using authoritative current window state | Window occurrence, current close time, global lead time, eligible students; no feedback data | Due at current close minus lead time; send only while current/open; once per recipient/window occurrence; close/cancel/reschedule invalidates obsolete work. |

## Implemented Shape

```csharp
// IitAcademicPortal.Application.Notifications
Task<NotificationRequestResult> INotificationRequestService.RequestAsync(
    NotificationEventRequest request, CancellationToken cancellationToken);
// NotificationEventRequest(NotificationEventType EventType, string OccurrenceId,
//                          IReadOnlyList<string> RecipientUserIds, IReadOnlyDictionary<string,string>? Context = null)
// NotificationRequestResult: Status (Accepted | Invalid | PersistenceFailed), Accepted, Duplicates, Suppressed, Failed counts

// Owners of results publication and feedback windows (contract-only today):
IAcademicNotificationProducers.ResultsPublishedAsync(occurrenceId, courseLabel, termLabel, authorizedStudentUserIds, ct)
IAcademicNotificationProducers.FeedbackWindowOpenedAsync(occurrenceId, courseLabel, windowLabel, closeAtUtc, eligibleStudentUserIds, ct)
// Reminders: the owner implements IFeedbackWindowSource.GetWindowsAsync; FeedbackReminderScheduler does the rest.

// A security event's owning workflow:
ISecurityContentProvider { EventType; CheckAsync(userId, destination, ct); PrepareAsync(userId, destination, ct) }
```

- The request carries no channel, provider, address, wording, or secret. Ordinary `Context` keys are limited to `courseName`, `termName`, `windowName`, and `closesAtUtc`; any other key makes the request `Invalid` and stores nothing. `portalUrl` is added by the service.
- Addresses are resolved when the work is accepted (`IRecipientDirectory`) and fixed for that execution. A recipient with no address is recorded as Failed (`NoDestination`), a deactivated account as Failed (`RecipientIneligible`).
- `RequestAsync` never throws. If the work cannot be retained it raises `INotificationFailureSignal.PersistenceFailed` (identifiers only) and returns `PersistenceFailed`; the owner replays the same occurrence to reconcile.
- A security provider's `PrepareAsync` runs once per delivery attempt and its values live only in memory. `Ineligible` (account removed, address changed) and `Unavailable` (no secure-link provider) make the notification Failed with that code and send nothing.
- The feedback window source reports only the occurrence, labels, close time, status, and eligible students; the scheduler cannot read submissions. A reminder is due at the current close time minus the global lead, is sent only while the window is open and its close time has not passed, and is issued at most once per occurrence and student.

## Existing Password-Recovery Handoff

The existing password-recovery request keeps its public generic response for known and unknown addresses, but its in-memory queue/worker must no longer send directly. Accepted requests enter the durable notification path once; the worker resolves account eligibility and obtains a fresh Identity proof only for an authorized delivery attempt. No parallel legacy sender may remain. Replay tests verify one logical delivery, unchanged public responses, and no persisted proof or rendered security content.

## Transaction and Failure Boundary

- The owner commits its business action independently of the email provider and delivery retries.
- After commit, persist the notification request durably. If persistence fails, do not fail the successful action; raise an operational alert and reconcile by replaying the stable occurrence.
- Owners retain authoritative facts to rebuild the same occurrence ID, eligible recipients, and safe context. Do not infer a delivery request from an audit event alone.
- Existing account workflows retain account-status, proof, anti-enumeration, and session rules. Sending an invitation never activates an account.
- The feature may wire teacher provisioning and password recovery. Confirmation, student invitation, publication, and feedback producers absent from source remain explicit dependencies. Contract tests do not claim end-to-end integration.

## Delivery and Privacy Rules

- Address each recipient separately; no recipient sees another's address or information.
- Ordinary content is snapshotted at acceptance and remains pinned to its revision/destination.
- Generate security proofs only for an authorized delivery attempt and keep them in memory. Preview may return a visibly synthetic, nonfunctional action-link placeholder; it never calls an owner proof generator or persists its output.
- Feedback eligibility, recipients, timing, content, and history depend only on window occurrence and the owner's eligible-recipient list, never feedback submission state.
- `Sent` means accepted by the configured email provider. Ambiguous provider acceptance becomes `Uncertain`, not `Sent`, and is not automatically retried.
