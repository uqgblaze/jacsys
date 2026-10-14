# JacSYS — API Contract and Compatibility Policy

**Status:** Draft for architectural review — not an approved or deployed specification  
**Version:** 0.1 | **Prepared:** 13 October 2026  
**Owner:** JacSYS project | **Implementer:** Codex, subject to approval  
**Related:** `DATA-SOURCES.md`, `OVERRIDES-AND-PUBLISHING.md`, `MIGRATION-AND-TESTING.md`

## 1. Purpose and scope

Define the public, static, versioned contract between **JacMIN** (the producer) and **JacDASH, JacBLACK, JacXEL** and other consumers. This document specifies the intended compatibility guarantees, resource identity, index resolution and publication conventions. It does **not** prescribe a particular web framework.

### Agreed project boundaries

- `uq-course-profiles/jacmin`: acquisition, scraping, normalisation, manual overrides, index generation, validation, publishing and administrative UI.
- `uq-course-profiles/api`: **generated static data**, including **JacSON** (individual JAC profile JSON) and **indexSON** (JSON indexes/groupings for programmes, plans, schools, faculties, courses and offerings).
- `uq-course-profiles/jacdash`, `/jacblack`, `/jacxel`: independent consumers. `/jacxel` is future work.
- `uq-course-profiles/jacsys`: landing page, architecture and documentation.
- Consumers must function even when JacMIN is offline.

### Proposed decisions (subject to sign-off)

1. Use an explicit **major contract version in the resource path** (`/v2/`, `/v3/`). Begin the redesigned schema at **v2** to distinguish it from historical unversioned output; do not imply that the old data already conforms to v2.
2. Keep **schema version**, **offering identity**, and **published revision** distinct.
3. Maintain stable versioned resource paths and, where historical content must not change, support **immutable revisions** in addition to ordinary versioned URLs.
4. Never use an unversioned `latest` resource for embedded production Blackboard content unless the author explicitly chooses moving data.

## 2. Resource layout

Repository-relative published paths (the deployed base URL is configured, never embedded in generated file content):

```text
api/
├── README.md
├── metadata/
│   ├── versions.json                 # currently offered contract versions
│   └── releases.json                 # release identities and timestamps
├── schemas/
│   └── v2/
│       ├── jacson.schema.json
│       └── indexson.schema.json
└── v2/
    ├── profiles/                     # JacSON
    │   └── {semester_code}/
    │       └── {course_code}-{class_code}-{semester_code}.json
    ├── index/                        # indexSON
    │   ├── courses.json
    │   ├── programs.json
    │   ├── schools.json
    │   ├── faculties.json
    │   ├── semesters.json
    │   ├── courses/{course_code}.json
    │   ├── programs/{program_id}.json
    │   ├── schools/{school_id}.json
    │   ├── faculties/{faculty_id}.json
    │   └── semesters/{semester_code}.json
    └── metadata/
        └── build.json
```

**Example** (actual known triplet used for path illustration):

```text
/v2/profiles/7660/MGTS2301-62448-7660.json
/v2/index/courses/MGTS2301.json
```

If deployed using GitHub Pages for the `api` repository, the public URL will normally include the repository's `/api/` base path. Do not prepend an additional `/api/` directory inside the repository without an explicit hosting decision. The final production base URL, domain ownership, availability and cross-origin settings remain to be verified during planning.

## 3. Identity and resolution

| Concept | Identity | Rule |
|---|---|---|
| Course | `course_code` | A catalogue entity; not a specific instance of delivery. Preserve code as a string. |
| Offering/profile | `{course_code}-{class_code}-{semester_code}` | Unique profile identity for a class/period. Do not collapse distinct classes. |
| Teaching period | `semester_code` | String, not integer; use a defined lookup to display semester names. |
| Programme / plan / school / faculty | Official stable identifier when available | Never invent an official-looking identifier; if an internal slug is needed, label it as such. |
| Revision | Build/hash/revision identifier | Identifies a frozen published payload, independently of schema major version. |

A course may have **multiple offerings in one semester**. Resolver behaviour must be deterministic:

- Course + semester + class: resolve exactly or return a not-found result.
- Course + semester with one offering: resolve that offering.
- Course + semester with multiple offerings: expose the available offerings and require a class selection, unless a formally documented disambiguation policy applies.
- Course alone: an optional convenience lookup may point to the latest *available* semester; return all candidates if the latest has multiple classes.
- Do not equate *latest available* with *latest approved* or *currently taught*.
- All references in indexSON must resolve relative to a defined API base or use a documented base-relative URL convention. Avoid ambiguous relative paths (`../..`) without a published resolution specification.

## 4. JacSON v2 minimum contract

The example below is **illustrative and not an assertion that these values are present for every course**. Codex must derive the full JSON Schema by comparing representative outputs and the source field-definition inventory. Missing, absent, null and empty values must have separately documented meanings.

```json
{
  "jacson": {
    "schema_version": "2.0",
    "source_url": "https://course-profiles.uq.edu.au/course-profiles/MGTS2301-62448-7660",
    "scraped_at": "2026-06-04T02:48:32Z"
  },
  "course": {
    "code": "MGTS2301",
    "title": "Business Strategy in Practice",
    "units": 2,
    "study_level": "Undergraduate",
    "school": "Business School"
  },
  "offering": {
    "class_code": "62448",
    "semester_code": "7660",
    "full_course_code": "MGTS2301-62448-7660"
  },
  "description": "...",
  "aims": "...",
  "requirements": {},
  "learning_outcomes": [],
  "assessments": [],
  "learning_activities": [],
  "staff": [],
  "resources": {}
}
```

### Field rules to specify in JSON Schema

- **Stable types:** strings for codes, numbers only for reliably parsed numeric quantities, arrays for repeatable objects.
- **Content fidelity:** preserve complete text, paragraph boundaries and source URLs. Do not silently remove legally/pedagogically important qualifiers.
- **Assessment fields:** parse weights to a number only when unambiguous (e.g. `30%`); preserve original display string for `20% Best 3 of 5`, alternative dates, attendance-specific instructions and uncertain parses.
- **Due dates:** retain original display text; provide an ISO datetime only when accurately parsed with appropriate timezone and context.
- **Learning outcomes:** normalise identifiers consistently, retain original labels/text, ensure assessment and activity references match known outcomes; flag inconsistencies instead of inventing mappings.
- **Rich content:** canonical readable text is preferred. If some source HTML needs retention, place it in an explicitly named source-fidelity field and sanitize it before any consumer renders it. No Blackboard-specific HTML/CSS/JS in JacSON.
- **Provenance:** include source and acquisition information, but never disclose internal admin usernames, private comments, credentials or audit history in the public dataset.
- **Extensibility:** documented optional fields may be added without reinterpreting or changing existing ones; all fields collected by the improved scraper should be inventoried and either mapped or explicitly excluded with a reason.

## 5. indexSON v2 contract

indexSON is JSON **data**, not the generation application. JacMIN generates it from verified catalogue information and JacSON availability.

Required minimum views:

1. **Course index:** lookup by course code; title, offering identities/paths, available semesters, optional linked programme/school/faculty identifiers, provenance.
2. **Programme index:** programme identity and appropriately qualified course memberships, including rule groups such as compulsory, elective selection, plan-specific and pathway-dependent courses where source material supports them.
3. **Plan/major index:** introduce as a dedicated entity where the official catalogue distinguishes plans from programmes; determine path and identifier policy before release.
4. **School/faculty index:** verified responsible and organisational relationships; do not infer an ownership relationship merely because a course appears in a programme.
5. **Semester index:** precise list of available profile identities and paths, not merely all course codes.
6. **Global manifest/build metadata:** dataset counts, generation time, input snapshot IDs, warnings, generator version and integrity hashes.

Each relationship should, where feasible, capture source, capture time and the effective year/catalogue rules. Historical programme membership **must not** be backdated using today's programme structure. An unresolved relationship remains unknown rather than false or assumed.

### Illustrative programme representation

```json
{
  "type": "program",
  "id": "example-program-id",
  "name": "Example programme (illustrative only)",
  "catalogue_year": 2026,
  "source_url": "https://programs-courses.uq.edu.au/...",
  "course_groups": [
    {
      "label": "Compulsory courses",
      "rule": "all",
      "courses": ["EXAM1001", "EXAM1002"]
    },
    {
      "label": "Choose one from",
      "rule": "choose",
      "choose": 1,
      "courses": ["EXAM2001", "EXAM2002"]
    }
  ]
}
```

Values above are deliberate placeholders; Codex must not copy them into published UQ indexes.

## 6. API versioning versus data revisions

**Important:** `/v2/` protects the *shape* of data, not necessarily the *exact historical contents*. A file could be updated tomorrow and still satisfy Schema v2.

Proposed policy:

- **Major schema versions are immutable contracts.** A breaking field/type/semantic change requires `/v3/` rather than modifying `/v2/`.
- **Ordinary versioned offering paths may be refreshed** with source corrections where explicitly allowed by policy. Such refreshes are audited and published with release metadata.
- **Immutable release/revision resources** must be available for consumers needing an exact historical snapshot. The storage/URL form (for example, revision-hash directory, Git tag-derived artifact, or immutable static release) must be selected in planning and tested with the actual host.
- **2025 Blackboard sites:** first inventory live URLs and fields. Where exact unchanged historical data is needed, pin a revision or preserve a read-only legacy endpoint. Simply changing to `/v2/` is not an adequate guarantee.
- **Deprecation:** publish notice and consumer migration path. Never silently delete an API major version or old endpoint known to be used by live teaching content.

## 7. Publication consistency and caching

One JacMIN release must contain a mutually consistent set of JacSON and indexSON outputs. The proposed flow is:

`build in staging → validate → compute cross-references/hashes → publish one Git commit or release artifact → verify deployed assets → switch the stable release manifest/pointer`.

A single Git commit is atomic in Git, **but static hosting and CDN caches need not update atomically**. To prevent clients mixing old and new indexes, use release-scoped immutable assets or a verified manifest/pointer strategy. Document cache headers and fallbacks at the chosen host. Do not claim the API is transactionally atomic without testing it.

Consumers should tolerate unavailable optional fields, network failures and stale caches; they must not silently display a different offering when a requested one cannot be found.

## 8. Compatibility and client rules

- JacDASH, JacBLACK and JacXEL should configure an API base URL and contract major version.
- JacBLACK embedded in production Learn.UQ should specify an offering/semester and a supported contract version; pin a frozen revision for exact historical content when needed.
- Javascript consumers should not rely on filenames constructed from a course code alone; resolve using indexSON.
- Validate cross-origin fetch support for GitHub Pages and any future custom domain, including Content Security Policy / Blackboard iframe restrictions.
- A `latest` resolver is convenience functionality, not a substitute for explicit class/semester selection in assessed or student-facing historical materials.

## 9. Decisions awaiting approval / verification

| ID | Decision | Proposed default | Verification required |
|---|---|---|---|
| API-01 | First redesigned major version | `v2` | Approval of naming; v1 legacy is not automatically supported by new schema. |
| API-02 | Canonical public base URL | Pages for `uq-course-profiles/api` | Check Pages publishing settings, redirect/hostname, CORS and longevity. |
| API-03 | Immutable revision mechanism | Release-scoped static assets + manifest | Demonstrate no broken legacy Blackboard links. |
| API-04 | Record selector ambiguity | Explicit class when multiple offerings | Test realistic multi-class course fixtures. |
| API-05 | Detailed schema | Normalised enriched scraper | Map all real fields and exceptions before finalising. |
| API-06 | Data retention | Keep historical profiles and revisions | Agree retention and UQ permissions before publishing. |
| API-07 | Programme index shape | Express rule groups, not flat membership only | Check official programme catalogue structures and extraction reliability. |

## 10. Acceptance tests

- Published v2 JacSON and indexSON files pass their respective published JSON Schemas.
- Every indexSON profile reference resolves to a valid file of the expected identity and major schema version.
- A multi-class teaching period is not silently resolved to an arbitrary class.
- A v2 consumer continues working when a v3 dataset is published.
- A pinned 2025 Blackboard example renders after migration with no dependency on JacMIN.
- An immutable revision remains byte-identical after a subsequent current-data refresh.
- A partially failed build cannot cause the stable manifest to point to a mixed or incomplete publication.
- Documentation distinguishes real observed fields from illustrative examples.

## Sources to inspect

- [Existing public JacSON repository](https://github.com/uq-course-profiles/jacson)
- [Enriched scraper repository](https://github.com/uqgblaze/uqbs-course-profiles)
- Uploaded `definitions.html` (source key dictionary), `assessment_details-v2.0.html` (legacy embedded consumer), and JacSYS PRD.
- See also `MIGRATION-AND-TESTING.md` for mandatory legacy URL inventory.
