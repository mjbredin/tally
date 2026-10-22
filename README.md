# Tally

A personal food, macro and weight log for one person, built to replace a paid
calorie-tracking subscription at zero monthly cost. Installs to the iPhone home
screen; reviews on a desktop.

Status: plan written, awaiting approval. No app code yet. See PLAN.md.

| File | What it holds |
|---|---|
| BUILD_PROMPT.md | The owner's brief |
| PLAN.md | Architecture, data model, build order, free-tier budget |
| REQUIREMENTS.md | Observed behaviors, numbered requirements, import tests |
| DECISIONS.md | Every non-obvious choice, one line of why |
| CLAUDE.md | Conventions, commands and traps for each working session |

## Before you cancel MFP: what to save

None of this goes in git. Keep it on your machine, or attach it to the session
when slice 2 starts. Everything is saved by hand from your own account; no
scripts touch MFP.

1. **Premium export, all time.** Web: Reports → Export Data, all three files, full range. It arrives by email as a zip.
2. **Printable diary, as far back as you care about.** First set the web diary's nutrient columns to everything you want recovered (at least fiber, sodium and sugar alongside calories and macros), because the report probably shows only those columns. Then Food → View Full Report (Printable), one quarter at a time, and save each page as HTML (Safari: File → Save As → Page Source; Chrome: Save as → Webpage, HTML only). Each month saved is a month the food analysis can cover.
3. **My Foods, My Meals, My Recipes.** Save each page as HTML, every page if a list is paginated.
4. **Screenshots of every goals screen:** daily, per meal, macros, nutrients, any day-of-week settings. You'll type the numbers into Tally yourself; these are the reference once MFP is gone.
5. **Your streak number.** A screenshot.
6. **Apple Health export** (Health → your picture → Export All Health Data). Needed to check Tally's watch adjustment against MFP's, and fills any gaps in weight history. It is large; only energy, steps and weight records are read.
7. **The settings behind MFP's watch adjustment:** your activity level, whether negative calorie adjustments are allowed, and a screenshot of one adjustment's "i" detail.

## Outbound calls

The phone talks only to Tally's own origin, and so does the iPhone Shortcut that
posts Apple Health energy, steps and weight to `/api/ingest`. Tally's server calls:

- `api.nal.usda.gov`: USDA FoodData Central, with a search term, UPC or FDC ID.
- `world.openfoodfacts.org`: Open Food Facts, with a UPC, or a search term when you ask for one.

Nothing else. No analytics, fonts or third-party scripts.

## Data credits

Food data from USDA FoodData Central (public domain) and Open Food Facts
(Open Database License, openfoodfacts.org).
