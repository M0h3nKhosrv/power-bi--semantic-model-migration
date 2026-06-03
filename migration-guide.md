# Power BI Semantic Model Migration — Process Guide

> **Purpose:** Universal step-by-step process for migrating a Power BI report
> from any legacy AS semantic model to any new target AS semantic model.
> The process is model-agnostic. Target-specific knowledge lives in
> `models/<target-model>/model-reference.md`.
> Updated: 2026-05-29

---

## Step 1 — Explore Source Report

Connect to the source Power BI Desktop instance and document the following
in an MD file:

- Tables connected to the legacy semantic model(s) — names, columns, and
  measures; distinguish EXTERNALMEASURE pass-throughs from locally-defined DAX
- All locally-defined measures — table location, DAX, and dependency chain
- Tables from other sources (SQL Server, Import, DirectQuery non-AS, etc.) —
  names, columns, and measures
- All relationships — between legacy tables, between non-legacy tables, and
  any cross-source relationships; note direction and active/inactive state

**Output:** `Step1_Source_Model.md`

**Pause and verify** the output before proceeding.

---

## Step 2 — Parse Report Layout (Optional)

Ask the user to share the Layout JSON file from their `.pbix` (extract by
opening the `.pbix` as a zip archive and locating `Report/Layout`).

If provided, parse the Layout JSON and append to the Step 1 MD file:

- Each page name and its visuals (type and title where available)
- For each visual: measures and columns bound to it (values, axes, rows,
  columns, legends, tooltips, etc.)
- Filters at report, page, and visual level
- For each field referenced: which table it comes from and that table's
  connection type (legacy AS, SQL Import, etc.)

Document only pages and visuals that reference legacy tables. Note skipped
pages explicitly.

**Parsing notes:**
- Layout file is UTF-16-LE encoded — decode before parsing as JSON
- Walk expression trees recursively to find all `Measure` and `Column`
  references inside `Query`, `filters`, and `config` blocks
- Check `singleVisual.projections` for role assignments (Rows, Columns,
  Values, etc.)
- Visual titles are in
  `singleVisual.vcObjects.title[].properties.text.expr.Literal.Value`
- This step can run in parallel with Step 3 if the Layout file is available

If no Layout file is provided, skip this step and skip Step 9.

**Output:** Section appended to `Step1_Source_Model.md`

**Pause and verify** before proceeding.

---

## Step 3 — Map to Target Semantic Model

Using the Step 1 documentation and the live target model (via auxiliary Power
BI Desktop file), produce two outputs.

### Target Model Exploration Checklist

Before mapping, answer these questions about the target model by querying the
live instance. If a model reference file already exists in
`models/<target-model>/model-reference.md`, read it first and only query the
live model for items marked as pending or unknown.

If no reference file exists yet, document everything discovered here and
**create the reference file** at the end of Step 3.

**Required discoveries:**

| # | Question | Why It Matters |
|---|---|---|
| 1 | What is the main fact table? | Replaces the legacy fact table in all measures |
| 2 | What is the time dimension table and how does it work? | Drives all date filtering — may require a specific dual-column pattern |
| 3 | Does the model require a point-in-time filtering pattern? | If yes, Static versions of measures must be created |
| 4 | Where do user-created measures live (measure host table)? | All new measures must be placed in the correct table |
| 5 | What are the base measure names for key stock types or categories? | Legacy measures often filter a generic base; the target may have typed measures |
| 6 | What are the dimension tables and their key columns? | Needed for relationship recreation and measure rewrites |
| 7 | Do legacy dimension column names exist in the target? | Mismatches must be resolved before Step 4 |
| 8 | Are there any tables the target model lacks entirely? | Flags gaps that need SM team input or substitutions |

---

### Output A — Table Mapping

For each legacy table in the source report, identify the equivalent in the
target model. Use status codes:

- ✅ Clean substitution — direct equivalent confirmed
- ⚠️ Partial — equivalent exists but column names or structure differ; add notes
- ❌ No equivalent — flag for manual decision or SM team
- ➡️ Kept as-is — non-legacy table; no action needed

Include a clear list of exactly which target model tables to add in Step 4.

---

### Output B — Measure Mapping

For each locally-defined measure that references the legacy model:

- Identify the equivalent base measure in the target model
- Note whether the mapping is a clean substitution or requires the target
  model's point-in-time filtering pattern
- Flag measures with no clear equivalent
- Identify which measures need a Static version alongside the general version
- Identify measures that are deferred pending reconnection of other sources

Group measures by dependency order — this becomes the creation sequence in
Step 7.

**Output:** `Step3_Mapping.md`

**Pause and verify** both outputs before proceeding.

---

## Step 4 — Add Target Model Tables to Source Report

Based on Output A from Step 3, provide the user with a clear list of required
target model tables. The user manually adds the target AS connection and
imports only those tables into the source report.

**Owner: User (manual step in Power BI Desktop)**

After the user completes this step, verify the tables are present by querying
the live model.

**Pause and verify** before proceeding.

---

## Step 5 — Recreate Relationships

Using the relationship documentation from Step 1, recreate all relationships
between the newly added target model tables and non-legacy tables (SQL Server
Import, etc.).

**Always do this before removing old relationships** to avoid breaking visuals
mid-migration.

Relationship types to recreate:
- Non-legacy internal relationships (e.g. SQL Server table chains)
- Cross-source relationships between target model dimensions and other sources
  (e.g. a shared Material ID joining target model to a demand planning source)

Note any tables that Power BI may have auto-renamed to avoid conflicts with
existing legacy tables of the same name (e.g. `Material` becoming `Material 2`).
Account for these renamed tables in all subsequent steps.

**Pause and verify** all new relationships are correct before proceeding.

---

## Step 6 — Remove Legacy Relationships

Remove all relationships between legacy semantic model tables and non-legacy
tables (SQL Server Import, etc.).

**Pause and verify** no unintended relationships were removed and all new
relationships remain intact before proceeding.

---

## Step 7 — Recreate Measures

Using Output B from Step 3, rewrite all locally-defined measures that
referenced the legacy model.

**Creation rules:**
- Follow the dependency order from Output B — base measures first
- Apply the target model's point-in-time filtering pattern where required
  (see the target model reference file for the specific pattern)
- Create Static versions alongside general versions for all applicable measures
- Place all user-created measures in the correct measure host table
- Place date label measures in the target model's time dimension table
- Delete each legacy measure immediately before creating its replacement
  (Power BI enforces globally unique measure names across the model)
- Measures flagged as deferred or with no clean mapping are noted for
  manual review — do not block the migration on these

**Static measure rule:** Static measures lock the calculation to the most
recent available date in the current period. They are needed for current-status
visuals (cards, summary tables). General (non-Static) versions are used for
historical trend visuals. The exact pattern for Static measures depends on the
target model's time dimension — see the model reference file.

**Pause and verify** measure results against expected values before proceeding.

---

## Step 8 — Remove Legacy Tables

Once all relationships and measures have been migrated and verified, the user
removes the legacy semantic model connection from the source report. This drops
all associated legacy tables in a single action.

**Owner: User (manual step in Power BI Desktop)**

Verify the model is clean — no broken measure references, no orphaned
relationships — before closing out.

**Pause and verify** before proceeding.

---

## Step 9 — Visual Replacement Guide

*Skip if no Layout file was provided in Step 2.*

Using the visual/measure map from Step 2 and the new measures from Step 7,
produce a replacement guide appended to the migration MD file.

For each page with legacy references, provide a table:

| Visual Title | Visual Type | Field / Filter | Legacy Measure or Column | Legacy Table | Replacement Measure or Column | Notes |
|---|---|---|---|---|---|---|

**Rules:**
- Only include visuals and filters that referenced a legacy measure or column
- Visuals with no legacy references are skipped
- Report-level and page-level filters are listed at the top of each page
  section before visual-level entries
- Where a legacy item had no clean equivalent, note that explicitly
- Flag any renamed tables (e.g. `Material 2`) in the Notes column

This guide is the user's checklist for manually updating visuals after the
model migration is complete.

**Output:** Section appended to migration MD file

---

## Checkpoints Summary

| Step | Action | Owner | Output |
|---|---|---|---|
| 1 | Explore source report | Claude | `Step1_Source_Model.md` |
| 2 | Parse Layout JSON | Claude (if file provided) | Appended to Step 1 MD |
| 3 | Map to target model | Claude | `Step3_Mapping.md` |
| 4 | Add target model tables | **User** | Tables present in model |
| 5 | Recreate relationships | Claude | Relationships created in model |
| 6 | Remove legacy relationships | Claude | Legacy relationships removed |
| 7 | Recreate measures | Claude | Measures created in model |
| 8 | Remove legacy tables | **User** | Clean model |
| 9 | Visual replacement guide | Claude (if Layout provided) | Appended to migration MD |

---

## Notes on Token Efficiency

Not every step justifies full AI execution. Apply this framework:

- **Full AI execution** — complex, programmatic, or high-volume tasks where
  AI handling is clearly superior: source exploration (Step 1), Layout parsing
  (Step 2), measure recreation (Step 7), visual replacement guide (Step 9)
- **AI-guided, user-executed** — simple tasks where clear instructions
  suffice: simple relationship changes (Step 5/6 when only 1-2 relationships
  are involved), straightforward mappings (Step 3 when the target is
  well-documented)
- **User action** — structural changes to the report that the user should own:
  adding connections (Step 4), removing connections (Step 8)

MCP connection reliability affects token efficiency significantly. Dropped
connections due to network instability, session expiry, or closed files require
full reconnection and context re-establishment. Keep all Power BI files open
and maintain a stable network connection for the duration of the migration.

---

*End of document.*
