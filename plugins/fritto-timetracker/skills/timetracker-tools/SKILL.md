---
name: timetracker-tools
description: Guide for using TimeTracker MCP tools. Covers time submission, approval workflows, user/cost management, and reporting. Use when interacting with TimeTracker (Fritto) via MCP.
---

# TimeTracker MCP Tools

29 tools for time tracking, approvals, user management, project assignments, and reporting.

**Conventions:**
- Dates: always `YYYY-MM-DD`
- IDs: UUIDs (`projectId`, `userId`, `timeRecordId`)
- On-behalf: many tools accept optional `userId` for managers acting for employees
- Business rules enforced server-side: lock dates, approval status, project assignments

---

## Time Submitter

### Available Tools

| Tool | Purpose |
|------|---------|
| `list_projects` | Get assigned projects with IDs and client names |
| `validate_day` | Check if a date is editable (not locked/approved) |
| `log_time` | Create a time entry |
| `get_time_records` | View entries for a date range (returns IDs, stamps) |
| `update_time_record` | Modify an existing entry |
| `delete_time_record` | Remove an entry (irreversible) |
| `submit_for_approval` | Submit draft entries for review |
| `get_own_time_reports` | Aggregated hours, utilization, billable metrics |
| `get_tracked_time` | Total hours logged against a task URL |

### Log Time

Always get project IDs first:
1. `list_projects` → note `projectId`
2. `log_time` with `projectId`, `hours` (decimal, max 24), `date`
   - Optional: `taskUrl` (issue link), `description` (max 1024 chars)

### View & Edit Entries

1. `get_time_records` with `from` and `to` dates
   - Returns per-day records with `timeRecordId`, `concurrencyStamp`, approval status
2. To update: `update_time_record` with `timeRecordId`, `projectId`, `hours`, `date`
   - Concurrency stamp auto-fetched if omitted
   - To move to different date: set `date` to new date, `originalDate` to current date
3. To delete: `delete_time_record` with `timeRecordId`

### Submit for Approval

`submit_for_approval` with `from` and `to` — converts all Draft days in range to Submitted.

### View Reports

- `get_own_time_reports` with `from`, `to` — aggregated metrics
  - Optional: `groupBy` (0=Employee, 1=Project, 2=EmployeeGroup, 3=Task)
  - Optional filters: `clientIds`, `projectIds`, `approvalStatuses`, `taskUrls`
- `get_tracked_time` with `taskUrl` — total hours across all users for a task

### Workflow: Log a Full Week

```
1. list_projects → get projectId
2. validate_day for Monday → confirm editable
3. log_time for Mon, Tue, Wed, Thu, Fri (one call each)
4. get_time_records from=Monday to=Friday → verify
5. submit_for_approval from=Monday to=Friday
```

### Error Scenarios

| Error | Meaning | Resolution |
|-------|---------|------------|
| `DateApproved` | Day already approved | Ask approver to decline first |
| `UserDayInLockedPeriod` | Date before lock date | Contact admin to adjust lock |
| `ProjectNotAssignedToEmployee` | Not assigned to project | Use `list_projects` for valid IDs |

---

## Time Reviewer / Approver

### Available Tools

| Tool | Purpose |
|------|---------|
| `search_pending_approvals` | Find entries by approval status |
| `approve_time_entries` | Bulk approve submitted entries |
| `decline_time_entries` | Bulk decline/reject entries |
| `get_all_time_reports` | Aggregated reports for all users |
| `export_time_reports_csv` | Export as CSV |
| `export_time_reports_xlsx` | Export as Excel |
| `export_time_reports_json` | Export as JSON (max 75 KB) |

Plus all Time Submitter tools with `userId` parameter for on-behalf operations.

### Approval Workflow

1. `search_pending_approvals` with `from`, `to`
   - Optional filters: `userIds`, `employeeGroupIds`
   - Default: searches Submitted entries (`approvalStatus: 1`)
   - Returns per-user breakdown with hours, utilization, daily timesheet IDs
2. Review the returned data (hours per user/day)
3. `approve_time_entries` with `from`, `to` to approve
   - Optional: `userIds`, `employeeGroupIds` to approve specific users/groups
4. OR `decline_time_entries` with same params to reject (moves back to Draft)

### On-Behalf Operations

Act for an employee using `userId` on these tools:
- `get_time_records` + `userId` — view their entries
- `log_time` + `userId` — log time for them
- `update_time_record` + `userId` — edit their entry
- `delete_time_record` + `userId` — remove their entry
- `submit_for_approval` + `userId` — submit their entries
- `list_projects` + `userId` — see their assigned projects

Requires `TrackOnBehalf` permission.

### Reports & Exports

- `get_all_time_reports` with `from`, `to` — aggregated metrics for all users
  - Filters: `userIds`, `employeeGroupIds`, `clientIds`, `projectIds`, `approvalStatuses`, `taskUrls`
  - `groupBy`: 0=Employee, 1=Project, 2=EmployeeGroup, 3=Task
- Export tools share same filters; output as CSV/XLSX (base64-encoded) or JSON
  - Size limit: 75 KB (~25K tokens). Narrow date range if exceeded.

### Workflow: Weekly Approval

```
1. search_pending_approvals from=Monday to=Friday
2. Review hours per user/day in response
3. approve_time_entries from=Monday to=Friday
   (or filter: approve_time_entries from=Monday to=Friday userIds=[...])
4. For issues: decline_time_entries for specific users
```

### Error Scenarios

| Error | Meaning | Resolution |
|-------|---------|------------|
| `NotAllDaysWithApprovalStatus` | Entry not in expected status | Re-search to get current status |
| `TimesheetsChanged` | Data modified since your search | Re-run `search_pending_approvals` |
| `UserDayInLockedPeriod` | Date in locked period | Cannot approve/decline locked dates |

---

## Company Admin

### User Management Tools

| Tool | Purpose |
|------|---------|
| `list_users` | Search/filter users with details |
| `get_user` | Full user profile (by ID, email, or name) |
| `create_user` | Create user + send invitation email |
| `update_user` | Update user info (find-then-update) |
| `delete_user` | Soft-delete (data retained) |
| `change_user_status` | Activate or deactivate a user |

### Cost Management Tools

| Tool | Purpose |
|------|---------|
| `get_user_costs` | Cost rate history for a user |
| `add_user_cost` | Add a new cost rate entry |
| `delete_user_cost` | Remove a cost rate entry by date |

### Project Assignment Tools

| Tool | Purpose |
|------|---------|
| `list_all_projects` | List all tenant projects with IDs and client names |
| `get_project_users` | Get user IDs assigned to a project |
| `manage_project_assignments` | Add, remove, or replace project↔user assignments |

### Other

| Tool | Purpose |
|------|---------|
| `list_employee_groups` | List departments/teams with member counts |

### Search & Lookup Users

- `list_users` — search with filters:
  - `search`: fuzzy name/email match
  - `employeeGroupIds`, `projectIds`, `roleIds`: filter by assignments
  - `userStatuses`: `['Active', 'Invited', 'Inactive']`
  - `includeCosts`: true/false (default true)
- `get_user` — full profile lookup (at least one required):
  - `userId` (UUID), `email` (exact), or `name` (fuzzy)
  - Optional: `asOfDate` for cost data as-of date

### Create & Update Users

- `create_user`: `email`, `firstName`, `lastName`, `defaultCapacityPerWeekHours` (typically 40)
  - Sends invitation email automatically
  - Optional: `phoneNumber`
- `update_user`: find user first, then set fields to change
  - Find by: `findByUserId`, `findByEmail`, or `findByName`
  - Update: `email`, `firstName`, `lastName`, `phoneNumber`, `defaultCapacityPerWeekHours`

### Activate / Deactivate / Delete

- `change_user_status` with `action`: `'activate'` or `'deactivate'`
  - Lookup by `userId`, `email`, or `name`
  - Cannot deactivate your own account
- `delete_user` — soft-delete (data retained for audit)
  - Fails if user has any tracked time → use deactivate instead

### Cost Rates

- `get_user_costs` — view cost history (lookup by `userId`, `email`, or `name`)
- `add_user_cost` — add entry: `costRate` (number), `currency` ('EUR'/'USD'), `validFrom` (YYYY-MM-DD)
  - Effective from `validFrom` until next entry's date
  - Lookup user by `userId`, `email`, or `name`
- `delete_user_cost` — remove by `validFrom` date + user lookup

### Workflow: Onboard New Employee

```
1. create_user email="jane@company.com" firstName="Jane" lastName="Smith" defaultCapacityPerWeekHours=40
2. add_user_cost userId=<new-id> costRate=85 currency="EUR" validFrom="2026-04-01"
3. manage_project_assignments mode="projectsToUser" operation="set" targetId=<new-id> assignIds=[<projectId1>, <projectId2>]
```

### Workflow: Offboard Employee

```
1. get_user email="jane@company.com" → verify identity
2. change_user_status email="jane@company.com" action="deactivate"
   (Do NOT delete — user likely has tracked time)
```

### Error Scenarios

| Error | Meaning | Resolution |
|-------|---------|------------|
| `UserEmailExist` | Email already in use | Use different email or find existing user |
| `MaxUserCountReached` | Subscription limit hit | Upgrade plan or deactivate unused users |
| `CannotDeactivateOwnUser` | Self-deactivation blocked | Ask another admin |
| `UserHasTrackedTime` | Cannot delete user with entries | Use `change_user_status` to deactivate instead |
| `ValidFromDatesNotUnique` | Cost entry date conflict | Delete existing entry first, then add new one |

### Project Assignments

- `list_all_projects` — all tenant projects (not just user's assigned ones)
- `get_project_users` — user IDs assigned to a specific project
- `manage_project_assignments` — unified tool for all assignment changes:
  - `mode`: `projectsToUser` (targetId=userId, assignIds=projectIds) or `usersToProject` (targetId=projectId, assignIds=userIds)
  - `operation` (REQUIRED): `add` (append), `remove` (subtract from existing), `set` (⚠️ DANGEROUS — replaces ALL current assignments)
  - **Default to `add` or `remove`** — only use `set` when the user explicitly asks to replace the entire assignment list
  - For add/remove, the tool fetches current assignments internally — no need to read first

### Workflow: Assign Projects to User

```
1. list_users search="John" → get userId
2. list_all_projects → get projectIds
3. manage_project_assignments mode="projectsToUser" operation="add" targetId=<userId> assignIds=[<projectId1>, <projectId2>]
```

### Workflow: Remove User from Project

```
1. manage_project_assignments mode="usersToProject" operation="remove" targetId=<projectId> assignIds=[<userId>]
```

---

## Tool Quick Reference

| Tool | Category | Read-Only | On-Behalf | Description |
|------|----------|-----------|-----------|-------------|
| `list_projects` | Time | Yes | Yes | Assigned projects with IDs |
| `validate_day` | Time | Yes | Yes | Check if date is editable |
| `log_time` | Time | No | Yes | Create time entry |
| `get_time_records` | Time | Yes | Yes | Entries for date range |
| `update_time_record` | Time | No | Yes | Modify time entry |
| `delete_time_record` | Time | No | Yes | Remove time entry |
| `submit_for_approval` | Approval | No | Yes | Submit drafts for review |
| `search_pending_approvals` | Approval | Yes | No | Find entries by status |
| `approve_time_entries` | Approval | No | No | Bulk approve |
| `decline_time_entries` | Approval | No | No | Bulk decline |
| `get_own_time_reports` | Reports | Yes | No | Own aggregated reports |
| `get_all_time_reports` | Reports | Yes | No | All users' reports |
| `get_tracked_time` | Reports | Yes | No | Hours for a task URL |
| `export_time_reports_csv` | Reports | Yes | No | Export as CSV |
| `export_time_reports_xlsx` | Reports | Yes | No | Export as Excel |
| `export_time_reports_json` | Reports | Yes | No | Export as JSON |
| `list_users` | Users | Yes | No | Search/filter users |
| `get_user` | Users | Yes | No | Full user profile |
| `create_user` | Users | No | No | Create + invite user |
| `update_user` | Users | No | No | Update user info |
| `delete_user` | Users | No | No | Soft-delete user |
| `change_user_status` | Users | No | No | Activate/deactivate |
| `get_user_costs` | Costs | Yes | No | Cost rate history |
| `add_user_cost` | Costs | No | No | Add cost rate entry |
| `delete_user_cost` | Costs | No | No | Remove cost rate entry |
| `list_employee_groups` | Groups | Yes | No | Departments/teams |
| `list_all_projects` | Projects | Yes | No | All tenant projects with IDs |
| `get_project_users` | Projects | Yes | No | Users assigned to a project |
| `manage_project_assignments` | Projects | No | No | Add/remove/set project↔user assignments |
