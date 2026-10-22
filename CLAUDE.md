# Tally: instructions for every session

A single-user replacement for a paid calorie-tracking subscription. The owner's
brief is BUILD_PROMPT.md; read it before changing anything. PLAN.md is the
approved shape (once approved), DECISIONS.md the reasons, REQUIREMENTS.md the
numbered requirements tests cite.

## Where we are

Phase 2: PLAN.md written, waiting on the owner's "go." No app code yet. Next:
slice 1 (log a day). Update this line at the end of every session.

## Hard rules (from the brief)

- One user. No second account, no sharing, no analytics, no third-party scripts.
- $0 a month. Free tiers only; anything paid is optional, off and bring-your-own-key.
- History stays true: entries snapshot nutrients; goals are dated periods.
- The phone talks only to Tally's origin. The server talks only to FoodData Central and Open Food Facts. A new outbound call needs the owner's OK and a README entry.
- Secrets live in environment variables. Real MFP data never enters git (`imports/` is ignored); tests use synthetic fixtures.
- Nothing touches MFP's servers, apps or database. REA rules in BUILD_PROMPT.md; Red means no.
- Ask first: anything that costs money, deletes or rewrites data, changes the schema after import, or touches auth.
- Every new dependency gets one sentence of why in its commit message.
- Tests where they protect data (import, serving math, recipe math, net carbs, goals, export round trip). Everything else gets a phone checklist.

## Reporting

Three lines: what works, what doesn't, what you need. Never "should work":
show the command and output or the URL and steps. Numbers come from running
something.

## Stack

Next.js, TypeScript, Tailwind on Vercel Hobby; Postgres on Neon Free through
`@neondatabase/serverless`; plain SQL migrations; Vitest. See PLAN.md for the rest.

## Commands

None yet. Slice 1 adds them here.

## Known traps

- iOS Safari has BarcodeDetector only behind a feature flag; the `barcode-detector` ponyfill is the real path.
- Passkeys in an iOS home-screen app are undocumented; the enrolment secret is the fallback login.
- Home-screen apps on iOS keep their own cookies, separate from Safari: log in inside the installed app.
- Neon suspends after 5 idle minutes; the first query can take seconds. Never block a screen on it.
- Open Food Facts: 15 product reads and 10 searches a minute per IP, custom User-Agent required, no search-as-you-type.
- FoodData Central: 1,000 requests an hour per key; going over blocks the key for an hour.
- `pg_dump` needs Neon's direct connection string, not the pooled one, and a client version at least as new as the server.
- Health data can't be read while the iPhone is locked; Shortcut runs that land then fail. Missing Health data is never zero.
- Shortcuts' Find Health Samples returns raw samples from every source; sum Apple Watch samples only or phone and watch double count.
- The Shortcut posts cumulative day totals: upsert, never add.
- MFP CSV: account-local dates, possible Excel-mangled dates, units in headers, blank micronutrients (null, not zero), notes with commas.
