# FRD: Talents and Profile Switching

**PRD refs:** §6.1, §7.3, FR-1 to FR-3 · **Milestone:** 2 · **Mockup:** talent profile view

## Purpose
Give each creator their own profile, make it fast to move between them, and let the agency see and track each talent's active campaigns.

## Requirements
- Talents list with search and a status filter (Active / Deactivated); create and edit.
- Fields: name, niche, email, phone, default commission %, contract start date, contract end date (optional, informational), notes, status.
- A new talent's default commission is pre-filled from the company default.
- **Talent switcher** in the sidebar, available on every page; switching keeps you on the same tab where possible. Only active talents appear by default, with a "Show deactivated" option.
- **Profile page:**
  - Header: avatar, name, niche, status badge, contract dates, Edit and Deactivate actions.
  - Summary cards: total earned, paid out (cash), **owed to talent**, outstanding advance balance, active campaigns.
  - Tabs: **Campaigns, Advances, Expenses, Payouts, Posts**.
  - **Connected accounts** card: Instagram/TikTok accounts connected for insights, status, Connect / Reconnect / Disconnect, and a button to copy the one-time connect link (insights FRD).
  - **Active and recent campaigns** (default tab): each row shows campaign title, brand, status, dates, invoiced vs received, and a settlement strip. Active and ongoing campaigns sort first, then recently completed ones. A "New campaign" button creates a campaign under this talent.

## Deactivation (contract end)
Talents are **deactivated**, never deleted, when their contract ends.

**`status` is the only source of truth.** `contract_end` is informational. When `contract_end` has passed and the talent is still active, the profile shows a "Contract ended" banner that offers Deactivate; nothing changes automatically.

- Deactivate dialog collects: contract end date (defaults to today) and an optional reason.
- The dialog warns about open items: active campaigns, unpaid invoices, amount available to pay, and any outstanding advance balance (with the net position, "owes" or "owed").
- Deactivating does **not** cancel active campaigns or erase the advance balance.

**What a deactivated talent can and cannot do.** Enforced on the server, in the same transaction as each write, not only by hiding buttons.

| Operation | Active | Deactivated |
| --- | --- | --- |
| Edit profile, view everything | yes | yes |
| New campaign | yes | **no** |
| New advance | yes | **no** |
| Existing campaigns: expenses, invoices, payments, status changes | yes | yes |
| Payouts and advance deductions | yes | **yes** |
| Cash repayment, write-off | yes | **yes** |
| Appears in the switcher | yes | only with "Show deactivated" |
| Appears in payout pickers and the Ready to pay list | yes | **always**, with an owes/owed badge |

- A deactivated talent shows a "Deactivated" badge and the end date, and remains fully viewable.
- **Reactivate** restores the talent to active and clears the deactivation reason and the contract end date.

## Acceptance criteria
- Switching talent updates every figure on screen to that talent only.
- Summary cards match the underlying campaign, advance and payout records.
- The Campaigns tab lists active campaigns first and shows a settlement strip per row.
- A deactivated talent rejects new campaigns and new advances (also when the request bypasses the UI) but still accepts payouts, cash repayments and write-offs.
- A deactivated talent with money owed to or by them is findable in the payout picker.
- A talent whose `contract_end` has passed stays active until the admin deactivates them, and shows the banner.
- Reactivating clears the reason and end date.
