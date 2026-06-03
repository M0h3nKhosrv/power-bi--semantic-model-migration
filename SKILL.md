---
name: pbi-semantic-model-migration
description: >
  Migrate any Power BI report from a legacy AS semantic model to a new target
  AS semantic model. Use this skill whenever the user mentions migrating a Power
  BI report to a new semantic model, reconnecting a report to a different AS
  source, recreating DAX measures in a new model, documenting an existing Power
  BI model for migration, or updating relationships after a semantic model
  change. Also trigger when the user asks about mapping legacy tables or
  measures to a new model, recreating point-in-time measures, or producing a
  visual replacement guide after a model change. Always read this skill before
  attempting any Power BI model exploration, mapping, or measure creation
  related to a semantic model migration.
compatibility: "Requires powerbi-modeling-mcp tools to be connected and available."
---

# Power BI Semantic Model Migration Skill

This skill guides migration of any Power BI report from a legacy AS semantic
model to a new target AS semantic model. The process is universal. The target
model knowledge is specific to each destination.

---

## How to Use This Skill

**Always read these files before starting any migration:**

1. `process/migration-guide.md` — The universal 9-step migration process.
   Follow it step by step regardless of source or target model.

2. `models/<target-model>/model-reference.md` — The reference file for the
   specific target model being migrated to. Contains its tables, confirmed
   columns, base measures, time dimension, filtering patterns, and known gaps.

If no model reference file exists for the target model yet, **create one during
Step 3** as part of the live model exploration. Use the Target Model Exploration
Checklist in `process/migration-guide.md` as the structure.

---

## Connection Protocol

**Always do this at the start of every session — ports change on every Desktop restart.**

1. Call `ListLocalInstances` to discover running Power BI Desktop instances
2. Connect to the source report instance and the target model auxiliary file separately
3. Connect: `Provider=MSOLAP;Data Source=localhost:<port>`
4. Call `List` on `database_operations` to confirm the active database
5. If `Connect` returns "No databases found" on the auxiliary file, ask the
   user to add a local model in Power BI Desktop before proceeding

---

## Key Rules

- **Read the target model reference before mapping** — never guess at table
  names, column names, or measure names in the target model
- **If no target model reference exists, build one** — use the exploration
  checklist in the migration guide to document it during Step 3, then save it
  to `models/<target-model>/model-reference.md` for future use
- **Source documentation goes in per-migration output files** — not in this skill
- **Never modify EXTERNALMEASURE pass-throughs** — they have no local DAX;
  changes won't persist
- **Always create measures in dependency order** — base measures first, then
  measures that depend on them
- **Pause and verify after every step** — do not proceed without user confirmation
- **Step 4 is always a user action** — Claude cannot add a new AS connection
  to a report; the user must do this manually in Power BI Desktop
- **Create before removing** — always create new relationships before removing
  old ones to avoid breaking visuals mid-migration

---

## Skill File Structure

```
pbi-semantic-model-migration/
├── SKILL.md                                   ← this file
├── process/
│   └── migration-guide.md                     ← universal 9-step process
└── models/
    └── <target-model>/
        └── model-reference.md                 ← one file per target model, created during Step 3
```

Add a new folder under `models/` for each new target model encountered.
Model reference files contain internal company knowledge — keep them private
and do not include them in any shared or public version of this skill.

---

## Reference Files

| File | Contents |
|---|---|
| `process/migration-guide.md` | Universal 9-step migration process |
| `models/<target-model>/model-reference.md` | Created during Step 3 for each new target model encountered |
