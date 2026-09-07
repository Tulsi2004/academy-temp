The ~20 browser steps in temp/TESTING_CHECKLIST.md. This is the real gap. Everything I could reach server-side is verified (63/63), but the capture sheet's focus and Enter-to-submit, the duplicate warning, search box URL sync, whole-row click, overdue styling, and the whole convert form have never been exercised by a human or by me. Three of the six bugs found so far were in code that looked correct and passed a first automated pass — I wouldn't assume the UI is clean until you've clicked it.

Clerk: set sign-ups to invitation-only. Until you do, anyone who finds the URL creates an account. They'd land on /no-access, so it's not a breach — but you accumulate orphan Clerk accounts.

Vercel env vars for academy-os-beta: delete ORG_SLUG and ORG_NAME, set NEXT_PUBLIC_CLERK_SIGN_UP_FALLBACK_REDIRECT_URL=/. Do this before the next deploy — the old value points at /setup, which no longer exists.

The deployed beta is still broken. The #441 error you opened with is fixed in this code, but I don't know whether academy-os-beta has redeployed since. Worth checking the URL once Vercel picks up the push.

The spec's own closing instruction, still outstanding: photograph the academy's paper application form and visitor register, and compare against the Enquiry fields. It says to change the schema before the UI, not after — and real data is about to go in.

Also worth knowing
Two capabilities are written but untestable: batch enrolment at conversion (no Courses or Batches exist) and browsing converted students (Students isn't built). Both are marked "Coming soon" per the spec's do-not-build list, so they're expected gaps rather than loose ends.