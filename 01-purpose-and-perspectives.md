# Purpose And Perspectives

## 1. Purpose
This document specifies the groups architecture used to model hierarchical organizational membership, authorization, and capture/reporting context.

It covers four perspectives at once:
- The domain model exposed to product features.
- The authoritative write model for taxonomy-based membership.
- The read models consumed by authorization and downstream projections.
- The current implementation shape in the repository, including important target-direction decisions.

Unless explicitly marked as a target refactor, this document describes the intended canonical model for the current codebase.

## 2. Core Perspectives

### 2.1 Domain Perspective
The domain is built from reusable groups and from hierarchical partitions of those groups inside a space.

Core concepts:
- `GroupCategory`: semantic category of a reusable group, such as grade or class.
- `Group`: reusable business group that can be attached to permissions and other domain objects.
- `JobInGroup`: the role a member has relative to a group, such as attendee or supervisor.
- `GroupJob`: conceptual reference `{ groupId, jobId }` to one explicit job slot inside a reusable group; this is a pair, not a first-class collection.
- `Space`: a grouping domain. A space owns taxonomies and defines how memberships are partitioned over periods.
- `Taxonomy`: a hierarchical axis inside a space.
- `GroupPart`: a node inside a taxonomy hierarchy. A part can either point to a concrete `Group` or be structural only.
- `PartAvailability`: the availability window of a `GroupPart` selection.

The important separation is:
- `Group` is the reusable business object used by authorization.
- `GroupPart` is the taxonomy-local placement of a group within a specific hierarchy.
- `Group` may host several jobs at once; no job is implicit or default.
- consumers that target one participant kind should store an explicit `GroupJob`; consumers that coordinate several jobs inside the same reusable group may store `groupId` plus separate role-to-job bindings.

### 2.2 Write Perspective
`TaxonomyMembership` is the authoritative source of taxonomy-derived membership.

It is intentionally pivoted:
- one document per `(memberId, spaceId, period)`
- one `history[]` array containing full-state snapshots for the period
- one dynamic `t_*` key per eligible taxonomy context inside each history entry

This model is optimized for atomic updates and consistency of hierarchical selections, not for application reads.

### 2.3 Read Perspective
`GroupMembership` is the unified membership read model consumed by authorization and downstream projections.

It is broader than taxonomy-derived membership alone:
- `source = 'taxonomy'` for membership derived from `TaxonomyMembership`
- `source = 'lessonSchedule'` for schedule-derived supervisor membership
- `source = 'custom'` reserved for future/manual producers

The repository treats `groupMemberships` as the single cross-cutting membership index for runtime checks.

### 2.4 Capture Perspective
The groups model also feeds capture/reporting flows.

`SpaceCapture`, `SpaceCaptureTaxonomy`, and `MemberCaptureTaxonomy` let the system answer:
- which taxonomies are relevant for a capture event
- which group a member belonged to at the capture date
- which group-part capture identifier should be attached to downstream evaluation/report rows

### 2.5 Authorization Perspective
Authorization uses two mechanisms:
- direct user-field matching on entity documents
- group-based matching through `groupMemberships`

Group-based checks are filtered by:
- target entity field(s)
- time overlap
- required group flags
- optional job constraint

This makes the group model an authorization substrate, not just an organizational model.
