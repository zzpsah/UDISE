# Automatic Master Import Workflow

This document defines how new **e-Shiksha Kosh** and **UDISE** master snapshots are imported into Supabase.

The preferred path is to use the database import functions so versioning, duplicate control, current-snapshot switching, downstream views, and snapshot dating are handled automatically. Manual Supabase imports are also supported if the snapshot metadata rules below are followed.

---

# 1. Snapshot date rule

The source XLSX/CSV filename does **not** determine the snapshot date.

When a new master file is imported normally, Supabase automatically stamps the database system date at import time:

```text
current_date
```

The original filename is retained only as `source_file` for provenance.

Therefore a file may have any filename. For example, all of these are acceptable:

```text
Student.xlsx
latest.xlsx
10160203806_Students_Details.xlsx
any_other_name.csv
```

The database-generated snapshot date and sequential version identify when that dataset became a new master snapshot.

For e-Shiksha Kosh the operational metadata is:

```text
updated_as_of   = system date by default
version_no      = automatically incremented
imported_at     = system timestamp
is_current      = current-master flag
source_file     = original filename only
```

For UDISE the operational metadata is:

```text
snapshot_date     = system date by default
snapshot_version  = automatically incremented
imported_at       = system timestamp
is_current        = current-master flag
source_file       = original filename only
```

If two different snapshots are imported on the same day, the version number determines their sequence.

---

# 2. e-Shiksha Kosh master import

Database function:

```text
core.import_eshiksha_master_snapshot(
  p_rows jsonb,
  p_source_file text,
  p_updated_as_of date default CURRENT_DATE,
  p_batch_label text default '2026-2027'
)
```

For normal imports, the caller does not need to supply a date. The database uses its current system date automatically.

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
2. validates `source_file`;
3. assigns the database system date when no date is supplied;
4. verifies every row has `student_code`;
5. computes a payload hash;
6. ignores the import if the exact payload was already imported successfully for that source/session;
7. removes duplicate `student_code` values inside the incoming dataset, keeping the last occurrence;
8. calculates the next `version_no` for the specified session;
9. marks the previous current rows for that session `is_current = false`;
10. inserts the new deduplicated rows with `is_current = true`;
11. records the run in `core.snapshot_import_runs`;
12. returns a JSON status result with version, snapshot date, and row counts.

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

# 3. UDISE active-student master import

Database function:

```text
core.import_udise_master_snapshot(
  p_rows jsonb,
  p_source_file text,
  p_snapshot_date date default CURRENT_DATE,
  p_school_udise_code text default '10160203806',
  p_academic_year text default '2026-2027',
  p_report_type text default 'ACTIVE_STUDENTS'
)
```

For normal imports, the caller does not need to supply a date. The database uses its current system date automatically.

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

1. validates payload and source filename;
2. assigns the database system date when no date is supplied;
3. validates every row has `class_name`;
4. computes a payload hash;
5. ignores exact repeated payload imports;
6. deduplicates the incoming dataset by the UDISE dedupe key;
7. calculates the next snapshot version for school + academic year;
8. marks the prior current snapshot non-current;
9. inserts the new snapshot as current;
10. generates `data_hash` for each row;
11. records the import in `core.snapshot_import_runs`;
12. returns version, snapshot date, and row counts.

---

# 4. Manual Supabase import rule

If a master snapshot is imported manually through the Supabase table/CSV interface instead of through the automatic import functions, Supabase's CSV importer cannot automatically calculate a shared snapshot version/date for the entire batch in the same transactional way.

For a manual import, use the actual import date as the snapshot date and keep that value identical for every row in the file.

## UDISE manual import

Every row in one manually imported UDISE snapshot should use the same:

```text
academic_year
snapshot_date     = date of manual import
snapshot_version
source_file
is_current
```

The filename itself does not need a date and must not be parsed to determine which snapshot is latest.

## e-Shiksha Kosh manual import

Every row in one manually imported e-Shiksha snapshot should use the same:

```text
batch_label
updated_as_of     = date of manual import
version_no
source_file
is_current
```

## How latest is determined

The current operational master is:

```text
is_current = true
```

Snapshot ordering is determined primarily by the sequential version within the relevant session. Date is useful for audit and human understanding.

Example:

```text
2026-09-17 / version 2
2026-09-17 / version 3   ← newer
```

The filename is provenance only.

## Manual import safety

Before marking a manually imported snapshot current:

1. use the actual system/import date consistently across all rows;
2. assign the next snapshot version consistently across all rows;
3. verify expected row count;
4. verify duplicates within the file;
5. mark the previous snapshot `is_current = false`;
6. mark the new snapshot `is_current = true`;
7. verify class views and `core.integration_status`.

The preferred path remains the automatic import functions because they perform these steps transactionally.

---

# 5. What updates automatically after import

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

# 6. Operator workflow for a new uploaded file

When a new e-Shiksha Kosh or UDISE XLSX/CSV is supplied:

```text
1. identify source system
2. inspect workbook/sheet and header rows
3. normalize columns into the importer field names
4. validate row count and required identifiers
5. convert rows to JSON array
6. call the appropriate core import function
7. let the database stamp the current system date automatically
8. read the returned status/version/counts
9. verify current master count
10. verify class views
11. verify core.integration_status
12. verify Class XI stream counts if e-Shiksha was imported
13. update docs/current-state.md when the checkpoint materially changes
```

Do not derive snapshot date from the filename. Do not manually toggle `is_current` or manually choose the next version during a normal automated import.

---

# 7. Validation after every import

Minimum checks:

```text
new snapshot version
system-generated snapshot date
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

# 8. Cross-source matching rule

Master import and cross-source reconciliation are separate concerns.

The importer preserves each source exactly enough for versioned history. The `core` layer links current records conservatively.

Never force a match solely because two names look similar. Ambiguous records should remain unmatched until better evidence is available.

---

# 9. Source-file safety

Uploaded school files may contain private student data. They belong in the working environment/Supabase only.

Do not commit source XLSX/CSV files, extracted student rows, Aadhaar data, mobile numbers, PEN lists, or student codes to GitHub.

Only non-sensitive counts, schema definitions, architecture, and operational rules belong in repository documentation.
