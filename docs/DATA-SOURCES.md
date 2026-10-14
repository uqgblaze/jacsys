# JacSYS — Data Sources, Acquisition and Relationship Model

**Status:** Draft for architectural review — not an approved extraction specification  
**Version:** 0.1 | **Prepared:** 13 October 2026  
**Producer:** JacMIN | **Outputs:** JacSON and indexSON

## 1. Purpose

Record which public UQ sources supply each type of information, how frequently JacMIN collects it, how source claims are reconciled, and what can and cannot legitimately be concluded about the university's curriculum structure.

**Terminology:** JacSON = individual published JAC/course-profile JSON; indexSON = generated JSON groupings and relationships (courses, programmes, plans/majors, schools, faculties, semesters and profile availability). **JacMIN contains the scraper/generator code for both.**

## 2. Known source systems

| Source | Supported/observed use | Intended JacSYS output | Caveat |
|---|---|---|---|
| `course-profiles.uq.edu.au` | Full course profile with learning outcomes, assessments, learning activities, staff and other sections | JacSON | Only publicly published profiles; source page variations require fixtures. |
| `programs-courses.uq.edu.au` | Course catalogue fields, offering links and programme/catalogue pages | Primarily indexSON; selected JacSON catalogue fields where well-defined | Some access paths return HTTP 405 from GitHub Actions/cloud hosts in Sean's code notes. |
| Existing public JacSON repo | Historical scraped offerings, known triplets and URLs | Migration/discovery seed | Not a reliable proof of complete university-wide coverage. |
| Sean's repository taxonomy and outputs | UQBS and reportedly six-faculty programme structures, teaching period lookup, LO overrides | Migration seed + verified relationships/overlays | Inspect provenance, coverage and completeness; some data layers are Business School specific. |
| JacMIN authorised manual overrides | Validated corrections to a specific identity/relationship | Effective published JacSON/indexSON | Must retain source record and audit history; never treat overrides as UQ-issued data. |

The current `uqgblaze/uqbs-course-profiles` README documents rich scraping from both official UQ domains. Its `PROJECT_LOG.md` (2026-06-08) reports six-faculty programme structures and extensive legacy work. **These are statements from that project log, not proof of complete/current programme coverage.** Codex must verify the source files, parsing logic, provenance and test coverage.

## 3. Proposed field ownership matrix

| Entity / field family | Preferred authority | Output | Verification expectation |
|---|---|---|---|
| Course/profile triplet, teaching period and source URL | Published course profile | JacSON | Match URL and scraped metadata. |
| Description, aims and requirements | Published course profile | JacSON | Preserve wording and multi-paragraph text. |
| Course learning outcomes | Published course profile | JacSON | Stable labels/ordering; identify source inconsistencies. |
| Assessments, conditions, AI statements, deadlines | Published course profile | JacSON | Do not normalise away exceptions, conditions or multiple deadlines. |
| Assessment-to-LO and activity-to-LO relationships | Explicit source page content | JacSON | If absent/contradictory, mark unknown or data-quality issue. |
| Contacts, staff, resources, activities, timetable | Published course profile | JacSON | Account for optional sections and differing layouts. |
| Course catalogue description, class hours and eligibility | UQ course catalogue | Catalogue-derived metadata; selected JacSON fields if schema supports | Label source per field where origins differ. |
| Programme/plan membership | Official programme/plan catalogue | indexSON | Preserve rule category, effective catalogue year and provenance. |
| Faculty/school identity and affiliation | Official organisational/catalogue source | indexSON | Distinguish academic ownership from programme inclusion or teaching contribution. |
| What profiles exist | Validated JacSON files | indexSON | Exact reference, semester/class identity and availability status. |
| Manual correction | JacMIN overlay | Effective JacSON/indexSON | Audit author privately and expose only appropriate public provenance. |

## 4. Two acquisition streams

### A. Profile acquisition (default: nightly; configurable)

1. Resolve requested course/semester/class triplets from known offering indexes and discovery sources.
2. Fetch publicly published course profiles with polite request rate, timeout and retry policy.
3. Store relevant fetch metadata and, where justified, a protected source snapshot for debugging.
4. Parse all supported fields, normalise, validate identity, compare with last good automatic record.
5. Keep latest valid automatic representation even when the effective record is in Manual mode.
6. If approved for publishing, apply manual override precedence and generate JacSON.
7. Rebuild only affected indexSON lookup/index files; validate before publish.

### B. Catalogue/relationship acquisition (default: weekly; fortnightly and manual options)

1. Discover catalogue programmes, plans/majors, schools/faculties and course relationships using an explicit source list and capture method.
2. Preserve structural constraints (core, restricted electives, choose-N, plan-specific, pathways) and source URLs.
3. Record catalogue year/effective period; generate a new change snapshot only when materially different.
4. Validate relationship references against available courses, but allow a catalogue course to have **no published JacSON offering**.
5. Build/refresh indexSON and report additions/removals/uncertain links for administrator review.

**Neither stream should automatically run because someone merely opened JacDASH or JacBLACK.** Any manager may trigger an authorised refresh from JacMIN if their role permits it.

## 5. Deployment-specific acquisition constraint

Sean's `scraper/discover_offerings.py` explicitly says `programs-courses.uq.edu.au` returns **405** from GitHub Actions/cloud address ranges and that this discovery script must be run from a non-cloud IP such as his local computer. His project log separately reports that `course-profiles.uq.edu.au` works from GitHub Actions.

Do **not** assume that moving catalogue acquisition to UQ Cloud or Raspberry Pi resolves this: test each target network independently against UQ's access policy. Recommended fallback is a supported alternate host or an approved manually triggered run which supplies validated catalogue artifacts; **do not implement proxy evasion or bypass access controls**. Publish last known-good indexes if the catalogue fetch fails and flag their age.

Codex planning deliverable: an access matrix for `course-profiles`, programme detail pages, course catalogue pages and any offering-discovery endpoints when called from GitHub Actions, UQ Cloud zone, Raspberry Pi and a permitted local workstation.

## 6. Data model: identity is not membership

Maintain distinct entities:

```text
Course ──offered as──> Offering ──belongs to──> Teaching period
   │
   ├──may appear in──> Programme requirement group
   └──may be coordinated/owned by──> School ──within──> Faculty
Programme ──contains──> Plan/Major ──may add requirements──> Course
```

One course may appear in multiple programmes, may be compulsory for one programme and elective for another, and may have no recently published profile. A course taught in one faculty's programme is **not necessarily owned by that faculty**.

Programme rules must not be flattened into simple membership if doing so destroys meaning. At minimum model: programme/plan identity; rules; course code; source; catalogue year/effective scope; and optional conditions/selection counts.

### Temporal integrity

- Keep **source capture timestamp** (`observed_at`) separate from **effective academic period/year** (`catalogue_year`, effective-from/to if source proves it).
- A 2026 programme webpage **does not prove** a course was required in 2025. Historical snapshots must only claim what historical sources support.
- If a course changes code, school, faculty or programme requirement, retain history instead of rewriting the past.
- Where a source cannot establish a relationship's effective date, use `effective_period: unknown` (or equivalent) and do not guess.
- For source disputes, retain both assertions privately, flag conflict, use a documented precedence rule or require an admin decision.

## 7. Source data provenance and completeness

Each acquisition result should record:

```text
source_url
source_system
observed_at
retrieval_status
parser_version
content_hash or normalised_hash
coverage_scope
warnings / extraction_confidence (if meaningfully defined)
```

Use a defined `not_observed`, `not_available`, `unsupported`, `source_unreachable`, `missing_required` vocabulary; absence is not proof of falsehood. IndexSON counts should include measured completeness, e.g. known catalogue courses vs available published profiles. Counts are metrics, not coverage guarantees.

### Candidate quality gates

- A profile triplet must match its filename and source metadata.
- No published indexSON path points to a nonexistent profile.
- No programme relation is silently removed because a page was unreachable or returned malformed HTML.
- Missing optional course sections are not incorrectly reported as parser failure; missing mandatory identity is an error.
- Obvious assessment weights and LO references are validated, but unusual legitimate values remain representable.
- Diffs from a catalogue refresh report source additions, removals, relabelling and unknowns separately.
- Alert on unexpected mass deletions or coverage drops; require review prior to publication.

## 8. Existing programme extraction — mandatory verification task

Before promising institution-wide indexes, Codex must answer from repository code and fixtures:

1. Which script parses **programme membership**, as opposed to course offering links?
2. Does `taxonomy/all-programs.json` derive from actual official programme pages, copied UQBS taxonomy, or combined sources? Is it automatically rebuildable?
3. Which six-faculty catalogues are covered, which years and programmes were omitted, and how are electives represented?
4. Can the process run from UQ Cloud or Raspberry Pi, and what happens on 405/429/timeout?
5. Does it capture minors, majors and plans distinctly or merge them?
6. Can it detect programme/course changes with an evidence-backed diff?

Document confirmed functions, gaps and manual steps. **Do not silently extrapolate UQBS coverage into all UQ.**

## 9. Scheduling and change management

| Job | Proposed default | Trigger | On partial failure |
|---|---|---|---|
| Targeted current profile refresh | Manual | JacMIN UI/CLI | Preserve last good record; expose error. |
| Current profile refresh | Nightly | Scheduler | Preserve last good; publish only valid changes. |
| Programme/plan catalogue | Weekly | Scheduler / manual | Preserve last good; mark freshness warning. |
| indexSON generation | After validated relevant data change | Pipeline | Reject orphaned/inconsistent index build. |
| Full source reconciliation | Monthly or manual | Administrator | Report drift; do not bulk delete automatically. |

Rate limits, retries, maximum run size and permitted scraping windows must be configurable and consistent with UQ site requirements. Change-detection hashes must operate on normalised semantic content when appropriate, not volatile page decoration.

## 10. Approval/verification register

- **DATA-01:** Identify authoritative public URL/page family for programme, plan and school ownership data.
- **DATA-02:** Confirm provenance of existing `taxonomy/all-programs.json` and programme coverage.
- **DATA-03:** Confirm whether historical offering discovery from legacy ECP archives is in scope of first release.
- **DATA-04:** Define course and programme entity identifiers, including renames/discontinued entities.
- **DATA-05:** Approve daily and weekly schedules and expected load on UQ websites.
- **DATA-06:** Test target-host accessibility, particularly catalogue 405 responses.
- **DATA-07:** Decide which UQBS overlays (AoL, LO corrections) are private, public or separately distributed; default is **not** to publish private overlays to the public API.

## 11. References

- [Enriched scraper repository README](https://github.com/uqgblaze/uqbs-course-profiles/blob/main/README.md)
- [Programme/offerings discovery script](https://github.com/uqgblaze/uqbs-course-profiles/blob/main/scraper/discover_offerings.py)
- [Project implementation log](https://github.com/uqgblaze/uqbs-course-profiles/blob/main/PROJECT_LOG.md)
- [Programme taxonomy directory](https://github.com/uqgblaze/uqbs-course-profiles/tree/main/taxonomy)
- Existing uploaded `definitions.html` and JacSYS PRD.
