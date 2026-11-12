# Talent Manager: Product Requirements Document

**Status:** Draft v0.4 (v0.2 challenge fixes; v0.3 adds WhatsApp and post insights; v0.4 adds insights date ranges, content style, talent rollup and CSV export)
**Date:** 2026-11-09
**Owner:** Agency admin

---

## 1. Overview

Talent Manager is a web application for an influencer / creative talent management agency. The agency keeps a roster of creators (food, travel, lifestyle, etc.), wins deals from brands (e.g. DoorDash, Uber, clothing and makeup brands), assigns each deal to the talent whose niche fits, and then has to track the money: what the brand pays, what the shoot cost, what the talent is owed, and what the agency has advanced to the talent.

This app is the agency's financial and operational back office, organised around each talent.

## 2. Problem

- Money for each talent is scattered across chats, spreadsheets and memory.
- It is hard to see the profit on a given campaign once shoot costs (bouncers, drivers, videographers, lunch) are counted.
- Advances to talent (e.g. ₦1,000,000 for an emergency) are easy to forget and hard to recover cleanly from later payouts.
- Invoices to brands are written and sent by hand, outside any system tied to the campaign.

## 3. Goals

1. One profile per talent, with fast switching between talents.
2. Create campaigns under a talent and track their lifecycle.
3. Log every expense against a campaign, and optionally flag lines as recoverable from the talent.
4. Write and send invoices to brands from inside the campaign, immediately.
5. Give cash advances to a talent and track the running balance until it is recovered, repaid or written off.
6. Calculate the talent payout and the agency profit per campaign.
7. Create invoices and expenses from WhatsApp, step by step.
8. Attach the posts made for a campaign by link and track how they perform.

## 4. Non-goals (v1)

- Per-page view-only or edit levels, approval workflows, two-factor authentication or SSO, and per-campaign access scoping (v1 has users, an admin and member role, a page checklist and per-talent scope: see `docs/features/team.frd.md`).
- Talent or brand login portals.
- Multi-currency and exchange rates (Naira only; see §9).
- A brand directory or CRM (brand details are typed per invoice; autocomplete from earlier entries is allowed).
- Receipt or document uploads on expenses, and CSV statement exports. (A full-data backup export is a separate requirement, see §11.2. The post insights CSV in §8.11 is an exception: it exports posts and their metrics, not financial statements.)
- Payment gateway integration (payments are recorded manually).
- Contract and deliverable management, content approval workflows.
- Tax computation (VAT/WHT).
- Mobile native apps (the web app should be responsive).
- **One talent per campaign.** A deal where one brand books several talents under one fee and one invoice must be entered as one campaign per talent. A multi-talent deal entity is not in v1.
- **Percentage commission only.** Flat-fee, fixed talent-rate and brand-pays-talent deals are not supported; they cannot be modelled without distorting the percentage.
- A notifications centre, and proactive WhatsApp messages (reminders, alerts).
- Talent logins. The one-time account-connect link for insights is a consent page, not a portal.
- Note attachments, version history and notes outside a campaign.
- Task notifications, reminders, subtasks, comments and attachments, and tasks outside a campaign.
- Pipeline tracking for inquiries (replied, converted, linked to campaigns), and storing email bodies or attachments.
- Third-party scraping services for post metrics; X/Twitter metrics; follower or audience analytics.

## 5. User

**Agency admin.** Runs the agency, creates talents, manages the team and company settings, and can do everything members can.

**Team member.** A colleague invited by an admin. Sees only the pages an admin ticked and only the talents assigned to them. Creates and edits campaigns, expenses, invoices, advances, payouts, tasks and notes within that access.

## 6. Core concepts and data model

```
Talent ──< Campaign ──< Expense ──< ExpenseLine
   │            ├──< Invoice ──< InvoiceLine
   │            │        ├──< Payment (brand paying the agency; has commission_amount)
   │            │        └──< InvoiceSend (send / resend log)
   │            ├──< Payout (agency paying the talent; talent_id kept on the row)
   │            └──< Post ──< PostSnapshot (views, likes, ... append-only)
   ├──< SocialAccount (connected platform account, for insights)
   └──< AdvanceLedgerEntry (signed: advance, deduction, cash_repayment, write_off, reversal)
```

**Conventions for every table**
- Amounts: `bigint` kobo (never int4; int4 overflows near ₦21.5M). Commission rates: integer **basis points** (20% = 2000).
- Every financial row has `created_at` and `created_by`.
- **Money records are immutable.** Payments, payouts, advance ledger entries and expenses are never edited in place or hard-deleted once they affect another record. Mistakes are corrected with a **reversal row** that points at the original (`reverses_id`); the original stays visible. See §7.4.
- Foreign keys on financial rows are `ON DELETE RESTRICT`. `campaign.talent_id` never changes after creation. Rows that carry both `campaign_id` and `talent_id` (expense, payout) use a composite foreign key `(campaign_id, talent_id)` so they cannot disagree with their campaign.
- Business timezone is **Africa/Lagos**. Dates (issue, due, payment, expense) are calendar dates in that zone; "today", overdue and period boundaries are evaluated in it.

### 6.1 Talent
| Field | Notes |
| --- | --- |
| name | required |
| niche | e.g. food, travel, lifestyle, beauty (free text or tag) |
| email, phone | optional |
| default_commission_bps | default for new campaigns of this talent; pre-filled from the company default when the talent is created |
| notes | optional |
| contract_start, contract_end | dates; contract_end optional and **informational only** |
| status | active / deactivated. **The single source of truth** for what is allowed (see §8.1 and the talents FRD) |
| deactivated_reason | optional; cleared on reactivation |

Derived (definitions in §7.3): earned, paid out, outstanding advance balance, owed to talent, active campaigns.

### 6.2 Campaign
| Field | Notes |
| --- | --- |
| talent_id | required, **immutable** |
| title | required |
| brand_name | typed text (no brand table in v1) |
| description | optional |
| agreed_amount | the deal value in ₦; used only for a warning when invoices exceed it |
| commission_bps | defaults from the talent (see precedence in §7.5), overridable; **frozen after the first payment** is recorded |
| currency | `NGN` in v1. Children (invoice, payment, expense, payout) inherit it and must match (see §9) |
| start_date, end_date | optional |
| status | draft / active / completed / cancelled, with the transitions in §7.6 |

Derived: total expenses, invoiced, received, commission, earned, paid out, available, agency profit, post views and engagement rate (§6.10).

### 6.3 Expense
Expenses are **created in the app, the same way invoices are** (a form with line items and an auto-generated number). There are no receipt or file uploads in v1. A one-line expense can be added with the **inline quick-add** row on the Expenses tab, which fills title, date and payee with defaults.

| Field | Notes |
| --- | --- |
| expense_number | auto-generated, sequential, never reused (e.g. EXP-0001) |
| campaign_id | set for campaign expenses, null for talent-level expenses |
| talent_id | always set; for campaign expenses it must equal the campaign's talent (composite FK) |
| title | short label, e.g. "Shoot day 1 costs" |
| payee | who was paid (optional, typed) |
| date | required |
| notes | optional |
| status | recorded / void |
| lines | **ExpenseLine**: category (bouncer, driver, videographer, food, equipment, other), description, quantity, unit cost, **recoverable_from_talent** (boolean per line, default false) |

Derived: total = sum of lines; recoverable total = sum of lines flagged recoverable.

Rules:
- Campaign expenses are absorbed by the agency by default. Lines flagged `recoverable_from_talent` reduce the talent's share on that campaign's payouts (§7.1).
- **Talent-level expenses (no campaign)** cannot be recovered through a campaign payout. A line flagged recoverable on a talent-level expense **auto-posts an `advance` entry** in the talent's advance ledger for that line's amount (linked to the expense), and is recovered like any other advance. Voiding such an expense posts a `reversal` of that entry.
- Until the first payout exists on the campaign, a campaign expense can be edited. After that it is **void-only**: void it and enter a corrected one.

### 6.4 Advance ledger
A cash advance to a talent, not tied to a campaign (e.g. a ₦1,000,000 emergency food advance). Everything that changes what a talent owes the agency is one row in a **per-talent signed ledger**.

| Field | Notes |
| --- | --- |
| talent_id | required |
| type | `advance` (+), `deduction` (−, created by a payout), `cash_repayment` (−, talent pays back in cash), `write_off` (−, agency forgives; reason required), `reversal` (opposite sign of the row it reverses) |
| amount | signed bigint kobo |
| date, reason | |
| payout_id | set for `deduction` rows |
| expense_id | set for advances auto-posted from a talent-level expense (§6.3) |
| reverses_id | set for `reversal` rows |

**Outstanding advance balance = SUM(amount)** for the talent. It can never go below zero: the check runs in the same transaction as the write. "Oldest unrecovered advance" is computed at read time by applying repayments and deductions to advances first-in-first-out. A balance left when a contract ends can be settled by `cash_repayment` or forgiven by `write_off`.

### 6.5 Invoice
Created inside a campaign and sent to the brand.

| Field | Notes |
| --- | --- |
| campaign_id | required; fixed once the invoice is sent (a draft may be reassigned) |
| invoice_number | auto-generated, never reused, `UNIQUE` (e.g. INV-0001) |
| bill_to_name, bill_to_email, bill_to_address | typed per invoice |
| issue_date, due_date | required |
| lines | description, quantity, unit price |
| notes / payment instructions | terms |
| status | **draft / sending / sent / void only.** Paid, part-paid and overdue are derived (§7.2) |
| sent_at, voided_at | set by the system |
| issuer snapshot | company name, address, contact, bank details, logo reference, copied onto the invoice **at send** (drafts show live Company settings) |
| pdf | generated once at send and stored; the stored file is what the brand received |
| delivery_status | informational, from email webhooks (delivered / bounced / complained) |

Derived: subtotal, total (no tax in v1), amount paid, balance due, payment state.

### 6.6 Payment (brand to agency)
Recorded against a **sent** invoice: amount, date, method, reference, and the **commission_amount** computed once at insert and stored (§7.1). Partial payments are supported. A payment is corrected by a reversal row (negative amount and negative commission), never edited or deleted. A payment cannot exceed the invoice's balance and cannot be recorded on a draft, sending or void invoice.

### 6.7 Payout (agency to talent)
Recorded against a campaign when the agency pays the talent.

| Field | Notes |
| --- | --- |
| campaign_id, talent_id | both required; composite FK to the campaign |
| payout_number | sequential, `UNIQUE` |
| gross_amount | chosen by the admin, up to *available* (§7.1); default = available |
| advance_deduction | chosen by the admin, up to the outstanding advance balance and the gross amount |
| net_paid | gross − advance deduction (the cash sent) |
| date, method, reference, note | |
| status | recorded / void (a void is a reversal row) |
| calculation snapshot | earned, commission, recoverable expenses, prior payouts, available and advance balance as they were at record time; plus the company details printed on the statement |
| idempotency_token | client-generated, `UNIQUE`, so a double-click or retry cannot record twice |

Recording a payout creates its `deduction` ledger entry in the same transaction.

### 6.8 Company settings
A single settings record for the agency, used on every invoice and payout statement.

| Field | Notes |
| --- | --- |
| company_name | required |
| logo | uploaded PNG or JPG, shown on invoice and PDF; stored under a **content-hashed key** so a replacement never changes an issued document |
| address, email, phone | shown on documents |
| bank_name, account_name, account_number | payment details printed on invoices |
| invoice_footer / default terms | optional |
| default_commission_bps | fallback for new talents |
| default_payment_due_days | pre-selects the due-date chip on new invoices |

Snapshot rule: **invoices** copy these values at **send**; **payout statements** copy them when the payout is recorded. Changing bank details later never alters an issued document.

### 6.9 WhatsApp assistant
Allowlisted admin phone numbers, step-by-step sessions and a message log, used to create invoices and expenses from WhatsApp. Invoices and expenses carry `source` (web / whatsapp). Full detail in `docs/features/whatsapp.frd.md`.

### 6.10 Post and snapshot
A **post** is a link to content published for a campaign.

| Field | Notes |
| --- | --- |
| campaign_id, talent_id | composite FK, as for expenses |
| platform | instagram / tiktok / youtube / other |
| url, external_id | URL unique per campaign |
| post_type, posted_at, label | |
| content_style | Optional free-text tag (e.g. "Money giveaway", "Skit"); used to compare which style performs best |
| collection_mode | api-public / api-connected / manual |
| status | tracking / unavailable / removed |

A **snapshot** is append-only: captured_at, views, reach, impressions, likes, comments, shares, saves (nullable), source (api / manual). The latest snapshot is the current value.

### 6.11 Social account
A talent's connected platform account used for insights: talent_id, platform, handle, encrypted token, expiry, status (connected / expired / revoked). Read-only insights access, granted by the talent through a one-time consent link. Full detail in `docs/features/insights.frd.md`.

### 6.12 Mailbox and inquiry
A **mailbox** is a registered address that receives a talent's business mail (talent_id, address, access_mode, status, last_heartbeat_at, consent confirmed date). An **inquiry** is a company's approach to a talent, logged by a separate intake service (n8n) from that mailbox: received date, sender, company, subject, short summary, confidence, thread, status (new / reviewed / dismissed), source (intake / manual). The full email body is never stored. Credentials stay in the intake service. Full detail in `docs/features/inquiries.frd.md`.

### 6.13 Task and team member
A **task** is a to-do on a campaign: title, optional assignee, optional due date, status (todo / in_progress / done), notes. Overdue is derived. A task's assignee is a real user (§6.15) who can see the campaign. Tasks hold no money. Full detail in `docs/features/tasks.frd.md`.

### 6.14 Note
A **note** is internal text on a campaign: title, type (meeting / call / general), note date, attendees, body (light Markdown), optional author label, pinned. Never shown on any brand-facing output. Full detail in `docs/features/notes.frd.md`.

### 6.15 User, role and access
A **user** is a person who signs in: name, email (unique), role (admin / member), status (invited / active / deactivated). A **member** has a **page checklist** (Overview, Talents, Campaigns, Invoices, Expenses, Advances, Payouts, Insights, Tasks, WhatsApp) and a list of **assigned talents**. Company settings and Team are admin-only. Users are invited by email link, never deleted (only deactivated), and every money record stores `created_by`. Permissions are checked on the server for every request. Full detail in `docs/features/team.frd.md`.

## 7. Money rules

All money is cash basis: nothing counts until a payment is recorded.

### 7.1 Per campaign

```
income_received       = sum of payments (net of reversals) on the campaign's invoices
commission            = sum of stored commission_amount on those payments
recoverable_expenses  = sum of recoverable lines on the campaign's non-void expenses
earned                = income_received − commission − recoverable_expenses     (talent's share before any payout)
paid_gross            = sum of gross_amount of the campaign's non-void payouts   (gross = net_paid + advance_deduction)
available             = max(0, earned − paid_gross)
over_paid             = max(0, paid_gross − earned)                              (shown as a warning state)

net_paid              = gross_amount − advance_deduction
absorbed_expenses     = sum of non-recoverable lines on non-void expenses
agency_profit         = commission − absorbed_expenses
```

- **Commission per payment:** `commission_amount = round_half_up(payment_amount × commission_bps / 10000)` computed once when the payment is recorded and stored on the payment. Sums over any date range therefore tie exactly, and later changes to a percentage never rewrite past income. Because the percentage is frozen after the first payment, all payments on a campaign use one rate.
- The admin may pay **any gross amount up to `available`**, not only the full amount. Instalment payments by the brand normally mean several payouts.
- The advance deduction is manual but **pre-filled** with `min(advance balance, gross amount)`. The admin may change it.
- `gross_amount`, available and the advance balance are **recomputed on the server inside the payout transaction**; the client's numbers are never trusted.
- Payouts are limited to money actually received, never the agreed amount.
- Recorded payouts keep their calculation snapshot; later edits do not rewrite them.

### 7.2 Invoice status
Stored: `draft`, `sending`, `sent`, `void`. Derived, in Africa/Lagos:
- **paid** when payments (net of reversals) ≥ total;
- **part-paid** when 0 < payments < total;
- **overdue** when due_date < today and balance > 0.

### 7.3 Definitions used across the app
- **Earned** (talent): sum of *earned* (7.1) over the talent's campaigns.
- **Paid out** (talent): sum of `net_paid` (cash actually sent). *Settled* = sum of gross.
- **Owed to talent**: sum of `available` over the talent's campaigns, as of today.
- **Outstanding advance balance**: the ledger sum (§6.4).
- **Net position** of a deactivated talent: owed to talent − outstanding advance balance. Shown as "owes ₦X" or "owed ₦X".

### 7.4 Corrections
No money record is edited or deleted after it has effect.
- A payment is reversed by a reversal payment. An invoice with payments cannot be voided until those payments are reversed.
- A payout is voided by a reversal that also posts a `reversal` of its deduction ledger entry.
- An expense is voided; a campaign with a payout allows void-only on expenses.
- A reversal that makes `paid_gross > earned` puts the campaign in the **over-paid** state; the admin sees the shortfall and can record it as a `cash_repayment` or `write_off` on the ledger.

### 7.5 Commission precedence
`company default_commission_bps` → pre-fills a new talent's `default_commission_bps` → pre-fills a new campaign's `commission_bps` (overridable until the first payment). Each level is a copy at creation, not a live link.

### 7.6 Campaign status
| From | To | Notes |
| --- | --- | --- |
| draft | active, cancelled | draft: expenses allowed; invoices can be written but not sent; no payouts |
| active | completed, cancelled | all operations allowed |
| completed | active (reopen) | new invoices and expenses blocked until reopened; payments and payouts still allowed |
| cancelled | active (reopen) | new invoices and expenses blocked; payments already received and payouts from them still allowed |

When total non-void invoices exceed `agreed_amount`, the invoice editor shows a warning (not a block).

### 7.7 Worked example (shared test fixture)
Talent has a ₦1,000,000 advance already on the ledger. Campaign: agreed ₦3,000,000, commission 20% (2000 bps). One expense with two lines: ₦150,000 recoverable, ₦100,000 absorbed.

| Step | Event | Commission | Earned | Paid gross | Available | Deduction | Net cash | Advance balance |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Brand pays ₦1,000,000 | 200,000 | 650,000 | 0 | 650,000 | | | 1,000,000 |
| 2 | Payout 1: gross 650,000, deduction 300,000 | | 650,000 | 650,000 | 0 | 300,000 | 350,000 | 700,000 |
| 3 | Brand pays ₦1,000,000 | 200,000 | 1,450,000 | 650,000 | 800,000 | | | 700,000 |
| 4 | Payout 2: gross 800,000, deduction 700,000 | | 1,450,000 | 1,450,000 | 0 | 700,000 | 100,000 | 0 |
| 5 | Brand pays ₦1,000,000 | 200,000 | 2,250,000 | 1,450,000 | 800,000 | | | 0 |
| 6 | Payout 3: gross 800,000, no deduction | | 2,250,000 | 2,250,000 | 0 | 0 | 800,000 | 0 |

Totals: income ₦3,000,000 = commission 600,000 + recoverable expenses 150,000 + deductions 1,000,000 + net cash to talent 1,250,000. Agency profit = 600,000 − 100,000 absorbed = **₦500,000**. Rounding check: a ₦333,333.33 payment at 20% stores a commission of ₦66,666.67 (half-up on kobo).

### 7.8 Expected data size
For test and performance targets: up to 50 talents, 500 campaigns, 5,000 invoices and 20,000 payments in total over three years.

## 8. Functional requirements

### 8.1 Talent management and switching
- FR-1: Create, edit, deactivate and reactivate talents. `status` alone decides what is allowed. A deactivated talent cannot get new campaigns or advances, but existing campaigns, invoices, expenses, payouts, cash repayments and write-offs keep working. The rule is enforced on the server in the same transaction as each write, not only in the UI. `contract_end` passing does not change the status; it shows a "contract ended" banner as a prompt to deactivate.
- FR-1a: The talent profile highlights active and recent campaigns so ongoing work can be tracked.
- FR-2: A persistent talent switcher (sidebar or header) lets the admin jump between talent profiles from any page.
- FR-3: The talent profile page shows summary cards (earned, paid out, owed to talent, advance balance, active campaigns) and tabs for **Campaigns, Advances, Expenses, Payouts and Posts**, plus a Connected accounts card.

### 8.2 Campaigns
- FR-4: Create a campaign under a talent with brand name, agreed amount and commission.
- FR-5: Commission defaults from the talent (§7.5) and can be overridden until the first payment.
- FR-6: Campaign page shows tabs for Overview, Expenses, Invoices, Payouts and Posts, plus the money summary from §7.
- FR-7: Update campaign status within the transitions in §7.6.

### 8.3 Expenses
- FR-8: Create expenses on a campaign using an expense form with line items, date, payee and a recoverable toggle **per line** (auto-numbered like invoices), plus an inline quick-add row. Edit until the first payout exists, void-only after.
- FR-9: Show campaign expense totals split into recoverable and absorbed.
- FR-10: Create a talent-level expense not tied to a campaign. Recoverable lines auto-post an advance (§6.3).
- FR-10a: Preview an expense as a printable page.

### 8.4 Advances
- FR-11: Create an advance for a talent with amount, date and reason.
- FR-12: The talent profile shows the advance ledger (advances, deductions, cash repayments, write-offs, reversals) and the running outstanding balance, plus the oldest unrecovered advance.
- FR-12a: Record a cash repayment or a write-off (with a reason) against an outstanding balance, including for a deactivated talent.

### 8.5 Invoices
- FR-13: Create an invoice from a campaign, with line items, and typed bill-to details. A global **New invoice** button asks for the campaign first.
- FR-14: Auto-number invoices sequentially: never a duplicate, never reused, and every number accounted for (a voided number stays visible as void). Numbers are issued from a counter row with a `UNIQUE` constraint.
- FR-15: Preview the invoice as a printable page and download it as PDF, with the agency logo and bank details from company settings.
- FR-16: **Send** shows a confirm-recipient step (email and total), then emails the invoice (stored PDF attached) to the bill-to email, and marks it `sent` with a timestamp **only after the email provider returns a message id**. While sending, the invoice is `sending`: locked and Send disabled. The send is idempotent.
- FR-16a: **Resend** a sent invoice or a corrected copy; each send is logged in the invoice's send log.
- FR-17: Record payments against a sent invoice, including partial payments. Payments are reversed, not edited.
- FR-18: Paid, part-paid and overdue are derived (§7.2), not stored.
- FR-19: Void an invoice (kept for audit, excluded from totals). Blocked while payments exist.
- FR-19a: **Correcting a sent invoice:** bill-to details, notes and dates can be edited on a sent invoice and sent again as a corrected copy under the same number. Lines and totals are editable the same way only while no payment is recorded; once a payment exists they are locked. Drafts can be reassigned to another campaign.
- FR-19b: **Collection actions:** "Record payment" and "Send reminder" on each row of the global Invoices list and of the overview's overdue list.

### 8.6 Payouts
- FR-20: Create a payout for a campaign, showing the §7 breakdown. Launched from a campaign or talent, the talent and campaign are pre-filled. A "Ready to pay" list shows campaigns where available > 0.
- FR-21: Choose the gross amount (up to available) and the advance deduction (pre-filled, capped by the advance balance and the gross amount).
- FR-22: Recording a payout is one transaction that creates the payout, its calculation snapshot and its deduction ledger entry. Voiding it reverses both.

### 8.7 Company overview
- FR-23: An elaborate home page showing all revenue and growth: headline KPIs (received, invoiced, agency income, agency profit, receivables, owed to talents, payouts, expenses, open advances), a revenue trend chart with period comparison, a growth panel, and a money-flow breakdown.
- FR-24: Every figure is filterable by date range, talents and campaigns (with a compare-to option), and the filters are kept in the URL.
- FR-24a: Breakdowns by talent, by brand, receivables aging, expenses by category, outstanding advances, a campaigns table and an active campaigns list.
- Full detail in `docs/features/company-overview.frd.md`.

### 8.10 WhatsApp assistant
- FR-29: Guided WhatsApp flows create a draft invoice (optionally sent after a second confirmation) and an expense, using the same rules as the web app.
- FR-30: Only allowlisted numbers are served; webhooks are signature-verified and idempotent.
- FR-31: Flows create nothing until the final confirmation; cancel and timeout leave no records.
- FR-32: Items created this way are marked as WhatsApp-sourced.

### 8.11 Post insights
- FR-33: Add post links to a campaign; the platform is detected and a collection mode chosen (public API, connected account, or manual).
- FR-34: Metric snapshots are pulled on a schedule (daily for 14 days, weekly to 90 days) or entered manually, always with an as-of time.
- FR-35: Campaign and talent Posts tabs show per-post and total views, engagement, and a trend.
- FR-36: The company overview has a Content performance section that respects the date, talent and campaign filters.
- FR-37: A talent can connect an Instagram or TikTok account once through a consent link; expired tokens fall back to manual with a Reconnect prompt.
- FR-38: Cost per 1,000 views and engagement rate are derived (definitions in the insights FRD).
- FR-64: The Insights page filters posts by posted date: 7, 30 or 90 days, this or last quarter, this year, all time, or a custom range. The range applies to every figure on the page.
- FR-65: Posts can carry a content style tag. The Insights page and each talent's Posts tab compare styles by engagement rate, average views, reach, likes and comments, so the agency can decide which style to make more of next quarter. It also shows reach by platform and the most liked and most commented posts.
- FR-66: Post performance rolls up to the talent through the campaign: totals per talent across all their campaigns, a ranked Performance by talent table on the Insights page, and the same review on the talent's Posts tab.
- FR-67: A post can be recorded with only its link (no figures), and the post table is searchable by talent, campaign, link and style, so past posts can be found without scrolling a social profile.
- FR-68: The posts in the current Insights view (range, platform, search) can be exported as CSV with a totals row, for a quarter-end record.

### 8.12 Inquiry tracking
- FR-39: A separately hosted intake service posts classified inquiries to the app through a signed, idempotent endpoint; the app stores no mailbox credentials and no email bodies.
- FR-40: Inquiries are logged per talent with sender, company, subject and a short summary; one inquiry per email thread.
- FR-41: A false positive is dismissed, never deleted; the admin can add a missed inquiry manually.
- FR-42: The talent profile has an Inquiries tab; the company overview has an Inquiries section (count, distinct companies, per talent, trend) respecting the date and talent filters.
- FR-43: Each mailbox shows when it was last checked and a warning when it has gone quiet, so silence is not read as "no inquiries".
- FR-44: Mailbox registration records that the talent consented to their inbox being read.

### 8.13 Campaign tasks
- FR-45: Each campaign has a Tasks tab where tasks are added, assigned, dated, reordered and moved between To do, In progress and Done.
- FR-46: Assignees are name labels from a Team list in settings; no extra logins or roles are introduced.
- FR-47: A Tasks page lists tasks across campaigns with filters for assignee, campaign, talent, status and due date, grouped by due date.
- FR-48: Overdue is derived from the due date (Africa/Lagos) and status, never stored.
- FR-49: Task templates can be added to a campaign in one step; later template edits do not change existing tasks.
- FR-50: The company overview shows open, due-this-week and overdue task counts.

### 8.14 Campaign notes
- FR-51: Each campaign has a Notes tab where notes are added, edited, pinned and deleted, with title, type, date, attendees and body.
- FR-52: Notes are internal only and never appear on invoices, PDFs, emails or WhatsApp messages.
- FR-53: Notes are searchable within the campaign and from the global search.
- FR-54: A note can start a task (title and campaign prefilled).
- FR-55: Notes are included in the full-data backup export and in data erasure for a talent.

### 8.15 Team management
- FR-56: Admins invite members by email with name, role, page checklist and assigned talents; the invite link is single use and expires in 7 days.
- FR-57: Members see only the pages ticked for them; each page owns its data, so numbers never appear elsewhere without the owning page.
- FR-58: Members see only their assigned talents everywhere (lists, links, search, switcher, totals); an unassigned talent is "not found".
- FR-59: Permissions are checked on the server on every request and take effect on the next request; deactivating a user ends their sessions immediately.
- FR-60: There is always at least one active admin; the last one cannot be deactivated or demoted.
- FR-61: Tasks are assigned to active users who can see the campaign; notes record the signed-in author.
- FR-62: Every money record stores who created and who voided it, and a Team activity list records invites, access changes and deactivations.
- FR-63: Password reset by email link exists for every user; each WhatsApp number is linked to one user and acts with that user's access.

### 8.8 Auth
- FR-25: Email and password sign-in for every user. All routes and every server action require a session, or a verified allowlisted WhatsApp sender linked to a user for the WhatsApp actions (§11.1).

### 8.9 Company settings
- FR-26: A settings page to edit company name, address, contact details, bank details, default terms, default commission and default payment-due days.
- FR-27: Upload and replace the company logo (PNG or JPG; the only file upload in v1).
- FR-28: Invoices snapshot these values at send; payout statements at record time.

## 9. Currency

v1 is Naira (₦) only. `currency` is stored on the **campaign** (default `NGN`); invoices, payments, expenses and payouts inherit it and a constraint requires them to match their campaign. Totals are never summed across currencies. Adding USD later is not free: it also needs exchange rates, per-currency commission and totals, and a second bank account on the company settings. Those are out of scope for v1 and the column only keeps the door open.

## 10. Key screens

1. **Login**
2. **Company overview** with filters (date range, talents, campaigns), revenue and growth, and breakdowns
3. **Talents list** with a create talent button
4. **Talent profile** with summary cards and tabs for Campaigns, Advances, Expenses, Payouts and Posts
5. **Campaign detail** with tabs for Overview, Expenses, Invoices, Payouts and Posts
6. **Invoice editor and preview** with Send, Download PDF and Record payment
7. **Invoices list** (global) with Record payment and Send reminder row actions
8. **Payout editor** showing the breakdown, gross amount and advance deduction; **Ready to pay** list
9. **Advance dialogs** (advance, cash repayment, write-off)
10. **Expense editor and preview** (same pattern as the invoice editor), plus inline quick-add
11. **Company settings** with logo upload, bank details and the WhatsApp allowlist
12. **Posts tab** (campaign and talent) and the **connect-account** consent page
13. **WhatsApp conversations** (not a screen; flows defined in the WhatsApp FRD)

## 11. Technical approach

| Area | Choice |
| --- | --- |
| Framework | Next.js (App Router), TypeScript |
| Database | Postgres (hosting provider not yet decided; Neon was the working assumption) |
| ORM | Drizzle |
| Email | Resend (or equivalent with idempotency keys and webhooks) |
| PDF | Server-rendered HTML to PDF |
| Auth | Users with email and password, session-based; roles, pages and talent scope checked per request (see team FRD) |
| UI | Tailwind with a component library (shadcn/ui) |
| WhatsApp | WhatsApp Business Cloud API (Meta), webhook receiver |
| Social data | YouTube Data API; Instagram Graph API and TikTok API through talent-granted tokens |
| Background jobs | Scheduled job runner for metric refresh and token renewal (for example Vercel cron or a queue) |
| Inquiry intake | Separate self-hosted n8n instance calling a signed intake endpoint (see inquiries FRD) |
| Hosting | Vercel (assumed) |

Notes: money stored as `bigint` kobo; documents numbered from a counter row (§6.5); environment secrets via env vars only. The transaction, locking and idempotency requirements for multi-row money writes are listed in `docs/challenge-resolutions.md` (item F4) and are deliberately left until the database provider is chosen.

### 11.1 Security baseline
- Every server action and route handler checks the session through one shared helper; middleware alone is not enough.
- Login is rate limited; sessions expire (absolute and idle); responses for authenticated pages send `Cache-Control: no-store`.
- PDF rendering escapes all user-typed text, runs with the network disabled, and never fetches remote URLs.
- Logo uploads accept PNG and JPG only, with a size limit (SVG is rejected unless sanitised).
- Bill-to email is validated, and Send shows a confirm-recipient step because the invoice carries bank details.
- Password reset by single-use email link; an operator script is the last resort for the admin.
- Authorisation is checked from the database on every request: role, page and assigned talent. Out-of-scope records return "not found". Invite and reset tokens are random, stored hashed, single use and expiring.
- SPF and DKIM must be set up on the sending domain before invoices are sent.

- WhatsApp: only allowlisted sender numbers; Meta signature check on every webhook; each message id processed once; no bank details typed or displayed in chat.
- Social tokens are encrypted at rest, scoped to read-only insights, revocable, and never shown in the UI or logs.
- Third-party platform terms apply; no scraping.

### 11.2 Data safety
- Money records are reversal-based (§7.4), so history is the audit trail; there is no generic audit-log table in v1. Money records also store `created_by` and the voiding user, and a Team activity list records access changes.
- A full-data backup export (not a statement export) and database point-in-time recovery or scheduled backups are required before real use.

## 12. Edge cases

- Advance deduction exceeding the outstanding balance or the gross amount: blocked.
- Payout gross above available: blocked. Available is never negative; an over-paid campaign shows a warning.
- Payment larger than the invoice balance, or on a draft, sending or void invoice: blocked.
- Editing a campaign expense after a payout exists: not allowed; void and re-enter.
- Deleting a talent, campaign or any financial record: not allowed. Deactivate the talent, cancel the campaign, void the record.
- Every invoice must belong to a campaign; there are no standalone invoices.
- Invoice email failure: the invoice returns to `draft` with the error shown and a retry. An invoice is never marked `sent` without a provider message id. A bounce arrives later and sets `delivery_status`, which is shown but does not change the status.
- Double-clicking Send, Record payout or Record payment must not create a duplicate.
- Campaign cancelled after partial payment: payments remain recorded; payout is still computed from money received.
- WhatsApp: a duplicate webhook, a stalled session or an ineligible campaign never creates a record; cancel and timeout create nothing.
- Insights: a deleted or private post becomes unavailable and keeps its last snapshot; an expired token falls back to manual entry.
- A talent leaves with an unrecovered advance: settle with a cash repayment or write it off; it does not stay on the overview forever.

## 13. Success metrics

- Creating and sending an invoice from a campaign takes under 2 minutes.
- Every talent's advance balance is visible and accurate at any time.
- Per-campaign profit is visible without a spreadsheet.
- Campaign, talent and overview figures for the same filters agree to the kobo.
- A new invoice can be saved as a draft from WhatsApp in under 90 seconds.
- Every tracked post shows current numbers with an as-of time, and campaign performance is visible without opening each platform.

## 14. Milestones

1. **Foundation:** project setup, database schema, users and sign-in, auth and security baseline, layout with talent switcher.
2. **Talents and campaigns:** CRUD, profile pages, status rules. **Team management:** invites, roles, page checklist, talent scope, Team page.
3. **Expenses and advances:** logging, per-line recoverable, advance ledger with repayment and write-off.
4. **Invoices:** editor, PDF, email send, payments, derived statuses, collection actions.
5. **Payouts:** breakdown, advance deductions, ready-to-pay.
6. **Company overview and polish:** filters, revenue and growth, breakdowns, responsive pass.
7. **WhatsApp assistant:** webhook, allowlist, invoice and expense flows.
8. **Post insights:** posts, manual snapshots, YouTube pull, connected Instagram/TikTok, scheduler, overview section.
9. **Inquiry tracking:** mailbox registration, signed intake endpoint, inquiries tab, overview section, health warnings. The n8n workflows are built and hosted separately.
10. **Campaign tasks:** team list, tasks tab, tasks page, templates, overview card.
11. **Campaign notes:** notes tab, editor, search, create-task action.

## 15. Open questions

Resolved decisions:
- Invoices carry the agency logo and bank details, via a company settings page.
- Expenses are created in-app like invoices; no receipt uploads.
- No CSV statement export in v1.
- Drizzle is the ORM.

Open for the new features:
- **Platform approvals:** Meta Business verification (WhatsApp) and Instagram/TikTok app review must be requested early; they can take weeks and gate the automatic modes. Manual entry covers the gap.
- **Send from WhatsApp:** v1 allows Send after a second confirmation. Confirm this is wanted, or restrict WhatsApp to drafts.
- **Brand-facing performance report:** sharing campaign results with the brand is a natural next step, not in v1.
- **Post links from WhatsApp:** not in v1.

Open for inquiry tracking (details in `docs/features/inquiries.frd.md`):
- **Inbox setup model:** talent's own mailbox, agency-created address, or forwarding; to be learned from how the agency operates. The app does not depend on the answer.
- **Classifier data exposure:** whether a hosted language model may read the emails.
- **Follow-up pipeline and brand matching:** left out of v1.

Open for campaign tasks (details in `docs/features/tasks.frd.md`):
- **Team access (details in `docs/features/team.frd.md`):** view-only levels, payout approvals, two-factor authentication and per-campaign scoping are left out of v1.
- **Notifications and talent-level tasks:** left out of v1.

Open for campaign notes (details in `docs/features/notes.frd.md`):
- **Talent-level notes and attachments:** left out of v1; attachments depend on the file store decision.

Open (deferred from the challenge review, critics disagreed; see `docs/challenge-resolutions.md`):
- **Overview scope:** keep all 12 sections, or ship a lean home first. Revisit once the §7 definitions are implemented.
- **Ledger scope:** the advance ledger is the only ledger for now. A single ledger for payments and payouts too is not adopted.
- **Withholding tax and short payment:** how to clear invoices a brand pays short because of tax withheld. Depends on whether the brands withhold tax on most invoices.
- **Brand as free text:** "Revenue by brand" groups typed names; revisit a brands table if that metric is required.
- **Pre-payment:** whether the agency may pay a talent before the brand pays and link it to the campaign. For now an early payment is an ordinary advance.

The logo is the only file the app stores, so it needs a place to live (for example an object-storage bucket). To be chosen at kickoff, together with the database provider.
