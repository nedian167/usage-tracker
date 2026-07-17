---
name: usage-tracker
description: |
  Track a user's own Copilot Cowork credit usage by logging one row per task in an
  Excel workbook. Classify tasks as Light, Medium, or Heavy from verified credit
  usage and summarize weekly or monthly consumption. Use when the user asks to log,
  track, record, review, or summarize Cowork task credits, including `/track-cost`,
  "weekly usage", and "monthly credit consumption". Do not use for Microsoft 365
  licensing, billing, or organization-wide usage reporting.
cowork:
  category: automation
  icon: DocumentBulletList
---

# Copilot Cowork Usage Tracker

Track personal Copilot Cowork consumption in one Excel workbook. Append exactly one row
for each logged task. Read existing rows to produce weekly and monthly summaries without
changing history.

## Configuration

Use this default workbook unless the user explicitly requests another location:

`Documents\Cowork\Cowork Credit Tracker.xlsx`

Use worksheet `Usage` and the user's current timezone. Do not hard-code a username, email
address, tenant identifier, drive identifier, or machine-specific absolute path.

On the first run, if the workbook does not exist, create it automatically at the default
path with the `Usage` sheet and schema below. Create the `Documents\Cowork` folder first
if necessary. Do not create duplicate tracker workbooks.

## Scope

Use this skill to:

- Log the current task's verified credit usage.
- Show recent entries.
- Summarize consumption for the current or requested calendar week.
- Summarize consumption for the current or requested calendar month.
- Break down tasks and credits by Light, Medium, and Heavy classifications.

Do not use it for subscription billing, license allocation, organization-wide analytics,
or unrelated spreadsheet work.

## Workbook Schema

Use columns A-J in this exact order:

`Date` | `Time` | `Task Name` | `Task Type` | `Task Size` | `Credits Used` | `Running Total` | `User Prompt` | `Notes` | `Model`

- **Date**: local date as `YYYY-MM-DD`.
- **Time**: local time as `HH:MM`.
- **Task Name**: short, descriptive title.
- **Task Type**: `one-off` or `recurring`.
- **Task Size**: derived only from verified `Credits Used`:
  - Light: fewer than 300 credits.
  - Medium: 300 through 700 credits.
  - Heavy: more than 700 credits.
  - Blank when credits are unknown.
- **Credits Used**: exact value supplied by the user after running `/cost`.
- **Running Total**: previous Running Total plus Credits Used; unknown credits count as
  zero until updated.
- **User Prompt**: the user's original request, truncated to 500 characters with `...`
  when needed.
- **Notes**: concise context. Use `pending - awaiting credit figure` when credits are
  unknown.
- **Model**: current runtime model when reliably available; otherwise blank.

Format the Task Size cell:

| Size | Fill | Font |
|---|---|---|
| Light | `#C6EFCE` | `#006100` |
| Medium | `#FFEB9C` | `#9C6500` |
| Heavy | `#FFC7CE` | `#9C0006` |

## Logging Workflow

1. Resolve the default workbook and `Usage` worksheet, creating them on first run when
   absent.
2. Capture the user's original prompt and local date/time.
3. Use only the exact credit figure explicitly supplied by the user. The assistant cannot
   read `/cost`; never estimate or infer credits.
4. Read the last populated row and its Running Total.
5. Derive Task Size from the thresholds above.
6. Append one row in place through the workbook API or the host's spreadsheet editing
   capability. Do not download and rebuild a cloud workbook when an in-place API exists.
7. Apply date, time, and Task Size formatting.
8. Verify the append from the write response or by reading only the new row.
9. Confirm the task name, credits or pending status, size, and Running Total.

If no credit figure is available, still log the task once with blank Credits Used and Task
Size, carry the Running Total forward, and set Notes to `pending - awaiting credit figure`.
If the user later provides the figure, update that pending row rather than adding a
duplicate, then recompute Running Total for that row and all later rows.

## Weekly Summary

Interpret "week" as Monday through Sunday unless the user specifies another convention.
For the requested local-date range:

1. Read rows whose Date falls within the range.
2. Exclude pending rows from credit totals, but report their count.
3. Calculate:
   - total verified credits;
   - total tasks and verified tasks;
   - Light, Medium, and Heavy task counts;
   - credits consumed by each size;
   - average credits per verified task;
   - highest-credit task;
   - change versus the immediately preceding equivalent week, when data exists.
4. Present the date range and a compact table. Do not append summary rows to `Usage`.

## Monthly Summary

Use the requested calendar month in the configured timezone:

1. Read rows whose Date falls within the month.
2. Compute the same metrics as the weekly summary.
3. Add a week-by-week credit breakdown within the month.
4. Compare with the previous calendar month when data exists.
5. Present a compact summary and call out pending entries. Do not modify task history.

## Reporting Format

Use this structure:

| Metric | Total | Light | Medium | Heavy |
|---|---:|---:|---:|---:|
| Tasks | ... | ... | ... | ... |
| Credits | ... | ... | ... | ... |

Then show:

- Average credits per verified task.
- Highest-credit task.
- Pending task count.
- Previous-period change, or `Not enough prior data`.

## Privacy and Safety

- Track only the requesting user's own usage unless they explicitly provide and authorize
  another data source.
- Never include workbook data in outbound messages without a preview and confirmation.
- Never commit or publish tracker workbooks, exports, prompts, usage records, identifiers,
  access tokens, or local configuration.
- Never fabricate credits, classifications, model names, or comparisons.
- Preserve all historical rows; only update a row to resolve its pending credit figure.
- Prefer in-place cloud workbook edits to preserve labels, permissions, and version history.
- Surface access and write failures clearly; do not claim a task was logged unless verified.
