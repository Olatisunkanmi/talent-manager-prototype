# FRD: Expenses

**PRD refs:** §6.3, §7.4, FR-8 to FR-10a · **Milestone:** 3

## Purpose
Record what a shoot or a talent costs, created in-app in the same way as an invoice. No receipt uploads.

## Requirements
- Expense form: title, payee (typed), date, line items (category, description, quantity, unit cost, **recoverable from talent** per line), notes.
- Categories: bouncer, driver, videographer, food, equipment, other.
- Auto-numbered (EXP-0001), sequential, `UNIQUE`, never reused.
- **Recoverable from talent is per line** (default off). To bill the talent for lunch but absorb the bouncer, put both on one expense and flag only the lunch line.
- **Inline quick-add** on the campaign Expenses tab: one row (category, description, amount, recoverable) that creates a one-line expense with title from the description and today's date, without opening the full editor.
- Two contexts: under a campaign, or at talent level with no campaign.
- Printable preview page.
- Campaign Expenses tab lists expenses with totals split into recoverable and absorbed.

## Rules
- Expenses can also be created from WhatsApp through a guided flow or the one-line `quick` shortcut; they carry a `source` badge and follow the same rules (whatsapp FRD).
- Total = sum of lines, stored in kobo (bigint).
- Only recoverable lines reduce the talent's share, and only on **campaign** expenses.
- **Talent-level expenses:** a recoverable line auto-posts an `advance` entry in the talent's ledger for that line's amount, linked to the expense (advances FRD). It is then recovered like any other advance. Voiding the expense reverses that entry.
- **Edit and void:** until the first payout exists on the campaign, a campaign expense can be edited. Once a payout exists it is **void-only**; void it and enter a corrected expense. Voided expenses stay visible and are excluded from totals. Expenses are never hard-deleted.

## Edge cases
- Voiding or replacing an expense after a payout exists: the over-paid warning appears if the talent's earned share drops below what was paid out (PRD §7.4).
- Zero or negative line amounts: rejected.
- Expenses on a completed or cancelled campaign: blocked until the campaign is reopened (PRD §7.6).

## Acceptance criteria
- Toggling recoverable on a line changes the campaign's earned figure immediately (before the first payout).
- One expense with a ₦150,000 recoverable line and a ₦100,000 absorbed line shows ₦150,000 recoverable and ₦100,000 absorbed.
- Expense numbers are never duplicated or reused.
- A ₦15,000 expense can be added from the quick-add row in under 10 seconds.
- A recoverable line on a talent-level expense appears as an advance on the talent's ledger.
