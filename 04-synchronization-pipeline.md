# Synchronization Pipeline

## 6. Synchronization Pipeline

This section describes the current projection pipeline that materializes derived collections.

### 6.1 Derived Groups
Current sync derives reusable groups from existing school structures:
- grades become `Group` rows with category `grade_cat`
- classes become `Group` rows with category `class_cat`

Both are currently flagged with:
- `schedulable`
- `homework-attached`

### 6.2 Derived Group Parts
Current sync derives taxonomy parts from grades and classes:
- grade parts live in taxonomy `education_grade`
- class parts live in taxonomy `education_grade_class`
- class parts reference the corresponding grade part as parent

The derived part stores:
- `groupId` pointing to the reusable grade/class group
- `ord` for ordering
- later `_path` for materialized hierarchical ordering

### 6.3 Derived Part Availabilities
Current sync derives part availabilities for group parts. These windows are later intersected with raw membership history.

### 6.4 Taxonomy Membership Seed/Projection
There are currently two write-side producers:
- a sync producer that derives student taxonomy memberships from student/class placement for the education space
- a method-based writer that adds taxonomy memberships from an explicit selected part availability and start date

The sync producer is currently bootstrap-oriented and hardcodes the education profile.

### 6.5 Flattening Taxonomy History
`makeMembershipsFlatHistoryStream` expands each `taxonomyMembership` by:
- computing each history entry's effective `ends`
- extracting one flat entry per dynamic `t_*` key

This produces a per-taxonomy/per-entry stream with:
- `memberId`
- `spaceId`
- `period`
- `taxonomyId`
- `groupPartId`
- `partAvailabilityId`
- raw membership range

### 6.6 Availability and Hierarchy Resolution
`makeMembershipGroupedStream` joins flat membership entries with:
- `partAvailabilities`
- `groupParts`
- `spaces`

Then it:
- resolves `groupId` from `groupPart.groupId`
- resolves `jobId` from `space.jobId`
- intersects raw membership range with part availability range
- propagates parent availability constraints down the taxonomy chain

The result is one resolved interval per effective taxonomy-derived concrete group membership.

### 6.7 Unified GroupMembership Read Model
`makeGroupMembershipsStream` writes taxonomy-derived rows into `groupMemberships` with:
- `source = 'taxonomy'`
- `taxonomyPeriod = {taxonomyId}_{period}`
- `memberTaxonomyId = {memberId}_{taxonomyId}_{period}`

Contribution constraint (current — to be migrated to `sources`):
- a taxonomy-derived row for `(groupId, jobId)` is only valid when the contributing `GroupPart._id` matches the one registered in `Group.sources[jobId].partId`

> **TODO - NOW:** Migrate contribution constraint to use `sources`:
> - look up `Group.sources[jobId]`; if absent, skip the row silently
> - skip silently if `sources[jobId].source !== 'taxonomy'` (another source owns this slot)
> - skip silently if `sources[jobId].partId !== groupPart._id`
> - on acceptance, write `periodTypeId` from the space and parent ids into `Group.taxonomy_parentIds[jobId]`
>
> All skips are silent — no error is thrown, other rows in the same pipeline run are unaffected.

`makeTeacherScheduleMembershipStream` writes schedule-derived rows into the same collection with:
- `source = 'lessonSchedule'`
- `jobId = 'supervisor'`
- `groupId = lessonSchedule.classId`
- `starts = lessonSchedule.startDate`
- `ends = lessonSchedule.endDate`

> **TODO - NOW:** `lessonSchedule` source must also apply strict contribution checks:
> - skip silently if `sources[jobId]` is absent or `sources[jobId].source !== 'lessonSchedule'`
> - on acceptance, write `periodTypeId` from the lesson schedule. Capture ids for that row come from `captureTypesSrc.supervisor[periodTypeId]`

This is a current implementation special case, not the target shape for all event-like sources.

Target direction:
- a recurrent-event producer will reference one reusable `groupId`
- the event definition will bind one or more event roles to jobs inside that same group
- the producer will materialize memberships for the configured target jobs without assuming any implicit/default participant job
- `lessonSchedule` is the first concrete instance of this broader pattern

This is the key architectural point:
- `groupMemberships` is the unified authorization/capture read model, not just a taxonomy materialization.

### 6.8 Flag Enrichment
`makeGroupMembershipFlagsStream` denormalizes `Group.flags` into `groupMemberships._flags`.

This allows permission checks to filter membership rows by flag without joining the `groups` collection.

This denormalization is important for performance, not just convenience:
- without flags, a group-based authorization query would have to consider all memberships of the user for the requested job/time window
- as the user's number of memberships grows, the candidate `groupId` set becomes large and the resulting entity query becomes much more expensive
- with flags, each collection only scans the subset of memberships whose groups are explicitly eligible for that collection

So `flags` are the mechanism that keeps group-based authorization query complexity bounded by relevant memberships rather than total memberships.

### 6.9 Capture Projections
`makeSpaceCaptureTaxonomyStream` expands each space capture into one taxonomy-specific capture row.

`makeMemberCaptureTaxonomyStream` then joins taxonomy-source `groupMemberships` by `taxonomyPeriod` and keeps rows whose interval contains the capture date.

This produces `memberCaptureTaxonomies`, which downstream evaluation/report projections consume.

### 6.10 Group Part Path Materialization
`makeGroupPartPathStream` materializes `_path` from parent `_path` and local `ord`.

This supports hierarchical ordering and group-path based downstream queries.
