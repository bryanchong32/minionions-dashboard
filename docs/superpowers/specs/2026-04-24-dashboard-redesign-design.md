# Boss View Dashboard Redesign — Design Spec

## Context

The Minionions Marketing Boss View dashboard (mom.solworks.io) was built as a flat reporting page with 11 charts and KPI cards. Bryan wants to evolve it into a structured 3-tab operational dashboard that separates lifetime overview, monthly operations, and ads management. This redesign also introduces an ad attribution dropdown on the CRM order form and scopes the CAPI integration for when WhatsApp Cloud API is available.

## Scope

### Building now
- Dashboard restructure into 3 tabs
- Tab 1: Lifetime Stats
- Tab 2: Monthly Operations (sales goals, new/repeat funnels, follow-up pipeline)
- Tab 3: Ads Control Panel (placeholder only — needs its own data model)
- Ad catalog sync + attribution dropdown on CRM order form
- MoM delta indicators on all key metrics

### Deferred (open items)
- WhatsApp Cloud API migration
- CTWA attribution (`ctwa_clid` webhook capture)
- Meta Conversions API (CAPI) integration
- Tab 3 full build (function scoring, test tracking, graduation pipeline)

---

## Tab Structure

Three tabs at the top of the dashboard. Tab 2 (Monthly) is the default landing tab.

| Tab | Purpose | Data freshness |
|-----|---------|----------------|
| Lifetime | Big picture — how is the business doing overall | Real-time (DB queries) |
| Monthly | This month's operations — sales, funnels, pipeline | Real-time (DB queries) |
| Ads | Marketing control panel | Placeholder for now |

---

## Tab 1: Lifetime Stats

### Financial Overview (top)
7 KPI cards in two rows, all with MoM delta indicators:

**Row 1:** Total Sales PV | Total Orders | Total Customers | Avg Order Size
**Row 2:** Total Commission (50% of PV) | Total Ad Spend | Net Profit

**MoM delta source:** Dashboard calls `/monthly-trend?months=2` and computes delta from the last two months' data. No additional endpoint needed — deltas are derived client-side from existing trend data. Lifetime totals themselves don't have deltas (they're cumulative); the deltas show "change contributed this month" (e.g., "+42 orders this month").

### Sales Trends
- Monthly Sales Trend — rolling 12-month bar chart (existing)
- Monthly Revenue and Cost — grouped bar, commission vs ad spend (existing)
- Monthly Net Profit — bar with red/green coloring (existing)

### Customer Composition
- New vs Repeat doughnut — lifetime PV split (existing)
- Region breakdown — HK / MO / Other with counts (new, from `customers.region`)
- Membership status — 4 KPI cells: Registered count (`membership_status = 'registered'`), Not Registered count, Registration Rate (registered / total * 100), Verified count (`membership_verified_status = 'active'`)
- PV and Gross Profit — grouped bar (existing)

### Data sources
All data from existing `orders`, `customers`, `meta_daily_metrics` tables. Region and membership are new queries but straightforward `COUNT` with `GROUP BY`.

---

## Tab 2: Monthly Operations

Vertical scroll layout. Month selector at top (defaults to current month). All sections visible on one scrollable page.

### Section 1: Sales Overview

**5 KPI cards:** Monthly Target (editable) | Current PV | Progress % | PV/Day Needed | Run Rate

**Run Rate formula:** `(total_pv_so_far / days_elapsed) * days_in_month` — projects month-end PV at current pace.

All with MoM delta indicators (derived client-side: call `/kpi` for current month and `/kpi?month=prev` for previous). Progress and PV/Day Needed color-coded (green/orange/red based on on-track status).

**Charts:**
- Monthly Goal — vertical progress bar (47% filled)
- Weekly Breakdown — W1-W5 horizontal bars normalized against weekly target (variable week count, matches existing `getWeeklyTarget()` logic)
- Daily Sales Trend — full-width stacked bar (delivered / cancelled / pending by day)

### Section 2: New Sales + Repeat Sales (side by side)

Two subsection cards displayed side by side.

**New Sales (green accent):**
- Badge: "30 orders" count
- KPIs: Reach* | Conversions | Conv. Rate (estimated*) | Basket Size (avg PV)
- Chart: Daily new orders (bars) vs daily reach (line), dual Y-axis
- *Note: "Reach" is from `meta_daily_metrics.reach` which actually stores `messaging_first_reply` (new messaging contacts), not true ad reach. This is a proxy for enquiries. Label as "Reach" with tooltip: "New messaging contacts from Meta ads. Proxy for enquiries until CTWA attribution is built." Conv. Rate = new orders / reach — directional only, not a true funnel conversion rate.

**Repeat Sales (orange accent):**
- Badge: "12 orders" count
- KPIs: Expiring Customers | Conversions | Conv. Rate | Basket Size (avg PV)
- Chart: Daily repeat orders (bar chart)
- "Expiring" = customers with `order_items.end_date` within 30 days of current date

### Section 3: Ad Spend Summary

Single muted KPI row (gray accent): Ad Spend | Commission | Net Profit | Cost per Order

"Full detail in Ads tab" label is a clickable link that switches to Tab 3.

All with MoM delta indicators.

### Divider

Visual separator between sales metrics and pipeline section.

### Section 4: Follow-up Pipeline

Three independent tracks with directional arrow flow and workload signals.

#### Track 1: Onboarding + Membership (side by side)

**Onboarding pipeline (3:2 width ratio):**
Dosage Guide → Compliance Check → Month Review → Graduated

Each stage shows:
- Customer count (big number) — how many customers are currently at this stage
- Completion % (top right, color-coded)
- Progress bar (filled = actioned/sent, empty = pending)
- Legend: "X sent / Y pending"
- SVG arrows between stages

Stage derivation logic: A customer's "current stage" = the earliest incomplete onboarding task for their most recent **delivered** order (`delivery_confirmed = true`). If all 3 are complete, they're "Graduated." Orders not yet delivered have no onboarding tasks generated.

Danger state: Stage border + background turns red when completion < 30%. Orange when 30-50%. Clean when > 50%.

**Membership pipeline:**
Ask Pending → Registered

"Ask Pending" counts customers with any pending `MEMBERSHIP_ASK` or `MEMBERSHIP_DEADLINE` task (both task types represent the membership conversion attempt). "Registered" = customers where `membership_status = 'registered'`. Same visual treatment. Shows conversion rate.

#### Track 2: Reorder Pipeline

Reorder Due → Converted → Lapsed

This is the real conversion funnel:
- **Reorder Due (409):** Customers with pending REORDER tasks. Progress bar shows % actioned. Product breakdown shown as chips (SPP 95, 補骨 89, etc.)
- **Converted (12):** Repeat orders placed this month. Shows conversion rate.
- **Lapsed (196):** 90+ days since last `delivery_confirmed_date`, 2+ lifetime orders. Shows win-back rate. Lapsed customers are **excluded** from the Reorder Due count to avoid double-counting — if a customer has lapsed, their old pending REORDER tasks are irrelevant.

#### Track 3: Rotation Track (per product)

One row per product group, sorted by customer count descending. Top 3 shown by default, "Show X more" toggle for the rest.

Each product row: Knowledge → Testimonial → Tips → Checkin → Done

Same visual treatment as onboarding — customer count at stage, completion %, progress bar, arrows, danger states.

Stage derivation logic: For each customer-product pair, the "current stage" = the earliest pending ROTATE task for that product. If all 4 are complete/skipped, they're "Done."

**Products:** 補骨 (MVO+LVO), 孖寶 (TML+IMG), SPP, BLZ, BGS, HMG

---

## Tab 3: Ads Control Panel (Placeholder)

Shows a "Coming soon" message with a brief description of what it will contain:
- Campaign structure overview
- Function scoring (thumb-stopper, educator, trust-builder, convincer, closer)
- Test tracking and graduation pipeline
- Creative brief queue

Requires a separate design session to define the ad evaluation data model before implementation.

---

## Ad Attribution Dropdown (CRM Order Form)

### New table: `meta_ad_catalog`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | PK |
| tenant_id | UUID | FK → tenants(id), NOT NULL |
| meta_ad_id | VARCHAR | Meta's ad ID |
| ad_name | VARCHAR | Ad name from Meta |
| campaign_id | VARCHAR | Meta campaign ID |
| campaign_name | VARCHAR | Campaign name |
| objective | VARCHAR | MESSAGES / POST_ENGAGEMENT |
| thumbnail_url | TEXT | Creative thumbnail URL |
| status | VARCHAR | active / paused |
| conversion_count | INTEGER | Orders attributed to this ad, DEFAULT 0 |
| last_synced_at | TIMESTAMPTZ | Last sync timestamp |
| created_at | TIMESTAMPTZ | auto |
| updated_at | TIMESTAMPTZ | auto |

**Unique constraint:** (tenant_id, meta_ad_id)

### Nightly sync job

Added to existing daily cron (`POST /api/cron/daily`):
1. For each tenant with Meta API credentials, call `GET /{ad_account_id}/ads?fields=id,name,campaign_id,campaign{name,objective},creative{thumbnail_url},status&limit=500`
2. Filter: active + recently paused (within 30 days)
3. Upsert into `meta_ad_catalog`

### Order form changes

Replace free-text `ad_source` field with a searchable dropdown:

**Groups:**
1. **Meta Ads** — from `meta_ad_catalog`, sorted by `conversion_count` DESC
   - Each option shows: thumbnail + ad name + campaign name
   - Searchable by ad name or campaign name
2. **Other sources** — fixed options: "Organic", "Referral"

**On order save:**
- Store selected `meta_ad_id` in orders table (new column: `meta_ad_id VARCHAR`)
- `conversion_count` is a **derived count** (`SELECT COUNT(*) FROM orders WHERE meta_ad_id = ? AND order_status != 'cancelled'`), not a stored counter. Recalculated on each ad catalog query. This handles order edits, cancellations, and re-attributions automatically.
- Existing free-text `ad_source` field preserved for backward compatibility

### Schema changes

```sql
-- New column on orders table
ALTER TABLE orders ADD COLUMN meta_ad_id VARCHAR;

-- New table
CREATE TABLE meta_ad_catalog ( ... as above ... );
```

---

## CAPI Integration (Deferred — requires WhatsApp Cloud API)

When Cloud API is available, the full flow will be:

1. Customer clicks CTWA ad → WhatsApp webhook fires with `ctwa_clid` + `ad_id`
2. CRM stores `ctwa_clid` keyed to customer phone number (new table or column)
3. Order form auto-prefills ad attribution from stored `ctwa_clid` (dropdown becomes fallback for organic/referral)
4. On order save, fire server-side CAPI `Purchase` event to Meta:
   - `event_name: 'Purchase'`
   - `action_source: 'business_messaging'`
   - `messaging_channel: 'whatsapp'`
   - `user_data: { ctwa_clid, phone (hashed) }`
   - `custom_data: { value (PV), currency: 'HKD' }`
5. 7-day attribution window applies

Not in this build scope.

---

## New API Endpoints

### Dashboard endpoints (on CRM backend, `/api/boss-view/`)

Existing endpoints to keep:
- `GET /kpi` — already returns lifetime + monthly + target
- `GET /monthly-trend` — already returns rolling monthly data
- `GET /daily` — already returns daily breakdown
- `GET /ad-daily` — already returns daily ad spend
- `GET /weekly-target` — already returns weekly PV vs target
- `GET /customer-mix` — already returns new vs repeat split
- `PUT /target` — already sets monthly target

New endpoints needed:

| Endpoint | Returns | Used by |
|----------|---------|---------|
| `GET /lifetime-composition` | Region breakdown, membership stats | Tab 1 |
| `GET /pipeline/onboarding` | Customer counts at each onboarding stage + completion %. Always current state (no month scoping). | Tab 2 Pipeline |
| `GET /pipeline/membership` | Pending ask count, registered count, rate. Always current state. | Tab 2 Pipeline |
| `GET /pipeline/reorder?month=YYYY-MM` | Reorder due (by product), converted count for that month, lapsed count + rates. `month` scopes the "Converted" count only; "Due" and "Lapsed" are always current state. | Tab 2 Pipeline |
| `GET /pipeline/rotation` | Per-product customer counts at each rotation stage + completion %. Always current state. | Tab 2 Pipeline |
| `GET /new-sales-funnel` | Enquiries, conversions, rate, basket size for selected month | Tab 2 New Sales |
| `GET /repeat-sales-funnel` | Expiring customers, conversions, rate, basket size | Tab 2 Repeat Sales |

### Ad catalog endpoints (on CRM backend, `/api/meta/`)

| Endpoint | Returns | Used by |
|----------|---------|---------|
| `GET /ad-catalog` | List of ads sorted by conversion_count DESC | Order form dropdown |

---

## Pipeline Stage Derivation Logic

### Onboarding stages

For each customer's most recent delivered order:
1. Find all onboarding tasks (DOSAGE_GUIDE, COMPLIANCE_CHECK, MONTH_REVIEW)
2. Order by due_date ASC
3. Customer's stage = first task where status is NOT in (completed, skipped)
4. If all 3 are completed/skipped → "Graduated"

### Rotation stages

For each customer-product pair:
1. Find all ROTATE tasks for that target_sku
2. Order: ROTATE_KNOWLEDGE → ROTATE_TESTIMONIAL → ROTATE_TIPS → ROTATE_CHECKIN
3. Customer's stage = first task where status is NOT in (completed, skipped). Statuses that count as "at this stage": `pending`, `sent`, `replied`, `no_response`, `no_response_final`.
4. If all 4 are completed/skipped → "Done"
5. Rotation scopes to **delivered** orders only (same as onboarding)

### Completion % at each stage

For a given stage (e.g., COMPLIANCE_CHECK):
- Count customers at that stage whose task status = 'sent', 'completed', 'replied' → "actioned"
- Count customers at that stage whose task status = 'pending' → "pending"
- Completion % = actioned / (actioned + pending) * 100

### Danger thresholds

| Completion % | State | Visual |
|-------------|-------|--------|
| > 50% | Good | Clean border, green progress bar |
| 30-50% | Warning | Orange border + background |
| < 30% | Danger | Red border + background |

---

## Design System

- **Fonts:** Outfit (headings/labels) + Geist Mono (numbers/data). This replaces the existing Fira Sans + Fira Code — the entire dashboard gets the new font stack.
- **Colors:** Zinc/slate neutral base. Single accent per section: blue (sales/onboarding), green (new sales/good), orange (repeat sales/warning/reorder), red (danger/lapsed), teal (rotation), purple (membership/pipeline)
- **Numbers:** All numeric values use `font-family: var(--mono)` (Geist Mono)
- **Data density:** No card boxing for KPI grids — use 1px gap grids. Cards only where elevation communicates hierarchy (subsections, product rows).
- **Deltas:** Every key metric shows MoM change as arrow + percentage. Green up, red down.
- **Pipeline arrows:** SVG arrow connectors between flow stages showing left-to-right direction.
- **Responsive:** 2-column → 1-column at 768px. Pipeline stages wrap to 2-per-row on mobile.

---

## Verification Plan

1. **Tab 1 KPIs:** Cross-check lifetime totals against Google Sheet Boss View and existing dashboard
2. **Tab 2 Sales:** Compare monthly PV, orders, customers against existing `/api/boss-view/kpi` response
3. **Tab 2 New/Repeat funnels:** Verify enquiry count matches `meta_daily_metrics.reach` sum for the month. Verify new order count matches `orders WHERE is_first_order = true` count.
4. **Tab 2 Pipeline — Onboarding:** Manually verify 5 customers' derived stages against their actual task statuses in the DB
5. **Tab 2 Pipeline — Rotation:** Same — pick 3 customers, verify their per-product rotation stage derivation
6. **Tab 2 Pipeline — Reorder:** Verify reorder count matches `tasks WHERE task_type = 'REORDER' AND status = 'pending'` count
7. **Ad catalog sync:** Trigger sync, verify `meta_ad_catalog` row count matches active + recently paused ads in Ads Manager
8. **Ad dropdown:** Create an order with ad attribution, verify `meta_ad_id` saved and `conversion_count` incremented
9. **Danger states:** Verify stages with <30% completion render red, 30-50% render orange
10. **Tab switch:** Verify "Full detail in Ads tab" link switches to Tab 3
11. **Mobile:** Test at 375px width — KPIs stack to 2-col, pipeline stages wrap, side-by-side sections stack
12. **MoM deltas:** Pick 2 metrics (e.g., monthly PV, ad spend), manually compute current vs previous month, verify delta arrow direction and percentage match
13. **Reach label:** Verify the New Sales "Reach" KPI has tooltip explaining it's a proxy for enquiries
