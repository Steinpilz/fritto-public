---
name: admin
description: Full Fritto (TimeTracker) administration via MCP — everything in time tracking and approvals PLUS user management, cost rates, project assignments, and employee groups. Use for company admins managing the whole tenant.
---

# Fritto — Company Admin

All 29 Fritto MCP tools: time tracking, approvals, reporting, user management, cost rates, project assignments, and employee groups.

**Conventions:**
- Dates: always `YYYY-MM-DD`
- IDs: UUIDs (`projectId`, `userId`, `timeRecordId`)
- On-behalf: 7 time-tracking tools accept an optional `userId` to act for an employee — see the On-Behalf column in the Tool Quick Reference
- Business rules enforced server-side: lock dates, approval status, project assignments

---

## Time Tracking

| Tool | Purpose |
|------|---------|
| `list_projects` | Assigned projects with IDs and client names (optional `userId`) |
| `validate_day` | Check if a date is editable (not locked/approved) |
| `log_time` | Create a time entry |
| `get_time_records` | View entries for a date range (returns record IDs + approval status) |
| `update_time_record` | Modify an existing entry |
| `delete_time_record` | Remove an entry (irreversible) |
| `submit_for_approval` | Submit draft entries for review |
| `get_tracked_time` | Total hours logged for a task URL, across all users and projects |

- `log_time`: `projectId`, `hours` (decimal, max 24), `date`; optional `taskUrl`, `description` (max 1024)
- `update_time_record`: `timeRecordId`, `projectId`, `hours`, `date`; concurrency stamp auto-fetched; set `originalDate` to move an entry to a new `date`
- All except `get_tracked_time` accept optional `userId` for on-behalf operations (`get_tracked_time` takes only `taskUrl`)

---

## Approvals

| Tool | Purpose |
|------|---------|
| `search_pending_approvals` | Find entries by approval status |
| `approve_time_entries` | Bulk approve submitted entries |
| `decline_time_entries` | Revoke approval on already-approved entries |

1. `search_pending_approvals` with `from`, `to` (filters: `userIds`, `employeeGroupIds`; default `approvalStatus: 1` = Submitted)
2. `approve_time_entries` with `from`, `to` — acts on Submitted entries
3. `decline_time_entries` with `from`, `to` — acts on already-**Approved** entries (`approvalStatus: 2`), sending them back for revision. It does NOT reject Submitted entries.

| Error | Meaning | Resolution |
|-------|---------|------------|
| `NotAllTimesheetsReadyForApproval` | Entry not in expected status | Re-search to get current status |
| `TimesheetsHaveChanged` | Data modified since your search | Re-run `search_pending_approvals` |
| `UserDayInLockedPeriod` | Date in locked period | Cannot approve/decline locked dates |

---

## Reports & Exports

| Tool | Purpose |
|------|---------|
| `get_own_time_reports` | Your aggregated hours, utilization, billable metrics |
| `get_all_time_reports` | Aggregated reports for all users |
| `export_time_reports_csv` | Export as CSV (supports filters) |
| `export_time_reports_xlsx` | Export as Excel (`from`/`to` only) |
| `export_time_reports_json` | Export as JSON (`from`/`to` + `prettyPrint`, max 75 KB) |

- `get_all_time_reports` with `from`, `to` — filters: `userIds`, `employeeGroupIds`, `clientIds`, `projectIds`, `approvalStatuses`, `taskUrls`; `groupBy`: 0=Employee, 1=Project, 2=EmployeeGroup, 3=Task
- Export filter support differs by format: **csv** supports the same filters; **xlsx** is `from`/`to` only; **json** is `from`/`to` plus `prettyPrint` (default true)
- Size limit: 75 KB (~25K tokens). Narrow date range if exceeded.

---

## User Management

| Tool | Purpose |
|------|---------|
| `list_users` | Search/filter users with details |
| `get_user` | Full user profile (by ID, email, or name) |
| `create_user` | Create user + send invitation email |
| `update_user` | Update user info (find-then-update) |
| `delete_user` | Soft-delete (data retained) |
| `change_user_status` | Activate or deactivate a user |

- `list_users` filters: `search` (fuzzy name/email), `employeeGroupIds`, `projectIds`, `roleIds`, `userStatuses` (`['Active','Invited','Inactive']`), `includeCosts` (default true)
- `get_user` (at least one of): `userId`, `email` (exact), or `name` (fuzzy); optional `asOfDate`
- `create_user`: `email`, `firstName`, `lastName`, `defaultCapacityPerWeekHours` (typically 40); optional `phoneNumber` — sends invitation email automatically
- `update_user`: find by `findByUserId`/`findByEmail`/`findByName`, then set `email`, `firstName`, `lastName`, `phoneNumber`, `defaultCapacityPerWeekHours`
- `change_user_status`: `action` `'activate'`/`'deactivate'` (lookup by `userId`/`email`/`name`; cannot deactivate your own account)
- `delete_user`: soft-delete; fails if the user has tracked time → deactivate instead

| Error | Meaning | Resolution |
|-------|---------|------------|
| `UserEmailExist` | Email already in use | Use a different email or find the existing user |
| `MaxUserCountReached` | Subscription limit hit | Upgrade plan or deactivate unused users |
| `CannotDeactivateOwnUser` | Self-deactivation blocked | Ask another admin |
| `UserHasTrackedTime` | Cannot delete user with entries | Use `change_user_status` to deactivate instead |

---

## Cost Management

| Tool | Purpose |
|------|---------|
| `get_user_costs` | Cost rate history for a user |
| `add_user_cost` | Add a new cost rate entry |
| `delete_user_cost` | Remove a cost rate entry by date |

- `add_user_cost`: `costRate` (number), `currency` (`'EUR'`/`'USD'`), `validFrom` (YYYY-MM-DD) — effective from `validFrom` until the next entry; lookup user by `userId`/`email`/`name`
- `delete_user_cost`: remove by `validFrom` date + user lookup

| Error | Meaning | Resolution |
|-------|---------|------------|
| `ValidFromDatesNotUnique` | Cost entry date conflict | Delete the existing entry first, then add the new one |

---

## Project Assignments & Groups

| Tool | Purpose |
|------|---------|
| `list_all_projects` | All tenant projects with IDs and client names |
| `get_project_users` | User IDs assigned to a project |
| `manage_project_assignments` | Add, remove, or replace project↔user assignments |
| `list_employee_groups` | Departments/teams with member counts |

- `manage_project_assignments`:
  - `mode`: `projectsToUser` (targetId=userId, assignIds=projectIds) or `usersToProject` (targetId=projectId, assignIds=userIds)
  - `operation` (REQUIRED): `add` (append), `remove` (subtract), `set` (⚠️ replaces ALL current assignments)
  - **Default to `add`/`remove`** — use `set` only when explicitly asked to replace the entire list
  - For add/remove the tool fetches current assignments internally — no need to read first

---

## Workflows

**Onboard a new employee:**
```
1. create_user email="jane@company.com" firstName="Jane" lastName="Smith" defaultCapacityPerWeekHours=40
2. add_user_cost userId=<new-id> costRate=85 currency="EUR" validFrom="2026-04-01"
3. manage_project_assignments mode="projectsToUser" operation="set" targetId=<new-id> assignIds=[<projectId1>, <projectId2>]
```

**Offboard an employee:**
```
1. get_user email="jane@company.com" → verify identity
2. change_user_status email="jane@company.com" action="deactivate"
   (Do NOT delete — user likely has tracked time)
```

**Assign projects to a user:**
```
1. list_users search="John" → get userId
2. list_all_projects → get projectIds
3. manage_project_assignments mode="projectsToUser" operation="add" targetId=<userId> assignIds=[<projectId1>, <projectId2>]
```

---

## Tool Quick Reference

| Tool | Category | Read-Only | On-Behalf |
|------|----------|-----------|-----------|
| `list_projects` | Time | Yes | Yes |
| `validate_day` | Time | Yes | Yes |
| `log_time` | Time | No | Yes |
| `get_time_records` | Time | Yes | Yes |
| `update_time_record` | Time | No | Yes |
| `delete_time_record` | Time | No | Yes |
| `submit_for_approval` | Approval | No | Yes |
| `search_pending_approvals` | Approval | Yes | No |
| `approve_time_entries` | Approval | No | No |
| `decline_time_entries` | Approval | No | No |
| `get_own_time_reports` | Reports | Yes | No |
| `get_all_time_reports` | Reports | Yes | No |
| `get_tracked_time` | Reports | Yes | No |
| `export_time_reports_csv` | Reports | Yes | No |
| `export_time_reports_xlsx` | Reports | Yes | No |
| `export_time_reports_json` | Reports | Yes | No |
| `list_users` | Users | Yes | No |
| `get_user` | Users | Yes | No |
| `create_user` | Users | No | No |
| `update_user` | Users | No | No |
| `delete_user` | Users | No | No |
| `change_user_status` | Users | No | No |
| `get_user_costs` | Costs | Yes | No |
| `add_user_cost` | Costs | No | No |
| `delete_user_cost` | Costs | No | No |
| `list_all_projects` | Projects | Yes | No |
| `get_project_users` | Projects | Yes | No |
| `manage_project_assignments` | Projects | No | No |
| `list_employee_groups` | Groups | Yes | No |
