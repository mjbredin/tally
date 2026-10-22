# Plan

Status: **awaiting "go".** No app code exists. Requirement IDs (R1…) refer to
REQUIREMENTS.md; the reasons behind each choice are one line each in
DECISIONS.md.

## In one paragraph

Tally is a Next.js app on Vercel's free tier with Postgres on Neon's free tier,
installed to the iPhone home screen. The phone keeps today's diary, recents and
your own foods in IndexedDB so screens render without waiting on the network,
and queues writes when the kitchen Wi-Fi drops. Every food-database call goes
through Tally's server, which caches results in Postgres. An iPhone Shortcut brings
Apple Watch energy, steps and weight in from Apple Health, which is what your
net calorie goal runs on. A diary entry
snapshots its nutrients, goals are dated, and imported MFP days keep MFP's own
totals as the number of record. Backups are encrypted nightly dumps kept by
GitHub Actions.

## Where I disagree with the brief

1. **Plan before recon.** Your files aren't pulled yet and slice 1 doesn't need them. Recon runs at the start of slice 2, and the import mapping here is provisional until then.
2. **Imported meals keep MFP's CSV total as the number of record.** Printable-diary rows are rounded per row, so their sum drifts from MFP's own total by a few units (R8). Cancel-test item 3 asks for a match "to the calorie and the gram," which only the CSV total can give. Items are stored and checked (R7) and drive food analysis, but the day's total for an imported meal is MFP's figure.
3. **Weekly review states facts, not advice.** Your answer. It compares against your goals but doesn't tell you what to change.
4. **Apple Health through a Shortcut is a Must, in slice 1.** Your weight and your watch's burn reach MFP from Apple Health, and you go by a net calorie goal the watch adjusts, so a day isn't usable without it. A web app can't read Health, and a native app costs $99 a year; DECISIONS.md has the details.
5. **Food analysis is a Must.** Your answer. It can only reach back as far as the printable diary you save, because the CSV has no food rows.
6. **Native BarcodeDetector won't be the main path.** iOS Safari ships it only behind a feature flag; the WASM decoder is what runs on your phone.
7. **Offline means cached reads and queued writes, not a sync engine.** One user and one main device; last write wins per row.
8. **Weight is stored as entered.** Converting pounds to kilograms and back drifts by 0.1, and reconciliation must match exactly.
9. **Photo and voice logging dropped** from Later to Not building, since you don't use MFP's AI features.
10. **No native iPhone app yet, and no Databricks.** Native is the only way to read Health live, but it doubles the build and costs more than MFP. A week of side-by-side numbers in slice 1 decides it. Databricks is for analytics, not per-tap writes; Neon is Databricks' own Postgres. DECISIONS.md has the reasoning.

## Architecture

```
iPhone (home-screen app)                   Vercel (Hobby)                 Neon (Free)
 ├─ service worker: app shell, offline      ├─ Next.js pages (UI)          └─ Postgres   
 ├─ IndexedDB: today, recents, my foods,    ├─ /api/* route handlers ──────────┘
 │   cached foods, write outbox             │    auth, diary, foods, goals,
 └─ talks only to Tally's origin ───────────┤    import, export, ingest
                                            └─ server-side lookups ──► FoodData Central
iPhone Shortcut (hourly) ─► /api/ingest                              ──► Open Food Facts
GitHub Actions (nightly) ──► pg_dump | age ──► encrypted artifact (90 days)
```

- **Writes** are idempotent `PUT /api/entries/{id}` with IDs generated on the phone, so a queued write replayed twice lands once.
- **Reads** render from IndexedDB first, then refresh from the server. Neon's cold start (up to a few seconds after 5 idle minutes) is never on screen for data you've already seen.
- **Search** runs locally first (frequents, then your foods and recipes), then the server (cached foods by trigram match, then live FoodData Central). Open Food Facts is used for barcodes, and for search only when you ask, because its search limit is 10 a minute and its docs say not to use it for type-ahead.
- **Dependencies beyond Next/React/Tailwind:** `@neondatabase/serverless` (database driver), `@simplewebauthn/server` and `/browser` (passkeys), `barcode-detector` (barcode decoding), `idb` (IndexedDB, about 1 KB). Charts are hand-drawn SVG. Tests use Vitest.

## Data model

Nutrient columns (`kcal, protein_g, carbs_g, fat_g, fiber_g, sugar_g, sodium_mg`,
plus the fat breakdown, cholesterol, potassium and the vitamin and mineral
columns MFP exports, with units confirmed at recon) appear on `food` and
`diary_entry` alike. Null means unknown (R4).

| Table | Key columns | Notes |
|---|---|---|
| `meal` | id, name, sort, active | Breakfast, Lunch, Dinner, Snacks to start |
| `food` | id, source (`fdc`,`off`,`user`,`recipe`,`mfp`), source_id, name, brand, upc, ref_amount, ref_unit, nutrients, gluten (`contains`,`may_contain`,`gf_label`,`unknown`), gf_safe, servings_per_recipe, archived | Unique (source, source_id); nutrients are per reference amount |
| `food_serving` | food_id, label, ref_multiplier, grams | "1 slice" = 0.28 × 100 g; grams enables the gram override (R13) |
| `recipe_ingredient` | recipe_food_id, food_id, serving_id, quantity, grams | Per-serving nutrients computed and stored on the recipe's food row (R14) |
| `saved_meal`, `saved_meal_item` | name; food_id, serving_id, quantity | Added to a meal in one tap |
| `diary_entry` | id (client UUID), day, meal_id, logged_at, kind (`food`,`quick_add`,`summary`), food_id, parent_id, name_snapshot, serving_label, quantity, grams, nutrients (snapshot), note, provenance, source_file, source_key, updated_at, deleted_at | Snapshot = history stays true (R1). An imported item's parent_id points at its meal's summary row. Unique (provenance, source_key) makes imports idempotent (R5) |
| `goal_period` | starts_on, kcal, macro_mode (`g`,`pct`), protein, carbs, fat, net_carbs, exercise_adjust_pct, weight_goal, weight_goal_unit | A day uses the latest period starting on or before it (R2) |
| `goal_weekday` | period_id, weekday, kcal, protein, carbs, fat | Overrides for a given weekday |
| `goal_meal` | period_id, meal_id, kcal or pct | Per-meal calorie goals (B3) |
| `goal_nutrient` | period_id, nutrient, direction (`min`,`max`), amount | Fiber floor, sodium ceiling (B6) |
| `weight` | measured_on, measured_at, value, unit, provenance, source_key | Stored as entered |
| `measurement` | measured_on, kind, value, unit, source, provenance, received_at | Body measurements, plus daily steps, active energy and resting energy from Health. Unique (measured_on, kind, source): a later post replaces the earlier one (R20) |
| `exercise` | day, name, kcal, minutes, detail (jsonb), provenance, source_key | Imported history and manual rows |
| `lookup_cache` | provider, kind, key, response (trimmed jsonb), fetched_at | Every FoodData Central and Open Food Facts response |
| `import_run` | provenance, file_name, file_sha256, started_at, finished_at, report (jsonb) | One per file; the report is R9 |
| `passkey` | credential_id, public_key, counter, transports | Your devices |
| `setting` | key, value (jsonb) | Timezone, units, dashboard nutrients, lockout state |

## Rules that get tests

Each is a pure function in `lib/` with a hand-worked case.

- Serving scaling and gram override (R13).
- Recipe per-serving math (R14).
- Grams from percent at 4/4/9 (R11); net carbs, floored at 0 (R12).
- Goal resolution: period by date, then weekday override, then meal goals (R2).
- Trend weight with α = 0.1, carried over gaps; weekly rate from a 14-day slope; distance to goal (R15).
- Net calories and the adjustment, including negatives and proration to the last sync (R16, R23).
- Ingest: cumulative totals replace, never add; missing Health data is not zero (R20–R22).
- Import: idempotence, rounding tolerance, null handling, date formats, duplicates (REQUIREMENTS.md, Import tests).
- Export, then import into an empty database, gives identical counts and hashes (R18).

## Screens

1. **Today.** Date strip; one section per meal with its calorie goal and macro totals; a sticky footer with calories left (goal − food + watch adjustment, with the time of the last Health sync) plus your dashboard nutrients. Tap a meal to add.
2. **Add.** One search box with recents, frequents and saved meals under it, a scan button, and quick add.
3. **Food.** Serving picker, quantity, gram override, nutrient preview, gluten warning, source shown.
4. **Scan.** Camera with a manual UPC field. On a miss it opens a new food with the UPC filled in.
5. **My foods.** Custom foods, recipes, saved meals.
6. **Weight.** Trend chart, weekly rate, distance to goal, log.
7. **Reports.** Week (averages against goals, days logged, protein adherence, weight change, per-meal averages); Food analysis (rank foods by any nutrient over any range); History (calendar of logged days and imported cards).
8. **Goals.** Periods, weekday overrides, meal goals, nutrient goals, net-carbs toggle.
9. **Settings.** Import and export, meals, units and timezone, passkeys, Shortcut token.

Dark mode follows the system and defaults to dark.

## Build order

Each slice is deployed and usable on your phone the day it lands. Each ends with
a five-item phone checklist and a known-broken list.

1. **Log a day.** Passkey and secret login (first thing tested on your phone, in home-screen mode), Today, Add with search (local, cache, FoodData Central), Food with serving picker, custom foods, quick add, copy meal and day, Goals (daily, weekday, meal, nutrient, net), the Apple Health Shortcut and its endpoint, dark mode, installable.
2. **Bring your history.** Recon on your files, then the importer (CSV, printable diary, My Foods and My Meals pages), the reconciliation report, history views, recents and frequents seeded from items, and the watch-adjustment formula checked against MFP's own adjustment rows (needs the Apple Health export).
3. **Log faster.** Barcode scanning (Open Food Facts, then FoodData Central Branded, then manual), gluten flag and your GF-safe tag, saved meals, recipes.
4. **See the trend.** Weight chart and trend, weekly review, food analysis, streaks.
5. **Make it durable.** Offline outbox and service worker, export and re-import round trip, nightly encrypted backup and one restore on purpose, a network capture of the deployed app, exercise logging.

## Free-tier budget

Computed from the assumptions in the footnote. Measured numbers replace these
for cancel-test item 9.

| Service and limit | Expected use per month | Share |
|---|---|---|
| Vercel Hobby: 1M function invocations | 5,010 | 0.50% |
| Vercel Hobby: 4 h active CPU | 0.06 h | 1.6% |
| Vercel Hobby: 100 GB data transfer | 0.15 GB | 0.15% |
| Neon Free: 100 CU-hours | 19.9 typical to 27.4 worst case | 20–27% |
| Neon Free: 0.5 GB storage | 44 MB | 8.7% |
| Neon Free: 5 GB egress | 0.2 GB | 3.9% |
| FoodData Central: 1,000 requests/hour | 40 in a heavy hour | 4% |
| Open Food Facts: 15 product reads/min, 10 searches/min | A few scans per meal | Well under |
| GitHub Actions (private repo, Free plan): 2,000 min | 240 (backups plus CI) | 12% |

Assumptions: 150 server calls a day at 50 ms CPU and 10 KB each, plus 17 hourly
Health syncs (6 a.m. to 10 p.m.); Neon compute pinned to 0.25 CU, each sync
waking it for 5.5 minutes, plus 1 hour (typical) or 2 hours (worst, no overlap
with syncs) of logging a day and a 6-minute nightly backup wake; 10 years of history at 4 meals a day, 12 months of printable items
at 20 a day, a year of new logging, 3,000 cached foods, doubled for indexes;
5 MB compressed dump nightly; backups at 2 minutes a night and CI at 3 minutes
per push for 60 pushes.

The one limit to watch is Neon compute, and the hourly sync is most of it.
Pinning to 0.25 CU keeps the worst case under 30%.

## Outbound calls

| From | To | Carries |
|---|---|---|
| Phone | Tally's own origin only | Everything |
| iPhone Shortcut | Tally's `/api/ingest` | Today's and yesterday's energy and steps, latest weight; `INGEST_TOKEN` |
| Tally server | `api.nal.usda.gov` (FoodData Central) | Search term, UPC or FDC ID; API key |
| Tally server | `world.openfoodfacts.org` | UPC, or a search term when you ask; User-Agent `Tally/1.0 (personal use)` |
| GitHub Actions | Neon; GitHub artifact storage | Nightly dump, encrypted before upload |

No fonts, analytics or scripts from anywhere else. A network capture of the
deployed app in slice 5 is the evidence.

## Environment variables

| Variable | Set in | Comes from |
|---|---|---|
| `DATABASE_URL` | Vercel | Neon console → Connection string, pooled |
| `DATABASE_URL_DIRECT` | Local `.env`, GitHub Actions secret | Neon console → Connection string, direct (migrations and `pg_dump` need it) |
| `FDC_API_KEY` | Vercel | api.data.gov/signup, free, emailed |
| `SESSION_SECRET` | Vercel | `openssl rand -base64 32` |
| `ENROLL_SECRET` | Vercel, your password manager | `openssl rand -base64 33`; logs you in and adds passkeys |
| `INGEST_TOKEN` | Vercel, the iPhone Shortcut | `openssl rand -hex 32` |
| `APP_ORIGIN` | Vercel | The production URL; passkeys bind to it |
| `AGE_RECIPIENT` | GitHub Actions secret | `age-keygen` on your machine; the private key goes to your password manager and nowhere else |

## Risks

1. **Passkeys inside an iOS home-screen app are undocumented.** If they fail, the enrolment secret is the login (autofilled from your password manager). Tested first in slice 1.
2. **Camera behavior in home-screen mode is unverified.** Permission may be asked again on each launch. Fallbacks: decode a photo taken through the file picker, and manual UPC entry.
3. **The printable diary probably shows only the nutrient columns chosen in the web diary's settings.** Set them before saving (README). The longest range MFP will print is unknown, so save in quarterly chunks.
4. **The export may not match the expected shape.** Public sources disagree on file names and columns. Each file type gets its own small adapter written after recon.
5. **Gluten data is sparse.** Open Food Facts covers many US products thinly. "Unknown" is shown as unknown, never as safe; only your own tag makes a food safe.
6. **Backups are only as good as the key.** Lose the `age` private key and the backups are unreadable. Keep it in two places.
7. **Health data is unreadable while the phone is locked.** Hourly runs that land then fail; the diary shows the last sync time, and a Sync button opens the Shortcut through a `shortcuts://` link (to verify on the phone).
8. **Raw Health samples overlap across phone and watch.** The Shortcut reads Apple Watch samples only; if a day looks doubled against the Fitness app, that filter is the first suspect.
9. **Tally's calories left may lag MFP's between syncs.** Measured in slice 1 against MFP; past 50 kcal at more than one meal in five, we go native.

## Clean-room check

Nothing in Tally talks to MFP's servers, and nothing we do to build it does
either. The evidence is files you exported or saved by hand from your own
account, plus your descriptions of how MFP behaves. The parsers are written from
the structure of those files. python-myfitnesspal is consulted only if a page
structure needs it, after its license is checked, with credit in DECISIONS.md.
Foods enter Tally from FoodData Central (US government, public domain), Open
Food Facts (ODbL, credited in the README) and your own entries. The only MFP
food data that comes in is what you personally logged, as part of your history.
Screens are designed from the jobs they do; no MFP names, copy, icons or layouts.

## What starts slice 1

- Your "go."
- The private `tally` repo, reachable by the Claude GitHub App.
- A Neon account (sign in with GitHub) and an FDC key. Your Vercel account already exists.
- The MFP harvest in README.md can run in parallel; slice 2 needs it.
