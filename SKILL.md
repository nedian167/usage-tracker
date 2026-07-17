---
name: usage-tracker
description: |
  Append one row to the user's existing "Cowork Credit Tracker.xlsx" workbook (in the
  OneDrive Documents\Cowork folder) at the end of a task. Use when the user says "log this task",
  "track this task's usage", "add this to my credit tracker", "record my credits",
  "update my usage/credit tracker", or types "/track-cost". Also runs when a task's
  personal instructions request end-of-task usage logging. Also captures a color-coded
  Heavy/Medium/Light task-size tier (derived from credits), the user's original prompt,
  and the runtime model that ran the task. Also use for weekly or monthly personal
  credit summaries and task-size breakdowns.

  Do NOT use for creating an unrelated spreadsheet (use the xlsx skill) or reporting
  Microsoft 365 subscription/billing usage. When the user only asks to view, chart, or
  analyze the tracker, read it without appending a row.
cowork:
  category: automation
  icon: DocumentBulletList
---

# Usage / Credit Tracker

Append **exactly one row per task** to the user's existing workbook **"Cowork Credit
Tracker.xlsx"**. Append only — never rewrite, reorder, or delete history.

Canonical file — always target this exact file, never create it elsewhere:
`Documents/Cowork/Cowork Credit Tracker.xlsx`.

**Fast path — skip discovery entirely.** The workbook's Graph `id` + `driveId` are persisted in
memory (key `fact-cowork-tracker-location`) the first time the file is created or resolved. On
every run, **`recall_memories` that key FIRST** and go straight to
`/me/drive/items/{id}/workbook/...` — no folder listing, no search, no path probing. Only if a
workbook call returns **404** (file moved / renamed / deleted) fall back to discovery (step 2),
then re-save the fresh id. This is the single biggest credit saver: on the happy path, discovery
costs **zero** calls.

**Edit the canonical file IN PLACE via the Graph workbook API** (primary path below). This
appends the row directly to the live file — no download, no local rebuild, no `output/`
copy, no sync wait, no relocate, and **no stray copies to merge**. It also preserves the
file's sensitivity label (a local openpyxl rebuild strips it). Only if the workbook API is
unavailable, fall back to the openpyxl + relocate path at the end.

## When NOT to Append
- **Viewing / charting / analyzing** the tracker → read the workbook without appending.
- **A different or unrelated spreadsheet** → use the **xlsx** skill.
- **M365 subscription or license billing** → out of scope; this logs per-task Cowork credits only.
- **Backfilling a past session's model or credits as if verified** → only the current session's
  values are verifiable; leave unknowns blank rather than guessing (see Guardrails).

## Schema — sheet `Usage` (do not invent columns)
Columns A–J, in order:

`Date` | `Time` | `Task Name` | `Task Type` | `Task Size` | `Credits Used` | `Running Total` | `User Prompt` | `Notes` | `Model`

- **Date** — YYYY-MM-DD in the user's current timezone. **Time** — HH:MM (24h) in the
  user's current timezone.
- **Task Name** — short description.  **Task Type** — `one-off` or `recurring`.
- **Task Size** (col E) — auto-derived from **Credits Used** (never asked, never guessed ahead of credits):
  Light <300 · Medium 300–700 · Heavy >700. Blank when Credits Used is blank.
  **Coloring is automatic** via standing conditional-format rules on column E (see step 5 / "Creating
  the tracker"): Light `#C6EFCE` / `#006100` · Medium `#FFEB9C` / `#9C6500` · Heavy `#FFC7CE` / `#9C0006`.
- **Credits Used** (col F) — the figure the user gives from `/cost`. Never estimated, inferred, or fabricated.
- **Running Total** (col G) — previous row's Running Total + this row's Credits Used (blank credits count as 0, carry forward).
- **User Prompt** (col H) — the user's verbatim original request (not a paraphrase). Auto-captured — never ask. Truncate to ~500 chars + "…" if longer.
- **Notes** (col I) — brief context, or `pending — awaiting /cost figure` when no credits are given.
- **Model** (col J) — see below.

## Credits and Model — what can and cannot be captured
- **Credits Used is NOT readable by the assistant.** `/cost` and `/usage` are interactive commands shown only
  to the user, so credits must come from the user. If none is given: leave Credits Used blank, leave Task Size
  blank, carry Running Total forward, and put `pending — awaiting /cost` in Notes.
- **Model is auto-captured** from the assistant's own current runtime model — verified for the live task (e.g.
  "Claude Opus 4.8"). Only if it genuinely can't be determined, ask for it in the **same message** where you
  request the `/cost` figure; if still unknown, leave Model blank and note `model not provided`.
- **Never fabricate, estimate, infer, or back-stamp** a credits number, a Task Size tier, or a Model — and never
  stamp a model onto a row for any session other than the current one.

## Workflow — edit the canonical in place (primary)
Use `/me/drive/items/{id}/…` paths only (never `/drives/{drive-id}/items/{id}`, which fails here). Reads are
Graph `GET` (QueryGraph); writes are Graph `PATCH`/`POST` (CallGraph).

1. Resolve the current date/time in the user's current timezone, capture the user's verbatim original prompt
   (truncate ~500 chars + "…"), and **`recall_memories("fact-cowork-tracker-location")`** for the
   stored `id` + `driveId`. Have it → jump to step 3 with `/me/drive/items/{id}/workbook`.
2. **Resolve the item id — only if the memory fast path missed (or a call 404s).** List the Cowork
   folder with `GetDriveChildren(item_path="/Documents/Cowork")` — its payload carries real `015…`
   item ids (the `:/children` QueryGraph render can collapse and hide the `.xlsx`, so don't rely on
   it). Take the `id` of `Cowork Credit Tracker.xlsx`; if absent, fall back once to
   `SearchM365(sources=["files"], query="Cowork Credit Tracker")`. **If it exists nowhere, create it
   (see "Creating the tracker") and stop.** On any create OR fresh resolve, immediately persist
   `{id, driveId}` via `save_memory(key="fact-cowork-tracker-location", …)` so every later run takes
   the fast path.
3. **Read row count + last total in ONE call** —
   `GET /me/drive/items/{id}/workbook/worksheets('Usage')/usedRange?$select=rowCount,values`.
   `rowCount` is `R` (new row = `R+1`); the previous Running Total is the last returned row's
   column-G value. No second range read.
4. **Append the row in ONE write** — fold values + date/time format into a single PATCH:
   `PATCH /me/drive/items/{id}/workbook/worksheets('Usage')/range(address='A{R+1}:J{R+1}')` with body
   `{"values": [[ …10 values… ]], "numberFormat": [["yyyy-mm-dd","hh:mm","General","General","General","General","General","General","General","General"]]}`.
   The API auto-types the `"YYYY-MM-DD"`/`"HH:MM"` strings into date/time serials; the inline
   `numberFormat` fixes their display in the same call — **no separate format PATCH**.
   (Excel-Table sheet? `POST …/worksheets('Usage')/tables/{table}/rows/add` with `{"values": [[ … ]]}`
   also works, auto-appending with no index.)
5. **Task-Size color = standing conditional formatting, NOT per-row PATCHes.** The `Usage` sheet
   carries three conditional-format rules on column E (Heavy/Medium/Light → tier fill+font), baked in
   at creation (see "Creating the tracker"), so an appended `"Heavy"`/`"Medium"`/`"Light"` value
   colors itself with **zero** extra calls. Do NOT PATCH `E{R+1}` fill/font when those rules exist.
   *Legacy fallback only* — a workbook created before the CF rules has none on column E: color the one
   cell manually (`PATCH …/range('E{R+1}')/format/fill` `{"color":"<fill>"}` + `…/format/font`
   `{"color":"<font>"}` per the tier) and tell the user to add the CF rules once in Excel (Home →
   Conditional Formatting → Highlight Cells → Equal To) to drop this step for good.
6. **Verify from the write response** — the range `PATCH` echoes the written `address`/`values`. That
   is the confirmation — do **not** re-download the workbook, and never run a second `SearchM365` to
   "find" what you just wrote (the index lags; the write response is authoritative).
7. Confirm to the user in one line (see Output format).

### Example — appending the 11th data row (sheet has header + 10 rows, `R = 11`, fast path hit)
```
recall_memories("fact-cowork-tracker-location") → id=015…, driveId=b!…   (0 discovery calls)
GET   …/workbook/worksheets('Usage')/usedRange?$select=rowCount,values   → rowCount 11; last G = 8521
PATCH …/workbook/worksheets('Usage')/range(address='A12:J12')
      {"values":[["2026-07-13","16:45","Optimize a skill","one-off","Light",250,8771,"<prompt>","<notes>","Claude Opus 4.8"]],
       "numberFormat":[["yyyy-mm-dd","hh:mm","General","General","General","General","General","General","General","General"]]}
# color: none — the column-E conditional-format rule paints "Light" automatically.
```
Three calls total (recall + read + write) versus the old ~6+. Session-less workbook calls persist to
the stored file immediately — no `workbook-session-id` header is available through the connector, and
none is needed. The change hits the server copy right away; if the user has the workbook open in
desktop Excel (or a synced local copy) it only appears after a refresh/reopen — say so when confirming.

## Creating the tracker (first run — bake in CF + pin the id)
If the workbook exists nowhere, create it ONCE, then never rediscover:
1. Build with `openpyxl` (or the xlsx skill): a `Usage` sheet with the A1:J1 header (fill `#7A1730`,
   bold white `#FFFFFF`) and a monthly `"<Mmm> Credits"` summary sheet (formula-driven via SUMIFS /
   COUNTIFS over `Usage` — auto-recomputes, no per-run writes).
2. **Bake the three conditional-format rules onto column E** so coloring is automatic forever (this is
   the credit win — every future append skips the fill/font PATCHes):
   ```python
   from openpyxl.formatting.rule import CellIsRule
   from openpyxl.styles import PatternFill, Font
   rng = 'E2:E100000'
   ws.conditional_formatting.add(rng, CellIsRule(operator='equal', formula=['"Heavy"'],
       fill=PatternFill('solid', fgColor='FFC7CE'), font=Font(color='9C0006')))
   ws.conditional_formatting.add(rng, CellIsRule(operator='equal', formula=['"Medium"'],
       fill=PatternFill('solid', fgColor='FFEB9C'), font=Font(color='9C6500')))
   ws.conditional_formatting.add(rng, CellIsRule(operator='equal', formula=['"Light"'],
       fill=PatternFill('solid', fgColor='C6EFCE'), font=Font(color='006100')))
   ```
   (CF cannot be added through the sessionless workbook API — the `conditionalFormats` endpoint needs
   a workbook session the connector can't open — so it MUST be set at openpyxl creation, or once by
   the user in Excel on a legacy file.)
3. Upload to `Documents/Cowork`, then **capture the returned item `id` + `driveId` and
   `save_memory(key="fact-cowork-tracker-location", …)`** immediately. Every subsequent run uses the
   fast path and never rediscovers.

## Output format
One confirmation line: what was logged, Credits Used (value or "pending"), the derived Task Size (value or
"pending"), the new Running Total, and that `Documents/Cowork/Cowork Credit Tracker.xlsx` itself now
reflects it (verified from the write response). If asked, show recent rows as a small `render-ui` table.

## Weekly and monthly reporting
For requests such as "show my usage this week" or "summarize this month's credits":
1. Read the existing `Usage` rows or the formula-driven monthly credits sheet. Do not append.
2. Use the user's current timezone. A week is Monday-Sunday unless the user specifies
   another convention.
3. Report total verified credits; total and verified task counts; Light, Medium, and Heavy
   counts and credits; average credits per verified task; highest-credit task; and pending
   task count.
4. For monthly summaries, include a week-by-week breakdown. Compare with the immediately
   preceding equivalent period when sufficient data exists.
5. Never estimate missing credits. Exclude pending rows from credit totals and identify
   their count.

## Fallback — only if the workbook API is unavailable
If the workbook endpoints error (feature off, unsupported), rebuild locally and relocate:
1. `ReadFileContent` the canonical file (its JSON gives the real `id` + `parentReference.id`).
2. **Self-heal merge** (needed only on this path, because it creates `output/` copies): from one
   `SearchM365(query="Cowork Credit Tracker")`, download any stray `Cowork Credit Tracker*.xlsx` under
   `Documents\Cowork\Tasks\*\output\` **only if listed**, merge rows not already present (match Date+Time+Task
   Name), drop placeholder/undated rows, re-sort, recompute Running Total.
3. `openpyxl` append one row (plain numeric Running Total); keep the column-E CF rules intact (they carry the
   coloring); build/validate in `working/` (`zipfile.is_zipfile` + reopen), then copy one clean file to
   `output/Cowork Credit Tracker.xlsx`.
4. **Relocate via metadata PATCH** (never content upload). Navigate the live drive (not the lagging search index)
   to this session's synced copy — `GET /me/drive/root:/Documents/Cowork/Tasks:/children?$orderby=lastModifiedDateTime desc`
   → newest task folder → `output` child → its item id. Then **single replace-move**:
   `PATCH /me/drive/items/{new-id}` `{"name":"Cowork Credit Tracker.xlsx","parentReference":{"id":"<Cowork folder id>"},"@microsoft.graph.conflictBehavior":"replace"}`.
   On `409` (tenant won't replace on move), rename the old canonical aside
   (`{"name":"Cowork Credit Tracker (superseded <date>).xlsx"}` — never delete) then re-issue the move without
   `conflictBehavior`. Verify from the PATCH response (`name` + `parentReference.id`), not a re-download.
5. Re-`save_memory` the new item id, and list any superseded/stray copies by path so the user can remove them
   manually (no delete tool exists).

## Guardrails
- **Fast path first** — `recall_memories("fact-cowork-tracker-location")` and go direct to
  `/me/drive/items/{id}/workbook`; only discover on a 404, then re-save the id. Persist the id at creation.
- **Prefer in-place workbook-API edits** — they avoid the round-trip, create no strays, and preserve the sensitivity label. Use the openpyxl+relocate fallback only when the API is unavailable.
- **Task-Size color comes from standing conditional-format rules** (set at creation), not per-row writes; only color manually as a legacy fallback when the sheet has no CF on column E.
- **Never fabricate or estimate** Credits Used, Task Size, or Model — blank + a `pending`/`not provided` note when unknown.
- **Task Size is strictly derived** from Credits Used thresholds, only once credits are known.
- **Model = the running model or the user's answer**, never a guess, never back-stamped onto another session's row.
- **Discover via `GetDriveChildren(item_path="/Documents/Cowork")`** (real `015…` ids); only broaden to `SearchM365` if absent. **Never run a second index search for something just written** — use the write response or live drive navigation.
- **Append only** — preserve every prior row (drop only an explicit "Sample row" / undated placeholder during a fallback merge).
- **Keep exactly one live workbook** at the canonical path.
- **Verify from the API/PATCH response, never a full re-fetch.** Never PUT raw content to an item's `/content` endpoint via `CallGraph` (JSON≠binary); edits are workbook-API or metadata PATCH only.
- **Never delete** — on the fallback path, free the canonical name by renaming the old file aside (recoverable).
