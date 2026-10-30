# Review: Ethiopia Process Monitoring – One Checklist, All Activities, Q3 2026

**Scope:** 2 XLSForms (Version 1, Exception Update Version 2) and 2 MoDa exports (v1: 15,256 submissions; v2: 21,836 submissions), plus their repeat-group sheets.
**Method:** Converted both forms with pyxform 4.5 + ODK Validate. Ran a custom semantic lint: skip-logic literals against choice codes, choice filters, constraints. Profiled the exports for timing, GPS, duplicates, collector behaviour, cascade use, outliers and repeat linkage.

---

## 1. Executive summary

| # | Severity | Finding | Impact |
|---|---|---|---|
| 1 | 🔴 Critical | **Kebele cascade is broken.** 100% of answered Kebele questions are "Other" (10,777/10,777 in v1, 15,685/15,685 in v2). | Kebele is free text only (2,766 and 3,543 distinct spellings). You cannot aggregate at kebele level. |
| 2 | 🔴 Critical | **Dead skip logic: 27 conditions test choice codes that don't exist.** Confirmed in data, e.g. *"Who informed you?"* has **0 answers** although **12,175** respondents said they were informed in advance. | Whole follow-up questions were never asked (see §2.2). The data is lost and can't be recovered. |
| 3 | 🔴 Critical | **Food basket: "distributed" and "entitlement" are answered in different units** (per person vs per household). In 17% of v1 rows, received > entitled, and in 827 v2 rows the beneficiary was marked "correct amount" while the figures differ by >10%. | Food-basket accuracy KPIs computed from these fields are unreliable. |
| 4 | 🟠 High | **Timestamps are unreliable as paradata.** 1,129 forms have a negative duration, 12,153 were left open more than 24 h, and 2,608 were completed in under 5 min. 3,065 have an IntDate more than 1 day away from the device start date. | You cannot verify that visits happened when reported. Interview length can't be used for QA. |
| 5 | 🟠 High | **v1 and v2 are different form IDs with unstable auto-generated column names** (`reserved_name_for_field_list_labels_781` → `_783`, etc.). v2 has **no `version`** in settings. v1 data contains 111 records from version `202606301311`, which matches neither XLSForm supplied. | Merging Q3 into one dataset needs a manual column map. The audit trail of which form produced which data is incomplete. |
| 6 | 🟠 High | **All 37,092 submissions were sent by one shared account (`subofficeModa`).** 41–45 devices are used by more than one collector. | There's no per-user accountability, and you can't use server permissions to restrict data by sub-office or partner. |
| 7 | 🟡 Medium | `monitoring_month` is entered by hand and disagrees with IntDate in 982 records. Values include April, May, November and December in a Q3 dataset. | Monthly reporting will misallocate visits. |
| 8 | 🟡 Medium | 132 of 245 numeric questions have no constraint. Values include a family size of **253,007,456**, a loan of 27.6 M ETB and warehouse stock of 15,837 t. | Means and totals are distorted unless you clean these values. |
| 9 | 🟢 Low | Form hygiene: 159 list names with stray spaces, 215 unused choice lists, 507–544 duplicate FDP labels, 12 orphan rows carrying `required=yes`, 112 constraints without a message, and images without `max-pixels`. | Mostly maintainability and enumerator usability. |

**What is working well:** both forms compile and pass ODK Validate. There are no duplicate `_uuid` or instanceID values, and only 7 content-level duplicates. Every GPS point falls inside Ethiopia and is plausible for its region. Repeat groups have no orphan rows. Consent "No" records stop correctly (0 of 63 have substantive answers). "Other, specify" text is good quality (≈0.1% junk entries). The v2 change adding `FDPOth` is working: 614 records captured FDPs that were missing from the list.

---

## 2. XLSForm review

### 2.1 Compilation
Both forms convert with code `101 – Ok with warnings`. The warnings are:
- 8 `image` questions without `max-pixels` (rows 616/645/1728/1761–1763/1784–1785 in v1). Add `max-pixels=1024` in `parameters` to reduce upload size. This matters because 1,100 photos were uploaded in v2 alone.
- 12 orphan rows that have only `required = yes` and no type or name (v1 rows 2012, 2102–2103, 2302–2304, 2351–2353, 2388–2389). These are leftovers of deleted questions and should be removed.
- `or_other` is used together with a language-tagged label column. pyxform warns that the generated "Other" label is untranslated.

### 2.2 Dead skip logic (value tested ≠ actual choice code)

These are the most important form defects. v2 row numbers are shown.

| Row | Question shown when… | Problem | Data evidence |
|---|---|---|---|
| 661 | `_1_1_If_yes_who_informed_you` ← `${Were_you_informed_in_advance_a}='yes'` | The list `pu3kq07` codes "Yes" as **`AUTOMATIC`** (a Kobo import artefact), not `yes`. | 0 answers; 12,175 "Yes" |
| 1400 | `FFV_Sell2` ← `${FFV_Sell1}='yes'` | `FFV_Sell1` uses list `Yesno`, where Yes = **`1`**. | 0 answers |
| 2047 | `garden_support_other` ← `selected(..., 'other_garden_support')` | The "Other" code is **`99`**. | 0 answers |
| 350 | `HHAsstMode_oth1` ← `selected(${HHAsstMode1},'999')` | `AsstModality1` has only 1/2; there is no Other option. | 0 answers |
| 1687 | `MOWhWhatComm_oth` ← `selected(...,'999')` | `CommoditiesDM` has no `999`. | 0 answers |
| 824, 838 | `oth_rowfood`, `othnotgood` ← `selected(${C09.2},'99')` | `C09.2` uses `or other`, so the code is **`other`**. `oth_rowfood` is also a *ration-size* select, not an "other, specify" text field. | 0 answers |
| 52, 98–100, 182–193, 241, 320–321 | Various CBT questions ← `selected(${HHAsstMode},'5'/'6')` | `AsstModality` only has 1, 2, 999. The `'5'`/`'6'` branches are harmless leftovers, but `MOAsstCBTRed` (row 321) depends **only** on `'6'`, so it can never show. | – |

**Fix:** correct the literals and re-deploy. Add a CI-style check that compares every `${q}='x'` and `selected(${q},'x')` against `q`'s choice list before each deployment.

### 2.3 Kebele cascade (root cause of finding #1)
`Kebele` uses `choice_filter: FDPbyKebele=${FDP} or FDPbyKebele=99`. In the `Kebele` list (62 rows), `FDPbyKebele` holds FDP **labels** such as `Bonta` and `Halalidebi`. `${FDP}` holds FDP **codes** such as `Bonta00xxx`. Only 1 of 62 values matches an FDP code, so the filter only ever returns option 99 "Other".

**Options:** (a) re-key the `FDPbyKebele` column to FDP codes and complete the list to cover all FDPs; or (b) drop the Kebele select and keep a single text field. Then build a cleaned kebele lookup from the 6,300+ free-text values already collected.

Related: the FDP list has 6,600+ codes built as label plus a 5-digit suffix. 507 labels (v1) / 544 labels (v2) appear more than once ("Alamata", "Babako", "Bala", …), and some codes don't match their labels (`2 Kebele00108` → "4 Kebele", `God-God 0004527` → "God-God 02"). Enumerators can't tell duplicates apart. Add the woreda or kebele to the label, or deduplicate the master list.

### 2.4 Constraint and logic errors
- `Hhsize` (row 67): the constraint is on a **note**, so it is never evaluated. It is also `.>0 or .<=20`, which is always true. `HH_Total` (row 70) is a `calculate` with the same `or` constraint, which is also never enforced. Use `. > 0 and . <= 30` on an input field.
- `MOIKCashRecHHSize` ("Family size based on ration card") has **no constraint**, and the data includes 253,007,456 and 7,080,312 (phone numbers typed into the field). Add `. >= 1 and . <= 30`.
- `HHAsstIKLast` ("how long ago…") allows 1–99 with no unit in the label. The 99 values look like a "don't know" code. Specify the unit (days/weeks/months) and use a separate "don't know" option.
- **Food basket repeat:** `MOAsstIKRecAmt` is labelled "amount that was distributed… in Kg" and `MOAsstIKRecAmt1` "entitlement the beneficiary anticipated…". In practice, monitors record the per-person ration in one and the household entitlement in the other (for example 10 kg against 50 kg, a ratio equal to household size). Make both labels say "for the **household**". Add `MOAsstIKRecCorrect` as a calculate or a constraint check (`abs(received − entitled)/entitled ≤ 5%`) instead of a separate subjective Yes/No.
- 132 numeric questions have no constraint, including `MOWhAvgTon`, `MOWhSizeMeterSq`, `SavingsAmount`, `TotalSavings`, `LoanAmount`, `landSize`, `HHlivestock*`, `CSB/PlumpyEntitlement/Received` and `Cash01–16`. Add plausible ranges, using soft warnings where your ODK client supports them.
- 112 constraints have no `constraint_message`. Enumerators see a generic error and often work around it.

### 2.5 Versioning and deployment
- **v2 `settings.version` is empty.** MoDa assigned `202608121315` automatically. Set an explicit version (for example `2026081201`) so the XLSForm file can be traced to the data.
- v1 data contains **111 submissions on version `202606301311`**, which is not in the XLSForm you supplied (`202603312116`). Archive every deployed revision.
- Deploying v2 under a **new form_id** instead of as a new version of the same form splits Q3 into two server datasets (`_xform_id` 388675 / 391761). Unless the change was structurally incompatible, a new version of the same form_id keeps one dataset.
- Questions inside `field-list` groups that use label-only rows generate `reserved_name_for_field_list_labels_NNN` columns. The number is the **row position**, so it changes whenever a row is inserted above it (781 → 783, 798 → 800, …). Several of these are select_multiple questions that hold real answers ("Foods are spoiled by rain", "Raw foods (cereals)…"). Give these rows explicit names so the columns stay stable across versions.

### 2.6 Hygiene and maintainability
- **Choice codes with spaces or punctuation:** `Activity` (for example `Activity 5 ( School based programs)`, with a double space and a stray parenthesis), `FDP1`, `woreda`, `Zone`, `Partner`, and staff-name lists. These work for select_one but are fragile in expressions, filters and exports. `select_Setting` codes contain line breaks (`'1\n'`). `InterviewTargetRespondent` codes contain non-breaking spaces. `safety_rating` has `' Safe'`. Use short ASCII codes (`act1`, `act2`, …).
- 159 `list_name` cells have leading or trailing spaces (`' yes_no_NA'`, `'Grade '`). pyxform tolerates these, but other tools may not.
- Type spellings are mixed (`select one`/`select_one`, `begin group`/`begin_group`, `or other`/`or_other`). Standardise them.
- 215 unused choice lists, `Sheet3`, `Sheet4` and `FuzzyLookup_AddIn_Undo_Sheet` (10,000+ rows) inflate the file. Move them to a separate master-data workbook.
- `required = yes` set on `begin_group`/`end_group` rows has no effect. Remove it.
- Staff-name lists (`name`, `WfamName`, `EnumeratortName`) contain duplicate labels (17/17/12) and inconsistent spelling or case. Use a staff ID as the code and the name as the label.
- The FDP `relevant` in v2 lists `'Market and Mills Monitoring'` twice. The `choice_filter` repeats `distrPointByWereda = 99` both inside and outside the activity clause, which is harmless but redundant.
- Metadata: add `username`/`phonenumber`, or better, per-user accounts (see §3.4). Rename `starttime`/`endtime` back to `start`/`end`, or document them for analysts.

---

## 3. Dataset review

### 3.1 Export format
- The first data row of every sheet is a **label row** ("Information collected by :", "_id", …). Drop it before analysis. The true counts are v1 = 15,256 and v2 = 21,836.
- Values and select_multiple dummy columns are exported as **labels** (`HHAsstMode/Food`). Labels with `/`, `'` or commas make column names awkward, and any later label edit breaks your scripts. Export with **XML values and headers** for analysis, and keep a label export for reading.
- 3,606/3,608 columns, of which 1,719 (v1) and 1,311 (v2) are completely empty. This is expected for a multi-activity form, but analysts should split the data by `Activity` × `DMSvyMode` before working with it.

### 3.2 Timing and paradata

| Check | v1 | v2 |
|---|---|---|
| Negative duration (end < start) | 424 | 705 |
| Duration < 5 min | 1,122 | 1,486 |
| Form left open > 24 h | 5,032 | 7,121 |
| Start time after server receipt | 53 | 369 |
| IntDate after upload date | 9 | 24 |
| Device clock wrong (start in 2023) | 0 | 2 |
| IntDate ≠ device start date by > 1 day | 1,386 | 1,679 |
| `monitoring_month` ≠ IntDate month | 519 | 463 |
| Upload lag > 7 days | 904 | 1,014 |

Interpretation:
- `end` records the **last save**, so drafts that are reopened inflate the duration (the median is 8–9 h). Negative values and start-after-upload point to device clock changes or edits of sent forms.
- `IntDate` is entered by hand, and 9–24 records carry dates later than their upload date.

Recommendations:
- Default `IntDate` to `today()` and constrain it to `. <= today() and . >= today() - 3`.
- Calculate `monitoring_month` from `IntDate` instead of asking for it.
- Enable the ODK `audit` meta question (with `track-changes=true`) for real per-question timings.
- Instruct devices to use network time.
- Under-5-minute forms cluster in a few collectors. 11 collectors with ≥30 forms have more than half of them under 5 minutes; supervisors should check these with back-checks. The collector names are in the separate flag file, not in this repository.

### 3.3 Productivity
- The median is 5 submissions per collector-day. 188 (v1) and 261 (v2) collector-days had ≥20 submissions, and 16/39 had ≥40, with a maximum of 82 in a day. For beneficiary interviews this is not plausible. Check whether those days were observation or food-basket checklists, which are legitimately short, and spot-check the rest.

### 3.4 Accounts and devices
- Every submission comes from the single account `subofficeModa`. 41 (v1) and 45 (v2) devices were shared by 2–4 collectors, and 4 collectors used more than 3 devices. Create one MoDa account per monitor, or at least per partner or sub-office. Attribution then no longer relies on a name picked from a dropdown.

### 3.5 GPS
- All points fall inside Ethiopia, and regional medians are consistent with the selected region.
- 271 (v1) and 475 (v2) fixes have precision > 50 m, and 21 have precision > 500 m. Set `accuracyThreshold=10` (or a similar value) on `HHCoord`.
- 148/177 submissions share identical coordinates with ≥5 others. This is acceptable for multiple interviews at one FDP, but cross-check it against the collector.

### 3.6 Content
- **Food basket repeat** (6,169 / 8,280 rows): received > entitled in 1,041 / 1,191 rows, and > 2× entitled in 612 / 537 rows (the unit issue in §2.4). In v2, 827 rows are flagged "correct" despite a > 10% difference, and 91 rows are flagged "not correct" while within 2%. There is 1 duplicate commodity within the same household.
- **Outliers** (examples): family size 253,007,456; `TotalSavings` 36,000,000; `LoanAmount` 27,659,560; `MOWhAvgTon` 15,837; `HHlivestock5` 2,323; `CSBEntitlement`/`Received` 3,000; `_3_2_How_many_Sachet_real_Plus_were_lost` 900. Run these through cleaning before you aggregate.
- **Coverage:** in v1 the Activity 5 (school-based programmes) checklist has **0** records. It first appears in v2 (461). Check whether this is a deployment gap or a planning decision.
- **Consent:** 63 refusals, all correctly stopped.
- **Duplicates:** 5 (v1) and 2 (v2) records have identical answers except for metadata. Review them and remove the extras.

---

## 4. Prioritised action plan

**Before the next deployment (Q4 form):**
1. Fix the 27 dead skip-logic literals (§2.2), especially `pu3kq07` (`AUTOMATIC` → `yes`), `FFV_Sell2`, `garden_support_other` and `oth_rowfood`/`othnotgood`.
2. Fix the Kebele cascade or replace it with text (§2.3), and deduplicate FDP labels.
3. Add range constraints and messages to all numeric fields. Fix the `Hhsize` and `HH_Total` constraints.
4. Clarify the food-basket units and calculate the "correct amount" flag.
5. Default and constrain `IntDate`, calculate `monitoring_month`, enable `audit`, and set a GPS accuracy threshold and `max-pixels`.
6. Set `settings.version`, keep one `form_id` per quarter, and give every field-list label row an explicit name.
7. Create per-user MoDa accounts.

**For the Q3 dataset now:**
1. Drop the label row, merge v1 and v2 using a column map (the reserved_name columns are offset by 2), and tag each record with its source version.
2. Use the flag file (counts below) to target back-checks. Prioritise records with ≥3 flags, and collectors with a high share of forms under 5 minutes.
3. Recode `KebeleOth` and `FDPOth` free text against the master location list.
4. Recalculate `monitoring_month` from `IntDate`.
5. Treat food-basket accuracy and the "who informed you" indicators as **not reportable** for Q3. Document this limitation.
6. Winsorise or blank out the impossible numeric values listed in §3.6.

---

### Appendix: per-submission flag counts
A record can carry several flags. 7,051 (v1) and 10,088 (v2) records have at least one flag. Most of these are only "open > 24 h", which on its own is weak evidence.

| Flag | v1 | v2 |
|---|---|---|
| negative_duration | 424 | 705 |
| under_5_min | 1,122 | 1,486 |
| open_over_24h | 5,032 | 7,121 |
| IntDate_after_upload | 9 | 24 |
| start_after_upload | 53 | 369 |
| device_clock_wrong | 0 | 2 |
| month_mismatch | 519 | 463 |
| IntDate_differs_from_start_by_>1d | 1,386 | 1,679 |
| gps_precision_over_50m | 271 | 475 |
| content_duplicate | 5 | 2 |
| hh_size_over_30 | 26 | 37 |
