# FRD: Invoices and Payments

**PRD refs:** §6.5, §6.6, §7.2, FR-13 to FR-19b · **Milestone:** 4 · **Mockup:** invoice editor

## Purpose
Write an invoice from inside a campaign, send it to the brand immediately, track payment, and chase late money.

## Campaign linkage
- **Every invoice belongs to exactly one campaign.** There is no standalone invoice.
- Invoices are created from the campaign's Invoices tab, or from the global **New invoice** button, which asks for the campaign first. The campaign (and therefore the talent) is shown in the editor breadcrumb.
- A **draft** may be reassigned to another campaign. Once sent, the campaign is fixed.
- The global Invoices page lists invoices across all campaigns, with campaign and talent columns, and links back to the campaign.
- Payments on an invoice count as income received for its campaign.

## Requirements
- Create from a campaign: bill-to name, email and address (typed, prefilled from the last invoice to the same brand), issue and due dates (due-date chips use the company's default payment-due days), line items (description, quantity, unit price), notes.
- **Numbering:** INV-0001 style, issued from a counter row, `UNIQUE`, **never reused**. "Gap-safe" means no duplicates and every number accounted for: a voided invoice keeps its number and shows as void. Invoices are voided, never deleted, so drafts that are abandoned are voided.
- Header shows company logo, address and bank details from the Company page. Drafts show live settings; they are **frozen onto the invoice at send**.
- Printable preview and PDF download. The PDF is generated **once at send and stored**; later downloads return that file.
- **Stored status:** `draft`, `sending`, `sent`, `void` only (with `sent_at`, `voided_at`). **Derived:** paid, part-paid and overdue, from payments and the due date in Africa/Lagos (PRD §7.2).
- **Send:**
  1. Shows a confirm-recipient step (email, total, brand).
  2. Moves the invoice to `sending`: the editor is locked and Send is disabled.
  3. Sends the email through the provider with a stored idempotency key, the stored PDF attached.
  4. Sets `sent` and `sent_at` **only after the provider returns a message id**. On failure it returns to `draft` with the error and a retry.
- **Delivery status:** bounces arrive later by webhook and set an informational `delivery_status` (delivered, bounced, complained) with a visible badge. They do not change the status.
- **Send log:** each send, resend and reminder is recorded (time, recipient, kind, provider id).
- **Record payment:** amount, date, method, reference. The server computes and stores the commission on the payment (PRD §7.1). Partial payments supported. Only on a `sent` invoice, never above the balance. The invoice row is locked while the payment is inserted.
- **Reverse payment:** a mistaken payment is corrected with a reversal row. The invoice shows both.
- **Void:** keeps the invoice for audit and excludes it from totals. **Blocked while payments exist**; reverse them first.
- **Correcting a sent invoice:**
  - Bill-to details, notes and dates can be edited, then **Resend corrected copy** (same number, logged).
  - Line items and totals can be edited the same way only while no payment exists. Once a payment is recorded they are locked.
  - A new number is needed only when the wrong invoice must be cancelled entirely: void it and create another.
- **Collection actions** on each row of the global Invoices list and of the overview overdue list:
  - **Record payment** (opens the payment dialog).
  - **Send reminder** (manual, uses the same email and is written to the send log).
- There is no notifications bell in v1.
- Invoices can also be created (and, after a second confirmation, sent) from WhatsApp; they carry a `source` badge. Rules are identical (whatsapp FRD).

## Edge cases
- Email failure: invoice returns to draft, error shown, retry available.
- Double-click on Send or Record payment: second action is ignored (idempotent).
- Payment larger than balance, or on a draft, sending or void invoice: blocked.
- Campaign of a deactivated talent can still be invoiced; invoices on a completed or cancelled campaign are blocked until it is reopened.
- Invoice total beyond the campaign's agreed amount: warning only.
- Mistyped recipient: caught by the confirm step; a bounce later shows on the badge.
- Prerequisite: SPF and DKIM configured on the sending domain.

## Acceptance criteria
- A new invoice can be created and sent in under 2 minutes.
- Payments on the invoice feed the campaign's income received, and each stores its own commission.
- An invoice cannot be created without a campaign.
- Sending twice (double-click, retry after timeout) delivers one email.
- An invoice is never `sent` without a provider message id.
- An invoice with a payment cannot be voided until the payment is reversed.
- Overdue status appears the day after the due date, in Lagos time, without any stored flag changing.
- Changing company bank details does not change a sent invoice or its stored PDF.
