# Seeded Education Profile

## 9. Seeded Education Profile

The current repository contains a bootstrap education profile that maps the abstract model onto existing school structures.

### 9.1 Space
Current seeded space:
- education space

### 9.2 Taxonomies
Current seeded taxonomies:
- `education_grade`
- `education_grade_class`

Meaning:
- grades are the first hierarchical level
- classes refine grades as a child taxonomy

### 9.3 Reusable Groups
Current derived reusable groups:
- one `Group` per grade
- one `Group` per class

Current categories:
- grade category
- class category

Current flags on both group families:
- `schedulable`
- `homework-attached`

Current education-owned jobs for these group families:
- `attendee` for the main student cohort membership of the class/grade
- `supervisor` for staff/teacher participation relative to the same reusable class/grade group

These jobs belong to the education profile. Other modules may contribute memberships to those jobs, but they do not define the job vocabulary itself.

### 9.4 Group Parts
Current derived group parts:
- one grade part per grade in the grade taxonomy
- one class part per class in the class taxonomy
- each class part is parented by its grade part

### 9.5 Membership Producers
Current education membership producers:
- student placement producer for taxonomy memberships
- lesson schedule producer for teacher/supervisor read memberships

Current lessonSchedule assumptions:
- the education profile owns the `attendee` and `supervisor` jobs used by class/grade groups
- taxonomy membership contributes the `attendee` job
- lessonSchedule currently contributes the `supervisor` job
- all lesson schedules are assumed to use the `periodType_schoolYear` period type

These are current constraints, not architectural requirements. See §10 for the target direction toward explicit job targeting and event role-to-job bindings.

Current consumer targeting rule in the education profile:
- homework and subject models should target an explicit `GroupJob`, not a bare `groupId`
- lesson schedules are a separate case because they coordinate multiple jobs within one referenced group

### 9.6 Important Scope Note
This education profile is not the conceptual limit of the model.

It is a seeded implementation profile used to bootstrap the generic architecture in the current codebase.

### 9.7 Migration Target
The goal is for back-sync to contain no education-specific logic once the migration is complete:
- the education *data* (groups, parts, memberships) will continue to flow through the generic taxonomy pipeline
- the hardcoded education sync streams (grade/class group derivation, the education-specific seeder) should be removed
- lessonSchedule will be refactored separately
- knowledge of the education spaceId and `periodType_schoolYear` should live only in the UI layer after migration
