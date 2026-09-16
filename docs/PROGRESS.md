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
