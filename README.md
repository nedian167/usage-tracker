# Copilot Cowork Usage Tracker

`usage-tracker` is a Copilot Cowork skill for recording per-task credit consumption and
reporting weekly and monthly usage. It classifies verified tasks as Light, Medium, or
Heavy and keeps a cumulative running total in an Excel workbook.

Copilot Cowork does not currently provide this level of personal, per-task usage analysis
out of the box. This skill fills that gap by giving users a simple history of their tasks,
credit consumption, workload mix, and weekly and monthly trends.

## Features

- Logs one workbook row per Cowork task.
- Derives task size from verified credits: Light `<300`, Medium `300-700`, Heavy `>700`.
- Preserves the user's original prompt and current runtime model when available.
- Reports weekly and monthly totals, averages, trends, and task-size breakdowns.
- Records unknown credit values as pending instead of estimating them.
- Creates one personal tracker at `Documents\Cowork\Cowork Credit Tracker.xlsx`.

## Install

1. Download [`SKILL.md`](SKILL.md) from this repository.
2. Attach `SKILL.md` to a Copilot Cowork conversation.
3. Ask Cowork:

   ```text
   Configure the attached usage-tracker skill for me.
   ```

4. After Cowork confirms that the skill is provisioned, ask it to remember the
   end-of-task reminder:

   ```text
   Remember that after each successfully completed task, you should ask me to run
   the /cost command and provide the credits used for that task so the usage-tracker
   skill can log them in Cowork Credit Tracker.xlsx.
   ```

Cowork will use the attached instructions to provision the skill for the user. The saved
memory makes the `/cost` reminder part of future successful task completions. No workbook
or usage data is included in the downloaded skill file.

## How to use

On its first run, the skill creates `Cowork Credit Tracker.xlsx` in the user's
`Documents\Cowork` folder, with the `Usage` sheet and required columns ready for logging.

The skill cannot read the task's credit consumption automatically. After completing a
Cowork task:

1. Run the `/cost` command in Cowork.
2. Copy the credit figure shown by `/cost`.
3. Invoke the skill and provide that exact value, for example:

   ```text
   Log usage of credits 245.
   ```

The skill appends the task to the tracker, derives its Light, Medium, or Heavy size, and
updates the running total. Users can then ask for summaries such as:

```text
Show my Cowork usage for this week.
Summarize my July 2026 credit consumption.
How many Heavy tasks did I run last month?
```

## Example Requests

```text
Log this task with 245 credits.
Track this task's usage; /cost shows 820 credits.
Show my Cowork usage for this week.
Summarize my June 2026 credit consumption.
How many Heavy tasks did I run last month?
```

## Workbook Screenshots

All task and credit data below is fictional. Model labels in the Usage preview match
models recorded by the tracker.

### Usage Tab

![Excel tracker sample with color-coded Light, Medium, and Heavy tasks](assets/usage-tracker-sheet.png)

*Fictional Excel-rendered Usage tab showing the complete A-J schema end to end. The
header matches the canonical workbook; Light is green, Medium is amber, Heavy is red,
and the Model column uses actual model names.*

### July Credits Tab

![Fictional July credit summary with weekly totals and task-size breakdown](assets/july-credits-sheet.png)

*Fictional Excel-rendered July Credits tab, proportioned to the Usage preview, showing
weekly consumption, the monthly total, and credits and prompt counts by task size.*

### Accessible Data Sample

| Date | Time | Task Name | Task Type | Task Size | Credits Used | Running Total | User Prompt | Notes | Model |
|---|---|---|---|---|---:|---:|---|---|---|
| 2026-07-02 | 09:10 | Summarize workshop notes | one-off | Light | 180 | 180 | Summarize these fictional workshop notes. | Completed | Claude Opus 4.8 |
| 2026-07-03 | 14:25 | Draft product launch plan | one-off | Medium | 460 | 640 | Draft a launch plan for a sample product. | Completed | Claude Sonnet 5 (1M context) |
| 2026-07-06 | 11:40 | Build market research report | one-off | Heavy | 920 | 1560 | Create a research report from sample data. | Completed | Claude Opus 4.8 |
| 2026-07-08 | 10:15 | Prepare meeting recap | one-off | Light | 240 | 1800 | Prepare a recap from fictional meeting notes. | Completed | Claude Opus 4.8 |

## Weekly Summary Sample

**Period:** July 5-11, 2026

| Metric | Total | Light | Medium | Heavy |
|---|---:|---:|---:|---:|
| Tasks | 3 | 1 | 1 | 1 |
| Verified credits | 1810 | 240 | 650 | 920 |

- Average: 603.3 credits per verified task
- Highest-credit task: Build market research report (920)
- Pending tasks: 0
- Previous-week change: Not enough prior data

## Monthly Summary Sample

| Week | Verified Credits |
|---|---:|
| Week of Jun 28 | 640 |
| Week of Jul 5 | 1810 |
| Week of Jul 12 | 1710 |
| Week of Jul 19 | 1300 |
| Week of Jul 26 | 210 |
| **July total** | **5670** |

## Repository Privacy

This repository intentionally contains no real tracker workbook, prompts, usage records,
email addresses, usernames, tenant IDs, drive IDs, access tokens, or machine-specific
paths. Sample values are fictional.

## License

MIT
