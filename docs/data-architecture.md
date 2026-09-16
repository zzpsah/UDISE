# UMV Tetahali Data Architecture

This document is the canonical design for the school data model used across UDISE+, e-Shiksha Kosh, OFSS, and Siwan Dropbox sources.

## Goals

- preserve historical snapshots instead of overwriting source data;
- keep source systems logically separate;
- derive class/stream views from master data instead of duplicating independent tables;
- use stable identifiers for cross-version continuity;
- make imports idempotent inside a snapshot version;
- avoid losing stream classification when a later source export omits stream information;
- keep sensitive student data in Supabase, not GitHub.

## Top-level schemas

```text
udise
e_shiksha_kosh
ofss
```

These schemas are the main logical containers in PostgreSQL/Supabase.

---

## UDISE

### `udise.active_student_snapshots`

Versioned master roster for active UDISE students.

Important snapshot fields include:

- `academic_year`
- `snapshot_version`
- `snapshot_date`
- `is_current`
- `source_file`
- `source_row_number`
- student identifiers and demographic fields

Current class views:

```text
udise.class_ix
udise.class_x
udise.class_xi
udise.class_xii
```

Each class view should select only rows from the current snapshot. No separate physical copy of each class roster is required.

### `udise.siwan_dropbox_master_snapshots`

Versioned master searchable copy of the Siwan district Dropbox student report.

Future Dropbox files must be inserted as new dated versions. Historical rows must not be overwritten.

Search/reconciliation keys include PEN, student name, block, previous school UDISE code, previous school name, and eligible class.

---

## e-Shiksha Kosh

### `e_shiksha_kosh.student_snapshots`

Canonical versioned master table for e-Shiksha Kosh exports.

Current class views:

```text
e_shiksha_kosh.class_ix
e_shiksha_kosh.class_x
e_shiksha_kosh.class_xi
e_shiksha_kosh.class_xii
```

A student may exist in several snapshot versions. This is expected historical duplication, not an error.

Within one snapshot version, the database enforces uniqueness on:

```text
(version_no, student_code)
```

This prevents the same Student Code from being loaded twice into the same version.

### Class XI stream classification

Current confirmed Class XI classification:

```text
Science   65
Arts      12
Commerce   1
Total     78
```

Current stream views are:

```text
e_shiksha_kosh.class_xi_science
e_shiksha_kosh.class_xi_arts
e_shiksha_kosh.class_xi_commerce
```

The stable key used for stream continuity is `student_code`.

## Target persistent stream architecture

A new master e-Shiksha Kosh export may contain students and classes but no Science/Arts/Commerce field. Stream information therefore must not depend exclusively on the imported snapshot row.

The durable target design is:

```text
e_shiksha_kosh.student_snapshots
        │
        │ student_code
        ▼
e_shiksha_kosh.stream_assignments
        │
        ├── class_xi_science [view]
        ├── class_xi_arts    [view]
        └── class_xi_commerce[view]
```

Recommended `stream_assignments` fields:

```text
student_code      text
academic_session  text
class_name        text
stream            text
source_type       text
source_reference  text
assigned_at       timestamptz
is_current        boolean
```

Recommended uniqueness:

```text
(student_code, academic_session, class_name)
```

This means one Student Code has one active stream classification for the same class/session.

## New master import behavior

When a new e-Shiksha Kosh XLSX/CSV arrives:

1. Create a new `version_no` and snapshot date.
2. Load each Student Code once for that version.
3. Mark the previous snapshot `is_current = false`.
4. Mark the new snapshot `is_current = true` after validation.
5. Do not require the source file to contain a stream column.
6. Resolve stream views by joining current master rows to persistent `stream_assignments` by `student_code` and session/class.
7. Existing students automatically retain their known stream assignment.
8. New Class XI students with no assignment remain unclassified until a trusted stream source is supplied.
9. Do not infer unknown streams from names or roll numbers.
10. Report unmatched/new students so their stream can be assigned explicitly.

### Duplicate handling

There are two different kinds of apparent duplicates:

**Historical duplicate — valid**

The same Student Code appears in Version 1 and Version 2. This preserves history and should remain.

**Within-version duplicate — invalid**

The same Student Code appears twice inside Version 2. The unique `(version_no, student_code)` constraint should reject or de-duplicate this before finalizing the import.

The import pipeline should therefore be idempotent at the snapshot-version level.

## Stream source provenance

Stream assignments should retain provenance. Examples:

- filtered e-Shiksha Kosh export supplied specifically as Class XI Science;
- filtered e-Shiksha Kosh export supplied specifically as Class XI Arts;
- OFSS admission list used for cross-checking classification;
- future authoritative school stream register.

Do not silently replace a trusted stream assignment from an unrelated generic master export.

---

## OFSS

Current OFSS session: `2026-2028`.

Physical source tables:

```text
ofss.science_2026_2028
ofss.arts_2026_2028
ofss.commerce_2026_2028
```

OFSS should remain a separate source system. It may support reconciliation or stream verification but should not be silently merged into e-Shiksha Kosh or UDISE master rows.

---

## Session rules

Current sessions:

```text
UDISE / e-Shiksha Kosh: 2026-2027
OFSS:                  2026-2028
```

New academic sessions should create new snapshots/session-aware records while retaining previous history.

---

## Security boundaries

- Never commit real student-level data to GitHub.
- Never commit passwords, OTPs, cookies, tokens, session keys, or private endpoint secrets.
- Supabase is the restricted operational data layer.
- GitHub stores architecture, migrations, code, schemas, documentation, and non-sensitive examples.
- Portal writes require explicit operator approval and verification.
- Production deployment requires explicit approval.

---

## Current implementation gap

The current Class XI stream views use a `stream` value on `student_snapshots`. The persistent `stream_assignments` layer described above is the intended next hardening step before future stream-less master imports are treated as fully automatic.

Until that table and join-based views are implemented, a new master import without stream values could cause current stream views to become incomplete. This architecture document therefore defines the migration target before the next master snapshot workflow is finalized.
