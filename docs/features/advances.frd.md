# FRD: Advances

**PRD refs:** §6.4, §7.3, §7.4, FR-11, FR-12, FR-12a · **Milestone:** 3

## Purpose
Track cash the agency gives a talent (e.g. ₦1,000,000 emergency) and what has been recovered, repaid or forgiven.

## Model
One **signed ledger per talent** (PRD §6.4). Entry types: `advance` (+), `deduction` (−, from a payout), `cash_repayment` (−), `write_off` (−, reason required), `reversal` (opposite sign, points at the entry it reverses).

- **Balance = SUM(amount)** of the talent's entries. The profile card and the ledger read the same sum.
- The balance can never go below zero. The check runs in the same transaction as the write.
- "Oldest unrecovered advance" is computed at read time by applying deductions and repayments to advances first-in-first-out. No allocation rows are stored.

## Requirements
- Create an advance on a talent: amount, date, reason. Not tied to a campaign.
- Talent Advances tab shows the ledger (advance, deduction linked to its payout, cash repayment, write-off, reversal) with a running balance and the oldest unrecovered advance.
- **Record cash repayment:** amount (capped by the balance), date, method, reference.
- **Write off:** amount (capped by the balance) and a required reason. Shown distinctly in the ledger and counted in the overview as "written off", not as recovered.
- An advance is also created automatically when a recoverable line is saved on a talent-level expense (PRD §6.3); it is linked to that expense and reversed if the expense is voided.

## Rules
- Entries are never edited or deleted. A mistaken advance is corrected with a `reversal` entry, allowed only if the balance stays at or above zero.
- Recovery through payouts is manual, chosen at payout time, with the deduction pre-filled (payouts FRD).
- Cash repayment and write-off are allowed for **deactivated talents**. New advances are not.

## Acceptance criteria
- After a ₦1,000,000 advance and a ₦300,000 deduction, the balance reads ₦700,000.
- A ₦200,000 cash repayment then a ₦500,000 write-off brings the balance to ₦0.
- A repayment or write-off larger than the balance is rejected.
- Balance on the profile card matches the ledger sum.
- Voiding the payout that made a deduction restores the balance by exactly that amount.
- A recoverable line on a talent-level expense creates an advance of that line's amount.
