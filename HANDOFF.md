# Project Handoff — `project-info` Kickoff Checklist app

A self-contained brief for picking up this project in a new session. Reflects the current
`main` (the app has evolved a lot — read this, not old assumptions).

## What this is
A single-page **Kickoff Checklist** intake form for a construction company. On Submit it
duplicates a monday.com board (via a Cloudflare Worker) into an "Active Jobs" folder and writes
project fields that a **Supabase-backed Dashboard** reads. Installable **PWA**.

## Repo / deploy
- **GitHub repo:** `jakenlcc-alt/project-info`
- **Deploy:** Netlify, auto-deploys from **`main`** on push. Live: https://nlc2projectinfo.netlify.app/
- **App is ONE file:** `index.html` (self-contained inline CSS/JS, no build step) + PWA assets
  (`manifest.webmanifest`, `icon.svg`, `icon-192.png`, `icon-512.png`, `favicon-32.png`,
  `apple-touch-icon.png`).
- **⚠️ Two sessions push to `main`.** Always `git fetch origin main` and pull/rebase before
  pushing, or you'll collide. Consider consolidating to one session.

## Form sections (top to bottom)
1. **Required Info** — 11 required fields; Submit is blocked until all are filled (inline
   "Required" errors, scroll-to + focus first empty, clears on input):
   - Jobsite Name `#project` · Jobsite Address `#projectAddress`
   - Contract Amount ($) `#contractAmount` · Prevailing Wage (Yes/No) `#prevailingWage`
   - Builder Name `#builderName` · Builder Phone `#builderPhone` · Builder Email `#builderEmail`
   - Estimate # `#estimateNumber`
   - Form of Billing (**Invoice / AIA**) `#formOfBilling`
   - Estimated Start Date `#startDate` · Estimated Duration `#duration`
   - Estimated End Date `#endDatePreview` — **read-only, auto-calculated** from start + duration
     (parses days/weeks/months/years); not a required input.
2. **Contact Information:** Company Address; Superintendent, Project Manager, Accounting
   (each Name / Phone / Email on one row + Availability). *(Builder moved to Required Info.)*
3. **Overview:** Project ID # `#projectId`.
4. **Invoicing:** Billing Frequency, Payment Terms, Schedule for Billing.
5. **Paperwork:** QuickBooks Invoice Number, Procore Number; editable checklist (Completed/Not
   Complete toggle + file upload, "+ Add item"). *(Contract Number was removed — replaced by
   Contract Amount in Required Info.)*
6. **Other – Binders:** Safety Plan Binder (toggle + upload).
7. **Hand-off Meeting:** Who, Date.

Buttons: **Submit** + **Preview data** (dumps collected JSON).

## `collect()` data shape
```
contacts:   { companyAddress, builder:{name,phone,email},
              superintendent:{name,phone,email,availability}, projectManager:{…}, accounting:{…} }
overview:   { project, projectId, projectAddress, prevailingWage }
timeSchedule:{ estimatedStartDate, estimatedDuration, estimatedEndDate(computed) }
invoicing:  { estimateNumber, formOfBilling, billingFrequency, paymentTerms, schedule }
paperwork:  { contractAmount, quickbooksInvoice, procoreNumber }
officeBinderNeeds:[{item,status,files[]}], otherBinders:[…], handoffMeeting:{who,date}, submittedAt
```

## monday + Supabase flow (on Submit, in `index.html`)
- `SUBMIT_URL = https://monday-token-keeper.jake-nlcc.workers.dev` — generic monday GraphQL proxy
  (injects the token; CORS `*`; JSON `{query,variables}` → `/v2`, multipart `variables[file]` → `/v2/file`).
- `TEMPLATE_BOARD_ID = 18405661261`.
1. `findActiveJobsFolder()` → `duplicateTemplate()` — `duplicate_board_with_pulses` into the
   "Active Jobs" folder (falls back to default placement).
2. `create_item` "Project Info – <Jobsite Name>".
3. `create_update` with the full checklist summary.
4. `writeProjectFields()` — **INTEGRATION CONTRACT:** creates text columns titled **`Project ID #`**,
   **`Site Address`** (=projectAddress), **`Estimated Start`** (=startDate) and sets them; the
   Dashboard reads these **by title** to build the Supabase `projects` row, keyed on Project ID #.
   **Do not rename/remove these, or the ids `project`/`projectId`/`projectAddress`/`startDate`.**
5. best-effort: `removeResourcesColumn`, `addTextColumn` "Contact Information" + "Kick Off",
   `uploadFiles` → "Documents" file column.

## ⚠️ Testing constraint
The monday/Supabase submit path can't be exercised from the sandbox: outbound network is blocked
to everything except `github.com` (`403 host_not_allowed` for the Worker + `api.monday.com`), and
the monday MCP is unauthorized (`403 MCP_AGENT_NOT_AUTHORIZED`). All **client-side** logic
(validation, end-date calc, collect() shape) is verifiable locally; the **board-creation path is
not**. To let Claude test it: add `monday-token-keeper.jake-nlcc.workers.dev` (+ `api.monday.com`)
to the environment's network allowlist, OR enable monday AI + Public Hosted MCP. Otherwise test
via a real browser Submit and read the status line.

## Open decisions (waiting on user)
- How to **break out documents and contacts inside monday / the Dashboard** (today: one
  "Documents" column, one "Contact Information" column).
- What the **Kick Off** column should hold (currently the full summary).

## Dev notes
- Render locally to check layout (Playwright is installed):
  `npx playwright screenshot --full-page "file:///home/user/project-info/index.html" out.png`
- Verify no JS errors + validation + `collect()` shape by loading in headless chromium and
  calling `collect()` via `page.evaluate`.
- Git: `git fetch origin main` first, then commit and push to `main`.
