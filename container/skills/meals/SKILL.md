---
name: meals
description: Manage the meal planner — add recipes (from URLs, pasted text, or description), search recipes, manage pantry. Use whenever the user mentions recipes, meals, cooking, groceries, or ingredients.
allowed-tools: Bash(curl:*), Bash(agent-browser:*)
---

# Meal Planner API

The meal planner runs at `http://meals.home:5678`. All API calls use `curl` against this host.

## Adding a recipe from a URL

1. Use `agent-browser` to open the URL and extract the recipe content:
   ```bash
   agent-browser open <url>
   agent-browser snapshot
   ```
2. Parse the page content into the recipe JSON structure below.
3. POST it to the API.

## Adding a recipe from text

When the user pastes or describes a recipe, parse it into the JSON structure and POST it.

## API Reference

Base: `http://meals.home:5678/api`

### Recipes

**List recipes:**
```bash
curl -s 'http://meals.home:5678/api/recipes'
curl -s 'http://meals.home:5678/api/recipes?q=chicken'       # search by title
curl -s 'http://meals.home:5678/api/recipes?tag=italian'      # filter by tag
```

**Get recipe detail:**
```bash
curl -s 'http://meals.home:5678/api/recipes/1'
```

**Create recipe:**
```bash
curl -s -X POST http://meals.home:5678/api/recipes \
  -H 'Content-Type: application/json' \
  -d '{
    "title": "Recipe Name",
    "instructions": "Step 1...\nStep 2...",
    "prep_time_min": 15,
    "cook_time_min": 30,
    "servings": 4,
    "source_url": "https://original-source.com/recipe",
    "tags": ["italian", "quick", "vegetarian"],
    "notes": "Optional notes",
    "image_url": "/images/filename.jpg or https://...",
    "ingredients": [
      {
        "name": "chicken breast",
        "quantity": 1.5,
        "unit": "lb",
        "display_text": "1.5 lb chicken breast, cubed",
        "category": "protein"
      }
    ]
  }'
```

**Update recipe:**
```bash
curl -s -X PUT http://meals.home:5678/api/recipes/1 \
  -H 'Content-Type: application/json' \
  -d '{ ... same fields as create ... }'
```

**Delete recipe:**
```bash
curl -s -X DELETE http://meals.home:5678/api/recipes/1
```

### Field reference

**Recipe fields:**
- `title` (required): Recipe name
- `instructions`: Cooking steps, newline-separated
- `prep_time_min`: Prep time in minutes
- `cook_time_min`: Cook time in minutes
- `servings`: Number of servings
- `source_url`: Original recipe URL (set when importing from a link)
- `source_type`: One of `manual`, `url_import`, `bulk_import`, `telegram`
- `tags`: Array of lowercase tags. Use descriptive tags from this set when applicable: `quick`, `weeknight`, `comfort`, `healthy`, `one-pan`, `batch-cook`, `meal-prep`, `vegetarian`, `spicy`, `salad`, `brunch`, `protein`, `seafood`. Add cuisine tags: `italian`, `mexican`, `indian`, `thai`, `japanese`, `korean`, `mediterranean`, `chinese`, `french`, `american`. Create new tags as needed.
- `notes`: Free-form notes
- `image_url`: Image URL (web URL or local path like `/images/filename.jpg`)

**Ingredient fields:**
- `name` (required): Canonical ingredient name (e.g., "chicken breast" not "boneless skinless chicken breast")
- `quantity`: Numeric amount
- `unit`: Measurement unit (`cup`, `tbsp`, `tsp`, `oz`, `lb`, `each`, `cloves`, `cans`, `heads`, `stalks`, `pint`, `sprigs`)
- `display_text`: Human-readable text when the simple name/qty/unit isn't enough (e.g., "2 medium sweet potatoes, cubed")
- `category`: One of `produce`, `dairy`, `protein`, `pantry`, `frozen`, `spices`, `other`

### Tags

**List all tags:**
```bash
curl -s http://meals.home:5678/api/tags
```

### Pantry

**List pantry items:**
```bash
curl -s http://meals.home:5678/api/pantry
```

**Add pantry item:**
```bash
curl -s -X POST http://meals.home:5678/api/pantry \
  -H 'Content-Type: application/json' \
  -d '{"name": "olive oil", "category": "pantry"}'
```

**Delete pantry item:**
```bash
curl -s -X DELETE http://meals.home:5678/api/pantry/1
```

## Guidelines

- When importing from a URL, always set `source_url` to the original link and `source_type` to `url_import`.
- When adding from Telegram text, set `source_type` to `telegram`.
- Extract as much structure as possible: ingredients with quantities/units, realistic prep/cook times, appropriate tags.
- For ingredient names, use canonical simple names (e.g., "onion" not "1 medium yellow onion, finely diced") — put the detailed description in `display_text`.
- Confirm with the user what you parsed before saving, especially if the source was ambiguous.
