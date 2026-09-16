# UMV Tetahali Education Data Hub

Private-data architecture and operational documentation for UMV Tetahali's school data workflows across **UDISE+**, **e-Shiksha Kosh**, **OFSS**, and district Dropbox datasets.

> Student-level records, credentials, cookies, OTPs, session keys, and other sensitive school data must stay in restricted systems such as Supabase. GitHub should contain architecture, migrations, code, and documentation only.

## Current database architecture

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
└── ofss
    ├── science_2026_2028
    ├── arts_2026_2028
    └── commerce_2026_2028
```

## Architecture principles

### 1. Snapshot-first design

Master source files are stored as dated/versioned snapshots rather than overwritten. Historical versions remain available for audit and comparison, while `is_current = true` identifies the current operational snapshot.

### 2. Derived class views

Class-wise datasets are database views derived from the current master snapshot. They are not independent duplicate copies. Updating the current master snapshot automatically changes the class views.

### 3. e-Shiksha Kosh stream segregation

Class XI currently has confirmed stream segregation:

- Science: 65 students
- Arts: 12 students
- Commerce: 1 student

The stable student identifier is `student_code`.

The long-term design should preserve stream assignment independently from snapshot imports so that a future master CSV/XLSX without a stream column does not lose Science/Arts/Commerce classification. The intended model is a persistent `stream_assignments` table keyed by `student_code`, joined to the current master snapshot by the stream views.

See `docs/data-architecture.md` for the target import and duplicate-handling design.

### 4. Duplicate policy

A student may appear in multiple snapshot versions because each snapshot is historical. Within a single e-Shiksha Kosh version, `(version_no, student_code)` is unique, preventing accidental duplicate rows inside the same snapshot.

### 5. OFSS remains a separate source system

OFSS admission data is maintained separately from e-Shiksha Kosh and UDISE. Current session tables are `science_2026_2028`, `arts_2026_2028`, and `commerce_2026_2028`. Cross-system reconciliation should be explicit rather than silently merging records.

### 6. UDISE Dropbox is a searchable historical source

`udise.siwan_dropbox_master_snapshots` stores versioned Siwan district Dropbox reports. Future Dropbox files should be added as new dated snapshot versions, never overwrite previous snapshots.

## Sessions currently represented

- UDISE / e-Shiksha Kosh: `2026-2027`
- OFSS admission cycle: `2026-2028`

Future sessions should be added without destroying prior-year history.

## Safety and workflow boundaries

1. Login, password, CAPTCHA and OTP remain under human control.
2. Read-only discovery, search, reconciliation and report generation may be automated.
3. Consequential portal changes require validation and explicit operator approval.
4. Credentials and private student records must never be committed to GitHub.
5. Supabase is the restricted operational data store; GitHub is the architecture/code/documentation layer.
6. Production deployment must not occur without explicit approval.

## Documentation

- [`docs/data-architecture.md`](docs/data-architecture.md) — canonical database architecture, snapshot/version rules, stream inheritance, and import behavior.

## Current checkpoint

- UDISE current roster is versioned in `active_student_snapshots` with class views derived from the current snapshot.
- Siwan Dropbox master snapshot is loaded and searchable as a versioned source dataset.
- e-Shiksha Kosh master snapshot is maintained in `student_snapshots` with Class IX-XII views.
- Class XI e-Shiksha Kosh stream segregation is complete: Science 65, Arts 12, Commerce 1.
- OFSS 2026-2028 is separated into Science, Arts and Commerce source tables.
- Cross-system reconciliation remains intentionally separate from source ingestion.
