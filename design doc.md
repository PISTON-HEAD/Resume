# IBE-Cloud Ticketing System — Design Overview

> **Repo:** [`philips-test-org/IBE-Cloud-Ticketing-System`](https://github.com/philips-test-org/IBE-Cloud-Ticketing-System)
> **Pattern:** GitHub Issue Ops — every ticket is a GitHub Issue; all routing, notifications, and configuration live in the repo itself.
> **Runtime:** GitHub Actions on `ubuntu-latest`. No Docker, no self-hosted runner, no SMTP server.

---

## 1. What We Built

An internal ticketing system for the IBE-Cloud team. Users raise a **structured GitHub Issue** using a fixed form. Automation:

- Parses the form
- Looks up the responsible owner from a config file
- Assigns the ticket + posts a confirmation comment
- Notifies the owner via GitHub's native "you were assigned" email

Everything — categories, owners, form fields, workflow logic — is **version-controlled in the repo**. Adding a category or changing an owner is a Pull Request, not a database change.

---

## 2. Repository Structure

```
IBE-Cloud-Ticketing-System/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── config.yml                              # Disables blank issues on the picker
│   │   └── cloud_ticket_request_template.yml       # THE ticket form (5 fields)
│   └── workflows/
│       ├── cloud_ticket_request_submission.yaml    # Runs on issue creation
│       └── sync-ticket-template.yaml               # Runs on config changes
├── configuration/
│   └── ticketing-request-configuration/
│       ├── ticket-categories.md                    # Source of truth: dropdown values
│       └── category-owner-mapping.md               # Source of truth: routing table
└── scripts/
    └── generate-ticket-template.js                 # Rewrites dropdown from categories.md
```

**Five files hold the whole system.** Anything you want to change is one of them.

---

## 3. The Ticket Form (What the Requester Sees)

Path: `.github/ISSUE_TEMPLATE/cloud_ticket_request_template.yml`

| Field | Type | Required | Notes |
|---|---|---|---|
| Title | Text | ✅ | Prefilled with `Ticket: ` |
| 📂 Issue Category | Dropdown | ✅ | Options come from `ticket-categories.md` |
| ⚡ Priority | Dropdown | ✅ | Low / Medium / High |
| 🌐 Environment | Dropdown | ✅ | Prod / Test / Dev / N/A |
| 🧩 Namespace | Text | ✅ | Enter `N/A` if not namespace-scoped |
| 📝 Description | Textarea | ✅ | Pre-filled placeholder with structure |

The issue automatically gets a `cloud-ticket` label from the template — that's the marker the workflow uses to filter genuine tickets from other issues.

---

## 4. Two Workflows, Two Flows

### 4.1 Flow A — Ticket Submission (`cloud_ticket_request_submission.yaml`)

**Trigger:** any new issue opened with the `cloud-ticket` label.

```
User submits ticket
   │
   ├─ Step 1: Checkout repo (needed to read the mapping file)
   ├─ Step 2: Generate short Ticket ID — <issue#>_<UTC_yyyymmddhhmmss>
   │          e.g., 47_20260702153045
   ├─ Step 3: Parse form fields (Category, Priority, Environment, Namespace)
   │          Read category-owner-mapping.md → find matching row
   │          If no match → fall back to "Other" row (visible warning)
   ├─ Step 4: Post confirmation comment on the issue
   │          (includes fallback warning if applicable)
   └─ Step 5: Assign the owner(s) to the issue
              → GitHub auto-emails the assignee(s)
```

### 4.2 Flow B — Config Sync (`sync-ticket-template.yaml`)

**Trigger:** push to `main` that touches:
- `configuration/ticketing-request-configuration/ticket-categories.md`
- `scripts/generate-ticket-template.js`
- `.github/workflows/sync-ticket-template.yaml`
- Or manual "Run workflow" click

```
Someone edits ticket-categories.md (PR merged)
   │
   ├─ Step 1: Checkout repo
   ├─ Step 2: Set up Node.js 20
   ├─ Step 3: Run scripts/generate-ticket-template.js
   │          → Reads categories from markdown
   │          → Regex-replaces ONLY the category dropdown options
   │            in cloud_ticket_request_template.yml
   │          → Leaves every other field untouched
   └─ Step 4: If the template changed → commit + push back to main
              (as github-actions[bot])
```

---

## 5. The Two Config Files

### 5.1 `ticket-categories.md` — Source of Truth for Dropdown Options

Format (a markdown table):

```markdown
## ✅ Approved Ticket Categories

| Category |
|---|
| `Certificate` |
| `Connectivity - POD` |
| `Rhapsody` |
| ...
| `Other` |
```

**Effect:** the values in the first column become the options on the `📂 Issue Category` dropdown after the sync workflow runs.

### 5.2 `category-owner-mapping.md` — Source of Truth for Routing

Format (a markdown table with 3 columns):

```markdown
## ✅ Approved Category → Owner / Email Mapping

| Category | Owner (GitHub Handle) | Notification Email |
|---|---|---|
| `Certificate` | `@gokul` | `gokul@philips.com` |
| `Connectivity - POD` | `@gokul` | `gokul@philips.com` |
| ...
| `Other` | `@a, @b, @c` | `a@philips.com, b@philips.com, c@philips.com` |
```

**Effect:** when a ticket with Category X is submitted, the workflow looks up row X and assigns the ticket to the listed GitHub handle(s). Multiple owners = comma-separated inside the backticks.

**Special row: `Other`** — this is the fallback. If a category has no dedicated row (because it was added to the dropdown but nobody added it to the mapping), the workflow routes to whoever is listed in the `Other` row.

---

## 6. How the System Handles Configuration Mistakes

| Mistake | System behavior | Any error? |
|---|---|---|
| Remove a category from `ticket-categories.md` but not from `category-owner-mapping.md` | Dropdown loses that option. Stale mapping row is just unused. | ❌ No |
| Add a category to `ticket-categories.md` but not to `category-owner-mapping.md` | Ticket routes to the `Other` row's owners. Confirmation comment shows a ⚠️ note telling the team to add the mapping. | ❌ No (graceful fallback) |
| Typo in a category name (`Certifcate` in one file, `Certificate` in the other) | Ticket falls through to `Other` because the exact-match lookup fails. Fallback ⚠️ note appears. | ❌ No (visible warning) |
| Remove the `Other` row entirely | Workflow fails on any unmapped ticket. Fails loud, easy to fix. | ✅ Only if fallback is needed |
| Category name in mapping doesn't match dropdown exactly (spaces, punctuation) | Falls through to `Other`. Visible warning. | ❌ No |

**Design principle:** the system prefers **graceful fallback with visible warnings** over hard failures. The team sees drift on real tickets and can fix it in a follow-up PR — no outages, no lost tickets.

---

## 7. How to Use the System (Cheat Sheets)

### 7.1 As a Requester (submitting a ticket)

1. Go to `https://github.com/philips-test-org/IBE-Cloud-Ticketing-System/issues/new/choose`
2. Pick **🎫 Submit a Cloud Support Ticket**
3. Fill Category, Priority, Environment, Namespace, Description
4. Click **Submit new issue**
5. Within ~30 seconds: a bot posts a confirmation comment, and the ticket is assigned to the responsible owner. You're auto-subscribed for future comments.

### 7.2 As a Maintainer — Add a New Category

1. Edit `configuration/ticketing-request-configuration/ticket-categories.md` — add a row like `` | `New Category Name` | ``
2. Edit `configuration/ticketing-request-configuration/category-owner-mapping.md` — add the matching row with owner + email
3. Commit both files in the same PR
4. On merge: the sync workflow regenerates the template. New option appears in the dropdown.

### 7.3 As a Maintainer — Remove a Category

1. Delete the row from `ticket-categories.md`
2. (Optional cleanup) delete the matching row from `category-owner-mapping.md`
3. Merge → sync workflow updates the template.

### 7.4 As a Maintainer — Change Who Owns a Category

Edit only `category-owner-mapping.md`. Change the handle / email columns. Merge. Next ticket in that category goes to the new owner. **No workflow re-run needed** — the mapping file is read live at ticket-submission time.

### 7.5 As an Owner — Handle an Assigned Ticket

1. You get a GitHub email "You were assigned to #47"
2. Open the issue on GitHub
3. Investigate, comment updates on the ticket as you go
4. When done, close the issue

---

## 8. Notifications Recap

| Event | Who gets notified | How |
|---|---|---|
| Ticket opened | Owner(s) from mapping | GitHub's native "you were assigned" email |
| Ticket opened | Requester | GitHub's native "you're watching" activity (auto-subscribed as author) |
| Comment added | Everyone watching the issue | GitHub native |
| Ticket closed | Everyone watching the issue | GitHub native |

**No custom SMTP** — everything runs off GitHub's native notifications. Simple, zero configuration, works for every Philips GitHub Enterprise user out of the box.

Custom-branded emails (from `ibe-cloud-noreply@philips.com` with rich HTML) can be added later — the workflow was designed to accommodate them as extra steps without changing the core flow.

---

## 9. Key Design Choices

| Decision | Why |
|---|---|
| Single issue template (not one per category) | Simpler for users; less to maintain |
| Category dropdown driven by markdown, not hardcoded YAML | Non-technical maintainers can change categories via PR |
| Owner routing via markdown, read at runtime | Same reason — governance-as-code, no redeploy |
| Surgical regex replace in the sync script (not full-template regeneration) | Manual template edits (adding a field, changing wording) survive future syncs |
| Fallback to `Other` on missing mapping + visible ⚠️ note | Graceful degradation; makes drift visible without breaking anything |
| GitHub native notifications only (no SMTP) | Zero setup, works from day one |
| Ubuntu-hosted runner | Free for internal repos; no Docker/infra ownership |
| Ticket ID = `<issue#>_<UTC timestamp>` | Short, sortable, unique across runs — good for reports |
| Namespace as required field with `N/A` allowed | Encourages precise reporting; still works for env-wide issues |

---

## 10. What's Not Built Yet (Future Work)

These are optional additions on top of the current system:

- **Custom HTML emails** for the owner and requester (via `dawidd6/action-send-mail` on O365 SMTP or Microsoft Graph API)
- **Closure workflow** — enforce `effort/*` label on close, lock the issue, log to a CSV
- **`/effort 2h` slash command** to close in one comment
- **Metrics dashboard** — static HTML page served by GitHub Pages, reading a CSV history file
- **Drift check** — a step in the sync workflow that warns when categories exist in the dropdown but not in the mapping file
- **Priority + Environment labels** auto-applied by the submission workflow (for filtering the Issues page)
- **Reopen policy** — auto-close reopened tickets after N days

Each of these is a self-contained addition that doesn't disturb the current design.

---

## 11. Reference: File-by-File Purpose

| File | Purpose | When to edit |
|---|---|---|
| `.github/ISSUE_TEMPLATE/config.yml` | Disables blank issue option on the picker | Rarely (only to add contact links) |
| `.github/ISSUE_TEMPLATE/cloud_ticket_request_template.yml` | The ticket form definition | For non-category changes (add a new field, change text) — category options are auto-managed |
| `.github/workflows/cloud_ticket_request_submission.yaml` | Runs on new tickets | To add new steps to the submission flow |
| `.github/workflows/sync-ticket-template.yaml` | Runs on config changes | Rarely |
| `configuration/ticketing-request-configuration/ticket-categories.md` | Category dropdown values | To add / remove / reorder categories |
| `configuration/ticketing-request-configuration/category-owner-mapping.md` | Category → owner routing | To change ownership |
| `scripts/generate-ticket-template.js` | Regenerates category dropdown from categories.md | Rarely — only if the template's category block structure changes |

---

*This document lives outside the repo (at `c:\Cloud Care\IBE Cloud Ticketing System\`) so it stays out of the codebase. Move it into the repo or keep it as an offline reference — your call.*
