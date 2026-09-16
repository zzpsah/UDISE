# UMV Tetahali UDISE+ Progress Log

## 2026-09-17 — Repository and discovery

- Confirmed `https://github.com/zzpsah/UDISE` was accessible but empty.
- Created the repository documentation and project handoff.
- Created a dedicated project task: `UMV Tetahali UDISE 2026-2027`.
- Verified central login through `https://auth.udiseplus.gov.in/login` and read-only SDMS access.
- Inspected School Dashboard, School Profile, List of All Students, EP/GP/FP profile flow, APAAR, Aadhaar MBU/Capture Status, Movement/Progression, Global Search, Track Student, Aadhaar Availability, Duplicate Records, Reporting, Certification, and validation guidance.
- No portal write, certification, finalisation, import, Dropbox action, Aadhaar revalidation, or APAAR generation was performed.

## 2026-09-17 — Exports and Supabase

- Downloaded and verified the List of All Students Excel export: 210 records, four classes, and 210 distinct PEN values.
- Verified columns include class/section, identity, PEN, parents, category, BPL/CWSN, entry status, Aadhaar validation/MBU, and APAAR status.
- Downloaded available reporting Excel files. Dropbox, inactive, imported, new-student, progression, and APAAR history files contained headers but no data rows at export time.
- In Supabase project `umv-db` (`ap-south-1`), created restricted schema `udise` with `student_history`, `dropbox_students`, `profile_updates`, `report_imports`, `dataset_catalog`, and class views.
- Enabled RLS on UDISE tables with no public policies.
- Loaded the 210-row List of All Students export into `udise.student_history` in batches and verified 210 rows, four classes, and 210 distinct PENs.
- Updated `udise.report_imports` to `active_student_list / 210 / IMPORTED`.

## 2026-09-17 — OFSS Science admission mapping

- Inspected `Admission_Register_9172026_020833AM.xls`; it contains 67 admission rows and 15 source columns.
- Created restricted Supabase table `udise.ofss_science_admissions_2026_27` for OFSS Class XI Science admissions.
- Loaded all 67 source rows with original values and a `raw_record` JSON copy.
- Matched 41 records to `udise.student_history` using normalized applicant name plus father name.
- The 41 matched rows include masked APAAR ID and APAAR status from the UDISE export. No unmasked APAAR values were available.
- 26 rows remain `UNMATCHED`; no fuzzy or speculative assignments were made.
- RLS is enabled with no public policies. No UDISE portal records were edited.

## 2026-09-17 — OFSS Arts and Commerce sources

- Classified `Admission_Register_9172026_021112AM.xls` as Arts: 1 source row.
- Classified `Admission_Register_9172026_021106AM.xls` as Commerce: 12 source rows.
- Stored them separately in `udise.ofss_arts_admissions_2026_27` and `udise.ofss_commerce_admissions_2026_27`.
- Each table preserves the OFSS `reference_no` as the source key and stores an `identity_key` made from normalized applicant name, father name, and DOB.
- Exact composite reconciliation against the 2026-27 UDISE student export produced 0 confirmed matches in these two files; no PEN or APAAR assignment was made.
- Added both datasets to `udise.dataset_catalog` as non-canonical, sensitive reference sources with independent duplicate handling.
- RLS remains enabled and no UDISE portal records were changed.

## 2026-09-17 — eShiksha Kosh refreshable snapshot

- Inspected `Student_1789591912.xlsx`, generated 17-Sep-2026 for UMV Tetahali.
- The workbook contains 220 student rows across Classes 9, 10, 11, and 12, with student code, student name, father name, DOB, gender, category, roll number, school, and masked Aadhaar fields.
- Loaded the current snapshot into `udise.eshiksha_kosh_students` with batch `2026-2027`, version `1`, and `updated_as_of = 2026-09-17`.
- The table is designed for replacement by future versioned snapshots: keep prior versions for audit, mark only the newest approved version `is_current = true`, and query current data by batch/version/class.
- RLS is enabled and no public policies were added. Raw student rows are not stored in GitHub.

## 2026-09-17 — OFSS batch naming

- OFSS is now treated as a separate batch namespace: `OFSS / 2026-2027 / Class XI`.
- Science, Arts, and Commerce remain separate datasets within that batch.
- Added `batch_label = 2026-2027` to all three OFSS tables and documented each table accordingly.
- Original uploaded filenames remain preserved in `source_file` for traceability.
- Future files should use the naming pattern `OFSS_2026-2027_Class-XI_<STREAM>_<SOURCE>.<ext>`.
- Planned Storage paths are `OFSS/2026-2027/Class-XI/Science/`, `Arts/`, and `Commerce/`; Storage tooling was not available in this session.

## Data policy

- `student_history` is audit/reference data and may contain repeated snapshots; it is not the canonical update source.
- `dropbox_students` is separate and must not be merged into active enrolment without reconciliation.
- `profile_updates` is the operational EP/GP/FP proposal and approval queue.
- Passwords, CAPTCHA, OTPs, cookies, session tokens, and encryption keys must never enter GitHub, Supabase tables, or cloud job payloads.

## Next work

1. Create private Supabase Storage paths for source reports.
2. Add ingestion scripts, mapping, duplicate reconciliation, tests, and CI.
3. Complete dedicated EP and FP option/validation capture.
4. Build class-wise query/export views.
5. Build before/after EP/GP/FP proposal workbooks.
6. Add explicit approval and post-save verification before any portal write workflow.

## Open items

- Supabase Storage management tools were unavailable during this run.
- Official UDISE+ API eligibility is not confirmed.
- The GitHub repository began with no commits; this documentation is the first project baseline.
