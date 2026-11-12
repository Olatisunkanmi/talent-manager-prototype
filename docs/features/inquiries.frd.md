# FRD: Inbox Inquiry Tracking

**PRD refs:** §6.12, FR-39 to FR-44 · **Milestone:** 9 · **Status:** draft, inbox access model open (see Open questions)

## Purpose
Brands and companies reach out to talent by email. Today nobody can say how many companies contacted a given talent, when, or what they wanted. This feature watches each talent's inbox through an automation tool, picks out the **inquiries** (a company approaching the talent for work), and logs them in Talent Manager so the agency can count and review them per talent and over time.

Scope is deliberately narrow: **log inquiries only**. Other mail (invoices, newsletters, personal, spam) is ignored and not stored.

## Two parts, hosted separately
| Part | Where it lives | Responsibility |
| --- | --- | --- |
| **Intake service** | Separate deployment: self-hosted n8n | Reads mail, classifies it, extracts fields, writes a short summary, sends the result to the app |
| **Talent Manager** | The main app | Receives, validates and stores inquiry records; shows them; never touches any mailbox |

The two are connected only by one authenticated HTTP call (below). The main app works normally if the intake service is down; inquiries just stop arriving, and the app says so (see Health).

## Inbox access (open, the app does not depend on it)
How each agency sets up inboxes per talent is not decided and will vary. The intake service may read a talent's own mailbox with their consent, a dedicated agency address per talent, or a forwarded copy in a central mailbox. The app is agnostic: it only needs each inquiry to arrive **tagged with the talent it is for**.

So the app stores a **mailbox registration** per talent, not credentials:
- talent_id, address (the address that receives the talent's business mail), `access_mode` (free label: talent-connected / agency-address / forwarded / other), status (active / paused), last_heartbeat_at.
- Credentials and tokens live only in the intake service, never in Talent Manager.
- Several mailboxes per talent are allowed (a talent may have a personal and a management address).

## Intake contract (service to app)
`POST /api/intake/inquiries`, one call per inquiry.

| Field | Notes |
| --- | --- |
| mailbox address or talent_id | Resolved to a registered mailbox; unknown mailbox is rejected |
| message_id, thread_id | The mail provider's ids. `message_id` is the idempotency key |
| received_at | When the mail arrived (UTC) |
| sender_name, sender_email | From the message |
| company | Extracted by the classifier; falls back to the sender's domain; "Unknown" for free-mail senders with no company in the text |
| subject | As received |
| summary | One to three sentences written by the intake service: what is being asked |
| confidence | 0 to 1 from the classifier |
| message_link | Optional deep link that opens the mail in its mailbox (only useful to someone with access) |

Rules:
- **Authentication:** shared secret plus HMAC signature over the body with a timestamp; requests older than 5 minutes are rejected. Secret rotatable in settings.
- **Idempotent:** the same `message_id` returns the existing record, creating nothing.
- **One inquiry per thread:** a later message in an existing `thread_id` is attached as a follow-up count on the first inquiry, not logged as a new inquiry, so replies do not inflate "companies that reached out".
- **Heartbeat:** the service calls `POST /api/intake/heartbeat` per mailbox on each successful poll, even when no mail is found.
- The app validates and stores; it never calls back into the mailbox.

## What is stored (minimal)
Per inquiry: talent, mailbox, received date, sender name and email, company, subject, summary, confidence, thread id, follow-up count, status, `message_link`. **Not stored:** the full message body, attachments, other recipients. The summary is generated inside the intake service so the body never reaches the app.

## Inquiry record
| Field | Notes |
| --- | --- |
| talent_id, mailbox_id | |
| message_id (unique), thread_id | |
| received_at | Used for all date filtering |
| sender_name, sender_email, company, subject, summary | |
| confidence | Shown when below 0.7 as "Low confidence" |
| status | `new` / `reviewed` / `dismissed` |
| source | `intake` or `manual` |
| followups | Count of later messages in the thread |

- **Inquiry records are never hard-deleted.** A false positive is marked `dismissed` and excluded from all counts.
- `new` becomes `reviewed` when the admin opens it or marks it. Status is for triage only; there is no pipeline, no conversion tracking and no link to campaigns in v1.
- **Manual add:** the admin can log an inquiry the classifier missed (talent, date, company, contact, note), with `source = manual`.

## Counting rules
- **Inquiries** = non-dismissed records with `received_at` in range.
- **Companies that reached out** = distinct normalised company among those inquiries. Normalisation: lower-case, trim, and use the sender's domain when the company is blank or the domain is not a free-mail provider. Two spellings of one company can still count twice; the admin can set a record's company to merge them.
- Per talent, over the Africa/Lagos date range chosen. A company contacting two talents counts once for each talent and once overall.
- Dates use Africa/Lagos for display and day boundaries.

## Where it appears
1. **Talent profile → Inquiries tab:** table of inquiries (received date, company, sender, subject, summary, status, confidence), filters for status and date, a "Mark reviewed" and "Dismiss" action, and "Add inquiry" for a manual entry. Header counts: inquiries and companies in the range.
2. **Company overview → new section "Inquiries":** total inquiries, distinct companies, inquiries per talent, a trend by week or month, and the latest ten. Respects the date range, talent filter and compare-to option. Rows link to the inquiry or talent.
3. **Talents list:** one new column, "Inquiries (30d)".
4. **Settings → Inquiry intake:** mailbox registrations, per-mailbox health, the intake secret and its rotation.

## Health and failure handling
- Each mailbox shows **last checked**. If no heartbeat for more than 2 hours, the mailbox shows "Not checked since ..." with a warning on the Inquiries tab and the settings page, so silence is never mistaken for "no inquiries".
- A mailbox whose access was revoked or expired is reported by the intake service as `paused` with a reason; the talent shows a "Reconnect" prompt for the admin to fix in the intake tool.
- Rejected requests (bad signature, unknown mailbox, malformed payload) are logged with a reason and counted in settings. The app never creates a partial record.
- Late arrival: backfilled mail older than the mailbox's registration date is accepted if the service sends it; `received_at`, not arrival time, decides which period it counts in.

## Privacy and consent
- Reading a talent's mail needs the talent's knowledge and agreement; the agency records this when registering the mailbox (a "consent confirmed" checkbox with a date).
- Minimal storage (above) is the main control. Mail content is processed by the intake service and, if a hosted language model does the classifying, that provider also sees the message. This is called out as an open question below.
- Nigeria's data protection law (NDPA) applies to the sender details stored; the data-safety rules in PRD §11.2 apply, and a mailbox's inquiries can be exported or erased on request.

## Edge cases
- Newsletter or automated mail misread as an inquiry: dismissed by the admin; no learning loop in v1.
- Inquiry from a company the agency already works with: still logged; there is no brand directory to match against in v1.
- One email addressed to two registered talents: logged once per talent (the contract carries a talent per call).
- Out-of-office or bounce replies are ignored by the intake service and never reach the app.
- Reply sent by the talent first (they started the conversation): not an inquiry and not logged.
- Deactivated talent: new inquiries are still accepted and shown; the mailbox is not paused automatically.
- Clock skew: signature timestamps allow a 5-minute window.

## Open questions
- **Inbox setup model:** how agencies set up inboxes per talent (talent's own, an agency address, or forwarding). To be learned from how the agency works; the design above does not depend on it. Decide before building the intake workflows.
- **Classifier and data exposure:** whether classification may use a hosted language model (the message text leaves your control) or must run on a model you host or on rules only.
- **Missed inquiries:** how much a miss costs, which sets how much review effort the classifier needs (a daily digest of "unsure" messages is one option, not in v1).
- **Follow-up pipeline:** replied / converted to campaign / declined, and linking an inquiry to a campaign. Deliberately left out of v1.
- **Brand directory:** matching inquiries to known brands needs the brands table already deferred in the PRD.
- **Intake hosting:** where the separate n8n instance runs and who maintains it.
