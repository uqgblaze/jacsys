# JacSYS — JacMIN Overrides, Editorial Workflow and Publishing

**Status:** Draft for architectural review — not a deployed workflow  
**Version:** 0.1 | **Prepared:** 13 October 2026  
**Owner:** JacMIN | **Affected outputs:** JacSON and indexSON

## 1. Purpose

Describe how a non-technical, authorised manager searches for and edits **every supported JacSON field**, switches a profile between automatic and manual control, reviews changes and publishes a consistent versioned static API. This document applies equivalent governance to indexSON relationships, while keeping the two JSON products distinct.

## 2. Agreed behaviour

- JacMIN has a searchable/filterable browser interface for administrators/managers.
- Selecting a course **offering** opens a complete editable profile, covering every field collected or supported by the JacSON schema, including arrays and nested structures.
- Editing a scraper-controlled profile offers an explicit switch to **Manual mode**; subsequent scrapes do not overwrite its effective published content.
- A manager can return it to **Automatic mode**.
- JacMIN retains the latest scraped copy while a manual record is active, so source changes can be compared and reverting is meaningful.
- JacMIN generates both JacSON and indexSON; API repository files are never edited directly.
- Roles and privileged actions require server-side authorisation.
- Staging, validation, audit trail and safe publication are required.

### Proposed simplifying initial choice

Use **profile-level Manual/Automatic ownership** for release 1, but retain field-level change provenance and design the stored override model so individual field overrides can be introduced later. Editing a single field initially makes the *effective whole profile* manually controlled, not just that field. Show this clearly in the UI.

## 3. Data stores and separation

JacMIN should persist at least four logically separate kinds of records:

```text
source_snapshots/
    validated last-good automatic payloads + acquisition provenance

manual_overrides/
    human-edited effective payloads or explicit change sets

audit_events/
    append-only account/action/timestamp/before/after/reason/history

jobs_and_releases/
    scrape, validation, index builds, staging, publish and rollback state
```

These paths are **logical examples**, not required file-system names. A lightweight database (e.g. SQLite with deliberate backups on a single-host installation) is appropriate for operational state; code should abstract persistence rather than assume JSON files are safe for concurrent edits. Shared/multi-instance deployment requires a reviewed locking/persistence strategy.

Only validated public output belongs in `uq-course-profiles/api`; audit log and credentials stay in JacMIN. Do not store an admin's internal personal identifiers in public JacSON unnecessarily.

## 4. Identity, immutability and edit boundaries

A profile's public identity is its full triplet `{course_code}-{class_code}-{semester_code}`. Changing one of these is **not an ordinary field edit**, because it changes URLs and references. Present identity fields for inspection, but use a special privileged **re-key/create-and-retire** workflow with impact preview, redirects/compatibility review and index repair. Never silently rename/delete a live historical resource.

Other identity-like values (official programme IDs, plan codes, school IDs) require analogous controls.

Preserve source and historical provenance when cloning a manually created or migrated profile; do not assert it was scraped from a UQ page that does not exist.

## 5. Record state model

Use two orthogonal dimensions:

**Control mode:** `automatic` or `manual`.  
**Workflow state:** `draft`, `validated`, `staged`, `published`, `publish_failed`, `archived` (proposed; refine based on implementation).

A separate **source freshness/health** status may be `current`, `source_changed`, `source_unavailable`, `parse_error`, or `unknown`.

| Event | Control mode | Effective data | What is retained? |
|---|---|---|---|
| Successful scrape of auto record | Automatic | New validated automatic payload | Prior version / audit + new source snapshot |
| Successful scrape of manual record | Manual | Existing manual payload | New automatic snapshot; compare flag if changed |
| User edits automatic record | Manual after confirmation | Human-edited payload (draft until published) | Latest automatic snapshot and prior state |
| User edits manual record | Manual | New human draft | Version history |
| User returns to automatic | Automatic after confirmation | Most recent *validated* automatic snapshot | Archived manual revision; mode-change audit |
| UQ source disappears/fails | Unchanged | Last known-good effective payload | Failure evidence; no bulk deletion |
| Publication fails | Unchanged published state | Prior deployed release | Failed job and staged candidate for retry |

Mode changes are not the same as publication: saving a manual draft **does not immediately alter what students see** unless explicit auto-publish is enabled by policy.

## 6. Search, filters and result identity

At a minimum search by: course code, course title, full profile triplet, class, semester, programme, school, faculty, campus, attendance mode, control mode, source status, validation status and updated date.

Result rows must display:

```text
Course code | Title | Semester | Class | School | Mode | Status | Last published
```

Search may first resolve course-level information via local indexes, but selecting a result must open a **specific offering**. If a course has multiple classes in the same semester, do not guess which one the manager intends to edit. Clearly separate public course/index searches from unpublished draft results.

## 7. Editing forms for all supported fields

Generate or maintain field-aware forms from schema metadata where practicable. Suggested sections:

- Profile identity and provenance (read-only or re-key workflow)
- Course overview, title, study level, units, location, delivery mode
- Description, aims and requirements
- Course contacts and staff
- Learning outcomes (ordered entries, add/remove/reorder, ID cross-reference checks)
- Assessments (summary/detail harmonised, weight/due-date display, conditions, exam and AI statement, learning outcome mapping)
- Learning activities (week/period, type, topic, outcome mapping)
- Resources and timetable
- Optional/unknown sections captured by the enriched source scraper
- Source comparison, revisions and validation results

**Completeness rule:** Every v2 schema field supported for publication must have either a labelled, safe UI control **or** a documented administrator-only structured editor that validates types. Do not claim “all fields editable” while hiding nested or optional fields. A raw JSON expert view may supplement forms but must not be the only usable method.

Provide keyboard-accessible add/reorder/delete controls for arrays. Warn before deleting a learning outcome referenced by an assessment or learning activity. Preserve dates and complex weights in non-lossy text when parsing is uncertain.

## 8. Edit and mode-change user experience

Suggested flow:

1. Manager searches for a profile and opens its **published** state.
2. Manager sees **Automatic/Manual** mode, source URL, last valid source scrape, publication and validation status.
3. Editing an automatic profile triggers an explicit confirmation: “Saving this edit will put the entire profile in Manual mode. New scrapes will continue to be collected but will not replace the manual version.”
4. Form auto-save may create a **draft** but must not trigger an unexpected publication.
5. Manager enters a reason (proposed default: required for published manual changes; optional for drafts).
6. Validate and show precise field errors/warnings.
7. Review an effective diff and affected indexSON resources.
8. Save draft / submit for publication / publish, according to role.
9. Show a traceable publication result and release reference after the host confirms success.

### Return to Automatic

The user must preview the latest validated automatic version and its changes **before confirming**. If the source has not been successfully refreshed recently, show its age and optionally require a new scrape first. Retain manual versions for audit/rollback even after automatic control resumes.

## 9. Provenance and conflict resolution

For every managed record track: profile identity, current mode, last valid automatic revision, active manual revision (if any), last publication, source-observed time, override reason, and administrator action events.

Store field-level differences using JSON paths or stable IDs rather than bare array positions wherever possible. Assessment order can change; mapping a change to `assessments[1]` can otherwise edit the wrong task after reordering.

If two managers edit the same draft, use optimistic version checking (revision/ETag/row version); show a conflict and allow a reviewed merge or retry. Do not silently apply last-write-wins.

The automatic source copy should remain **immutable per saved revision**; never mutate the snapshot to “match” a manual edit.

## 10. indexSON overrides

indexSON is generated, but source catalogue data can require administrative correction. Manage relationship overrides in JacMIN against **typed entities/edges** (e.g. programme-year → course with a membership rule) instead of manually patching generated `index/programs/...json` files.

For each relationship edit retain:

- entity IDs, catalogue year/effective period, relation type and rule details;
- official source assertion (or explicit unknown); overridden assertion;
- administrator reason and audit record;
- affected forward/reverse indexes; validation warnings.

A relationship change may affect a programme, course, plan, school and faculty index. Rebuild all affected views together.

## 11. Build, validation and release workflow

```text
Acquisition / manual draft
    ↓
Candidate effective JacSON + relationship set
    ↓
Schema, identity and reference validation
    ↓
Generate affected indexSON
    ↓
Validate complete staged API, check data quality and diffs
    ↓
Review proposed changes / approver authorisation
    ↓
Create one coherent release in Git / immutable assets
    ↓
Confirm deployed URLs and release health
    ↓
Advance current-release pointer only after confirmation
```

**Do not mistake one Git commit for atomic delivery over a CDN.** An explicit release manifest and pinned or content-addressed artifacts may be needed to avoid consumers loading an updated index with old profile files. Exact strategy belongs in `API-CONTRACT.md` and must be tested on the intended host.

### Mandatory publish diff

Display at least: profile additions/modifications/removals; changed course identities; added/removed programme/course edges; altered school/faculty links; records newly in manual mode; unexpected coverage loss; schema version; size/hash of changes; and impacted URLs/consumers where known.

Bulk deletion, schema-breaking changes or migration of pinned historical resources must require additional explicit approval and rollback capability.

## 12. GitHub publication model

JacMIN's publisher should authenticate with **least-privilege GitHub credentials** restricted to `uq-course-profiles/api` where feasible. Use GitHub App or narrowly scoped token according to organisational policy. Use a publisher service account rather than an individual developer's personal token if feasible.

Recommended release method: prepare a staging tree, validate all files, produce one coherent commit on a controlled branch, apply a protected publish workflow and verify GitHub Pages/static-host deployment. No secret may be committed or included in public logs, traces or UI responses. Avoid the legacy model of hundreds of independent `PUT /contents/...` updates with no whole-build validation.

Rollback is a new approved publication referencing the last good release, without deleting historical audit records.

## 13. Permissions and approvals

The project currently prefers **hardcoded/configured system admins** who can grant lower roles, while users added through the UI cannot grant other users access. The full authorisation matrix is in `ACCESS-AND-DEPLOYMENT.md`.

Proposed initial rules:

- Viewer: read/search only.
- Editor/manager: create/modify drafts, run limited approved targeted scrapes; cannot manage users.
- Publisher: approve/publish validated releases; no automatic right to manage users.
- System admin: configure jobs, roles and publication credentials; only preconfigured admins can delegate non-admin access.

**All permission checks must be server-side**, including endpoints behind UI buttons and background jobs.

## 14. Testing scenarios

1. Edit one simple title field: control mode changes only on confirmation; the last automatic snapshot stays intact.
2. Edit nested assessments, dates, multiple LOs and activities: order and cross-references remain valid.
3. Scrape manual profile again: automatic snapshot advances, manual published content does not.
4. Return manual to automatic: preview new source; manual audit persists.
5. Two concurrent edits: stale revision cannot overwrite newer change silently.
6. A malformed/HTML-injection payload is rejected or sanitised appropriately; validation errors show the affected field.
7. Change course title or school: all required indexSON views update consistently.
8. Attempt to change profile triplet: normal form disallows silent re-key and shows impact workflow.
9. Publisher credentials unavailable: staged edits remain intact and live API unchanged.
10. Cloud/CDN build incomplete: current release pointer remains on last good publication.
11. Non-admin calls admin endpoint directly: denied regardless of page controls.
12. Audit revision and rollback survive restart and backup restoration.

## 15. Decisions needing confirmation

| ID | Decision | Proposed default |
|---|---|---|
| OVR-01 | Manual granularity | Whole-profile control first, field diff/provenance retained; per-field override later. |
| OVR-02 | Publishing approval | Explicit staging + approval by publisher/admin. |
| OVR-03 | Reason for override | Required for publication; optional for unsaved draft. |
| OVR-04 | Operational store | Lightweight transactional DB backed up separately from GitHub API repo. |
| OVR-05 | Time-based review | Flag stale manual overrides; do not auto-expire or replace them. |
| OVR-06 | Identity edits | Separate privileged re-key workflow, not normal inline edit. |
| OVR-07 | Auto-publish | Disabled by default for manual edits; scheduled clean scraper builds may auto-publish only with explicit policy. |

## 16. References

- JacSYS PRD; uploaded legacy scripts `sheets_updater.py`, `generate_course_list.py`, `upload_profiles.py` and `update_status.py` (legacy flags and publishing patterns, **not** an endorsed implementation).
- `API-CONTRACT.md` and `ACCESS-AND-DEPLOYMENT.md`.
