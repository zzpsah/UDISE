# UMV Tetahali Education Data Hub

Durable architecture and operational documentation for UMV Tetahali's school data workflows across **UDISE+**, **e-Shiksha Kosh**, **OFSS**, and Siwan district Dropbox data.

> **Important:** student-level records, credentials, cookies, OTPs, session keys, and other sensitive school data belong in restricted systems such as Supabase. GitHub should contain architecture, migrations, code, and non-sensitive documentation only.

## Source-of-truth rule

For a new chat, machine, or operator, do **not** rely on conversation memory. Recover the project from this repository plus the live Supabase schema.

Recommended recovery order:

1. Read [`docs/current-state.md`](docs/current-state.md).
2. Read [`docs/data-architecture.md`](docs/data-architecture.md).
3. Read [`docs/import-workflow.md`](docs/import-workflow.md).
4. Check the live Supabase project before making changes.
5. Follow [`docs/new-chat-handoff.md`](docs/new-chat-handoff.md) for continuation rules.

## Current Supabase architecture

Supabase project: `umv-db`

```text
umv-db
├── udise
│   ├── active_student_snapshots
│   ├── siwan_dropbox_master_snapshots
│   ├── class_ix      [view]
│   ├── class_x       [view]
│   ├── class_xi      [view]
│   └── class_xii     [view]
│
├── e_shiksha_kosh
│   ├── student_snapshots
│   ├── class_ix              [view]
│   ├── class_x               [view]
│   ├── class_xi              [view]
│   ├── class_xii             [view]
│   ├── class_xi_science      [view]
│   ├── class_xi_arts         [view]
│   └── class_xi_commerce     [view]
│
├── ofss
│   ├── science_2026_2028
│   ├── arts_2026_2028
│   └── commerce_2026_2028
│
└── core
    ├── stream_assignments
    ├── snapshot_import_runs
    ├── student_master         [view]
    ├── ofss_students          [view]
    ├── integration_status     [view]
    └── latest_snapshot_imports[view]
```

## Current integration state

Current operational counts at the documented checkpoint:

- e-Shiksha Kosh current master: **220** students, version **1**.
- UDISE current active-student master: **210** students, version **2**.
- Class XI stream segregation: **78/78** classified.
  - Science: **65**
  - Arts: **12**
  - Commerce: **1**
- Automatic e-Shiksha ↔ UDISE safe links: **138**.
- Automatic OFSS links visible in the core integration: **56**.
- Remaining e-Shiksha students without a safe UDISE link are intentionally left unmatched rather than guessed.

These numbers are a checkpoint, not a permanent constant. Always verify live Supabase state before relying on them.

## Automatic master-snapshot imports

Two database import functions are installed:

```text
core.import_eshiksha_master_snapshot(...)
core.import_udise_master_snapshot(...)
```

They handle version assignment, duplicate removal within the incoming dataset, previous-current deactivation, new-current activation, and duplicate-import detection.

The exact workflow and required input fields are documented in [`docs/import-workflow.md`](docs/import-workflow.md).

## Stream persistence

Class XI stream is stored independently in:

```text
core.stream_assignments
```

The stream views join current e-Shiksha master rows to this persistent mapping by `student_code` and academic session. Therefore, a future e-Shiksha master file **does not need to contain a stream column** for already classified students.

## Source roles

- **UDISE+** — active school enrolment/status source and historical snapshots.
- **e-Shiksha Kosh** — state student master source, keyed primarily by `student_code`.
- **OFSS** — Class XI admission and stream evidence for the 2026-2028 admission cycle.
- **Siwan Dropbox** — searchable district reference/reconciliation source, not active-enrolment truth.
- **core** — automatic linking and persistent cross-source metadata; not a replacement for source history.

## Sessions currently represented

- UDISE / e-Shiksha Kosh: `2026-2027`
- OFSS admission cycle: `2026-2028`

Future sessions must retain previous history rather than overwrite it.

## Planned data sources

BSEB is intentionally postponed for now. When added later, the planned logical schema is:

```text
bseb
├── registration_snapshots
└── exam_snapshots
```

BSEB data should be versioned because registration/exam records may change after corrections. Registration number and BSEB unique/candidate identifiers should be retained as authoritative linking keys where available.

## Safety and workflow boundaries

1. Login, password, CAPTCHA and OTP remain under human control.
2. Read-only discovery, search, reconciliation and report generation may be automated.
3. Consequential portal changes require validation and explicit operator approval.
4. Never commit real student records, credentials, session data, Aadhaar data, or private source files to GitHub.
5. Supabase is the restricted operational data store; GitHub is the architecture/code/documentation layer.
6. Production deployment must not occur without explicit approval.
7. Do not silently fuzzy-match uncertain students. Ambiguous cases remain unmatched for review.

## Documentation index

- [`docs/current-state.md`](docs/current-state.md) — exact implementation checkpoint and live-object inventory.
- [`docs/data-architecture.md`](docs/data-architecture.md) — canonical database architecture and design rules.
- [`docs/import-workflow.md`](docs/import-workflow.md) — UDISE/e-Shiksha master import rules and database functions.
- [`docs/new-chat-handoff.md`](docs/new-chat-handoff.md) — instructions for recovering and continuing the project in a new chat.

## Repository administration note

The project name used in documentation is **UMV Tetahali Education Data Hub**. A more descriptive repository name than the historical `UDISE` name is recommended, such as `UMV-Tetahali-Data-Hub`. Repository visibility should be private before storing any additional non-public implementation material. No private student data should be committed even after making the repository private.
