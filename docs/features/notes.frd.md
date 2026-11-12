# FRD: Campaign Notes

**PRD refs:** §6.14, FR-51 to FR-55 · **Milestone:** 11

## Purpose
A place on each campaign to write down what was discussed and decided: meeting notes, call summaries, brand feedback, anything worth remembering. Today this lives in chats, phone notes and memory, and is lost when a person is unavailable. Notes keep the campaign's context with the campaign.

This is a simple notes log, not a document editor or a wiki.

## Note
| Field | Notes |
| --- | --- |
| campaign_id | Required. A note belongs to one campaign (and therefore one talent) |
| title | Required, up to 200 characters |
| type | `meeting` / `call` / `general` (a label for filtering) |
| note_date | The date of the meeting or event, defaults to today (Africa/Lagos) |
| attendees | Optional free text (names, companies) |
| body | Text with light formatting (see below), up to 20,000 characters |
| author | The signed-in user, recorded automatically; also `edited_by` on changes |
| pinned | Boolean; pinned notes stay at the top |
| created_at, updated_at | Shown on the note |

- **Formatting:** plain text with a small Markdown subset: headings, bold, italic, bullet and numbered lists, links, and checkbox lines rendered as plain bullets. No images, tables, embeds or attachments (PRD §4: the logo is the only stored file).
- **Editing:** a note can be edited at any time; the note shows "edited <date>". There is no version history in v1.
- **Who can change a note:** members can edit or delete only their own notes; admins can edit or delete any note. Deleting asks for confirmation and removes the note for good, since notes hold no money.
- **Who can read:** anyone with the Campaigns page and the campaign's talent in scope (team FRD).

## Where it appears
1. **Campaign page → Notes tab:** newest first by note date, pinned notes on top. Each row shows title, type, note date, author and the first lines of the body; clicking opens the full note. Filter by type; search within the campaign's notes. **New note** opens an editor with title, type, date, attendees and body.
2. **Campaign Overview tab:** the three latest notes with a link to the Notes tab.
3. **Global search:** matches note titles and bodies and links to the note, alongside talents, campaigns and invoices.
4. **Create task from a note:** a note has a "Create task" action that opens the new-task form (see the tasks FRD) with the campaign set and the note title as the task title. The task stores no link back to the note in v1.

## Visibility and privacy
- Notes are **internal only**. They never appear on invoices, PDFs, emails, WhatsApp messages or any brand-facing output.
- Notes may contain personal or commercial information (rates, disputes, contact details). They are covered by the data-safety rules in PRD §11.2: included in the full-data backup export, and erased with the campaign data when the agency erases a talent's data on request.
- The note body is never sent to a third party by the app. If the WhatsApp or inquiry features use external language models, notes are not included in what they see.

## Campaign and talent status rules
- Notes can be added to a campaign in any status, including completed and cancelled.
- A deactivated talent's campaign notes stay readable and editable; notes carry no money, so the deactivation matrix (PRD §6.1) does not restrict them.
- Deleting a campaign is not offered (campaigns are cancelled, not deleted), so notes are never orphaned.

## Edge cases
- Empty body: allowed if a title exists (a quick reminder).
- Very long note: the list shows an excerpt; the full text opens in the note view.
- Two people editing the same note: last save wins; the note shows its updated time. A warning "This note changed since you opened it" is shown when saving over a newer version.
- Pasted rich text (from email or documents): reduced to the supported formatting; unsupported content is stripped.
- Notes created before team management have no author and show "Unattributed".
- Note dated in the future: allowed (for example an agenda for next week's meeting).

## Out of scope for v1
- Attachments, images, audio recordings and transcription, version history, comments or mentions, and shared note links.
- Notes on talents, invoices or other records (campaign notes only).
- Notifications, reminders, and notes sent from WhatsApp.
- Templates for meeting notes.

## Open questions
- **Talent-level and company-level notes:** whether notes are also needed on a talent profile (for example relationship context across campaigns). Depends on how the operations team uses them.
- **Attachments:** meeting notes often come with decks or contracts; adding files would need the file store decided in PRD §15.
- **Voice or WhatsApp capture:** adding a note by sending a message to the assistant is a natural later step.
