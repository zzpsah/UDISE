# UMV Tetahali Data Architecture

This is the canonical database design for UMV Tetahali's current school-data integration across UDISE+, e-Shiksha Kosh, OFSS, and Siwan Dropbox.

## Design goals

- preserve historical source snapshots instead of overwriting them;
- keep source systems logically separate;
- derive class and stream views from current master data;
- use stable identifiers for continuity across versions;
- preserve Class XI stream assignments even when later master exports omit stream;
- automatically connect safe cross-source matches without guessing ambiguous students;
- keep sensitive student records in Supabase, never in GitHub.

## Top-level schemas

```text
udise
e_shiksha_kosh
ofss
core
```

`udise`, `e_shiksha_kosh`, and `ofss` preserve source-system data. `core` is the small automatic integration layer.

---

# 1. UDISE

## `udise.active_student_snapshots`

Versioned master roster for active UDISE students.

Important fields include:

```text
academic_year
snapshot_version
snapshot_date
is_current
source_file
source_row_number
pen
student_id
student_name
dob
gender
father_name
mother_name
class_name
section_name
status
raw_record
```

Current class views:

```text
udise.class_ix
udise.class_x
udise.class_xi
udise.class_xii
```

The class views select from the current UDISE snapshot; they are not independent copies.

### Current documented checkpoint

```text
Academic session: 2026-2027
Current snapshot version: 2
Current students: 210
```

## `udise.siwan_dropbox_master_snapshots`

Versioned searchable copy of the Siwan district Dropbox list.

Current documented source snapshot:

```text
Snapshot date: 2026-07-11
Rows: 41,401
Unique PENs: 41,401
Blocks: 19
```

The Dropbox dataset is a reference/reconciliation source. It must not be treated as the active school roster.

Future Dropbox files should become new dated versions rather than overwrite history.

---

# 2. e-Shiksha Kosh

## `e_shiksha_kosh.student_snapshots`

Canonical versioned e-Shiksha Kosh master table.

Primary stable identifier:

```text
student_code
```

Snapshot fields include:

```text
batch_label
version_no
updated_as_of
source_file
source_row_number
student_code
student_name
father_name
dob
gender
social_category
class_name
roll_no
school
aadhaar_masked
raw_record
is_current
```

Within a snapshot version, the database enforces uniqueness on:

```text
(version_no, student_code)
```

The same student appearing in multiple versions is valid historical duplication.

Current class views:

```text
e_shiksha_kosh.class_ix
e_shiksha_kosh.class_x
e_shiksha_kosh.class_xi
e_shiksha_kosh.class_xii
```

### Current documented checkpoint

```text
Academic session: 2026-2027
Current snapshot version: 1
Current students: 220
Class 9: 33
Class 10: 34
Class 11: 78
Class 12: 75
```

---

# 3. Persistent Class XI stream architecture

Stream classification is now implemented independently from the e-Shiksha snapshot row.

## `core.stream_assignments`

Persistent stream mapping keyed by e-Shiksha `student_code` and academic session.

Current fields include:

```text
student_code
academic_session
class_name
stream
source
source_reference
updated_at
```

Current confirmed Class XI assignments:

```text
Science   65
Arts      12
Commerce   1
Total     78
```

Current stream views:

```text
e_shiksha_kosh.class_xi_science
e_shiksha_kosh.class_xi_arts
e_shiksha_kosh.class_xi_commerce
```

These views resolve stream using persistent `core.stream_assignments` and fall back to a snapshot stream value if present.

Therefore a future e-Shiksha master file may omit Science/Arts/Commerce and already classified students still remain in the correct stream view.

Unknown/new Class XI students must remain unclassified until supported by a trusted source. Never infer stream from name, roll number, or weak assumptions.

---

# 4. OFSS

Current OFSS admission cycle:

```text
2026-2028
```

Physical tables:

```text
ofss.science_2026_2028
ofss.arts_2026_2028
ofss.commerce_2026_2028
```

Current documented counts:

```text
Science  67
Arts     12
Commerce  1
Total    80
```

OFSS remains a separate source system. It is useful for Class XI admission and stream evidence but does not replace UDISE or e-Shiksha Kosh.

The current integration exposes OFSS through:

```text
core.ofss_students
```

---

# 5. Core integration layer

The `core` schema is intentionally small. It should be automatically maintained by database logic and import workflows rather than by manual row-by-row editing.

## `core.student_master`

Unified current operational view based primarily on the current e-Shiksha master, enriched with:

- persistent stream assignment;
- safe UDISE link where available;
- OFSS link where available.

It does not replace source history. The original source schemas remain authoritative for what each system reported.

## `core.integration_status`

Summary view for current integration health.

Current documented checkpoint:

```text
Total current e-Shiksha students: 220
UDISE linked: 138
UDISE unlinked: 82
OFSS linked: 56
Stream classified: 78
Class XI unclassified: 0
```

These counts may change after future imports and must be verified live.

## `core.ofss_students`

Normalized union view over the current OFSS stream tables.

## `core.snapshot_import_runs`

Import audit/log table used by the automatic master-snapshot functions.

It stores source system, academic session, source file, snapshot version/date, input rows, inserted rows, duplicate rows, payload hash, status, and timing information.

## `core.latest_snapshot_imports`

Convenience view for recent/current import-run information.

---

# 6. Cross-source matching policy

Automatic linking must be conservative.

Current safe e-Shiksha ↔ UDISE rule uses normalized student name + father name only where the result is unique and unambiguous. At the documented checkpoint this yielded 138 links and zero ambiguous automatic matches.

Do not silently use weak fuzzy matching to force the remaining records together.

Preferred future matching priority:

```text
1. exact stable identifier where a verified cross-source mapping exists
2. exact normalized demographic combination with unique result
3. manual confirmation for unresolved/ambiguous cases
```

An unmatched student is safer than a wrong student link.

---

# 7. Master snapshot rules

UDISE and e-Shiksha Kosh are the two automatically managed master snapshots at present.

The rule is:

```text
new source file
    ↓
validate required fields
    ↓
deduplicate inside incoming dataset
    ↓
ignore exact repeated payload import
    ↓
calculate next version for session
    ↓
mark previous snapshot non-current
    ↓
insert new snapshot as current
    ↓
derived views/core update automatically
```

Historical versions are retained.

OFSS remains separate and is not currently managed by the same frequent master-snapshot cycle.

See `import-workflow.md` for exact importer behavior.

---

# 8. Session policy

Current sessions:

```text
UDISE / e-Shiksha Kosh: 2026-2027
OFSS:                  2026-2028
```

Future sessions must be added as new session-aware records/snapshots while preserving prior years.

Version numbering for e-Shiksha imports is session-aware.

---

# 9. Planned BSEB architecture

BSEB is planned but not yet implemented.

When added, the intended schema is:

```text
bseb
├── registration_snapshots
└── exam_snapshots
```

BSEB should also use versioned snapshots because candidate records can change after corrections. Registration number and BSEB unique/candidate identifiers should be preserved as authoritative keys where available.

Do not add BSEB tables until the source format has been inspected and explicitly approved.

---

# 10. Security boundaries

- Never commit real student-level records to GitHub.
- Never commit Aadhaar details, credentials, OTPs, cookies, access tokens, session keys, or private source exports.
- Supabase is the restricted operational data layer.
- GitHub contains architecture, code, migrations, non-sensitive schemas, and documentation.
- Portal login/CAPTCHA/OTP remains human-controlled.
- Consequential portal writes require explicit approval and verification.
- Production deployment requires explicit approval.

---

# 11. Core principle

Source systems remain separate and historical:

```text
UDISE             → active school enrolment/status
E-Shiksha Kosh    → state student master
OFSS              → Class XI admission/stream evidence
Siwan Dropbox     → district search/reference
Core              → automatic safe linking only
```

The integration layer should simplify work, not erase source provenance.
