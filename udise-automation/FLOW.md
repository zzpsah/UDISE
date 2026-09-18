# UDISE+ Student Profile Automation — GP Blood Group Fill Flow

**School:** UMV Tetahali (PEN 10160203806)  
**Academic Year:** 2026-27  
**Target Class:** IX (then X, XI, XII)  
**User:** ANAMIKA KUMARI (School User)  
**Scope:** General Profile — Blood Group only (fill "Under Investigation" if empty, then Save)  
**Tool:** `computer_use` (Chrome SOM mode) + manual mouse-follow  

---

## Pattern (one student)

```
OPEN STUDENT
    │
    ▼
Navigate to student's General Profile
    │
    ▼
Check 4.20 Blood Group dropdown
    │
    ├── Already "Under Investigation" ?
    │       │
    │       ▼
    │   Click Save
    │       │
    │       ▼
    │   Wait for "General data updated successfully" modal
    │       │
    │       ▼
    │   Click Close on modal
    │       │
    │       ▼
    │   Done — move to next student
    │
    └── Empty / other value ?
            │
            ▼
        Click on Blood Group dropdown (#4.20)
            │
            ▼
        Select "Under Investigation - Result will be updated soon"
            │
            ▼
        Click Save
            │
            ▼
        Wait for modal
            │
            ▼
        Click Close
            │
            ▼
        Done — move to next student
```

## Element IDs (Chrome SOM, Chrome zoomed 67%)

These shift per student/render — recompute each time from a fresh SOM capture.

| Element | Approx role | Approx label | Note |
|---|---|---|---|
| Blood Group dropdown | ComboBox | "Under Investigation …" (or blank) | Rule label 4.20, near y=718-843 in last render |
| Save button | Button | "Save" | Below form, near y=809-934 |
| Confirmation modal | Window | "General data updated successfully." | Appears after Save |
| Modal Close button | Button | "Close" | Inside the modal |

## Confirmation modal behaviour

- Clicking **Save** always triggers a modal: **"General data updated successfully."**
- Modal has a green checkmark + blue **Close** button.
- Must click **Close** before the next action — the modal blocks the page otherwise.
- If the modal does NOT appear, re-check Save click landed (re-capture and verify).

## Per-student checklist

1. SOM capture (current page)
2. Identify Blood Group dropdown element
3. Read current value
4. If empty → open dropdown → select "Under Investigation - Result will be updated soon"
5. Click Save
6. SOM/vision capture — confirm "General data updated successfully" modal appeared
7. Click modal Close button
8. Log: student_key, PEN, name, class, action=BloodGroup|UnderInvestigation, result=Saved
9. Next student

## Resume / crash recovery

- After each student, record last_completed_step + timestamp in `checkpoints/`.
- On resume: re-capture page, identify current student from the URL/student name, continue from the last incomplete step.

## What this doc does NOT cover (yet)

- Class-wise student list loading
- Navigating from student list → individual student profile
- EP / FP profiles
- Other GP mandatory fields
- Batch APPLY/SUBMIT beyond this single-field pattern

These extend the same skeleton — capture → act → confirm → log → next.
