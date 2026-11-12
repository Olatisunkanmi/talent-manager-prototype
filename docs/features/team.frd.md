# FRD: Team Management

**PRD refs:** §5, §6.15, FR-56 to FR-63, §11.1 · **Milestone:** 1 (users and sign-in) and 2 (Team page, scoping) · **Supersedes:** the single-login rule in the auth FRD

## Purpose
The agency is more than one person. Admins invite colleagues, decide which pages each person can open and which talents they can see, and assign tasks to real people. Because every talent's earnings, bank details and advances sit in this app, access control is enforced on the server for every request, not only hidden in the interface.

## Roles and access
There are two roles.
- **Admin:** sees everything, manages the team, and is the only role that can reach Company settings, Team, intake settings, the data export and task templates. There must always be at least one active admin.
- **Member:** sees only the pages an admin ticked for them, and only the talents an admin assigned to them.

### Assignable pages (the checklist)
Overview, Talents, Campaigns, Invoices, Expenses, Advances, Payouts, Insights, Tasks, WhatsApp. **Company** and **Team** are admin-only and are not on the checklist.

A ticked page gives full use of that page on the member's talents. There is no view-only level in v1 (see Open questions).

### What each page owns
Content is shown only if the member has the page that owns it. This applies inside pages too.

| Where | Needs the page |
| --- | --- |
| Campaign Expenses tab, expenses list | Expenses |
| Campaign Invoices tab, invoices list, invoice screens | Invoices |
| Campaign Payouts tab, payouts list | Payouts |
| Campaign Posts tab, Insights page | Insights |
| Campaign Tasks tab, Tasks page | Tasks |
| Campaign Notes tab, campaign overview, campaign list | Campaigns |
| Talent profile, talents list | Talents |
| Advances tab and ledger | Advances |
| Money panel on a campaign overview (income, commission, earned, available, agency profit) | Invoices or Payouts |
| Company overview sections | The page that owns each section's data (revenue and receivables need Invoices; payouts and profit need Payouts; content performance needs Insights; and so on). A section the member cannot see is not rendered and not computed for them |

A member without a needed page never sees its numbers elsewhere. For example, without Payouts they do not see "available to pay" on a campaign.

### Talent scope
- Each member has a list of **assigned talents**. Admins always see all talents.
- Everything tied to a talent follows that scope: the talent profile, campaigns, invoices, expenses, advances, payouts, posts, tasks and notes, the talent switcher, global search, and every overview total (computed over the member's talents only, never company-wide).
- A talent the member is not assigned is **invisible**, including through a direct link or API call. The response is "not found", the same as a record that does not exist, so its existence is not revealed.
- A new talent is visible to no member until an admin assigns it.
- Only admins create talents and assign talents to members. Members with the right pages can create campaigns, expenses, invoices, advances and payouts for their assigned talents.

## Team member (user)
| Field | Notes |
| --- | --- |
| name, email | Email unique, case-insensitive; used to sign in |
| role | admin / member |
| status | `invited` (not yet accepted), `active`, `deactivated` |
| pages | Checklist (members only) |
| assigned talents | List (members only) |
| created_at, invited_by, last_sign_in_at | |

- Deactivation is the only removal. A user is never deleted, because their name stays on the records they created.
- An admin cannot deactivate or demote themself if they are the last active admin.

## Inviting a member
1. Admin opens **Team → Invite member** and enters name, email, role, pages and talents (a few **presets**, such as Finance or Account manager, pre-tick the checklist and can then be edited).
2. The app emails an **invite link**. The token is random, stored hashed, valid for **7 days** and **single use**.
3. The invitee opens the link, sees the agency name, and sets a password that meets the password rules below. The account becomes `active` and they are signed in.
4. **Resend** issues a new link and invalidates the old one. **Revoke** cancels a pending invite.

Rules:
- The invite goes only to the typed address. The admin never sees, sets or resets a member's password.
- Inviting an email that already exists is refused ("already on the team").
- An invite for a deactivated user is a **reactivate**, which sends a fresh link and forces a new password.
- An expired or used link shows a neutral page and offers to ask the admin for a new invite.
- The invite email says who invited them and warns "If you did not expect this, ignore it".

## Sign-in, passwords and sessions
- Email and password, as in the auth FRD, for every user.
- Passwords: minimum 12 characters, checked against a short list of common passwords, stored with argon2 or bcrypt. Admins cannot see them.
- **Password reset by email link** now exists for all users (a single-use token valid for 1 hour, the response never reveals whether an email is registered). The operator reset script remains as the last resort for the admin.
- A member changes their own password from their profile (current password required); this signs out their other sessions.
- Rate limiting and lock-out apply per account and per IP, as before.
- **Permissions are checked on every request** from the database, never cached in the session. A change to someone's role, pages, talents or status takes effect on their next request. **Deactivation ends all their sessions immediately.**
- Idle timeout and absolute lifetime as in the auth FRD.

## Team page (admin only)
- Table: name, email, role, status (Invited / Active / Deactivated), pages summary ("6 of 10"), talents count, last sign-in.
- Row actions: Edit, Resend invite, Revoke invite, Deactivate or Reactivate.
- **Edit member:** role, page checklist, talent assignment (searchable multi-select with "assign all"), shown with a plain summary of what the person will see.
- Warnings on removal: taking a talent away from a member shows how many **open tasks assigned to them** exist on that talent; removing the Tasks page does the same. The admin chooses to keep (tasks stay but the member can no longer open them) or reassign.
- **Team activity** list: invites sent, accepted, revoked; role, page and talent changes; deactivations; password resets; failed sign-in lockouts. Each line shows who did it and when. It is append-only and visible to admins.

## Tasks and notes with real users
- A task's **assignee** must be an active user who can see that campaign (has the Tasks page and the talent). The picker lists only those users, so a task is never assigned to someone who cannot open it. Existing name labels are not assignees (see Migration).
- A task shows **created by** and, when completed, **completed by**.
- A note's **author is the signed-in user**, recorded automatically; the "Written by" picker goes away. A note shows author and **edited by**. Members can edit or delete only their own notes; admins can edit or delete any note.
- If an assignee is deactivated, their open tasks keep their name, show "(deactivated)" and appear in an admin's "Needs reassignment" filter on the Tasks page.
- The **Tasks page** adds an "Assigned to me" shortcut that uses the signed-in user (replacing the name-picker workaround).

## Who did it
Because several people now act on money, every money record stores **created_by**: invoices, payments, expenses, advance entries, payouts and reversals, plus the voiding user and time. These are shown on the record's detail page ("Recorded by Amaka Eze, 04 Nov"). This extends the reversal-based history in PRD §7.4; there is still no generic audit log beyond the Team activity list.

## WhatsApp
- Each allowlisted WhatsApp number is **linked to one user** by an admin and acts with that user's pages and talents. A flow only offers the user's own talents, and a flow for a page the user lacks is refused with "You don't have access to that."
- Unlinked numbers are not served. Deactivating a user disables their number immediately.

## Migration from the single-login design
- The existing single admin account becomes the first admin.
- Task assignee labels and note author labels from the earlier "Team (names only)" list are not accounts. After rollout, an admin sees a one-time **Match old names** step: each old name can be matched to an invited member or left as plain text. Unmatched tasks keep the name as text, show as "Unassigned (was: Name)" and can be reassigned. No history is lost.

## Edge cases
- A member with no pages ticked can sign in and sees "Ask an admin for access".
- A member with pages but no talents sees empty states with "No talents assigned yet".
- A page removed while the member is on it: the next request redirects them to their first permitted page.
- Two admins editing the same member: last save wins and the page shows when the member was last changed.
- An invited person who never accepts: the invite expires, the row shows "Invite expired" with Resend.
- Changing a member's email: not edited in place; revoke and re-invite, so ownership of the new address is proven.
- A talent is deactivated: it stays visible in scope with the existing deactivation rules.
- A user acts on a record (for example a payout) and is later deactivated: the record keeps their name.
- Global search, talent switcher and counts never include out-of-scope talents, so a member cannot discover them by searching.

## Out of scope for v1
- View-only versus edit levels per page.
- Approval workflows (for example a payout needing an admin's approval).
- Two-factor authentication and single sign-on.
- Per-campaign scoping (scope is by talent).
- Talent and brand logins.
- A generic audit log beyond the Team activity list and `created_by` fields.

## Acceptance criteria
- A member cannot reach a page that is not ticked: the route, the server action and the API all refuse it, whatever the interface shows.
- A member cannot read, search or count a record belonging to an unassigned talent, including by a direct link; the response is "not found".
- Overview totals for a member equal the sum over their assigned talents only.
- Deactivating a member signs them out on their next request and blocks sign-in.
- An invite link works once, expires after 7 days, and cannot be reused after a resend.
- The last active admin cannot be deactivated or demoted.
- A task cannot be assigned to a user who cannot see its campaign.
- Every money record shows who recorded it.

## Open questions
- **View-only level:** whether some members (for example a viewer who reads reports) need read-only pages. Left out to keep the first version small.
- **Approvals:** whether recording a payout should require an admin's approval, since it moves money.
- **Two-factor authentication:** worth requiring for admins and anyone with Payouts or Company access; decide before launch.
- **Scoping by campaign:** if a talent is shared by two account managers who must not see each other's campaigns.
- **Can members create talents?** Currently admin only, so assignment stays in admin hands.
