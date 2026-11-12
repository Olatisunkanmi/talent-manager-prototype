# FRD: Campaign Tasks

**PRD refs:** §6.13, FR-45 to FR-50, FR-56 to FR-63 · **Milestone:** 10

## Purpose
A campaign involves many small jobs (confirm the brief, collect the draft, chase the invoice, log the post links, settle the talent). Today they live in chats and memory. Tasks give each campaign a checklist that the agency can assign, date and tick off, and a cross-campaign view of what is due.

This is a lightweight to-do list, not a project-management tool.

## Real team members as assignees
Tasks use real users from team management (`team.frd.md`).
- A task's **assignee** is an **active user who can see the campaign**: they have the Tasks page and the campaign's talent in scope. The picker lists only those users, so a task is never assigned to someone who cannot open it. Unassigned is allowed.
- Members see only tasks on their assigned talents, and only if they have the Tasks page. Admins see all.
- Any member who can see a task can edit it. Anyone who can see a task can delete it, after a confirmation.
- The signed-in user is recorded as **created by**, and **completed by** when a task is ticked.
- An assignee who is later deactivated keeps their name on open tasks, marked "(deactivated)"; admins get a "Needs reassignment" filter.

## Task
| Field | Notes |
| --- | --- |
| campaign_id | Required. A task belongs to one campaign (and therefore one talent) |
| title | Required, up to 200 characters |
| assignee_id | Optional active user with access to the campaign |
| due_date | Optional date (Africa/Lagos day) |
| status | `todo` / `in_progress` / `done` |
| notes | Optional text |
| completed_at | Set when status becomes done; cleared if reopened |
| created_at, created_by, completed_by, sort_order | Created by and completed by are the signed-in users |

- No subtasks, comments, attachments, priorities or dependencies in v1.
- **Overdue** is derived, never stored: `due_date` is before today (Africa/Lagos) and status is not done.
- A task can be deleted by any user who can see it. Tasks hold no money, so deletion is a real delete, after a confirmation.

## Where it appears
1. **Campaign page → Tasks tab:** list grouped by status (To do, In progress, Done), each row showing title, assignee, due date and an overdue marker. Add task inline (title, optional assignee and due date), tick to complete, drag or arrow to reorder, click to edit notes. The Overview tab gets a small summary: open, overdue, done counts.
2. **Tasks page (sidebar):** all tasks across campaigns, filters for assignee, campaign, talent, status and a date range (Overdue, Due today, This week, No date). Grouped by due date. An **Assigned to me** shortcut uses the signed-in user, and admins also get **Needs reassignment**.
3. **Company overview:** one small card, "Tasks": open, due this week, overdue. Links to the Tasks page. Respects the talent and campaign filters; the date range applies to due date.
4. **Talent profile:** task counts per campaign are shown on the existing campaigns tab.

## Templates
- **Settings → Task templates** holds one or more named checklists (for example "Standard campaign"), each a list of titles with an optional due date offset (days from campaign start or end).
- On a campaign, **Add from template** inserts the template's tasks as ordinary tasks. Editing a template later does not change tasks already added.
- A new campaign can optionally start from a template (an unticked option on the create form).

## Campaign status rules
- Tasks can be added to a campaign in any status. Cancelling or completing a campaign does not close its tasks; the Tasks tab shows a banner "Campaign is completed, N tasks still open" so nothing is silently orphaned.
- Open tasks of cancelled campaigns are excluded from the Tasks page by default (a filter shows them).
- A deactivated talent's campaign tasks stay visible; the talent deactivation rules (PRD §6.1) are unchanged because tasks carry no money.

## Money and records
Tasks hold no money and do not affect any calculation. They are not linked to invoices, expenses, payouts or posts in v1 (a task may mention an invoice number in its title or notes).

## Edge cases
- Assignee deactivated: existing tasks keep the name, shown as "(deactivated)", and appear under "Needs reassignment".
- Talent or the Tasks page removed from the assignee: the task stays assigned but they can no longer open it; the admin is warned when making the change (team FRD).
- Due date in the past on create: allowed, shown overdue immediately.
- Many tasks on a campaign (200+): the tab paginates the Done group, newest first.
- Two people editing at once: last write wins per task; the edit shows the updated time.
- Template with date offsets and a campaign missing the start or end date: the tasks are added without a due date and the user is told which ones.

## Out of scope for v1
- Notifications, reminders, email digests and WhatsApp task creation (PRD §4 non-goal on notifications).
- Subtasks, comments, attachments, dependencies, recurring tasks, calendars, time tracking.
- Tasks that are not tied to a campaign (for example a talent-level or company-level to-do).

## Open questions
- **Notifications:** whether overdue tasks should reach people on WhatsApp or email. Left out to respect the no-notifications rule.
- **Talent-level tasks:** whether tasks not tied to a campaign (for example "follow up on advance") are needed; the answer depends on how the operations team actually works.
