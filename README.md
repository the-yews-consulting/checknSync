# ChecknSync

**Manual testing that writes back to Azure DevOps.** Version 0.3.0.

ChecknSync is for tests done by hand: you write the steps, run them in a browser that ChecknSync watches, collect evidence as you go, and save the result to the user story. It sits alongside [ReqnSync](../ReqnSync/), the automated suite's dashboard, but is a separate app. Its code, tests and runs are its own, it has its own violet theme, and it never reads or writes the Gherkin feature files.

> Everything runs on your machine. The dashboard calls ADO directly with your PAT, and the small PowerShell runner only reads and writes files under `ManualTests\` and talks to the browser it opened.

---

## Contents

- [Quick start](#quick-start)
- [Requirements](#requirements)
- [Folders](#folders)
- [Writing tests](#writing-tests)
- [Running a test](#running-a-test)
- [Saving to Azure DevOps](#saving-to-azure-devops)
- [Charts](#charts)
- [Backup and restore](#backup-and-restore)
- [Settings](#settings)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Runner API](#runner-api)

---

## Quick start

1. Start ChecknSync from the repo root:

   ```cmd
   tools\CheckNSync\checknsync.cmd
   ```

   It opens `http://localhost:5079/`. The `.cmd` uses PowerShell 7 (`pwsh`) if it is installed, and Windows PowerShell 5.1 if not.
2. Click the **ADO** chip in the header and connect with a PAT that has **Work Items: Read & write**.
3. In **⚙ Settings**, pick the story field on the Testing tab that the tests are written to (for example *Testing Steps · Custom.TestingSteps*).
4. Create a test with **＋ New test** or **📋 From ACs**, then press **▶ Test**.

To get back to the automated suite, use **⇄ ReqnSync**. ReqnSync's **✅ ChecknSync** button opens this app while it is running. ReqnSync itself starts with `tools\ReqnSync\reqnsync.cmd`.

## Requirements

| What | Why |
|---|---|
| Windows, with PowerShell 7 or Windows PowerShell 5.1 | The runner, `checknsync-runner.ps1` |
| Microsoft Edge or Google Chrome | The browser you test in, watched through its DevTools port |
| An ADO PAT with Work Items: Read & write | Writing results to stories and raising Bugs |
| Azure CLI (`az`) | Only for 🗄 Query with *Microsoft Entra MFA* or *Azure CLI* sign-in |

## Folders

```
tools\CheckNSync\            the app (in git)
  checknsync.cmd             start it
  checknsync-runner.ps1      local web server and browser control
  checknsync.html            the dashboard
  README.md

ManualTests\                 the data, at the repo root
  checknsync.json            settings: environments, databases, Testing field...
  Tests\<Area>\*.md          the tests (in git)
  Queries\**\*.sql           query files steps can run (in git)
  Data\**                    expected-results files, CSVs... (in git)
  runs\<runId>\              each saved run: run.json, report.html, evidence (git ignores it)
  _to_delete\                tests you removed (moved, never deleted; git ignores it)
```

Backups are kept outside the repo; see [Backup and restore](#backup-and-restore).

## Writing tests

A test is a Markdown file shaped like an ADO Test Case:

```markdown
# Cross ILR Change Report X-002 displays attribute changes

Story: 282568 · Sign in as: Provider · Environments: TST
Updated by russell on 10/10/2026 11:17

## Preconditions

* Two ILR files are available in TST that generate an X-002 report
* The Compare ILR journey is available

## Steps

| # | Action | Expected result | Data |
|---|--------|-----------------|------|
| 1 | Select the database | The job rows are returned | query: Queries/JM_JOB.sql |
| 2 | Generate the X-002 report | The report is displayed | |
| 3 | In the report, locate the rows for learner 30101305 | A row per learning aim | download: X-002*.csv |
```

- **Preconditions** appear first in a run, as a checklist to tick.
- **The Data column** is optional, and appears only when a step uses it:
  - `query: Queries/x.sql` runs a query file;
  - `file: Data/y.csv` opens a data file;
  - `download` or `download: *.csv` opens the latest matching file in your Downloads folder.
- **Other `## sections`** are kept as they are. ChecknSync writes and replaces only the `## Last run` section.

**The editor** (✎ Edit) lets you:
- change a test's area and file name, and its runs follow it;
- add steps with phrase buttons (*Navigate to*, *In the report, locate*, *View the row for*…);
- write a step's query with the **🗄** button. It is saved as `Queries/<area>/<file>-step-N.sql`.

**📋 From ACs** turns a work item's Acceptance Criteria into a draft test, in one of two ways:
- **Without AI:** Given lines become preconditions.
- **With Copilot:** copy the prompt, and paste Copilot's answer back. Either form is accepted, the Markdown or the rendered chat view. The first line becomes the title, preconditions are recognised, and step numbers are dropped.

## Running a test

Press **▶ Test** and choose the environment. ChecknSync opens the browser at that environment's start page and keeps track of the tabs and the pages you visit.

| While testing | |
|---|---|
| ✓ ✗ ⊘ – | Pass, fail, block or skip a step. Fail and block ask for a note. |
| **All steps** bar | ✓ Pass all, ✗ Fail all, ⊘ Block all, – Skip all, ↺ Clear. If some steps are already marked, it asks whether to change those too or mark only the rest. |
| 📝 | A note on a step. **Session notes** at the top are for the whole test. |
| 📷 / 📷 Full page | A screenshot of the test browser, on a step or for the whole test |
| Ctrl+V | Paste a screenshot (for example from Win+Shift+S) onto the selected step |
| **✏️ Annotate** | Click a screenshot thumbnail to draw on it: arrow, box, highlight, pen, numbered markers, text, and **▦ Hide** to black out learner details. ↺ Original undoes it until the run is saved. |
| ⏺ Record | Records a window or the whole screen to WebM |
| 🗄 Query | Runs SQL against a test database. **▶ Run query** on a step runs it at once and keeps the results on that step as `query-NN.csv` and `.sql`. |
| 📄 Open file / 📥 Downloads | Opens CSV, TSV, JSON or Excel files, filters them, and adds the file or just the rows shown as evidence |

**About queries:**
- Queries run inside a transaction that is **rolled back** unless you tick *Commit*, and Commit is never allowed on PROD.
- Up to 5,000 rows are kept for each result set.
- A query file says which database it runs on with `-- db: <name>` on its first line.

**■ Finish** stops the browser. Then choose what to save.

## Saving to Azure DevOps

When you save, you can choose any of these:

- **Last run:** writes each step's result, note and evidence into the test file's `## Last run` section.
- **Testing tab:** writes all of the story's manual tests and their last results to the field you chose in ⚙ Settings. ChecknSync only overwrites text it wrote itself; for anything else it asks first.
- **Report to the story:** attaches `<runId>-report.html` (steps, results, notes, screenshots and query results) to the story's Attachments.
- **Bug:** raises a Bug linked to the story, with the failed steps, the notes, screenshots inline, query results, files, the recording and the report.
- **Recording to the story:** attaches the WebM. ADO's limit is about 125 MB.

Every run is also saved in `ManualTests\runs\<runId>\`. The **Runs** tab lists them, with 📄 Report and 🎬 Video. To attach the report of a run you saved earlier, open 📄 Report and press **⇧ Attach to #story**.

## Charts

The **📊 Charts** tab summarises the saved runs. You can filter by test, environment and period, or switch to **▦ Table view**.

- **Totals:** tests and steps, runs, % of runs passed, steps checked, steps failed and blocked, average run time.
- **Step results, run by run:** a stacked bar for each run. Click a bar to open that run's report.
- **Steps passed:** the % of each run's steps that passed, as a line.
- **Where it fails:** fails and blocks by test, or, with one test chosen, by step.
- **Each test:** runs, results, pass rate, last result, and the step it fails at most.

## Backup and restore

**💾 Backup** makes a zip of any of these parts:

- the tests;
- query files;
- data files;
- `checknsync.json`;
- this browser's settings (ADO organisation and project and the environment, **never the PAT**);
- run history;
- evidence of the runs, which is off by default because it can be large.

Backups are saved **outside the repo**, by default in `Downloads\ChecknSync backups\`, so they survive the repo or tools folder being deleted or cloned again. You can choose another folder in ⚙ Settings, but not one inside the repo.

**↺ Restore…** works like this:
- It first backs up the current state as **pre-restore**, so a restore can be undone.
- It writes the chosen parts back over what is there and deletes nothing.
- It only writes inside `ManualTests\`.

**⬇** downloads a zip, and **⬆ Restore from a file…** adds one from another machine.

## Settings

Change these in **⚙ Settings**. They are stored in `ManualTests\checknsync.json`:

| Key | Default | Meaning |
|---|---|---|
| `testingField` / `testingFieldName` | — | The story field the tests are written to |
| `databases[]` | — | `{ name, environment: tst\|preprod\|prod\|any, server, database, auth: entra-mfa\|azure-cli\|windows\|sql, user, tenant, trustServerCertificate }`, each with a **🔌 Test connection** button |
| `downloadsFolder` | your Downloads folder | Where 📥 Downloads looks |
| `backupFolder` | `Downloads\ChecknSync backups` | Where backups go. It must be outside the repo. |

These are only in the file:

| Key | Default | Meaning |
|---|---|---|
| `environments` | read from the suite's appsettings | Start page for each environment |
| `port` | `5079` | ChecknSync's port |
| `reqnsyncPort` | `5077` | ReqnSync's port, for **⇄ ReqnSync** |
| `cdpPort` | `9339` | The test browser's DevTools port |
| `browser`, `browserArgs` | Edge, then Chrome | The browser to test in |

**Database sign-in:**
- **Microsoft Entra MFA** works as it does in SSMS. When needed, it signs you in through `az login` in the browser, with MFA, and keeps the token in memory only.
- **SQL login** keeps the password in the page's memory only.

## Security

- **No passwords or PATs are written to disk.** The PAT is kept in the browser only if you tick *Remember*.
- **Runner access:** the runner listens on `localhost` only, and every call needs the token it gives the page when it starts.
- **File access:** the API only reads and writes inside `ManualTests\`, and it refuses paths with `..`.
- **Removing:** removed tests are moved to `ManualTests\_to_delete\<time>\`, never deleted.
- **SQL:** queries are rolled back by default, and Commit is blocked on PROD.
- **Personal data:** ▦ Hide in the annotator lets you black out learner details before a screenshot goes to ADO.

## Troubleshooting

| Problem | Fix |
|---|---|
| *"No such call: …"* or the version shown differs from the page | An older runner is still running. Close its window, run `checknsync.cmd` again, and refresh the page. |
| Tabs and pages aren't tracked, and 📷 is greyed out | The browser's DevTools port is turned off (an IT policy?). Take screenshots with Win+Shift+S and paste them with Ctrl+V. |
| ⏺ Record does nothing | An IT policy may block screen capture (`getDisplayMedia`). |
| A query fails to sign in | Check that `az` is installed, then use **🔌 Test connection** in ⚙ Settings. For Entra MFA, set the tenant (for example `education.gov.uk`). |
| Excel files don't open | The Excel reader (SheetJS) loads from cdnjs. Set `window.CHECKNSYNC_XLSX` to a local copy if the CDN is blocked. |
| *"The backup folder … is inside the repo"* | Choose a backup folder outside the repo in ⚙ Settings, or leave it empty to use Downloads. |

## Runner API

`checknsync-runner.ps1` serves the dashboard and a small JSON API. Every call needs the `X-ChecknSync-Token` header.

| Area | Calls |
|---|---|
| Tests | `GET /api/tests`, `GET/POST /api/test`, `POST /api/test/remove`, `POST /api/test/move` |
| Runs | `GET /api/runs`, `GET /api/run-file` |
| Session | `POST /api/session/start`, `GET /api/session/state`, `POST /api/session/shot`, `/paste`, `/shot-image`, `/upload`, `/finish`, `/save` |
| Data | `GET /api/library`, `POST /api/query-file`, `GET /api/data-file`, `GET /api/downloads`, `GET /api/downloads/file` |
| SQL | `POST /api/sql`, `/api/sql/signin`, `/api/sql/test` |
| Backup | `GET /api/backups`, `GET /api/backup`, `GET /api/backup/file`, `POST /api/backup`, `/api/backup/restore`, `/api/backup/upload` |
| Settings | `GET /api/info`, `POST /api/config` |

---

ChecknSync Dashboard. Author: Russell Ball.
