# Notification Management API Contract

**Status**: Implemented contract for [spec.md](../spec.md). Browser-facing routes use explicit DTOs; event producers use [internal producer contracts](notification-producer-contracts.md).

## Common Behavior

- Every route requires an authenticated session whose active role is Admin through the existing `PortalPolicies.AdminModule`. Anonymous requests receive 401; any non-Admin active role receives 403. UI visibility is not authorization.
- State-changing requests require `X-CSRF-Token`. Template, reminder-policy, and resend mutations stage safe audit events through the existing audit capability.
- Validate server-side and use Problem Details. Never return provider diagnostics, proof material, submitted feedback, rendered security content, or internal exceptions.
- Timestamps are UTC. Date-range lower bounds are inclusive and upper bounds exclusive.
- History is append-only through portal routes. `Sent` means the delivery service accepted the message, not that it reached or was read in the recipient's inbox.

## Routes

| Method and route | Contract |
|---|---|
| `GET /api/admin/notification-templates` | List seven event/email templates, current revision, enabled state, and approved placeholder names. No proof values. |
| `GET /api/admin/notification-templates/{eventType}/email` | Return the current revision for one event. |
| `PUT /api/admin/notification-templates/{eventType}/email` | Body: `expectedRevision`, `subject`, `body`, `enabled`. Validate strict placeholders, required action links, plain-text syntax, and security rules. Save creates an immutable revision. Stale revision returns 409; invalid content returns field-keyed 400. Disabling a security template returns 400 without changing state. |
| `POST /api/admin/notification-templates/{eventType}/email/previews` | Body: draft `subject`, `body`, and synthetic `sampleValues`. Return rendered subject/body after validation; security templates may show only a visibly synthetic, nonfunctional action-link placeholder. Does not send, resolve a live account, generate or return a live link/proof, or persist preview output. |
| `GET /api/admin/notifications` | Query: optional `recipient` (stable ID or recorded destination), `eventType`, `status`, `fromUtc`, `toUtc`, `pageSize` (default 25, range 1-100), and opaque `cursor`. Return bounded rows ordered by request time descending then ID descending, with `nextCursor`; no numbered pages or total count. |
| `GET /api/admin/notifications/{notificationId}` | Return safe metadata and immutable attempts. Ordinary detail includes actual destination and exact submitted subject/body. Security detail omits subject/body/proof/link and returns `contentWithheld: true`. |
| `POST /api/admin/notifications/{notificationId}/resends` | Empty body; Admin confirmation is the request. Require `Idempotency-Key`. Accept only `Failed`. Create a linked background execution without changing predecessor attempts. Ordinary resend uses recorded destination/content after current eligibility validation; security resend revalidates the owner and generates a fresh proof. Ineligible, uncertain, pending, sent, or suppressed states return a safe 409 reason. |
| `GET /api/admin/notification-settings/feedback-reminder` | Return configured positive `leadTimeMinutes`, or explicit unconfigured state. |
| `PUT /api/admin/notification-settings/feedback-reminder` | Body: positive integer `leadTimeMinutes`. No default is inferred. Invalid values leave the last valid policy unchanged; successful changes are audited. |

## Response Shape

Notification collection response:

```json
{
  "items": [
    {
      "id": "uuid",
      "eventType": "ResultsPublished",
      "recipientUserId": "stable-id",
      "destination": "student@example.test",
      "channel": "Email",
      "status": "Sent",
      "requestedAtUtc": "2026-10-15T12:00:00Z",
      "sentAtUtc": "2026-10-15T12:00:02Z",
      "contentWithheld": false
    }
  ],
  "nextCursor": null
}
```

Ordinary detail adds `subject` and `body`. Security detail has `contentWithheld: true` and omits both fields. Attempt details include timestamps, safe outcome codes, and provider-accepted time when known; they never include raw provider errors.

## Implemented Details

- `{eventType}` in a route is the kebab-case slug (`password-reset`, `results-published`, `feedback-closing-reminder`, ...) or the enum name (`PasswordReset`); an unknown value is 404. In JSON, `eventType` is the enum name and `slug` is also returned.
- **Templates**: each item has `eventType`, `slug`, `name`, `channel`, `isSecurity`, `enabled`, `revision`, `subject`, `body`, `placeholders`, `requiredPlaceholders`, `updatedAtUtc`. `PUT` requires `expectedRevision`, `subject`, `body`, and `enabled` (missing ones are 400). A save that changes nothing returns 200 with no new revision and no audit event; a change that only flips `enabled` keeps the revision number. Wording rules: plain text only; a placeholder is `{{name}}` from the event's own list; markup (`<b>`, comments) and expression syntax (`{%`, `${`, `<%`) are rejected; a security message must include `{{actionLink}}`; the subject is a single line of at most 200 characters and the body at most 20,000.
- **Preview** returns `{ subject, body, html, synthetic: true }`: the plain text, and the same message in the branded email layout (every value encoded, built from sample data, never stored). A secure message's link is never a link in either version. `sampleValues` may name only the event's placeholders (each a single line of at most 200 characters). For a security event the link is always the fixed text `[SAMPLE LINK - not functional]`, whatever is supplied.
- **History** returns `{ items, nextCursor }` only. `recipient` matches the stable user ID exactly or the recorded destination ignoring case; `eventType` accepts the slug or name; `status` is `Pending`, `Retrying`, `Sent`, `Failed`, `Suppressed`, or `Uncertain`; `fromUtc` is inclusive and `toUtc` exclusive, and an empty or reversed range is 400 on `toUtc`. Each item has `id`, `eventType`, `recipientUserId`, optional `destination`, `channel`, `status`, `requestedAtUtc`, optional `sentAtUtc`, `contentWithheld`, optional `parentNotificationId`, and optional `failureCode` (a short classification, never provider text).
- **Detail** returns `{ notification, occurrenceId, templateRevision, subject?, body?, attempts[] }`; `subject` and `body` are present only for an ordinary message.
- **Resend** returns **202** with the new notification's detail and a `Location`, for the first request and for any repeat with the same `Idempotency-Key`. A missing or malformed key (1 to 128 of letters, digits, `. _ : -`) is 400 on `idempotencyKey`. The 409 `code` is one of `InProgress`, `AlreadySent`, `Suppressed`, `DeliveryUncertain` (the record's own status), `ResendInProgress`, `AlreadyResent` (a resend already exists), `RecipientIneligible`, `DestinationChanged`, `NoDestination`, or `OwnerWorkflowUnavailable` (a secure message whose owning workflow cannot authorize it, which today includes every invitation).
- **Reminder setting**: `GET` returns `{ configured, leadTimeMinutes? }`; `PUT` takes `{ leadTimeMinutes }`, a whole number from 1 to 525,600 (a sanity bound of one year; the specification sets none), and any other value, a non-integer, or an unknown field is 400 with the last valid value kept.
- Unknown JSON fields are rejected everywhere (400). There is no route that updates or deletes a notification, a revision, an attempt, or a per-window reminder setting.

## Errors

- `400`: invalid filter range, template, placeholder, policy value, or request shape; use field-keyed Problem Details where applicable.
- `401` / `403`: unauthenticated or active role is not Admin; do not return restricted data.
- `404`: unknown notification/template resource.
- `409`: stale template revision, duplicate/ineligible resend, or non-resendable status; return a stable safe reason code.

Follow the existing ASP.NET Core OpenAPI and Problem Details conventions. Final schema names/error extensions are verified alongside implementation tests.
