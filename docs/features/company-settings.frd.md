# FRD: Company (General) Settings

**PRD refs:** §6.8, FR-26 to FR-28 · **Milestone:** 1 · **Mockup:** company page

## Purpose
The company's general page. It stores the agency's identity and bank details once, so every invoice and payout statement looks right.

## Fields
Company name (required), WhatsApp allowlist (phone numbers allowed to create invoices and expenses), logo, address, email, phone, bank name, account name, account number, default invoice terms/footer, default commission % (pre-fills new talents), default payment-due days (pre-selects the due-date chip on new invoices).

## Requirements
- A single page titled "Company" with a form on the left and a live preview of the invoice header (logo, name, contact, bank line) on the right, using the same split-pane pattern as the document editors.
- One settings record only.
- Logo upload (**PNG or JPG**, size-limited) with preview and replace. SVG is rejected unless sanitised. Logos are stored under a content-hashed key, so replacing one never changes a document already issued.
- **Snapshot rule** (matches PRD §6.8):
  - **Invoices** copy company name, address, contact, bank details and the logo reference **when sent**. A draft shows the live settings.
  - **Payout statements** copy the same details **when the payout is recorded**.
  - Later edits never change a sent invoice or a recorded payout statement.
- Changing bank details is a sensitive edit: show a confirmation, and record who changed it and when.

## Edge cases
- Creating an invoice before settings exist: prompt to complete the Company page first.
- Oversized or wrong-type logo: reject with a clear message.

## Acceptance criteria
- Saved settings appear on the next draft, and on the PDF once that invoice is sent.
- Changing the bank account does not alter an already-sent invoice or a recorded payout statement.
- Replacing the logo does not change the logo on any issued document.
