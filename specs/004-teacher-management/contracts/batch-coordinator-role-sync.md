# Batch Coordinator Role Synchronization Contract

**Status**: Internal application integration contract for [Teacher Management](../spec.md); not a public HTTP API.

## Caller

The application use case that creates, changes, or removes a teacher's coordinator assignment for a batch. No such assignment model or workflow exists in the current service. The future owning feature must call this contract in the assignment transaction after computing authoritative post-change state and before commit.

## Operation

```text
SynchronizeCoordinatorRoleAsync(teacherUserId, hasActiveCoordinatorAssignments, cancellationToken)
```

- `teacherUserId` identifies the existing Identity account linked to the TeacherProfile.
- `hasActiveCoordinatorAssignments` is the caller's authoritative post-change state: true if at least one active batch-coordinator assignment remains; false otherwise.
- The caller invokes the synchronizer in the same unit of work as the assignment change so the assignment and role membership cannot diverge; it must not commit the assignment if synchronization fails.

## Behavior

- If true, ensure the user has Coordinator.
- If false, remove Coordinator if present.
- Never remove the required Teacher role, assign Student, or change Admin membership.
- If removing Coordinator invalidates the stored default role, resolve a valid assigned role using the existing `PortalRoles.ResolveDefault` convention.
- Stage an allowlisted `teacher.coordinator-role.synchronized` business audit event in the caller's unit of work; do not record batch or student personal data beyond approved identifiers.
- The operation is idempotent: repeated calls with the same user and assignment state do not create duplicate role changes or audit events.
- Reject unknown users and active teacher-profile inconsistencies without partially changing Identity roles.

## Implemented Shape

```csharp
// IitAcademicPortal.Application.Teachers
Task<CoordinatorSyncOutcome> ITeacherCoordinatorRoleSynchronizer.SynchronizeCoordinatorRoleAsync(
    string teacherUserId, bool hasActiveCoordinatorAssignments, CancellationToken cancellationToken);
// CoordinatorSyncOutcome: Unchanged | Added | Removed
```

- The synchronizer **stages** the Coordinator role row, any default-role correction, and the `teacher.coordinator-role.synchronized` audit event on the request's `PortalDbContext` and does **not** save. The assignment workflow saves its own assignment change in the same `SaveChanges` (ideally inside its own transaction), so the role, the audit event, and the assignment commit together or not at all.
- It throws `CoordinatorSyncException` (nothing staged) for an unknown user, an account with no teacher profile, or `hasActiveCoordinatorAssignments = true` for a deactivated teacher. Removing Coordinator from a deactivated teacher is allowed.
- It is registered in dependency injection as a scoped service. The audit actor is the signed-in account of the request, or null outside a request.

## Verification Boundary

Teacher Management tests the synchronization service directly for true/false states, multiple assignments, idempotency, default-role correction, and audit rollback. This does not verify an assignment producer. The batch-coordinator assignment owner must add tests proving every create/change/end operation calls the synchronizer in the same unit of work, and must pass those tests before claiming the portal-wide automatic-sync requirement is complete. End-to-end producer tests are deferred until that owner exists.
