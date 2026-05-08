# Ads Workbench (Tab 3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Tab 3 (Ads) in the Boss View dashboard with a workbench reflecting the 3-tier graduation system, CRM-attributed ROAS, and interactive recommendation panels for ad management actions.

**Architecture:** Two-repo change. Backend (mom-crm-webapp) adds 3 new endpoints + 1 migration + 2 field additions. Frontend (minionions-dashboard) rewrites the Ads tab JS/HTML rendering with new classification logic, 5-section layout, and action execution UI. Backend changes deploy first since the frontend depends on them.

**Tech Stack:** Node.js/Express/Knex (backend), vanilla HTML/CSS/JS + Chart.js (frontend), Meta Marketing API v21.0, PostgreSQL

**Spec:** `docs/superpowers/specs/2026-05-08-ads-workbench-design.md`
**Mockup:** `.superpowers/brainstorm/23456-1778233982/full-design.html`

---

## File Map

### Backend (mom-crm-webapp)

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `server/src/services/metaApiService.js:382` | Add `adset_id`, `adset_name` to ad insights fields |
| Modify | `server/src/services/metaApiService.js` (new function) | Add `fetchActiveCampaigns()` for created_time + budgets |
| Create | `server/migrations/20260508_070_create_meta_action_log.js` | New table for action audit trail |
| Create | `server/src/models/metaActionLogModel.js` | CRUD for meta_action_log |
| Create | `server/src/controllers/metaActionsController.js` | Execute pause/graduate actions via Meta API |
| Modify | `server/src/controllers/metaController.js` | Add getCampaignRoas, getActiveCampaigns handlers |
| Modify | `server/src/models/adCatalogModel.js` | Add attribution query (orders JOIN meta_ad_catalog) |
| Modify | `server/src/routes/bossView.js:30-37` | Register new routes |

### Frontend (minionions-dashboard)

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `index.html:2691-3293` | Replace entire Ads tab (classification, rendering, actions) |

---

## Task 1: Add adset fields to Meta ad insights (backend)

**Files:**
- Modify: `server/src/services/metaApiService.js:382`

- [ ] **Step 1: Add adset_id and adset_name to the fields list**

In `_fetchAdInsightsCore()` at line 382, add `adset_id,adset_name` to the fields string:

```js
// Line 382 — add adset_id,adset_name after ad_id
const fields = 'ad_name,ad_id,adset_id,adset_name,campaign_name,campaign_id,objective,optimization_goal,spend,impressions,reach,cpm,cpp,ctr,frequency,inline_link_clicks,cost_per_inline_link_click,actions,action_values,cost_per_action_type,video_thruplay_watched_actions,cost_per_thruplay,purchase_roas';
```

- [ ] **Step 2: Add adset_id and adset_name to the ad result mapping**

In the result mapping section (~line 458-533), add the new fields to the returned object:

```js
adset_id: row.adset_id,
adset_name: row.adset_name,
```

Add these after the `ad_id` mapping line.

- [ ] **Step 3: Verify existing tests still pass**

Run: `cd ~/Documents/mom-crm-webapp && npm test`
Expected: All existing tests pass (no breakage from field additions).

- [ ] **Step 4: Manual verification**

Start the dev server and call the endpoint:
```bash
curl -s "http://localhost:3001/api/boss-view/meta/ad-insights?month=2026-05&objective=REPLIES" \
  -H "Authorization: Bearer <token>" | jq '.ads[0] | {adset_id, adset_name}'
```
Expected: Both fields populated with valid values.

- [ ] **Step 5: Commit**

```bash
git add server/src/services/metaApiService.js
git commit -m "feat(meta): add adset_id and adset_name to ad insights response"
```

---

## Task 2: Add fetchActiveCampaigns to Meta service (backend)

**Files:**
- Modify: `server/src/services/metaApiService.js` (add new function before module.exports at line 575)
- Modify: `server/src/controllers/metaController.js` (add handler)
- Modify: `server/src/routes/bossView.js:30-37` (add route)

- [ ] **Step 1: Add fetchActiveCampaigns function to metaApiService.js**

Add before `module.exports` (~line 575):

```js
async function fetchActiveCampaigns(tenantId) {
  const { token, adAccountId } = await loadMetaCredentials(tenantId);
  const url = `${META_API_BASE}/${adAccountId}/campaigns`;
  const params = new URLSearchParams({
    fields: 'id,name,objective,status,daily_budget,lifetime_budget,created_time',
    filtering: JSON.stringify([{ field: 'effective_status', operator: 'IN', value: ['ACTIVE'] }]),
    limit: '100',
    access_token: token,
  });
  const response = await fetch(`${url}?${params}`);
  const data = await response.json();
  if (data.error) throw new Error(`Meta API error: ${data.error.message}`);
  return (data.data || []).map(c => ({
    campaign_id: c.id,
    campaign_name: c.name,
    objective: c.objective,
    status: c.status,
    daily_budget: c.daily_budget ? Number(c.daily_budget) / 100 : null,
    lifetime_budget: c.lifetime_budget ? Number(c.lifetime_budget) / 100 : null,
    created_time: c.created_time,
  }));
}
```

Add `fetchActiveCampaigns` to module.exports.

- [ ] **Step 2: Add controller handler in metaController.js**

Add before `module.exports` (~line 349):

```js
async function getActiveCampaigns(req, res) {
  try {
    const campaigns = await metaApiService.fetchActiveCampaigns(req.tenantId);
    res.json({ campaigns });
  } catch (err) {
    console.error('getActiveCampaigns error:', err.message);
    res.status(500).json({ error: err.message });
  }
}
```

Add `getActiveCampaigns` to module.exports.

- [ ] **Step 3: Register route in bossView.js**

Add after the existing meta routes (~line 33):

```js
router.get('/meta/active-campaigns', metaController.getActiveCampaigns);
```

- [ ] **Step 4: Manual verification**

```bash
curl -s "http://localhost:3001/api/boss-view/meta/active-campaigns" \
  -H "Authorization: Bearer <token>" | jq '.campaigns[:2]'
```
Expected: Array of campaigns with `campaign_id`, `campaign_name`, `daily_budget`, `lifetime_budget`, `created_time`.

- [ ] **Step 5: Commit**

```bash
git add server/src/services/metaApiService.js server/src/controllers/metaController.js server/src/routes/bossView.js
git commit -m "feat(meta): add active-campaigns endpoint with created_time and budgets"
```

---

## Task 3: Campaign ROAS endpoint (backend)

**Files:**
- Modify: `server/src/models/adCatalogModel.js` (add attribution query)
- Modify: `server/src/controllers/metaController.js` (add handler)
- Modify: `server/src/routes/bossView.js` (add route)

- [ ] **Step 1: Add attribution query to adCatalogModel.js**

Add before `module.exports` (~line 72):

```js
async function getAttribution(tenantId, since, until) {
  const ads = await db('meta_ad_catalog as mac')
    .select(
      'mac.meta_ad_id',
      'mac.ad_name',
      'mac.campaign_id',
      'mac.campaign_name',
    )
    .count('o.id as conversions')
    .sum('o.total_pv as attributed_pv')
    .leftJoin('orders as o', function () {
      this.on('o.meta_ad_id', '=', 'mac.meta_ad_id')
        .andOn('o.tenant_id', '=', 'mac.tenant_id')
        .andOnVal('o.order_status', '!=', 'cancelled')
        .andOnVal('o.delivery_confirmed_date', '>=', since)
        .andOnVal('o.delivery_confirmed_date', '<=', until);
    })
    .where('mac.tenant_id', tenantId)
    .groupBy('mac.meta_ad_id', 'mac.ad_name', 'mac.campaign_id', 'mac.campaign_name');

  const attribution = await db('orders')
    .where({ tenant_id: tenantId })
    .whereNot('order_status', 'cancelled')
    .whereBetween('delivery_confirmed_date', [since, until])
    .select(
      db.raw('COUNT(CASE WHEN meta_ad_id IS NOT NULL THEN 1 END) as attributed_orders'),
      db.raw('COUNT(*) as total_orders'),
    )
    .first();

  return {
    ads: ads.map(row => ({
      ad_id: row.meta_ad_id,
      ad_name: row.ad_name,
      campaign_id: row.campaign_id,
      campaign_name: row.campaign_name,
      conversions: Number(row.conversions) || 0,
      attributed_pv: Number(row.attributed_pv) || 0,
    })),
    attribution_rate: attribution.total_orders > 0
      ? Number(attribution.attributed_orders) / Number(attribution.total_orders)
      : 0,
    total_orders: Number(attribution.total_orders),
    attributed_orders: Number(attribution.attributed_orders),
  };
}
```

Add `getAttribution` to module.exports.

- [ ] **Step 2: Add controller handler**

In `metaController.js`, add before `module.exports`:

```js
async function getCampaignRoas(req, res) {
  try {
    const { since, until } = req.query;
    if (!since || !until) return res.status(400).json({ error: 'since and until required' });
    const result = await adCatalogModel.getAttribution(req.tenantId, since, until);
    res.json({ period: { since, until }, ...result });
  } catch (err) {
    console.error('getCampaignRoas error:', err.message);
    res.status(500).json({ error: err.message });
  }
}
```

Add `getCampaignRoas` to module.exports. Add `const adCatalogModel = require('../models/adCatalogModel');` to imports if not already present.

- [ ] **Step 3: Register route**

In `bossView.js`, add:

```js
router.get('/meta/campaign-roas', metaController.getCampaignRoas);
```

- [ ] **Step 4: Manual verification**

```bash
curl -s "http://localhost:3001/api/boss-view/meta/campaign-roas?since=2026-04-24&until=2026-05-08" \
  -H "Authorization: Bearer <token>" | jq '{attribution_rate, total_orders, ads_count: (.ads | length)}'
```
Expected: JSON with `attribution_rate` (0-1), `total_orders`, and `ads` array.

- [ ] **Step 5: Commit**

```bash
git add server/src/models/adCatalogModel.js server/src/controllers/metaController.js server/src/routes/bossView.js
git commit -m "feat(meta): add campaign-roas endpoint with CRM order attribution"
```

---

## Task 4: Meta action log migration + model (backend)

**Files:**
- Create: `server/migrations/20260508_070_create_meta_action_log.js`
- Create: `server/src/models/metaActionLogModel.js`

- [ ] **Step 1: Create migration**

```js
// server/migrations/20260508_070_create_meta_action_log.js
exports.up = async function (knex) {
  await knex.schema.createTable('meta_action_log', (table) => {
    table.increments('id').primary();
    table.integer('tenant_id').notNullable().references('id').inTable('tenants');
    table.string('action_type', 20).notNullable(); // pause, graduate, create_adset, create_ad
    table.string('target_type', 20);               // ad, adset, campaign
    table.string('target_id', 50);
    table.string('from_campaign_id', 50);
    table.string('to_campaign_id', 50);
    table.jsonb('payload');
    table.string('status', 20).notNullable().defaultTo('pending'); // pending, success, failed
    table.text('error_message');
    table.string('executed_by', 50).defaultTo('dashboard');
    table.timestamp('created_at').defaultTo(knex.fn.now());
    table.index('tenant_id', 'idx_action_log_tenant');
    table.index('created_at', 'idx_action_log_created');
  });
};

exports.down = async function (knex) {
  await knex.schema.dropTableIfExists('meta_action_log');
};
```

- [ ] **Step 2: Create model**

```js
// server/src/models/metaActionLogModel.js
const db = require('../db');

async function log(tenantId, entry) {
  const [row] = await db('meta_action_log')
    .insert({
      tenant_id: tenantId,
      action_type: entry.action_type,
      target_type: entry.target_type || null,
      target_id: entry.target_id || null,
      from_campaign_id: entry.from_campaign_id || null,
      to_campaign_id: entry.to_campaign_id || null,
      payload: entry.payload ? JSON.stringify(entry.payload) : null,
      status: entry.status || 'pending',
      error_message: entry.error_message || null,
      executed_by: entry.executed_by || 'dashboard',
    })
    .returning('*');
  return row;
}

async function updateStatus(id, status, errorMessage) {
  await db('meta_action_log')
    .where({ id })
    .update({ status, error_message: errorMessage || null });
}

module.exports = { log, updateStatus };
```

- [ ] **Step 3: Run migration**

```bash
cd ~/Documents/mom-crm-webapp && npx knex migrate:latest
```
Expected: Migration 070 applied successfully.

- [ ] **Step 4: Commit**

```bash
git add server/migrations/20260508_070_create_meta_action_log.js server/src/models/metaActionLogModel.js
git commit -m "feat: add meta_action_log table and model for workbench audit trail"
```

---

## Task 5: Actions execution endpoint (backend)

**Files:**
- Create: `server/src/controllers/metaActionsController.js`
- Modify: `server/src/services/metaApiService.js` (add pauseObject, createAdSet, createAd, getCreativeId)
- Modify: `server/src/routes/bossView.js` (add route)

- [ ] **Step 1: Add Meta API write functions to metaApiService.js**

Add before `module.exports`:

```js
async function pauseObject(tenantId, objectId) {
  const { token } = await loadMetaCredentials(tenantId);
  const response = await fetch(`${META_API_BASE}/${objectId}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({ status: 'PAUSED', access_token: token }),
  });
  const data = await response.json();
  if (data.error) throw new Error(`Meta API pause error: ${data.error.message}`);
  return data;
}

async function getCreativeId(tenantId, adId) {
  const { token } = await loadMetaCredentials(tenantId);
  const response = await fetch(`${META_API_BASE}/${adId}?fields=creative{id}&access_token=${token}`);
  const data = await response.json();
  if (data.error) throw new Error(`Meta API creative fetch error: ${data.error.message}`);
  return data.creative?.id;
}

async function createAdSet(tenantId, campaignId, name, dailyBudgetCents) {
  const { token, adAccountId } = await loadMetaCredentials(tenantId);
  const params = new URLSearchParams({
    campaign_id: campaignId,
    name,
    daily_budget: String(dailyBudgetCents),
    optimization_goal: 'CONVERSATIONS',
    billing_event: 'IMPRESSIONS',
    destination_type: 'MESSENGER',
    targeting: JSON.stringify({ geo_locations: { countries: ['HK'] } }),
    status: 'ACTIVE',
    access_token: token,
  });
  const response = await fetch(`${META_API_BASE}/${adAccountId}/adsets`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: params,
  });
  const data = await response.json();
  if (data.error) throw new Error(`Meta API create adset error: ${data.error.message}`);
  return data;
}

async function createAd(tenantId, adSetId, name, creativeId) {
  const { token, adAccountId } = await loadMetaCredentials(tenantId);
  const params = new URLSearchParams({
    adset_id: adSetId,
    name,
    creative: JSON.stringify({ creative_id: creativeId }),
    status: 'ACTIVE',
    access_token: token,
  });
  const response = await fetch(`${META_API_BASE}/${adAccountId}/ads`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: params,
  });
  const data = await response.json();
  if (data.error) throw new Error(`Meta API create ad error: ${data.error.message}`);
  return data;
}
```

Add `pauseObject, getCreativeId, createAdSet, createAd` to module.exports.

- [ ] **Step 2: Create metaActionsController.js**

```js
// server/src/controllers/metaActionsController.js
const metaApiService = require('../services/metaApiService');
const actionLog = require('../models/metaActionLogModel');

async function executeActions(req, res) {
  const { actions } = req.body;
  if (!actions || !Array.isArray(actions) || actions.length === 0) {
    return res.status(400).json({ error: 'actions array required' });
  }

  const results = [];

  for (const action of actions) {
    try {
      if (action.type === 'pause') {
        const logEntry = await actionLog.log(req.tenantId, {
          action_type: 'pause',
          target_type: action.target_type || 'ad',
          target_id: action.target_id,
          payload: action,
          status: 'pending',
        });
        await metaApiService.pauseObject(req.tenantId, action.target_id);
        await actionLog.updateStatus(logEntry.id, 'success');
        results.push({ action: 'pause', target_id: action.target_id, status: 'success' });

      } else if (action.type === 'graduate') {
        // Step 1: Find Proving Ground campaign
        const campaigns = await metaApiService.fetchActiveCampaigns(req.tenantId);
        const pgCampaign = campaigns.find(c => c.campaign_name.toLowerCase().includes('proving ground'));
        if (!pgCampaign) {
          results.push({ action: 'graduate', ad_id: action.ad_id, status: 'failed', error: 'Proving Ground campaign not found' });
          break; // Stop all — graduate failure is critical (budget impact)
        }

        // Step 2: Get creative ID from source ad
        const creativeId = await metaApiService.getCreativeId(req.tenantId, action.ad_id);
        if (!creativeId) {
          results.push({ action: 'graduate', ad_id: action.ad_id, status: 'failed', error: 'Could not fetch creative_id' });
          break; // Stop all — can't graduate without creative
        }

        // Step 3: Create ad set in Proving Ground
        const adSetName = action.ad_name || `Graduated ${action.ad_id}`;
        const logAdSet = await actionLog.log(req.tenantId, {
          action_type: 'create_adset',
          target_type: 'adset',
          to_campaign_id: pgCampaign.campaign_id,
          payload: { ad_set_name: adSetName, daily_budget: 2500 },
          status: 'pending',
        });
        const adSetResult = await metaApiService.createAdSet(
          req.tenantId, pgCampaign.campaign_id, adSetName, 2500
        );
        await actionLog.updateStatus(logAdSet.id, 'success');

        // Step 4: Create ad in new ad set
        const logAd = await actionLog.log(req.tenantId, {
          action_type: 'create_ad',
          target_type: 'ad',
          target_id: adSetResult.id,
          payload: { creative_id: creativeId, ad_name: adSetName },
          status: 'pending',
        });
        await metaApiService.createAd(req.tenantId, adSetResult.id, adSetName, creativeId);
        await actionLog.updateStatus(logAd.id, 'success');

        // Step 5: Pause source ad in Testing
        const logPause = await actionLog.log(req.tenantId, {
          action_type: 'pause',
          target_type: 'ad',
          target_id: action.ad_id,
          from_campaign_id: action.from_campaign || null,
          payload: { reason: 'Graduated to Proving Ground' },
          status: 'pending',
        });
        await metaApiService.pauseObject(req.tenantId, action.ad_id);
        await actionLog.updateStatus(logPause.id, 'success');

        results.push({ action: 'graduate', ad_id: action.ad_id, status: 'success', new_adset_id: adSetResult.id });

      } else {
        results.push({ action: action.type, status: 'skipped', reason: 'Unknown action type' });
      }
    } catch (err) {
      results.push({ action: action.type, target_id: action.target_id || action.ad_id, status: 'failed', error: err.message });
      break; // Stop on first failure
    }
  }

  const allSuccess = results.every(r => r.status === 'success' || r.status === 'skipped');
  res.status(allSuccess ? 200 : 207).json({ results });
}

module.exports = { executeActions };
```

- [ ] **Step 3: Register route**

In `bossView.js`, add import and route:

```js
const metaActionsController = require('../controllers/metaActionsController');
// ... in routes section:
router.post('/meta/actions', metaActionsController.executeActions);
```

- [ ] **Step 4: Commit**

```bash
git add server/src/services/metaApiService.js server/src/controllers/metaActionsController.js server/src/routes/bossView.js
git commit -m "feat(meta): add actions endpoint for pause/graduate execution with audit logging"
```

---

## Task 6: Deploy backend changes

- [ ] **Step 1: Run full test suite**

```bash
cd ~/Documents/mom-crm-webapp && npm test
```
Expected: All tests pass.

- [ ] **Step 2: Deploy to staging**

```bash
git push origin main
```
Coolify auto-deploys staging on push.

- [ ] **Step 3: Run migration on staging**

Verify migration 070 applied automatically (Coolify runs migrations on deploy). Check logs if needed.

- [ ] **Step 4: Verify endpoints on staging**

```bash
# Active campaigns
curl -s "https://app.solworks.io/api/boss-view/meta/active-campaigns" -H "Authorization: Bearer <token>" | jq '.campaigns | length'

# Campaign ROAS
curl -s "https://app.solworks.io/api/boss-view/meta/campaign-roas?since=2026-04-24&until=2026-05-08" -H "Authorization: Bearer <token>" | jq '.attribution_rate'

# Ad insights (verify adset fields)
curl -s "https://app.solworks.io/api/boss-view/meta/ad-insights?month=2026-05&objective=REPLIES" -H "Authorization: Bearer <token>" | jq '.ads[0] | {adset_id, adset_name}'
```

- [ ] **Step 5: Commit/tag deploy**

```bash
git tag backend-ads-workbench-v1
git push origin --tags
```

---

## Task 7: Frontend — Classification logic + KPI bar + Funnel bar

**Files:**
- Modify: `index.html` (replace `classifyCampaign` at line 2691, rewrite top of `loadAdsTab` at line 3196)

- [ ] **Step 1: Replace classifyCampaign function**

Replace lines 2691-2696 with:

```js
const SEASONAL_KEYWORDS = ['雙親', '新年', '38', '聖誕', '中秋', '雙11'];

// campaignMeta is optional — from /meta/active-campaigns (has lifetime_budget)
function classifyCampaign(name, campaignMeta) {
  const n = name.toLowerCase();
  if (n.includes('hook test')) return 'testing';
  if (n.includes('proving ground')) return 'proving';
  if (n.includes('omnipresence') || n.startsWith('pe ')) return 'branding';
  if (SEASONAL_KEYWORDS.some(kw => name.includes(kw))) return 'seasonal';
  if (campaignMeta && campaignMeta.lifetime_budget && !campaignMeta.daily_budget) return 'seasonal';
  return 'proven';
}
```

- [ ] **Step 2: Add KPI bar rendering function**

Add after the classification function:

```js
function renderKpiBar(allAds, peAds, roasData, activeCampaigns) {
  // allAds = REPLIES-objective ads, peAds = POST_ENGAGEMENT-objective ads
  // Total Spend must include BOTH to match "sum across all campaigns"
  const totalSpend = [...allAds, ...peAds].reduce((s, a) => s + Number(a.spend || 0), 0);
  const totalMsgs = allAds.reduce((s, a) => s + getAdMessages(a), 0);
  const totalComments = [...allAds, ...peAds].reduce((s, a) => {
    const actions = a.actions || [];
    const c = actions.find(x => x.action_type === 'comment' || x.action_type === 'onsite_conversion.post_net_comment');
    return s + (c ? Number(c.value) : 0);
  }, 0);
  const avgCpr = (totalMsgs + totalComments) > 0 ? totalSpend / (totalMsgs + totalComments) : 0;
  const totalPv = roasData ? roasData.ads.reduce((s, a) => s + a.attributed_pv, 0) : 0;
  const roas = totalSpend > 0 ? totalPv / totalSpend : 0;
  const attrPct = roasData ? Math.round(roasData.attribution_rate * 100) : 0;

  const dailyBudget = activeCampaigns.reduce((s, c) => s + (c.daily_budget || 0), 0);
  const overCap = dailyBudget > 1050;

  return `
    <div style="display:flex;gap:12px;align-items:stretch;">
      <div style="flex:5;">
        <div class="kpi-row kpi-row--5">
          <div class="kpi-cell"><div class="kpi-label">Total Spend</div><div class="kpi-value">RM ${totalSpend.toLocaleString('en', {minimumFractionDigits: 0, maximumFractionDigits: 0})}</div></div>
          <div class="kpi-cell"><div class="kpi-label">Messages</div><div class="kpi-value">${totalMsgs.toLocaleString()}</div></div>
          <div class="kpi-cell"><div class="kpi-label">Comments</div><div class="kpi-value">${totalComments.toLocaleString()}</div></div>
          <div class="kpi-cell"><div class="kpi-label">Avg CPR</div><div class="kpi-value">RM ${avgCpr.toFixed(2)}</div><div class="kpi-delta" style="color:var(--text-3);">spend / (msgs+comments)</div></div>
          <div class="kpi-cell"><div class="kpi-label">ROAS</div><div class="kpi-value">${roas.toFixed(2)}x</div><div class="kpi-delta" style="color:${attrPct < 50 ? 'var(--orange)' : 'var(--text-3)'};">${attrPct}% attributed</div></div>
        </div>
      </div>
      <div style="flex:1;">
        <div style="background:var(--surface);border:1px solid var(--border);border-radius:12px;padding:18px 20px;height:100%;display:flex;flex-direction:column;justify-content:center;">
          <div class="kpi-label">Daily Budget</div>
          <div class="kpi-value">RM ${dailyBudget.toLocaleString()}</div>
          ${overCap ? `<div class="kpi-delta" style="color:var(--orange);">cap RM 1,050 (+${dailyBudget - 1050} seasonal)</div>` : ''}
        </div>
      </div>
    </div>`;
}
```

- [ ] **Step 3: Add funnel bar rendering function**

```js
function renderFunnelBar(groups) {
  const tier = (key, label, color) => {
    const g = groups[key] || { ads: [], campaigns: [] };
    const spend = g.ads.reduce((s, a) => s + Number(a.spend || 0), 0);
    const msgs = g.ads.reduce((s, a) => s + getAdMessages(a), 0);
    const budget = g.campaigns.reduce((s, c) => s + (c.daily_budget || 0), 0);
    const cpr = msgs > 0 ? spend / msgs : 0;
    return `<div style="flex:1;padding:14px 18px;text-align:center;border-right:1px solid var(--border-light);position:relative;">
      <div style="font-size:10px;font-weight:600;text-transform:uppercase;letter-spacing:0.06em;color:${color};margin-bottom:4px;">${label}</div>
      <div style="font-family:var(--mono);font-size:14px;font-weight:600;">${g.ads.length} ads · RM${budget}/d</div>
      <div style="font-size:11px;color:var(--text-3);margin-top:2px;">${msgs > 0 ? `CPR RM${Math.round(cpr)} · ${msgs} msgs` : 'No data yet'}</div>
      ${key !== 'proven' ? '<span style="position:absolute;right:-10px;top:50%;transform:translateY(-50%);z-index:1;color:var(--border);font-size:16px;">→</span>' : ''}
    </div>`;
  };
  return `<div style="display:flex;background:var(--surface);border:1px solid var(--border-light);border-radius:12px;overflow:hidden;margin-bottom:24px;">
    ${tier('testing', 'Testing (M)', 'var(--orange)')}
    ${tier('proving', 'Proving Ground (M)', '#CA8A04')}
    ${tier('proven', 'Proven (M)', 'var(--green)')}
  </div>`;
}
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: new classification logic + KPI bar + funnel bar for ads workbench"
```

---

## Task 8: Frontend — Rewrite Ads tab HTML template

**Files:**
- Modify: `index.html` (lines ~1118-1202 — the static HTML for the Ads tab panel)

- [ ] **Step 1: Replace the existing Ads tab HTML structure**

The current HTML at lines ~1118-1202 uses old 3-section layout with DOM IDs like `adsTestBanner`, `adsConvBanner`, `adsBrandBanner`, `adsTestCampaigns`, `adsConvCampaigns`, `adsBrandCampaigns`. Replace the entire `<div id="panelAds" class="tab-panel">` content with a single container that `loadAdsTab()` will populate via `innerHTML`:

```html
<div id="panelAds" class="tab-panel">
  <div id="adsWorkbench" class="content">
    <div class="loading">Loading ads data…</div>
  </div>
</div>
```

The JS `loadAdsTab()` will build the entire layout dynamically (KPI bar, funnel, all 5 sections) and set `adsWorkbench.innerHTML`.

- [ ] **Step 2: Add new CSS for workbench components**

Add CSS for all new components that don't exist in the current stylesheet: day tracker, milestones, proving ground cards, recommendation panels, seasonal section, funnel bar, budget card. Reference the mockup CSS for exact styles.

Key new CSS classes needed: `.day-tracker`, `.day-node`, `.day-dot`, `.day-line-wrap`, `.day-line-fill`, `.pulse-dot`, `.milestone`, `.pg-card`, `.pg-progress`, `.reco-panel`, `.reco-item`, `.reco-check`, `.budget-table`, `.seasonal-head`, `.seasonal-sub`, `.funnel-bar`, `.funnel-tier`, `.collapsible-toggle`, `.collapsible-body`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: rewrite ads tab HTML template + add workbench CSS"
```

---

## Task 9: Frontend — Seasonal section

**Files:**
- Modify: `index.html` (add rendering function)

- [ ] **Step 1: Add renderSeasonalSection function**

This renders the collapsible seasonal section with ad-level rows. Include thumbnail support, CPR color coding, and ROAS per ad. Full implementation — see mockup for visual reference.

Key elements:
- Collapsible campaign KPIs (click title to expand)
- Countdown badge (red when ≤14 days)
- Sub-campaign grouping by (P)/(V) prefix matching
- Ad rows with thumbnail, name, Spend/Conv/CPR/ROAS

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: seasonal section with collapsible KPIs and ad-level rows"
```

---

## Task 10: Frontend — Branding (PE) + Proven (M) sections

**Files:**
- Modify: `index.html` (add rendering functions)

- [ ] **Step 1: Add renderBrandingPeSection function**

Replaces existing `renderBrandingSection`. Shows: Spend | Engagements | Comments | Saves | CTR (all) per ad. Expandable (top 3 + toggle).

- [ ] **Step 2: Add renderProvenSection function**

Replaces existing `renderConversionSection`. Campaign rows with Spend | Conv | CPR | ROAS. Collapsed by default, expand to show ads with thumbnails. CRM badge for campaigns with attributed orders.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: branding (PE) and proven (M) sections with ROAS and thumbnails"
```

---

## Task 11: Frontend — Testing (M) section with day tracker

**Files:**
- Modify: `index.html` (add rendering function)

- [ ] **Step 1: Add renderTestingSection function**

Replaces existing `renderTestSection`. Key elements:
- Campaign header with date range
- Day tracker timeline with animated pulse (position = progress between milestones)
- Day counter ("Day X of 14")
- Milestone snapshot tabs (computed from Meta API date ranges)
- Ad set groups with thumbnails and verdict badges
- Day 14 recommendation panel (locked until day 14, interactive checkboxes when active)
- Budget impact table (shown only when Graduate checked)
- Confirm button that POSTs to `/meta/actions`

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: testing section with day tracker, milestones, and recommendation panel"
```

---

## Task 12: Frontend — Proving Ground (M) section

**Files:**
- Modify: `index.html` (add rendering function)

- [ ] **Step 1: Add renderProvingGroundSection function**

Key elements:
- Per-ad cards with 48px thumbnail
- 5-column stats: Spend | Conv | CPR | ROAS | Budget
- Progress bar (day X / 7) with color coding
- Day 7 verdict panel (per-ad, unlocks at 7 days)

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: proving ground section with progress bars and day 7 verdict panel"
```

---

## Task 13: Frontend — Wire up loadAdsTab + API calls

**Files:**
- Modify: `index.html` (rewrite `loadAdsTab` at line 3196)

- [ ] **Step 1: Rewrite loadAdsTab**

Replace lines 3196-3293. New flow:
1. Build date params (keep existing logic)
2. Fetch 5 parallel API calls via `Promise.all([apiSafe(...)])`:
   - `/meta/campaign-insights?${dateParams}`
   - `/meta/ad-insights?${dateParams}&objective=REPLIES`
   - `/meta/ad-insights?${dateParams}&objective=POST_ENGAGEMENT`
   - `/meta/active-campaigns`
   - `/meta/campaign-roas?since=${since}&until=${until}`
3. Build campaign meta lookup map from active-campaigns (for `classifyCampaign` lifetime_budget check)
4. Classify all campaigns using new `classifyCampaign(name, campaignMeta)`
5. Group ads by tier (testing/proving/proven/seasonal/branding)
6. Build `adsWorkbench.innerHTML` with all sections: KPI bar → Funnel → Seasonal → Branding+Proven (two-col) → Testing+Proving (two-col)
7. Wire up time pill click handlers (keep existing pattern)
8. Wire up collapsible toggles, campaign expand/collapse, and recommendation confirm button → POST `/meta/actions`

- [ ] **Step 2: Test full page load**

Open `http://mom.solworks.io`, navigate to Ads tab. Verify:
- KPI bar shows correct 5+1 layout
- Funnel bar shows correct tier counts
- All 5 sections render with data
- Time pills switch correctly
- Campaign rows expand/collapse
- Thumbnails load

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: wire up ads workbench with all sections and API calls"
```

---

## Task 14: Frontend deploy + visual QA

- [ ] **Step 1: Push to main**

```bash
cd ~/Documents/minionions-dashboard && git push origin main
```
Coolify auto-deploys (or manual curl deploy).

- [ ] **Step 2: Visual QA against mockup**

Open `http://mom.solworks.io` and compare each section against the mockup at `.superpowers/brainstorm/23456-1778233982/full-design.html`. Check:
- KPI values match expected
- Funnel bar counts match
- Seasonal countdown correct
- Branding shows Saves + Comments + CTR
- Proven shows ROAS with attribution %
- Testing day tracker pulse position correct
- Proving Ground progress bars correct

- [ ] **Step 3: Test recommendation flow (if Day 14 reached)**

If Hook Test Round 2 has reached Day 14, verify:
- Recommendations auto-generated from threshold rules
- Checkboxes toggle correctly
- Budget table appears/hides based on Graduate selections
- Confirm button sends POST to `/meta/actions`
- Results reflected in Meta Ads Manager

- [ ] **Step 4: Final commit if any fixes**

```bash
git add index.html && git commit -m "fix: visual QA adjustments for ads workbench"
```
