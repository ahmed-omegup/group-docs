# Invariants And Non-Goals

## 11. Invariants

### 11.1 Authoritative Membership Invariants
- `TaxonomyMembership` is authoritative only for taxonomy-derived membership.
- `GroupMembership` is the unified read model for runtime consumption.
- `TaxonomyMembership.history` entries are full snapshots, not partial patches.
- Consecutive identical states should be deduplicated.

### 11.2 Hierarchy Invariants
- Child taxonomy selections are only meaningful within a valid parent chain.
- Producers should write ancestor context explicitly.
- Structural `GroupPart` nodes may exist without `groupId`.

### 11.3 Timing Invariants
- A history entry ends where the next entry starts, or at document `_ends`.
- Effective taxonomy-derived membership timing is the intersection of raw membership timing and resolved part availability timing.
- Parent availability constraints propagate downward through the hierarchy.
- Current write flows assume moves are never future-dated.

### 11.4 Authorization Invariants
- Runtime group-based authorization reads from `groupMemberships`, not directly from `taxonomyMemberships`.
- Flag filtering is part of the intended authorization cost model.
- Non-taxonomy producers may write valid `groupMemberships` rows.

### 11.5 Field Mutability And Constraint Drift
Group fields (e.g. `captureTypesSrc`, `sources`, `_captureTypes`) are **not immutable** by default.
When a constraint-bearing field changes on a group, dependents (linked groups, memberships, captures) that no longer satisfy the constraint are treated as **invalid links** — equivalent to a link to a deleted group — and should be handled reactively, not prevented at write time.

If specific fields need to be locked in the future (e.g. an event requiring a linked group to permanently retain a given `captureTypeId`), the mechanism will be one or more **immutability flags** on `Group`, and the relevant write methods will be updated to enforce them. No field should be hardcoded as immutable in the schema; all immutability is opt-in via flags.

## 12. Non-Goals
- This spec does not define a generic event-sourced history engine.
- This spec does not require raw `TaxonomyMembership` documents to be ergonomic for application reads.
- This spec does not yet introduce a shared `Range` type.
- This spec does not solve interval-query performance purely through schema shape.
- This spec does not require every hierarchy node to map to a reusable `Group`.
- This spec does not assume the seeded education profile is the only supported profile.
- This spec does not enforce field immutability at the schema level; immutability is a future opt-in via flags, not a current constraint.
