# Decisions

One line of why for every non-obvious choice. Newest section last. "Owner" means
Mike said it; everything else is Claude's call and can be reversed by saying so.

## 2026-10-22 · Phase 0

Owner's answers, verbatim in substance: logs every food daily by Breakfast,
Lunch, Dinner, Snacks; Premium; calorie goals for the day and for each meal;
macro goals and nutrient goals; barcode scanner and search; saved meals
occasionally; weight arrives from Apple Health; uses the food analysis
(highest foods by calories, protein); watches fiber and sodium; does not use
meal planning, the AI assistant, coaching or the weekly digest; "more about the
facts." On the remaining Phase 0 questions: "use your best judgement."

### Scope

- **Weekly review is facts only.** No coaching, no "what to change." Owner.
- **Food analysis is a Must:** rank foods by any nutrient over any date range. Owner uses it in MFP.
- **Per-meal calorie goals are a Must** and amend cancel-test item 5. Owner sets them in MFP.
- **Nutrient goals carry a direction.** Fiber is a floor, sodium a ceiling; the UI colors accordingly. A single "goal" number cannot express both.
- **Shortcut ingestion moves from Should to Must, slice 5 to slice 4.** Weight reaches MFP from Apple Health today; cancelling breaks that link unless we replace it.
- **Not building: meal planning, AI assistant, coaching, weekly digest, photo or voice logging.** Owner doesn't use them; photo/voice was Later, now dropped.
- **Recipes stay, built small.** Owner didn't mention them; they reuse the saved-meal tables plus a servings count, so the cost is low.
- **Exercise: import the history, log manually, adjust the budget by 0% by default.** Owner didn't mention exercise; recomp lifters rarely eat it back. The setting (0–100%) exists. If you eat back exercise calories in MFP, flip it.
- **Dashboard defaults: protein, fiber, sodium.** Net carbs available, off by default; owner didn't mention it.

### Stack and infrastructure

- **App name Tally, repo `tally`, private.** No MFP in the name, per the brief; short enough for a home-screen label.
- **Next.js + TypeScript + Tailwind on Vercel Hobby.** Owner's stack; a Vercel account already exists.
- **Postgres on Neon Free, not Supabase.** Supabase free projects pause when idle and need a manual restore; Neon suspends after 5 minutes and wakes on the next query, and the phone hides the wake behind a local cache.
- **Not Turso.** It would work, but Postgres is what you query fluently and `pg_dump` is the backup story.
- **Plain SQL migrations and `@neondatabase/serverless`; no ORM.** You read SQL; one dependency; the schema is reviewable as SQL.
- **All food-database calls go through our server, never from the phone.** The key stays server-side, cached results are shared, and the phone talks to one origin, so the item-8 network capture is one line long.
- **Free `*.vercel.app` domain.** A custom domain costs money. Passkeys bind to the domain, so moving later means enrolling a passkey again.
- **Barcode decoding through the `barcode-detector` ponyfill (ZXing compiled to WASM).** iOS Safari still ships BarcodeDetector only behind a feature flag; the ponyfill uses the native API wherever it exists.
- **Offline: cached reads plus a persisted write outbox with client-generated IDs.** Not a general sync engine: one user, one main device, and last-write-wins per row is acceptable.
- **Backups: nightly GitHub Actions job, `pg_dump` encrypted with `age` to a public key, kept as a 90-day workflow artifact.** Free, off Vercel and off Neon, and GitHub never sees plaintext. The private key lives only in your password manager. Neon's 6-hour restore window covers same-day mistakes.

### Auth

- **Passkey (Face ID) first; a long enrolment secret is the fallback login and the way to add a passkey.** No source confirms passkeys inside an iOS home-screen web app, so the secret covers that case for a few lines of code. Five wrong secrets lock login for 15 minutes. Verified on your phone in slice 1 before anything else is built on it.
- **Sessions are signed HTTP-only cookies, 90 days, rolling.** Rotating `SESSION_SECRET` signs out every device.

### Data

- **Every diary entry snapshots its nutrients at log time.** History stays true; the brief requires it.
- **Goals are dated periods with weekday overrides; a day uses the period in effect on that date.** Editing a goal starts a new period today.
- **Weight is stored as entered (value plus unit), converted only for display.** Pound-to-kilogram round trips drift by 0.1 and reconciliation must match MFP to the decimal.
- **One home timezone setting; diary days are local dates.** MFP CSV dates are already account-local, so they map straight to a diary day.
- **Blank nutrient means unknown (null), never zero.** MFP leaves micronutrients blank when a food doesn't carry them; zero would understate averages.
- **Real MFP data never enters git.** `imports/` is gitignored; tests use synthetic rows shaped like the real files.

### Process

- **The plan comes before data reconnaissance; recon moves to the start of slice 2.** The files aren't pulled yet and slice 1 doesn't depend on them; waiting would stall everything. The import mapping in PLAN.md is provisional until recon.
- **Goals are entered by you in the app's Goals screen, not sent to me.** They don't export, and the app is where they belong. Screenshots of MFP's goal screens are still worth keeping as the reference for cancel-test item 5.
- **REA is not installed in the build session; CSV and HTML are parsed directly.** The brief allows it, and REA adds nothing over a parser for these formats.
- **Yellow-tier observation of your MFP web session: not authorized by default.** Saved HTML first; ask again only if it leaves a specific gap.
- **No cancel date set.** Don't let an annual renewal go through before the cancel test passes; switch to monthly if one is near.
