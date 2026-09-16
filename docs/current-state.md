# Current State — UMV Tetahali Education Data Hub

This file is the first recovery checkpoint for a new chat or operator. Values here describe the implementation state when last documented; always verify the live Supabase project before assuming counts are still current.

## Project identity

- School: UMV Tetahali / UCHCH MADHYAMIK VIDYALAY, TETAHALI
- Supabase project: `umv-db`
- Supabase project ref: `sxfnrwugsyfypqgfglzc`
- Current primary academic session: `2026-2027`
- Current OFSS admission cycle: `2026-2028`
- Historical GitHub repo name: `zzpsah/UDISE`
- Preferred descriptive repo name: `UMV-Tetahali-Data-Hub`

## Current schemas

```text
udise
e_shiksha_kosh
ofss
core
```

## UDISE

### `udise.active_student_snapshots`

- Versioned master active-student roster.
- Current documented version: **2**.
- Current documented students: **210**.
- Current views: `class_ix`, `class_x`, `class_xi`, `class_xii`.
- New master imports should use `core.import_udise_master_snapshot(...)` rather than manually editing version/current flags.

Two known historical snapshots existed at the checkpoint:

```text
v1 — 2026-09-16 — 210 rows
v2 — 2026-09-17 — 210 rows — current
```

### `udise.siwan_dropbox_master_snapshots`

Current documented master snapshot:

```text
Source date: 2026-07-11
Rows: 41,401
Unique PENs: 41,401
Blocks: 19
Eligible class counts:
  9  → 19,625
  10 →    933
  11 → 19,990
  12 →    853
```

This is a district reference/search dataset, not the active school roster.

## e-Shiksha Kosh

### `e_shiksha_kosh.student_snapshots`

- Current documented version: **1**.
- Current documented rows: **220**.
- Current source file at checkpoint: `Student_1789591912.xlsx`.
- Current class counts:

```text
Class 9  → 33
Class 10 → 34
Class 11 → 78
Class 12 → 75
```

- Stable source identifier: `student_code`.
- New master imports should use `core.import_eshiksha_master_snapshot(...)`.

## Class XI stream segregation

Persistent table:

```text
core.stream_assignments
```

Confirmed current mapping:

```text
Science   65
Arts      12
Commerce   1
Total     78
```

Current e-Shiksha stream views:

```text
e_shiksha_kosh.class_xi_science
e_shiksha_kosh.class_xi_arts
e_shiksha_kosh.class_xi_commerce
```

Known source evidence at checkpoint:

- `Student_1789597350.xlsx` — user-confirmed Class XI Science list — 65 students.
- `Student_1789597416.xlsx` — corrected/user-confirmed Class XI Arts list — 12 students; cross-checked against OFSS Arts admission data.
- Remaining Class XI student classified as Commerce: MOHAMMAD SHAD (`student_code` `202410161302252`).

Do not store these real student details in additional GitHub files; this line exists only as a recovery note and should be removed if repository privacy/security policy requires stricter minimization.

## OFSS

Physical current tables:

```text
ofss.science_2026_2028  → 67 rows
ofss.arts_2026_2028     → 12 rows
ofss.commerce_2026_2028 → 1 row
```

OFSS is a source/evidence system for Class XI admission and stream. It is not the canonical current student master.

## Core integration

Current objects:

```text
core.stream_assignments        [table]
core.snapshot_import_runs      [table]
core.student_master            [view]
core.ofss_students             [view]
core.integration_status        [view]
core.latest_snapshot_imports   [view]
```

Current documented integration status:

```text
Total current e-Shiksha students: 220
UDISE linked: 138
UDISE unlinked: 82
OFSS linked: 56
Stream classified: 78
Class XI unclassified: 0
```

Automatic cross-source matching is intentionally conservative. Do not force unmatched students together with weak fuzzy matching.

## Automatic import functions

Installed database functions:

```text
core.import_eshiksha_master_snapshot(...)
core.import_udise_master_snapshot(...)
```

They provide:

- required-input validation;
- within-file deduplication;
- duplicate-payload import detection;
- automatic next-version calculation;
- previous-current deactivation;
- new-current insertion;
- import logging in `core.snapshot_import_runs`.

See `import-workflow.md` for details.

## What is intentionally NOT implemented yet

- BSEB Registration master snapshots.
- BSEB Exam master snapshots.
- Full automatic weak/fuzzy identity resolution.
- Production portal-write automation without explicit operator approval.

Planned BSEB design:

```text
bseb.registration_snapshots
bseb.exam_snapshots
```

BSEB records should be versioned because post-correction versions may change candidate details.

## Recovery rule

When resuming this project:

1. read this file;
2. inspect live Supabase state;
3. compare live state with the documented checkpoint;
4. continue from live state, not from stale counts;
5. update this file whenever architecture or important counts materially change.
