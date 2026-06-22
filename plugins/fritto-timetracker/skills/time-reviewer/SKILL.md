---
name: time-reviewer
description: Review and approve team time in Fritto (TimeTracker) via MCP — search pending approvals, approve submitted timesheets, revoke approvals, track time on behalf of employees, and run/export team-wide reports. Use for managers, leads, and approvers.
---

# Fritto — Time Reviewer / Approver

Tools for reviewing and approving team time, acting on behalf of employees, and running team-wide reports and exports. Includes the personal time-tracking tools (reviewers also track their own time).

**Conventions:**
- Dates: always `YYYY-MM-DD`
- IDs: UUIDs (`projectId`, `userId`, `timeRecordId`)
- On-behalf: 7 tools accept an optional `userId` to act for an employee (requires `TrackOnBehalf` permission) — listed under On-Behalf Operations below; report tools (`get_own_time_reports`, `get_tracked_time`) do not
- Business rules enforced server-side: lock dates, approval status, project assignments

## Approval Tools

| Tool | Purpose |
|------|---------|
| `search_pending_approvals` | Find entries by approval status |
| `approve_time_entries` | Bulk approve submitted entries |
| `decline_time_entries` | Revoke approval on already-approved entries |
| `get_all_time_reports` | Aggregated reports for all users |
| `export_time_reports_csv` | Export as CSV (supports filters) |
| `export_time_reports_xlsx` | Export as Excel (`from`/`to` only) |
| `export_time_reports_json` | Export as JSON (`from`/`to`, max 75 KB) |

## Time-Tracking Tools (own time + on-behalf)

| Tool | Purpose |
|------|---------|
| `list_projects` | Assigned projects with IDs and client names |
| `validate_day` | Check if a date is editable (not locked/approved) |
| `log_time` | Create a time entry |
| `get_time_records` | View entries for a date range (returns record IDs + approval status) |
| `update_time_record` | Modify an existing entry |
| `delete_time_record` | Remove an entry (irreversible) |
| `submit_for_approval` | Submit draft entries for review |
| `get_own_time_reports` | Your aggregated hours, utilization, billable metrics |
| `get_tracked_time` | Total hours logged for a task URL, across all users and projects |

## Approval Workflow

1. `search_pending_approvals` with `from`, `to`
   - Optional filters: `userIds`, `employeeGroupIds`
   - Default: searches Submitted entries (`approvalStatus: 1`)
   - Returns per-user breakdown with hours, utilization, daily timesheet IDs
2. Review the returned data (hours per user/day)
3. `approve_time_entries` with `from`, `to` to approve
   - Optional: `userIds`, `employeeGroupIds` to approve specific users/groups
   - Acts on Submitted entries (`approvalStatus: 1`)

To **reverse** an approval, use `decline_time_entries` with `from`, `to` (and optional `userIds`/`employeeGroupIds`). It acts on already-**Approved** entries (`approvalStatus: 2`), sending them back for revision. It does NOT reject Submitted entries — to send a Submitted day back, approve then decline, or have the employee re-edit before approval.

## On-Behalf Operations

Act for an employee using `userId` on these 7 tools (requires `TrackOnBehalf` permission):
- `list_projects` + `userId` — see their assigned projects
- `validate_day` + `userId` — check if a date is editable for them
- `log_time` + `userId` — log time for them
- `get_time_records` + `userId` — view their entries
- `update_time_record` + `userId` — edit their entry
- `delete_time_record` + `userId` — remove their entry
- `submit_for_approval` + `userId` — submit their entries

`get_own_time_reports` and `get_tracked_time` do not take `userId` — use `get_all_time_reports` (with `userIds` filter) for another user's metrics.

## Reports & Exports

- `get_all_time_reports` with `from`, `to` — aggregated metrics for all users
  - Filters: `userIds`, `employeeGroupIds`, `clientIds`, `projectIds`, `approvalStatuses`, `taskUrls`
  - `groupBy`: 0=Employee, 1=Project, 2=EmployeeGroup, 3=Task
- Export tools output base64-encoded files. Filter support differs per format:
  - `export_time_reports_csv` — supports the same filters as `get_all_time_reports` (`userIds`, `employeeGroupIds`, `clientIds`, `projectIds`, `approvalStatuses`, `taskUrls`)
  - `export_time_reports_xlsx` — `from`/`to` only (no filters)
  - `export_time_reports_json` — `from`/`to` plus `prettyPrint` (boolean, default true); no filters
  - Size limit: 75 KB (~25K tokens). Narrow date range if exceeded.

## Logging Your Own Time

Same flow as a regular employee: `list_projects` → `log_time` (`projectId`, `hours` max 24, `date`; optional `taskUrl`, `description` max 1024) → `get_time_records` to verify → `submit_for_approval`. Use `update_time_record`/`delete_time_record` to amend; set `originalDate` to move an entry to a new `date`.

## Workflow: Weekly Approval

```
1. search_pending_approvals from=Monday to=Friday
2. Review hours per user/day in response
3. approve_time_entries from=Monday to=Friday
   (or filter: approve_time_entries from=Monday to=Friday userIds=[...])
4. To revoke a mistaken approval: decline_time_entries for specific users
```

## Error Scenarios

| Error | Meaning | Resolution |
|-------|---------|------------|
| `NotAllTimesheetsReadyForApproval` | Entry not in expected status | Re-search to get current status |
| `TimesheetsHaveChanged` | Data modified since your search | Re-run `search_pending_approvals` |
| `UserDayInLockedPeriod` | Date in locked period | Cannot approve/decline locked dates |
| `DateApproved` | Day already approved | Decline it first to re-open |
| `ProjectNotAssignedToEmployee` | User not assigned to project | Use `list_projects` (with their `userId`) for valid IDs |
