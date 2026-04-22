# Capture And Reporting

## 8. Capture And Reporting

### 8.1 Purpose
Capture flows need a stable answer to this question:
- which taxonomy-derived group placement did a member belong to at a specific capture date?

The capture model avoids recomputing that answer ad hoc across downstream reporting pipelines.

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
