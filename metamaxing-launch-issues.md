# MetaMaxing launch tool - issues from the BubbleDry launch (17 Nov 2026)

What we tried to build: a BubbleDry (BD) test campaign set up like `VT|WW|CBO|Broad|Test1|2` (one ad per video, 2 texts, 2 headlines), with 2 ad sets like `FS|CBO|WW|WinnersBlackFriday|Test#178|Nov26` (same creatives, one ad set on the HeyWondr page, one on Megan's Mom Finds), 70 extra excluded countries, £70/day, starting at midnight.

What we got: one ad set only, starting "now" instead of midnight, after a failed first build that left a half-built campaign behind (`120254293872490577`, which has to be deleted by hand).

---

## Must fix (blocked or broke the launch)

### 1. A start time in the future breaks the build
- **What happened:** the build stopped at "Setting the ad set's name, start and changes". Meta's error: `"start_time" cannot be edited if the ad set has already started. [code 100]`
- **Cause (inferred from the error):** the tool copies a live ad set (VT, running since 16 Nov), so the copy appears to keep the old start date and Meta treats it as already started. The tool then tries to change the start time afterwards, and Meta refuses.
- **Also affects the default:** the default start time is "00:00 tomorrow", so a launch that keeps the default could fail the same way.
- **Fix:**
  - Pass `start_time` inside the copy request itself (Meta's ad set `/copies` endpoint accepts `start_time` / `end_time`), or create the ad set with a plain `POST /adsets` using the template's targeting, pixel and optimisation.
  - Never edit `start_time` after the ad set exists.
  - Make "start at 00:00 on date X, UK time" a supported, tested option.

### 2. A failed build leaves half-built objects that can't be fixed or removed
- **What happened:** after the failure, the campaign and ad set existed in Meta, but the ad set had not yet had its name, start time or country exclusions applied. `plan_launch` then refused any change: `daily_budget, ad_set_name, remove_countries, start_time can't change here`.
- **The only way out:** start a new draft, which created a second campaign with the same name and left the broken one behind.
- **Risk:** someone could switch on the broken campaign, which has the wrong targeting.
- **Fix:**
  - Do every ad set change in the create call so there is no half-configured state, or roll back on failure (delete or archive whatever the draft created).
  - Add a `cleanup_launch(draft_id)` / "delete what this draft created" action.
  - Allow editing a half-built draft's campaign and ad set settings: name, budget, countries, start time, Page.

### 3. Only one ad set per launch, and no choice of Page
- **What happened:** the FS Test#178 pattern (2 ad sets, same creatives, different Pages: HeyWondr and Megan's Mom Finds) can't be built. The Page is always copied from the template ad set.
- **Existing support:** the database already supports this. `upload_batches.targets[]` and `batch_ads` record one ad per creative per target, with `page_id`, `instagram_user_id` and `ad_set_name`. The bulk uploader can do it; the launch tool can't.
- **Fix:**
  - Add `ad_sets: [{name, page_id, instagram_user_id, countries?, start_time?}]` to `plan_launch`.
  - Upload each creative once and reuse it in every ad set.
  - Or accept "copy the structure of campaign X" and repeat its ad sets and Pages.

### 4. Can't build an ad set from scratch or choose the template
- **What happened:** the template is picked automatically: the latest campaign of the same strategy, from any product. You can't say "use this campaign as the template". The user gave a campaign link, and it had to be matched by hand.
- **Fix:**
  - Accept a template campaign or ad set (ID, name, or Ads Manager link).
  - Add a "fresh ad set" mode that builds from saved settings (pixel, optimisation, placements, countries) without copying a live ad set. `launch_presets` looks like the place for this.

### 5. Nothing can be edited after a launch
- **Wanted:** change the budget, start time, countries, names or Page, or add another ad set to the campaign afterwards. Today the only write actions are "create a new paused campaign" and "add a new ad set to an existing campaign".
- **Fix:** add simple edits on objects the tool created: rename, budget, start time, countries, add or remove ads.

---

## Upload page

### 6. The upload link changes on every plan change, and old links stop working
- **What happened:** every change to a plan (budget, file names) produced a new drop page, and the previous one returned 404. The user was sent 4 different links.
- **Unconfirmed:** the user reported the first link "not working" before any change. The cause wasn't confirmed.
- **Fix:** one fixed upload page per draft that survives plan changes.

### 7. File names must be known in advance and match exactly
- **What happened:** the plan needs every file name upfront. Placeholder names (`..._17_11_2026`) didn't match the real files (`..._09_11_2026` for C1-C3, `..._17_11_2026` for C4-C5). Each correction meant re-planning and getting a new link.
- **Fix:**
  - Let the user drop any files and take the ad names from the real file names.
  - Show the resulting ad names to confirm.
  - Or allow renaming without making a new plan.

### 8. A new draft needs every file uploaded again
- **What happened:** draft 5 asked for all 5 videos again, even though they were already on Meta. The same video IDs were reused, so the upload was wasted effort.
- **Fix:** let a draft reuse creatives from an earlier draft, or reuse video IDs already uploaded.

---

## Plan and validation

### 9. Country names with special characters are rejected
- **What happened:** `Côte d’Ivoire` (curly apostrophe) failed with "Unknown country", so the whole list had to be converted to ISO codes.
- **Fix:** normalise apostrophes and accents, and accept common names and aliases: Türkiye/Turkey, Côte d'Ivoire/Ivory Coast, Palestine, Caribbean Netherlands, Sint Maarten, Eswatini, Cabo Verde, DR Congo / Republic of the Congo.

### 10. The plan promised a setting Meta then rejected
- **What happened:** the summary said "Kept on: translate_text". During the build, Meta rejected translation on all 5 ads and the tool left it out, which only showed up in the build notes afterwards.
- **Fix:** check which settings Meta accepts while planning (a dry-run validation call), or show a warning in the summary before go.

### 11. Duplicate campaign names
- **What happened:** the retry got the same name, `BD|WW|CBO|Broad|Test1|1`, as the broken campaign. Name numbering comes from stored history, which doesn't yet include campaigns created minutes earlier.
- **Fix:** check Meta (not only stored history) for existing names before suggesting one, and warn on duplicates.

### 12. The state of a half-built launch can't be checked
- **What happened:** the new campaign and ad set weren't in `ads_entities` yet, so there was no way to see what the half-built ad set actually had (name, countries, start).
- **Fix:** `get_launch` should read back the live state from Meta for every object it created.

### 13. The summary is hard to read
- **What happened:** the countries section lists about 200 country codes twice.
- **Fix:** show counts and only the change, e.g. "Excluded: 147 countries (VT's 129 + your 70, 52 already excluded)", with the full list available on request.

---

## Suggested launch flow once fixed

```
plan_launch(
  product="BD",
  template="VT|WW|CBO|Broad|Test1|2",          # explicit template
  strategy="per-creative",
  daily_budget=70,
  start_time="2026-11-18 00:00",               # works at create time
  remove_countries=[... names or codes ...],
  ad_sets=[
    {name: "Testing1|Broad|PP|Vid - HW",      page: "HeyWondr"},
    {name: "Testing1|Broad|PP|Vid - HW - 2Fb", page: "Megan's Mom Finds"},
  ],
)
-> one fixed upload page, any file names, ad names from the files
-> dry-run check against Meta (start time, settings, names)
-> launch: all or nothing (rolls back on failure)
-> edit later: budget, start, countries, names, Page, add ad set
```
