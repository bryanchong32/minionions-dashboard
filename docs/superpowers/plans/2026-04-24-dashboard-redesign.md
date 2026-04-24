# Boss View Dashboard Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the Minionions Boss View dashboard from a flat 11-chart page into a 3-tab operational dashboard with sales funnels, follow-up pipeline, and ad attribution.

**Architecture:** Two independent workstreams: (A) Backend API — new endpoints on the CRM Express backend for pipeline, composition, and funnel data; (B) Frontend — rewrite the single `index.html` dashboard with tab structure, new sections, and pipeline visualization. Plus (C) Ad catalog sync + attribution dropdown on the CRM order form.

**Tech Stack:** Node.js/Express backend (Knex + PostgreSQL), vanilla HTML/CSS/JS frontend with Chart.js, Outfit + Geist Mono fonts.

**Spec:** `docs/superpowers/specs/2026-04-24-dashboard-redesign-design.md`

**Repos:**
- CRM backend: `~/Documents/mom-crm-webapp` (branch: `feature/boss-view-v2`)
- Dashboard frontend: `~/Documents/minionions-dashboard` (branch: `main`)

---

## File Structure

### CRM Backend (~/Documents/mom-crm-webapp)

**Create:**
- `server/migrations/20260424_064_ad_catalog_and_meta_ad_id.js` — new `meta_ad_catalog` table + `meta_ad_id` column on orders
- `server/src/models/pipelineModel.js` — pipeline stage derivation queries (onboarding, rotation, reorder, membership)
- `server/src/models/adCatalogModel.js` — CRUD for `meta_ad_catalog`
- `server/src/controllers/adCatalogController.js` — ad catalog list endpoint
- `server/src/services/adCatalogSync.js` — nightly sync from Meta API

**Modify:**
- `server/src/models/bossViewModel.js` — add `getLifetimeComposition()`, `getNewSalesFunnel()`, `getRepeatSalesFunnel()`
- `server/src/controllers/bossViewController.js` — add handlers for new endpoints
- `server/src/routes/bossView.js` — mount new routes
- `server/src/routes/meta.js` — mount ad catalog route
- `server/src/index.js` — add ad catalog sync to daily cron
- `server/src/controllers/orderController.js` — handle `meta_ad_id` on order save
- `SCHEMA.md` — document new table + column

### Dashboard Frontend (~/Documents/minionions-dashboard)

**Modify:**
- `index.html` — full rewrite: 3-tab structure, new design system, all new sections

---

## Phase A: Backend API Endpoints

### Task 1: Migration — `meta_ad_catalog` table + `orders.meta_ad_id`

**Files:**
- Create: `server/migrations/20260424_064_ad_catalog_and_meta_ad_id.js`

- [ ] **Step 1: Write migration**

```javascript
exports.up = async function (knex) {
  await knex.schema.createTable('meta_ad_catalog', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.uuid('tenant_id').notNullable().references('id').inTable('tenants');
    table.string('meta_ad_id').notNullable();
    table.string('ad_name');
    table.string('campaign_id');
    table.string('campaign_name');
    table.string('objective');
    table.text('thumbnail_url');
    table.string('status').defaultTo('active');
    table.timestamp('last_synced_at');
    table.timestamps(true, true);
    table.unique(['tenant_id', 'meta_ad_id']);
  });

  await knex.schema.alterTable('orders', (table) => {
    table.string('meta_ad_id');
  });
};

exports.down = async function (knex) {
  await knex.schema.dropTableIfExists('meta_ad_catalog');
  await knex.schema.alterTable('orders', (table) => {
    table.dropColumn('meta_ad_id');
  });
};
```

- [ ] **Step 2: Run migration on staging**

```bash
ssh -i ~/.ssh/id_ed25519 deploy@5.223.49.206 "docker exec rqpvl54p59lr1z3ev8ev2m1t psql -U postgres -d ecomwave_crm -c \"
CREATE TABLE IF NOT EXISTS meta_ad_catalog (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  meta_ad_id VARCHAR NOT NULL,
  ad_name VARCHAR,
  campaign_id VARCHAR,
  campaign_name VARCHAR,
  objective VARCHAR,
  thumbnail_url TEXT,
  status VARCHAR DEFAULT 'active',
  last_synced_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(tenant_id, meta_ad_id)
);
ALTER TABLE orders ADD COLUMN IF NOT EXISTS meta_ad_id VARCHAR;
GRANT ALL ON meta_ad_catalog TO ecomwave_user;
GRANT SELECT ON meta_ad_catalog TO backup_readonly;
\""
```

- [ ] **Step 3: Commit**

```bash
git add server/migrations/20260424_064_ad_catalog_and_meta_ad_id.js
git commit -m "feat: migration 064 — meta_ad_catalog table + orders.meta_ad_id column"
```

---

### Task 2: Pipeline Model — Stage Derivation Queries

**Files:**
- Create: `server/src/models/pipelineModel.js`

This is the most complex model. It derives customer journey stages from task data.

- [ ] **Step 1: Create pipelineModel.js with onboarding pipeline query**

The onboarding query groups customers by their earliest incomplete onboarding task on their most recent delivered order.

```javascript
const db = require('../db');

// Task type ordering for stage derivation
const ONBOARDING_ORDER = ['DOSAGE_GUIDE', 'COMPLIANCE_CHECK', 'MONTH_REVIEW'];
const ROTATION_ORDER = ['ROTATE_KNOWLEDGE', 'ROTATE_TESTIMONIAL', 'ROTATE_TIPS', 'ROTATE_CHECKIN'];
// Use exclusion list — more robust if new statuses are added in the future
const COMPLETED_STATUSES = ['completed', 'skipped'];

async function getOnboardingPipeline(tenantId) {
  // Get all onboarding tasks for each customer's most recent delivered order
  const rows = await db.raw(`
    WITH latest_delivered AS (
      SELECT DISTINCT ON (customer_id) customer_id, id as order_id
      FROM orders
      WHERE tenant_id = ? AND delivery_confirmed = true AND order_status != 'cancelled'
      ORDER BY customer_id, delivery_confirmed_date DESC
    ),
    onboarding_tasks AS (
      SELECT t.customer_id, t.task_type, t.status
      FROM tasks t
      JOIN latest_delivered ld ON t.order_id = ld.order_id
      WHERE t.tenant_id = ?
        AND t.task_type IN ('DOSAGE_GUIDE', 'COMPLIANCE_CHECK', 'MONTH_REVIEW')
    )
    SELECT customer_id, task_type, status FROM onboarding_tasks
  `, [tenantId, tenantId]);

  // Group by customer, derive stage
  const customerTasks = {};
  for (const row of rows.rows) {
    if (!customerTasks[row.customer_id]) customerTasks[row.customer_id] = {};
    customerTasks[row.customer_id][row.task_type] = row.status;
  }

  const stages = { DOSAGE_GUIDE: { total: 0, actioned: 0, pending: 0 }, COMPLIANCE_CHECK: { total: 0, actioned: 0, pending: 0 }, MONTH_REVIEW: { total: 0, actioned: 0, pending: 0 }, GRADUATED: { total: 0 } };

  for (const tasks of Object.values(customerTasks)) {
    let currentStage = 'GRADUATED';
    for (const type of ONBOARDING_ORDER) {
      const status = tasks[type];
      if (status && !COMPLETED_STATUSES.includes(status)) {
        currentStage = type;
        break;
      }
    }
    if (currentStage === 'GRADUATED') {
      stages.GRADUATED.total++;
    } else {
      stages[currentStage].total++;
      if (tasks[currentStage] === 'pending') {
        stages[currentStage].pending++;
      } else {
        stages[currentStage].actioned++;
      }
    }
  }

  return stages;
}
```

- [ ] **Step 2: Add membership pipeline query**

```javascript
async function getMembershipPipeline(tenantId) {
  const pending = await db('tasks')
    .where('tenant_id', tenantId)
    .whereIn('task_type', ['MEMBERSHIP_ASK', 'MEMBERSHIP_DEADLINE'])
    .whereIn('status', ACTIVE_STATUSES)
    .countDistinct('customer_id as count')
    .first();

  const registered = await db('customers')
    .where('tenant_id', tenantId)
    .where('membership_status', 'registered')
    .count('id as count')
    .first();

  const total = await db('customers')
    .where('tenant_id', tenantId)
    .count('id as count')
    .first();

  // Completion % for pending stage: how many have been sent vs still pending
  const sentCount = await db('tasks')
    .where('tenant_id', tenantId)
    .whereIn('task_type', ['MEMBERSHIP_ASK', 'MEMBERSHIP_DEADLINE'])
    .whereIn('status', ['sent', 'replied', 'completed'])
    .countDistinct('customer_id as count')
    .first();

  const pendingCount = parseInt(pending.count, 10);
  const registeredCount = parseInt(registered.count, 10);
  const totalCustomers = parseInt(total.count, 10);
  const actionedCount = parseInt(sentCount.count, 10);

  return {
    ask_pending: pendingCount,
    actioned: actionedCount,
    pending_untouched: pendingCount - actionedCount,
    registered: registeredCount,
    rate: totalCustomers > 0 ? Math.round((registeredCount / totalCustomers) * 1000) / 10 : 0,
  };
}
```

- [ ] **Step 3: Add reorder pipeline query**

```javascript
async function getReorderPipeline(tenantId, yearMonth) {
  // Reorder due — exclude lapsed customers
  const reorderDue = await db.raw(`
    SELECT t.target_sku, t.status, COUNT(DISTINCT t.customer_id) as count
    FROM tasks t
    JOIN customers c ON t.customer_id = c.id
    WHERE t.tenant_id = ?
      AND t.task_type = 'REORDER'
      AND t.status IN ('pending', 'sent', 'no_response')
      AND (c.last_order_date IS NULL OR c.last_order_date > NOW() - INTERVAL '90 days' OR c.total_orders < 2)
    GROUP BY t.target_sku, t.status
  `, [tenantId]);

  // Group by product
  const products = {};
  let totalDue = 0;
  let totalActioned = 0;
  let totalPending = 0;

  for (const row of reorderDue.rows) {
    if (!products[row.target_sku]) products[row.target_sku] = 0;
    const count = parseInt(row.count, 10);
    products[row.target_sku] += count;
    totalDue += count;
    if (row.status === 'pending') totalPending += count;
    else totalActioned += count;
  }

  // Converted this month — repeat orders placed in the given month
  const [year, month] = yearMonth.split('-').map(Number);
  const monthStart = `${yearMonth}-01`;
  const daysInMonth = new Date(year, month, 0).getDate();
  const monthEnd = `${yearMonth}-${String(daysInMonth).padStart(2, '0')}`;

  const converted = await db('orders')
    .where('tenant_id', tenantId)
    .whereNot('order_status', 'cancelled')
    .where('is_first_order', false)
    .whereBetween('order_date', [monthStart, monthEnd])
    .count('id as count')
    .first();

  // Lapsed
  const lapsed = await db('tasks')
    .where('tenant_id', tenantId)
    .where('task_type', 'LAPSED')
    .whereIn('status', ACTIVE_STATUSES)
    .count('id as count')
    .first();

  const lapsedCompleted = await db('tasks')
    .where('tenant_id', tenantId)
    .where('task_type', 'LAPSED')
    .where('status', 'completed')
    .count('id as count')
    .first();

  const lapsedCount = parseInt(lapsed.count, 10);
  const lapsedWon = parseInt(lapsedCompleted.count, 10);
  const convertedCount = parseInt(converted.count, 10);

  return {
    due: { total: totalDue, actioned: totalActioned, pending: totalPending, by_product: products },
    converted: { count: convertedCount, rate: totalDue > 0 ? Math.round((convertedCount / totalDue) * 1000) / 10 : 0 },
    lapsed: { count: lapsedCount, won_back: lapsedWon, rate: (lapsedCount + lapsedWon) > 0 ? Math.round((lapsedWon / (lapsedCount + lapsedWon)) * 1000) / 10 : 0 },
  };
}
```

- [ ] **Step 4: Add rotation pipeline query**

```javascript
async function getRotationPipeline(tenantId) {
  const rows = await db.raw(`
    SELECT t.target_sku, t.task_type, t.status, COUNT(DISTINCT t.customer_id) as count
    FROM tasks t
    WHERE t.tenant_id = ?
      AND t.task_type IN ('ROTATE_KNOWLEDGE', 'ROTATE_TESTIMONIAL', 'ROTATE_TIPS', 'ROTATE_CHECKIN')
    GROUP BY t.target_sku, t.task_type, t.status
  `, [tenantId]);

  // Build per-product pipeline
  // For each customer-product pair, find their current stage
  // Filter to delivered orders only (same as onboarding)
  const customerProductTasks = await db.raw(`
    SELECT t.customer_id, t.target_sku, t.task_type, t.status
    FROM tasks t
    JOIN orders o ON t.order_id = o.id
    WHERE t.tenant_id = ?
      AND t.task_type IN ('ROTATE_KNOWLEDGE', 'ROTATE_TESTIMONIAL', 'ROTATE_TIPS', 'ROTATE_CHECKIN')
      AND o.delivery_confirmed = true
    ORDER BY t.target_sku, t.customer_id
  `, [tenantId]);

  const productData = {};

  for (const row of customerProductTasks.rows) {
    const key = `${row.customer_id}:${row.target_sku}`;
    if (!productData[row.target_sku]) {
      productData[row.target_sku] = {
        customers: new Set(),
        stages: {},
        _customerTasks: {},
      };
    }
    productData[row.target_sku].customers.add(row.customer_id);
    if (!productData[row.target_sku]._customerTasks[row.customer_id]) {
      productData[row.target_sku]._customerTasks[row.customer_id] = {};
    }
    productData[row.target_sku]._customerTasks[row.customer_id][row.task_type] = row.status;
  }

  const results = [];
  for (const [sku, data] of Object.entries(productData)) {
    const stages = {};
    for (const type of ROTATION_ORDER) {
      stages[type] = { total: 0, actioned: 0, pending: 0 };
    }
    stages.DONE = { total: 0 };

    for (const tasks of Object.values(data._customerTasks)) {
      let currentStage = 'DONE';
      for (const type of ROTATION_ORDER) {
        const status = tasks[type];
        if (status && !COMPLETED_STATUSES.includes(status)) {
          currentStage = type;
          break;
        }
      }
      if (currentStage === 'DONE') {
        stages.DONE.total++;
      } else {
        stages[currentStage].total++;
        if (tasks[currentStage] === 'pending') {
          stages[currentStage].pending++;
        } else {
          stages[currentStage].actioned++;
        }
      }
    }

    delete data._customerTasks;
    results.push({
      sku,
      total_customers: data.customers.size,
      stages,
    });
  }

  // Sort by customer count descending
  results.sort((a, b) => b.total_customers - a.total_customers);
  return results;
}

module.exports = { getOnboardingPipeline, getMembershipPipeline, getReorderPipeline, getRotationPipeline };
```

- [ ] **Step 5: Commit**

```bash
git add server/src/models/pipelineModel.js
git commit -m "feat: pipeline model — onboarding, membership, reorder, rotation stage derivation"
```

---

### Task 3: New Boss View Model Functions + Controller + Routes

**Files:**
- Modify: `server/src/models/bossViewModel.js`
- Modify: `server/src/controllers/bossViewController.js`
- Modify: `server/src/routes/bossView.js`

- [ ] **Step 1: Add `getLifetimeComposition()` to bossViewModel.js**

Append to bossViewModel.js:

```javascript
async function getLifetimeComposition(tenantId) {
  const region = await db('customers')
    .where('tenant_id', tenantId)
    .select('region')
    .count('id as count')
    .groupBy('region');

  const membership = await db('customers')
    .where('tenant_id', tenantId)
    .select(
      db.raw("COUNT(*) as total"),
      db.raw("COUNT(CASE WHEN membership_status = 'registered' THEN 1 END) as registered"),
      db.raw("COUNT(CASE WHEN membership_verified_status = 'active' THEN 1 END) as verified"),
    )
    .first();

  const total = parseInt(membership.total, 10);
  const registered = parseInt(membership.registered, 10);
  const verified = parseInt(membership.verified, 10);

  return {
    region: region.map(r => ({ region: r.region || 'Unknown', count: parseInt(r.count, 10) })),
    membership: {
      registered, not_registered: total - registered,
      rate: total > 0 ? Math.round((registered / total) * 1000) / 10 : 0,
      verified,
    },
  };
}
```

- [ ] **Step 2: Add `getNewSalesFunnel()` and `getRepeatSalesFunnel()`**

```javascript
async function getNewSalesFunnel(tenantId, yearMonth) {
  const [year, month] = yearMonth.split('-').map(Number);
  const monthStart = `${yearMonth}-01`;
  const daysInMonth = new Date(year, month, 0).getDate();
  const monthEnd = `${yearMonth}-${String(daysInMonth).padStart(2, '0')}`;

  const orders = await db('orders')
    .where('tenant_id', tenantId)
    .whereNot('order_status', 'cancelled')
    .where('is_first_order', true)
    .whereBetween('order_date', [monthStart, monthEnd])
    .select(
      db.raw('COUNT(*) as conversions'),
      db.raw('COALESCE(SUM(total_pv), 0) as total_pv'),
    )
    .first();

  const reach = await db('meta_daily_metrics')
    .where('tenant_id', tenantId)
    .whereBetween('date', [monthStart, monthEnd])
    .select(db.raw('COALESCE(SUM(reach), 0) as total_reach'))
    .first();

  const conversions = parseInt(orders.conversions, 10);
  const totalPv = parseFloat(orders.total_pv) || 0;
  const totalReach = parseInt(reach.total_reach, 10) || 0;

  return {
    reach: totalReach,
    conversions,
    rate: totalReach > 0 ? Math.round((conversions / totalReach) * 1000) / 10 : 0,
    basket_size: conversions > 0 ? Math.round((totalPv / conversions) * 100) / 100 : 0,
  };
}

async function getRepeatSalesFunnel(tenantId, yearMonth) {
  const [year, month] = yearMonth.split('-').map(Number);
  const monthStart = `${yearMonth}-01`;
  const daysInMonth = new Date(year, month, 0).getDate();
  const monthEnd = `${yearMonth}-${String(daysInMonth).padStart(2, '0')}`;

  // Expiring customers: order_items with end_date within 30 days from now
  const expiring = await db('order_items')
    .join('orders', 'order_items.order_id', 'orders.id')
    .where('orders.tenant_id', tenantId)
    .whereNot('orders.order_status', 'cancelled')
    .where('orders.delivery_confirmed', true)
    .whereBetween('order_items.end_date', [db.raw('NOW()'), db.raw("NOW() + INTERVAL '30 days'")])
    .countDistinct('orders.customer_id as count')
    .first();

  const orders = await db('orders')
    .where('tenant_id', tenantId)
    .whereNot('order_status', 'cancelled')
    .where('is_first_order', false)
    .whereBetween('order_date', [monthStart, monthEnd])
    .select(
      db.raw('COUNT(*) as conversions'),
      db.raw('COALESCE(SUM(total_pv), 0) as total_pv'),
    )
    .first();

  const expiringCount = parseInt(expiring.count, 10) || 0;
  const conversions = parseInt(orders.conversions, 10);
  const totalPv = parseFloat(orders.total_pv) || 0;

  return {
    expiring: expiringCount,
    conversions,
    rate: expiringCount > 0 ? Math.round((conversions / expiringCount) * 1000) / 10 : 0,
    basket_size: conversions > 0 ? Math.round((totalPv / conversions) * 100) / 100 : 0,
  };
}
```

Also add `getLifetimeCustomerMix()` for the Tab 1 doughnut (lifetime new vs repeat, not month-scoped):

```javascript
async function getLifetimeCustomerMix(tenantId) {
  const row = await db('orders')
    .where('tenant_id', tenantId)
    .whereNot('order_status', 'cancelled')
    .select(
      db.raw('COUNT(CASE WHEN is_first_order THEN 1 END) as new_orders'),
      db.raw('COUNT(CASE WHEN NOT is_first_order THEN 1 END) as repeat_orders'),
      db.raw('COALESCE(SUM(CASE WHEN is_first_order THEN total_pv ELSE 0 END), 0) as new_pv'),
      db.raw('COALESCE(SUM(CASE WHEN NOT is_first_order THEN total_pv ELSE 0 END), 0) as repeat_pv'),
    )
    .first();

  return {
    new_customers: parseInt(row.new_orders, 10),
    repeat_customers: parseInt(row.repeat_orders, 10),
    new_pv: parseFloat(row.new_pv) || 0,
    repeat_pv: parseFloat(row.repeat_pv) || 0,
  };
}
```

Add to module.exports: `getLifetimeComposition, getLifetimeCustomerMix, getNewSalesFunnel, getRepeatSalesFunnel`

Add route: `router.get('/lifetime-customer-mix', bossViewController.getLifetimeCustomerMix);`

- [ ] **Step 3: Add controller handlers in bossViewController.js**

```javascript
const pipelineModel = require('../models/pipelineModel');

async function getLifetimeComposition(req, res, next) {
  try {
    const data = await bossViewModel.getLifetimeComposition(req.tenantId);
    res.json(data);
  } catch (err) { next(err); }
}

async function getNewSalesFunnel(req, res, next) {
  try {
    const yearMonth = req.query.month || getCurrentYearMonth();
    const data = await bossViewModel.getNewSalesFunnel(req.tenantId, yearMonth);
    res.json(data);
  } catch (err) { next(err); }
}

async function getRepeatSalesFunnel(req, res, next) {
  try {
    const yearMonth = req.query.month || getCurrentYearMonth();
    const data = await bossViewModel.getRepeatSalesFunnel(req.tenantId, yearMonth);
    res.json(data);
  } catch (err) { next(err); }
}

async function getPipelineOnboarding(req, res, next) {
  try {
    const data = await pipelineModel.getOnboardingPipeline(req.tenantId);
    res.json(data);
  } catch (err) { next(err); }
}

async function getPipelineMembership(req, res, next) {
  try {
    const data = await pipelineModel.getMembershipPipeline(req.tenantId);
    res.json(data);
  } catch (err) { next(err); }
}

async function getPipelineReorder(req, res, next) {
  try {
    const yearMonth = req.query.month || getCurrentYearMonth();
    const data = await pipelineModel.getReorderPipeline(req.tenantId, yearMonth);
    res.json(data);
  } catch (err) { next(err); }
}

async function getPipelineRotation(req, res, next) {
  try {
    const data = await pipelineModel.getRotationPipeline(req.tenantId);
    res.json(data);
  } catch (err) { next(err); }
}
```

Add all to module.exports.

- [ ] **Step 4: Add routes in bossView.js**

```javascript
router.get('/lifetime-composition', bossViewController.getLifetimeComposition);
router.get('/new-sales-funnel', bossViewController.getNewSalesFunnel);
router.get('/repeat-sales-funnel', bossViewController.getRepeatSalesFunnel);
router.get('/pipeline/onboarding', bossViewController.getPipelineOnboarding);
router.get('/pipeline/membership', bossViewController.getPipelineMembership);
router.get('/pipeline/reorder', bossViewController.getPipelineReorder);
router.get('/pipeline/rotation', bossViewController.getPipelineRotation);
```

- [ ] **Step 5: Commit**

```bash
git add server/src/models/bossViewModel.js server/src/controllers/bossViewController.js server/src/routes/bossView.js server/src/models/pipelineModel.js
git commit -m "feat: boss view v2 — pipeline, composition, funnel endpoints"
```

---

### Task 4: Ad Catalog Sync + Model

**Files:**
- Create: `server/src/models/adCatalogModel.js`
- Create: `server/src/services/adCatalogSync.js`
- Create: `server/src/controllers/adCatalogController.js`
- Modify: `server/src/routes/meta.js`
- Modify: `server/src/index.js` — add sync to daily cron

- [ ] **Step 1: Create adCatalogModel.js**

```javascript
const db = require('../db');
const TABLE = 'meta_ad_catalog';

async function listWithConversions(tenantId, { search } = {}) {
  const query = db(TABLE)
    .where(`${TABLE}.tenant_id`, tenantId)
    .select(
      `${TABLE}.*`,
      db.raw(`(SELECT COUNT(*) FROM orders WHERE orders.meta_ad_id = ${TABLE}.meta_ad_id AND orders.tenant_id = ${TABLE}.tenant_id AND orders.order_status != 'cancelled') as conversion_count`)
    )
    .orderByRaw('conversion_count DESC NULLS LAST');

  if (search) {
    query.where(function () {
      this.whereILike('ad_name', `%${search}%`)
        .orWhereILike('campaign_name', `%${search}%`);
    });
  }
  return query;
}

async function upsert(data, tenantId) {
  const existing = await db(TABLE)
    .where({ tenant_id: tenantId, meta_ad_id: data.meta_ad_id })
    .first();

  if (existing) {
    return db(TABLE).where({ id: existing.id })
      .update({ ...data, updated_at: db.fn.now(), last_synced_at: db.fn.now() })
      .returning('*').then(r => r[0]);
  }
  return db(TABLE)
    .insert({ ...data, tenant_id: tenantId, last_synced_at: db.fn.now() })
    .returning('*').then(r => r[0]);
}

module.exports = { listWithConversions, upsert };
```

- [ ] **Step 2: Create adCatalogSync.js**

```javascript
const db = require('../db');
const adCatalogModel = require('../models/adCatalogModel');
const META_API_BASE = 'https://graph.facebook.com/v21.0';

async function syncAdCatalog(tenantId) {
  const settings = await db('tenant_settings').where('tenant_id', tenantId).first();
  if (!settings?.meta_api_token || !settings?.meta_ad_account_id) return { synced: 0 };

  const token = settings.meta_api_token;
  const adAccountId = settings.meta_ad_account_id;

  const url = `${META_API_BASE}/${adAccountId}/ads?fields=id,name,campaign_id,campaign{name,objective},creative{thumbnail_url},effective_status&limit=500&access_token=${token}`;

  const response = await fetch(url);
  const data = await response.json();
  if (data.error) throw new Error(`Meta API error: ${data.error.message}`);

  let synced = 0;
  const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);

  for (const ad of (data.data || [])) {
    // Only sync active or recently paused
    const isActive = ad.effective_status === 'ACTIVE';
    const isPaused = ad.effective_status === 'PAUSED';
    if (!isActive && !isPaused) continue;

    await adCatalogModel.upsert({
      meta_ad_id: ad.id,
      ad_name: ad.name,
      campaign_id: ad.campaign?.id || null,
      campaign_name: ad.campaign?.name || null,
      objective: ad.campaign?.objective || null,
      thumbnail_url: ad.creative?.thumbnail_url || null,
      status: isActive ? 'active' : 'paused',
    }, tenantId);
    synced++;
  }

  return { synced };
}

module.exports = { syncAdCatalog };
```

- [ ] **Step 3: Create adCatalogController.js and mount route**

```javascript
// adCatalogController.js
const adCatalogModel = require('../models/adCatalogModel');

async function listAds(req, res, next) {
  try {
    const search = req.query.search || null;
    const ads = await adCatalogModel.listWithConversions(req.tenantId, { search });
    res.json(ads);
  } catch (err) { next(err); }
}

module.exports = { listAds };
```

In `server/src/routes/meta.js`, add:
```javascript
const adCatalogController = require('../controllers/adCatalogController');
router.get('/ad-catalog', adCatalogController.listAds);
```

- [ ] **Step 4: Add ad catalog sync to daily cron in index.js**

In the `forEachTenant` block of the daily cron handler, after the Meta sync section, add:

```javascript
// Ad catalog sync
if (tenant.meta_api_token && tenant.meta_ad_account_id) {
  try {
    const adCatalogSync = require('./services/adCatalogSync');
    tenantResult.adCatalogSync = await adCatalogSync.syncAdCatalog(tenant.id);
  } catch (err) {
    console.error(`[Cron] Ad catalog sync failed for tenant ${tenant.slug || tenant.id}:`, err.message);
    tenantResult.adCatalogSync = { error: err.message };
  }
}
```

- [ ] **Step 5: Commit**

```bash
git add server/src/models/adCatalogModel.js server/src/services/adCatalogSync.js server/src/controllers/adCatalogController.js server/src/routes/meta.js server/src/index.js
git commit -m "feat: ad catalog sync from Meta API + ad-catalog endpoint for order form dropdown"
```

---

### Task 5: Order Form — `meta_ad_id` Support

**Files:**
- Modify: `server/src/controllers/orderController.js` — allow `meta_ad_id` field on order create/update
- Modify: `server/src/models/orderModel.js` — add `meta_ad_id` to allowed fields

- [ ] **Step 1: Add `meta_ad_id` to orderModel.js update allowlist**

In `server/src/models/orderModel.js`, find the `update()` function's allowed fields array (around line 145-151). Add `'meta_ad_id'` to the array.

- [ ] **Step 2: Add `meta_ad_id` to orderController.js create and update**

In `server/src/controllers/orderController.js`:
- In `createOrder` handler (around line 190): extract `meta_ad_id` from `req.body` and include it in the order data object passed to the model.
- In `updateOrder` handler (around line 403-406): add `'meta_ad_id'` to the allowed fields array.

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: orders accept meta_ad_id field for ad attribution"
```

---

### Task 6: Update SCHEMA.md + Deploy Backend

**Files:**
- Modify: `SCHEMA.md`

- [ ] **Step 1: Add meta_ad_catalog table and orders.meta_ad_id to SCHEMA.md**

- [ ] **Step 2: Commit all docs**

```bash
git commit -m "docs: add meta_ad_catalog table + orders.meta_ad_id to SCHEMA.md"
```

- [ ] **Step 3: Merge and deploy**

```bash
git checkout dev && git merge feature/boss-view-v2 --no-edit
git checkout staging && git merge dev --no-edit && git push origin staging
# Wait for Coolify auto-deploy, then test API endpoints
```

- [ ] **Step 4: Test all new endpoints via curl**

```bash
# Read token from tenant_settings or use the one stored in dashboard localStorage
TOKEN="$DASHBOARD_TOKEN"
BASE="https://app.solworks.io/api/boss-view"

curl -s "$BASE/lifetime-composition" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s "$BASE/new-sales-funnel?month=2026-04" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s "$BASE/repeat-sales-funnel?month=2026-04" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s "$BASE/pipeline/onboarding" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s "$BASE/pipeline/membership" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s "$BASE/pipeline/reorder?month=2026-04" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s "$BASE/pipeline/rotation" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

---

## Phase B: Frontend Dashboard Rewrite

### Task 7: Rewrite index.html — Tab Structure + Design System + Tab 1

**Files:**
- Modify: `~/Documents/minionions-dashboard/index.html`

This is a full rewrite of the existing 1,194-line file. The new version uses:
- Outfit + Geist Mono fonts (replacing Fira Sans + Fira Code)
- Tab navigation (Lifetime / Monthly / Ads)
- Design system from mockups (zinc/slate neutral, 1px gap grids, no card boxing for KPIs)

- [ ] **Step 1: Rewrite with tab structure, password gate, config bar, Tab 1 (Lifetime)**

Reference the mockup at `.superpowers/brainstorm/81986-1776992803/tab1-lifetime.html` for the design.

**Important:** Tab 2 (Monthly) is the default landing tab, not Tab 1. On page load, show Tab 2. Tab 1 is accessible via click.

Tab 1 reuses existing API endpoints (`/kpi`, `/monthly-trend`, `/customer-mix`) plus the new `/lifetime-composition` endpoint. Charts reuse existing Chart.js rendering functions.

- [ ] **Step 2: Test Tab 1 locally**

```bash
cd ~/Documents/minionions-dashboard && python3 -m http.server 3333
# Open http://localhost:3333, enter password, configure API, verify Tab 1 loads
```

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: dashboard v2 — tab structure, design system, Tab 1 (Lifetime)"
```

---

### Task 8: Tab 2 — Sales Overview + New/Repeat Funnels

**Files:**
- Modify: `~/Documents/minionions-dashboard/index.html`

- [ ] **Step 1: Build Tab 2 sales section**

Reference mockup at `.superpowers/brainstorm/81986-1776992803/tab2-full.html`.

Sections to build:
- Sales Overview: 5 KPI cards + monthly goal bar + weekly breakdown + daily trend
- New Sales + Repeat Sales side by side with funnel KPIs and charts
- Ad Spend Summary with clickable "Full detail in Ads tab" link

Uses existing endpoints (`/kpi`, `/weekly-target`, `/daily`, `/ad-daily`) plus new (`/new-sales-funnel`, `/repeat-sales-funnel`).

- [ ] **Step 2: Test Tab 2 sales section locally**

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: Tab 2 — sales overview, new/repeat funnels, ad spend summary"
```

---

### Task 9: Tab 2 — Follow-up Pipeline

**Files:**
- Modify: `~/Documents/minionions-dashboard/index.html`

- [ ] **Step 1: Build pipeline section**

Reference mockup at `.superpowers/brainstorm/81986-1776992803/pipeline-v6.html`.

Three tracks:
1. Onboarding + Membership (side by side) — flow stages with arrows, progress bars, danger states
2. Reorder Pipeline — funnel flow (Due → Converted → Lapsed)
3. Rotation Track — per-product rows with 4 stages + Done, "Show X more" toggle

Uses new endpoints: `/pipeline/onboarding`, `/pipeline/membership`, `/pipeline/reorder`, `/pipeline/rotation`.

Danger state logic: completion < 30% → red border + bg. 30-50% → orange. > 50% → clean.

- [ ] **Step 2: Test pipeline with real data**

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: Tab 2 — follow-up pipeline with onboarding, reorder, rotation tracks"
```

---

### Task 10: Tab 3 Placeholder + MoM Deltas + Final Polish

**Files:**
- Modify: `~/Documents/minionions-dashboard/index.html`

- [ ] **Step 1: Add Tab 3 placeholder**

"Coming soon" message with bullet list of planned features.

- [ ] **Step 2: Add MoM delta indicators**

For Tab 1: derive from `/monthly-trend?months=2` (current vs previous month contribution).
For Tab 2: call `/kpi` twice (current + previous month), compute deltas client-side.

Delta display: green arrow up + percentage for positive, red arrow down for negative.

- [ ] **Step 3: Add responsive breakpoints**

Test at 375px, 768px, 1280px. Ensure:
- KPIs stack to 2-col on mobile
- Side-by-side sections stack
- Pipeline stages wrap to 2-per-row
- Charts resize

- [ ] **Step 4: Commit and push**

```bash
git add index.html
git commit -m "feat: Tab 3 placeholder, MoM deltas, responsive polish"
git push origin main
```

---

### Task 11: Deploy + Verify

- [ ] **Step 1: Deploy CRM backend to production**

```bash
cd ~/Documents/mom-crm-webapp
git checkout main && git merge staging --no-edit && git push origin main
```

- [ ] **Step 2: Run ad catalog sync manually**

Trigger via daily cron or direct API call to populate `meta_ad_catalog` for the first time.

- [ ] **Step 3: Redeploy dashboard**

```bash
cd ~/Documents/minionions-dashboard && git push origin main
# Trigger Coolify redeploy
```

- [ ] **Step 4: End-to-end verification**

Run through the verification plan from the spec:
1. Tab 1 KPIs match existing dashboard
2. Tab 2 Sales match `/kpi` data
3. Pipeline stages verified against 5 sample customers in DB
4. Reorder count matches pending REORDER tasks
5. MoM deltas are correct for 2 metrics
6. "Full detail in Ads tab" link switches tabs
7. Mobile responsive at 375px
8. Danger states render correctly

- [ ] **Step 5: Commit session docs**

Update STATUS.md, DECISIONS.md, SCHEMA.md, Efforts files.

---

## Phase C: CRM Order Form — Ad Attribution Dropdown

> **Note:** This phase modifies the CRM React frontend, not the dashboard. It can be done independently after Phase A deploys the ad catalog API.

### Task 12: Order Form Dropdown

**Files:**
- Modify: `client/src/pages/NewOrder.jsx` — replace `ad_source` text input with searchable dropdown
- Modify: `client/src/pages/OrderDetail.jsx` — show attributed ad with thumbnail

- [ ] **Step 1: Read NewOrder.jsx and OrderDetail.jsx to understand current ad_source field placement**

- [ ] **Step 2: Create an AdAttributionDropdown component**

Searchable dropdown that:
- Fetches `/api/meta/ad-catalog` on mount
- Groups: Meta Ads (sorted by conversion_count DESC) | Other (Organic, Referral)
- Shows thumbnail + ad name + campaign name per option
- On select, sets both `ad_source` (for backward compat) and `meta_ad_id`

- [ ] **Step 3: Wire into NewOrder.jsx and OrderDetail.jsx**

- [ ] **Step 4: Test order creation with ad attribution**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: ad attribution dropdown on order form — replaces free-text ad_source"
```
