# UMV Tetahali UDISE+ Automation Project

Cloud-ready project for authorised UDISE+ discovery, validation, reporting, reconciliation, and controlled workflow automation for academic session **2026-2027**.

## Current data systems

- **UDISE+ / SDMS**: student profile, enrolment, facility, progression, APAAR, Aadhaar-status and reporting workflows.
- **Supabase `umv-db`**: restricted operational and historical datasets.
- **eShiksha Kosh**: maintained separately as a versioned source dataset and linked to UDISE history where matched.
- **OFSS**: Class XI admission datasets kept as separate source datasets and reconciled against UDISE when appropriate.

## Supabase structure

### `udise`

- `student_history` — historical/audit UDISE snapshots; current 2026-2027 load: 210 rows.
- `profile_updates` — controlled EP/GP/FP change queue; no portal mutation should occur without validation and explicit approval.
- `report_imports` — source report manifest/import register.
- `dropbox_students` — separate Dropbox dataset; never merge into active enrolment without reconciliation.
- `dataset_catalog` — dataset purpose, duplicate policy and operational guidance.
- class views: `class_ix`, `class_x`, `class_xi`, `class_xii`.
- OFSS source tables for Science, Arts and Commerce admissions for 2026-2027.

### `eshiksha_kosh`

- `student_snapshots` — canonical versioned eShiksha Kosh snapshot table, with UDISE linkage fields for reconciliation.

## Academic-session policy

All operational records are session-aware. Canonical session values use the format:

```text
2026-2027
```

Future progression should retain historical rows and advance the active session to the next academic year rather than overwriting prior-year data.

## Safety and workflow boundaries

1. UDISE+ login, password, CAPTCHA and OTP remain under human control.
2. Read-only discovery and exports may be automated.
3. EP/GP/FP changes must pass validation, before/after review and explicit approval before submission.
4. Consequential actions such as certification, finalisation, import, Dropbox movement, Aadhaar verification/revalidation and progression submission require explicit operator approval.
5. Do not store credentials, session cookies, encryption keys or private student data in this public repository.
6. Student-level sensitive data belongs in restricted systems such as Supabase, not GitHub.

## Current checkpoint

- Authenticated read-only SDMS discovery completed across the main student modules.
- 210-row UDISE student history dataset loaded for 2026-2027.
- Class views reconcile to 210 students.
- eShiksha Kosh snapshots are maintained separately and linked to UDISE where matched.
- OFSS admission datasets are stored separately for later reconciliation.
- Supabase RLS is enabled on the UDISE and eShiksha Kosh data tables.

## Next operations

Build the authorised operator workflow around reconciliation, validation, query/export generation and controlled EP/GP/FP updates while preserving audit history and session separation.
