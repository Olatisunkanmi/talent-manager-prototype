# FRD: Payouts

**PRD refs:** §6.7, §7.1, §7.4, §7.7, FR-20 to FR-22 · **Milestone:** 5 · **Mockup:** payout view

## Purpose
The company creates a payout to pay a talent their share of a campaign and recover advances.

## Entry points
- Sidebar "Payouts": list of all payouts, a **Ready to pay** list, and a New payout button.
- Campaign page, Payouts tab (talent and campaign pre-filled).
- Talent profile, Payouts tab (talent pre-filled).

## Formula (PRD §7.1)
```
earned     = income received − commission − recoverable expenses
paid_gross = sum of gross_amount of the campaign's non-void payouts   (gross = net paid + advance deduction)
available  = max(0, earned − paid_gross)
```
Gross, not net, is subtracted so a payout that recovered an advance is not paid twice.

## Requirements
- Payout editor uses the split-pane pattern: form on the left, payout statement preview on the right, sticky Cancel / Record payout footer.
- **Context is not re-asked.** When opened from a campaign or talent, those fields are pre-filled and locked. From the sidebar, pick a talent, then one of that talent's campaigns with available > 0.
- The form shows the breakdown: income received, commission, recoverable expenses, earned, already paid out (gross), **available**.
- The **settlement strip** visualises the split.
- **Gross amount:** any amount up to *available*; default = available. Partial payouts are normal when the brand pays in instalments.
- **Advance deduction:** **pre-filled** with min(advance balance, gross amount) and editable, capped by the talent's outstanding advance balance and the gross amount. The advance balance and the oldest unrecovered advance are shown beside it.
- Net paid = gross − advance deduction; record date, method, reference, and optional note.
- **Ready to pay** list: campaigns with available > 0, with talent, available amount and advance balance, each with a Pay action. A deactivated talent shows an "owes ₦X" or "owed ₦X" badge (PRD §7.3).
- Recording a payout is **one transaction** that: recomputes earned, paid gross, available and the advance balance on the server (the client's numbers are not trusted), rejects over-cap amounts, inserts the payout with a calculation snapshot and company details, and posts its `deduction` ledger entry.
- A client-generated **idempotency token** (`UNIQUE`) makes a double-click or retry record the payout once.
- Payouts are snapshots: later changes to expenses or commission do not rewrite them.
- Printable payout statement for the talent, using the company details copied at record time.
- Payouts list: payout number, talent, campaign, date, gross, deduction, net, status (recorded / void).
- **Void** a payout: creates a reversal. It also posts a `reversal` of the deduction ledger entry. If the advance had since been repaid so the reversal would not be possible, voiding is blocked with an explanation.

## Rules
- Works for deactivated talents too (their final settlement and advance recovery). They always appear in the payout picker and the Ready to pay list.
- Payout is limited to money actually received on that campaign's invoices.
- Campaign must not be `draft`. Completed and cancelled campaigns can still be paid from money received.
- The payout's talent must equal the campaign's talent (composite foreign key).
- If earned drops below paid gross (a payment is reversed, an expense voided), the campaign shows an **over-paid** warning; the shortfall can be recorded as a cash repayment or write-off on the talent's ledger.

## Edge cases
- Nothing received yet, or available is 0: payout is blocked.
- Gross above available, or deduction above cap: input rejected with the cap shown.
- Voiding a payout reverses its advance deduction.
- Two browser tabs recording the same payout: the second is rejected because available has changed.

## Acceptance criteria
- Payout figures match the worked example in PRD §7.7 step by step: 650,000 gross with 300,000 deduction, then 800,000 with 700,000, then 800,000 with none; total net cash ₦1,250,000.
- Advance balance drops by exactly the deduction saved.
- Two payouts on one campaign never pay more than earned in total.
- Opening the editor from a campaign does not ask for talent or campaign.
- The deduction field is pre-filled when the talent has an advance balance.
- A deactivated talent with ₦700,000 outstanding and ₦200,000 available appears in Ready to pay with an "owes ₦500,000" badge.
