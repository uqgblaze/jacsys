
# JacSYS — Codex Instructions

## Project overview

JacSYS is an ecosystem for collecting, managing,
publishing and consuming University of Queensland
course profile data.

The master specification is `PRD.md`.

The ecosystem consists of six GitHub repositories:

- `jacsys` — ecosystem documentation and landing page.
- `api` — versioned static API, containing JacSON and indexSON.
- `jacmin` — server-side scraping, administration, overrides
  and publishing.
- `jacdash` — human-readable course profile viewer.
- `jacblack` — Blackboard Ultra integration.
- `jacxel` — Microsoft Excel integration.

JacMIN generates both JacSON and indexSON.

JacSON represents individual UQ course profiles.
indexSON represents indexes and relationships between
courses, programs, plans, schools, faculties and semesters.

All consumer applications depend on the static API,
not on JacMIN.

## Documentation references

Read the relevant documentation when a task requires it:

- `PRD.md` — overall product requirements and scope.
- `docs/API-CONTRACT.md` — API design, URLs, versions and compatibility.
- `docs/API-DEFINITIONS.md` — documented existing JacSON fields.
- `docs/DATA-SOURCES.md` — scraping sources and data provenance.
- `docs/OVERRIDES-AND-PUBLISHING.md` — manual overrides and publishing.
- `docs/ACCESS-AND-DEPLOYMENT.md` — authentication, permissions and hosting.
- `docs/MIGRATION-AND-TESTING.md` — migration and test requirements.

Do not read every document for every minor change.
Consult the documents relevant to the current task.

## Architectural requirements

1. Maintain the boundaries between all JacSYS components.
2. Keep data generation inside JacMIN.
3. Keep the static API independently accessible.
4. Keep presentation-specific HTML, CSS and JavaScript
   outside canonical JacSON records.
5. Do not introduce unnecessary databases, services or
   infrastructure.
6. Prefer reusing proven existing functionality over
   rewriting it without justification.

## API compatibility

- Published API versions must maintain their documented schemas.
- Breaking changes require a new API major version.
- Preserve historical course profiles and compatibility
  with existing Blackboard integrations.
- Do not remove or rename published API paths without
  an approved compatibility or migration strategy.
- Validate JacSON and indexSON before publication.

## Data and schema changes

When modifying the JacSON schema:

1. Consult the existing field definitions.
2. Compare them with the enriched scraper.
3. Identify retained, renamed, transformed and removed fields.
4. Preserve information unless its removal is justified.
5. Update the schema, documentation and associated tests.

Never infer that a field is unsupported solely because
it is absent from an example JSON record.

## JacMIN administrative rules

- Manual overrides must survive future automatic scrapes.
- Preserve automatically scraped data for comparison.
- Preserve audit history when returning to Automatic mode.
- Validate changes before publishing.
- Prevent unauthorised users from granting elevated permissions.
- Never expose administrator credentials or audit records
  through the public API.

## Working practices

- Inspect existing code before proposing a replacement.
- Make focused, understandable changes.
- Document significant architectural decisions.
- Provide automated tests for important functionality.
- Report test results and any known limitations.
- Distinguish confirmed behaviour from assumptions.
- Flag decisions that could affect existing integrations.

## Safety and permissions

- Do not commit secrets, access tokens or credentials.
- Do not modify existing source repositories without
  explicit authorisation.
- Do not delete historical data without approval.
- Do not deploy to UQ Cloud or Raspberry Pi on the
  user's behalf.
- Do not publish production API changes without approval.
- Provide manual deployment instructions when requested.

## Planning and implementation

For major changes, prepare a plan identifying:

- Proposed approach and architecture.
- Affected repositories.
- Dependencies and risks.
- Migration and compatibility requirements.
- Testing and acceptance criteria.
- Rollback strategy.

Obtain approval before executing destructive or
production-impacting changes.

## Completion reporting

When completing a task, summarise:

- What changed.
- Which files were modified.
- Which tests were run and their results.
- Any outstanding risks or decisions.
