# Authorization Model

## 7. Authorization Model

### 7.1 General Pattern
Authorization rules may match records in two ways:
- by checking direct user reference fields on the entity
- by checking whether one of the user's effective `groupMemberships` matches the entity's group field

The second path is what makes groups part of the authorization substrate.

### 7.2 Rule Shape
Current group-based permission definitions in the codebase specify concepts like:
- `field`: entity field containing the relevant group identifier, or the `groupId` component of an explicit group-job reference
- `jobField`: optional entity field containing an explicit target `jobId` when the entity is job-specific
- `flags`: required group capabilities for that entity family
- `timeField`: entity time field used for overlap checks when relevant
- `defaultAgainst`: fallback behavior when no match applies

These rules are implemented in the backend permission layer.

### 7.3 Runtime Membership Query
At runtime, a group-based check resolves candidate `groupMemberships` for the acting user by filtering on:
- `memberId`
- `jobId` when required by the permission or carried explicitly by the entity
- group flags through denormalized `_flags`
- time overlap between membership interval and entity interval/date

Then the matching entity query is constrained by the resulting eligible `groupId` set.

If the entity stores an explicit `GroupJob` reference, there is no implicit attendee/default fallback: both `groupId` and `jobId` are part of the meaning of the link.

This is one of the reasons `GroupPart` must stay separate from `Group`:
- permissions are evaluated on the reusable `Group`
- membership contribution is produced through a job-specific `GroupPart`
- for the same reusable group, `supervisor` and `attendee` memberships may lead to different permission sets
- so a single reusable group id cannot be identified with a single taxonomy part id

### 7.4 Why Flags Matter
`Group.flags` are not decorative metadata.

They exist primarily to constrain authorization query scope.

Without flag-based filtering:
- the backend would need to consider all memberships held by a user for the relevant job/time window
- the candidate set of `groupId`s would grow with every additional membership
- downstream entity queries would become much broader and more expensive

With flag-based filtering:
- only groups explicitly marked as relevant for that entity family are considered
- authorization cost scales with relevant memberships instead of total memberships

This is why `makeGroupMembershipFlagsStream` denormalizes flags into `groupMemberships._flags`.

### 7.4.1 The `searchable` Flag
`searchable` is a flag that marks a group as eligible to contribute to the `User.memberIn` denormalized capture index.

Semantic role:
- only `searchable`-flagged groups appear in `User.memberIn`, keeping the index small and query-relevant
- the tree-view UI and other member-search surfaces rely on this index to quickly resolve which groups a user belonged to at a given capture snapshot
- the flag does not affect authorization checks; it only governs the `memberIn` denormalization pipeline

Capture type filtering:
- not all capture types are relevant for search surfaces; for example, the tree-view typically only needs `schoolYear_end` captures
- `User.memberIn` is therefore keyed by `captureId` rather than by `groupId`, so consumers can select the right capture granularity for their surface

### 7.5 Time Semantics
Current permission checks use overlap logic on `starts` / `ends`.

Representative pattern:
- membership starts before or at the entity boundary
- membership ends after or at the entity boundary

This works for many current cases, but it is still an interval-overlap query over the `groupMemberships` read model.

### 7.6 Current Limitation
The current authorization path does not first prune by period partition.

That means the main runtime filter remains an overlap predicate on `starts` / `ends`, which is harder to scale efficiently than direct equality or prefix lookups.

This limitation is documented here because it affects the operational role of time in the model:
- time is not only semantic state
- time is also part of how runtime authorization cost is shaped

### 7.7 Lesson-Schedule Memberships
Some memberships do not originate from taxonomy history.

In particular, lesson schedule supervisors are materialized directly into `groupMemberships` with `source = 'lessonSchedule'`.

Authorization therefore depends on the unified read model rather than on replaying taxonomy logic.
