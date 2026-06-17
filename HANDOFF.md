# Project Handoff — `project-info` Kickoff Checklist app

A self-contained brief for picking up this project in a new session.

## What this is
A single-page **Kickoff Checklist** intake form for a construction company. On submit it
creates a project workspace in **monday.com** via a Cloudflare Worker. Built from a Word doc
("Kickoff Check list").

## Repo / branch / deploy
- **GitHub repo:** `jakenlcc-alt/project-info` (scope your session to this repo)
- **Work branch:** `claude/upbeat-mayer-uvzvne` (also fast-forward `main` and push both)
- **Deploy:** Netlify, auto-deploys from `main` on push. Live: https://nlc2projectinfo.netlify.app/
- **Whole app is ONE file:** `index.html` (self-contained, inline CSS/JS, no build step).
- **Git pattern used each change:** commit on `claude/…` → `git branch -f main HEAD` → push both.

## Form sections (top to bottom)
1. **Contact Information** (top): Builder POC (free text); Superintendent, Project Manager,
   Accounting Department — each Phone + Email.
2. **Overview:** Project, Project ID #, Builder.
3. **Time Schedule:** Estimated Start Date, Estimated Duration.
4. **Invoicing** (textarea).
5. **Resources:** Material ordering, What supplier(s), Material quotes, Equipment needs, Manpower.
6. **Additional Office Binder Needs:** 13 items, each with a green **Completed** / red
   **Not Complete** toggle (solid = selected, outline = not, tap active to clear) + a file upload.
7. **Other – Binders:** "Safety Plan Binder" (same toggle + upload).
8. **Hand-off Meeting:** Who, Date.

Buttons: **Submit** + **Preview data** (dumps collected JSON).

## monday integration (in `index.html`)
- `SUBMIT_URL = https://monday-token-keeper.jake-nlcc.workers.dev`
- `TEMPLATE_BOARD_ID = 18405661261`
- The Worker (**monday-token-keeper**) is a **generic monday GraphQL proxy**: it injects the
  monday token and forwards the POST body to `https://api.monday.com/v2`. CORS is `*`.
  - JSON body → expects `{ query, variables }` (GraphQL) → forwarded to `/v2`
  - multipart → forwarded to `/v2/file` (file uploads; use field `variables[file]`)
- On Submit the form does:
  1. `duplicate_board(board_id: TEMPLATE_BOARD_ID, duplicate_type: duplicate_board_with_structure_and_items, board_name: <Project>)` → new board
  2. `create_item(new board, "Project Info – <Project>")`
  3. `create_update(item, full checklist summary)` — guaranteed data capture
  4. **best-effort:** create long_text columns **Contact Information** + **Kick Off** and set them;
     create a file column **Documents** and `add_file_to_column` for each uploaded file.

## ⚠️ Critical blocker — monday is UNTESTED
The monday flow has **never run successfully** — Claude could not reach monday from the sandbox:
- The **sandbox network policy blocks all outbound hosts except `github.com`** (everything else
  returns `403 host_not_allowed` at the egress proxy — the Netlify site, the Worker, and
  `api.monday.com`).
- The **monday MCP connector is not authorized** (`403 MCP_AGENT_NOT_AUTHORIZED` — account admin
  must enable **monday AI** + **Public Hosted MCP**).

**To unblock (either one):**
- Add `monday-token-keeper.jake-nlcc.workers.dev` (and optionally `api.monday.com`) to **this
  environment's network allowlist** (Claude Code on the web → environment network policy; may
  need a fresh session to take effect). Then test by POSTing to the Worker.
- **OR** enable the **monday MCP** on the account (separate channel, not affected by the firewall).

**Likely things to verify/fix once reachable:** the `duplicate_board` enum, the long_text value
format (`change_simple_column_value` uses a plain string), and the multipart file upload
(`variables[file]` → `add_file_to_column`).

## Open design decisions (waiting on user)
- How to **break out documents and contacts inside monday**. Currently all docs go to one
  "Documents" column and all contacts to one "Contact Information" column; user wants them broken
  out (likely a column or sub-item per document type / per contact). **Final structure TBD.**
- What the **Kick Off** column should hold (currently the full summary).
- Confirm duplicate-board fully replaces the old single-item path (assumed yes).
- Optional: add a **Name** field to each contact (offered, not yet added).

## Dev notes
- Can't reach the live site from the sandbox (firewall). To verify layout, render `index.html`
  locally with Playwright (installed):
  `npx playwright screenshot --full-page "file:///home/user/project-info/index.html" out.png`
- Confirm no JS errors by loading the file in headless chromium and checking `pageerror`/console.

## Immediate next step
Confirm whether the network allowlist (or monday MCP) is now enabled.
- **If yes:** POST a sample kickoff through the Worker (duplicate board `18405661261` → Project
  Info item + columns), report exactly what monday returns, and fix anything broken.
- **If no:** that's the blocker to resolve first.
