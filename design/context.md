---
schema_version: 1
updated_at: 2026-11-09
---

## Users & Purpose
A single agency admin at a talent-management company, working at a desk, juggling several creators' money at once. They create campaigns, log shoot expenses, write and send invoices to brands, give advances to talent, and pay talent out. They need to answer, at a glance: what is owed to us, what do we owe, and who has an open advance.

## Brand Personality
Calm, trustworthy, precise. Airy like a well-run finance tool, not a trading floor. Numbers are the hero; chrome stays quiet.

## Aesthetic Direction
**References**

| Reference | Take | Leave |
| --- | --- | --- |
| Markly dashboard | Soft off-white canvas, white rounded cards, KPI cards with sparklines, calm area chart with tooltip, segmented range control, roomy tables, pale active nav pill | AI-agent / outreach framing; reach metrics (views, clicks, CPM); green primary (we use blue) |
| Ofacc+ invoice editor | Split-pane form with live paper preview, labeled inputs, small-caps section labels, quick-pick due-date chips, sticky Cancel / primary footer, top search bar, blue primary | Tax / VAT fields; multi-language and currency pickers (Naira only in v1); dark toggle (light only in v1) |

**Visual movement:** soft-UI finance, airy and card-based, with document-style editors.

**Domain concepts:** ledger, advance, payout, retainer, rate card, run-of-show, call sheet, settlement.

**Color world:** naira-note green, ink on paper, ledger cream, brass/gold for money, warm terracotta for "owed".

**Signature element: the settlement strip.** A horizontal segmented bar showing how campaign income splits into commission, recoverable expenses, advance deduction and talent share. Appears on the campaign overview, talent campaign cards, the payout dialog, the dashboard, and the payout summary on documents.

**Defaults rejected**
- Blue-for-everything SaaS look: blue is used for action and focus only; money states have their own palette.
- Inter as the UI font: Figtree instead.
- Red/green on every number: green = received/paid, terracotta = owed/overdue, everything else neutral.

## Concrete Constraints
- Light mode only for v1.
- Primary is blue; green appears only as the "income / paid" semantic color.
- Money uses tabular numerals and always shows the ₦ symbol.
- Document numbers (INV-0001, EXP-0001) use the mono font; nothing else does.
- Cards: 16px radius, hairline border, one soft shadow. No stacked heavy shadows, no gradients.
- Tables: 56px rows, no zebra striping, hairline dividers.
- Document editors (invoice, expense, payout) use the split-pane pattern: form left, live paper preview right, sticky footer.
- Money-moving actions (Send, Record payment, Pay out) always have a text label.

## Design Principles
1. **Numbers first.** Figures are the largest, highest-contrast element on any screen.
2. **One accent for action.** Blue means "you can act here"; semantic colors mean money state.
3. **Show the split.** Wherever money is divided, show the settlement strip, not just a number.
4. **Preview what you send.** Anything that leaves the app (invoices, expenses) is edited beside its live preview.
5. **Calm density.** Airy by default; never more than one idea per card.

## Design Tokens

### Color
| Token | Light | Dark | Role |
|-------|-------|------|------|
| `--canvas` | oklch(0.972 0.004 255) | n/a (v1 light only) | App background |
| `--surface` | oklch(1 0 0) | n/a | Cards, panels, table |
| `--surface-sunken` | oklch(0.985 0.003 255) | n/a | Table header, form groups |
| `--line` | oklch(0.92 0.006 255) | n/a | Hairline borders, dividers |
| `--ink` | oklch(0.21 0.02 260) | n/a | Primary text |
| `--ink-muted` | oklch(0.52 0.015 260) | n/a | Secondary text, labels |
| `--primary` | oklch(0.55 0.21 262) | n/a | Buttons, links, active nav text, chart main line, focus ring |
| `--primary-soft` | oklch(0.955 0.03 262) | n/a | Active nav pill, chips, selected row |
| `--income` | oklch(0.62 0.15 160) | n/a | Money received, paid status |
| `--owed` | oklch(0.60 0.16 40) | n/a | Outstanding advance, overdue |
| `--recoverable` | oklch(0.78 0.14 80) | n/a | Recoverable expenses |

**Settlement strip segments:** commission `--ink`, recoverable expenses `--recoverable`, advance deduction `--owed`, talent share `--primary`.

**Rationale:** blue follows the Ofacc+ reference and signals action. The money palette (green, terracotta, brass) comes from the domain's color world and keeps financial states readable without defaulting to red/green everywhere.

### Typography
- **Primary:** Figtree. Friendly but precise, and not a generic default. Verify tabular-numeral (`tnum`) support at build; fall back to a font that has it if missing.
- **Mono:** Geist Mono, for document numbers only.
- **Scale:** xs 12, sm 14, base 15, lg 18, xl 22, 2xl 28, 3xl 36 (KPI figures).
- **Weights:** 400 body, 500 labels and table headers, 600 card titles and buttons, 700 KPI figures.
- **Section labels:** 11px, small caps, 0.08em tracking, `--ink-muted` (e.g. "MY DETAILS", "DATES").

### Spacing
- **Base:** 4px
- **Scale:** micro 4, sm 8, md 16, lg 32, xl 48, 2xl 64
- **Specifics:** card padding 24, card gap 16, table row 56, form field height 56 (floating label).

### Depth
- **Strategy:** surface tints plus hairline border plus one very soft shadow.
- **Why:** matches the Markly card-on-canvas look and keeps the UI calm.
- **Levels:** card `0 1px 2px oklch(0.21 0.02 260 / 0.04)`; invoice preview paper `0 8px 24px oklch(0.21 0.02 260 / 0.08)`; popovers and tooltips `0 8px 24px oklch(0.21 0.02 260 / 0.10)`.

### Border Radius
- **Scale:** sm 8 (chips, small buttons), md 12 (inputs, buttons), lg 16 (cards, preview paper), pill (segmented control, status badges)
- **Character:** soft and friendly, but not bubbly.

### Motion
- **Micro:** 150ms, `cubic-bezier(0.2, 0, 0, 1)` (hover, press)
- **Transition:** 220ms, `cubic-bezier(0.2, 0, 0, 1)` (tabs, panels, dialogs)
- **Entrance:** chart lines draw once on load; nothing bounces or loops.

### Layout patterns
- **Shell:** left sidebar (workspace and talent switcher on top, "Main menu" and "Other" groups, Support and Settings pinned at the bottom, active item is a `--primary-soft` pill) plus a top bar (search, notifications, avatar).
- **Dashboard:** four KPI cards with sparklines, an area chart with a Years / Months / Weeks segmented control, and a recent-campaigns table with search.
- **Document editor:** breadcrumb and title, form sections with small-caps labels on the left, live paper preview on the right, sticky footer with Cancel and the primary action. Invoices get quick due-date chips (3 / 7 / 14 / 30 days).

### Anti-patterns
- No reach metrics (views, clicks, CPM) in tables.
- No red/green on every number.
- No decorative gradients or AI-agent chrome from the Markly sidebar.
- No icon-only buttons for money actions.
- No dark-mode toggle in v1.
