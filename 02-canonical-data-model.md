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
- `_starts?`
- `_ends?`

Notes:
- `Group` represents the reusable identity, not its placement in a taxonomy.
- `Group.name` should be self-contained and distinguishable on its own, even outside any parent context.
- `_starts` / `_ends` express the lifecycle of the reusable group itself when needed.
- Timing on `Group` is distinct from timing on `PartAvailability`, which describes when a taxonomy placement is valid.

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
- `parentPartId?`
- `groupId?`
- `ord`
- `_path?`

Meaning:
- `GroupPart` is taxonomy-local.
- `GroupPart.name` is the local label inside its parent context and does not need to be globally self-contained.
- Example: a part under `Grade 3` may be named `Science`, while the linked reusable `Group.name` should be something like `Grade 3 Science`.
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
- A shared `Range` abstraction is deferred.

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

Meaning:
- This collection materializes effective group membership intervals regardless of producer.
- It is the runtime index used by authorization and capture/reporting.

Source variants:
- `taxonomy`: produced from taxonomy membership history and availability resolution
- `lessonSchedule`: produced from lesson schedule supervisor assignments
- `custom`: reserved for future producers

### 3.6 Capture Layer

#### SpaceCapture
Capture event scoped to a space and period.

Fields:
- `_id`
- `spaceId`
- `subid`
- `period`
- `date`

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

Meaning:
- This layer resolves a member's taxonomy-derived placement at the exact capture date.
- Downstream evaluation/report pipelines consume it instead of replaying history ad hoc.
