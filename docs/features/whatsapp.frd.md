# FRD: WhatsApp Assistant (invoices and expenses)

**PRD refs:** §6.9, FR-29 to FR-32, §11.1 · **Milestone:** 7

## Purpose
Let the admin create **invoices** and **expenses** from WhatsApp, step by step, when away from the web app. WhatsApp is only a second front end: it calls the same server functions, rules and validations as the web forms. It adds no new money rules.

## Channel and access
- Official **WhatsApp Business Cloud API** (Meta). Prerequisites: a verified business account, a dedicated number, and the webhook URL registered with Meta.
- **Per-user authorisation:** each allowlisted number is linked to one user and acts with that user's pages and talents (team FRD); only phone numbers on the admin's allowlist (Company settings → WhatsApp) are served. Messages from any other number get no reply and are logged.
- Webhook requests are verified with the Meta signature header; unsigned or invalid requests are rejected. Each WhatsApp message id is processed once (`UNIQUE`), so Meta retries never create duplicates.
- The shared auth helper (PRD §11.1) accepts either a web session or a verified allowlisted WhatsApp sender. Nothing else can reach a money action.
- The app only **replies** inside the 24-hour customer-service window, so no pre-approved templates are needed in v1. Proactive messages (reminders, paid alerts) are out of scope.

## Conversation model
- A **session** per sender holds the current flow, the step and the answers so far. It expires after 30 minutes idle.
- Commands work at any step: `menu`, `back` (previous step), `cancel` (discard the session), `help`.
- Prompts use WhatsApp reply buttons (up to 3 choices) or list messages (up to 10 rows). When a list is longer, the bot asks the admin to type part of the name and shows matches.
- Amounts accept `800000`, `800,000`, `800k` and `1.5m`, in naira. Dates accept `today`, `tomorrow`, `+14`, or `23 Nov`. Anything unreadable gets one clear re-prompt that shows an example.
- Images, voice notes and documents are not accepted (no receipt uploads in v1); the bot says so.

**Menu:** 1 New invoice · 2 New expense · 3 Cancel.

## Flow A: New invoice
1. **Talent:** pick from active talents.
2. **Campaign:** pick from that talent's campaigns that can invoice (active; draft campaigns can only produce drafts). A completed or cancelled campaign is not listed, with a note on how to reopen it.
3. **Bill to:** if the brand has a previous invoice, offer "Use DoorDash Nigeria · billing@… ?" (Yes / Edit). Otherwise ask name, then email (validated), then address (or `skip`).
4. **Line items:** one message per line in the form `description, quantity, unit price` (for example `Instagram reel, 1, 800k`). After each: "Add another" / "Done". Maximum 15 lines.
5. **Due date:** buttons for the company's default (for example 14 days) and 7 / 30 days, or type a date.
6. **Summary:** talent, campaign, bill-to, lines, total, due date, and any warning (for example "Invoices now exceed the agreed amount").
7. **Choose:** **Save draft** · **Send invoice** · **Cancel**.
   - *Save draft* creates a draft invoice and replies with its number and a link to the web editor.
   - *Send invoice* asks one more confirmation that repeats the recipient email and total ("Send INV-0013 ₦1,000,000 to billing@doordash.example?") and then runs the normal send (PRD FR-16: `sending`, idempotent, `sent` only on provider confirmation). The bot replies with the result and attaches the PDF to the chat.

## Flow B: New expense
1. **Where:** "For a campaign" or "Talent-level (no campaign)".
2. **Talent**, then (if a campaign) **campaign**. Expenses can be added to draft and active campaigns; completed and cancelled ones are refused with a reason.
3. **Title**, then **payee** (`skip` allowed). Date defaults to today (buttons: Today / Yesterday / type a date).
4. **Lines:** for each line, ask category (buttons/list: bouncer, driver, videographer, food, equipment, other), then `description, quantity, unit cost`, then **Recoverable from talent?** (Yes / No). After each: "Add another" / "Done".
5. **Summary:** total, recoverable and absorbed split. For talent-level expenses with a recoverable line, add the note "This will be added to Ada's advance balance."
6. **Save** or **Cancel**. Save creates the expense with its normal number (EXP-…) and replies with the number and a web link.

**Shortcut:** `quick` starts a one-line expense (campaign, category, description, amount, recoverable?) for small costs such as a ₦15,000 driver payment.

## Rules
- Everything created is flagged `source = whatsapp` and shows that badge in the web app.
- The same server-side validations apply (talent status, campaign status gates, positive amounts, email validity, immutability after payout). If a rule blocks the action, the bot explains it in one line and returns to the menu.
- A flow creates nothing until the final Save, Save draft or Send. Cancelling or timing out creates nothing.
- Retrying a final action (the admin taps twice, or Meta resends) creates one record: the final step carries an idempotency key.
- Bank details are never typed or shown in chat. They appear only inside the attached PDF.
- Voiding, editing, payments, payouts and advances are not available in WhatsApp in v1.

## Data
- `whatsapp_session`: sender, flow, step, answers (json), expires_at.
- `whatsapp_message`: wa_message_id (`UNIQUE`), direction, sender, type, timestamp, flow/step, outcome. Message text is kept for 90 days for troubleshooting.
- `source` (web | whatsapp) on invoices and expenses.

## Edge cases
- Meta delivery delay or duplicate: processed once by message id.
- Admin starts a second flow while one is open: the bot asks to continue or discard the first.
- Campaign becomes ineligible mid-flow (cancelled in the web app): the final step re-checks and refuses with the reason.
- Brand email mistyped: caught by the confirm step, and a bounce later shows on the invoice (invoices FRD).
- WhatsApp or Meta outage: the web app is unaffected; sessions simply expire.
- Number changed or lost: the allowlist is edited in Company settings; removing a number ends its sessions immediately.

## Acceptance criteria
- A message from a number not on the allowlist produces no reply and no data.
- A new invoice for a known brand can be saved as a draft in under 90 seconds, in no more than 8 messages from the admin.
- The same Meta webhook delivered twice creates one invoice.
- "Send invoice" never emails the brand without the second confirmation.
- An invoice created by WhatsApp has the next number in sequence, a `WhatsApp` badge, and identical totals to one made on the web.
- A one-line quick expense can be recorded in under 30 seconds.
- A talent-level expense with a recoverable line created from WhatsApp posts the matching advance.
- Cancelling at any step leaves no records.
