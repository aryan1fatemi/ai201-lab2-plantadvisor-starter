# Spec: Tool Functions

**File:** `tools.py`
**Status:** `get_seasonal_conditions` — Pre-implemented, read through. `lookup_plant` — complete spec fields before implementing.

---

## Purpose

These two functions are the tools the agent can call. They retrieve structured data from the local plant database and seasonal data files and return it to the agent loop, which passes it to the LLM as context for generating a response.

---

## Function 1: `lookup_plant()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `plant_name` | `str` | The plant name as entered by the user or chosen by the LLM — may be any casing, common name, scientific name, or alias |

**Output:** `dict`

When the plant is **found**, return:
```python
{"found": True, "plant": <the full plant dict from _plant_db>}
```

When the plant is **not found**, return:
```python
{"found": False, "name": <normalized input>, "message": <helpful string>}
```

---

### Design Decisions

*Complete the two blank fields below before writing code. The others are pre-filled for you.*

---

#### Input normalization

Strip leading/trailing whitespace and convert to lowercase before any comparison.

```python
normalized = plant_name.strip().lower()
```

---

#### Search order

Search in this order: direct key → display name → aliases. Keys are the fastest
lookup (O(1) dict access), so check those first. Display names are the next most
likely match for clean user input. Aliases are the broadest net, so they go last.

```
1. Direct key match: normalized in _plant_db
2. Display name match: plant["display_name"].lower() == normalized
3. Alias match: normalized in [alias.lower() for alias in plant["aliases"]]
```

---

#### Alias matching approach

For each plant in the database, convert every alias to lowercase and compare against
the normalized input. This is done with a list comprehension:

```
for each plant in _plant_db:
    if normalized in [alias.lower() for alias in plant["aliases"]]:
        return found result

This is O(n * a) where n is the number of plants and a is the average number of aliases
per plant. With 15 plants and ~4 aliases each, this is effectively constant. If the
database grew to thousands of plants, a flat alias-to-key dict built once at module
load would make lookups O(1):

  _alias_index = {
      alias.lower(): key
      for key, plant in _plant_db.items()
      for alias in plant["aliases"]
  }
```

---

#### Not-found message

When a plant isn't found, the agent will read your message and use it to decide what to tell the user:

```
"No plant matching '{plant_name}' was found in the database. The database contains
common houseplants like pothos, monstera, snake plant, ZZ plant, peace lily, aloe vera,
philodendron, calathea, orchid, fiddle leaf fig, rubber plant, boston fern, spider plant,
chinese evergreen, and succulents. Acknowledge that this specific plant is not in the
database. Offer general care advice based on the plant type (tropical, succulent, fern,
etc.) if you can infer it from the name — but do not invent specific data as if it came
from the database."
```

The message is written to the LLM, not to the user. It gives the LLM both the factual
result (not found) and an explicit behavioral instruction (don't fabricate, but do offer
general guidance). This is more useful than a bare "not found" string.

---

#### Implementation Notes

**Test: does `"devil's ivy"` return the pothos entry?**
```
Yes. "devil's ivy" is listed in pothos's aliases array. After normalizing the input to
lowercase and stripping whitespace, the alias match loop finds it and returns
{"found": True, "plant": <pothos dict>}.
```

**Test: does `"SNAKE PLANT"` return the snake plant entry?**
```
Yes. "SNAKE PLANT".strip().lower() → "snake plant", which matches the display_name
"Snake Plant".lower() == "snake plant" in the display name pass.
```

**One edge case you discovered while implementing:**
```
The slug keys in _plant_db use underscores (e.g., "snake_plant", "zz_plant"), but users
and the LLM typically say "snake plant" or "zz plant" with a space. These do NOT match
on the direct key pass, so the display name pass is what catches them. This makes the
display name check more important in practice than the key check for most user inputs.
```

---

## Function 2: `get_seasonal_conditions()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `season` | `str \| None` | One of `"spring"`, `"summer"`, `"fall"`, `"winter"`, or `None` to auto-detect |

**Output:** `dict`

The full season dict from `_season_data`, plus one additional field:

| Added field | Type | Value |
|-------------|------|-------|
| `"detected_season"` | `bool` | `True` if auto-detected from the month; `False` if season was passed as an argument |

---

### Design Decisions

*This function is pre-implemented — read through these fields and the code before working on `lookup_plant`.*

---

#### Auto-detection logic

When `season` is `None`, get the current calendar month with `datetime.now().month`
and look it up in the `_MONTH_TO_SEASON` dict, which maps month numbers to season strings.

```python
current_month = datetime.now().month
season_key = _MONTH_TO_SEASON[current_month]
```

---

#### Season validation

If the caller passes an invalid season string (e.g., `"monsoon"`), the function
falls back to auto-detection — same as if `None` were passed. The `VALID_SEASONS`
set acts as the gate:

```python
VALID_SEASONS = {"spring", "summer", "fall", "winter"}
if season and season.lower() in VALID_SEASONS:
    ...  # use provided season
else:
    ...  # auto-detect
```

---

#### Return structure

The full season dict from `_season_data`, plus a `detected_season` boolean. Example for spring:

```python
{
    "season": "spring",
    "watering": "Increase watering frequency as plants break dormancy ...",
    "fertilizing": "Resume feeding with a balanced fertilizer ...",
    "light": "Days are lengthening — move plants closer to windows ...",
    "pests": "Watch for spider mites and aphids as temperatures rise ...",
    "detected_season": True   # True = auto-detected; False = caller specified
}
```

---

#### Implementation Notes

**Test: does calling with `season=None` return the correct season for the current month?**
```
Current month: June (month 6)
Expected season: summer
Returned season: summer ✓ — detected_season=True confirms auto-detection was used
```

**Test: does calling with `season="winter"` return winter data regardless of the current month?**
```
Yes. Passing season="winter" skips auto-detection entirely and returns the winter entry
from _season_data with detected_season=False.
```
