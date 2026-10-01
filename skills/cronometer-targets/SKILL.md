---
name: cronometer-targets
description: Read and change Cronometer macro targets through the cronometer MCP tools (get_macro_targets, set_macro_targets, list_macro_templates, create_macro_template, set_weekly_macro_schedule). Use whenever the user asks what their calorie or protein target is, wants to change a target for today or one date, wants different targets on lifting and rest days, wants a new target set saved and applied to some or all weekdays, or wants Cronometer's targets to match a plan, even if they never say "template" or "schedule".
---

# Cronometer macro targets

Targets live in two layers. Know which one the user means before writing.

| Layer | What it is | Tools |
|---|---|---|
| **Weekly schedule** | A saved **template** assigned to each weekday. The recurring default for all future dates | `create_macro_template`, `set_weekly_macro_schedule` |
| **Per-date override** | One date's targets, changed on their own | `set_macro_targets` |

## Pick the path

| The user wants | Use |
|---|---|
| "What's my protein target?" | `get_macro_targets()` |
| The whole week's targets | `get_macro_targets(target_date="all")` |
| "Bump today to 3000 kcal" or one date | `set_macro_targets` |
| New targets from now on, every day | `create_macro_template(..., assign_to_all_days=True)` |
| Different targets on some weekdays | `create_macro_template` per set, then `set_weekly_macro_schedule` per set |
| Use a template that exists | `list_macro_templates`, then `set_weekly_macro_schedule` |
| How they did against target | the cronometer-review-nutrition skill |

## Read

- `get_macro_targets()` returns `targets` for today: `protein_g`, `fat_g`, `carbs_g`, `calories`, `template_name`. Pass `target_date="2026-10-01"` for another date.
- `target_date="all"` returns `schedules`: seven entries (`day_of_week` 0 = Sunday to 6 = Saturday, `day_name`, the macros, `template_name`, `template_id`). This is the weekly default, not any per-date override.
- To see what applies on a date, call the dated form; a per-date override may make it differ from the weekly form.
- `list_macro_templates()` returns `templates` with `template_id`, `template_name`, `protein_g`, `fat_g`, `carbs_g`, `calories`.

**`carbs_g` is net carbs** everywhere here (the tool descriptions say so). Don't compare it with a total-carbs figure from a label without saying so.

## Change one date

```
set_macro_targets(protein_grams=230, calories=3100, target_date="2026-10-04")
```

- It reads that date's current targets first and changes only what you pass, so `calories=3100` alone keeps the macros.
- `target_date` defaults to today.
- `template_name` only labels the override; the default is `"Custom Targets"`. It does not select a saved template. To apply a saved one, use `set_weekly_macro_schedule`.
- It does not touch the weekly schedule. The result gives `previous` and `updated`; show both.

## Save a set and schedule it

```
create_macro_template(template_name="Lifting Day", protein_grams=230, fat_grams=90,
                      carbs_grams=330, calories=3000)
set_weekly_macro_schedule(template_name="Lifting Day", days="Monday,Wednesday,Friday")
```

- `create_macro_template` refuses a name that already exists and returns the existing template. There is **no tool to edit or delete a template**: to change numbers, create a new name and reassign the days. Say so before creating.
- It returns `template_id`. `assign_to_all_days=True` also sets all seven days to it in the same call.
- **Every template it creates is tagged with the program name "Rigorous"**, hardcoded in the request. The program is not a parameter, so what that label does in Cronometer is unknown. Mention it when you create one.
- When `fat_grams` equals `carbs_grams`, the code builds the request differently (a GWT back-reference). Read the template back with `list_macro_templates` to confirm the values.
- `set_weekly_macro_schedule` needs the **exact** template name (case-sensitive; on a miss it returns `available_templates`). `days` is `"all"` (default) or comma-separated **full day names** (`"Monday,Wednesday"`); `"Mon"` is refused. It sets the default for all **future** dates and returns the full `current_schedule`; it writes only the weekday assignments, so check the dated form for any date that was overridden.
- Days are applied one at a time, so a mid-call failure can leave the week half changed. After any schedule write, re-read with `get_macro_targets(target_date="all")` and report the seven days.

## Check the numbers before writing

Cronometer stores calories separately from the macros and doesn't reconcile them. Compute `4 x protein + 4 x carbs + 9 x fat` and tell the user if it is more than about 5 % from `calories` (net carbs and fiber account for some gap). Ask which to change; don't silently adjust one.

## Matching the fitness-app tracker

The tracker keeps targets per person and **day type** (`meal_list_people`), not per weekday, and nothing syncs them. To mirror them, read the tracker's targets, make one template per day type, and ask the user which weekdays are which type. Don't guess the mapping.

## Report back

- What changed, as before and after: per-date override or weekly schedule, and for which dates or days.
- The seven-day schedule after any schedule write.
- The macro and calorie arithmetic gap, if there is one.
- Any template created, with its `template_id`, and that it cannot be edited or deleted from here.

Keep it short.
