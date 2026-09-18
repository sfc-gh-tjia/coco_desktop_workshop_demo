---
name: campaign-validator
description: "Validate campaign data tables against business rules: no negative spend, impressions >= clicks, no future dates, all required channels present. Use when: validating campaign data, checking campaign quality, pre-flight campaign check, campaign QA, campaign data validation, verify campaign table, campaign health check."
---

# Campaign Validator

Validates a campaign data table against standard business rules and produces a pass/fail report with violation counts and sample rows.

## Workflow

### Step 1: Identify Target Table

**Goal:** Determine which table to validate.

**Actions:**

1. Extract the fully qualified table name from the user's message (DATABASE.SCHEMA.TABLE).
2. If not specified, ask: "Which table should I validate? Provide the fully qualified name (e.g., DB_MERKLE_WORKSHOP.RAW.CLIENT_CAMPAIGNS)."
3. Verify the table exists and identify columns:
   ```sql
   DESCRIBE TABLE <TABLE>;
   ```
4. Map columns to expected roles. The skill needs:
   - A **spend** column (numeric) — match columns named: spend, cost, budget, amount
   - An **impressions** column (numeric) — match: impressions, views, reach
   - A **clicks** column (numeric) — match: clicks, click_count
   - A **date** column (date/timestamp) — match: start_date, date, created_date, campaign_date
   - A **channel** column (varchar) — match: channel, media_channel, channel_name

   If any required column cannot be auto-detected, ask the user to map it.

### Step 2: Run Validation Rules

**Goal:** Execute all 4 validation rules in a single query for efficiency.

**Actions:**

Run this combined validation query (substitute detected column names):

```sql
SELECT
    -- Rule 1: No negative spend
    COUNT(CASE WHEN <spend_col> < 0 THEN 1 END) AS negative_spend_count,

    -- Rule 2: Impressions >= Clicks
    COUNT(CASE WHEN <clicks_col> > <impressions_col> THEN 1 END) AS clicks_exceed_impressions_count,

    -- Rule 3: No future-dated records
    COUNT(CASE WHEN <date_col> > CURRENT_DATE() THEN 1 END) AS future_dated_count,

    -- Total rows for context
    COUNT(*) AS total_rows
FROM <TABLE>;
```

Then check required channels:

```sql
-- Rule 4: All required channels present
SELECT channel_value FROM (
    SELECT column1 AS channel_value FROM VALUES
    ('Email'), ('Social Media'), ('Display'), ('Search'), ('Video'), ('Connected TV')
) required
WHERE channel_value NOT IN (
    SELECT DISTINCT <channel_col> FROM <TABLE>
);
```

### Step 3: Collect Sample Violations

**Goal:** For each failing rule, retrieve up to 3 sample rows to help with investigation.

**Actions:**

Only run these for rules that have violations (count > 0):

```sql
-- Samples for Rule 1 violations
SELECT * FROM <TABLE> WHERE <spend_col> < 0 LIMIT 3;

-- Samples for Rule 2 violations
SELECT * FROM <TABLE> WHERE <clicks_col> > <impressions_col> LIMIT 3;

-- Samples for Rule 3 violations
SELECT * FROM <TABLE> WHERE <date_col> > CURRENT_DATE() LIMIT 3;
```

### Step 4: Generate Report

**Goal:** Present a structured pass/fail report.

**Format:**

```
## Campaign Validation Report

**Table:** <TABLE>
**Rows Scanned:** <total_rows>
**Timestamp:** <current_timestamp>

### Results

| # | Rule | Status | Violations | % Affected |
|---|------|--------|------------|------------|
| 1 | No negative spend | PASS/FAIL | <count> | <pct>% |
| 2 | Impressions >= clicks | PASS/FAIL | <count> | <pct>% |
| 3 | No future-dated records | PASS/FAIL | <count> | <pct>% |
| 4 | All required channels present | PASS/FAIL | <missing_list> | — |

### Overall: PASS / FAIL (<n> of 4 rules passed)

### Violations Detail
[For each failing rule: description + sample rows]

### Recommendations
[Actionable next steps for each failing rule]
```

## Stopping Points

- After Step 1: If columns cannot be auto-detected, ask user to map them
- After Step 4: Present report. Offer to generate fix SQL for violations.

## Output

A structured pass/fail report with:
- Per-rule status (PASS/FAIL) with violation counts and percentages
- Sample violation rows for investigation
- Actionable fix recommendations for each failure
