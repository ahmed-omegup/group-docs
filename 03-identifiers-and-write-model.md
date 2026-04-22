# Identifiers And Write-Side Semantics

## 4. Canonical Identifier And Dynamic Key Rules

### 4.1 Stable Identifiers
The canonical identifiers are:
- taxonomy: root `{spaceId}_{subid}`, child `{parentId}_{subid}`
- group part: root `{taxonomyId}_{subid}`, child `{parentId}_{taxonomy.subid}_{subid}`
- part availability: `{groupPartId}_{subid}`
- taxonomy membership: `{memberId}_{spaceId}_{period}`
- space capture: `{spaceId}_{subid}_{period}`
- capture taxonomy: `{spaceCaptureId}_{taxonomyId}`
- member capture taxonomy: `{memberId}_{spaceCaptureTaxonomyId}`

### 4.2 Dynamic Membership Keys
Dynamic `TaxonomyMembership.history` keys are scoped by taxonomy context.

Canonical format:
- root taxonomy context: `t_{taxonomySubid}`
- nested taxonomy context: `t_{parentContext}_{taxonomySubid}`

Canonical value:
- `{ part: groupPart.subid, range?: partAvailability.subid }`

Constraints:
- Keys must remain valid Mongo field names.
- Key producers must use the exact `t_` prefix.
- Sub-identifiers embedded in keys must avoid ambiguous separators.

## 5. Write-Side Semantics

### 5.1 Authoritative State
`taxonomyMemberships` is the only authoritative source for taxonomy-driven member placement.

Application code should not query it as a general-purpose read model.

### 5.2 History Representation
For a fixed `(memberId, spaceId, period)`:
- `history[]` is ordered by `starts`.
- The effective end of an entry is the next history entry's `starts`, or `_ends` for the last entry.
- Each entry is a full state snapshot for all relevant taxonomy contexts.

### 5.3 Membership Writes
When assigning a member to a concrete part availability:
- resolve the selected `PartAvailability`
- resolve the selected `GroupPart`
- walk ancestors to recover the entire taxonomy chain
- populate one dynamic key per taxonomy context in the hierarchy
- merge this state into all affected period documents starting from the entry date

Current invariant:
- Moves are never scheduled in the future.
- The last history state of the active period is therefore treated as the current state carried forward to that period's `_ends`.
- This invariant is relied on by end-of-period capture flows: the final capture of the period expects the active period's terminal state to be the authoritative current state.

When merging into an existing history:
- split the existing history at the write start point if needed
- merge the new subtree into the existing taxonomy tree
- preserve unaffected sibling branches
- deduplicate consecutive equal states

### 5.4 Parent Consistency
The canonical model is pessimistic:
- child placement is only meaningful when the parent chain is also satisfied
- write-side producers should populate ancestors explicitly
- read-side producers may ignore invalid orphaned child states if they ever appear
