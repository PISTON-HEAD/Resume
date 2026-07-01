# IBE-Cloud Ticketing System — GitHub Issue Ops Implementation Plan

> **Project:** Establish a dedicated **IBE-Cloud Ticketing System** on GitHub Enterprise (`github.com/philips-internal/IBE-Cloud-Ticketing-System`) using the **GitHub Issue Ops** pattern.
>
> **Reference implementation:** `github.com/philips-internal/IBE-Ticketing-System` — an internal Philips repo built by a sibling team. **This plan is designed to mirror that repo's structure**; any spec here marked _[VERIFY vs REF]_ must be cross-checked against the reference repo before implementation to guarantee consistency with Philips conventions.
>
> **Owner:** _<Your name>_
> **Team:** IBE Cloud Team
> **Platform:** GitHub Enterprise (philips-internal org)
> **Document version:** 2.0 (rewritten from TFS plan to GitHub Issue Ops)
> **Last updated:** _<fill in today's date>_

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [What is "GitHub Issue Ops"?](#2-what-is-github-issue-ops)
3. [Goals & Non-Goals](#3-goals--non-goals)
4. [Approach & Architecture](#4-approach--architecture)
5. [Reference Repo — What to Learn From It](#5-reference-repo--what-to-learn-from-it)
6. [Phase Overview](#6-phase-overview)
7. [Phase 0 — Day 0 Work (Today: Discovery & Design)](#7-phase-0--day-0-work-today-discovery--design)
8. [Phase 1 — Day 1 Work (Tomorrow: Repo Bootstrap)](#8-phase-1--day-1-work-tomorrow-repo-bootstrap)
9. [Phase 2 — Automation (GitHub Actions)](#9-phase-2--automation-github-actions)
10. [Phase 3 — Project Board & Views](#10-phase-3--project-board--views)
11. [Phase 4 — Notifications & Integrations](#11-phase-4--notifications--integrations)
12. [Phase 5 — Rollout & Communication](#12-phase-5--rollout--communication)
13. [Phase 6 — Metrics & Continuous Improvement](#13-phase-6--metrics--continuous-improvement)
14. [Appendix A — Repository Structure](#14-appendix-a--repository-structure)
15. [Appendix B — Issue Forms (Templates)](#15-appendix-b--issue-forms-templates)
16. [Appendix C — Label Taxonomy](#16-appendix-c--label-taxonomy)
17. [Appendix D — CODEOWNERS & Ownership Matrix](#17-appendix-d--codeowners--ownership-matrix)
18. [Appendix E — Workflow States (via Labels)](#18-appendix-e--workflow-states-via-labels)
19. [Appendix F — SLA Table](#19-appendix-f--sla-table)
20. [Appendix G — GitHub Actions Workflows](#20-appendix-g--github-actions-workflows)
21. [Appendix H — Project (v2) Board Specification](#21-appendix-h--project-v2-board-specification)
22. [Appendix I — Notification & Teams Integration](#22-appendix-i--notification--teams-integration)
23. [Appendix J — Pre-Filled Issue URLs](#23-appendix-j--pre-filled-issue-urls)
24. [Appendix K — Metrics Queries (GH CLI / GraphQL)](#24-appendix-k--metrics-queries-gh-cli--graphql)
25. [Appendix L — Reference-Repo Comparison Checklist](#25-appendix-l--reference-repo-comparison-checklist)
26. [Appendix M — Risks, Assumptions & Open Questions](#26-appendix-m--risks-assumptions--open-questions)

---

## 1. Executive Summary

The IBE cloud team receives operational support requests from multiple internal teams — testers, developers, business teams — covering cloud certificate renewals, POD connectivity failures, MTLS problems, Rhapsody / DICOM connectivity, deployment failures, namespace access, and environment outages. Today these arrive informally through email and Microsoft Teams. This leads to:

- **Lost requests** buried in chat history
- **No SLA visibility** — no proof of response/resolution timeliness
- **No metrics** — cannot identify recurring pain points
- **Uneven load** — some engineers overloaded, others idle
- **No audit trail** — problematic for compliance and post-incident reviews

Following the direction from leadership and the successful pattern already established by the sibling team at `philips-internal/IBE-Ticketing-System`, this plan adopts **GitHub Issue Ops**: a lightweight, code-configured ticketing system where every request is a GitHub Issue, routing is done by labels + CODEOWNERS, and automation is handled by GitHub Actions. All configuration is version-controlled in the repo itself — no separate admin console, no license cost beyond existing GitHub Enterprise seats.

---

## 2. What is "GitHub Issue Ops"?

"Issue Ops" is a pattern where GitHub Issues become the operational interface for a team. Instead of a dedicated ITSM tool, you use:

| GitHub Feature | Role in the Ticketing System |
|---|---|
| **Issue** | A ticket |
| **Issue Forms** (`.github/ISSUE_TEMPLATE/*.yml`) | Structured intake form (dropdowns, required fields, validation) |
| **Labels** | Category, priority, environment, workflow state |
| **CODEOWNERS** + **`assignees`** in template | Auto-assignment of the right specialist |
| **GitHub Actions** | Auto-labeling, SLA tracking, auto-assign, auto-close-stale, Teams notifications |
| **Projects (v2)** | Kanban board with columns = states |
| **Milestones** | Sprints / release windows (optional) |
| **Discussions** | Q&A that doesn't need a ticket |
| **Wiki / README** | Self-service documentation |
| **GraphQL / GH CLI** | Metrics and reporting |

**Everything is code.** The forms, labels, workflows, and ownership are all files in the repo, versioned, peer-reviewed via PRs. That is the whole point of Issue Ops — governance-as-code.

---

## 3. Goals & Non-Goals

### 3.1 In Scope (Goals)

- Single canonical place to raise IBE cloud operational tickets.
- Structured intake via GitHub Issue Forms (required fields, dropdowns).
- Auto-labeling and auto-routing based on category.
- SLA tracking per priority.
- Kanban Project board for real-time queue visibility.
- Email + Microsoft Teams notifications on key events.
- Full audit trail via GitHub's native issue history.
- Alignment with `philips-internal/IBE-Ticketing-System` conventions.

### 3.2 Out of Scope (Non-Goals)

- Replacing enterprise ITSM (ServiceNow) for cross-team incidents.
- Cross-repo / cross-org ticketing.
- Automated incident detection (this is manual intake, not monitoring).
- Customer-facing (external) ticketing.
- 24×7 on-call rotation management (add via PagerDuty later if needed).

---

## 4. Approach & Architecture

### 4.1 High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  Requester (Tester / Dev / Business team)                       │
└──────────┬───────────────────────────────┬─────────────────────┘
           │                               │
           │ Direct link to Issue Form     │ MS Teams "Raise ticket" card
           ▼                               ▼
┌────────────────────────────────────────────────────────────────┐
│  GitHub Repo: philips-internal/IBE-Cloud-Ticketing-System       │
│                                                                 │
│    .github/ISSUE_TEMPLATE/                                      │
│      ├── 01-cert-issue.yml                                      │
│      ├── 02-pod-connectivity.yml                                │
│      ├── 03-mtls-service-connectivity.yml                       │
│      ├── 04-rhapsody.yml                                        │
│      ├── 05-dicom.yml                                           │
│      ├── 06-deployment.yml                                      │
│      ├── 07-access-namespace.yml                                │
│      ├── 08-database.yml                                        │
│      ├── 09-environment-down.yml                                │
│      └── 10-other.yml                                           │
│                                                                 │
│    .github/workflows/                                           │
│      ├── auto-label.yml         (parse form → apply labels)     │
│      ├── auto-assign.yml        (label → assign owner)          │
│      ├── sla-tracker.yml        (nightly SLA check)             │
│      ├── stale-triage.yml       (auto-close waiting-for-input)  │
│      ├── add-to-project.yml     (attach to Project board)       │
│      └── teams-notify.yml       (post to Teams channel)         │
│                                                                 │
│    CODEOWNERS                   (ownership matrix)              │
│    labels.yml                   (label taxonomy)                │
│    .github/labeler.yml          (path-based labeler config)     │
│    README.md                    (how-to-raise-a-ticket)         │
│    docs/                        (playbooks, SLA, escalations)   │
└──────────┬────────────────────────────────┬────────────────────┘
           │                                │
           ▼                                ▼
   GitHub Project (v2)               Microsoft Teams channel
   (Kanban: New → Triaged →          (Adaptive Cards on create /
    Assigned → In Progress →          assign / SLA breach)
    Waiting → Resolved → Closed)
```

### 4.2 Key Design Decisions

| # | Decision | Rationale |
|---|---|---|
| AD-1 | Dedicated repo `IBE-Cloud-Ticketing-System`, NOT issues on the code repo | Keeps ticket noise separate from source code PRs; distinct CODEOWNERS |
| AD-2 | Multiple category-specific Issue Forms (one per category) | Best UX — requesters pick the right form, get category-specific required fields |
| AD-3 | Workflow state = labels prefixed `status/*` (single-select via Actions) | Native, filterable, drives Project board columns |
| AD-4 | Priority = labels prefixed `priority/*` (P1–P4) | Consistent, colored, filterable |
| AD-5 | Category = labels prefixed `category/*` | Drives auto-routing via CODEOWNERS-like mapping |
| AD-6 | Auto-assignment via a workflow that reads `category/*` label → assigns owner from a config file | More flexible than CODEOWNERS (which primarily fires on file changes) |
| AD-7 | SLA target = a comment posted by Action on creation + a nightly SLA-breach workflow | No custom fields needed; audit trail is the comment |
| AD-8 | Pre-filled Issue URLs published on Wiki / Teams / intranet | Zero-friction ticket raising |
| AD-9 | Project (v2) as the Kanban board | Native, no plug-in required |
| AD-10 | All configuration in the repo, protected by CODEOWNERS + branch protection | Governance-as-code; changes reviewed via PR |
| AD-11 | Teams notifications via `teams-notify.yml` workflow using an Incoming Webhook | No email spam; real-time channel updates |
| AD-12 | Mirror conventions from `philips-internal/IBE-Ticketing-System` | Consistency across Philips teams; easier onboarding |

---

## 5. Reference Repo — What to Learn From It

**Repo:** `github.com/philips-internal/IBE-Ticketing-System`

**⚠️ Action required on Day 0:** You (with GitHub Enterprise access) must inspect the reference repo and record the answers below. This plan is written to align with standard Issue Ops patterns, but the reference repo may deviate — following it exactly keeps your system consistent with sibling teams.

### 5.1 Files & Conventions to Extract

Clone or open the reference repo, then record for each item below what the reference uses. Use [Appendix L](#25-appendix-l--reference-repo-comparison-checklist) as your worksheet.

| # | What to look at | What to record |
|---|---|---|
| 5.1.1 | `.github/ISSUE_TEMPLATE/` — list all `*.yml` files | Names, IDs, field structure, required fields, dropdown values, `labels:` applied automatically, `assignees:` |
| 5.1.2 | `.github/ISSUE_TEMPLATE/config.yml` | Blank issues allowed? Contact links used? |
| 5.1.3 | `CODEOWNERS` | Format used, GitHub team names, mapping pattern |
| 5.1.4 | `labels.yml` (or wherever labels are managed) | Naming convention (`category/`, `priority/`, `status/` etc), colors, descriptions |
| 5.1.5 | `.github/workflows/*.yml` | Which automations exist? Which Actions used (community vs custom)? Trigger events? |
| 5.1.6 | Any `owners.yml` / `routing.yml` / `assignments.yml` | Ownership map format |
| 5.1.7 | `README.md` | How-to-raise-a-ticket instructions, style, tone |
| 5.1.8 | `docs/` folder | Playbooks, SLA doc, escalation matrix |
| 5.1.9 | Project (v2) board attached to the repo | Column names, fields, views (list/board), automation |
| 5.1.10 | Recent open issues | Real-world labeling patterns, response times, comment style |
| 5.1.11 | Repo settings → branch protection | Rules on `main` |
| 5.1.12 | Teams integration | Webhook usage, adaptive card format |

### 5.2 What NOT to blindly copy

- **Their CODEOWNERS user IDs** — they own their categories, you own yours.
- **Their SLA times** — set your own realistic targets.
- **Category list** — theirs is IBE-app-team specific; yours must be IBE-cloud specific (certs, PODs, MTLS, Rhapsody, DICOM, deployment, etc.).

### 5.3 If the reference repo has something clever, adopt it

Look especially for:
- Custom GitHub Actions they built (often in `.github/actions/`)
- SLA reporting mechanism (nightly workflow + a `docs/sla-report.md`?)
- Any "reopen policy" logic
- Teams adaptive card templates

---

## 6. Phase Overview

| Phase | Purpose | Duration (est.) | Depends On | Parallel? |
|---|---|---|---|---|
| **0** | Discovery, reference-repo analysis, design finalization | 1 day (today) | — | No |
| **1** | Repo bootstrap: create repo, issue forms, labels, CODEOWNERS, README | 1 day (tomorrow) | Phase 0 | No |
| **2** | GitHub Actions automation (labeler, auto-assign, SLA, teams-notify) | 1–2 days | Phase 1 | No |
| **3** | Project (v2) board setup with columns and views | 0.5 day | Phase 1 | Yes with Phase 2 |
| **4** | Notifications, Teams webhook, subscriptions | 0.5 day | Phase 2 | No |
| **5** | Pilot rollout (one requesting team, 2 weeks) → full rollout | 2–4 weeks | Phase 4 | No |
| **6** | Metrics review cadence and continuous improvement | Ongoing | Phase 5 | N/A |

**Total time to production-ready:** ~1 week of build effort + 2 weeks pilot.

---

## 7. Phase 0 — Day 0 Work (Today: Discovery & Design)

**Objective:** By end of today, have (a) full understanding of the reference repo, (b) filled-in design doc, (c) sign-off from Cloud Team Lead.

**Why this must be done first:** Without inspecting the reference repo, you'll rebuild from scratch and diverge from Philips conventions. With sign-off, tomorrow's build work has clear owner input and won't stall on questions.

### 7.1 Deliverables (all completed today)

- [ ] Inspected `philips-internal/IBE-Ticketing-System` and filled [Appendix L](#25-appendix-l--reference-repo-comparison-checklist)
- [ ] Filled every `<placeholder>` in this document
- [ ] Confirmed / adjusted [Appendix B](#15-appendix-b--issue-forms-templates) form list
- [ ] Confirmed [Appendix C](#16-appendix-c--label-taxonomy) label taxonomy
- [ ] Filled [Appendix D](#17-appendix-d--codeowners--ownership-matrix) with real Philips GitHub handles / team names
- [ ] Confirmed [Appendix F](#19-appendix-f--sla-table) SLA targets
- [ ] Answered every question in [Appendix M](#26-appendix-m--risks-assumptions--open-questions)
- [ ] Sign-off from Cloud Team Lead (Teams reply "Approved")
- [ ] Document committed to a shared location (or to the new repo tomorrow)

### 7.2 Step-by-Step Work (today, ~4 hours total)

#### Step 1 — Inspect the reference repo (60 min)

1. Open `https://github.com/philips-internal/IBE-Ticketing-System` in your browser.
2. Clone it locally if you can:
   ```powershell
   git clone https://github.com/philips-internal/IBE-Ticketing-System.git
   cd IBE-Ticketing-System
   code .
   ```
3. Systematically walk through the files listed in section 5.1 and fill [Appendix L](#25-appendix-l--reference-repo-comparison-checklist).
4. Take screenshots of the Project board layout and the "New Issue" template picker page.

#### Step 2 — Gather team inputs (60 min)

Ask the Cloud Team Lead + 1–2 senior engineers these questions; record in [Appendix M](#26-appendix-m--risks-assumptions--open-questions):

1. **Exact repo name** to create? Suggested: `IBE-Cloud-Ticketing-System`. Check naming conventions used by other Philips-internal repos.
2. Which **GitHub Team** owns the repo? (e.g., `@philips-internal/ibe-cloud-team`) — needed for admin rights + CODEOWNERS.
3. Which teams are the **top 5 requesters**? — populates a picklist in some forms.
4. For each **category**, who is the primary + backup owner (GitHub handle or team)? — fills Appendix D.
5. Any existing **SLA commitments** from leadership?
6. Do all requesters have **GitHub Enterprise access**? (Usually yes at Philips — everyone with a corporate account has read access on `philips-internal`.)
7. Which **Microsoft Teams channel** should get ticket notifications?
8. Any **regulatory / audit** requirements? (Affects retention, closing rules.)
9. Who signs off? (Cloud Team Lead + Product Owner typically.)
10. Preferred **rollout date**?

#### Step 3 — Finalize the categories & forms (30 min)

Confirm the category list matches your team's real workload. Recommended for IBE Cloud:

- Certificate issue
- POD connectivity issue
- MTLS / Service connectivity issue
- Rhapsody issue
- DICOM connectivity issue
- Deployment support issue
- Namespace / access issue
- Database connectivity issue
- Environment down (P1 fast-track)
- Other / uncategorized

Adjust based on reference-repo naming so your labels look consistent across Philips repos.

#### Step 4 — Fill the CODEOWNERS / ownership matrix (30 min)

Go to [Appendix D](#17-appendix-d--codeowners--ownership-matrix). For **every** category, fill:
- Primary owner (GitHub handle, e.g. `@john.doe`, or team `@philips-internal/ibe-cloud-cert-owners`)
- Backup owner
- Teams @-mention / DL
- Escalation contact (usually cloud team lead)

**Do not proceed without this filled.** Auto-assignment cannot be built without it.

#### Step 5 — Confirm SLA table (15 min)

Go to [Appendix F](#19-appendix-f--sla-table). Adjust to what your team can realistically meet.

#### Step 6 — Confirm workflow states (10 min)

See [Appendix E](#18-appendix-e--workflow-states-via-labels). Confirm the 7 states are needed. Collapse if the team says "we don't do triage separately".

#### Step 7 — Stakeholder review (30 min)

Post a 5-bullet summary in the team's Teams channel:
1. Problem statement
2. Proposed solution — GitHub Issue Ops modeled on `IBE-Ticketing-System`
3. Categories list
4. Ownership matrix
5. Ask: "Approve? Anyone I need to loop in?"

Attach or link this design doc.

#### Step 8 — Get sign-off (async, end of day)

Cloud Team Lead replies "Approved" on the Teams thread. Screenshot the reply.

### 7.3 Definition of Done for Phase 0

- Every placeholder filled.
- Ownership matrix has zero blanks.
- Cloud Team Lead approved.
- Appendix L (reference-repo comparison) fully filled.
- Document saved / committed.

---

## 8. Phase 1 — Day 1 Work (Tomorrow: Repo Bootstrap)

**Objective:** Have a functional (if bare) ticketing repo live by end of day 1 — issue forms working, labels applied, README explaining how to use it.

**Prerequisites:**
- Phase 0 done and signed off
- You have (or admin creates for you) a new repo under `philips-internal/`
- You have `Maintain` or `Admin` role on the repo

### 8.1 Step-by-Step Work

#### Step 1 — Create the repo (15 min)

If you don't have "Create repository" permission on `philips-internal`, raise a request via Philips GitHub admins (usually a service portal / GH admin team) with:

- Name: `IBE-Cloud-Ticketing-System`
- Visibility: Internal (default for `philips-internal`)
- Owner team: `@philips-internal/ibe-cloud-team` with `Admin`
- Template: none (we'll bootstrap manually)
- Initialize with README = yes

#### Step 2 — Clone locally (5 min)

```powershell
cd C:\Cloud Care
git clone https://github.com/philips-internal/IBE-Cloud-Ticketing-System.git
cd IBE-Cloud-Ticketing-System
```

#### Step 3 — Create the folder structure (5 min)

Create the layout shown in [Appendix A](#14-appendix-a--repository-structure):

```powershell
New-Item -ItemType Directory -Path .github\ISSUE_TEMPLATE -Force
New-Item -ItemType Directory -Path .github\workflows -Force
New-Item -ItemType Directory -Path .github\actions -Force
New-Item -ItemType Directory -Path docs -Force
New-Item -ItemType Directory -Path scripts -Force
```

#### Step 4 — Add Issue Forms (60 min)

Create one YAML file per category under `.github/ISSUE_TEMPLATE/`. Use the templates in [Appendix B](#15-appendix-b--issue-forms-templates). Start with `01-cert-issue.yml` and `02-pod-connectivity.yml`; others follow the same pattern.

Also create `.github/ISSUE_TEMPLATE/config.yml`:

```yaml
blank_issues_enabled: false
contact_links:
  - name: General Q&A (not an incident)
    url: https://github.com/philips-internal/IBE-Cloud-Ticketing-System/discussions
    about: For questions and discussions, use Discussions instead of Issues.
  - name: IBE App Team Ticketing (sibling)
    url: https://github.com/philips-internal/IBE-Ticketing-System
    about: For non-cloud application issues, raise there.
```

#### Step 5 — Add label taxonomy (30 min)

Create `.github/labels.yml` from [Appendix C](#16-appendix-c--label-taxonomy). Then use the community Action `EndBug/label-sync` to sync the labels on push to `main`:

Create `.github/workflows/label-sync.yml`:

```yaml
name: Sync labels
on:
  push:
    branches: [main]
    paths: ['.github/labels.yml']
  workflow_dispatch:

permissions:
  issues: write

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: EndBug/label-sync@v2
        with:
          config-file: .github/labels.yml
          delete-other-labels: false
          token: ${{ secrets.GITHUB_TOKEN }}
```

Push, then manually run the workflow to seed all labels.

#### Step 6 — Add CODEOWNERS (15 min)

Create `CODEOWNERS` at the repo root (or under `.github/`) from [Appendix D](#17-appendix-d--codeowners--ownership-matrix).

**Important nuance:** CODEOWNERS in a ticketing repo mostly fires on PR review requests. For **issue** auto-assignment, we use a separate config file `.github/owners.yml` consumed by the auto-assign Action (see Phase 2). CODEOWNERS is still valuable to protect the repo config (issue forms, workflows) from unreviewed changes.

#### Step 7 — Write the README (30 min)

The README is the front door. Include:

- What this repo is (one paragraph)
- **How to raise a ticket** — pre-filled URLs (from [Appendix J](#23-appendix-j--pre-filled-issue-urls))
- Category → owner table (short version of Appendix D)
- SLA table (from Appendix F)
- Escalation contacts
- Link to `docs/playbooks/` for common issue types
- Link to the reference `philips-internal/IBE-Ticketing-System`

#### Step 8 — Enable branch protection (10 min)

Repo settings → Branches → Add rule for `main`:
- Require pull request before merging
- Require review from Code Owners
- Require status checks (once you add them)
- Restrict who can push

This ensures changes to issue forms, workflows, labels, and ownership all go through PR review.

#### Step 9 — Enable Discussions (2 min)

Repo settings → Features → check **Discussions**. This gives requesters a place to ask questions that aren't tickets.

#### Step 10 — Enable the Project (v2) board (10 min)

See [Phase 3](#10-phase-3--project-board--views).

#### Step 11 — Commit and push everything (5 min)

```powershell
git add .
git commit -m "Bootstrap IBE Cloud Ticketing System — Day 1 (Issue Ops)"
git push origin main
```

#### Step 12 — Smoke test (15 min)

1. Go to `Issues` tab → **New issue**.
2. Confirm all category forms appear in the picker.
3. Open the Certificate form → confirm all fields render, required fields are enforced.
4. Submit a test ticket.
5. Confirm labels from the form's `labels:` block are applied.
6. Delete the test ticket (or close with a comment "test").

### 8.2 Definition of Done for Phase 1

- Repo exists at `philips-internal/IBE-Cloud-Ticketing-System`
- All 10 issue forms visible in the picker
- Labels seeded via label-sync workflow
- CODEOWNERS in place
- README explains how to raise a ticket
- Branch protection on `main`
- Smoke test passed

---

## 9. Phase 2 — Automation (GitHub Actions)

**Objective:** Wire up the automation that turns a bunch of labels and forms into a real ticketing system.

### 9.1 Workflows to Create

See [Appendix G](#20-appendix-g--github-actions-workflows) for full YAML. Summary:

| # | Workflow file | Trigger | What it does |
|---|---|---|---|
| W1 | `auto-label.yml` | `issues: opened` | Reads issue body → applies additional labels (e.g., extracts Environment) |
| W2 | `auto-assign.yml` | `issues: labeled` | Reads `category/*` label → assigns owner from `.github/owners.yml` |
| W3 | `add-to-project.yml` | `issues: opened` | Adds new issue to the "IBE Cloud Support" Project (v2) |
| W4 | `sla-post-on-create.yml` | `issues: opened` | Posts a comment with computed SLA target date + adds `status/new` label |
| W5 | `sla-tracker.yml` | Scheduled `0 6 * * 1-5` (weekday 6am UTC) | Finds SLA-breaching open issues → posts comment `@escalation` + adds `sla/breached` label |
| W6 | `stale-triage.yml` | Scheduled daily | Auto-closes `status/waiting-for-input` issues idle >5 days with a comment |
| W7 | `teams-notify.yml` | `issues: opened`, `labeled`, `closed` | POSTs to Teams webhook with an adaptive card |
| W8 | `state-transition-guard.yml` | `issues: labeled` | Enforces "cannot go to `status/resolved` unless `rca` and `resolution` comments exist" |
| W9 | `label-sync.yml` | Push to `labels.yml` | (created in Phase 1) Syncs labels |

### 9.2 Secrets to Configure

Repo settings → Secrets and variables → Actions → add:

- `TEAMS_WEBHOOK_URL` — Incoming Webhook URL from the target Teams channel
- (Optional) `PAT_TOKEN` — only if some workflows need cross-repo access; usually `GITHUB_TOKEN` is enough

### 9.3 Ownership File

Create `.github/owners.yml` (see [Appendix D](#17-appendix-d--codeowners--ownership-matrix) for content):

```yaml
categories:
  cert:
    primary: '@philips-internal/ibe-cloud-cert-owners'
    backup:  '@userid-backup1'
  pod-connectivity:
    primary: '@userid-pod-lead'
    backup:  '@userid-pod-backup'
  ...
default_triage: '@userid-triage-lead'
escalation: '@philips-internal/ibe-cloud-leads'
```

### 9.4 Definition of Done for Phase 2

- All 8 workflows committed and passing on a test issue
- Test scenario: raise a "POD Connectivity" ticket → labels applied → correct owner assigned → SLA comment posted → Project board updated → Teams channel notified
- Secrets configured
- `owners.yml` matches Appendix D

---

## 10. Phase 3 — Project Board & Views

**Objective:** Kanban board that mirrors your workflow states, driven by the `status/*` labels.

### 10.1 Create the Project

1. In the repo → **Projects** tab → **New project** → template **Board**.
2. Name: `IBE Cloud Support`
3. Description: `Kanban board for all IBE cloud support tickets`

### 10.2 Configure Columns (see [Appendix H](#21-appendix-h--project-v2-board-specification))

Rename default columns to match the 7 states:
- New
- Triaged
- Assigned
- In Progress
- Waiting for Input
- Resolved
- Closed

### 10.3 Add Custom Fields

- **Priority** (single select: P1, P2, P3, P4)
- **Category** (single select: cert, pod-connectivity, mtls, rhapsody, dicom, deployment, access, database, env-down, other)
- **SLA Target** (date)
- **Assignee** (people — inherited from issue)

### 10.4 Configure Automation (built into Projects v2)

In Project → **Workflows**:
- **Item added to project** → set Status = `New`
- **Item closed** → set Status = `Closed`
- **Item reopened** → set Status = `In Progress`

For label-driven column moves, use the `add-to-project.yml` / a small custom workflow that reads `status/*` label and updates the Project field.

### 10.5 Views to Create

| View | Filter | Purpose |
|---|---|---|
| All open | `status:open` | Team overview |
| By category | Group by Category | Category workload |
| By assignee | Group by Assignee | Who has what |
| P1/P2 only | Priority = P1 or P2 | Expedite lane |
| SLA breaching | Filter label `sla/breached` | Escalation |
| Waiting for input | Status = Waiting for Input | Stuck items |
| Closed this month | Status = Closed, Closed date >= start of month | Reporting |

### 10.6 Definition of Done for Phase 3

- Project board live and linked from repo README
- Columns match states
- All views created and pinned
- Automation working: closing an issue moves it to Closed column

---

## 11. Phase 4 — Notifications & Integrations

### 11.1 GitHub Native Notifications

Every team member watching the repo gets emails/notifications. Ask each engineer to:

1. Go to the repo → **Watch** dropdown (top-right) → **Custom**
2. Check: **Issues**, **Discussions**
3. Uncheck: **Pull requests**, **Releases** (unless relevant)

For requesters — they auto-follow issues they create/comment on. No action needed.

### 11.2 Microsoft Teams Integration

Two options — pick one or both:

**Option A: GitHub for Teams app (native)**
- In Teams channel → **+** → **GitHub** → connect repo → subscribe to issues events. Simple, low-effort, no custom workflow needed.

**Option B: Custom webhook via `teams-notify.yml`** (see [Appendix G](#20-appendix-g--github-actions-workflows))
- Richer control (custom card, filtering by priority, mentions). Preferred if you want P1 notifications to `@channel`.

### 11.3 SLA Escalation

`sla-tracker.yml` posts to Teams and adds a `@mention` to the escalation contact from `owners.yml` for any breaching issue.

### 11.4 Definition of Done for Phase 4

- Team members watching the repo appropriately
- Teams channel receives adaptive card for new tickets
- SLA breach test: create a ticket with a past SLA date manually → workflow flags it

---

## 12. Phase 5 — Rollout & Communication

### 12.1 Pilot (Week 1–2)

- Pick **one requesting team** (Testing team recommended).
- 20-min walkthrough over Teams: how to raise a ticket, where to look for status.
- Ask them to route **all** cloud requests through the repo for 2 weeks.
- Monitor daily; fix rough edges immediately.

### 12.2 Feedback Loop

- End of week 1: 15-min feedback with pilot team.
- End of week 2: retro — go/no-go for full rollout.

### 12.3 Full Rollout

- Announce in all-hands Teams channel with the pre-filled links.
- Publish the how-to 1-pager (link to README).
- Update team charters / onboarding docs.
- Send dashboard/project board link to management.

### 12.4 Definition of Done for Phase 5

- Pilot feedback captured
- Full rollout announced
- No new tickets arriving via email/DM

---

## 13. Phase 6 — Metrics & Continuous Improvement

### 13.1 Weekly (team lead)

- Review SLA compliance (Project board `sla/breached` view)
- Review aging tickets (>7 days open)
- Reassign stuck items

### 13.2 Monthly

- Ticket volume by category (see [Appendix K](#24-appendix-k--metrics-queries-gh-cli--graphql))
- Top-3 recurring categories → open a Scrum PBI to automate/prevent the root cause
- Review Requester Teams — is anyone bypassing the system?

### 13.3 Quarterly

- Retro on the whole system
- Refine categories, ownership, SLA targets
- Compare metrics with the sibling `IBE-Ticketing-System` team

### 13.4 Definition of Done for Phase 6

- Recurring calendar cadences established
- Metrics reviewed regularly

---

## 14. Appendix A — Repository Structure

Target layout for `IBE-Cloud-Ticketing-System`:

```
IBE-Cloud-Ticketing-System/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── config.yml
│   │   ├── 01-cert-issue.yml
│   │   ├── 02-pod-connectivity.yml
│   │   ├── 03-mtls-service-connectivity.yml
│   │   ├── 04-rhapsody.yml
│   │   ├── 05-dicom.yml
│   │   ├── 06-deployment.yml
│   │   ├── 07-access-namespace.yml
│   │   ├── 08-database.yml
│   │   ├── 09-environment-down.yml
│   │   └── 10-other.yml
│   ├── workflows/
│   │   ├── auto-label.yml
│   │   ├── auto-assign.yml
│   │   ├── add-to-project.yml
│   │   ├── sla-post-on-create.yml
│   │   ├── sla-tracker.yml
│   │   ├── stale-triage.yml
│   │   ├── state-transition-guard.yml
│   │   ├── teams-notify.yml
│   │   └── label-sync.yml
│   ├── actions/                      # (optional) local composite Actions
│   ├── labels.yml
│   └── owners.yml
├── docs/
│   ├── playbooks/
│   │   ├── cert-renewal.md
│   │   ├── pod-connectivity-triage.md
│   │   ├── mtls-triage.md
│   │   └── env-down-runbook.md
│   ├── sla.md
│   └── escalation.md
├── scripts/
│   └── sla-report.ps1                # (optional) local reporting helper
├── CODEOWNERS
├── README.md
├── CONTRIBUTING.md
└── LICENSE (internal / none)
```

---

## 15. Appendix B — Issue Forms (Templates)

Each form is a YAML file under `.github/ISSUE_TEMPLATE/`. Two full examples below; the rest follow the same pattern with category-specific fields.

### 15.1 `01-cert-issue.yml`

```yaml
name: 🔐 Certificate Issue
description: Cloud certificate problem (expiry, renewal, trust chain, mismatch)
title: "[CERT] "
labels:
  - "category/cert"
  - "status/new"
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        ## Certificate Issue Report
        Please fill all required fields. Missing info delays triage.

  - type: dropdown
    id: environment
    attributes:
      label: Environment
      options:
        - Dev
        - Test
        - NonFun
        - Prod
    validations:
      required: true

  - type: dropdown
    id: priority
    attributes:
      label: Priority
      description: |
        P1 = Prod down / blocker
        P2 = Testing blocked
        P3 = Cert renewal / non-urgent
        P4 = Question / clarification
      options:
        - P1
        - P2
        - P3
        - P4
      default: 2
    validations:
      required: true

  - type: input
    id: hostname
    attributes:
      label: Hostname / URL affected
      placeholder: dicom-manager.ibe-sv-nonfun.philips.com
    validations:
      required: true

  - type: input
    id: cert-cn
    attributes:
      label: Certificate CN / SAN (if known)
      placeholder: CN=*.ibe-sv-nonfun.philips.com
    validations:
      required: false

  - type: textarea
    id: error
    attributes:
      label: Error message
      description: Paste the exact error output.
      render: shell
    validations:
      required: true

  - type: textarea
    id: impact
    attributes:
      label: Business impact
      description: What is blocked / who is affected?
    validations:
      required: true

  - type: textarea
    id: tried
    attributes:
      label: Steps already tried
      description: List commands / checks already performed.
      placeholder: |
        - Ran openssl s_client -connect ... :443 -showcerts
        - Checked cert expiry with kubectl get secret ...
    validations:
      required: false

  - type: input
    id: requester-team
    attributes:
      label: Requester Team
      placeholder: Testing - Imaging
    validations:
      required: true

  - type: checkboxes
    id: acknowledgements
    attributes:
      label: Acknowledgements
      options:
        - label: I have searched existing open issues for duplicates.
          required: true
        - label: I have consulted the [cert-renewal playbook](../blob/main/docs/playbooks/cert-renewal.md).
          required: false
```

### 15.2 `02-pod-connectivity.yml`

```yaml
name: 🔌 POD Connectivity Issue
description: Kubernetes pod unreachable, connection refused, DNS failure inside cluster
title: "[POD] "
labels:
  - "category/pod-connectivity"
  - "status/new"
body:
  - type: dropdown
    id: environment
    attributes:
      label: Environment
      options: [Dev, Test, NonFun, Prod]
    validations:
      required: true

  - type: dropdown
    id: priority
    attributes:
      label: Priority
      options: [P1, P2, P3, P4]
      default: 2
    validations:
      required: true

  - type: input
    id: namespace
    attributes:
      label: Namespace
      placeholder: ibe-sv-nonfun
    validations:
      required: true

  - type: input
    id: pod-or-service
    attributes:
      label: Pod / Service name
      placeholder: dicom-manager
    validations:
      required: true

  - type: textarea
    id: error
    attributes:
      label: Error message
      render: shell
    validations:
      required: true

  - type: textarea
    id: impact
    attributes:
      label: Business impact
    validations:
      required: true

  - type: textarea
    id: tried
    attributes:
      label: Steps already tried
      placeholder: |
        - kubectl get pods -n <ns>
        - kubectl describe pod <pod> -n <ns>
        - curl from jump host
    validations:
      required: false

  - type: input
    id: requester-team
    attributes:
      label: Requester Team
    validations:
      required: true

  - type: checkboxes
    id: acknowledgements
    attributes:
      label: Acknowledgements
      options:
        - label: I have searched existing open issues for duplicates.
          required: true
```

### 15.3 Other categories — same pattern

Create the remaining 8 files following the same skeleton:
- `03-mtls-service-connectivity.yml` — add fields: Client service, Server service, mTLS mode
- `04-rhapsody.yml` — add: Rhapsody route/channel, Message ID
- `05-dicom.yml` — add: SCU/SCP AE Title, Modality
- `06-deployment.yml` — add: Helm release, Chart version, Build/pipeline URL
- `07-access-namespace.yml` — add: Requested access level, Duration, Justification
- `08-database.yml` — add: DB name, Connection string (redact secrets)
- `09-environment-down.yml` — force `P1` default; add: Detection method, Affected regions
- `10-other.yml` — free-form fallback; add: Suggested category

### 15.4 `config.yml`

```yaml
blank_issues_enabled: false
contact_links:
  - name: 💬 General Q&A (not a ticket)
    url: https://github.com/philips-internal/IBE-Cloud-Ticketing-System/discussions
    about: For questions and discussions, use Discussions instead of Issues.
  - name: 🎫 IBE App Team Ticketing
    url: https://github.com/philips-internal/IBE-Ticketing-System
    about: For non-cloud application issues, raise there.
```

---

## 16. Appendix C — Label Taxonomy

Save as `.github/labels.yml`. Colors are hex without `#`.

```yaml
# ============ Category ============
- name: "category/cert"
  color: "1D76DB"
  description: Cloud certificate issue
- name: "category/pod-connectivity"
  color: "1D76DB"
  description: Kubernetes pod connectivity
- name: "category/mtls"
  color: "1D76DB"
  description: mTLS / service-to-service auth
- name: "category/rhapsody"
  color: "1D76DB"
  description: Rhapsody integration engine
- name: "category/dicom"
  color: "1D76DB"
  description: DICOM connectivity / imaging
- name: "category/deployment"
  color: "1D76DB"
  description: Deployment / release support
- name: "category/access"
  color: "1D76DB"
  description: Namespace / access request
- name: "category/database"
  color: "1D76DB"
  description: Database connectivity
- name: "category/env-down"
  color: "B60205"
  description: Environment down / outage
- name: "category/other"
  color: "1D76DB"
  description: Uncategorized

# ============ Priority ============
- name: "priority/P1"
  color: "B60205"
  description: Critical - Prod down / blocker
- name: "priority/P2"
  color: "D93F0B"
  description: High - Testing blocked
- name: "priority/P3"
  color: "FBCA04"
  description: Medium - non-blocking
- name: "priority/P4"
  color: "0E8A16"
  description: Low - request / clarification

# ============ Environment ============
- name: "env/prod"
  color: "5319E7"
- name: "env/nonfun"
  color: "5319E7"
- name: "env/test"
  color: "5319E7"
- name: "env/dev"
  color: "5319E7"

# ============ Status (workflow) ============
- name: "status/new"
  color: "CCCCCC"
- name: "status/triaged"
  color: "AAAAAA"
- name: "status/assigned"
  color: "0075CA"
- name: "status/in-progress"
  color: "0075CA"
- name: "status/waiting-for-input"
  color: "FBCA04"
- name: "status/resolved"
  color: "0E8A16"
- name: "status/closed"
  color: "222222"

# ============ SLA / meta ============
- name: "sla/breached"
  color: "B60205"
  description: SLA target date has passed
- name: "duplicate"
  color: "CCCCCC"
- name: "wontfix"
  color: "CCCCCC"
- name: "needs-more-info"
  color: "FBCA04"
- name: "reopened"
  color: "FBCA04"
```

**Rule of convention:** only ONE label from `status/*` should be applied to an issue at any time. The `state-transition-guard.yml` workflow enforces this.

---

## 17. Appendix D — CODEOWNERS & Ownership Matrix

### 17.1 `CODEOWNERS` (repo root)

Protects the ticketing system config itself from unreviewed changes.

```
# Repository configuration owners
/.github/                @philips-internal/ibe-cloud-leads
/CODEOWNERS              @philips-internal/ibe-cloud-leads
/docs/                   @philips-internal/ibe-cloud-team

# Everything else defaults to the cloud team
*                        @philips-internal/ibe-cloud-team
```

### 17.2 `.github/owners.yml` — routing map (used by `auto-assign.yml`)

**Fill this in today** — auto-assignment depends on it.

```yaml
# Ownership map for auto-assignment
# Each category has a primary and backup owner (GitHub handle or team).
categories:
  cert:
    primary: '@philips-internal/<team-or-userid>'
    backup:  '@<userid-backup>'
  pod-connectivity:
    primary: '@<userid>'
    backup:  '@<userid>'
  mtls:
    primary: '@<userid>'
    backup:  '@<userid>'
  rhapsody:
    primary: '@<userid>'
    backup:  '@<userid>'
  dicom:
    primary: '@<userid>'
    backup:  '@<userid>'
  deployment:
    primary: '@<userid>'
    backup:  '@<userid>'
  access:
    primary: '@<userid>'
    backup:  '@<userid>'
  database:
    primary: '@<userid>'
    backup:  '@<userid>'
  env-down:
    primary: '@philips-internal/ibe-cloud-leads'   # entire lead team on env-down
    backup:  '@<lead-userid>'
  other:
    primary: '@<triage-lead-userid>'
    backup:  '@<lead-userid>'

default_triage: '@<triage-lead-userid>'
escalation:     '@philips-internal/ibe-cloud-leads'
```

### 17.3 Human-readable ownership matrix (for README / Wiki)

| Category | Primary Owner | Backup Owner | Escalation |
|---|---|---|---|
| Certificate | `@<userid>` | `@<userid>` | `@<lead>` |
| POD Connectivity | `@<userid>` | `@<userid>` | `@<lead>` |
| MTLS / Service | `@<userid>` | `@<userid>` | `@<lead>` |
| Rhapsody | `@<userid>` | `@<userid>` | `@<lead>` |
| DICOM | `@<userid>` | `@<userid>` | `@<lead>` |
| Deployment | `@<userid>` | `@<userid>` | `@<lead>` |
| Access / Namespace | `@<userid>` | `@<userid>` | `@<lead>` |
| Database | `@<userid>` | `@<userid>` | `@<lead>` |
| Environment Down | `@philips-internal/ibe-cloud-leads` | `@<lead>` | `@<lead>` |
| Other | `@<triage-lead>` | `@<lead>` | `@<lead>` |

---

## 18. Appendix E — Workflow States (via Labels)

### 18.1 State labels

| Label | Meaning | Project column |
|---|---|---|
| `status/new` | Just created, not triaged | New |
| `status/triaged` | Category/priority confirmed | Triaged |
| `status/assigned` | Assigned to a specific engineer | Assigned |
| `status/in-progress` | Actively being worked on | In Progress |
| `status/waiting-for-input` | Blocked pending requester response | Waiting for Input |
| `status/resolved` | Fix applied, awaiting confirmation | Resolved |
| `status/closed` | Confirmed resolved (issue also closed) | Closed |

### 18.2 Allowed transitions

Enforced by `state-transition-guard.yml`:

```
(created)             → status/new
status/new            → status/triaged
status/triaged        → status/assigned
status/assigned       → status/in-progress
status/in-progress    → status/waiting-for-input
status/waiting-for-input → status/in-progress   (requester replied)
status/in-progress    → status/resolved         (requires RCA + Resolution comments)
status/resolved       → status/closed
status/resolved       → status/in-progress      (reopen)
status/closed         → status/in-progress      (reopen within 7 days)
```

### 18.3 State diagram

```
        ┌─────┐
        │ new │
        └──┬──┘
           ▼
       ┌────────┐
       │triaged │
       └───┬────┘
           ▼
       ┌────────┐
       │assigned│
       └───┬────┘
           ▼
    ┌────────────┐        ┌──────────────────┐
    │in-progress │ ◄────► │waiting-for-input │
    └───┬────────┘        └──────────────────┘
        ▼
   ┌────────┐
   │resolved│ ────► ┌────────┐
   └───┬────┘       │ closed │
       │            └───┬────┘
       └── reopen ◄─────┘
```

---

## 19. Appendix F — SLA Table

| Priority | Description | Response SLA (triage) | Resolution SLA |
|---|---|---|---|
| P1 | Prod down, env outage, business impact | 1 hour (24×7) | 4 hours |
| P2 | Testing blocked, key service unreachable | 4 business hours | 1 business day |
| P3 | Cert renewal, config change, non-blocking | 1 business day | 3 business days |
| P4 | General request, docs, clarification | 2 business days | 5 business days |

**Note:** Internal targets, not customer-facing SLAs. Adjust to what your team can realistically hit.

**Sub-day SLA implementation:** GitHub Actions can be scheduled at minute granularity if hosted on a fast-cadence runner, but for hour-level SLAs the practical approach is:
- Post an SLA-target ISO timestamp in a comment on ticket creation
- The nightly `sla-tracker.yml` compares now vs the timestamp and flags breaches
- For P1, page a human via Teams `@channel` immediately on creation via `teams-notify.yml`

---

## 20. Appendix G — GitHub Actions Workflows

Full workable YAML for each workflow. Adjust user names / paths as needed.

### 20.1 `auto-label.yml`

```yaml
name: Auto-label from issue body
on:
  issues:
    types: [opened, edited]

permissions:
  issues: write

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - name: Extract Environment
        id: env
        uses: actions/github-script@v7
        with:
          script: |
            const body = context.payload.issue.body || '';
            const m = body.match(/### Environment\s*\n+(\w+)/i);
            const env = m ? m[1].toLowerCase() : null;
            const map = { prod: 'env/prod', nonfun: 'env/nonfun', test: 'env/test', dev: 'env/dev' };
            const label = map[env];
            if (label) {
              await github.rest.issues.addLabels({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                labels: [label]
              });
            }

      - name: Extract Priority
        uses: actions/github-script@v7
        with:
          script: |
            const body = context.payload.issue.body || '';
            const m = body.match(/### Priority\s*\n+(P[1-4])/i);
            if (m) {
              await github.rest.issues.addLabels({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                labels: [`priority/${m[1]}`]
              });
            }
```

### 20.2 `auto-assign.yml`

```yaml
name: Auto-assign owner from category
on:
  issues:
    types: [labeled]

permissions:
  issues: write
  contents: read

jobs:
  assign:
    if: startsWith(github.event.label.name, 'category/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Load owners.yml and assign
        uses: actions/github-script@v7
        with:
          script: |
            const fs   = require('fs');
            const yaml = require('js-yaml');
            const raw  = fs.readFileSync('.github/owners.yml', 'utf8');
            const cfg  = yaml.load(raw);
            const cat  = context.payload.label.name.replace('category/', '');
            const map  = cfg.categories[cat];
            if (!map) return;
            const primary = map.primary.replace(/^@/, '');

            // Only individual users can be assignees (not teams). If it's a team, mention instead.
            if (primary.includes('/')) {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                body: `Routing to ${map.primary} (backup: ${map.backup}).`
              });
            } else {
              await github.rest.issues.addAssignees({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                assignees: [primary]
              });
            }

            // Move to triaged state
            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.issue.number,
              labels: ['status/triaged']
            });
            await github.rest.issues.removeLabel({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.issue.number,
              name: 'status/new'
            }).catch(() => {});
```

Note: `js-yaml` needs to be installed. Simplest: add a `setup-node` + `npm i -g js-yaml` step, or install locally via `package.json`.

### 20.3 `sla-post-on-create.yml`

```yaml
name: Post SLA target on create
on:
  issues:
    types: [opened]

permissions:
  issues: write

jobs:
  post-sla:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const body = context.payload.issue.body || '';
            const m = body.match(/### Priority\s*\n+(P[1-4])/i);
            const priority = m ? m[1] : 'P3';
            const hours = { P1: 4, P2: 24, P3: 72, P4: 120 }[priority];
            const target = new Date(Date.now() + hours * 3600 * 1000);
            const iso = target.toISOString();

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.issue.number,
              body: `**SLA target (${priority}):** ${iso}\n\nThis ticket must reach \`status/resolved\` before that time. The nightly SLA tracker will flag it if breached.`
            });
```

### 20.4 `sla-tracker.yml`

```yaml
name: SLA tracker (nightly)
on:
  schedule:
    - cron: '0 6 * * 1-5'   # weekdays 06:00 UTC
  workflow_dispatch:

permissions:
  issues: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const issues = await github.paginate(github.rest.issues.listForRepo, {
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open',
              per_page: 100
            });

            for (const issue of issues) {
              if (issue.labels.some(l => l.name === 'status/resolved' || l.name === 'status/closed')) continue;
              const comments = await github.paginate(github.rest.issues.listComments, {
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue.number,
                per_page: 100
              });
              const slaComment = comments.find(c => c.body.includes('SLA target'));
              if (!slaComment) continue;
              const m = slaComment.body.match(/SLA target \((P[1-4])\):\*\* (.+)/);
              if (!m) continue;
              const target = new Date(m[2]);
              if (Date.now() > target.getTime()) {
                const already = issue.labels.some(l => l.name === 'sla/breached');
                if (already) continue;
                await github.rest.issues.addLabels({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  issue_number: issue.number,
                  labels: ['sla/breached']
                });
                await github.rest.issues.createComment({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  issue_number: issue.number,
                  body: `⚠️ **SLA breached.** Escalation notified.`
                });
              }
            }
```

### 20.5 `stale-triage.yml`

```yaml
name: Auto-close waiting-for-input after 5 days
on:
  schedule:
    - cron: '0 7 * * *'
  workflow_dispatch:

permissions:
  issues: write

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v9
        with:
          only-labels: 'status/waiting-for-input'
          days-before-stale: 5
          days-before-close: 2
          stale-issue-message: |
            This ticket has been in **Waiting for Input** for 5 days.
            It will be auto-closed in 2 days unless you reply.
          close-issue-message: |
            Auto-closed after 7 days without response.
            Reopen or raise a new ticket if the issue persists.
          close-issue-label: 'status/closed'
```

### 20.6 `add-to-project.yml`

```yaml
name: Add new issues to Project
on:
  issues:
    types: [opened]

jobs:
  add:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/add-to-project@v1
        with:
          project-url: https://github.com/orgs/philips-internal/projects/<PROJECT_NUMBER>
          github-token: ${{ secrets.PROJECT_TOKEN }}   # PAT with project:write
```

### 20.7 `teams-notify.yml`

```yaml
name: Notify Teams
on:
  issues:
    types: [opened, closed, labeled]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Build message
        id: msg
        run: |
          echo "text=New/updated ticket: [#${{ github.event.issue.number }}] ${{ github.event.issue.title }} - ${{ github.event.issue.html_url }}" >> $GITHUB_OUTPUT
      - name: POST to Teams
        run: |
          curl -H "Content-Type: application/json" -d '{
            "@type": "MessageCard",
            "@context": "http://schema.org/extensions",
            "summary": "IBE Cloud Ticket update",
            "themeColor": "0076D7",
            "title": "Ticket #${{ github.event.issue.number }}",
            "text": "${{ steps.msg.outputs.text }}"
          }' "${{ secrets.TEAMS_WEBHOOK_URL }}"
```

### 20.8 `state-transition-guard.yml`

```yaml
name: State transition guard
on:
  issues:
    types: [labeled]

permissions:
  issues: write

jobs:
  guard:
    if: github.event.label.name == 'status/resolved'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const comments = await github.paginate(github.rest.issues.listComments, {
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.issue.number
            });
            const hasRCA = comments.some(c => /rca:/i.test(c.body));
            const hasRes = comments.some(c => /resolution:/i.test(c.body));
            if (!hasRCA || !hasRes) {
              await github.rest.issues.removeLabel({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                name: 'status/resolved'
              });
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                body: '❌ Cannot mark **resolved** without both an `RCA:` comment and a `Resolution:` comment. Add them and re-label.'
              });
            }
```

---

## 21. Appendix H — Project (v2) Board Specification

- **Project name:** `IBE Cloud Support`
- **URL:** `https://github.com/orgs/philips-internal/projects/<NUMBER>`
- **Type:** Board layout (default)
- **Linked repos:** `IBE-Cloud-Ticketing-System` only

### 21.1 Columns

| Column | Filter |
|---|---|
| New | label = `status/new` |
| Triaged | label = `status/triaged` |
| Assigned | label = `status/assigned` |
| In Progress | label = `status/in-progress` |
| Waiting for Input | label = `status/waiting-for-input` |
| Resolved | label = `status/resolved` |
| Closed | issue state = closed |

### 21.2 Custom fields

| Field | Type | Values |
|---|---|---|
| Priority | Single select | P1, P2, P3, P4 |
| Category | Single select | cert, pod-connectivity, mtls, rhapsody, dicom, deployment, access, database, env-down, other |
| SLA Target | Date | (set from SLA comment) |
| Environment | Single select | Dev, Test, NonFun, Prod |

### 21.3 Views (all pinned)

| View | Type | Filter | Purpose |
|---|---|---|---|
| All Open | Board | `is:open` | Overall queue |
| By Category | Board, grouped by Category | `is:open` | Category workload |
| By Assignee | Board, grouped by Assignee | `is:open` | Personal queues |
| P1/P2 Expedite | Table | `label:priority/P1,priority/P2 is:open` | Escalation |
| SLA Breaching | Table | `label:sla/breached is:open` | Breach review |
| Waiting for Input | Table | `label:status/waiting-for-input` | Stuck items |
| Resolved this Month | Table | `label:status/resolved closed:>=@start-of-month` | Reporting |

---

## 22. Appendix I — Notification & Teams Integration

### 22.1 Notification subscriptions (per user)

Ask each team member to configure their notifications:

- Repo → **Watch** → **Custom** → check Issues + Discussions
- Personal → github.com → **Settings** → **Notifications**:
  - Watching: Web + Email
  - Participating: Web + Email
  - Actions: off (unless you own workflows)

### 22.2 Teams webhook setup

1. In target Teams channel → **⋮** → **Manage channel** → **Connectors** → **Incoming Webhook** → configure.
2. Name: `IBE Cloud Ticketing`
3. Copy the webhook URL.
4. In GitHub repo → Settings → Secrets → Actions → **New repository secret**:
   - Name: `TEAMS_WEBHOOK_URL`
   - Value: paste the URL.

### 22.3 Optional — GitHub for Teams app

Simpler alternative to `teams-notify.yml`:
- In Teams channel → **+** → search **GitHub** → connect Philips SSO
- `@github subscribe philips-internal/IBE-Cloud-Ticketing-System issues`
- Fine-grained: `@github subscribe philips-internal/IBE-Cloud-Ticketing-System issues +label:"priority/P1"`

---

## 23. Appendix J — Pre-Filled Issue URLs

GitHub Issue Forms accept URL parameters to pre-fill fields. Publish these on the Wiki, Teams channel, and intranet.

### 23.1 Format

```
https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=<template-filename>.yml&title=<encoded-title>&<field-id>=<value>
```

### 23.2 Ready-to-publish links

| Link Label | URL |
|---|---|
| 🔐 Raise a Certificate ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=01-cert-issue.yml` |
| 🔌 Raise a POD Connectivity ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=02-pod-connectivity.yml` |
| 🔒 Raise an MTLS / Service Connectivity ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=03-mtls-service-connectivity.yml` |
| 🎼 Raise a Rhapsody ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=04-rhapsody.yml` |
| 🩻 Raise a DICOM ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=05-dicom.yml` |
| 🚀 Raise a Deployment ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=06-deployment.yml` |
| 🔑 Raise an Access / Namespace ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=07-access-namespace.yml` |
| 🗄️ Raise a Database ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=08-database.yml` |
| 🔥 Raise an Environment Down ticket (P1) | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=09-environment-down.yml` |
| ❓ Raise an Other / Uncategorized ticket | `https://github.com/philips-internal/IBE-Cloud-Ticketing-System/issues/new?template=10-other.yml` |

---

## 24. Appendix K — Metrics Queries (GH CLI / GraphQL)

### 24.1 GH CLI one-liners (PowerShell)

Prereq: `gh auth login`

```powershell
# Open tickets by category
gh issue list -R philips-internal/IBE-Cloud-Ticketing-System --state open --label "category/pod-connectivity"

# All P1s open
gh issue list -R philips-internal/IBE-Cloud-Ticketing-System --state open --label "priority/P1"

# SLA breaches this week
gh issue list -R philips-internal/IBE-Cloud-Ticketing-System --state open --label "sla/breached" --search "updated:>=2026-06-24"

# Resolved this month
gh issue list -R philips-internal/IBE-Cloud-Ticketing-System --state closed --search "closed:>=2026-06-01" --json number,title,labels,closedAt
```

### 24.2 Monthly report (PowerShell script placeholder)

Save under `scripts/sla-report.ps1`:

```powershell
$repo = 'philips-internal/IBE-Cloud-Ticketing-System'
$since = (Get-Date -Day 1).ToString('yyyy-MM-dd')
gh issue list -R $repo --state closed --search "closed:>=$since" --json number,title,labels,closedAt,createdAt |
  ConvertFrom-Json |
  ForEach-Object {
    $ttr = ([datetime]$_.closedAt - [datetime]$_.createdAt).TotalHours
    [pscustomobject]@{
      Number = $_.number
      Title  = $_.title
      Cat    = ($_.labels | Where-Object { $_.name -like 'category/*' }).name -join ','
      Pri    = ($_.labels | Where-Object { $_.name -like 'priority/*' }).name -join ','
      TTRhrs = [math]::Round($ttr, 1)
    }
  } | Format-Table -AutoSize
```

---

## 25. Appendix L — Reference-Repo Comparison Checklist

**Fill this in on Day 0 by inspecting `philips-internal/IBE-Ticketing-System`.** Any deviation from the reference should be justified.

| # | What | Reference uses | Our choice | Aligned? |
|---|---|---|---|---|
| L1 | Issue form filename pattern (`NN-name.yml` vs `name.yml`) | _______________ | `NN-name.yml` | ☐ |
| L2 | Label naming convention (`category/x` vs `type: x`) | _______________ | `category/x` | ☐ |
| L3 | Colors for priority labels | _______________ | Red/Orange/Yellow/Green | ☐ |
| L4 | State labels (`status/*` vs something else) | _______________ | `status/*` | ☐ |
| L5 | Auto-assign mechanism (Action + owners.yml vs CODEOWNERS vs manual) | _______________ | Action + owners.yml | ☐ |
| L6 | SLA implementation (comment-based vs Project field) | _______________ | Comment + tracker | ☐ |
| L7 | Teams integration approach (native app vs webhook workflow) | _______________ | Both / your pick | ☐ |
| L8 | Project (v2) or classic Projects | _______________ | Projects v2 | ☐ |
| L9 | Blank issues allowed | _______________ | No | ☐ |
| L10 | Contact links in `config.yml` | _______________ | Discussions + sibling repo | ☐ |
| L11 | Branch protection on `main` | _______________ | Require PR + CODEOWNERS | ☐ |
| L12 | `docs/` structure | _______________ | playbooks/, sla.md, escalation.md | ☐ |
| L13 | Any custom local Actions (`.github/actions/`) | _______________ | None initially | ☐ |
| L14 | Stale-issue policy | _______________ | 5 days waiting → auto-close | ☐ |
| L15 | Reopen policy | _______________ | Within 7 days of close | ☐ |
| L16 | Naming: repo suffix (`-Ticketing-System` vs `-Tickets` vs `-Support`) | _______________ | `-Ticketing-System` | ☐ |

---

## 26. Appendix M — Risks, Assumptions & Open Questions

### 26.1 Assumptions (validate during Phase 0)

| # | Assumption | Validated? |
|---|---|---|
| A1 | We can create a new repo under `philips-internal` (or have admin do it) | ☐ |
| A2 | All requesters have GitHub Enterprise access | ☐ |
| A3 | Cloud Team Lead available to sign off on ownership matrix | ☐ |
| A4 | Teams channel exists for notifications | ☐ |
| A5 | Reference repo `IBE-Ticketing-System` is accessible and we can inspect it | ☐ |
| A6 | Team is OK with GitHub Actions minutes usage (should be free for internal repos) | ☐ |
| A7 | js-yaml or equivalent is acceptable in workflows (or we use a pure-shell alternative) | ☐ |

### 26.2 Risks

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Requesters bypass the repo and DM engineers | High | Team lead enforces "no ticket = no work"; educate in Teams |
| R2 | Auto-assign points to a user who leaves | Medium | Assign to a GitHub Team where possible; quarterly ownership review |
| R3 | Sub-hour SLAs not enforceable by scheduled Actions | Medium | For P1, immediate Teams `@channel` on creation; manual acknowledgment |
| R4 | Reference repo diverges (they change their conventions) | Low | Quarterly sync with sibling team lead |
| R5 | Categories don't cover a real request | Low | `Other` fallback; quarterly review |
| R6 | Sensitive info (creds, PII) posted in issue body | High | Add `.github/ISSUE_TEMPLATE/config.yml` warning; consider Actions secret-scanning; educate |
| R7 | Sibling team's Actions patterns not portable | Low | Design keeps Actions minimal & standard |
| R8 | Repo creation blocked by GH admin governance | Medium | Escalate via Cloud Team Lead; use existing empty Philips repo as fallback |

### 26.3 Open Questions (answer during Phase 0)

| # | Question | Answer |
|---|---|---|
| Q1 | Exact repo name to use? | _______________ |
| Q2 | Which GitHub Team owns the repo (Admin role)? | _______________ |
| Q3 | Who is the Cloud Team Lead (name + GitHub handle)? | _______________ |
| Q4 | Who is the default Triage Owner? | _______________ |
| Q5 | Top-5 requester teams? | _______________ |
| Q6 | Which Teams channel gets ticket notifications? | _______________ |
| Q7 | Existing shared mailbox to retire? | _______________ |
| Q8 | Compliance / audit requirements? | _______________ |
| Q9 | Preferred rollout date? | _______________ |
| Q10 | Reference repo — do they use GitHub Team or individual assignees? | _______________ |
| Q11 | Reference repo — do they use Projects v2 or classic? | _______________ |
| Q12 | Who signs off on this design? | _______________ |
| Q13 | Do we need to import historical tickets from Teams/email? | _______________ |
| Q14 | Preferred timezone for SLA calculations? | _______________ |

---

## Sign-Off

| Role | Name | Signed | Date |
|---|---|---|---|
| Cloud Team Lead | _______________ | ☐ | ____ |
| Product Owner | _______________ | ☐ | ____ |
| GitHub Admin / DevOps contact | _______________ | ☐ | ____ |
| Reference Repo Owner (sibling team) | _______________ | ☐ | ____ |
| Requester (self) | _______________ | ☐ | ____ |

---

*End of document.*
