# Capture And Reporting

## 8. Capture And Reporting

### 8.1 Purpose
Capture flows need a stable answer to this question:
- which group placement did a member belong to at a specific capture date?

Two capture paths exist, each addressing a different question:
- **Taxonomy capture** (`SpaceCapture` → `SpaceCaptureTaxonomy` → `MemberCaptureTaxonomy`): preserves full taxonomy hierarchy context (`groupPartId`), period-scoped through `SpaceCapture`
- **Group capture** (`Capture` → `CaptureGroupMembership`): source-agnostic snapshot of all effective memberships at a point in time, driven by the unified `CaptureType`/`Capture` layer **[TODO - NOW]**

Both paths are anchored to the same `CaptureType` / `Capture` infrastructure. `SpaceCapture` can be derived from `CaptureSpaceType` + `Capture`, so both paths share the same temporal trigger.

The capture model avoids recomputing answers ad hoc across downstream reporting pipelines.

### 8.2 Capture Infrastructure: CaptureType and Capture

`CaptureType` defines a conceptual instant relative to any period of a given `PeriodType` (e.g. "end of school year").

`Capture` is the concrete instantiation: a specific period and actual date for a `CaptureType`.

`_id = {captureTypeId}_{period}` — this id is also used as the suffix when constructing `GroupMembership._captures` entries, so no join is needed to determine which captures a membership is eligible for.

Creating a `Capture` document is the trigger for both downstream paths:
- for taxonomy capture: if a `CaptureSpaceType` references the same `captureTypeId`, a `SpaceCapture` is derived
- for group capture: `groupMemberships` with `_captures ∋ capture._id` are resolved into `CaptureGroupMembership` rows

### 8.3 Space Capture Expansion
A `SpaceCapture` represents a capture event within a space and period.

It can be created directly or derived from a `CaptureSpaceType` + `Capture` pair.

That capture is expanded into `SpaceCaptureTaxonomy` rows so that each relevant taxonomy is handled explicitly.

### 8.4 Member Capture Resolution
`MemberCaptureTaxonomy` resolves the member's placement for the capture date by joining:
- taxonomy-source `groupMemberships`
- the relevant `taxonomyPeriod`
- the capture date containment condition

The result stores the concrete group context needed downstream:
- `groupId`
- `groupPartId`
- capture/taxonomy linkage identifiers

### 8.5 Why Capture Uses The Read Model
Capture uses materialized `groupMemberships`, not raw `taxonomyMemberships`, because the read model has already resolved:
- full-state history flattening
- concrete part availability windows
- parent availability constraints
- group identity mapping through `groupPart.groupId`

That keeps downstream capture/report consumers simple and stable.

### 8.6 Dependence On Current Write Invariants
End-of-period capture depends on the current write invariant that moves are never scheduled in the future.

Because of that invariant:
- the last history state in the active period is treated as the member's current terminal state for that period
- the period's final capture can rely on that state without reasoning about future planned transitions

If future-dated moves are introduced later, capture semantics will need an explicit redesign.

### 8.7 Group Capture: CaptureGroupMembership **[TODO - NOW]**

`CaptureGroupMembership` is the unified group membership snapshot for a given `Capture`, across all sources.

**How `GroupMembership._captures` is populated:**
- For each `captureTypeId` in `group._captureTypes[membership.jobId]`, construct id `{captureTypeId}_{membership.period}`
- Pure id construction — no join to `Capture` collection required
- The resulting ids are stored in `GroupMembership._captures`
- Some ids may reference non-existent `Capture` documents; those produce no downstream row and are otherwise harmless

**Resolution pipeline:**
- For a given `captureId`: find all `groupMemberships` where `_captures ∋ captureId` (indexed equality)
- Then filter: `starts <= capture.date < ends`
- Group by `(memberId, groupId)` and aggregate matched rows into one `CaptureGroupMembership` row, collecting `jobs` and `sources` arrays

This turns what would be a timeline range scan into a bounded indexed lookup.

**Period consistency:**
- Because contributing pipelines only write a `period` field (and therefore a valid `_captures` entry) when `Group.periodTypeId` matches the source's period type (§5.5), there are no cross-period mismatches to defend against at read time.

**Relationship to taxonomy capture:**
- `CaptureGroupMembership` is not a replacement for `MemberCaptureTaxonomy`
- `MemberCaptureTaxonomy` preserves taxonomy hierarchy context (`groupPartId`) and is consumed by evaluation/report pipelines that need to know *where in the hierarchy* a member was classified
- `CaptureGroupMembership` answers *who was in this group at this moment*, regardless of how they got there

### 8.8 User.memberIn Denormalization

`User.memberIn` is a denormalized capture-based group index written directly onto the user document.

Shape:
```
memberIn: {
  // key = captureId = `{captureTypeId}_{period}` = `{periodTypeId}_{captureSubId}_{period}`
  [captureId: string]: { jobId: string; groupId: string }[]
}
```

Key construction:
- `captureTypeId = {periodTypeId}_{captureSubId}`
- `captureId = {captureTypeId}_{period}` = `{periodTypeId}_{captureSubId}_{period}`
- Which `{period}` and `{captureSubId}` to use for a given `periodTypeId` is defined in settings: `periods: { [periodTypeId]: { period, captureSubId } }`

Production rule:
- for each `CaptureGroupMembership` row where the group has the `searchable` flag, write an entry under the corresponding `captureId` key on the member's `User` document
- only `searchable`-flagged groups contribute; this keeps the index small

Query pattern:
- `memberIn.{periodTypeId}_{captureSubId}_{period}: { jobId: someJob, groupId: { $in: selectedGroups } }`
- use `$or` across multiple keys when more than one period/captureSubId is relevant
- the group filter in UI should restrict selectable groups to those whose `_captureTypes[jobId]` contains the corresponding `captureTypeId` — groups without it are filtered out or greyed

This denormalization is read-optimized and should not be treated as authoritative; `CaptureGroupMembership` remains the source of truth.

### 8.9 User.classedIn Denormalization

`User.classedIn` generalizes the education-specific `attendeeIn` into a taxonomy+capture-scoped classification index.

Shape:
```
classedIn: {
  // key = SpaceCaptureTaxonomy._id = `{spaceId}_{captureSubId}_{period}_{taxonomyId}`
  // e.g. classedIn['education_endYear_2025_education_grade']
  //       spaceId=education, captureSubId=yearEnd, period=2025, taxonomyId=education_grade
  [spaceCaptureTaxonomyId: string]: { groupId: string; groupPartId: string; group: string; groupPart: string; path: string }
}
```

Key construction:
- the key is the `SpaceCaptureTaxonomy._id`, which is `{spaceCaptureId}_{taxonomyId}` and expands to `{spaceId}_{captureSubId}_{period}_{taxonomyId}`
- this is the same information as the old `{taxonomyId}_{period}_{captureSubId}` key but reordered so it matches the existing SpaceCaptureTaxonomy document id directly — no key derivation needed

Usage:
- display the canonical class assignment of a student at end-of-year 2025: `classedIn['education_endYear_2025_education_grade']`
- sort students in a term grid using `classedIn['education_term1_2025_education_grade']`
- any space/taxonomy pair can contribute as long as a matching `SpaceCapture` + `CaptureSpaceType` exists

Migration note: this replaces `attendeeIn` (which was hardcoded to the education space with key `{period}_grade{''|'_class'}`). `attendeeIn.2025_grade` corresponds to `classedIn.education_endYear_2025_education_grade`.
