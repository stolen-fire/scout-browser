---
description: Schedule an exported workflow to run automatically via OS task manager
argument-hint: "[list | workflow-name <when> | delete <name-or-id>]"
allowed-tools:
  - "mcp__plugin_scout_scout__schedule_create"
  - "mcp__plugin_scout_scout__schedule_list"
  - "mcp__plugin_scout_scout__schedule_delete"
  - "Read"
  - "AskUserQuestion"
---

Schedule exported Scout workflows to run automatically using the OS task manager (Windows Task Scheduler / macOS launchd / Linux cron).

## Parse the argument

Determine the operation:

- **No argument or `list`**: Go to List
- **`delete <name-or-id>`**: Go to Delete
- **`<workflow-name>` or `<workflow-name> <when>`**: Go to Create

---

## Create a schedule

1. **Find the workflow.** Check that `workflows/<name>/<name>.py` exists using Read. If not found, tell the user: "No exported workflow found at `workflows/<name>/`. Run `/scout:export-workflow <name>` first."

2. **Parse schedule from argument or ask the user.**

   If `<when>` was provided, parse it:
   - **Natural language**: "every weekday at 9am", "daily at midnight", "every hour", "every Monday at 3pm"
   - **Cron-like**: Map to MCP tool parameters

   If no `<when>` provided, ask using AskUserQuestion:
   - "How often should this run?" — Daily / Weekly / Weekdays (Mon-Fri) / One-time
   - "What time?" — Accept flexible formats (6:45am, 06:45, 6:45 AM), convert to HH:MM 24-hour
   - If Weekly: "Which days?" — MON, TUE, WED, THU, FRI, SAT, SUN

3. **Map to MCP tool parameters:**
   - Daily → `schedule="DAILY"`
   - Weekly → `schedule="WEEKLY"`, `days="MON,WED,FRI"`
   - Weekdays → `schedule="WEEKLY"`, `days="MON,TUE,WED,THU,FRI"`
   - One-time → `schedule="ONCE"`

   Call `schedule_create` with `name`, `workflow_dir` (absolute path), `schedule`, `time`, and `days`.

4. **Report:**
   ```
   Scheduled "enrollment" to run daily at 6:45 AM
   Platform: Windows (Task Scheduler)
   Script: workflows/enrollment/enrollment.py

   To view all schedules: /scout:schedule list
   To remove this schedule: /scout:schedule delete enrollment
   ```

---

## List schedules

Call the `schedule_list` MCP tool. Display as a table:

| Workflow | Schedule | Time | Days | Status | Next Run |
|----------|----------|------|------|--------|----------|
| enrollment | Daily | 6:45 AM | | Ready | 2/28/2026 |

If no tasks: "No scheduled tasks found. Export a workflow with `/scout:export-workflow` first, then schedule it."

---

## Delete a schedule

1. **Confirm:** "Delete the scheduled task for **<name>**? This removes it from the system scheduler."
2. Call `schedule_delete` with the task name.
3. Confirm deletion.

---

## Error Handling

- Workflow not found: "No exported workflow at `workflows/<name>/`. Run `/scout:export-workflow <name>` first."
- MCP tool errors: Surface the error message from the tool.
