---
name: cronometer-review-nutrition
description: Read and review what is in Cronometer through the cronometer MCP tools (get_daily_nutrition, get_food_log, get_micronutrients, export_raw_csv, sync_cronometer, get_fasting_history, get_fasting_stats, get_recent_biometrics) and manage fasts and biometrics (add_biometric, remove_biometric, delete_fast, cancel_active_fast). Use whenever the user asks how they ate today or this week, whether they hit protein or calories, what a day or meal had in it, which vitamins or minerals they are short on, what they weighed or how a trend moved, how their fasting is going, wants a number logged (weight, body fat, glucose, heart rate), or wants a fast or biometric entry removed, even if they never say "Cronometer".
---

# Review nutrition, fasting and biometrics

Pick the read tool that answers the question. They don't overlap cleanly, and one of them has a known gap.

## Pick the tool

| The question | Use |
|---|---|
| Calories and macros for a day or range, how close to target | `get_daily_nutrition` |
| What was in each meal, which foods, what time | `get_food_log` |
| Vitamins, minerals, gaps over a week | `get_micronutrients` |
| Exercise, notes, whether a day is marked complete, anything the others hide | `export_raw_csv` |
| Files on disk (JSON plus `food-log.md`) for other tools | `sync_cronometer` |
| Fasting history, current fast, fasting totals | `get_fasting_history`, `get_fasting_stats` |
| Recent weight, glucose, heart rate, body fat | `get_recent_biometrics` |
| Targets to compare against | the cronometer-targets skill |
| Log or change food | the cronometer-log-food skill |

All dates are `YYYY-MM-DD`.

## Nutrition reads

**`get_daily_nutrition(start_date, end_date)`** is the source for kcal and macros. It returns one row per day with `macros` (`Energy (kcal)`, `Protein (g)`, `Carbs (g)`, `Fat (g)`, `Fiber (g)`, ...) and `micros`. Defaults: 7 days ago to today. Zero values are left out, so a missing key means 0.

**`get_food_log(start_date, end_date)`** returns every entry as `date`, `time`, `meal` (the diary group), `food`, `amount` (e.g. "100.00 g"), `category`, `macros`, `micros`, grouped by day with `total_calories`, `total_protein`, `total_carbs`, `total_fat`. Default: today only.

- **Its `macros` and `micros` can come back empty with day totals of 0**, while `get_daily_nutrition` has the right numbers (seen on entries logged that day). So take kcal and macros from `get_daily_nutrition`, and use `get_food_log` for the list of foods, amounts and times.
- If you need per-entry nutrition, call `export_raw_csv(export_type="servings", ...)` and read the cells yourself. Report if they are blank too.
- It carries no serving ids, so it can't be used to remove an entry.

**`get_micronutrients(start_date, end_date)`** returns `daily_breakdown` and `period_averages`, from the same daily summary. Caveats:
- Zeros are dropped before averaging, so an average covers only the days that had a non-zero value and overstates a nutrient that was often 0.
- Today and any partly logged day are included. A low day may be an unfinished log; check completeness with `export_raw_csv(export_type="daily_summary")`, whose `Completed` column the other tools hide.
- No reference intakes come with it. Don't call a value "low" without a source, or say you are comparing to a general guideline.

**`export_raw_csv(export_type, start_date, end_date)`**: `export_type` is `servings`, `daily_summary`, `exercises`, `biometrics` or `notes`. Defaults: today. Output over 50,000 characters is cut with `truncated: true` and `total_chars`, so narrow the range. It is the only way to read exercises and notes, and the only dated biometric history.

**`sync_cronometer(start_date, end_date, days=14, diet_label=None)`** writes JSON exports and a `food-log.md` to `~/.local/share/cronometer-mcp/` (or `CRONOMETER_DATA_DIR`) and returns only file paths and counts. Don't use it to answer a question; use it when the user wants the files.

Every read goes through Cronometer's CSV export, which throttles. Make one call for a range, not one per day.

## Fasting

- `get_fasting_history(start_date, end_date)` returns `fasts` with `fast_id`, `recurrence_id`, `name`, `recurrence_rule`, `start_ts`, `end_ts`, `is_active`, plus counts of total, active and completed fasts. **Pass both dates or neither**: with only one, the filter is ignored and the full history comes back.
- `start_ts` and `end_ts` are raw encoded strings, not dates. Don't present them as times; say that durations come from `get_fasting_stats`.
- `is_active` is true when there is no end. A fast from a recurring schedule shows in `recurrence_rule`.
- `get_fasting_stats()` returns `total_hours`, `longest_fast_hours`, `seven_fast_avg_hours`, `completed_count`.

## Biometrics

- `get_recent_biometrics()` takes no dates. Each entry has `biometric_id`, `value`, `metric_id` and `date`, with no metric name, so you can't tell weight from body fat by the entry alone. For a named, dated history use `export_raw_csv(export_type="biometrics", ...)`.
- `add_biometric(metric_type, value, entry_date)`: `metric_type` is `weight` (lbs), `blood_glucose` (mg/dL), `heart_rate` (bpm) or `body_fat` (%). Convert from kg first. It returns `biometric_id`; keep it.

```
add_biometric(metric_type="weight", value=218.5, entry_date="2026-10-01")
```

- **Check every add** with `get_recent_biometrics`: the entry should appear with the right value and date. In the code, `body_fat` sends the same flags and position as `weight`, so a body-fat entry may land as a weight. If it did, `remove_biometric` it and tell the user.
- Other metrics (blood pressure, etc.) can be read but not added.

## Deleting and cancelling

`remove_biometric(biometric_id)`, `delete_fast(fast_id)` and `cancel_active_fast(fast_id)` can't be undone.

**Confirm with the user first**: state which entry (type, value and date, or the fast's name and `fast_id`) and wait for a yes. The one exception is an entry you added yourself a moment ago by mistake.

- `delete_fast` removes the fast's entry.
- `cancel_active_fast` stops an in-progress fast (`is_active` true) and **keeps the recurring schedule**. Use it for "end this fast", and `delete_fast` for "remove it from my history".
- Find ids first with `get_fasting_history` or `get_recent_biometrics`. Don't guess one.

## Report back

- The numbers answered: kcal, P/C/F (and fiber or micros if asked), with dates and the target when known.
- The tool you used, and if `get_food_log` macros were empty, that the totals come from `get_daily_nutrition`.
- Partly logged days or truncated output that affect the answer.
- For writes: what was added or removed and the id.

Keep it short.
