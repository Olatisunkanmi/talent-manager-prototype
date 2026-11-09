# FRD: Campaigns

**PRD refs:** §6.2, §7.5, §7.6, FR-4 to FR-7 · **Milestone:** 2

## Purpose
Represent a brand deal assigned to **one** talent, and be the hub for its expenses, invoices and payouts. A deal that books several talents under one fee is entered as one campaign per talent (PRD §4).

## Requirements
- Create a campaign from a talent profile: title, brand name (typed, with autocomplete from earlier campaigns), description, agreed amount, commission %, start/end dates.
- Commission defaults from the talent (company default → talent → campaign, PRD §7.5) and can be overridden **until the first payment is recorded**. After that it is locked and shown read-only.
- Commission is stored as basis points (20% = 2000); the form shows a percentage.
- `talent_id` is **immutable**. To move a deal to another talent, cancel it and create a new one.
- Status: draft, active, completed, cancelled, with the transitions in PRD §7.6. Status gates money operations:
  - draft: expenses allowed; invoices can be written but not sent; no payouts.
  - active: everything allowed.
  - completed / cancelled: no new invoices or expenses until reopened; payments already due and payouts from received money are still allowed.
- Campaign page tabs: Overview, Expenses, Invoices, Payouts, Posts (post links and performance, see insights FRD).
- Overview shows the money summary and settlement strip: income received, commission, recoverable expenses, absorbed expenses, earned, paid gross, available, agency profit (rules in PRD §7.1). An over-paid warning appears when paid gross exceeds earned.
- **Tracking:** active and ongoing campaigns are surfaced first on the talent profile (Campaigns tab) and on the overview, so the company can follow what is running; completed campaigns stay as recent history.
- **Invoices link to campaigns:** all invoices live under a campaign (see invoices FRD); the Invoices tab lists them with status and balance.
- A warning (not a block) shows in the invoice editor when total non-void invoices exceed `agreed_amount`.

## Edge cases
- New campaigns cannot be assigned to a deactivated talent; this is checked server-side in the same transaction that creates the campaign.
- Existing campaigns of a talent who is later deactivated continue normally.
- Cancelled campaign with payments received: payments remain and a payout can still be made from them.
- Changing commission after a payment: not possible. Payments already carry their stored commission.
- Campaigns with financial records cannot be deleted; cancel them.

## Acceptance criteria
- A campaign cannot exist without a talent, and its talent cannot be changed.
- Summary figures update when expenses or payments change.
- Editing the commission field after the first payment is rejected.
- A draft campaign cannot send an invoice or record a payout.
- An expense or payout cannot be recorded against a talent different from the campaign's.
