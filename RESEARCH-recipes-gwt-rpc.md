# Cronometer GWT-RPC Research: Creating and Deleting Recipes

**Source:** one browser capture of the web app saving a recipe by hand (HAR, kept out of the repo: it holds a live session cookie).
**Recipe saved:** name `asdfaefafe`, notes `afeafeaef`, ingredients 100 g oats (`464877`) and 50 g egg (`464674`). Servings were not touched.
**Redaction:** nonce -> `{nonce}`, account email -> `{email}`, user id -> `{user_id}`, GWT strong name -> `{gwt_header}`. The redacted bodies are `tests/fixtures/`.

---

## `addFood` (creates the recipe)

```
addFood(String nonce, int userId, Food food, IngredientSubstitutions subs)
```

Request layout: `7|0|{table size}|{string table}|{data}|`. The string table is 1-based and filled in the order each string or type name is first written, so equal strings share one slot. The data starts `1|2|3|4` (module, strong name, service, method), then `4` (parameter count) and the four parameter type refs `5|6|7|8`, then the values.

### String table (36 entries)

| # | Value | # | Value |
|---|-------|---|-------|
| 1 | `https://cronometer.com/cronometer/` | 19 | `{email}` |
| 2 | `{gwt_header}` | 20 | `g` |
| 3 | `...rpc.CronometerService` | 21 | `...models.NutrientMap` |
| 4 | `addFood` | 22 | `...models.NutrientMap$NutrientFilter` |
| 5 | `java.lang.String` | 23 | `java.lang.Integer` |
| 6 | `I` | 24 | `...models.Nutrient` |
| 7 | `...models.Food` | 25 | `...models.Nutrient$Type` |
| 8 | `...models.IngredientSubstitutions` | 26 | `advancedServingSize` |
| 9 | `{nonce}` | 27 | `false` |
| 10 | `java.util.ArrayList` | 28 | `Custom` |
| 11 | `afeafeaef` (notes) | 29 | `java.util.HashSet` |
| 12 | `...models.Ingredient` | 30 | `...models.Translation` |
| 13 | `...foods.NutritionLabelType` | 31 | `...user.models.Language` |
| 14 | `...models.FoodMeasures` | 32 | `en` |
| 15 | `...models.Measure` | 33 | `English` |
| 16 | `full recipe` | 34 | `https://cdn1.cronometer.com/media/flags/us.png` |
| 17 | `java.util.HashMap` | 35 | `asdfaefafe` (name) |
| 18 | `...models.Measure$Type` | 36 | `...foods.FoodType` |

### Data tokens, in order

Parameters 1 and 2:

| Token | Meaning |
|-------|---------|
| `9` | param 1, `nonce`: string ref |
| `{user_id}` | param 2, `userId`: int |

Parameter 3, the `Food` (`7` = object, type ref). Field names are inferred from position and from the matching `getFood` response; the wire carries none.

| Tokens | Meaning |
|--------|---------|
| `7` | `Food` object |
| `0`, `0` | two zero fields (unknown) |
| `10`, `0` | `ArrayList`, size 0: the barcode list (the `getFood` response for the egg carries two barcode strings here) |
| `0` | zero field (unknown) |
| `11` | notes: string ref (`0` = null; the client sends `0` when notes are empty, which the capture does not show) |
| `0`, `0` | two zero fields (unknown) |
| `0` | Food id: 0 = new. The response fills it in. |
| `10`, `2` | `ArrayList` of ingredients, size 2 |
| `12`,`100`,`464877`,`A`,`1073268`,`0`,`0` | `Ingredient`: type, grams, food id (what `getFood` takes), long id `A` = 0 = new, measure id, two zero fields |
| `12`,`50`,`464674`,`A`,`1072101`,`0`,`0` | second `Ingredient` |
| `13`,`1` | `NutritionLabelType` enum, ordinal 1 |
| `A` | a long, 0 |
| `14`,`0` | `FoodMeasures`; its default measure id (0 = new) |
| `10`,`3` | `ArrayList` of measures, size 3 |

Measure layout: `15`(type) `1`(quantity) `0` `0`(food id) `0` `0`(measure id) `0` `{desc}` `17`,`0`(empty HashMap) `{Measure$Type}` `{weight}`.

| Tokens | Measure |
|--------|---------|
| `15`,`1`,`0`,`0`,`0`,`0`,`0`,`16`,`17`,`0`,`18`,`3`,`1` | `full recipe`; first `Measure$Type` (type ref 18, ordinal 3); last token `1` |
| `15`,`1`,`0`,`0`,`0`,`0`,`0`,`19`,`17`,`0`,`-11`,`1` | named `{email}`; `-11` is a back-reference to the `Measure$Type` object; last token `1` |
| `15`,`1`,`0`,`0`,`0`,`0`,`0`,`20`,`17`,`0`,`-11`,`150` | `g`; last token `150` = total grams |

The last token is the measure's weight field, as in `getFood` responses. For `g` it is the recipe's total weight, not 1. The meaning of "full recipe" and the email-named measure both being `1` is not understood. The server accepted the capture and echoed the same values back.

Nutrient map:

| Tokens | Meaning |
|--------|---------|
| `21` | `NutrientMap` |
| `22`,`0` | `NutrientFilter` enum, ordinal 0 |
| `17`,`93` | `HashMap`, 93 entries |
| `23`,`-1205`,`24`,`58.8155`,`-1205`,`25`,`0` | entry 1: `Integer` key, `Nutrient` {amount, id}, then the `Nutrient$Type` object (type ref 25, ordinal 0) |
| `23`,`203`,`24`,`19.79`,`203`,`-21` | entry 2 onwards: the same, with `-21` back-referencing the `Nutrient$Type` |
| ... 91 more entries ... | |

Tail:

| Tokens | Meaning |
|--------|---------|
| `17`,`1`,`5`,`26`,`5`,`27` | `HashMap` of one entry, String -> String: `advancedServingSize` -> `false` |
| `0` | zero field |
| `28` | source: `Custom` |
| `29`,`0` | `HashSet`, size 0 (tags) |
| `10`,`1` | `ArrayList` of translations, size 1 |
| `30`,`31`,`32`,`33`,`34`,`33`,`35`,`0` | `Translation`: `Language` {`en`, `English`, flag url, `English`}, then the name (`asdfaefafe`) and a zero field. The name appears once. |
| `36`,`1` | `FoodType` enum, ordinal 1. Assumed to mean recipe; the capture has no other value to compare with. |
| `{user_id}` | owner id |
| `0` | param 4, `IngredientSubstitutions`: null |

### Back-references

GWT numbers every object (not strings) in the order it is first written. A negative token `-k` refers to object `k`.

With `n` ingredients the `Measure$Type` is object `9 + n` and the first `Nutrient$Type` is object `19 + n`. For `n` = 2 those are `-11` and `-21`.

- Food, the barcode list and the ingredient list are objects 1-3.
- The `n` ingredients are 4 to `3 + n`.
- Then NutritionLabelType, FoodMeasures, the measure list, the "full recipe" Measure, its HashMap and `Measure$Type`. That makes `Measure$Type` object `9 + n`.
- Then two more Measures with a HashMap each, `NutrientMap`, `NutrientFilter` and its HashMap.
- Then entry 1's `Integer`, `Nutrient` and `Nutrient$Type`, which is object `19 + n`.

### Nutrient values

Each value is the sum over ingredients of `per-100g amount * 0.01 * grams`, evaluated in that order. `value * grams / 100` and `value * (grams / 100)` differ in the last digit for 5 and 3 of the 93 values; `value * 0.01 * grams` matches all 93 exactly. Whole numbers are written without `.0` (`680`, not `680.0`); other doubles are written as their shortest round-trip form (`0.9900000000000001`).

- Check: kcal (id 208) = 379 * 1 + 155 * 0.5 = 456.5.
- The per-100 g amounts come from each ingredient's `getFood` response, which carries 95 nutrient entries.
- The app sent 93. It left out ids `318`, `325` and `326`, which `getFood` carries for both foods. Other foods carry more ids than that (up to 151 in other captured responses).
- The ids are sent in a fixed non-numeric order: `-1205`, then `203` upwards, then `-203`, `-205`, `-204`, `-221`, then `10005`. This looks like the web app's own nutrient list. The code reproduces the order from a list of the 93 ids and drops everything else.
- Ids missing from an ingredient's `getFood` count as 0.

### Ingredient measure id

The measure id in each `Ingredient` is the food's **default measure id**, not necessarily its "g" measure: 1073268 for oats (which is its `g`), 1072101 for egg (`large`; the egg's `g` is 1072109). The amount in the `Ingredient` is still grams (egg added 50 * 0.01 * 155 kcal).

In a `getFood` response that id is the token just before the `FoodMeasures` type ref, which is followed by a quoted base64 long.

---

## `addFood` response

```
//OK[{user_id},1,29,93343118,28,26,...,"[string table]",0,7]
```

The response echoes the saved `Food`. GWT writes responses in reverse, so the data section is read from the end. The tail of the data section is:

```
...,2,2,84011882,0,0,3,0,0,2,0,0,1
```

Read from the end: `1` (Food type ref), `0`, `0`, `2` (barcode list), `0` (its size), `0`, `3` (notes), `0`, `0`, `84011882`. The tenth token from the end is the new food id, **84011882**. It is the same id `findMyFoods` then lists for the recipe and `getFood` takes.

The `getFood` responses for oats and egg do not put their id there: the first field is a different id (73101340 and 69011855) and the food id comes later. The recipe response has `0` in that first field. The tenth-from-end rule is established for the recipe response only.

The response also carries the three new measure ids (the server's ids for "full recipe", the email-named measure and "g") and the recipe's nutrient map in the server's own hash order.

---

## `deleteFood` (not captured)

Read from the live bundle: `deleteFood(String nonce, int foodId)`. Same shape as `getFood`:

```
7|0|7|https://cronometer.com/cronometer/|{gwt_header}|com.cronometer.shared.rpc.CronometerService|deleteFood|java.lang.String/2004016611|I|{nonce}|1|2|3|4|2|5|6|7|{food_id}|
```

Not run against a live account yet. The response format is unknown, so `delete_recipe` accepts any `//OK` response.

---

## `getFood` response fields used by `add_recipe`

Layout of an entry in the `getFood` data section (GWT writes responses in reverse):

- **Nutrient entry:** `{Nutrient$Type ref or back-ref}, {id}, {amount per 100 g}, {Nutrient ref}, {id}, {Integer ref}`. The amount is a double and always has a decimal point.
- **Measure:** the `Measure` type ref at `i`, then: `i-1` quantity `1.0`, `i-3` food id, `i-5` measure id, `i-7` description ref. The existing `measures` parser reads the measure id at `i-4`, which is the `0` beside it, so `get_food_details` reports `measure_id` 0 for these responses. Left as it was.
