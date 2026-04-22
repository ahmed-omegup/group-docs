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

### 9.4 Group Parts
Current derived group parts:
- one grade part per grade in the grade taxonomy
- one class part per class in the class taxonomy
- each class part is parented by its grade part

### 9.5 Membership Producers
Current education membership producers:
- student placement producer for taxonomy memberships
- lesson schedule producer for teacher/supervisor read memberships

### 9.6 Important Scope Note
This education profile is not the conceptual limit of the model.

It is a seeded implementation profile used to bootstrap the generic architecture in the current codebase.
