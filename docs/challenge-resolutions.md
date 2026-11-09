# Challenge review resolutions

Source: `/mine.challenge` run on 2026-11-09 over `prd.md` and `features/*.frd.md` (28 findings). Full critic reports and findings are in `/tmp/claude-mine-challenge-8yqb62/` (temporary).

## Applied to the PRD and FRDs
| # | Finding | Resolution | Where |
| --- | --- | --- | --- |
| F1 | Payout formula and prior payouts | A: available = max(0, earned − paid gross); partial payouts; over-paid state; worked example | PRD §6.7, §7.1, §7.7; payouts FRD |
| F2 | Void/edit/delete semantics | A: immutable money records, reversal rows | PRD §6 conventions, §7.4; invoices, expenses FRDs |
| F3 | Advance model, repayment, write-off | A: per-talent signed ledger, FIFO at read time | PRD §6.4; advances FRD |
| F5 | Talent-level recoverable expenses | B: auto-post an advance entry | PRD §6.3; expenses, advances FRDs |
| F6 | Commission rounding and storage | A: bigint, basis points, stored per payment, frozen after first payment | PRD §6, §7.1; campaigns, invoices FRDs |
| F7 | Overview definitions | A: cash basis, provable strip, Owed to talents | company-overview FRD |
| F8 | Deactivation | A: status is the single source, server-side matrix | PRD FR-1; talents FRD |
| F9 | Snapshots | B: invoice issuer fields frozen at send, stored PDF; payout snapshot; hashed logo | PRD §6.5, §6.7, §6.8; settings, invoices, payouts FRDs |
| F10 | Invoice send atomicity | fix: `sending` state, idempotency key, message id before `sent`, bounce webhook, send log | PRD FR-16; invoices FRD |
| F11 | Numbering | C: no duplicates, all numbers accounted, counter row, UNIQUE, void not delete | PRD FR-14; invoices FRD |
| F12 | Invoice status | fix: store draft/sending/sent/void; derive paid, part-paid, overdue; Africa/Lagos | PRD §6.5, §7.2; invoices FRD |
| F13 | Campaign/expense/payout integrity | A: immutable talent_id, composite FKs, RESTRICT, status gates | PRD §6 conventions, §7.6; campaigns FRD |
| F14 | Security baseline | fix: new PRD §11.1; auth FRD expanded | PRD §11.1; auth FRD |
| F15 | Correcting sent invoices | A: edit and resend corrected copy; lock lines once paid; global New invoice | PRD FR-19a; invoices FRD |
| F16 | Collection workflow | fix: Record payment and Send reminder row actions | PRD FR-19b; invoices, overview FRDs |
| F17 | Commission model | B: percent only, locked after first payment, flat deals unsupported | PRD §4 |
| F18 | One talent per campaign | A: kept, documented limit | PRD §4; campaigns FRD |
| F19 | Expense entry | A: per-line recoverable, inline quick-add | PRD §6.3, FR-8; expenses FRD |
| F20 | Payout form | A: prefilled context and deduction, Ready to pay list | PRD FR-20; payouts FRD |
| F21 | Audit and backup | B: reversal rows, created_by/at, backup export and recovery requirement | PRD §6, §11.2 |
| F22 | Currency claim | B: currency on the campaign only, children inherit, §9 reworded | PRD §9 |
| F23 | Inconsistencies | fix: worked example, data size, definitions, commission precedence, tab list, minimum history | PRD §7.3, §7.5, §7.7, §7.8; overview FRD |

## Filed, not applied
- **F4: Transactions, locking and idempotency.** The driver problem is specific to the database provider (Drizzle's neon-http driver cannot run interactive transactions), and the provider is not decided. Decision: not solved now. Requirements to carry into the technical design once the provider is chosen:
  - Payout, deduction, payment and advance writes must each be atomic and use a row lock (talent row for payouts and ledger writes, invoice row for payments).
  - Server recomputes available and advance balance inside the transaction and rejects over-cap amounts.
  - Client idempotency token with a unique index on payouts; stored provider idempotency key on invoice sends.
  - Pick a driver that supports interactive transactions (for Neon: the WebSocket Pool driver, or node-postgres).
  - PRD §11 now names the provider as undecided. No GitHub issue was created because this folder is not a git repository.

## Deferred (critics disagreed; recorded in PRD §15)
- F24 Overview scope: 12 sections versus a lean home.
- F25 Ledger scope: advance-only ledger versus a unified ledger.
- F26 Withholding tax and short payment.
- F27 Brand as free text versus a brands table.
- F28 Pre-payment of talents before the brand pays.

## Not updated
- The HTML mockup (`~/.claude/diagrams/talent-manager-mockup.html`) still shows the earlier design (for example the 8-KPI strip, the notifications bell, recoverable per expense).
