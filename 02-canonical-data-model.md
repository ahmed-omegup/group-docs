# Canonical Data Model

## 3. Canonical Data Model

### 3.1 Reusable Group Layer

#### GroupCategory
Semantic family of a reusable group.

Fields:
- `_id`
- `subid`
- `name`
- `flags: Flag[]`

#### Group
Reusable business group referenced by permissions and projections.

Fields:
- `_id`
- `subid`
- `name`
- `categoryId`
- `flags: Flag[]`
- `periodTypeId?` — when set, the group participates in group capture; contributing source pipelines must match this `periodTypeId` or silently skip insertion
- `_captureTypes?: { [jobId: string]: captureTypeId[] }` — computed; for each job, the list of `CaptureType` ids whose snapshots this group participates in. Population logic deferred (**LATER**). **[TODO - LATER]**
- `starts?` — lifecycle start of the reusable group. **[DONE]**
- `ends?` — lifecycle end of the reusable group. **[DONE]**
- `sources?: { [jobId: string]: { source: string } & SourceData[source] }` — per-job source ownership lock and source-specific payload. **[TODO - NOW]**
- `${source}_parentIds?: { [jobId: string]: string[] }` — per-source synced parent id sets, e.g. `taxonomy_parentIds`. Written only by the sync pipeline that owns that source. **[TODO - NOW]**
- `_parentIds?: { [jobId: string]: string[] }` — computed view: routes each job to its owning source's `parentIds` bucket. **[TODO - NOW]**

Notes:
- `Group` represents the reusable identity, not its placement in a taxonomy.
- `Group.name` should be self-contained and distinguishable on its own, even outside any parent context.
- One `Group` may aggregate memberships for multiple jobs.
- `sources[jobId]` acts as both a contribution gate and a source ownership lock:
  - the `source` field identifies which sync producer owns this `(groupId, jobId)` slot
  - once set to source A, no other source is permitted to register or write for that slot
  - the remaining fields carry source-specific payload (e.g. `partId: GroupPart._id` for `taxonomy` source)
  - if `sources[jobId]` is absent, the slot is unclaimed and **no contribution is allowed** — the pipeline silently skips the row
- `${source}_parentIds[jobId]` is written exclusively by the sync pipeline that owns that source; pipelines must not touch other sources' fields
- `_parentIds[jobId]` is a computed view: `sources[jobId].source + '_parentIds'[jobId]`
- `starts` / `ends` express the lifecycle of the reusable group itself.
- Timing on `Group` is distinct from timing on `PartAvailability`, which describes when a taxonomy placement is valid.

> **Deferred (LATER):** A shared `Range` abstraction for `starts`/`ends` across `Group` and `PartAvailability` has been considered and deferred. Current decision: keep direct fields on each entity until the abstraction earns its weight.

#### JobInGroup
Member role relative to the group.

Fields:
- `_id`
- `subid`
- `name`

### 3.2 Space / Period / Taxonomy Layer

#### Space
Top-level grouping domain.

Fields:
- `_id`
- `subid`
- `name`
- `periodTypeId`
- `jobId`

Meaning:
- A `Space` defines which `PeriodType` partitions membership state.
- A `Space` also defines the default `jobId` used when taxonomy memberships are materialized into `groupMemberships`.

#### PeriodType
Technical partitioning model for a space.

Fields:
- `_id`
- `subid`
- `name`
- `mapFromDate(date) -> period`
- `mapToEnd(period) -> endDate`
- `mapToString(period) -> label`

Current meaning:
- `PeriodType` is not just display metadata.
- It defines the partition boundary used by authoritative `TaxonomyMembership` documents.
- Today this boundary is formula-based.

Target direction:
- The domain may later move toward explicit persisted boundaries plus successor lookup.
- Even in that future, the concept remains structurally important because membership history and capture flows are period-scoped.

#### Taxonomy
Hierarchy axis inside a space.

Fields:
- `_id`
- `subid`
- `name`
- `spaceId`
- `parentTaxonomyId?`
- `ord`

Meaning:
- Root taxonomies partition the space directly.
- Child taxonomies refine a parent taxonomy context.

### 3.3 Taxonomy Placement Layer

#### GroupPart
Node inside a taxonomy hierarchy.

Fields:
- `_id`
- `subid`
- `name`
- `taxonomyId`
- `parentId?`
- `groupId?`
- `ord`
- `_path?`
- `_jobId?` — computed from the owning `Space.jobId`; prefixed `_` because it is derived and may lag until sync completes

Meaning:
- `GroupPart` is taxonomy-local.
- `GroupPart.name` is the local label inside its parent context and does not need to be globally self-contained.
- Example: a part under `Grade 3` may be named `Science`, while the linked reusable `Group.name` should be something like `Grade 3 Science`.
- `GroupPart` is intentionally not the same entity as `Group`, and it must not reuse the same identifier.
- The job context of a `GroupPart` is inherited from `Space.jobId`, not stored directly; `_jobId` is a denormalized convenience field that sync populates asynchronously.
- This separation is required because permissions are job-sensitive: for the same reusable group, a `supervisor` membership may have write permission while an `attendee` membership only has read permission.
- Because memberships for different jobs are contributed and interpreted differently, the system cannot identify the reusable group with any one job-specific taxonomy part.
- `groupId` is optional because some nodes are structural only.
- When `groupId` is set, the part points to the reusable `Group` represented at that node.

#### PartAvailability
Selectable availability window for a concrete part.

Fields:
- `_id`
- `subid`
- `groupPartId`
- `starts`
- `ends`

Meaning:
- A part may have multiple availability windows over time.
- Membership assignment targets a `PartAvailability`, not just a `GroupPart`.
- Effective membership intervals are derived by intersecting raw membership history with resolved availability windows.

Current decision:
- `PartAvailability` keeps direct `starts` / `ends` fields.
- `Group` also uses direct `starts` / `ends` fields. **[DONE]**

> **Deferred (LATER):** A shared `Range` abstraction covering both `PartAvailability` and `Group` timing has been considered and deferred. The refactor does not currently buy enough clarity to justify migration cost. Keep direct fields on each entity until the abstraction earns its weight.

### 3.4 Authoritative Membership Layer

#### TaxonomyMembership
Authoritative write model for taxonomy membership.

Fields:
- `_id = {memberId}_{spaceId}_{period}`
- `memberId`
- `spaceId`
- `period`
- `_ends`
- `history[]`

History entry shape:
- `starts`
- dynamic `t_*` keys

Dynamic key value shape:
- `part`
- `range?`

Meaning:
- Each document stores the membership state of one member in one space for one period.
- Each history entry is a full state snapshot from its `starts` until the next history entry or document `_ends`.
- The dynamic keys represent one selected taxonomy context each.

### 3.5 Unified Read Membership Layer

#### GroupMembership
Unified read model for authorization and downstream projections.

Fields:
- `_id`
- `memberId`
- `groupId`
- `jobId`
- `starts`
- `ends`
- `source`
- `taxonomyId?`
- `period?`
- `taxonomyPeriod?`
- `memberTaxonomyId?`
- `_flags?`
- `_captures?` — computed by ID construction: for each `captureTypeId` in `group._captureTypes[jobId]`, append `{captureTypeId}_{period}`. No join is required; some constructed ids may reference non-existent `Capture` documents, which simply produce no downstream row. **[TODO - NOW]**

Meaning:
- This collection materializes effective group membership intervals regardless of producer.
- It is the runtime index used by authorization and capture/reporting.

Source variants:
- `taxonomy`: produced from taxonomy membership history and availability resolution
- `lessonSchedule`: produced from lesson schedule supervisor assignments
- `custom`: reserved for future producers

### 3.6 Capture Layer

#### CaptureType
Defines a conceptual instant relative to any period of a given period type.

Fields:
- `_id = {periodTypeId}_{subid}`
- `periodTypeId`
- `subid`
- `name`

Meaning:
- `CaptureType` is reusable across all groups and spaces that share the same `periodTypeId`.
- Example: `schoolYear_end` defines "end of school year" for the `schoolYear` period type.
- It does not carry a date itself; the date is resolved via `Capture`.

#### Capture
Concrete instantiation of a `CaptureType` for a specific period.

Fields:
- `_id = {captureTypeId}_{period}`
- `captureTypeId`
- `period`
- `date`

Meaning:
- `Capture` is the actual time point for a given capture type in a given period.
- Example: `schoolYear_end_2026` → `date = 2026-07-15`.
- Creating a `Capture` document triggers the downstream resolution pipelines for both group capture and taxonomy capture paths.

#### CaptureSpaceType
Bridges the taxonomy capture path to the unified `CaptureType` infrastructure.

Fields:
- `_id`
- `captureTypeId`
- `spaceId`
- `subid`
- `taxonomyIds`

Meaning:
- When a `Capture` is created for `(captureTypeId, period)`, a sync pipeline derives a `SpaceCapture` for each `CaptureSpaceType` referencing that `captureTypeId`.
- This replaces manually created `SpaceCapture` documents with a declarative subscription model.
- The derived `SpaceCapture._id = {spaceId}_{subid}_{period}`, `date` from the `Capture`.

#### SpaceCapture
Capture event scoped to a space and period.

Fields:
- `_id = {spaceId}_{subid}_{period}`
- `spaceId`
- `subid`
- `period`
- `date`

Note: `SpaceCapture` may be created directly or derived from a `CaptureSpaceType` + `Capture` pair.

#### SpaceCaptureTaxonomy
One capture row per taxonomy relevant to a `SpaceCapture`.

Fields:
- `_id`
- `spaceCaptureId`
- `taxonomyId`

#### MemberCaptureTaxonomy
Resolved member/taxonomy capture context.

Fields:
- `_id`
- `memberId`
- `spaceCaptureTaxonomyId`
- `groupId`
- `groupPartId`
- `jobId` — from the contributing taxonomy's `Space.jobId`; singular because each taxonomy has exactly one job context

Meaning:
- This layer resolves a member's taxonomy-derived placement at the exact capture date, preserving hierarchy context (`groupPartId`).
- Downstream evaluation/report pipelines consume it instead of replaying history ad hoc.

#### CaptureGroupMembership
Unified group membership snapshot for a specific capture, across all sources. **[TODO - NOW]**

Fields:
- `_id = {memberId}_{captureId}_{groupId}`
- `memberId`
- `captureId`
- `groupId`
- `jobs: string[]` — all job roles the member held in this group at capture time, aggregated across all matching `groupMembership` rows (one per source/job combination)
- `sources: string[]` — the contributing sources for the above jobs

Meaning:
- Produced by scanning `groupMemberships` where `_captures ∋ captureId`, then filtering by `starts <= capture.date < ends`.
- One row per `(memberId, captureId, groupId)` — jobs from different sources (e.g. `attendee` from taxonomy, `supervisor` from lessonSchedule) are aggregated into the same row.
- Unified across all sources (`taxonomy`, `lessonSchedule`, future sources).
