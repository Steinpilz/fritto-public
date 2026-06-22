---
name: time-submitter
description: Track your OWN time in Fritto (TimeTracker) via MCP — log hours against projects, view/edit/delete your entries, submit timesheets for approval, and view your own reports. Use for personal time tracking by a regular employee.
---

# Fritto — Time Submitter

Tools for tracking your own time: logging hours, editing entries, submitting for approval, and viewing your own reports.

**Conventions:**
- Dates: always `YYYY-MM-DD`
- IDs: UUIDs (`projectId`, `timeRecordId`)
- Business rules enforced server-side: lock dates, approval status, project assignments

## Available Tools

| Tool | Purpose |
|------|---------|
| `list_projects` | Get your assigned projects with IDs and client names |
| `validate_day` | Check if a date is editable (not locked/approved) |
| `log_time` | Create a time entry |
| `get_time_records` | View your entries for a date range (returns record IDs + approval status) |
| `update_time_record` | Modify an existing entry |
| `delete_time_record` | Remove an entry (irreversible) |
| `submit_for_approval` | Submit draft entries for review |
| `get_own_time_reports` | Your aggregated hours, utilization, billable metrics |
| `get_tracked_time` | Total hours logged against a task URL |

## Log Time

Always get project IDs first:
1. `list_projects` → note `projectId`
2. `log_time` with `projectId`, `hours` (decimal, max 24), `date`
   - Optional: `taskUrl` (issue link), `description` (max 1024 chars)

## View & Edit Entries

1. `get_time_records` with `from` and `to` dates
   - Returns per-day records with `timeRecordId` (the record `id`) and approval status
2. To update: `update_time_record` with `timeRecordId`, `projectId`, `hours`, `date`
   - Concurrency stamp auto-fetched if omitted
   - To move to a different date: set `date` to the new date, `originalDate` to the current date
3. To delete: `delete_time_record` with `timeRecordId`

## Submit for Approval

`submit_for_approval` with `from` and `to` — converts all Draft days in range to Submitted.

## View Your Reports

- `get_own_time_reports` with `from`, `to` — aggregated metrics
  - Optional: `groupBy` (0=Employee, 1=Project, 2=EmployeeGroup, 3=Task)
  - Optional filters: `clientIds`, `projectIds`, `approvalStatuses`, `taskUrls`
- `get_tracked_time` with `taskUrl` — total hours across all users for a task

## Workflow: Log a Full Week

```
1. list_projects → get projectId
2. validate_day for Monday → confirm editable
3. log_time for Mon, Tue, Wed, Thu, Fri (one call each)
4. get_time_records from=Monday to=Friday → verify
5. submit_for_approval from=Monday to=Friday
```

## Error Scenarios

| Error | Meaning | Resolution |
|-------|---------|------------|
| `DateApproved` | Day already approved | Ask your approver to decline it first |
| `UserDayInLockedPeriod` | Date before lock date | Contact an admin to adjust the lock |
| `ProjectNotAssignedToEmployee` | Not assigned to project | Use `list_projects` for valid IDs |
