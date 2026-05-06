# Capture And Reporting

## 8. Capture And Reporting

### 8.1 Purpose
Capture flows need a stable answer to this question:
- which group placement did a member belong to at a specific capture date?

Two capture paths exist, each addressing a different question:
- **Taxonomy capture** (`SpaceCapture` → `MemberCaptureTaxonomy`): period-scoped, taxonomy-hierarchy-aware, preserves `groupPartId`
- **Group capture** (`GroupCapture` → `MemberGroupCapture`): group-scoped, source-agnostic, covers all `groupMembership` sources **[TODO - NOW]**

The capture model avoids recomputing answers ad hoc across downstream reporting pipelines.

### 8.2 Space Capture Expansion
A `SpaceCapture` represents a capture event within a space and period.

That capture is expanded into `SpaceCaptureTaxonomy` rows so that each relevant taxonomy is handled explicitly.

### 8.3 Member Capture Resolution
`MemberCaptureTaxonomy` resolves the member's placement for the capture date by joining:
- taxonomy-source `groupMemberships`
- the relevant `taxonomyPeriod`
- the capture date containment condition

The result stores the concrete group context needed downstream:
- `groupId`
- `groupPartId`
- capture/taxonomy linkage identifiers

### 8.4 Why Capture Uses The Read Model
Capture uses materialized `groupMemberships`, not raw `taxonomyMemberships`, because the read model has already resolved:
- full-state history flattening
- concrete part availability windows
- parent availability constraints
- group identity mapping through `groupPart.groupId`

That keeps downstream capture/report consumers simple and stable.

### 8.5 Dependence On Current Write Invariants
End-of-period capture depends on the current write invariant that moves are never scheduled in the future.

Because of that invariant:
- the last history state in the active period is treated as the member's current terminal state for that period
- the period's final capture can rely on that state without reasoning about future planned transitions

If future-dated moves are introduced later, capture semantics will need an explicit redesign.

### 8.6 Group Capture **[TODO - NOW]**

`GroupCapture` is a point-in-time capture event scoped to a group rather than a space.

Fields:
- `_id = {groupId}_{subid}_{period}`
- `groupId`
- `period` — computed from `date` using `Group.periodTypeId`
- `subid`
- `date`

Prerequisite: the target `Group` must have `periodTypeId` set. `GroupCapture` creation must be rejected for groups without it.

`MemberGroupCapture` resolves all members of the group at the capture date by joining:
- `groupMemberships` filtered by `groupId` and `period` (equality), then date containment
- all sources are included — taxonomy, lessonSchedule, and any future source

Fields:
- `_id = {memberId}_{groupCaptureId}`
- `memberId`
- `groupCaptureId`
- `groupId`
- `jobId`
- `starts` / `ends` — the effective membership interval that contains `date`

The join is bounded by `(groupId, period)` equality first, then `starts <= date < ends` within that bounded set. Without the `period` equality key the join would require scanning the full group timeline.

Period consistency invariant:
- each `groupMembership` row contributing to this group must have been written by a pipeline that matched `Group.periodTypeId`
- this is guaranteed by the strict contribution checks in §5.5 — pipelines that do not match `periodTypeId` are silently skipped at write time, so no mismatched period rows ever reach `groupMemberships` for this group
