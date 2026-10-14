# Student Discontinue & Re-admission Management System

A dedicated administrative subsystem for managing student departure (discontinuation) and subsequent re-admission across multiple academic years without data loss, preserving historical fee dues up to the departure date, and recording multi-year gap trajectories with auditable notes in the existing Vanilla JS + HTML5 + Tailwind CSS + Cloud Firestore architecture.

---

## User Review & Critical Decisions

> [!IMPORTANT]
> The following product requirements and architectural decisions were confirmed based on the user's explicit responses and clarifications:

- **Confirmed Decision 1 (Navigation Placement)**: The Discontinue & Re-admission portal will be placed in a dedicated **"Continuity & Transfers"** section within the admin navigation drawer (rather than buried inside sub-menus).
- **Confirmed Decision 2 (Fee Ledger Preservation & Profile Collection)**: When a student is discontinued, all unpaid fee dues prior to discontinuation are preserved in the ledger up to the departure date. In the Fee Collection screen and student profiles, discontinued students appear marked with a distinct "Discontinued" badge, allowing administrators to collect preserved prior dues. Crucially, **no fees are assessed or generated for intervening gap years**. Fee generation resumes strictly from the return date / target academic year.
- **Confirmed Decision 3 (Re-admission Target Assignment)**: During re-admission, administrators select the target active academic year and assign the student to their return class via a manual class/division dropdown.
- **Confirmed Decision 4 (Class Register & Table Visibility)**: Discontinued students are excluded from active class registers, regular attendance lists, and active student tables by default, and are centrally managed within `#student-continuity`.
- **Confirmed Decision 5 (Optional TC Generation)**: During the discontinuation workflow, an optional **"Generate Transfer Certificate (TC) Entry"** checkbox allows administrators to create a corresponding TC record/draft alongside the discontinuation log.
- **Confirmed Decision 6 (Zero Architecture / Stack Mutation)**: The existing Vanilla JS, HTML5, Tailwind CSS, and Cloud Firestore stack is strictly maintained without introducing TypeScript compilers, React, or unnecessary external dependencies.

---

## 1. Overview & Core Concept

### What It Does
Provides madrasa administrators with a dedicated workspace to manage student departure and return:
1. **Class-Filtered Discontinuation**: Admins select a class, select a student, enter the departure date, reason (e.g., family relocation, transfer, medical, madrasa change), and optionally generate a TC record. The student transitions to `Discontinued` status.
2. **Dedicated Discontinued Student Archive**: A searchable registry displaying all discontinued students with their admission number, name, original class, exit date, exit reason, preserved dues, and inactive duration (e.g., "1 Year 4 Months / 2 Academic Years").
3. **Multi-Year Re-admission Wizard**: When a student returns after 1, 2, or 3+ academic years, clicking **"Re-admit Student"** opens a guided modal. Admins select the target Academic Year, assign the new Class and Division, enter the return date and notes, and preview the historical gap and preserved prior fee balance.
4. **Historical Trajectory Integrity**: Discontinuations and re-admissions are recorded in the student's `continuityLog` array, `academicYears` map, and Firestore audit logs, preventing any loss of past academic records.

### Target Audience
Madrasa Principals, Headmasters (Sadrs), Office Administrators, and Super Admins managing student cohorts across Kerala madrasa academic calendars (May/June to March/April).

### Key Value
Eliminates confusing manual deactivations or deletion of student records, maintains unbroken academic audit trails across years, and ensures returning students resume their studies with accurate fees and class levels.

---

## 2. User Experience & Visual Design

### Key User Flows

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DISCONTINUE & RE-ADMISSION WORKFLOW                   │
└─────────────────────────────────────────────────────────────────────────────┘

  [1. STUDENT DEPARTURE]
  Admin Navigation ──> #student-continuity
       │
       ▼
  Top Action Bar:
  ├── Select Class (e.g. "Class 4 B")
  ├── Select Student from dropdown or live search
  ├── Enter Exit Date & Reason (Notes)
  ├── Checkbox: [ ] Generate Transfer Certificate (TC) draft
  └── Click "Discontinue Student"
       │
       ▼
  [Confirmation Modal] ──> Commit to Firestore:
       • status: 'Discontinued'
       • continuityLog appended with exit details & prior balance
       • academicYears[currentYear].status = 'Discontinued'
       • Optional TC record created if selected
       • Audit log generated

  ─────────────────────────────────────────────────────────────────────────────

  [2. DISCONTINUED DIRECTORY]
  Real-time List Table:
  ├── Photo, Adm No, Name, Gender
  ├── Class at Departure & Academic Year
  ├── Exit Date & Recorded Reason (clean unboxed text)
  ├── Inactive Gap Duration (e.g., "2 Years 1 Month")
  ├── Outstanding Fees prior to exit (₹ Preserved)
  └── [Re-admit Student] Button & [View Trajectory] Button

  ─────────────────────────────────────────────────────────────────────────────

  [3. STUDENT RETURNS (RE-ADMISSION)]
  Click "Re-admit Student" ──> Guided Modal:
  ├── Current Info: Name, Adm No, Last Studied Class
  ├── Target Academic Year: Dropdown of active/future years
  ├── Target New Class: Dropdown with manual class/division selection
  ├── Return Date & Re-admission Notes
  ├── Gap Summary: "2 missed academic years: 2024-2025, 2025-2026"
  ├── Fee Status Preview: "Preserved prior dues: ₹400. No gap fees charged."
  └── Commit Re-admission ──> Student status becomes 'Active', joins new class!
```

### Visual Identity & Theme
- **Layout & Structure**: Adheres to the Universal Frontend Design Constitution (SaaS Dashboard domain). Full 1440px desktop width, single-elevation cards with clean hairline borders (`border border-slate-200`), and generous whitespace.
- **Color Palette & Discipline (60-30-10)**:
  - `60% Neutral Canvas`: Crisp slate background (`bg-slate-50 / bg-white`).
  - `30% Structural Surfaces`: Clean white cards with `border-slate-200` and subtle slate headers.
  - `10% Accent Palette`:
    - Discontinue action: Crimson/rose accents (`bg-rose-600 hover:bg-rose-700`, text badges `bg-rose-50 text-rose-700 border-rose-200`).
    - Re-admission action: Emerald/teal accents (`bg-emerald-600 hover:bg-emerald-700`, text badges `bg-emerald-50 text-emerald-700 border-emerald-200`).
    - Gap metadata: Unboxed muted typography with bullet separators (`·`).
- **Typography & Tabular Discipline**:
  - Headings: Clean sans font.
  - Metrics, Adm Numbers, Dates, and Fees: `font-mono tabular-nums` for alignment without jitter.
  - All application UI text strictly in English.
- **Zero-Pill Discipline**:
  - Tags and metadata (dates, missed years) rendered as unboxed quiet text separated by `·`.
  - Filter controls rendered as functional segmented button tabs.

---

## 3. Key Product Decisions & Trade-Offs

### Decision 1: Single Student Document with `continuityLog` and `academicYears` Map
- **Chosen Approach**: Retain the student's unique ID (`admissionNo` / doc ID). Add an array `continuityLog` storing each status transition, and update `academicYears[yearLabel]` with the exact class and status for that specific year.
- **Why**: Keeps student fees, receipts, attendance, and exam marks connected to one master identity. When generating multi-year reports or certificates, the full history is readily available without creating duplicated or orphaned student records.
- **Alternative Considered**: Creating a new student document upon re-admission was rejected because it fragments student ID, breaks fee continuity, and loses past attendance history.

### Decision 2: Fee Preservation & Gap Immunity
- **Chosen Approach**: When a student is discontinued:
  1. All payments made and pending fee items generated up to their discontinuation date remain intact.
  2. The fee engine filters out `Discontinued` students when generating monthly or annual fee dues.
  3. Payments against prior balances can still be collected directly from the collection screen / student profile.
  4. Upon re-admission, active fee generation starts from the re-admission date / target academic year.
- **Why**: Strictly adheres to the rule that dues up to departure date are preserved, gap years accrue zero fees, and new fees begin upon return.

### Decision 3: Dedicated "Continuity & Transfers" Drawer Section
- **Chosen Approach**: Add a dedicated navigation heading `Continuity & Transfers` in `admin.html` with direct access to `#student-continuity`.
- **Why**: Matches administrative workflow where student lifecycle events (departures, returns, transfer certificates) are tracked distinctly from regular student registration.

---

## 4. Technical Architecture & Data Strategy

### System Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ADMIN PORTAL COMPONENT MAP                           │
└─────────────────────────────────────────────────────────────────────────────┘

  admin.html
  ├── Sidebar Navigation:
  │   └── <div class="drawer-heading">Continuity & Transfers</div>
  │       └── <a href="#student-continuity">
  │
  ├── Section: #student-continuity-page
  │   ├── Header & Contextual Academic Year Indicator
  │   │
  │   ├── Panel A: "Discontinue Student"
  │   │   ├── Class Selector (#continuity-class-select)
  │   │   ├── Student Selector (#continuity-student-select)
  │   │   ├── Departure Date (#continuity-exit-date)
  │   │   ├── Departure Reason / Notes (#continuity-exit-reason)
  │   │   ├── Checkbox: #continuity-generate-tc
  │   │   └── Action: #execute-discontinue-btn
  │   │
  │   └── Panel B: "Discontinued Students Archive"
  │       ├── Metrics Counter (Total Discontinued, Current Year Departures)
  │       ├── Search Input & Class Filter (#continuity-search-input)
  │       ├── Responsive Table (#discontinued-students-table-body)
  │       └── Action Buttons:
  │           ├── Re-admit (#btn-open-readmit)
  │           └── View History & Notes (#btn-view-continuity-history)
  │
  └── Modals:
      ├── #readmission-popup: Target Year, Target Class, Date, Notes, Gap Review
      └── #continuity-history-popup: Multi-year visual timeline of departures/returns

  ─────────────────────────────────────────────────────────────────────────────

  scripts/pages/admin-page.js
  ├── State: rawStudents, students, academicYears, feeItems, allPayments
  ├── Router: '#student-continuity' page activation in PAGE_PERMISSIONS
  ├── Handlers:
  │   ├── renderContinuityPage()
  │   ├── populateContinuityStudentDropdown()
  │   ├── handleDiscontinueStudent()
  │   ├── openReadmitModal()
  │   ├── handleExecuteReadmission()
  │   └── openContinuityHistoryModal()
  └── Firestore Mutations:
      └── updateDoc(studentRef, { status, continuityLog, academicYears, ... })

  scripts/pages/collection-page.js
  └── Ensure discontinued students can be searched with 'Discontinued' badge
      for prior dues settlement, while excluding them from new fee generation.
```

### Data Schema Extensions (`firebase-blueprint.json` & Firestore)

1. **Student Entity Updates**:
   - `status`: String enum includes `'Active' | 'Inactive' | 'Graduated' | 'Discontinued'`.
   - `discontinuationDetails`:
     ```typescript
     {
       date: string;          // ISO date
       academicYear: string;  // e.g. "2024-2025"
       lastClass: string;     // e.g. "Class 4"
       reason: string;        // Notes on why student left
       recordedBy: string;    // Admin user email
       tcGenerated: boolean;  // TC flag
     }
     ```
   - `readmissionDetails`:
     ```typescript
     {
       date: string;          // ISO date
       targetYear: string;    // e.g. "2026-2027"
       targetClass: string;   // e.g. "Class 6"
       reason: string;        // Notes on return
       gapYears: string[];    // e.g. ["2024-2025", "2025-2026"]
       recordedBy: string;
     }
     ```
   - `continuityLog`: Array of historical events:
     ```typescript
     Array<{
       id: string;
       type: 'discontinue' | 'readmit';
       timestamp: string;
       academicYear: string;
       class: string;
       reason: string;
       adminEmail: string;
       priorBalance?: number;
     }>
     ```

2. **Fee Generation & Collection Scoping**:
   - `collection-page.js` & `admin-page.js` fee calculations treat `Discontinued` students as non-accruing for any fee cycles occurring during their gap periods.
   - Any payments made against prior balances will continue to be recorded against the student's unique ID.

---

## 5. Execution & Verification Plan

1. **Update `firebase-blueprint.json`**:
   - Add `Discontinued` status and specify `continuityLog`, `discontinuationDetails`, and `readmissionDetails` properties on the Student entity.
2. **Add Navigation & UI Markup in `admin.html`**:
   - Add `Continuity & Transfers` section in sidebar with link to `#student-continuity`.
   - Insert `#student-continuity-page` markup with Discontinue Action Panel, Discontinued Directory table, `#readmission-popup`, and `#continuity-history-popup`.
3. **Implement Logic in `scripts/pages/admin-page.js`**:
   - Register `student-continuity` in `PAGE_PERMISSIONS` with `students.manage`.
   - Implement dropdown binding, table rendering, gap calculations, `handleDiscontinueStudent`, and `handleExecuteReadmission`.
   - Add optional TC generation hook if checkbox is checked.
4. **Insulate Fee Ledger & Update Collection Visibility**:
   - Ensure `collection-page.js` allows searching/viewing discontinued students to settle prior dues with badge indicator, while excluding them from active periodic fee demands.
5. **Compile & Verification**:
   - Run `compile_applet` to ensure syntax and build pass cleanly.
