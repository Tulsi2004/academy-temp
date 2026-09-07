# Academy OS — testing checklist (Parts 1–4)

All four parts of `ENQUIRIES_BUILD_SPEC.md` are built. Run `npm run dev` and work
through this in order. Each section maps to a checkpoint in the spec.

Useful at any point:

- `npm run check:tenancy` — read-only. Prints every organization with its
  enquiries, and a count of any enquiry belonging to no organization (must be 0).
- `npx prisma studio` — click around the database directly.

---

## Part 1 — Tenant context

1. Sign in. You should land on **/enquiries**, not a dashboard.
2. Sidebar shows **The Tulsi Academy** at the top and **Tulsi /
   tulsimani04@gmail.com** at the bottom. Neither is a button any more — account
   actions are the Clerk avatar in the top bar.
3. Sidebar: everything except Enquiries is greyed out with a **Soon** pill and is
   not clickable. Dashboard is greyed too.
4. Visit `/` directly — it should redirect to `/enquiries`.
5. Visit `/setup` — should 404. That self-serve onboarding flow is gone.
6. **No-access path.** Sign out, sign up with a *different* email that has no
   `User` row. You should land on **/no-access** with a sign-out button, not on a
   crash or an empty dashboard. Then sign back in as yourself.
   - If you have not yet set Clerk sign-ups to invitation-only, do that after
     this test, otherwise strangers can create orphan accounts.
7. `npm run check:tenancy` — your `User` row shows a `clerkUserId`.

---

## Part 2 — Capture

8. Click **New Enquiry**. A sheet slides in: from the right on desktop, up from
   the bottom on a phone. The **Phone** field is already focused.
9. Type a name and phone, press **Enter** without touching the mouse. The sheet
   closes and the row appears at the top of the list.
10. **Save and add another**: fill it in, click that button. The sheet stays
    open, fields clear, focus returns to Phone, and a green
    *"Saved <name>. Add the next one."* line appears. Add a third this way.
11. **Phone normalisation.** Save one enquiry with the phone typed as
    `+91 98765 43210`. Check the list — it must display as `9876543210`.
12. **Duplicate detection.** Open the sheet and type a number you already saved.
    After ~400ms an amber box appears above the form:
    *"<name> enquired today — New"* with an **Open existing** link. Click it —
    the sheet closes and the enquiry opens.
13. Try the same number in three formats — `9876543210`, `+91 98765 43210`,
    `098765 43210`. All three must raise the same duplicate warning.
14. **Validation.** Try saving with a landline (`022 2222 3333`), a 5-digit
    number, and a blank name. Each should show an inline field error and save
    nothing.
15. `npm run check:tenancy` — every enquiry sits under **The Tulsi Academy** and
    the orphan count is 0.

---

## Part 3 — List, search, follow-ups

16. **Search.** Type a partial phone (e.g. the last 4 digits) in the search box.
    The list filters and the URL gains `?q=…`. Refresh the page — the filter and
    the box contents survive.
17. Search a partial name, in lower case. It should match regardless of case.
    Also search a name that **contains a digit** (e.g. `Test2`) — it must match
    that name only, not every enquiry whose phone happens to contain a 2.
    A term is treated as a phone search only when the whole term is a number.
18. **Filters keep search.** With a search active, click a status chip. The URL
    should carry both `?q=` and `?status=`, and the search must not be lost.
19. Press the browser **back** button — the previous filter state returns.
20. **Whole row is clickable.** Click anywhere on a row, not just the name.
21. **Overdue is obvious.** On one enquiry set a follow-up date of *yesterday*
    and save. Back on the list it should have a red left bar and a tinted row,
    and sort above the others. The header should read "1 due for follow-up".
22. **Follow-ups tab.** Set one enquiry to *today* and one to *next week*.
    Open **Follow-ups** — only yesterday's and today's appear, grouped Overdue
    and Today. Next week's must not be there.
23. Set an enquiry to *yesterday* and mark it **Lost**. It should disappear from
    Follow-ups and lose its red styling in the list.
24. If you have more than 25 enquiries, check the Previous/Next pager and that
    `?page=` survives a refresh.

---

## Part 4 — Notes timeline and conversion

25. **Notes are append-only.** Open an enquiry, write a note in *Add a note*,
    save. It appears in the **Notes** panel with your name and a timestamp.
    Write a second note and save — the first one must still be there, with the
    newest on top. Nothing is overwritten.
26. The note you typed in the capture sheet when first creating an enquiry
    should be the oldest entry in that enquiry's timeline.
27. **Convert.** On an unconverted enquiry click **Convert to student**. The form
    pre-fills the student's first/last name from the enquiry, and the parent
    phone from the enquiry phone. Fill in DOB, address, parent name, and a
    registration fee, then convert.
28. You land back on the enquiry. It now shows **Converted**, a green
    *"Admitted as <name>"* banner, and the **Convert to student** button is gone.
    A timeline note records the conversion.
29. Visit `/enquiries/<that id>/convert` directly — it should bounce you back to
    the enquiry rather than letting you convert twice.
30. **Parent reuse.** Convert a second enquiry using the *same parent phone* as
    the first. In `npx prisma studio`, the `Parent` table must have **one** row
    for that number, with two students attached — not two parent rows.
31. In Prisma Studio check the new rows:
    - `Student` — has the address and DOB you entered.
    - `Payment` — `type=REGISTRATION`, `status=PAID`, `studentId` set and
      `enquiryId` **null** (never both).
    - `Enquiry` — `status=ADMITTED` and `convertedStudentId` pointing at the
      student.
32. Convert an enquiry **without** a registration fee — no `Payment` row should
    be created at all.

---

## Known gaps (expected — not bugs)

- **No batch can be picked at conversion.** No Courses or Batches exist and
  those modules are still "Coming soon", so the Enrolment section says so and
  the student is admitted without a batch. The code path for creating an
  `Enrollment` is written and will work once batches exist, but you cannot
  exercise it yet.
- **Converted students are not browsable.** Students is not built, so the
  outcome is only visible on the enquiry and in Prisma Studio.
- **`parentName`, `email` and `experience` have no input** on the capture sheet
  by design (four fields only). They read "—" on the detail page until
  conversion collects them.
- **`courseId` exists on `Enquiry` but has no UI** — the capture sheet is
  restricted to four fields and Courses is not built.
- **Future follow-ups have no view.** The Follow-ups tab is deliberately
  "due today or earlier". A follow-up set for next week is only visible in the
  main list's Follow-up column.
- **Text selection inside a list row** is awkward now that the whole row is a
  click target. Say the word if copying phone numbers off the list matters.

## Things only you can do

- **Clerk dashboard**: set sign-ups to restricted / invitation-only.
- **Vercel env vars for `academy-os-beta`**: delete `ORG_SLUG` and `ORG_NAME`
  (both now unused), and set
  `NEXT_PUBLIC_CLERK_SIGN_UP_FALLBACK_REDIRECT_URL=/`. The old `/setup` route no
  longer exists.
- **Compare against the real paper form.** The spec's closing note asks for this
  and it has not been done. If the academy's application form asks for something
  the `Enquiry` fields miss, change the schema before real data goes in.
- A stray empty organization `your-academy` (0 users, 0 enquiries) is still in
  the database. Nothing resolves to it. Delete it when you like.
