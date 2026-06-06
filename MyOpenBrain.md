# My Open Brain — MCP Tools Reference

A reference for all tools available across your connected Open Brain MCP servers.

---

## Open Brain (`open-brain`)

Your personal memory layer. Captures, searches, and retrieves thoughts semantically across any AI client.

| Tool | What It Does |
|------|-------------|
| `capture_thought` | Save a thought to your brain — notes, decisions, insights, ideas. Auto-generates an embedding and extracts metadata (type, topics, people, action items). |
| `search_thoughts` | Semantic search over everything you've captured. Finds matches by meaning, not just keywords. |
| `list_thoughts` | Browse recently captured thoughts. Filter by type (`observation`, `task`, `idea`, `reference`, `person_note`), topic, person, or number of days back. |
| `thought_stats` | Summary of your brain: total count, breakdown by type, top topics, and most-mentioned people. |

**Key parameters:**
- `search_thoughts`: `query` (required), `limit` (default 10), `threshold` (default 0.5 — lower = wider net)
- `list_thoughts`: `type`, `topic`, `person`, `days`, `limit`
- `capture_thought`: `content` (required) — write it as a standalone statement that will make sense when retrieved later

---

## Family Calendar (`claude.ai Family Calendar`)

Schedule management for your household — activities, recurring events, important dates, and family members.

| Tool | What It Does |
|------|-------------|
| `add_activity` | Schedule a one-time or recurring activity (sports, medical, school, social, etc.) for a family member or the whole family. |
| `add_family_member` | Add a person to your household roster with name, relationship, and birth date. |
| `add_important_date` | Save a date to remember — birthdays, anniversaries, deadlines. Supports yearly recurrence and advance reminders. |
| `get_upcoming_dates` | See all important dates coming up in the next N days (default 30). |
| `get_week_schedule` | View all activities for a given week, grouped by day. Filter by family member. |
| `search_activities` | Search activities by title, activity type, or family member name. |

**Key parameters:**
- `add_activity`: `title` (required), `start_date`, `start_time`, `end_time`, `activity_type`, `day_of_week` (for recurring), `family_member_id`, `location`, `notes`
- `add_important_date`: `title` + `date_value` (required), `recurring_yearly`, `reminder_days_before`
- `get_week_schedule`: `week_start` (required, Monday YYYY-MM-DD), `family_member_id`

---

## Home Maintenance (`claude.ai Home Maintenance`)

Track recurring and one-time home maintenance tasks, log completed work, and stay ahead of what's due.

| Tool | What It Does |
|------|-------------|
| `add_maintenance_task` | Create a new maintenance task — recurring (e.g. every 90 days) or one-time. Assign category, priority, and due date. |
| `log_maintenance` | Record that a task was completed. Updates `last_completed` and calculates the next due date automatically. |
| `get_upcoming_maintenance` | List all tasks due in the next N days (default 30). |
| `search_maintenance_history` | Search past maintenance logs by task name, category, or date range. |

**Key parameters:**
- `add_maintenance_task`: `name` (required), `category` (hvac, plumbing, exterior, appliance, landscaping), `frequency_days` (null = one-time), `next_due`, `priority` (low/medium/high/urgent)
- `log_maintenance`: `task_id` (required), `performed_by`, `cost`, `notes`, `next_action`, `completed_at`
- `get_upcoming_maintenance`: `days_ahead`

---

## Meal Planning (`claude.ai Meal Planning`)

Build a recipe library, plan weekly meals, and generate shopping lists automatically.

| Tool | What It Does |
|------|-------------|
| `add_recipe` | Save a recipe with ingredients, instructions, prep/cook time, servings, cuisine, tags, and rating. |
| `update_recipe` | Edit an existing recipe by ID. |
| `search_recipes` | Find recipes by name, cuisine, tag, or ingredient. |
| `create_meal_plan` | Assign recipes (or custom meals) to days and meal types for a given week. |
| `get_meal_plan` | View the full meal plan for a given week. |
| `generate_shopping_list` | Auto-aggregate ingredients across all recipes in a week's meal plan into a shopping list. |

**Key parameters:**
- `add_recipe`: `name` + `ingredients` + `instructions` (required); ingredients format: `[{name, quantity, unit}]`
- `create_meal_plan`: `week_start` (required, Monday YYYY-MM-DD), `meals` array: `[{day_of_week, meal_type, recipe_id?, custom_meal?, servings?, notes?}]`
- `generate_shopping_list`: `week_start` (required) — must have a meal plan created first

---

## Household Knowledge (`claude.ai Household Knowledge`)

A knowledge base for your home — paint colors, appliances, measurements, documents, and service vendors.

| Tool | What It Does |
|------|-------------|
| `add_household_item` | Save anything about your home: paint colors, appliance model numbers, room measurements, documents, warranties, etc. |
| `get_item_details` | Retrieve full details of a specific item by its ID. |
| `search_household_items` | Search items by name, category, or location in the home. |
| `add_vendor` | Save a service provider (plumber, electrician, landscaper, etc.) with contact info and rating. |
| `list_vendors` | List all saved vendors, optionally filtered by service type. |

**Key parameters:**
- `add_household_item`: `name` (required), `category` (paint, appliance, measurement, document), `location`, `details` (flexible JSON string for structured metadata), `notes`
- `add_vendor`: `name` (required), `service_type`, `phone`, `email`, `website`, `rating` (1–5), `last_used`, `notes`
- `search_household_items`: `query`, `category`, `location`

---

## Quick Reference

| Want to... | Use |
|-----------|-----|
| Remember something for later | `open-brain` → `capture_thought` |
| Find a note you saved before | `open-brain` → `search_thoughts` |
| See what's on the family schedule | `family-calendar` → `get_week_schedule` |
| Check upcoming birthdays/events | `family-calendar` → `get_upcoming_dates` |
| Log a maintenance task as done | `home-maintenance` → `log_maintenance` |
| See what home tasks are overdue | `home-maintenance` → `get_upcoming_maintenance` |
| Find a recipe by ingredient | `meal-planning` → `search_recipes` |
| Build a grocery list for the week | `meal-planning` → `generate_shopping_list` |
| Look up a paint color or appliance | `household-knowledge` → `search_household_items` |
| Find a contractor's phone number | `household-knowledge` → `list_vendors` |
