# UDISE+ Project Handoff — UMV Tetahali

## Identity and status

- School: UMV Tetahali
- Session: 2026–2027
- Authentication: `https://auth.udiseplus.gov.in/login`
- Student system: SDMS
- Repository purpose: durable context for future AI agents and enhancements
- Current state: read-only portal discovery completed; no UDISE+ record was edited, saved, submitted, imported, deleted, certified, finalised, Aadhaar revalidated, or sent to Dropbox.

## Portal inventory

Observed modules: School Dashboard, School Profile, List of All Students, APAAR, Aadhaar MBU, Aadhaar Capture Status, Class/Section Shift, Progression Activity, Import, Send to Dropbox, Inactivate Student, Dropbox Student List, Reporting, School Certification, Duplicate Records, Global Student Search, Track Student, Form Fields Validation Guide, and Student Aadhaar Availability.

## EP/GP/FP

Student profile has General Profile, Enrolment Profile, Facility Profile, and Profile Preview tabs.

General Profile: name, gender, DOB, state code, mother/father/guardian, Aadhaar and Aadhaar name, address/pincode, primary/alternate mobile, email, mother tongue, social/minority category, BPL/AAY/EWS, CWSN/impairment/disability, nationality, out-of-school/mainstreaming, blood group.

Enrolment Profile: admission number/date, roll number, medium, academic stream, dynamic subjects, previous-year schooling status/class, RTE, exam result, marks, attendance.

Facility Profile: textbook, uniform, transport, bicycle, hostel, mobile/tablet/computer, CWSN facilities, competitions/Olympiads, NCC, NSS, Scouts and Guides, height, weight, residence distance, guardian education.

Rules include DD/MM/YYYY dates, admission-date constraints, mobile/pincode validation, conditional BPL/CWSN/out-of-school/stream fields, maximum three impairments, Aadhaar-verified-field restrictions, and APAAR identity restrictions. The portal provides per-field rule references and an Age Matrix.

## Main modules

### List of All Students

Class, entry-status, Aadhaar-status, sort/order, and text-search filters. Export columns include class, section, name, gender, initialisation year, PEN, state code, parents, social/minority category, BPL, CWSN, impairment, repeater/duplicate flags, entry status, Aadhaar validation/MBU status, APAAR ID/status. A populated Excel export was downloaded and verified locally with 210 records.

### Search and tracking

Global Student Search supports PEN, student name, and Aadhaar last-four search. Name search accepts optional DOB, parents, state, district, block, and school; the portal requires a name plus at least two optional fields. Track Student accepts PEN and shows identity, school history, Aadhaar/APAAR status, last update, and schooling details.

### APAAR and Aadhaar

APAAR supports class/section selection, search, status table, generation history, class-wise statistics, and APAAR-status download. Generation requests were not activated.

Aadhaar MBU supports class/Aadhaar filters, search, counts, student details, validation/MBU status, export, and re-validation controls. Capture Status gives school/class totals for Aadhaar provided, UIDAI verification, MBU age bands, not required, not applicable, and status-check pending. Aadhaar availability accepts a 12-digit number; no number was entered during discovery.

### Movement, duplicates, certification

Progression contains manual PDF, activity, section summary, schooling-status correction, student-wise activity, and finalization. Movement actions and finalization were not activated. Duplicate Records searches by PEN; the inspected school returned no records. Certification requires all students completed and no pending requests; later additions/imports/shifts/inactivation/Dropbox/progression correction may decertify automatically. Submit was not activated.

## Reports and exports

Excel reports: Dropbox list, inactive history, imported history, active students, new students added by school, new students added by admin, progression details, and APAAR generation history. PDFs: age-wise, social-category-wise, and minority-wise counts. The downloaded history reports contained headers but no data rows; the List of All Students export contained 210 records.

## Supabase design

Existing project: `umv-db`, region `ap-south-1`.

Created restricted schema `udise`:

- `student_history`: audit/reference snapshots only; duplicates across reports/years are expected and it is not canonical.
- `dropbox_students`: separate Dropbox records; never merge into active enrolment without reconciliation.
- `profile_updates`: canonical EP/GP/FP proposed-change queue with approval/submission statuses.
- `report_imports`: source-file manifest; `imported_rows` must mean rows actually loaded.
- `dataset_catalog`: purpose and duplicate policy.
- `class_ix`, `class_x`, `class_xi`, `class_xii`: class-filtered views.

RLS is enabled and no public policies were added. Keep raw sensitive data restricted and do not expose this schema through the public API without an authorised access model. Planned Storage paths are `udise/2026-27/student-history/`, `udise/2026-27/dropbox/`, class folders, and `reports/`; the Storage bucket has not yet been created. The populated List of All Students export has been inserted into `student_history`: 210 rows, four classes, and 210 distinct PENs for 2026-27. The import register is marked imported. Dropbox and other history reports remain empty because their downloaded files contained headers only.

## Notebook assessment

The supplied Colab notebooks paste cookies/XSRF tokens/session encryption keys, call private endpoints, perform direct profile POSTs, automate Aadhaar/PEN lookup, and contact an external Apps Script telemetry endpoint. Do not use unchanged. Remove secrets and telemetry; use manual login plus visible UI or an officially authorised API; add validation, before/after review, approval, post-save verification, and audit logs.

## Future workflow

Manual login → read-only collection → local validation/deduplication → before/after workbook → explicit approval → authorised portal update → post-save verification → restricted Supabase ingestion → class-wise queries and audited exports.

## Open items

- Create and secure the private Supabase Storage bucket/folders.
- Reconcile the loaded 210-row workbook against later source snapshots when populated history exports become available.
- Define authorised Supabase operator policies.
- Confirm official UDISE+ API eligibility.
- Resolve 26 unmatched OFSS admissions using PEN, exact DOB, or another authorised identifier before assigning UDISE/APAAR values.
- Add repository `AGENTS.md`, `.ai/` state, ingestion code, migrations, tests, and CI after project requirements are agreed.
