# Academy OS — Complete Product Specification

Repo: `academy-os` · Next.js App Router, TypeScript, Prisma, Supabase Postgres,
Clerk, shadcn/ui, Zod, TanStack Table

This is the full product across every module. It is **sequenced**, not a
build-everything list. Each phase ends at a gate. Do not cross a gate on
assumption — cross it on something a real academy told you.

---

## Table of contents

- [Cross-cutting rules](#cross-cutting-rules)
- [Phase 0 — Done](#phase-0--done)
- [Phase 1 — Beta](#phase-1--beta)
- [GATE 1](#gate-1--real-usage)
- [Phase 2 — Daily operations](#phase-2--daily-operations)
- [Phase 3 — Money](#phase-3--money)
- [GATE 2](#gate-2--first-payment)
- [Phase 4 — The academy year](#phase-4--the-academy-year)
- [Phase 5 — Running the business](#phase-5--running-the-business)
- [Phase 6 — Only on demand](#phase-6--only-on-demand)
- [Schema evolution summary](#schema-evolution-summary)

---

## Cross-cutting rules

These apply to every module below without exception.

**Tenancy.** `organizationId` comes from `getOrgContext()` and nowhere else.
Never from a form field, URL param, or header. Every query filters on it. Never
`findUnique({ where: { id } })` — always `findFirst({ where: { id,
organizationId } })`.

**Phone is the universal key.** Indian academies identify everyone by mobile
number. Every list search matches phone. Every create form checks for an
existing person by phone before saving. Global search (the header magnifier)
should eventually take a phone number and return the parent, their children,
their enquiries, payments, and attendance in one view. This is the single most
distinctive thing the product can do — build toward it in every module.

**Roles.** `OWNER`, `ADMIN`, `RECEPTIONIST`, `TEACHER`, `ACCOUNTANT`. Enforce
in server actions, not just by hiding UI. A hidden button is not a permission.

**Server actions for writes, server components for reads.** Zod-validate every
action input. `revalidatePath` after writes. No client-side data fetching
unless a screen genuinely needs it.

**Entry speed beats completeness.** Every form that reception uses daily must
work in under 30 seconds. If a field isn't needed to complete the task in
front of them, it belongs on a later screen.

**Never build a module because the sidebar has a slot for it.**

---

## Phase 0 — Done

Auth (Clerk), tenant context, Enquiries create / list / detail, search, status
filters, follow-up dates, notes timeline, app shell with "Soon" markers.

---

## Phase 1 — Beta

Detailed separately in `BETA_BUILD_SPEC.md`. Summary:

**Courses** — create + list. Feeds the enquiry dropdown.
**Convert to student** — one transaction: reuse-or-create Parent by phone,
create Student, record registration `Payment`, close the Enquiry.
**Students** — read-only list + detail, searchable by student or parent phone.

Closes the loop: walk-in → follow-up → enrolled → student.

---

## GATE 1 — Real usage

**Do not build Phase 2 until:**

- The receptionist has logged at least 10 real enquiries herself
- You have watched her do it without helping
- She has told you what's missing

Everything below this line is a prediction. Her answers replace the
predictions. If she says fees matter more than attendance, skip Phase 2 and go
straight to Phase 3.

---

## Phase 2 — Daily operations

### 2.1 Teachers

**Why:** batches need an owner, and the academy already tracks who teaches what.

Model exists. Add nothing yet.

Screens: list (name, phone, active, batch count), create/edit sheet, detail
showing their batches.

Rules: never hard-delete a teacher — set `active = false`. Historical batches
and attendance must survive staff leaving.

### 2.2 Batches

**Why:** the real unit of academy operations. A course is a syllabus; a batch is
a specific group at a specific time with a specific teacher.

Schema additions to `Batch`:

```prisma
model Batch {
  // existing: name, courseId, teacherId, capacity, active
  startTime  String?   // "17:30" — keep as string, no timezone complexity
  endTime    String?
  daysOfWeek Int[]     // [1,3,5] = Mon/Wed/Fri
  room       String?
}
```

Screens: list grouped by course, create/edit, detail showing enrolled students
and remaining capacity.

**This is where `Enrollment` finally gets used.** On the student detail page,
add "Enroll in batch". One student, many batches — that's why the join model
exists.

Rules: warn at capacity, don't block. Academies overfill and will not accept
software telling them no.

### 2.3 Attendance

**Why:** replaces the paper register teachers currently mark.

New model:

```prisma
enum AttendanceStatus { PRESENT ABSENT LATE EXCUSED }

model Attendance {
  id             String           @id @default(cuid())
  organizationId String
  batchId        String
  studentId      String
  date           DateTime         @db.Date
  status         AttendanceStatus
  markedByUserId String
  note           String?
  createdAt      DateTime         @default(now())
  updatedAt      DateTime         @updatedAt

  organization Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  batch        Batch        @relation(fields: [organizationId, batchId],   references: [organizationId, id])
  student      Student      @relation(fields: [organizationId, studentId], references: [organizationId, id])

  @@unique([batchId, studentId, date])
  @@index([organizationId])
  @@index([batchId, date])
  @@index([studentId])
}
```

**Screen design matters more than the data model here.** Teachers mark
attendance on a phone, standing up, in under 60 seconds:

- Pick batch → today's date preselected
- Full roster, everyone defaulted to Present
- Tap only the absentees
- One Save

Anything slower and they keep using the register.

The `@@unique([batchId, studentId, date])` makes re-marking idempotent — use
`upsert`, so a teacher correcting a mistake doesn't create duplicates.

Views: by batch/date (marking), by student (history + percentage), and a
low-attendance flag on the student detail page.

**Do not build:** biometric, QR check-in, geofencing, parent absence
notifications. Not yet.

---

## Phase 3 — Money

**This is the module that gets you paid.** The founder buys two answers:
*who owes me money* and *how much came in this month*. Everything else is
convenience; this is the reason to sign up.

### 3.1 Fee structures

New model — without it, someone types the same amount 300 times:

```prisma
enum FeeInterval { ONE_TIME MONTHLY QUARTERLY ANNUAL }

model FeeStructure {
  id             String      @id @default(cuid())
  organizationId String
  courseId       String?
  name           String
  amount         Decimal     @db.Decimal(10, 2)
  interval       FeeInterval
  active         Boolean     @default(true)
  createdAt      DateTime    @default(now())
  updatedAt      DateTime    @updatedAt

  organization Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  course       Course?      @relation(fields: [organizationId, courseId], references: [organizationId, id])

  @@unique([organizationId, id])
  @@index([organizationId])
}
```

### 3.2 Payments

`Payment` already exists and already handles the hard case — it links to either
a `Student` or an `Enquiry`, so a registration fee taken before admission is
recordable.

Additions:

```prisma
model Payment {
  // existing: type, amount, status, paidAt, notes
  method          PaymentMethod?  // CASH UPI CARD BANK_TRANSFER OTHER
  reference       String?         // UPI txn ref, cheque number
  screenshotUrl   String?         // the WhatsApp screenshot, attached properly
  recordedByUserId String?
  dueDate         DateTime?
}
```

`screenshotUrl` matters more than it looks. The current workflow is: parent
sends a screenshot to WhatsApp, staff eyeball it, staff update Excel. Attaching
that image to the payment record is the whole point — it's the evidence trail
the academy currently loses.

### 3.3 Screens

**Record payment** — the daily action. Student (phone search), amount, type,
method, date, optional screenshot upload. Under 20 seconds.

**Outstanding dues** — the screen the owner opens. Every student with a pending
or overdue balance, sorted by days overdue, with total at the top. This one
screen justifies the subscription.

**Student fee history** — on the student detail page: all payments, running
balance.

**Collection summary** — this month vs last, by fee type. Not a chart-heavy
dashboard; one honest table.

### 3.4 Deliberately not built

Payment gateway integration, auto-generated invoices, automatic reminders,
recurring billing runs. Every one is a real project. Recording what already
happened is enough to replace the Excel sheet, and replacing the Excel sheet is
the sale.

---

## GATE 2 — First payment

**Do not build Phase 4 until one academy has paid you real money.**

Not a free pilot. Not your own academy. An academy that hands over money is the
only evidence that any of this is a business. Everything past this gate is
worth building only if that has happened.

---

## Phase 4 — The academy year

### 4.1 Documents

**Why:** exam documents currently scatter across Drive folders and hard drives.

```prisma
enum DocumentType { PHOTO ID_PROOF BIRTH_CERTIFICATE EXAM_FORM OTHER }

model Document {
  id             String       @id @default(cuid())
  organizationId String
  studentId      String
  type           DocumentType
  fileUrl        String
  fileName       String
  uploadedByUserId String?
  verifiedAt     DateTime?
  createdAt      DateTime     @default(now())

  organization Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  student      Student      @relation(fields: [organizationId, studentId], references: [organizationId, id])

  @@index([organizationId])
  @@index([studentId])
}
```

Storage: Supabase Storage, private bucket, signed URLs. Never public.

**Tokenized parent upload.** Generate a one-time link per student, reception
sends it over WhatsApp, the parent taps and uploads from their phone. It
attaches to the right student automatically. **No login, no account** — the
signed token carries the `organizationId`. This is the feature that actually
kills the Drive-folder mess.

**⚠️ Legal — resolve before building.** India's DPDP Rules were notified in
November 2025, with full compliance due May 2027, and most of these students
are minors, which requires verifiable parental consent. Storing Aadhaar numbers
carries restrictions beyond ordinary personal data. **Talk to a lawyer before
building Aadhaar collection.** Strongly consider storing a verification status
rather than the number itself. Retrofitting this after ten academies have
uploaded data is expensive and possibly unlawful.

Build consent capture into the upload flow from day one.

### 4.2 Exams

```prisma
model Exam {
  id             String   @id @default(cuid())
  organizationId String
  courseId       String?
  name           String
  examDate       DateTime?
  registrationDeadline DateTime?
  fee            Decimal? @db.Decimal(10, 2)
  // + ExamRegistration join: examId, studentId, status, documentsComplete
}
```

Replaces the Google Form. The valuable screen is **eligibility and readiness**:
which students are eligible, who has registered, whose documents are still
missing. That last column is the current pain.

### 4.3 Events

```prisma
model Event {
  id             String   @id @default(cuid())
  organizationId String
  name           String
  eventDate      DateTime?
  venue          String?
  participationFee Decimal? @db.Decimal(10, 2)
  // + EventParticipant join: eventId, studentId, confirmed, feeStatus
}
```

Annual day, competitions. Participant list, fee status per participant,
costume requirements as free text initially.

---

## Phase 5 — Running the business

### 5.1 Settings

Academy profile (name, address, phone, logo), course and fee defaults, and
**Users & Roles** — the owner adds a receptionist, which creates a `User` row
plus a Clerk invitation. Same manual pattern you use, moved in-app.

### 5.2 Dashboard

Build this **last**, not first. Only when there is real data:

- Enquiries this month, conversion rate
- Fees collected vs outstanding
- Today's attendance
- Follow-ups due

Four honest numbers. Not twelve widgets.

### 5.3 Reports

Export to CSV/Excel: student lists, fee collection, attendance summaries.
Sounds unglamorous; it's what makes an academy trust that its data isn't
trapped. It also removes the "what if I want to leave" objection in a sales
conversation.

### 5.4 Global search

The header search bar, finally doing what the whole design has been pointing
at. Type a phone number → parent, children, enquiries, payments, attendance.
Type a name → same. One box for the entire academy.

---

## Phase 6 — Only on demand

Do not build any of these speculatively. Build one only when multiple academies
have asked, unprompted.

- **Notes / learning materials** — attach files to a batch; replaces WhatsApp
  group uploads
- **Costumes & jewellery rental** — inventory, issue/return, deposits. Real for
  dance academies, irrelevant to music schools. Genuinely differentiating if
  academies keep asking.
- **Public enquiry form** — a per-academy URL for their Instagram bio. Replaces
  their Google Form. Cheap to build; do it when they ask.
- **Parent portal** — phone + OTP, **never** email/password. Indian parents will
  not maintain a password for a dance academy. Read-only: fees, attendance,
  events.
- **Teacher mobile app** — only if the responsive web attendance screen proves
  insufficient in practice.
- **WhatsApp Business API** — fee reminders, absence alerts. Expensive, needs
  approval, high ongoing cost. Only with clear demand.
- **Payment gateway** — Razorpay/Cashfree. Requires GST and a business account.
- **Multiple branches** — a `Branch` model under `Organization`. Only when an
  academy with real branches signs up.
- **Inventory** — beyond costumes. Probably a separate product.

---

## Schema evolution summary

| Phase | New models | Modified |
|---|---|---|
| 1 | — | `Enquiry` (+courseId, +convertedStudentId) |
| 2 | `Attendance` | `Batch` (+timing, +days, +room) |
| 3 | `FeeStructure` | `Payment` (+method, +reference, +screenshotUrl, +dueDate) |
| 4 | `Document`, `Exam`, `ExamRegistration`, `Event`, `EventParticipant` | — |
| 5 | — | `Organization` (+profile fields) |

**Every new model follows the same pattern:** `organizationId` with
`onDelete: Cascade`, composite FKs `[organizationId, id]` to any tenant-owned
parent, `@@unique([organizationId, id])` if anything references it, and
`@@index([organizationId])`.

Use `prisma migrate dev` from here on. Never `db push` again — there is real
data now.

---

## How to use this document

Build Phase 1. Show it to the academy. Then **come back and rewrite Phase 2
based on what they said.**

The phases below Gate 1 are informed guesses. They're here so you can see the
shape of the whole product and make architecture decisions that don't box you
in — not so you can build them all before anyone has used the first one.

The most likely way this product fails is not a missing module. It is building
eleven modules that nobody asked for while the one that mattered stayed
shallow.
