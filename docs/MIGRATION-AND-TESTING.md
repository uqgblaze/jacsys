# JacSYS — Migration, Verification and Test Plan

**Status:** Draft for architectural review — plan before implementation  
**Version:** 0.1 | **Prepared:** 13 October 2026  
**Scope:** Transition two existing projects to the JacSYS six-repository ecosystem without breaking historical content.

## 1. Goals

Create maintainable, independently deployed components and a stable static API while **retaining all functioning and appropriately licensed/reusable behaviour** from:

1. [Original JacSON](https://github.com/uq-course-profiles/jacson)
2. [Enriched UQBS/course profile scraper and viewer](https://github.com/uqgblaze/uqbs-course-profiles)
3. The earlier uploaded JacSON scripts, Blackboard example, field definitions and UQ Cloud deployment guidance.

Do not alter either upstream/source repository in the planning phase. Do not delete or relocate public endpoints until actual usage and compatibility impact are known.

## 2. Target repositories

| Repository | Role | Migration destination |
|---|---|---|
| `uq-course-profiles/jacsys` | Landing page, ecosystem docs, diagrams and source-to-component migration records | New lightweight documentation site |
| `uq-course-profiles/api` | Generated static JSON output only | JacSON profiles, indexSON indexes, schemas, metadata and versioned publication artifacts |
| `uq-course-profiles/jacmin` | Producer + administrator | Scrapers, UQ catalogue acquisition, parser, override UI, indexes, validation, scheduler, publishing, jobs |
| `uq-course-profiles/jacdash` | Human-readable static viewer | Sean's viewer, all-UQ browser, course/programme views and safe UQBS overlays |
| `uq-course-profiles/jacblack` | Blackboard integration | Legacy `embed/`, assessment-details example, generated/dynamic content components |
| `uq-course-profiles/jacxel` | Future Excel add-in | Placeholder architecture/contract documentation only for initial release |

JacSON and indexSON are **names for generated JSON data products**, not separate application-code repositories.

## 3. Verified starting points and limitations

These are observations from currently available source files/repository documentation, not a claim that a complete code audit has occurred:

- Original `uq-course-profiles/jacson` contains `profiles/`, `embed/`, `demo/` and `assets/`, as well as a minimal README. Its historical published URLs/embedded assets require inventory.
- Uploaded older JacSON scripts demonstrate `course-list.csv` include/exclude logic, Google Sheets flags, profile scraping and GitHub `/contents` uploads. Treat them as migration references, **not production-ready target implementations**.
- Sean's README describes rich profile extraction, UQBS/all-UQ modes, dual manifests and GitHub Pages viewer deployment.
- Sean's `PROJECT_LOG.md` documents history/timeline/diff work, legacy offerings to 2009, UQBS/all-UQ separation, LO mapping override and claimed six-faculty programme structures. Those features must be inspected and individually classified; the project log can be stale relative to the current code.
- `scraper/discover_offerings.py` states that catalogue discovery must run from a non-cloud IP because some cloud requests return HTTP 405. This is a testable deployment constraint, not a universal assertion that all programme APIs are inaccessible.
- Sean's `taxonomy/all-programs.json` exists (large file); its source completeness and generation workflow must be verified before promising all-UQ programme resolution.
- The uploaded `assessment_details-v2.0.html` builds a URL at `https://uqgblaze.github.io/jacson/profiles/{jsonPath}.json`, expects old `assessments[]` fields such as `assessment_title`, `weighting`, `task_description`, and receives assessment number through a query parameter. This is direct evidence of potential student-facing migration breakage.
- The uploaded `definitions.html` is a valuable field dictionary for the enriched flat schema. It includes conditional fields and original HTML-like content, so migration must not drop nested content silently.

## 4. Mandatory discovery inventory

Before code changes, Codex must produce `MIGRATION-INVENTORY.md` with one row for each significant source component:

| Source repo/path | Purpose | Known consumers/URLs | Fields/dependencies | Proposed action | Destination | Risk |
|---|---|---|---|---|---|---|
| Old `jacson/profiles/` | Public historical data | Existing URLs | Legacy flat schema | Preserve + transform/copy as justified | `api` | High |
| Old `jacson/embed/` | Embedded Blackboard UI | Live course sites **to verify** | Legacy fields/path assumptions | Preserve compatibility + refactor | `jacblack` | Critical |
| Old JacSON scripts | Acquisition/Google Sheets/publishing | Legacy workflows | Sheets API, CSV, GitHub token | Selectively refactor | `jacmin` | Medium |
| Sean `scraper/scrape.py` | Enriched course scrape | Viewer/manifest | Existing schema, taxonomy, GitHub index | Refactor core | `jacmin` | High |
| Sean catalogue/taxonomy tools | Offerings/programme relationships | Programme pages/overlays | Current & historical catalogue sources | Validate then adapt | `jacmin` → `api/index` | High |
| Sean `docs/` + build scripts | Public viewer | All-UQ/UQBS editions | Manifests/assets/overlays | Move + update consumer | `jacdash` | High |
| Sean legacy/timeline/diff tools | Historical comparison | Viewer | Legacy ECP data | Evaluate preservation | `jacmin` + `jacdash` | High |
| Sean UQBS AoL/LO overrides | Business School enrichment | UQBS only | Manual overlays, may be private | Classify/publication review | Controlled overlay | High |

Add exact file/commit SHA, license/attribution, maintenance status and test coverage during the real audit. Do not silently discard features just because they are not in the current PRD.

## 5. Preserve existing URLs and static consumers

**Critical migration gate:** survey all known public URL patterns and embedded scripts currently used in Blackboard, JacDASH, teaching material and other integrations. In particular:

```text
https://uqgblaze.github.io/jacson/profiles/{semester}/{full_course_code}.json
https://uqgblaze.github.io/jacson/embed/...
https://uq-course-profiles.github.io/jacson/profiles/...
https://uqsmitc6.github.io/uqbs-course-profiles/...
```

These are **candidate historical endpoints**, not a claim that every pattern is live or still controlled. Verify in source files, repository Pages settings, actual links and user-provided Blackboard examples.

### Three compatible migration approaches

1. **Preserve old hosting paths:** keep existing files/JS available as read-only compatibility data while new `/api/v2/` ships independently.
2. **Compatibility adapter:** provide validated legacy-shape JSON/HTML at legacy paths where the old host is controlled; never assume a redirect is possible from someone else's Pages site.
3. **Consumer migration:** update JacBLACK embedded content when courses can be safely edited; keep fallback when live Blackboard pages cannot be changed.

For each path choose one strategy, name an owner, validate it in a 2025 course copy/test harness, and maintain a retirement decision record. A new v2 schema alone does not protect old URL consumers.

## 6. Historical data and schema migration

Maintain separate treatments for:

- Current/2024+ enriched course profile format.
- Older original JacSON field names/types and HTML-bearing values.
- Legacy ECP profiles with different URL identity/markup (pre-current publishing system).
- Catalogue structures and programme-year snapshots.

For every source file collect:

```text
source_repo
source_commit_sha
original_path
legacy_url (if known)
profile_identity
source_schema/type
conversion_version
conversion_status
warnings
output_path
```

Transform to JacSON v2 **only** when field semantics can be preserved. If a field cannot be mapped faithfully, record a warning and retain the legacy artifact rather than fabricating or silently dropping it. Do not flatten assessment rules, HTML tables, learning-outcome links or multi-value conditions into a misleading single string. Retain originals under an appropriately documented archival or legacy compatibility mechanism, subject to UQ publication rules.

**No destructive rewrite of historical 2025 profiles** during first publication. Compare exact paths and hashes where pinned consumers require them.

## 7. Implementation phases and approval gates

| Phase | Work | Required acceptance evidence |
|---|---|---|
| **P0 — Audit** | Clone/read both sources and supplementary documents; map features, licenses, URLs and hosting; capture baseline metrics | Reviewed inventory; unknowns identified; **no production changes** |
| **P1 — Contract** | Agree JacSON v2, indexSON v2, identity, version/revision and catalogue semantics | Schemas, examples, resolver test suite, owner sign-off |
| **P2 — JacMIN core** | Port enriched scraper with CLI, acquisition separation, validation, source snapshots | Golden-fixture tests and isolated staging output |
| **P3 — indexSON** | Generate course/offering/programme/school/faculty indexes from verified source data | Cross-reference and multi-offering tests; coverage report |
| **P4 — Admin** | Course search, full-field forms, manual/automatic state, audits, editor roles and jobs | End-to-end override tests; non-admin rejection |
| **P5 — API publish** | Stage, diff, validate and publish versioned static assets | Immutable revision/manifest and rollback tests |
| **P6 — JacDASH** | Refactor Sean's public viewer, timelines, programme navigation and overlays | Browser regression tests and static-only operation |
| **P7 — JacBLACK** | Extract old embeds, add v2 client/adapter, preserve 2025 content | Old URL smoke test + pinned historical rendering |
| **P8 — Ops** | User-executed UQ Cloud / Pi deployment procedures and hardening | Documentation, tested backup/restore and access matrix |
| **Later — JacXEL** | Office add-in | Not a blocker for initial JacSYS release |

Phases may be implemented in an order that protects existing student-facing services. Do **not** create/modify production repositories, rotate credentials or deploy without explicit owner approval.

## 8. Test fixtures and baseline corpus

Create a reproducible, non-sensitive fixture set drawn from real published representative profiles, with source capture dates and permissions for redistribution:

- UG and PG course; normal course and unusual contact/staff layout;
- many learning outcomes; assessment-to-LO mapping including missing/inconsistent mappings;
- exam; group-based assessment; multiple due dates; non-standard weight wording;
- AI-permitted and AI-restricted assessment statements;
- mixed activity types, breaks and multi-row tables; missing optional learning activities;
- unusual prerequisites/incompatibilities and catalogue metadata;
- same course/semester with multiple classes;
- 2025 pinned Blackboard example and any current legacy `embed/` use;
- historical ECP/legacy profile with an older page template;
- programme with core, elective, plan-specific and choose-N rules;
- school/faculty attribution that differs from programme management;
- UQ website unavailable, 403/405/429, content-change, broken HTML and HTML injection attempts.

Record expected outputs (“golden” fixtures) only after a human verifies meaning. Golden fixture updates require a reviewed rationale, not an automatic overwrite following scraper changes.

## 9. Automated test levels

### Unit tests

Parsers/normalisers, identity extraction, semantic assessment conversion, LO linking, override precedence, programme rule parsing, URL construction, and sanitisation.

### Schema/contract tests

All output against published JSON Schema; stable types, required identifiers, missing-field semantics, per-major-version compatibility; static API base-path and CORS where practical.

### Relationship tests

Forward/reverse membership; orphan detection; semester/class preservation; source provenance; catalogue-year boundaries; missing profiles permitted in programme indexes.

### JacMIN administrative tests

Search/filter, non-technical editing including nested arrays, save-as-manual, rescrape while manual, compare, return-to-auto, audit log, concurrent save conflict, staged release approvals and role escalation attempt.

### End-to-end consumer tests

Realistic JacDASH browsing; old assessment embed fields/URL behaviour; 2025 Blackboard pinned course; future v3 added without breaking v2; API accessible when JacMIN stopped.

### Pipeline/security tests

Rate limiting, retry/backoff, partial scrape failures, staging validation, repository publish failures, last-good rollback, no secrets in output/logs, spoofed headers and untrusted HTML rendering.

## 10. Quantitative migration gates (baseline-driven)

Before migration, compute an inventory baseline by source repository and semester: distinct full-course identities, total valid files, enriched field coverage, programs/plans known, schools/faculties, current vs legacy ECP data and available public endpoints.

Proposed gates:

1. **No unexplained loss of profile identities** relative to approved migration scope. Every omission appears in an exception report.
2. **No unexplained loss of populated fields** from the enriched input sample. Reductions require an explicit semantics/quality justification.
3. **No invalid cross-references** in a release. Named exceptions must remain warnings/unknowns, not dangling links.
4. **No accidental publication of UQBS private/local-only data** (Sean's log notes some AoL data was not public).
5. **No regression of known legacy Blackboard examples** and pinned 2025 URLs.
6. **No permission escalation**, direct endpoint bypass or unauthorised publication path.
7. **Every release has rollback artifacts and a smoke-test report.**

Set measured numeric thresholds only after the actual baseline audit; do not invent total UQ course/programme counts or claim 100% coverage without evidence.

## 11. Rollback and cutover

- Keep the existing public repositories/endpoints operational while new repositories are built and tested.
- Deploy new v2 API side-by-side; initially have JacDASH/JacBLACK test against it without replacing legacy links.
- Publish a verified release manifest and test sample profile/index URLs over the actual static host.
- Confirm old course embeds, old GitHub Pages URLs and 2025 course copies still work.
- Switch consumers incrementally with explicit owner approval.
- Roll back by restoring the previous published release pointer/artifacts and consumer configuration; do not delete audit history or the new source work.
- Record exact SHA, deployment URL, checklist results, owner and release date for each cutover.

## 12. Outstanding decisions for Codex to surface, not assume

- Does the enriched project **actually** automate all-faculty programme→course extraction from official catalogue pages, including complex elective rules, or is the dataset partially seeded/manual?
- Which legacy 2025 embedded Blackboard URLs are still live, and who controls their host?
- Which elements of UQBS taxonomy, LO mapping overrides and AoL are permitted in public APIs?
- How are source licenses, contributor attribution and University publication policies handled when splitting two repositories into six?
- Which schema major will be released first, and how are immutable revisions hosted?
- Does the selected UQ Cloud zone network allow official catalogue acquisition? Is a Pi an approved alternative?
- What is the authoritative role set for editing, approval and managing users?
- What counts as an editable JacSON field versus immutable profile identity/re-key workflow?

## 13. Final planning deliverables requested from Codex

**Before implementation approval**, deliver:

1. `MIGRATION-INVENTORY.md` with source→destination mapping and links to code.
2. `LEGACY-URL-INVENTORY.md` including Blackboard/embed references and compatibility strategy.
3. `DATA-COVERAGE-BASELINE.md` with measured counts by source, schema and teaching period.
4. A phased implementation and deployment/rollback plan with effort/risk estimates.
5. A verified source/acquisition network matrix for UQ Cloud/Pi/GitHub Actions.
6. Candidate JacSON/indexSON v2 schemas, real sample conversions and test-fixture plan.
7. Decisions needing owner sign-off, clearly separated from non-breaking technical choices.

No production coding, deployment or repository mutations until this planning package has been reviewed.

## 14. References

- [Original JacSON](https://github.com/uq-course-profiles/jacson)
- [Enriched UQBS / all-of-UQ scraper and viewer](https://github.com/uqgblaze/uqbs-course-profiles)
- [Project log](https://github.com/uqgblaze/uqbs-course-profiles/blob/main/PROJECT_LOG.md)
- [Discovery script](https://github.com/uqgblaze/uqbs-course-profiles/blob/main/scraper/discover_offerings.py)
- [UQBS edition deployment instructions](https://github.com/uqgblaze/uqbs-course-profiles/blob/main/DEPLOY-UQBS-edition.md)
- User-supplied `definitions.html`, `assessment_details-v2.0.html`, UQ Cloud guide and legacy JacSON Python scripts.
