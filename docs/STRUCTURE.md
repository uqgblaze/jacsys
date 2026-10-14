## Existing JacSYS Repository Structure

JacSYS already exists as a working monorepo. The implementation described in this PRD must work within this existing structure rather than replacing it wholesale.

The current repository structure is approximately:

```text
jacsys/
├── .codex/
├── .dev_guides/
├── api/
│   ├── v1/
│   └── v2/
│       ├── profile/
│       └── index/
│           └── {index_type}/
├── indexson/
├── jacblack/
├── jacdash/
├── jacmin/
├── jacson/
└── jacxel/
```

### `.codex/`

Codex-specific working material.

This directory contains:

- the current `PRD.md`;
- implementation notes or planning documents required by Codex;
- other temporary or persistent Codex working material where appropriate.

Codex may update its own working documentation here, but application code should remain within the relevant Jac component directory.

---

### `.dev_guides/`

Reference documentation for the development and deployment environment.

This currently contains material including:

- UQcloud Zones documentation;
- the UQcloud starter/template project;
- other deployment or infrastructure documentation relevant to JacSYS.

This directory should be treated as **reference material**, not application code.

Where implementation decisions concern UQcloud, nginx, UQ SSO, Python/uWSGI or other UQ-specific infrastructure, Codex should consult these documents rather than assuming a conventional public-cloud deployment model.

---

### `api/`

The published, machine-readable JacSYS API.

This directory should contain **generated/published data only**. Scraper implementation code, administrative data and application state should not be stored here.

#### `api/v1/`

Legacy API.

Version 1 is being deprecated but must remain available for backwards compatibility with existing consumers.

It is effectively **read-only from the perspective of the new JacMIN implementation**.

The repository owner may manually patch or upload V1 files when required.

**Codex must not migrate, rewrite, delete or automatically update V1 data unless explicitly instructed.**

#### `api/v2/`

Current API version and the publishing target for the work described by this PRD.

Course profiles will be published beneath:

```text
/api/v2/profile/
```

The precise profile hierarchy should follow the V2 API specification defined elsewhere in this PRD.

Generated IndexSON data will be published beneath:

```text
/api/v2/index/{index_type}/{name}.json
```

For example:

```text
/api/v2/index/course/MINE3120.json
/api/v2/index/program/GRAD-DIP-RESOURCE-DEVELOPMENT.json
/api/v2/index/school/mechanical-and-mining-engineering.json
/api/v2/index/faculty/engineering-architecture-and-information-technology.json
```

The supported index types should be defined centrally rather than scattered throughout the implementation.

The API directory is the **published output layer**, not the source-of-truth application database.

---

### `indexson/`

Legacy IndexSON implementation.

This contains the earlier indexing system that was integrated with the previous JacSON scraper, currently associated with JacSON `v1.2.3-alpha`.

It should be treated primarily as:

- reference implementation;
- reusable legacy logic where appropriate;
- evidence of the existing IndexSON schema and behaviour.

The new system should not simply run this implementation unchanged.

Relevant functionality should instead be evaluated and, where appropriate, incorporated into the new JacMIN-managed indexing pipeline.

The long-term architecture should make **JacMIN responsible for initiating/regenerating IndexSON**, with the resulting static JSON files published into:

```text
/api/v2/index/
```

---

### `jacblack/`

JacBLACK contains the Blackboard-facing presentation and integration assets.

This includes HTML, CSS, JavaScript and related components intended for use within Blackboard Ultra.

JacBLACK is a **consumer of JacSON/API data**, not the owner of the scraper or API-generation process.

The distinction should remain:

```text
JacMIN
   ↓
JacSON / IndexSON data
   ↓
API V2
   ↓
JacBLACK
   ↓
Blackboard Ultra
```

Changes to JacBLACK are outside the primary scope of the current JacMIN work unless modifications are required to maintain compatibility with the V2 API.

---

### `jacdash/`

JacDASH is the existing human-readable dashboard/interface developed primarily by Sean.

It should be preserved as a distinct JacSYS component.

Of particular importance, JacDASH currently contains a scraper implementation that appears to be:

- more comprehensive than the existing JacSON `v1.2.3-alpha` scraper;
- more consistent in how information is extracted;
- a stronger starting point for the new JacMIN scraper.

The intention is therefore to **evaluate and adopt the JacDASH scraper as the basis of the new scraping engine used by JacMIN**.

This does **not** mean that JacDASH itself becomes JacMIN.

The separation remains:

- **JacDASH** — human-readable browsing/dashboard interface;
- **JacMIN** — administrative interface, scraper controller and publishing system.

Where scraping code currently exists inside JacDASH, Codex should identify the reusable scraping functionality and refactor or migrate that functionality into an appropriate JacMIN service/module.

JacDASH should subsequently consume published API data rather than being responsible for maintaining the authoritative scraper.

---

### `jacmin/`

JacMIN is the primary application being developed by this PRD.

This directory previously contained a project called **JacDash**, but that project has now been renamed **JacMIN** to distinguish the administrative system from Sean's JacDASH interface.

JacMIN is intended to become the administrative/control plane for JacSYS.

Its responsibilities include:

- controlling course-profile scraping;
- providing the administrative WebUI;
- maintaining the list of managed courses;
- allowing authorised administrators to request scraping;
- displaying scrape status and failures;
- supporting automatic versus manual profile management;
- allowing manual editing/override of JacSON data;
- controlling regeneration of IndexSON;
- validating generated data;
- publishing approved output into `/api/v2/`;
- maintaining appropriate audit/history information;
- exposing appropriate operational controls without requiring administrators to work directly with files or command-line scripts.

Where practical, scraper implementation should belong to or be controlled by JacMIN rather than existing as an independent scheduled JacSON application.

Conceptually:

```text
                ┌───────────────┐
                │    JacMIN     │
                │ Admin + Jobs  │
                └───────┬───────┘
                        │
             ┌──────────┴──────────┐
             │                     │
       Profile Scraper       Index Generator
             │                     │
             └──────────┬──────────┘
                        │
                  Validation /
                 Override Layer
                        │
                        ▼
                   /api/v2/
```

---

### `jacson/`

Legacy/current JacSON scraper.

This currently contains the older JacSON implementation, approximately `v1.2.3-alpha`.

It is the predecessor to the scraping functionality being incorporated into JacMIN.

JacSON historically:

- retrieved UQ course profiles;
- extracted learning outcomes;
- extracted assessment information;
- extracted learning activities;
- wrote structured JSON files for consumption by other tools.

The existing implementation confirms that JacSON was designed specifically as a structured JSON extraction layer for UQ course profiles.

The existing scraper also demonstrates the previous profile-output model, where course data was saved under a semester-based `profiles/{semester_code}/` hierarchy.

The `jacson/` directory should therefore be treated as:

- legacy implementation;
- schema reference;
- migration/reference source;
- potential source of reusable extraction logic.

It should **not** remain the independent production scheduler once JacMIN assumes responsibility for scraping.

Do not delete or substantially rewrite the legacy JacSON implementation during the initial JacMIN migration unless explicitly required. Keeping it intact provides a useful reference and rollback point while the V2 pipeline is established.

---

### `jacxel/`

Reserved for the future JacXEL Excel add-in/extension.

The intended future purpose is to allow Microsoft Excel to retrieve JacSYS information through simple functions or equivalent integrations.

For example, a future JacXEL implementation may expose course-profile or index information directly from the V2 API.

**JacXEL is explicitly out of scope for the current implementation.**

Codex should:

- preserve the directory;
- avoid introducing dependencies on JacXEL;
- avoid implementing JacXEL functionality as part of this PRD.

The V2 API should nevertheless remain sufficiently clean and stable that JacXEL can consume it in the future.

---

## Component Ownership

For architectural clarity, responsibilities should be understood as follows:

| Component | Primary responsibility | Current status |
|---|---|---|
| `.codex` | Codex planning/working material | Active |
| `.dev_guides` | Infrastructure/development reference | Active |
| `api/v1` | Legacy published API | Deprecated / maintained manually |
| `api/v2` | Current published static API | Active target |
| `indexson` | Legacy index-generation implementation | Reference / migration source |
| `jacblack` | Blackboard presentation/integration layer | Active, mostly out of scope |
| `jacdash` | Human-readable dashboard | Active |
| `jacmin` | Administration, scraping, overrides, publishing | **Primary scope** |
| `jacson` | Previous JacSON scraper | Legacy / migration source |
| `jacxel` | Excel integration | Future / out of scope |

## Architectural Direction

The target architecture is therefore **not** to maintain JacSON, IndexSON and JacMIN as three separate operational pipelines.

Instead:

```text
                     UQ Jac
                        │
                        ▼
                 ┌─────────────┐
                 │   JacMIN    │
                 │             │
                 │ Scraper     │
                 │ Admin UI    │
                 │ Overrides   │
                 │ Validation  │
                 │ Indexing    │
                 │ Publishing  │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │   API V2    │
                 ├─────────────┤
                 │ profile/    │
                 │ index/      │
                 └──────┬──────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       JacDASH        JacBLACK      JacXEL
                                     (future)
```

In this model, **JacSON and IndexSON describe the data products**, while **JacMIN owns the process that creates and manages them**.

This distinction is important when refactoring the existing repository. Codex should prefer extracting useful functionality from the legacy `jacson/`, `indexson/` and `jacdash/` implementations into maintainable JacMIN modules rather than creating additional parallel scraping or indexing pipelines.

## Migration Principle

Existing code should be treated as potentially valuable source material rather than either:

1. automatically preserved because it already exists; or
2. automatically discarded because a new implementation is being created.

For each relevant legacy implementation, Codex should:

1. inspect the existing behaviour;
2. identify functionality worth retaining;
3. compare overlapping implementations where they exist;
4. adopt the stronger implementation;
5. refactor reusable functionality into the appropriate JacMIN module;
6. add tests around the adopted behaviour;
7. preserve backwards compatibility where required;
8. publish new production output exclusively through API V2 unless explicitly instructed otherwise.

In particular, the **JacDASH scraper should be evaluated first as the preferred basis for the new JacMIN scraper**, while the JacSON `v1.2.3-alpha` scraper should be used as a secondary reference to ensure existing data fields and useful behaviour are not unintentionally lost.