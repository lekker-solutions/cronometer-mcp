---
name: cronometer-create-recipe
description: Create or delete a recipe in Cronometer through the cronometer MCP tools (create_recipe, delete_recipe, search_foods). Use whenever the user wants a recipe, a meal-prep batch, a shake or a sauce built in Cronometer from an ingredient list or weights, wants a fitness-app (tracker) recipe copied into Cronometer so it can be logged there, wants a wrong or duplicate Cronometer recipe removed, or asks how to log a batch they cooked, even if they never say "create_recipe".
---

# Create a Cronometer recipe

`create_recipe` builds a recipe from ingredients **in grams**. The server computes nothing: the tool fetches each ingredient, sums its per-100 g nutrients and saves the total. Your job is the inputs: the right food for each line, and the right grams.

## Pick the path

| The user gives you | Use |
|---|---|
| A typed or pasted ingredient list with weights | resolve each line, then `create_recipe` |
| A fitness-app recipe name | `meal_get_recipe` (fitness MCP), convert to grams, then `create_recipe`; see below |
| "That recipe is wrong, delete it" | `delete_recipe(food_source_id)` |
| "Log 300 g of it" | the cronometer-log-food skill |
| A Cronometer recipe into the tracker | fitness-app `meal_import_cronometer`, the reverse direction (see the meal-add-recipe skill) |

## 1. Resolve each ingredient

Each ingredient is `{"food_source_id": int, "grams": float}` or `{"query": str, "grams": float}`.

- **Use `food_source_id` for anything that could be ambiguous.** Run `search_foods(query=...)` and pick the hit as in cronometer-log-food: prefer `NCCDB` or `USDA` unless the user names a brand, and note raw vs cooked.
- A `query` takes the **top search hit with no source preference**, which can be a branded or user-made entry. Use it only for plain items ("olive oil", "salt").
- `grams` is the weight for the **whole recipe**. There is no servings parameter: the saved recipe has the measures "full recipe" and "g" (total weight), plus one named after the account email that the app adds.
- Volumes and counts ("2 cups", "3 cloves") have to become grams first. Use `get_food_details(food_source_id)` for the food's measure `weight_grams` (not reliable for recipes), or ask. Don't guess a density.
- "To taste" lines are 0 g: leave them out.

## 2. Create it

```
create_recipe(name="SE Asia - Peanut Sauce", notes="Source: tracker",
  ingredients=[
    {"food_source_id": 1055762, "grams": 190},
    {"query": "garlic", "grams": 9}])
```

- **Name in the house style:** `Cuisine - Dish` (`SE Asia - Peanut Sauce`, `Indian - Elk Keema`), or a plain name for staples (`Egg Bake`).
- The code has no duplicate check, so `search_foods(query=<name>)` first. A second call with the same name makes a second recipe.
- The result: `recipe` (`name`, `food_source_id`, `total_grams`, `kcal`, `protein_g`, `carbs_g`, `fat_g`, all for the whole recipe) and `ingredients`.

### Check the matches

For every ingredient given as a `query`, read `matched_name` and `matched_source` in the result. A wrong match is silent otherwise (a branded "garlic powder" for fresh garlic). Ingredients given as `food_source_id` have no `matched_name`; you chose them, so recheck the id against your search.

If one is wrong:
1. `delete_recipe(food_source_id=<the new recipe's id>)`. This is the recipe you just made, so no confirmation is needed. Anything else needs the user's say-so.
2. Run `create_recipe` again with an explicit `food_source_id` for that line.

Don't leave the first recipe behind.

### Check the numbers

`kcal` is the sum of each ingredient's per-100 g value x grams / 100, and matches the web app. Compare it, and P/C/F, to what the user expects (a label, their app, their own total). A gap means a wrong food or wrong grams. Say which line is likely.

## From a fitness-app recipe

1. `meal_get_recipe(name=...)` returns `lines` (each with `ingredient`, `qty`, `unit`, `retained`, `grams`), `batch_g`, `batch_macros` and `gaps`.
2. Each line becomes one Cronometer ingredient:
   - Grams are the line's `grams`. That is the full amount; for a line with `retained` below 1 (a marinade mostly poured off), use `grams x retained` so Cronometer matches the tracker's `batch_macros`. Say you did.
   - `grams: null` ("missing unit conversion") has no weight. Stop and fix it in the tracker, or ask the user.
   - `qty: 0` lines ("to taste") are left out.
   - Find the Cronometer food with `search_foods`. The tracker's ingredient name is only a hint; `meal_get_ingredient` shows its `source`, which may name the food.
3. `create_recipe`, then compare `recipe.kcal`, `protein_g`, `carbs_g`, `fat_g` with `batch_macros.kcal`, `protein_g`, `carbs_g`, `fat_g`. **Flag any gap over 5 %**, per macro and kcal, and name the likely line. A small kcal gap can be normal: the tracker derives kcal from P/C/F, while Cronometer sums its own kcal values.
4. Portions differ. Cronometer's "g" total (`total_grams`) is the sum of the **raw** ingredient grams, while the tracker's `batch_g` is the finished weight (`cooked_batch_g` or raw plus adjustment). To log a cooked portion, scale: `grams in Cronometer = eaten_g / batch_g x total_grams`. Tell the user the conversion when logging.

A `serving` recipe in the tracker (a shake, a bar) is 1 serving: log the whole `total_grams`.

## Report back

- The new recipe: name, `food_source_id`, `total_grams`, kcal and P/C/F.
- For queries: each `matched_name` and `matched_source`, and any you replaced.
- For a tracker recipe: the macro comparison with percentages, and any line you scaled by `retained` or left out.
- Whether a wrong recipe was deleted.
- Every assumption: grams converted from a volume, generic vs branded pick, raw vs cooked.

Keep it short.
