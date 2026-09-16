# Automatic Master Import Workflow

This document defines how new **e-Shiksha Kosh** and **UDISE** master snapshots are imported into Supabase.

The preferred path is to use the database import functions so versioning, duplicate control, current-snapshot switching, and downstream views are handled automatically. Manual Supabase imports are also supported if the snapshot metadata rules below are followed.

---

# 1. e-Shiksha Kosh master import

Database function:

```text
core.import_eshiksha_master_snapshot(
  p_rows jsonb,
  p_source_file text,
  p_updated_as_of date,
  p_batch_label text default '2026-2027'
)
```

## Required input

`p_rows` must be a non-empty JSON array derived from the uploaded XLSX/CSV.

Every row must contain:

```text
student_code
```

Expected optional/normal source fields include:

```text
student_name
father_name
dob
gender
social_category
class_name
roll_no
school
aadhaar_masked
stream
source_row_number
```

The source file does **not** need to contain `stream` for already classified Class XI students because persistent stream mapping lives in `core.stream_assignments`.

## Automatic behavior

The function:

1. validates that the payload is a non-empty array;
2. validates `source_file` and `updated_as_of`;
3. verifies every row has `student_code`;
4. computes a payload hash;
5. ignores the import if the exact payload was already imported successfully for that source/session;
6. removes duplicate `student_code` values inside the incoming dataset, keeping the last occurrence;
7. calculates the next `version_no` for the specified session;
8. marks the previous current rows for that session `is_current = false`;
9. inserts the new deduplicated rows with `is_current = true`;
10. records the run in `core.snapshot_import_runs`;
11. returns a JSON status result with version and row counts.

## Duplicate rules

Valid historical duplicate:

```text
same student_code in version 1 and version 2
```

Invalid within-file duplicate:

```text
same student_code repeated twice in the same incoming file
```

The importer deduplicates the second case automatically.

Exact repeated payload import:

```text
status = IGNORED_DUPLICATE_IMPORT
```

No new version should be created for the exact same payload.

---

# 2. UDISE active-student master import

Database function:

```text
core.import_udise_master_snapshot(
  p_rows jsonb,
  p_source_file text,
  p_snapshot_date date,
  p_school_udise_code text default '10160203806',
  p_academic_year text default '2026-2027',
  p_report_type text default 'ACTIVE_STUDENTS'
)
```

## Required input

`p_rows` must be a non-empty JSON array.

Every row must contain:

```text
class_name
```

Useful source fields include:

```text
pen
student_id
student_name
dob
gender
father_name
mother_name
guardian_name
section_name
aadhaar_number
aadhaar_name
mobile_number
apaar_id
event_date
status
source_row_number
```

## Automatic dedupe key

The importer chooses the strongest available key in this order:

```text
1. PEN
2. UDISE student_id
3. fallback demographic hash of normalized student_name + dob + father_name
```

The fallback exists only for within-file deduplication when stable source identifiers are missing. It is not a license to fuzzy-link uncertain students across different systems.

## Automatic behavior

The function:

1. validates payload/source/date;
2. validates every row has `class_name`;
3. computes a payload hash;
4. ignores exact repeated payload imports;
5. deduplicates the incoming dataset by the UDISE dedupe key;
6. calculates the next snapshot version for school + academic year;
7. marks the prior current snapshot non-current;
8. inserts the new snapshot as current;
9. generates `data_hash` for each row;
10. records the import in `core.snapshot_import_runs`;
11. returns version and row counts.

---

# 3. Manual Supabase import rule

If a master snapshot is imported manually through the Supabase table/CSV import interface, every row in that file must carry consistent snapshot metadata.

## UDISE manual import

Every row in one manual UDISE snapshot should use the same:

```text
academic_year
snapshot_date
snapshot_version
source_file
is_current
```

Recommended filename convention:

```text
UDISE_Active_Students_2026-09-17.xlsx
```

The database should identify the newest UDISE snapshot by:

```text
academic_year + snapshot_date + snapshot_version
```

`is_current` is an operational flag, not the only evidence of which snapshot is newest.

## e-Shiksha Kosh manual import

Every row in one manual e-Shiksha snapshot should use the same:

```text
batch_label
updated_as_of
version_no
source_file
is_current
```

Recommended filename convention:

```text
eShikshaKosh_Master_2026-09-17.xlsx
```

The database should identify the newest e-Shiksha snapshot by:

```text
batch_label + updated_as_of + version_no
```

## Why both date and version are required

The **date** makes the snapshot understandable to a human and allows chronological sorting. The **version** resolves cases where two different snapshots are imported on the same date.

Example:

```text
2026-09-17 / version 2
2026-09-17 / version 3   ← newer even though the date is the same
```

The filename date is useful for humans, but the database must rely on the actual date/version columns, not filename text alone.

## Manual import safety

Before marking a manually imported snapshot current:

1. verify all rows share the same date/version/session;
2. verify expected row count;
3. verify duplicates within the file;
4. mark the previous snapshot `is_current = false`;
5. mark the new snapshot `is_current = true`;
6. verify class views and `core.integration_status`.

The preferred path remains the automatic import functions because they perform these steps transactionally.

---

# 4. What updates automatically after import

After a successful master import, current-state views update because they read from `is_current = true` source rows.

For e-Shiksha Kosh this includes:

```text
e_shiksha_kosh.class_ix
e_shiksha_kosh.class_x
e_shiksha_kosh.class_xi
e_shiksha_kosh.class_xii
e_shiksha_kosh.class_xi_science
e_shiksha_kosh.class_xi_arts
e_shiksha_kosh.class_xi_commerce
core.student_master
core.integration_status
```

For UDISE this includes:

```text
udise.class_ix
udise.class_x
udise.class_xi
udise.class_xii
core.student_master
core.integration_status
```

The Class XI stream views preserve existing stream assignments through `core.stream_assignments` even if a new e-Shiksha master has no stream field.

---

# 5. Operator workflow for a new uploaded file

When a new e-Shiksha Kosh or UDISE XLSX/CSV is supplied:

```text
1. identify source system
2. inspect workbook/sheet and header rows
3. normalize columns into the importer field names
4. validate row count and required identifiers
5. convert rows to JSON array
6. call the appropriate core import function
7. read the returned status/version/counts
8. verify current master count
9. verify class views
10. verify core.integration_status
11. verify Class XI stream counts if e-Shiksha was imported
12. update docs/current-state.md when the checkpoint materially changes
```

Do not manually toggle `is_current` or manually choose the next version during a normal automated import.

---

# 6. Validation after every import

Minimum checks:

```text
new snapshot version
current row count
previous snapshot retained
only intended session marked current
within-file duplicates reported
class totals sum to master total
stream counts remain valid
core integration status is reasonable
```

If a count unexpectedly drops, do not silently accept the import. Compare source row count, required-field failures, and duplicate count first.

---

# 7. Cross-source matching rule

Master import and cross-source reconciliation are separate concerns.

The importer preserves each source exactly enough for versioned history. The `core` layer links current records conservatively.

Never force a match solely because two names look similar. Ambiguous records should remain unmatched until better evidence is available.

---

# 8. Source-file safety

Uploaded school files may contain private student data. They belong in the working environment/Supabase only.

Do not commit source XLSX/CSV files, extracted student rows, Aadhaar data, mobile numbers, PEN lists, or student codes to GitHub.

Only non-sensitive counts, schema definitions, architecture, and operational rules belong in repository documentation.
