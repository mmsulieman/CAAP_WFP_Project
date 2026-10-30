# Skip-logic fixes – Exception Update Version 2 XLSForm

Fixes the 27 dead skip-logic conditions listed in §2.2 of [the Q3 2026 review](Q3_2026_Process_Monitoring_Review.md). They were applied to `Ethiopia_Process_Monitoring_One_Checklist_All_Activities_Third_Quarter_2026_Exception_Version_2.xlsx`.

The fixed form is **not stored in this repository**. This repository is public, and the form's choice lists contain staff names and site lists.

**Validation:** the fixed form converts with pyxform 4.5 and passes ODK Validate, with the same warnings as before. Every `${q}='x'` and `selected(${q},'x')` literal in the form now matches a code in the compiled choice list of `q`.

Row numbers below are from the **original** v2 file. The fixed file has one extra row at 307, so rows from 307 down shift by +1.

## Survey sheet: `relevant` changes

| Row | Question | Before | After | Why |
|---|---|---|---|---|
| 52 | `HHAsstCashMode` | `…'2') or …'5') or …'6')` | `selected(${HHAsstMode}, '2')` | `AsstModality` has only 1/2/999 |
| 98 | `HHAsstDMEntCBT` | same as above | `selected(${HHAsstMode}, '2')` | as above |
| 99 | `HHAsstCBTRec` | `…'1') and (…'2') or …'6'))` | `selected(${HHAsstDMEntCBT}, '1') and selected(${HHAsstMode}, '2')` | as above |
| 100 | `HHAsstCBTExp` | `…'2') or …'6')` | `selected(${HHAsstMode}, '2')` | as above |
| 182, 183, 186, 191, 193 | `HHAsstPartNb`, `HHDTPTranspHow`, `HHDTPTranspHow1`, `HHAsstTrvlDay`, `HHDTPDelay` | `…'1') or …'2') or …'5') or …'6')` | `selected(${HHAsstMode}, '1') or selected(${HHAsstMode}, '2')` | as above |
| 241 | `HHAsstAccessCBTPrb` | `…'1') and (…'2') or …'5') or …'6'))` | `selected(${HHAsstAccess}, '1') and selected(${HHAsstMode}, '2')` | as above |
| 320 | `MOAsstIKRed` | `selected(${HHAsstMode},'1') or selected(${HHAsstMode},'5') and (Act 1 or Act 3)` | `selected(${MOAsstModeObs}, '1') and (Act 1 or Act 3)` | `HHAsstMode` is not asked in this module, so the question never showed. The missing parentheses also dropped the activity filter. |
| 321 | `MOAsstCBTRed` | `selected(${HHAsstMode},'2') or selected(${HHAsstMode},'6') and (Act 1 or Act 3)` | `selected(${MOAsstModeObs}, '2') and (Act 1 or Act 3)` | as above |
| 824 | `oth_rowfood` | `selected(${C09.2},'99')` | `selected(${C09.2},'other')` | `C09.2` uses `or other`, whose code is `other` |
| 838 | `othnotgood` | `selected(${C09.2},'99')` | `selected(${C09.2},'other')` | as above |
| 1400 | `FFV_Sell2` | `${FFV_Sell1} = 'yes'` | `${FFV_Sell1} = 1` | `Yesno` codes Yes as `1` |
| 2047 | `garden_support_other` | `selected(…, 'other_garden_support')` | `selected(${garden_support_types}, '99')` | the "Other" code is `99` |

Unchanged expressions, fixed through their choice lists instead:
- row 350 `HHAsstMode_oth1`
- row 661 `_1_1_If_yes_who_informed_you`
- row 1687 `MOWhWhatComm_oth`

## Survey sheet: structural changes

- **New row 307: `MOAsstModeObs`** (`select_multiple AsstModalityObs`, required). It asks "What type of assistance is being distributed at this site?" and is shown when `${DMSvyMode} = '2'`. It is the first question of `DistributionCond_submodule`, and it drives rows 320–321.
- **Row 1686 `MOWhWhatComm`:** the type changed from `select_multiple CommoditiesDM` to `select_multiple CommoditiesDM_Wh`. This list adds an `Other` option without touching the 10 other questions that share `CommoditiesDM`. Eight of those already use `or other` and would otherwise show two "Other" options.

## Choices sheet

| List | Change |
|---|---|
| `pu3kq07` | Code `AUTOMATIC` (label "Yes") renamed to `yes` |
| `AsstModality1` | Added `999` Other |
| `AsstModalityObs` | New list: `1` Food, `2` Cash |
| `CommoditiesDM_Wh` | New list: a copy of `CommoditiesDM` plus `999` Other, placed before 777/888 |

## Impact on data

- **Labels:** exports by label are unaffected; "Yes" stays "Yes".
- **XML values:** in XML-value exports, `Were_you_informed_in_advance_a` changes from `AUTOMATIC` to `yes` for new records. Recode `AUTOMATIC` → `yes` when you merge with Q3 data.
- **New fields:** `MOAsstModeObs`, `MOAsstIKRed`, `MOAsstCBTRed`, `_1_1_If_yes_who_informed_you`, `FFV_Sell2`, `garden_support_other`, `HHAsstMode_oth1`, `MOWhWhatComm_oth`, `oth_rowfood` and `othnotgood` start collecting data from this version onward. There is no Q3 data for them.
- **Fewer columns:** `selected(…,'5'/'6')` never matched anything, so removing those clauses changes no behaviour. It only removes the dead clauses.

## Deployment

`settings.version` is still empty, as in the original. Set a version greater than the deployed `202608121315` (for example `202611010900`) before replacing the form on MoDa.
