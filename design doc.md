# IBE-Cloud Ticketing System — Design Overview

> **Repo:** [`philips-test-org/IBE-Cloud-Ticketing-System`](https://github.com/philips-test-org/IBE-Cloud-Ticketing-System)
> **Pattern:** GitHub Issue Ops — every ticket is a GitHub Issue; all routing, notifications, and configuration live in the repo itself.
> **Runtime:** GitHub Actions on `ubuntu-latest`. No Docker, no self-hosted runner, no SMTP server.

---

## 1. What We Built

An internal ticketing system for the IBE-Cloud team. Users raise a **structured GitHub Issue** using a fixed form. Automation:

- Parses the form (Category, Priority, Environment, Namespace, Description)
- Reads the cloud team roster from a config file
- Assigns the **entire team** to the ticket + posts a confirmation comment
- Notifies each team member via GitHub's native "you were assigned" email

Everything — categories, team members, form fields, workflow logic — is **version-controlled in the repo**. Adding a category or adding/removing a team member is a Pull Request, not a database change.

Routing is deliberately **flat**: every ticket goes to every team member. Category, Priority, and Environment are still captured on the form because they're useful for filtering the Issues page and future dashboards — they just don't drive routing.

---

## 2. Repository Structure

```
IBE-Cloud-Ticketing-System/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── config.yml                                # Disables blank issues on the picker
│   │   └── cloud_ticket_request_template.yml         # THE ticket form (5 fields)
│   └── workflows/
│       ├── cloud_ticket_request_submission.yaml      # Runs on issue creation
│       ├── cloud_ticket_more_info_request.yaml       # Runs on /details slash-command
│       ├── cloud_ticket_more_info_response.yaml      # Runs when requester replies
│       └── sync-ticket-template.yaml                 # Runs on category config changes
├── configuration/
│   └── ticketing-request-configuration/
│       ├── ticket-categories.md                      # Source of truth: dropdown values
│       └── cloud-team-members.md                     # Source of truth: who to assign / notify
└── scripts/
    └── generate-ticket-template.js                   # Regenerates dropdown from categories.md
```

**Seven files hold the whole system.** Anything you want to change is one of them.

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

## 4. The Four Workflows

### 4.1 Flow A — Ticket Submission (`cloud_ticket_request_submission.yaml`)

**Trigger:** any new issue opened with the `cloud-ticket` label.

```
User submits ticket
   │
   ├─ Step 1: Checkout repo (needed to read the team-members file)
   ├─ Step 2: Generate short Ticket ID — <issue#>_<UTC_yyyymmddhhmmss>
   │          e.g., 47_20260703140445
   ├─ Step 3: Parse form fields (Category, Priority, Environment, Namespace)
   │          Read cloud-team-members.md → extract all GitHub handles
   ├─ Step 4: Post confirmation comment on the issue
   └─ Step 5: Assign every team member to the issue
              → GitHub auto-emails each assignee
```

### 4.2 Flow B — `/details` Command (`cloud_ticket_more_info_request.yaml`)

**Trigger:** a comment on any Issue whose body starts with `/details`.

```
Cloud team member comments: /details What is the pod IP? What is the load?
   │
   ├─ Guard: comment must start with "/details" and be on an Issue (not a PR)
   ├─ Step 1: Generate Action ID — <issue#>_<UTC_yyyymmddhhmmss>
   │          e.g., 47_20260703150217
   ├─ Step 2: Verify commenter is in cloud-team-members.md
   │          → If not authorized: post rejection comment, workflow fails.
   │          → If body is empty ("/details" alone): post usage help, fail.
   ├─ Step 3: Ensure the "needs-more-info" label exists in the repo
   │          → If missing: create with color #FBCA04
   └─ Step 4: Add "needs-more-info" label + post formatted reply that
              @-mentions the requester with the questions clearly restated.
              → Requester gets 1–2 notification emails (mention + comment).
```

### 4.3 Flow C — Requester Replies (`cloud_ticket_more_info_response.yaml`)

**Trigger:** a comment on an Issue that:
- Has the `needs-more-info` label
- Was posted by the Issue's original author (the requester)
- Does NOT start with `/` (so slash commands don't count as a "reply")

```
Requester posts a comment with the requested info
   │
   ├─ Guard: all 4 conditions above hold
   ├─ Step 1: Generate Action ID — <issue#>_<UTC_yyyymmddhhmmss>
   │          e.g., 47_20260703160845
   ├─ Step 2: Load cloud team roster from cloud-team-members.md
   └─ Step 3: Remove the "needs-more-info" label
              Post a comment that @-mentions every team member so they get
              GitHub "you were mentioned" emails
```

**Repeat-safe:** the label acts as a state gate. Once removed, subsequent requester comments don't re-fire this workflow. If the team runs `/details` again, the label is re-added and the cycle repeats.

### 4.4 Flow D — Config Sync (`sync-ticket-template.yaml`)

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

## 4b. The Action ID Convention

Every workflow produces a short ID in the format `<issue#>_<UTC yyyymmddhhmmss>`:

| Workflow | ID name | Example |
|---|---|---|
| Submission | Ticket ID | `47_20260703140445` |
| `/details` command | Action ID | `47_20260703150217` |
| Requester reply | Action ID | `47_20260703160845` |

The ID appears in the workflow logs and in every bot-posted comment. Because `issue_number` stays constant across the ticket's lifetime and the timestamp changes per action, every action gets a unique, sortable reference — useful for debugging concurrent activity and future reporting.

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

### 5.2 `cloud-team-members.md` — Source of Truth for Assignment

Format (a markdown table with 2 columns):

```markdown
## ✅ Approved IBE-Cloud Team Members

| GitHub Handle | Notification Email |
|---|---|
| `GokulKrishna-Philips` | `gokulkrishnack.jaihind@philips.com` |
| `@member2` | `member2@philips.com` |
| `@member3` | `member3@philips.com` |
```

**Effect:** every ticket, regardless of Category / Priority / Environment / Namespace, is assigned to **every** GitHub handle in this table. The second column (email) is reserved for a future custom SMTP email step — the current workflow doesn't use it because GitHub's native "you were assigned" email is enough.

**Constraints:**

- Maximum 10 members (GitHub caps `assignees` at 10 per issue).
- Handles must be **real GitHub users** with at least Read access to the repo.
- Changes take effect **immediately** on the next ticket — the workflow reads this file live.

---

## 6. How the System Handles Configuration Mistakes

| Mistake | System behavior | Any error? |
|---|---|---|
| Remove a category from `ticket-categories.md` | Dropdown loses that option on next sync. Existing tickets keep the old label. | ❌ No |
| Add a category to `ticket-categories.md` without touching any other file | New option appears in the dropdown. Tickets still go to the whole team. | ❌ No |
| Remove a member from `cloud-team-members.md` | They stop being assigned to new tickets. Existing tickets keep them as assignee. | ❌ No |
| Empty the `cloud-team-members.md` table | Workflow fails loud on the next ticket ("No team members found"). Easy to fix. | ✅ Only if table is empty |
| Typo in a member's GitHub handle | GitHub silently drops that handle from `addAssignees`. The workflow logs a warning but doesn't fail. | ⚠️ Warning only |
| More than 10 members in the table | GitHub caps at 10 assignees. The workflow slices to the first 10. | ⚠️ Silent cap |
| Someone who is NOT in `cloud-team-members.md` runs `/details ...` | Bot posts an authorization-denied comment; workflow fails visibly. Nothing else changes. | ⚠️ Rejection comment |
| Someone runs `/details` with no questions after it | Bot posts a usage-example comment; workflow fails visibly. | ⚠️ Rejection comment |
| Requester tries to use `/details` | Rejected (not in cloud-team-members.md). Their comment stays but no state change. | ⚠️ Rejection comment |
| Requester replies but `needs-more-info` label not present | The requester-reply workflow silently skips. Comment goes through normally. | ❌ No |
| Team member replies while label is present | The requester-reply workflow skips (commenter != issue author). `/details` may fire if body starts with `/details`. | ❌ No |
| Multiple `/details` runs in a row on the same ticket | Each adds the label (idempotent), each posts a fresh question comment with its own Action ID. | ❌ No |

**Design principle:** the system prefers **graceful behavior with loud logs** over hard failures. Real tickets never get lost — at worst a badly-formed slash command produces a rejection comment.

---

## 7. How to Use the System (Cheat Sheets)

### 7.1 As a Requester (submitting a ticket)

1. Go to `https://github.com/philips-test-org/IBE-Cloud-Ticketing-System/issues/new/choose`
2. Pick **🎫 Submit a Cloud Support Ticket**
3. Fill Category, Priority, Environment, Namespace, Description
4. Click **Submit new issue**
5. Within ~30 seconds: a bot posts a confirmation comment, and the ticket is assigned to the whole IBE-Cloud team. You're auto-subscribed for future comments.

### 7.2 As a Maintainer — Add a New Category

1. Edit `configuration/ticketing-request-configuration/ticket-categories.md` — add a row like `` | `New Category Name` | ``.
2. Commit + merge to main.
3. On merge: the sync workflow regenerates the template. New option appears in the dropdown.

No other file needs to change — all categories route to the same team.

### 7.3 As a Maintainer — Remove a Category

1. Delete the row from `ticket-categories.md`.
2. Merge → sync workflow updates the template.

### 7.4 As a Maintainer — Add / Remove a Team Member

Edit `configuration/ticketing-request-configuration/cloud-team-members.md`. Add or remove a row. Merge. Next ticket in the repo assigns to the new roster. **No workflow re-run needed** — the file is read live at ticket-submission time.

### 7.5 As a Team Member — Handle an Assigned Ticket

1. You get a GitHub email "You were assigned to #47"
2. Open the issue on GitHub
3. Investigate. Add comments to keep others (including the requester) informed as you go.
4. When done, close the issue.

### 7.6 As a Team Member — Ask the Requester for More Info

1. Post a comment on the ticket starting with `/details`:
   ```
   /details What is the pod IP right now? What's the current load? Please attach fresh logs.
   ```
   Or multi-line:
   ```
   /details
   - What is the pod IP right now?
   - What is the current load?
   - Please attach fresh logs from the last 30 minutes.
   ```
2. Within ~30 seconds the bot posts a formatted comment @-mentioning the requester, and applies the `needs-more-info` label.
3. The requester gets a strong notification (mention email + comment email, usually merged into one).
4. Wait for their reply. When they respond (via a normal comment), the label is auto-removed and every team member is @-mentioned.
5. Repeat with another `/details ...` if you still need more info.

### 7.7 As a Requester — Reply to a `/details` Request

1. You'll get an email from `notifications@github.com` mentioning you and quoting the questions.
2. Click the link to open the ticket on GitHub.
3. Post a **normal comment** with your answers (don't use any slash command).
4. Attach logs / screenshots by dragging into the comment box.
5. Submit — within ~30 seconds the cloud team is auto-notified and the `needs-more-info` label is removed. No action needed on your side beyond commenting.

---

## 8. Notifications Recap

| Event | Who gets notified | How |
|---|---|---|
| Ticket opened | Every member listed in `cloud-team-members.md` | GitHub's native "you were assigned" email |
| Ticket opened | Requester | GitHub's native "you're watching" activity (auto-subscribed as author) |
| `/details` command used | Requester | 1–2 emails: "you were mentioned" + comment-as-author (often merged by inbox) |
| Requester replies to `/details` | All cloud team members | "You were mentioned" email from GitHub |
| Any comment added | Everyone watching the issue | GitHub native |
| Ticket closed | Everyone watching the issue | GitHub native |

**No custom SMTP** — everything runs off GitHub's native notifications. Simple, zero configuration, works for every Philips GitHub Enterprise user out of the box.

Custom-branded emails (from `ibe-cloud-noreply@philips.com` with rich HTML) can be added later — the workflows were designed to accommodate them as extra steps without changing the core flow.

---

## 9. Key Design Choices

| Decision | Why |
|---|---|
| Single issue template (not one per category) | Simpler for users; less to maintain |
| Category dropdown driven by markdown, not hardcoded YAML | Non-technical maintainers can change categories via PR |
| Flat routing: whole team assigned to every ticket | Small team; category-based routing added complexity for little gain. Anyone available picks up the ticket. |
| Team roster in markdown, read at runtime | Governance-as-code; no redeploy to add/remove a member |
| Same roster file gates `/details` authorization | One source of truth for "who's on the team". Adding someone gives them ticket assignments AND `/details` command access. |
| Slash command (`/details`) instead of just labels | Explicit, greppable, unambiguous. Easy to expand later (`/close`, `/effort`, `/escalate`…). |
| Label as state gate (`needs-more-info`) | Removes any need for external state. Prevents duplicate notifications on repeat comments. |
| Action ID `<issue#>_<UTC timestamp>` in every workflow | Sortable, uniquely traces every action even under concurrent activity. |
| Surgical regex replace in the sync script (not full-template regeneration) | Manual template edits (adding a field, changing wording) survive future syncs |
| GitHub native notifications only (no SMTP) | Zero setup, works from day one |
| Ubuntu-hosted runner | Free for internal repos; no Docker / infra ownership |
| Namespace as required field with `N/A` allowed | Encourages precise reporting; still works for env-wide issues |

---

## 10. What's Not Built Yet (Future Work)

These are optional additions on top of the current system:

- **Custom HTML emails** for owners and requesters (via `dawidd6/action-send-mail` on O365 SMTP or Microsoft Graph API)
- **Closure workflow** — enforce `effort/*` label on close, lock the issue, log to a CSV
- **`/effort 2h` slash command** to close in one comment
- **`/close`, `/escalate`, `/reassign` slash commands** — same pattern as `/details`
- **Metrics dashboard** — static HTML page served by GitHub Pages, reading a CSV history file
- **Priority + Environment labels** auto-applied by the submission workflow (for filtering the Issues page)
- **Reopen policy** — auto-close reopened tickets after N days
- **Auto-detect requester email + branded confirmation email** — optional prettier notification on submit

Each of these is a self-contained addition that doesn't disturb the current design.

---

## 11. Reference: File-by-File Purpose

| File | Purpose | When to edit |
|---|---|---|
| `.github/ISSUE_TEMPLATE/config.yml` | Disables blank issue option on the picker | Rarely (only to add contact links) |
| `.github/ISSUE_TEMPLATE/cloud_ticket_request_template.yml` | The ticket form definition | For non-category changes (add a new field, change text) — category options are auto-managed |
| `.github/workflows/cloud_ticket_request_submission.yaml` | Runs on new tickets — posts confirmation + assigns whole team | To add new steps to the submission flow |
| `.github/workflows/cloud_ticket_more_info_request.yaml` | Runs on `/details` command — adds `needs-more-info` label + @-mentions requester | To tweak the question format or allowed commands |
| `.github/workflows/cloud_ticket_more_info_response.yaml` | Runs when requester replies — removes label + @-mentions team | To tweak the info-provided notification format |
| `.github/workflows/sync-ticket-template.yaml` | Runs on category config changes | Rarely |
| `configuration/ticketing-request-configuration/ticket-categories.md` | Category dropdown values | To add / remove / reorder categories |
| `configuration/ticketing-request-configuration/cloud-team-members.md` | Team roster — who gets assigned + who can run `/details` | To add / remove a team member |
| `scripts/generate-ticket-template.js` | Regenerates category dropdown from categories.md | Rarely — only if the template's category block structure changes |

---

## 12. Labels

| Label | Created how | Purpose |
|---|---|---|
| `cloud-ticket` | Manual (one-time via GitHub UI or `gh label create`) | Applied by the issue template. Guards all workflows. Without this label, submissions get no automation. |
| `needs-more-info` | Auto-created by `cloud_ticket_more_info_request.yaml` on first `/details` | State gate. Added when team asks for more info; removed when requester replies. |
| `priority-high/medium/low` | Optional — pre-created for manual filtering | Not applied by any workflow yet. |
| `env-prod/test/dev/na` | Optional — pre-created for manual filtering | Not applied by any workflow yet. |
| `category-*` | Optional — pre-created for manual filtering | Not applied by any workflow yet. |

---

*This document lives outside the repo (at `c:\Cloud Care\IBE Cloud Ticketing System\`) so it stays out of the codebase. Move it into the repo or keep it as an offline reference — your call.*
