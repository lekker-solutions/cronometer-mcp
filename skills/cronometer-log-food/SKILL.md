---
name: cronometer-log-food
description: Log food to the Cronometer diary through the cronometer MCP tools (search_foods, add_food_entry, remove_food_entry, copy_day, add_repeat_item, set_day_complete). Use whenever the user says they ate or drank something, wants a meal, a portion of a recipe or a weighed amount put in Cronometer, wants a logged entry undone or moved, wants yesterday's meals copied to today, wants a daily item (coffee, a supplement) to log itself, or wants a day marked complete, even if they never say "Cronometer" or "diary".
---

# Log food to Cronometer

Log by **grams**. Find the food, pick the right match, call `add_food_entry`, then check the day's totals moved. Don't guess ids: every id comes from `search_foods`.

## Pick the path

| The user wants | Use |
|---|---|
| "I had 150 g of chicken breast" | `search_foods`, then `add_food_entry` |
| A portion of a recipe they made | Same. The recipe is a search hit with `type` RECIPE, `source` Custom |
| Meals in a named slot (Lunch, Pre-workout) | `list_diary_groups`, then `diary_group` on the add |
| Take back an entry | `remove_food_entry` with its `serving_id` |
| "Same as yesterday" | `copy_day`; see below |
| A daily item that logs itself | `add_repeat_item`; see below |
| "Close out the day" | `set_day_complete` |
| A recipe that does not exist yet | the cronometer-create-recipe skill |
| What they ate, totals, targets | the cronometer-review-nutrition skill |

## 1. Find the food

`search_foods(query="chicken breast")` returns `food_id`, `food_source_id`, `name`, `measure_desc`, `score`, `source` and `type`. Up to 50 hits, best first.

- **Prefer `NCCDB` or `USDA`** generic entries unless the user names a brand. `FDCBranded` and `CRDB` hits are specific branded products: use them when the user names that product.
- A recipe the user built is `type` RECIPE, `source` Custom. Search for its exact name; if two hits share a name, ask which.
- Raw vs cooked changes the macros a lot (chicken, rice, pasta). If the user did not say, ask or state the assumption in the report.
- Nothing matches: shorten the query (`"oats"` not `"rolled oats dry organic"`), then ask. Don't log a loose substitute silently.

## 2. Add the entry

```
add_food_entry(food_id=461776, food_source_id=1055762, weight_grams=150,
               date="2026-10-01", diary_group="Lunch")
```

- `date` is required (`YYYY-MM-DD`). There is no default to today.
- **Leave `measure_id` out.** The tool looks up the food's own "g" measure and logs `weight_grams` in it. This works for regular foods and for recipes (100 g of a recipe and 100 g of oats both show "100.00 g" in the diary). Don't pass `measure_id` or `quantity` unless the user wants another unit, such as "2 large"; then take the id from `get_food_details`.
- A recipe is the same call, with its `food_id` and `food_source_id` from `search_foods`. For the whole batch, log the recipe's total grams. Logging by "full recipe" with `measure_id` is untested.
- Different weights of one food are separate calls.
- The result gives `entry.serving_id`. **Keep it**: it is the only handle for `remove_food_entry`, and no read tool returns serving ids later.

### Diary groups

Group names are user-configured. They are not always Breakfast/Lunch/Dinner/Snacks, and the account may use all 8 slots.

- Call `list_diary_groups()` once per conversation before the first write. It returns `wire_index`, `name` and `enabled` for each slot.
- Pass `diary_group` as a name (case-insensitive) or the 0-based `wire_index`. Disabled groups, unknown names and out-of-range indices are refused with the real group list; use that list to fix the call.
- If you omit `diary_group`, the default comes from the **current clock**: before 11:00 a group named "breakfast", before 16:00 "lunch", before 21:00 "dinner", else "snacks". That looks only at enabled groups with exactly those names, and errors if the account has none. It ignores the `date` you pass, so logging yesterday's dinner at 9 a.m. goes to Breakfast. **Pass `diary_group` whenever the meal is not the one for the current time, or the date is not today.**

### Check it worked

`add_food_entry` returns only the ids, not the macros. Confirm with the change in `get_daily_nutrition(start_date=d, end_date=d)` totals (`Energy (kcal)` under `macros`). Read once after a batch of writes, not after each one: the read tools all go through the CSV export, which throttles after bursts of writes.

Don't verify with `get_food_log`: its `macros` can come back empty. See cronometer-review-nutrition.

## Undo and move

- `remove_food_entry(serving_id="D80lp$")` removes one entry. Without the id you cannot remove an entry through this server; say so and point the user to the app.
- To move an entry to another group or day, remove it and add it again with the new `diary_group` or `date`. Check the totals after.

## Copy a day

`copy_day(source_date="2026-09-30", destination_date="2026-10-01")` copies **all** entries (food, exercise, notes, biometrics) and is additive. Running it twice doubles the day. Check the destination first with `get_daily_nutrition`; if it already has food, say so before copying. Verify with the change in totals.

## Repeat items

A repeat item logs itself on chosen weekdays.

```
get_repeated_items()
add_repeat_item(food_id=5828, food_source_id=16104, quantity=2, food_name="Coffee",
                diary_group="Breakfast", days_of_week="weekdays")
delete_repeat_item(repeat_item_id=658545)
```

- `quantity` is in the food's **default serving**, not grams: 12 for a coffee whose default serving is 1 cup means 12 cups. Read `measure_desc` from `search_foods` ("1 cup - 240g") and convert; confirm the number with the user.
- `days_of_week` is `"all"` (default), `"weekdays"`, `"weekends"`, or comma-separated numbers `0` (Sun) to `6` (Sat).
- `get_repeated_items` returns `repeat_item_id`, `food_name`, `food_source_id`, `food_id`, `quantity`, `days_of_week`. It does not return the diary group. Days that GWT back-references come back as `null`.
- Check `get_repeated_items` before adding, so you don't create a duplicate. Delete only an item the user named.

## Complete a day

`set_day_complete(date="2026-10-01")`; pass `complete=False` to reopen it. Do it only when the user says the day is finished.

## Report back

- Each entry logged: food name and source, grams, group, date, `serving_id`.
- The day's kcal and P/C/F from `get_daily_nutrition`, and the change the add caused.
- Every assumption: raw vs cooked, a branded vs generic pick, a repeat `quantity` conversion.
- Anything refused (a bad group name, no match) and what you did about it.

Keep it short.
