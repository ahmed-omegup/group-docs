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
- conceptual group-job reference: `{ groupId, jobId }` (pair only; no standalone allocated id in the current model)

Group and group-part identifiers are intentionally different.

Rationale:
- a `GroupPart` identifies one taxonomy-local placement and one `jobId`
- a `Group` identifies the reusable target over which several jobs may coexist
- therefore a reusable group cannot be identified by any single job-specific part id
- when a consumer needs one explicit participant kind, it must persist the `groupId` / `jobId` pair rather than infer a job from the reusable group id

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

### 5.5 Group / Job Contribution Constraints

> **TODO - NOW:** Implement `Group.sources` and per-source `_parentIds` fields.

Contribution to `groupMemberships` for a given `(groupId, jobId)` is governed by `Group.sources`.

Contribution is **strict**: a pipeline may only insert a row if all of the following hold:
- `Group.sources[jobId]` is set
- `sources[jobId].source` matches the contributing pipeline's source identifier
- for `taxonomy` source: `sources[jobId].partId` matches the contributing `GroupPart._id`

A mismatch of calendars does not skip the row. The membership is inserted with `period` and `periodTypeId` taken from the source (the space for taxonomy, the lesson schedule for supervisors). `_captures` is filled only when `captureTypesSrc[jobId][periodTypeId]` has ids; otherwise the membership stays and `_captures` is null.

If any condition fails, the row is not inserted. There is no permissive fallback.

This design ensures that misconfigured scopes do not propagate bad data and do not block healthy scopes in the same pipeline run.

Parent synchronization:
- each source writes its parent ids into its own `${source}_parentIds[jobId]` field on `Group`
- sources must not write into another source's `_parentIds` field
- `Group._parentIds[jobId]` is a computed view routing each job to `${sources[jobId].source}_parentIds[jobId]`
- parent relations at the reusable-group layer remain job-specific and source-scoped
- parent synchronization is job-preserving: a source may only emit parent links for the same `jobId` as the child slot it is writing
- these parent links represent substitution/refinement semantics, so cross-job coordination must not be encoded as parenthood
- taxonomy is therefore required to define one explicit job for the whole parent/child chain it contributes
- target direction: this relation may later be represented explicitly between `GroupJob`s rather than stored indirectly on `Group` under job-keyed fields

### 5.6 Entity Targeting Rule
When an entity is semantically about one participant kind within a reusable group, it must store an explicit `GroupJob` reference.

Rules:
- there is no implicit or default job derived from `groupId`
- job-specific entities should store `{ groupId, jobId }` explicitly
- current intended examples include homework, subject, and similar assignment-like models whose audience is one job only
- entities that coordinate several jobs inside the same reusable group may store `groupId` once and keep explicit role-to-job bindings beside it
- future recurrent events follow that second shape: one referenced group, multiple explicit job bindings inside that group
- ancestry/substitution queries should be interpreted at the `GroupJob` level, not as cross-job relations inside the reusable `Group`
