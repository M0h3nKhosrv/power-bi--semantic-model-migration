# Power BI Semantic Model Migration with Claude

A reusable skill for migrating Power BI reports from a legacy Analysis Services semantic model to a new target AS semantic model — using Claude as the migration agent.

## What This Skill Does

This skill gives Claude a structured, repeatable 9-step process for handling AS semantic model migrations in Power BI. It covers source report exploration, visual and measure mapping, relationship management, measure recreation, and a visual replacement guide — with deliberate human checkpoints at every step.

The process is model-agnostic. It works for any source report and any target AS semantic model.

## What's Included

- **`SKILL.md`** — the entry point Claude reads to understand the skill and how to apply it
- **`process/migration-guide.md`** — the full 9-step migration process with a target model exploration checklist, token efficiency guidance, and checkpoint structure

## What's Not Included

Target model reference files are not included. These contain internal knowledge about specific semantic models and are company assets. When this skill is used for a migration, the target model reference is created during Step 3 and stored locally by the team doing the migration.

## Requirements

- Claude Desktop with the `powerbi-modeling-mcp` server connected
- Both the source Power BI report and an auxiliary file connected to the target AS model open in Power BI Desktop during the migration
- A stable network connection for the duration of the migration — MCP connection drops require costly reconnection cycles

## How to Use

1. Place this folder in your Claude skills directory
2. Start a conversation with Claude and describe the migration you want to do
3. Claude will read the skill and follow the 9-step process, pausing for your verification at each step

## Human-in-the-Loop Design

This skill is built around deliberate human oversight. Claude handles the complex, programmatic, and high-volume work. The user owns structural decisions, approves each step before proceeding, and retains control over irreversible actions. Two steps always require manual user action regardless of AI capability: adding the target model connection (Step 4) and removing the legacy connection (Step 8).

## Origins

This skill was developed and validated during a real Power BI report migration. The process, the token efficiency framework, and the checkpoint structure all emerged from that experience. A write-up of the migration is available as a companion mini case document.

---

## The 9-Step Process

### Step 1 — Explore Source Report
Claude connects to the live source report via the Power BI MCP server and extracts all tables, measures, and relationships — separating locally-defined DAX from EXTERNALMEASURE pass-throughs, and identifying which tables belong to the legacy model versus unchanged data sources.

### Step 2 — Parse Report Layout *(Optional)*
If the user provides the Layout JSON file from the `.pbix`, Claude parses it to map every visual, measure binding, and filter across all report pages. This produces a complete picture of which legacy measures each visual depends on — and which pages are unaffected by the migration.

### Step 3 — Map to Target Semantic Model
Claude explores the live target model and produces two outputs: a table mapping (which legacy tables map to which target tables, and which tables to add) and a measure mapping (how each locally-defined measure should be rewritten, in dependency order). Gaps where no equivalent exists are flagged for human decision.

### Step 4 — Add Target Model Tables *(User Action)*
Based on the exact list Claude produces, the user adds the target AS connection to the source report and imports only the required tables. Claude verifies the tables are present before proceeding.

### Step 5 — Recreate Relationships
New relationships between target model tables and unchanged data sources are created before any legacy relationships are removed — to avoid breaking visuals mid-migration.

### Step 6 — Remove Legacy Relationships
Legacy cross-source relationships are removed once the new ones are confirmed correct.

### Step 7 — Recreate Measures
All locally-defined measures are deleted from legacy tables and recreated in the target model in dependency order. Point-in-time Static versions are created alongside general versions where required by the target model's filtering pattern.

### Step 8 — Remove Legacy Tables *(User Action)*
The user removes the legacy AS connection from the source report, dropping all associated tables in a single action. Claude verifies the model is clean before closing out.

### Step 9 — Visual Replacement Guide *(If Layout file was provided)*
Claude produces a page-by-page, visual-by-visual replacement table — telling the user exactly which new measure or column to use in place of every legacy reference across all affected visuals and filters.

---

## Token Efficiency

Not every step justifies full AI execution. The skill applies this framework:

| Mode | When to Apply | Steps |
|---|---|---|
| **Full AI execution** | Complex, programmatic, or high-volume tasks | 1, 2, 7, 9 |
| **AI-guided, user-executed** | Simple tasks where clear instructions suffice | 3 (mapping doc), 5–6 (few relationships) |
| **User action** | Structural report changes the user should own | 4, 8 |

MCP connection reliability also affects token efficiency significantly. Dropped connections due to network instability, session expiry, or closed files require full reconnection and context re-establishment. Keep all Power BI files open and maintain a stable network connection for the duration of the migration.

---

## Adding a New Target Model

When migrating to a target model not yet documented in this skill:

1. During Step 3, use the Target Model Exploration Checklist in `process/migration-guide.md` to discover the model's structure
2. Document findings in a new file at `models/<target-model>/model-reference.md`
3. Use that reference for all subsequent steps in the migration

Target model reference files contain internal company knowledge. Keep them private — do not include them in any shared or public version of this skill.
