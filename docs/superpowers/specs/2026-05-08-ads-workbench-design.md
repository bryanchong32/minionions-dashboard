# Ads Workbench — Tab 3 Redesign

**Date:** 2026-05-08
**Project:** minionions-dashboard (Boss View)
**Scope:** Replace current Ads tab (Tab 3) with a workbench reflecting the 3-Tier Testing & Graduation System + seasonal campaign tracking + action execution

## Problem

The current Tab 3 classifies campaigns by objective (Tests / Conversion / Branding). Bryan has implemented a 3-tier graduation system (Hook Test → Proving Ground → Proven) that the dashboard doesn't reflect. He also needs to take action directly from the dashboard (approve/reject recommendations) instead of going through CC agent sessions.

## Design Overview

Tab 3 becomes an **Ads Workbench** — not just a read-only dashboard but a decision surface. Layout top-to-bottom:

1. KPI bar (5+1 split)
2. Funnel summary bar (Testing → Proving Ground → Proven)
3. Seasonal (full width, collapsible KPIs)
4. Branding (PE) + Proven (M) — side by side
5. Testing (M) + Proving Ground (M) — side by side, with ad thumbnails and action panels

## Section 1: KPI Bar

**Layout:** 5 performance KPIs in a row, separated by a gap from 1 standalone budget card.

| KPI | Source | Calculation |
|-----|--------|-------------|
| Total Spend | Meta API `spend` field | Sum across all campaigns in time range |
| Messages | Meta API `actions` → `onsite_conversion.messaging_conversation_started_7d` | Sum |
| Comments | Meta API `actions` → `comment` or `onsite_conversion.post_net_comment` (use existing `extractAction()` logic in backend) | Sum across all campaigns |
| Avg CPR | Derived | Total Spend / (Messages + Comments) |
| ROAS | CRM + Meta API | Attributed PV (from CRM orders with `meta_ad_id`) / Total Spend. Show "(X% attributed)" sub-label based on % of orders with non-null `meta_ad_id` |

**Daily Budget** card (separated): Sum of `daily_budget` from all active campaigns (via the new campaigns metadata call, deduplicated by campaign ID). For ABO campaigns without campaign-level budget, sum ad set `daily_budget` values. Shows cap warning when exceeding RM1,050.

**Time range pills:** 7d | 14d | 30d | MTD — same as current implementation.

## Section 2: Funnel Summary Bar

Horizontal 3-tier bar showing the graduation pipeline at a glance:

| Tier | Label | Metrics shown |
|------|-------|---------------|
| Testing (M) | Orange | Ad count · daily budget · CPR · msgs |
| Proving Ground (M) | Yellow | Ad count · daily budget · day status |
| Proven (M) | Green | Ad count · daily budget · CPR · msgs |

Arrow separators (→) between tiers. Clicking a tier scrolls to its section.

**Classification logic** (by campaign name, checked in order):
1. Contains "hook test" (case-insensitive) → Testing
2. Contains "proving ground" (case-insensitive) → Proving Ground
3. Contains "omnipresence" OR starts with "pe " (case-insensitive) → Branding
4. Campaign has `lifetime_budget` set (not `daily_budget`) OR campaign name matches seasonal naming pattern (contains promo keywords like "雙親", "新年", "38", "聖誕") → Seasonal. The pattern list is maintained as a `SEASONAL_KEYWORDS` array in the frontend — new promos require adding the keyword.
5. Everything else → Proven

This replaces the existing `classifyCampaign()` function. The fallback changes from "conversion" to "Proven."

## Section 3: Seasonal

**Full width.** Shown only when active seasonal campaigns exist.

**Header row:** Campaign name + date range + countdown badge (red when ≤14 days). Click to expand/collapse campaign KPIs. Inline summary stats visible when collapsed (orders · PV · ROAS).

**Campaign KPIs (collapsible):** Orders | Unique Customers | Total PV | Msgs Started | ROAS (with attribution %).

**Ad-level rows:** Each ad shown with:
- Thumbnail (40x40, hover for 200px preview). Source: `thumbnail_url` from `meta_ad_catalog` table (synced daily). For ads not yet in catalog, fetch on-demand via existing `/meta/ad-creative/:ad_id` endpoint. Batch-fetch is not needed — the catalog covers active ads.
- Ad name + ad set name
- Spend | Conv Started | CPR | ROAS

Sub-campaigns grouped by type: (P) ABO section and (V) CBO section with expandable ad lists.

## Section 4: Branding (PE) + Proven (M) — Side by Side

### Branding (PE) — Left column

Single campaign card (PE Omnipresence) with header showing total impressions + engagements.

**Ad-level rows:** Thumbnail + ad name + 5 metrics:
- Spend | Engagements | Comments | Saves | CTR (all)

Expandable — show top 3, "+N more ads ▾" toggle.

### Proven (M) — Right column

All legacy + graduated proven campaigns. Collapsed by default — campaign row shows:
- Campaign name + type (Solo/Multi-ad) + daily budget
- Spend | Conv Started | CPR | ROAS

Click to expand → shows individual ads with thumbnails and same 4 metrics.

**CRM attribution badge:** If a campaign has CRM-attributed orders, show inline badge (e.g. "2 CRM orders · RM2,500 PV"). Particularly important for ads where CPR looks bad but actual sales are strong (mofu9 case).

**CPR color coding:**
- Green: ≤ RM35 (account avg)
- Orange: RM35-50
- Red: > RM50

## Section 5: Testing (M) + Proving Ground (M) — Side by Side

### Testing (M) — Left column

**Campaign header:** Name + date range.

**Day tracker timeline:** Three milestone nodes (Day 3 / Day 7 / Day 14) connected by lines.
- Completed milestones: green fill
- Current position: animated pulse dot between the last completed and next upcoming node
- Pulse position = (current_day - last_milestone) / (next_milestone - last_milestone) as percentage along the line segment
- Day counter: "Day X of 14" centered below the timeline

**Day counter calculation:** Calendar days since campaign `created_time` (from Meta API). Paused days still count.

**Backend change required:** The existing `fetchCampaignInsights` endpoint does not return `created_time`. Add a new companion call in the backend: `GET /{ad_account_id}/campaigns?fields=id,name,created_time,daily_budget,lifetime_budget,status&filtering=[effective_status IN ACTIVE]`. Return alongside campaign insights. This also provides the `daily_budget` needed for the budget card KPI (summed from actual Meta campaign budgets, deduplicated by campaign ID).

**Ad set data:** The existing `fetchAdInsights` endpoint does not return `adset_id` or `adset_name`. Add these to the `fields` list in `metaApiService._fetchAdInsightsCore()` so the frontend can group ads by ad set.

**Milestone snapshot tabs:** Three clickable tabs showing:
- Completed milestones: spend + msgs at that checkpoint (computed from Meta API date ranges — always available retroactively). Notes field is auto-generated from the data: "CBO distributing across N ad sets" for Day 3, "X hooks at RM100+ / 0 msgs" for Day 7.
- Active milestone: live data from current Meta API call
- Upcoming milestones: locked/greyed out with description of expected action

**No persistence needed for milestone data** — all snapshots are derived from Meta API with computed date ranges. Notes are formulaic, not user-entered.

**Milestone data source:** Meta API with `time_range` computed from campaign start date:
- Day 3 snapshot: `since=start_date, until=start_date+3`
- Day 7 snapshot: `since=start_date, until=start_date+7`
- Day 14 snapshot: `since=start_date, until=start_date+14`

**Ad set groups:** Each ad set collapsible, showing:
- Ad set name + ad count
- Individual ad cards with: thumbnail | ad name | spend + msgs | CPR | verdict badge

**Verdict badges** (per ad, based on Ads Strategist thresholds):
- Keep: CPR ≤ RM35 (green)
- Watch: CPR RM35-50 (orange)
- Kill: CPR > RM50 (red)
- Untested: spend < RM100 AND < 7 days (grey)

Untested ads shown at reduced opacity (0.6).

**Minimum data rule:** No verdict until RM100+ spend OR 7+ days delivery.

**Day 14 Recommendation Panel:**
- Locked state: dashed border, "Unlocks May 20" message
- Active state: orange dashed border, shows auto-generated recommendations

**Recommendations** (generated from Ads Strategist rules):

| Condition | Recommendation |
|-----------|---------------|
| CPR ≤ RM45 with 5+ msgs | Graduate → Proving Ground |
| CPR RM45-80 but 1 standout ad CPR < RM40 | Graduate that ad |
| CPR > RM80 with RM100+ spend | Kill (but check CRM first) |
| Spend < RM30 total | Round N+1 candidate (untested) |

**CRM cross-reference:** Before recommending Kill, check `orders.meta_ad_id` for attributed orders. If found, flip recommendation to "Kill? But has X CRM orders (RM Y PV)" with warning.

**Interactive checkboxes:** Each recommendation has a checkbox. User selects which to approve.

**Budget impact table:** Shown when Graduate actions are checked. Shows before/after budget per affected campaign. Net total is zero for pure Graduate batches (budget shifts from Testing → Proving Ground). For mixed Graduate+Kill batches, net total may be negative (budget freed from kills). The table shows the actual impact — no forced zero-net constraint.

**Confirm button:** "Confirm X of Y actions" — executes checked actions via Meta API:
- Graduate: create ad set in Proving Ground campaign, reuse creative, pause in Testing
- Kill: pause ad set / ad
- Round N+1: no action (informational — feeds next test cycle planning)

**Stale recommendation badge:** Recommendations are regenerated on every page load (they're computed from live Meta API data + threshold rules, not persisted). "Staleness" only applies to recommendations that were previously confirmed but partially failed — those are tracked in `meta_action_log`. For normal flow, recommendations are always fresh. The stale badge is removed from the design — it was based on a persistence model we're not using.

**Multiple concurrent tests:** If multiple campaigns match "hook test", stack them vertically with independent day trackers.

### Proving Ground (M) — Right column

**Header:** "Each ad gets RM25/day fixed. 7-day validation cycle."

**Per-ad cards** (larger than Testing cards): Thumbnail (48x48) + ad name + graduated-from label + "Validating" badge.

**Metrics:** Spend | Conv Started | CPR | ROAS | Budget (RM25/d)

**Progress bar:** Visual progress toward 7-day validation. Fill color:
- Early (< 3 days): yellow
- On track (3-7 days): green

**Day 7 verdict panel** (per ad, not batched):
- Unlocks when individual ad reaches 7 days
- CPR ≤ RM45 → Graduate to Proven (move ad to proven CBO campaign)
- CPR RM45-60 → Extend 7 more days
- CPR > RM60 → Kill

**Empty state:** "Slots available — graduates from Testing land here"

## New Backend Endpoint

### `GET /api/boss-view/meta/campaign-roas`

Merges CRM order attribution with Meta campaign spend.

**Query params:** `?since=YYYY-MM-DD&until=YYYY-MM-DD` (matches the time range pills). Filters orders by `delivery_confirmed_date` (the CRM's canonical date field for commission attribution).

**Query:**
```sql
-- Per-ad attribution
SELECT mac.meta_ad_id, mac.ad_name, mac.campaign_id, mac.campaign_name,
       COUNT(o.id) as conversions,
       COALESCE(SUM(o.total_pv), 0) as attributed_pv
FROM meta_ad_catalog mac
LEFT JOIN orders o ON o.meta_ad_id = mac.meta_ad_id
  AND o.tenant_id = mac.tenant_id
  AND o.order_status != 'cancelled'
  AND o.delivery_confirmed_date BETWEEN ? AND ?
WHERE mac.tenant_id = ?
GROUP BY mac.meta_ad_id, mac.ad_name, mac.campaign_id, mac.campaign_name

-- Attribution rate (response-level, not per-campaign)
SELECT
  COUNT(CASE WHEN meta_ad_id IS NOT NULL THEN 1 END) as attributed_orders,
  COUNT(*) as total_orders
FROM orders
WHERE tenant_id = ? AND order_status != 'cancelled'
  AND delivery_confirmed_date BETWEEN ? AND ?
```

**Response shape:**
```json
{
  "period": { "since": "2026-04-24", "until": "2026-05-08" },
  "attribution_rate": 0.72,
  "total_orders": 50,
  "attributed_orders": 36,
  "ads": [
    {
      "ad_id": "456",
      "ad_name": "...",
      "campaign_id": "123",
      "campaign_name": "...",
      "conversions": 2,
      "attributed_pv": 2500
    }
  ]
}
```

Campaign-level aggregation is done client-side by grouping `ads` by `campaign_id`.

ROAS = attributed_pv / spend (spend comes from the existing Meta API insights call, joined client-side).

### Action Execution Endpoints

New endpoints for the workbench to execute confirmed recommendations:

**`POST /api/boss-view/meta/actions`**
```json
{
  "actions": [
    { "type": "pause", "target_type": "adset", "target_id": "120244...", "reason": "Kill: RM120 spent, 0 msgs" },
    { "type": "graduate", "ad_id": "120244...", "creative_id": "269058...", "from_campaign": "Hook Test", "to_campaign": "Proving Ground" }
  ]
}
```

**Graduate action execution (3 sequential Meta API calls):**
1. Find Proving Ground campaign by name match (contains "proving ground"). If none exists, return error — Bryan must create it first via Ads Manager or the agent.
2. Create ad set in the Proving Ground campaign: ABO, RM25/day (2500 cents), optimization_goal=CONVERSATIONS, destination_type=MESSENGER, broad HK targeting (copy from existing Proving Ground ad sets). Ad set created as ACTIVE.
3. Create ad in the new ad set, reusing the `creative_id` from the source ad (preserves social proof). The `creative_id` is fetched via `GET /{ad_id}?fields=creative{id}` before execution.
4. Pause the source **ad** (not ad set) in the Testing campaign: `POST /{ad_id} status=PAUSED`.

**Kill action:** Pauses the target (ad or ad set) via `POST /{target_id} status=PAUSED`.

**Partial failure handling:** Actions execute sequentially, stop on first failure. Response returns all results so far plus the error. Each step is logged to `meta_action_log` before execution so orphaned state can be cleaned up manually.

**`meta_action_log` table schema (new migration):**
```sql
CREATE TABLE meta_action_log (
  id SERIAL PRIMARY KEY,
  tenant_id INTEGER NOT NULL REFERENCES tenants(id),
  action_type VARCHAR(20) NOT NULL,  -- 'pause', 'graduate', 'create_adset', 'create_ad'
  target_type VARCHAR(20),           -- 'ad', 'adset', 'campaign'
  target_id VARCHAR(50),             -- Meta object ID
  from_campaign_id VARCHAR(50),
  to_campaign_id VARCHAR(50),
  payload JSONB,                     -- full request payload for audit
  status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'success', 'failed'
  error_message TEXT,
  executed_by VARCHAR(50) DEFAULT 'dashboard',  -- no user identity in bossViewAuth, log as 'dashboard'
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_action_log_tenant ON meta_action_log(tenant_id);
CREATE INDEX idx_action_log_created ON meta_action_log(created_at);
```

## Edge Cases

1. **Stale reviews:** Recommendations persist as "pending review" with staleness counter. New milestone snapshots stack below. No auto-action.
2. **Overlapping test cycles:** Multiple Hook Test campaigns stack in Testing section, each with own day tracker.
3. **Budget impact on graduation:** Confirm screen shows budget before/after table. Only appears when Graduate actions are checked.
4. **CRM cross-reference:** Kill recommendations check for attributed orders and warn if found.
5. **Cherry picks:** Kill operates at ad level, not hook level. Individual ads can be saved from killed hooks.
6. **Day counter with pauses:** Calendar days since `created_time`, paused days still count.
7. **No seasonal campaigns:** Seasonal section hidden entirely.

## Data Flow

```
Meta API ──→ CRM Backend (proxy + cache) ──→ Dashboard Frontend
                    │
                    ├── /meta/campaign-insights (spend, impressions, actions)
                    ├── /meta/ad-insights (per-ad breakdown with creative type)
                    ├── /meta/ad-creative/:id (thumbnail, title, body)
                    ├── /meta/campaign-roas (NEW — CRM orders × Meta campaigns)
                    └── /meta/actions (NEW — execute confirmed recommendations)
```

## Visual Reference

Mockup at `.superpowers/brainstorm/23456-1778233982/full-design.html`

## Out of Scope

- Tab 4 (Campaign Tracker) — parallel session
- CAPI integration (deferred, needs WhatsApp Cloud API)
- Ad attribution dropdown on CRM order form (separate CRM feature)
- Automated scheduled analysis (cron) — workbench is human-in-the-loop
