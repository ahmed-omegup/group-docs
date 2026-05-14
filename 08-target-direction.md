# Target Direction

## 10. Target Direction And Deferred Refactors

### 10.1 Explicit Period Boundaries
Current `PeriodType` behavior is formula-based.

Target direction:
- represent period boundaries explicitly rather than deriving them purely from formulas
- use ordered/successor lookup to find the current period and its end boundary

Rationale:
- supports irregular calendars
- decouples business period definition from embedded date formulas
- makes boundary management more explicit

But this is a target refactor, not the current canonical implementation.

### 10.2 Why Period Still Matters Structurally
Even if period boundaries become explicit rows, period remains part of the domain architecture because:
- authoritative taxonomy membership is period-scoped
- history carries only within a period document
- capture flows are period-aware
- synchronization identifiers include period-scoped keys

So `PeriodType` may become more technical in implementation, but the partition it defines remains structurally significant.

### 10.3 Group Timing Vs Placement Timing
Current decision:
- reusable `Group` has `starts` / `ends` (direct fields, **[DONE]** — previously `_starts` / `_ends`)
- taxonomy placement timing stays on `PartAvailability.starts` / `ends`

This keeps two concerns separate:
- lifecycle of the reusable business group
- lifecycle of a particular taxonomy placement window

### 10.4 Deferred Shared Range Abstraction
A shared `Range` abstraction was considered for group lifetime and part availability.

Current decision (deferred — **LATER**):
- do not introduce it yet
- keep direct `starts` / `ends` on `PartAvailability`
- keep direct `starts` / `ends` on `Group`

Reason:
- the refactor does not currently buy enough clarity to justify the migration cost
- the more urgent design issue is query behavior, not field reuse

### 10.5 Temporal Query Optimization As A Separate Concern
Efficient interval querying is a real concern, but it should be treated as a general backend/query-engine concern rather than as the primary driver of the domain model.

In other words:
- the domain model should express valid temporal state cleanly
- query/index strategies should optimize how that state is searched
- those two concerns are related, but not the same thing

### 10.6 Explicit Jobs And Event Role Bindings
Current lessonSchedule membership uses two hardcoded jobs: `supervisor` and `attendee`.

Target direction:
- there should be no implicit/default job derived from a bare `groupId`
- job-specific entities should point to an explicit `GroupJob` reference `{ groupId, jobId }`
- parenthood should be modeled semantically as a relation between `GroupJob`s, not as a cross-job relation on the reusable `Group`
- that parenthood must preserve `jobId`: a child group-job may only inherit from a parent group-job with the same job
- this is because parenthood means substitution/refinement; cross-job links such as `attendee -> supervisor` are coordination bindings, not ancestry
- recurrent-event-like entities are a different shape: they reference one reusable `groupId` and bind several event roles to jobs inside that same group
- event roles such as `source` / `target` belong to the event type; jobs such as `attendee` / `supervisor` belong to the domain/profile of the referenced group
- event producers should read role-to-job bindings from event configuration rather than hardcode `supervisor` or any other job in sync code
- there should be a generic way to sync event-related membership/read models for the configured target roles/jobs
- this generalizes the current pattern of syncing a teacher's lesson schedule presence into `groupMemberships`
- lessonSchedule is the first concrete instance of this pattern; the architecture should eventually generalize it
- the current `teacherSupervising` concept is expected to broaden into an event-target membership/read model rather than remain teacher/supervisor-specific
- `Group` then remains useful as the container that ties several same-entity jobs together for multi-role features, while the actual hierarchy and substitutability semantics live on `GroupJob`

This is a target refactor, not the current implementation.

### 10.7 Current Overlap Limitation
Today, timed authorization and related lookups rely on overlap predicates on `starts` / `ends`.

Those predicates are hard to index optimally at scale because overlap is not a simple equality lookup.

### 10.8 Bucketized Overlap Direction
A promising general optimization direction is bucketized overlap decomposition.

For query interval `[s1, e1)` and stored interval `[s, e)`:
- choose `p = floor(s1)` or another bucket pivot
- match either intervals starting in the relevant forward bucket range or intervals whose midpoint buckets include the pivot
- still require `e > s1`

A practical form discussed in this spec is:
- `((p <= s < e1) || (p in midpointBuckets)) && (e > s1)`

With supporting indexes such as:
- one index led by `starts`
- one index led by `(midpointBuckets, ends)`

This is documented as a backend implementation note, not as a required domain-level abstraction.

### 10.9 Back-Sync Education-Logic Removal Plan
Currently, back-sync contains hardcoded knowledge of the education space (school year period type, grade/class group derivation, education-specific seeder streams).

Target state:
- back-sync contains no education-specific logic
- education data flows through the generic taxonomy pipeline like any other space
- the only education-aware code remaining in back-sync will be the lessonSchedule producer (pending its own refactor into a generic event-job model — see §10.6)
- space-specific knowledge (spaceId, period type assumptions, job names) lives in configuration or in the UI layer, not in sync code

This removal is tied to the successful deployment of the generic group/source/capture pipeline. It should not happen before the pipeline is verified to be producing correct results for the education space.
