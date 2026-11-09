# FRD: Company Overview

**PRD refs:** FR-23, FR-24, §7 (expanded) · **Milestone:** 6 · **Mockup:** Overview (default landing page)
**Replaces:** the earlier "Dashboard" FRD.

## Purpose
The company's home page and single source of truth for how the agency is doing. It shows all revenue, how it is growing, where it comes from and where it goes, and it can be sliced by date range, talent and campaign.

## Filters (apply to every figure on the page)
| Filter | Behaviour |
| --- | --- |
| **Date range** | Presets: 7D, 30D, 90D, This month, YTD, 12M, All time, plus Custom (start and end date). Default: YTD. |
| **Talents** | Multi-select. Default: all talents. Includes a toggle "Include deactivated talents" (on by default so history is complete). |
| **Campaigns** | Multi-select. Options are narrowed to the selected talents. Default: all campaigns. |
| **Compare to** | Previous period (same length, immediately before), Previous year (same dates a year earlier), or Off. Default: previous period. |
| **Granularity** | Day, Week, Month, Quarter. Chosen automatically from the range, overridable on the trend chart. |

- Filters live in a sticky bar at the top and are stored in the URL, so a filtered view can be bookmarked or shared.
- A summary line under the bar always states what is being shown (for example "6 talents, 14 campaigns, 01 Jan to 09 Nov 2026, compared with 01 Jan to 09 Nov 2025").
- "Clear all" resets to the defaults. Selected talents and campaigns appear as removable chips.
- Changing filters updates all sections together; there is no per-section filter.

## Metric definitions
Money rules follow PRD §7. All amounts are in naira (stored in kobo). **Everything is on a cash basis**: nothing counts until a payment is recorded. Dates are calendar dates in Africa/Lagos.

| Metric | Definition | Dated by |
| --- | --- | --- |
| **Revenue received** | Sum of payments (net of reversals) on invoices | Payment date |
| **Invoiced** | Sum of totals of **sent** invoices (drafts and void excluded) | Sent date |
| **Agency income** | Sum of the commission stored on each payment | Payment date |
| **Expenses** | Non-void expenses, split into recoverable and absorbed lines | Expense date |
| **Agency profit** | Agency income − absorbed expenses (absorbed expenses come out of commission) | Payment / expense date |
| **Talent payouts** | Net cash paid to talents (after advance deductions) | Payout date |
| **Advance deductions recovered** | Deductions applied at payout | Payout date |
| **Outstanding receivables** | Balances of sent invoices with balance > 0 (not yet due and overdue) | As of today |
| **Owed to talents** | Sum of *available* (PRD §7.1) over campaigns | As of today |
| **Open advances** | Advance ledger balance (advances − deductions − repayments − write-offs) | As of today |

Rules:
- Void invoices, voided expenses and voided payouts are excluded everywhere; reversal rows net against their originals.
- "Outstanding receivables", "Owed to talents" and "Open advances" are balances **as of today**, so they ignore the date range but respect the talent and campaign filters. They are labelled "as of today" on the page.
- Campaign-level figures roll up through the campaign's talent. Talent-level expenses and advances (no campaign) are included when only talents are filtered and **excluded** when specific campaigns are selected.
- Growth % = (current − comparison) ÷ comparison. When the comparison value is zero, show "new" instead of a percentage.
- **Minimum history:** growth and comparison need at least one full comparison period with data, otherwise show "not enough history". The run-rate needs at least three full months of data, otherwise it is hidden with the same message.
- **Deactivated talents** are always included in every total. The "Include deactivated talents" toggle only controls whether they are listed in the talent breakdown (section 6) and the Talents filter list.

## Sections
1. **Filter bar and summary line** (above).
2. **Headline KPI cards (9):** Revenue received, Invoiced, Agency income, Agency profit, Outstanding receivables, Owed to talents, Talent payouts, Expenses, Open advances. Each card shows the value, the change versus the comparison period (percentage and arrow), and a sparkline over the selected range.
3. **Revenue trend chart:** lines for Invoiced, Received and Agency profit across the range, with the comparison period drawn as a dashed line. Hover shows a tooltip with all series and the change versus comparison. Granularity switcher on the chart.
4. **Growth panel:** period-on-period growth, month-on-month growth (with a small bar chart of monthly growth %), year-on-year growth, average monthly revenue, best month, and a run-rate projection (average of the last three full months × 12).
5. **Where the money goes (money flow):** a settlement strip for the whole selection made only of provable identities: income received = agency commission + recoverable expenses + advance deductions + net paid to talent + owed to talents (not yet paid). Absorbed expenses are **not** a slice of income; they appear beside the strip as a deduction from commission, leading to Agency profit. Each amount shows its share.
6. **Revenue by talent:** ranked bars with revenue, share of total, growth vs comparison, and a status badge for deactivated talents (listed only when the "Include deactivated talents" toggle is on; their revenue is in the totals either way). Clicking a row sets the talent filter.
7. **Revenue by brand:** ranked bars with revenue, share and growth. Brand is the typed brand name on campaigns, normalised case-insensitively; typos split a brand into two rows until a brand directory exists (open question in PRD §15).
8. **Receivables aging:** open invoice balances grouped as Not yet due, 1 to 30 days late, 31 to 60, and over 60, with counts. Includes a list of the most overdue invoices linking to each invoice, with **Record payment** and **Send reminder** actions on each row.
9. **Expenses by category:** bars for bouncer, driver, videographer, food, equipment and other, with the recoverable portion marked.
10. **Advances outstanding:** per-talent open advance balance, the date of the oldest unrecovered advance (first-in-first-out), amounts written off in the range, and a link to the talent's advance ledger.
11. **Campaigns table:** one row per campaign in the selection with talent, brand, status, invoiced, received, expenses, agency profit margin %, views, engagement rate, and a mini settlement strip. Sortable by any column, searchable, and active campaigns can be filtered to the top. Clicking a row opens the campaign.
12. **Active campaigns:** a compact list of ongoing campaigns across all selected talents so work in progress is visible agency-wide.
13. **Content performance:** from tracked posts (insights FRD): total views, engagements, engagement rate, posts tracked, views by platform (per-platform first, mixed total labelled), top 5 posts, and campaigns ranked by views and by cost per 1,000 views. Post date follows the date range; talent and campaign filters apply; compare-to applies where both periods have snapshots. Every figure shows its as-of time and a manual marker where figures were typed. Posts with no snapshot are listed as "no data yet", not zero.

## Behaviour and edge cases
- **Empty selection** (no data for the filters): each section shows a short empty state with a "Clear filters" action; the page never shows zeros that look like real data without the empty-state message.
- **Single-point ranges** (for example 7D with Month granularity): granularity falls back automatically.
- **Previous-year compare with insufficient history:** comparison shows "no data" rather than zero.
- **Large data:** aggregation is done in the database (grouped SQL queries), not in the browser. Each section loads independently with skeletons so the page is usable while heavy sections load.
- **Consistency:** the numbers on the Overview must equal the sum of the per-talent and per-campaign pages for the same filters.
- **Navigation:** every row and chart element that represents a talent, campaign or invoice links to its page.

## Out of scope for v1
Exports (CSV or PDF) of the overview, saved filter presets, forecasting beyond the simple run-rate, scheduled email reports, multi-currency totals.

## Acceptance criteria
- Changing the date range, talents or campaigns updates every section, the URL and the summary line.
- Revenue received for a filter equals the sum of payments on the matching campaigns' invoices in that range.
- Selecting a single campaign shows only that campaign's invoices, expenses and payouts, and excludes talent-level expenses.
- Growth percentages match a hand calculation for a worked example.
- Receivables aging buckets sum to the Outstanding receivables card.
- Agency income for any range equals the sum of the stored commission on the payments in that range, and equals the sum of the campaign pages' commission for the same filters.
- Income received = commission + recoverable expenses + advance deductions + net paid + owed to talents, for any filter.
- The worked example in PRD §7.7 produces: received ₦3,000,000, agency income ₦600,000, agency profit ₦500,000, payouts ₦1,250,000, deductions ₦1,000,000, owed ₦0.
- The page loads its first meaningful content (filters and KPI cards) within 2 seconds on the expected data size (PRD §7.8: 50 talents, 500 campaigns, 5,000 invoices, 20,000 payments).
