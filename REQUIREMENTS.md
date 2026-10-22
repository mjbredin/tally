# Requirements

Status: **draft, written before any MFP file was available.** Everything under
"Source evidence" is the expected shape from public descriptions; public sources
disagree on file names and columns, so none of it is trusted until recon on the
real files at the start of slice 2. Requirements and observed behaviors below
come from the owner's description and the brief.

## Observed MFP behavior

Each behavior is something the owner uses. "Check" is a question to answer in
MFP while the subscription is still active; answers go back into this file.

| ID | Behavior | What Tally does | Check in MFP |
|---|---|---|---|
| B1 | Diary by meal: Breakfast, Lunch, Dinner, Snacks | Same four meals, renameable, more addable | Any meal renamed or added? |
| B2 | Daily calorie goal | Goal period with daily kcal | — |
| B3 | Calorie goal per meal (Premium) | Per-meal kcal goal, stored as kcal or % of daily | Set in kcal or %? Do meal goals have to sum to the daily goal? On a day-of-week override, do meal goals scale or stay fixed? |
| B4 | Macro goals in grams or percent | Both modes; grams computed from % at 4/4/9 kcal per gram | When set in %, how does MFP round the gram figure? |
| B5 | Macros shown per meal | Per-meal macro totals under each meal | Are there per-meal macro *goals*, or totals only? |
| B6 | Nutrient goals (fiber, sodium watched) | Floor or ceiling per nutrient | Which other nutrient goals are set? Sodium figure? |
| B7 | Weight arrives from Apple Health | iPhone Shortcut posts weight daily | What writes weight to Health (scale app, manual)? Pounds or kilograms? How often you weigh. |
| B8 | Food analysis: highest foods by calories, protein | Rank foods by any nutrient over any date range | Ranked by total over the range, or per serving? Which ranges do you use? |
| B9 | Barcode scan and search | Scan, then search; manual UPC fallback | — |
| B10 | Saved meals (occasional) | Save a meal from the diary, add it in one tap | How many saved meals exist? |
| B11 | Apple Health syncs into MFP; the watch's burn adjusts the day's calories; owner goes by a net calorie goal | Shortcut ingests Apple Watch active and resting energy; remaining = goal − food + adjustment | Your MFP activity level? "Allow negative calorie adjustments" on or off? Do you also log workouts by hand? Screenshot one adjustment's "i" detail mid-day and again at night. |
| B12 | Complete diary, streak | Streak counted from logged days, seeded from import | Do you press "Complete diary"? Current streak number. |

## Requirements

Numbered so tests and slices can cite them.

**History and integrity**
- R1. A diary entry stores the nutrients it was logged with. Editing or deleting a food never changes a past entry.
- R2. A goal change creates a new goal period starting on a chosen date (default today). Past days are judged against the period in effect on that day.
- R3. Every row that did not originate in Tally carries provenance (`mfp_csv`, `mfp_printable`, `apple_health`, `shortcut`, `tally_export`) and the source file name.
- R4. Unknown nutrients are null, never zero. Totals over a day report how many entries had the nutrient.

**Import**
- R5. Importing the same file twice changes nothing (row counts and content hashes identical before and after the second run).
- R6. A day with only CSV meal totals is stored as one summary entry per meal, shown as an "Imported from MFP" card, and counts as a logged day.
- R7. A day with both CSV totals and printable-diary items keeps the items and checks their sum against the CSV per meal. A difference beyond tolerance is flagged in the report, never silently resolved.
- R8. Tolerance accounts for rounding: each of *n* printable rows is rounded (error up to 0.5) and so is the CSV total, so a meal may differ by up to *n*/2 + 0.5 units per nutrient without anything being wrong. Beyond that it is a mismatch.
- R9. Every import writes a report: rows read, days covered, days skipped and why, duplicates, mismatches, row counts before and after.
- R10. Imported items seed recents and frequents.

**Goals and math**
- R11. Grams from percent: protein and carbs at 4 kcal/g, fat at 9 kcal/g.
- R12. Net carbs = total carbs − fiber, floored at 0, computed per entry and summed. Shown only when the toggle is on.
- R13. Serving scaling: entry nutrients = food nutrients per base unit × (serving size in base units) × quantity. A gram override replaces the serving when the food has a gram weight.
- R14. Recipe per-serving nutrients = sum of ingredient entries ÷ servings. Editing a recipe changes future entries only (R1).
- R15. Trend weight: exponentially weighted moving average, α = 0.1 per day, carried across missing days. Weekly rate = least-squares slope of the trend over the last 14 days × 7. Distance to goal = trend − goal weight.
- R16. Remaining = calorie goal − food + adjustment × share (default 100%). The adjustment comes from Health energy (R20) plus any exercise logged by hand, by the formula recon selects (DECISIONS.md, Apple Health).

**Data out**
- R17. Export writes every table to JSON (lossless) and CSV (readable), with a manifest of row counts and hashes.
- R18. Importing an export into an empty database reproduces the same row counts and hashes.

**Privacy**
- R19. The phone talks only to Tally's own origin. Tally's server talks only to FoodData Central and Open Food Facts, and only with a search term, UPC or food ID.

**Apple Health**
- R20. `/api/ingest` accepts cumulative daily totals (active energy, resting energy, steps) and weight samples, authenticated by `INGEST_TOKEN`. One row per day, metric and source; a later post for the same day replaces the earlier value.
- R21. Each Shortcut run posts today and yesterday.
- R22. The diary shows when Health data last arrived. A day with no Health data shows the adjustment as missing, not zero.

## Source evidence (provisional)

Expected from the brief and public descriptions. Recon replaces this section with
what the files actually contain.

**Premium export (zip of CSVs, emailed).**
- Nutrition: one row per day per meal. Date, meal name, calories, macros, a set of micronutrients (fat breakdown, cholesterol, sodium, potassium, fiber, sugar, vitamins A and C, calcium, iron), possibly a time, a note. Units probably in headers ("Sodium (mg)").
- Progress: date and weight; possibly other body measurements as extra rows or columns.
- Exercise: date, exercise name, type, calories, minutes; strength rows may carry sets, reps and weight; possibly steps; a note.

**Printable diary report (saved HTML).** A page per date range: a heading per day, a table per meal, one row per food. The food cell mixes name, brand and serving text ("Brand - Food, 2 slices"). The nutrient columns are probably the ones chosen in the web diary's settings, so a column not shown there may be missing from the report; confirm on the first saved page.

**My Foods, My Meals, My Recipes pages (saved HTML).** Lists, possibly paginated. Whether nutrients appear on the list pages or only on each item's page is unknown.

## Import and ingest tests

Derived from the requirements; each runs on synthetic fixtures shaped like the
real files, never on real data.

| Test | Covers |
|---|---|
| Re-importing a file leaves counts and hashes unchanged | R5 |
| A CSV-only day becomes per-meal summary entries and counts as logged | R6 |
| A day with items and totals within tolerance produces no flag | R7, R8 |
| A day with a 40 kcal gap produces exactly one mismatch flag | R7 |
| A note with commas, quotes and a newline round-trips exactly | R5, R17 |
| `3/4/24` and `2024-03-04` both parse; an ambiguous day/month order fails loudly | Import |
| A blank sodium cell is stored as null and excluded from the sodium average | R4 |
| `Sodium (mg)` maps to sodium in milligrams | Import |
| A duplicate day+meal row is reported, not double-counted | R9 |
| Export, then import into an empty database, gives identical counts and hashes | R17, R18 |
| Posting today's active energy as 300, then 450, leaves one row of 450 | R20 |
| A post for yesterday after midnight updates yesterday, not today | R21 |
| A day with no Health post shows the adjustment as missing and remaining as food-only | R22 |
