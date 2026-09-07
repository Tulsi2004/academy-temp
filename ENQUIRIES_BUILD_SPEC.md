# Task: Academy OS — Tenant context + Enquiries module

Repo: `academy-os` · Stack: Next.js App Router, TypeScript, Prisma, Supabase
Postgres, Clerk, shadcn/ui, Zod

Work through this in order. Stop and verify at each checkpoint. Do not skip
ahead — Part 1 is the foundation everything else sits on.

---

## Current state

- Clerk auth works; sign-in reaches the dashboard
- App shell built: sidebar, header, dashboard and enquiries pages
- Unbuilt modules already marked "Coming soon" in the sidebar
- Prisma schema is done: composite tenant FKs, `Enrollment` join model,
  `Payment` linkable to either `Student` or `Enquiry`, cascade rules
- **Nothing reads or writes the database yet.** The academy switcher shows a
  hardcoded "Your Academy" and the sidebar footer shows a hardcoded
  "Admin User / admin@academy.os"

## Goal of this task

A receptionist signs in, sees her own academy's name, logs a walk-in enquiry in
under 30 seconds, and finds it again by phone number.

---

# PART 1 — Tenant context (do this first)

## 1.1 Rename the Clerk id field

The schema currently has `supabaseUserId` on `User` from an earlier plan.
Rename it:

```prisma
model User {
  // ...
  clerkUserId String? @unique
  active      Boolean @default(true)
}
```

Nullable on purpose: staff rows are created by an admin **before** that person
has ever signed in. The field is filled on their first login.

```bash
npx prisma migrate dev --name rename_clerk_user_id
```

## 1.2 Seed the first academy

Create `prisma/seed.ts`. This exists because onboarding is manual — there is no
self-serve signup, and there should not be one yet.

Create:
- One `Organization` — use the real academy name and a slug
- One `User` — **your own real email**, role `OWNER`, `clerkUserId` left null

Wire it up in `package.json`:

```json
"prisma": { "seed": "tsx prisma/seed.ts" }
```

```bash
npx prisma db seed
```

## 1.3 `lib/auth/org-context.ts`

This is the single most important file in the codebase. Every database query in
the app resolves its tenant through it.

```ts
import { auth, currentUser } from "@clerk/nextjs/server";
import { prisma } from "@/lib/prisma";
import { cache } from "react";

export type OrgContext = {
  userId: string;          // our User.id, NOT Clerk's
  organizationId: string;
  role: UserRole;
  name: string;
};

export const getOrgContext = cache(async (): Promise<OrgContext> => {
  const { userId: clerkUserId } = await auth();
  if (!clerkUserId) throw new Error("UNAUTHENTICATED");

  // 1. Already linked — the normal path
  let user = await prisma.user.findFirst({
    where: { clerkUserId, active: true },
  });

  // 2. First sign-in — link by email, once
  if (!user) {
    const clerkUser = await currentUser();
    const email = clerkUser?.primaryEmailAddress?.emailAddress;
    if (!email) throw new Error("NO_EMAIL");

    const pending = await prisma.user.findFirst({
      where: { email, clerkUserId: null, active: true },
    });
    if (!pending) throw new Error("NO_ACCESS");

    user = await prisma.user.update({
      where: { id: pending.id },
      data: { clerkUserId },
    });
  }

  return {
    userId: user.id,
    organizationId: user.organizationId,
    role: user.role,
    name: user.name,
  };
});
```

**Why lazy email linking instead of Clerk webhooks:** webhooks fail, retry and
arrive out of order, and you end up writing reconciliation code for a problem
you do not have. Linking on first sign-in needs no endpoint and no sync. A
Clerk account with no matching `User` row simply gets `NO_ACCESS`.

`cache()` is React's request-level memoisation — many server components can
call `getOrgContext()` in one request and only one query runs.

## 1.4 Handle `NO_ACCESS`

Add `app/no-access/page.tsx`: a plain page saying the account isn't linked to an
academy, with a contact line and a sign-out button. Catch `NO_ACCESS` in the
dashboard layout and redirect there.

Also, in the **Clerk dashboard**, set sign-ups to restricted / invitation-only.
Without this, anyone who finds the URL creates an account. They would get no
access, but you accumulate orphan Clerk accounts.

## 1.5 Replace the hardcoded shell values

- Academy switcher: real `Organization.name`
- Sidebar footer: real user name and email from context
- Do this in a server component and pass values down as props

### ✅ Checkpoint 1

Sign in and see your real academy name and your own email in the sidebar.
Check the database — your `User` row now has `clerkUserId` filled in.

**Tenancy works end to end. Do not continue until this passes.**

---

# PART 2 — Enquiries: create

## 2.1 Two small schema additions

```prisma
model Enquiry {
  // ...
  courseId           String?
  convertedStudentId String? @unique
}
```

- `courseId` — a parent often says "something for my 7-year-old", so keep the
  free-text `interestedIn`, but capture a real course when they name one
- `convertedStudentId` — without it you cannot answer "how many enquiries became
  admissions this month", which is the number the academy owner actually cares
  about

## 2.2 `lib/validations/enquiry.ts`

```ts
export const createEnquirySchema = z.object({
  studentName: z.string().min(1).max(100),
  phone: z.string().regex(/^[6-9]\d{9}$/, "Enter a 10-digit mobile number"),
  interestedIn: z.string().max(100).optional(),
  courseId: z.string().optional(),
  notes: z.string().max(1000).optional(),
});
```

Indian mobile numbers start 6–9 and are 10 digits. Normalise before saving:
strip spaces, `+91`, and leading `0`. The phone number is the key everything
else is found by, so inconsistent formatting breaks lookup later.

**`organizationId` is NOT in this schema and must never be.** It comes from
`getOrgContext()`. If it ever arrives from a form field, URL or header, that is
a tenant-isolation hole.

## 2.3 `app/(dashboard)/enquiries/actions.ts`

```ts
"use server";

export async function createEnquiry(input: unknown) {
  const { organizationId } = await getOrgContext();
  const data = createEnquirySchema.parse(input);

  const enquiry = await prisma.enquiry.create({
    data: { ...data, organizationId, status: "NEW" },
  });

  revalidatePath("/enquiries");
  return { ok: true, id: enquiry.id };
}
```

## 2.4 The quick-capture sheet

**This is the most important screen in the product.** If it takes longer than
the paper register, the receptionist stops using it and the product is dead.

- A shadcn `Sheet` or `Dialog` opened by the existing "New Enquiry" button —
  **not** a separate page. She never navigates away from the list.
- Autofocus the **phone** field on open
- Exactly four fields: phone, student name, interested in, note
- Everything else — DOB, address, parent details, experience — is collected at
  admission, not here
- Enter submits; the sheet closes and the row appears at the top
- Keep it open with cleared fields on "Save and add another"

**Target: under 30 seconds, and it should work one-handed on a phone.**

## 2.5 Phone duplicate detection

Debounce the phone field (400ms). Once 10 digits are entered, query existing
`Enquiry` and `Parent` rows in this organization.

If matched, show inline, above the form:

> **Kavya Sharma** enquired 12 days ago — Follow-up
> [Open existing]

This is the feature that makes a receptionist understand why the app beats the
register. It is impossible on paper, it is cheap to build, and it is the
beginning of phone-number-as-universal-key across the whole product.

### ✅ Checkpoint 2

Add three enquiries. They appear in the database with the correct
`organizationId`. Re-entering a saved number surfaces the duplicate warning.

---

# PART 3 — Enquiries: list and detail

## 3.1 List

Server component reading through `getOrgContext()`. Never `findMany` without
`organizationId` in the where clause.

- Columns: name, phone, interested in, status, follow-up date, created
- Search box → matches name **or** phone
- Status filters — the tabs already exist in the UI, wire them up
- Sort: follow-ups due first, then newest
- Keep search, filter and page in **URL search params**, so state survives
  refresh and back

Rows overdue for follow-up need to be visually obvious. That list is the
product's daily value.

## 3.2 Detail

Route: `/enquiries/[id]`. Always scope the fetch:

```ts
where: { id, organizationId }
```

Never `findUnique({ where: { id } })` — that is a cross-tenant read waiting to
happen.

Shows: all fields, status changer, follow-up date picker, notes timeline, and a
**Convert to student** button.

## 3.3 Follow-ups view

The "Follow-ups" toggle already exists in the header. Wire it to
`followUpDate <= today AND status NOT IN (ADMITTED, LOST)`.

This is what the receptionist opens every morning.

### ✅ Checkpoint 3

Search by partial phone number finds the enquiry. Status filters work. Setting
a follow-up date for today makes it appear in Follow-ups.

---

# PART 4 — Convert to student

One server action inside a transaction, because a partial conversion leaves
orphaned rows:

```ts
await prisma.$transaction(async (tx) => {
  // 1. create or reuse Parent (match on phone within this org)
  // 2. create Student
  // 3. create Enrollment if a batch was picked
  // 4. update Enquiry: status ADMITTED, convertedStudentId
});
```

This is the screen where the fuller details get collected — DOB, parent name,
address, experience. Justified here because the person is actually enrolling.

Registration fee: create a `Payment` linked to the new `Student`. The
`enquiryId` route on `Payment` is for fees taken **before** conversion; do not
use both.

---

# Rules

- **Never** accept `organizationId` from the client. Session only.
- **Never** query without an `organizationId` filter.
- Validate every server action input with Zod.
- Do not build Students, Parents, Courses, Batches, Teachers, Attendance, Fees,
  Events, Exams or Documents. They stay "Coming soon".
- Do not build the dashboard widgets. Change the post-login landing page to
  `/enquiries` — that is where the work happens.
- Do not add a public enquiry form, parent portal, WhatsApp integration, or
  bulk import yet.
- No optimistic UI or realtime. Server actions with `revalidatePath` are enough.

---

# Definition of done

A receptionist signs in, logs a walk-in in under 30 seconds without training,
finds them again by phone, marks a follow-up for tomorrow, and sees them in
Follow-ups when she opens the app the next morning.

Everything else is secondary.

---

# Before building Part 2

The `Enquiry` fields above are still an assumption. Photograph the academy's
paper application form and the visitor register, and compare. If the real form
asks something these fields miss, change the schema **before** building the UI,
not after.
